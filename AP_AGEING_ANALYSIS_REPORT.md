# AP Ageing Analysis Report - Technical Documentation

**Version**: 1.8
**Date**: 2026-06-22
**Author**: System Analysis
**Status**: Stable — Opening debit notes now show, NULL branch_id matched by supplier_id (v1.8); Per-company exchange rate + cost's booked rate so report matches the document (v1.7); Single supplier total row + end-of-report supplier summary with grand total (v1.2); Currency Conversion Fix (v1.1)

---

## Table of Contents

1. [Report Purpose](#report-purpose)
2. [Technical Architecture](#technical-architecture)
3. [Data Flow](#data-flow)
4. [Calculation Logic](#calculation-logic)
5. [Currency Conversion](#currency-conversion)
6. [User Interface](#user-interface)
7. [Output Formats](#output-formats)
8. [Known Issues & Fixes](#known-issues--fixes)
9. [Code References](#code-references)

---

## Report Purpose

### Business Objectives

1. **Payables Management**: Monitor outstanding amounts owed to suppliers
2. **Cash Flow Planning**: Forecast upcoming payment obligations
3. **Supplier Relationship**: Track overdue payments to maintain supplier trust
4. **Credit Utilization**: Monitor supplier credit limit usage
5. **Aging Visibility**: Categorize payables into aging buckets for prioritization

### Key Metrics Provided

- Outstanding cost amounts per supplier (in PKR)
- Aging breakdown: Current, 1-30, 31-60, 61-90, 91-120, 121+ days
- Days overdue per XO document
- Supplier credit limits and credit days
- Debit note adjustments (reduce outstanding)
- Supplier advance payments/deposits (reduce outstanding)

---

## Technical Architecture

### Frontend Components

**File**: `psfront/src/pages/Report/APAgeingAnalysisReport.jsx`

```
React Component Structure:
├── Filter Form
│   ├── Supplier Number Filter (isNotBlank, isBlank, isEqual, between)
│   │   └── LiveComboBox (server-side search via /supplier)
│   ├── As Of Date (default: today, single DateInput)
│   ├── Cost Date Filter (isNotBlank, isBlank, =, <, <=, >, >=, <>, between)
│   │   └── DateInput (single or range based on operator)
│   └── Branch Filter (isNotBlank, isBlank, isEqual, between)
│       └── Combobox (client-side, pre-loaded up to 1000 branches)
├── Generate Button (dropdown: PDF / Excel)
└── Report Viewer (navigates to /reports/:report_number)
```

### Backend Controller

**File**: `psback/controllers/report.controller.js`
**Function**: `exports.getAPAgeingAnalysisReport`

### API Endpoint

- **Route**: `POST /report/getAPAgeingAnalysisReport`
- **Authentication**: JWT required
- **Permission**: `Ap-Ageing-Analysis-Report`
- **Timeout**: 30 seconds (configured in index.js)

### Database Tables Used

| Table | Purpose |
|-------|---------|
| `suppliers` | Supplier info, credit limit, credit days |
| `costs` | Cost records (published_rate, commission, currency, etc.) |
| `cost_taxes` | Tax amounts per cost |
| `services` | Links costs to orders and suppliers |
| `orders` | Branch filtering, order numbers |
| `documents` | XO document numbers |
| `payment_settlement_costs` | Settlement amounts (in PKR) |
| `debit_notes` | Debit note adjustments |
| `supplier_deposits` | Advance payments |
| `currency_codes` | Maps currency ID → currency code (e.g., 118 → SAR) |
| `currencies` | Exchange rates (from_currency → PKR) |

---

## Data Flow

```
1. Frontend sends filter params → POST /report/getAPAgeingAnalysisReport
2. Backend fetches suppliers matching filter (company_code scoped)
3. Pre-fetches exchange rates from currency_codes + currencies tables
4. For each supplier:
   a. Fetch costs (status: Printed/Partially Paid) with:
      - Service → Order → Branch, TCID, Customer
      - Service → ServiceType (for Air date logic)
      - Cost taxes
      - Documents (costing type)
      - Payment settlements
   b. Apply date filter (cost created_at)
   c. Calculate cost amount in original currency
   d. Convert to PKR using exchange rate
   e. Subtract settled amounts → outstanding
   f. Skip fully settled costs
   g. Group by XO document number
   h. Calculate aging bucket based on (asOfDate - dueDate)
   i. Add debit notes (negative amounts, reduce outstanding)
   j. Add supplier deposits (negative amounts, reduce outstanding)
5. Generate PDF (EJS template) or Excel (ExcelJS)
6. Upload to S3/MinIO storage
7. Return report record with report_number
```

---

## Calculation Logic

### Cost Amount Calculation (per cost record)

The cost amount is calculated in the **original currency** first, matching the XO Document display:

```
commissionAmount = published_rate × (commission% / 100)
netRate = published_rate - commissionAmount
sstAmount = commissionAmount × (sst% / 100)
taxAmount = SUM(cost_taxes.tax_amount)
costPerUnit = netRate + free_of_cost + taxAmount + sstAmount
totalCostLocal = costPerUnit × quantity
```

### Currency Conversion to PKR

```
currencyCode = currency_codes[cost.currency] → e.g., "SAR", "USD"
exchangeRate = cost.exchange_rate (the rate the cost was booked at)
             → fallback: currencies[currencyCode → PKR] for THIS company only
totalCostPKR = totalCostLocal × exchangeRate
```

The report prefers the **rate stored on the cost** (`cost.exchange_rate`) so the PKR
figure matches the costing document and its journal entry. The live `currencies`
lookup is only a fallback for costs with no stored rate, and is **scoped to the
logged-in company** (`company_code`) so it can never pick another company's rate.

For PKR costs, exchange rate = 1 (no conversion needed).

### Outstanding Amount

```
totalSettled = SUM(payment_settlement_costs.amount)  // already in PKR
totalOutstanding = totalCostPKR - totalSettled
```

If `totalOutstanding <= 0`, the cost is fully settled and skipped.

### Aging Bucket Assignment

```
dueDate = documentDate + creditDays
daysOverdue = asOfDate - dueDate

if daysOverdue <= 0  → Current (Within Credit Period)
if daysOverdue 1-30  → 1-30 Days
if daysOverdue 31-60 → 31-60 Days
if daysOverdue 61-90 → 61-90 Days
if daysOverdue 91-120 → 91-120 Days
if daysOverdue > 120  → 121+ Days
```

### Document Date Logic

- **Air services**: Uses `ticket_issue_date` (fallback: `cost.created_at`)
- **Other services**: Uses `document.created_at` (fallback: `cost.created_at`)

### XO Grouping

Multiple cost line items under the same XO document number are grouped together. Their outstanding amounts are summed into a single row.

### Debit Notes

- `doc_status = 'Printed'`, then matched company-safely via an OR:
  - **Normal DNs**: `supplier_name` match **AND** `branch_id` in the company's branches.
  - **Opening DNs**: `supplier_id = this supplier's id`. Opening DNs are imported with
    `branch_id = NULL` (so the branch filter alone drops them) but carry a `supplier_id`,
    which is inherently this company's supplier — no cross-company leak.
- Amount is **negated** (reduces outstanding)
- Order number fetched from linked refund record (blank for opening DNs, which have none)

### Supplier Deposits (Advance Payments)

- Queried by `supplier_id`, `status IN ('Printed', 'Partially Paid')`, `current_amount > 0`
- Amount is **negated** (reduces outstanding)

---

## Currency Conversion

### How It Works

The `costs` table stores amounts in the original currency (SAR, USD, AED, CNY, etc.). The `currency` field stores the `currency_codes.id` (e.g., 118 for SAR) or sometimes the string "PKR".

Exchange rates are pre-fetched at report generation time:

1. `currency_codes` table maps ID → currency code (e.g., 118 → "SAR")
2. `currencies` table maps currency → PKR exchange rate (e.g., SAR → 77.10)
3. For each cost, look up the exchange rate and multiply

### Example: LYXO00000005

| Field | Value |
|-------|-------|
| published_rate | 302.00 SAR |
| commission | 0% |
| free_of_cost (extra charges) | 90.00 SAR |
| net_rate | 302.00 SAR |
| costPerUnit | 302 + 90 = 392.00 SAR |
| exchange rate | 1 SAR = 77.1 PKR |
| **Total in PKR** | **392 × 77.1 = 30,223.20** |

### Important Note on `total_costing` Field

The `total_costing` field in the `costs` table is **unreliable** for PKR conversion:
- Some records include `free_of_cost` in the total, some don't
- The field is inconsistently populated depending on when/how the cost was saved
- **Do NOT use `total_costing` for report calculations** — always recalculate from raw fields + exchange rate

---

## User Interface

### Filter Options

| Filter | Type | Options |
|--------|------|---------|
| Supplier Number | Dropdown + Search | isNotBlank, isBlank, isEqual, between |
| As Of Date | Date Input | Default: today |
| Cost Date | Dropdown + Date | isNotBlank, isBlank, =, <, <=, >, >=, <>, between |
| Branch | Dropdown + Combobox | isNotBlank, isBlank, isEqual, between |

### Report Columns

| Column | Description |
|--------|-------------|
| Doc No. | XO document number (linked to costing page) |
| TC Code | Travel consultant code |
| Order No. | Booking/order number |
| Description | "XO", "Debit Note", or "Advance Payment" |
| Doc Date | Document date (YYYY-MM-DD) |
| Days Overdue | Number of days past due date |
| Current | Amount within credit period |
| 1-30 Days | Amount 1-30 days overdue |
| 31-60 Days | Amount 31-60 days overdue |
| 61-90 Days | Amount 61-90 days overdue |
| 91-120 Days | Amount 91-120 days overdue |
| 121+ Days | Amount over 120 days overdue |
| Total Outstanding | Total outstanding amount |

### Supplier Section

Each supplier shows:
- Header: Supplier name, number, credit limit
- Individual line items (XOs, debit notes, deposits)
- **Supplier Total PKR**: Single total row per supplier — sum of days overdue in the Days Overdue column plus sums of all aging buckets and Total Outstanding. (The duplicate "Total in PKR" row was removed in v1.2.)

### Grand Total (end of report, v1.2)

After the last supplier, both PDF and Excel show a single **Grand Total** row (separated from the supplier sections by a blank spacer row): column-wise sums of the 7 amount columns (Current … 121+ Days, Total Outstanding) across all suppliers. Excel renders it with a dark fill and white bold text. Suppliers with no data rows contribute nothing. (A per-supplier summary table was briefly added in v1.2 and removed per client requirement — only the Grand Total row remains.)

---

## Output Formats

### PDF
- **Template**: `psback/views/pages/reports/ap_ageing_analysis.ejs`
- **Layout**: A4 Landscape
- **Features**: Clickable XO links, supplier grouping, totals per supplier

### Excel
- **Library**: ExcelJS
- **Layout**: 13 columns (A..M), matching the PDF exactly.
- **Columns (same order as PDF)**: Doc No., TC Code, Order No., Description, Doc Date, Days Overdue, Current (Within Cr. Period), 1-30 Days, 31-60 Days, 61-90 Days, 91-120 Days, 121+ Days, Total Outstanding.
- **Per-supplier block** (identical structure to PDF):
  1. **Supplier header row** — `Name (supp_no)` merged across cols A..K, `Credit Limit` label in col L, value in col M (gray background, bold).
  2. **Column header row** — bold, light-gray fill, bordered. Amount columns right-aligned.
  3. **Data rows** — one per XO/Debit Note/Advance Payment, amounts locale-formatted with 2 decimals.
  4. **"Supplier Total PKR:" row** — label spans cols A..E, sum of `days_overdue` in col F, sums in cols G..M. (Single total row since v1.2; the duplicate "Total in PKR" row was removed.)
- **Grand Total row** (end of sheet, v1.2): label merged A..E, dark `#333333` fill, white bold text, column-wise sums in G..M across all suppliers.
- **Features**: Frozen header rows (first 7), supplier grouping, auto-sized columns, thin borders, formatted amounts.

---

## Known Issues & Fixes

### Fix: Opening debit notes now show (NULL branch_id) (v1.8, 2026-06-22)

**Problem**: Opening debit notes (imported supplier opening balances) never appeared on the
report, even though they are `Printed` and belong to the company.

**Root cause**: Opening and normal debit notes are stored with mutually exclusive anchors:
- **Normal** DNs always have a `branch_id` (and `supplier_id` is NULL).
- **Opening** DNs have `branch_id = NULL` (and carry a `supplier_id`).

The debit-note query required `branch_id IN (company branches)`. `NULL IN (...)` is never
true, so all opening DNs were filtered out. (Matching opening DNs by `supplier_name` is also
unreliable — e.g. a DN name `"TAHIR TESTING SUPPLIER"` vs supplier `"TAHIR TESTING SUPPLIER "`
with a trailing space.)

**Fix** (`getAPAgeingAnalysisReport`): the debit-note query now uses an OR —
`supplier_name + branch_id IN company branches` (normal DNs) **OR** `supplier_id = supplier.id`
(opening DNs). `supplier.id` is this company's supplier, so it stays company-scoped with no
leak, and no row can match both sides (normal DNs have no `supplier_id`, opening DNs have no
`branch_id`).

**Files Changed**:
- `psback/controllers/report.controller.js` — `getAPAgeingAnalysisReport`,
  `getAPAgeingAnalysisSummaryReport`, `getAPAgeingAnalysisDetailReport` (debit-note query)

**Scope**: The opening-DN fix was applied to all three variants (Report, Summary, Detail).
The **exchange-rate + manual-JE** fix (v1.7) is now in **Report and Summary**; the **Detail**
report still keeps the old unscoped, live-rate lookup and doesn't subtract manual JE.

### Fix: Per-company rate + booked rate so report matches document (v1.7, 2026-06-22)

**Problem**: For company `9876`, XO `TTXO00000023` showed **7,928 PKR** outstanding on the
report, but the costing document showed **10,000 PKR**.

**Root causes** (two faults in the currency conversion):
1. **Wrong company's rate**: the exchange-rate prefetch ran
   `SELECT ... FROM currencies WHERE to_currency='PKR' ORDER BY id DESC` with **no
   `company_code` filter**, keeping the highest-id row per currency. For AED that was
   company `1008`'s rate **75.83**, not 9876's 76.45.
2. **Wrong rate basis**: it re-converted at a live `currencies` rate instead of the rate the
   cost was actually booked at (`cost.exchange_rate`). The cost was 800 AED booked at
   **78.42** → 62,736 PKR; the report used 75.83 → 60,664.
   - Report: 800 × 75.83 = 60,664 − 17,236 (payment) − 35,500 (Manual JE) = **7,928**.
   - Document: 800 × 78.42 = 62,736 − 52,736 = **10,000**.

**Fix** (`getAPAgeingAnalysisReport`):
1. The fallback rate query now filters `AND company_code = :companyCode`, so it can only ever
   use the logged-in company's rates.
2. Each cost now converts using its **own stored `cost.exchange_rate`** (the booked rate);
   the company-scoped live rate is used only when the cost has no stored rate. PKR costs stay
   at rate 1. `TTXO00000023` now reads **10,000**, matching the document.

**Files Changed**:
- `psback/controllers/report.controller.js` — `getAPAgeingAnalysisReport` (rate prefetch query + per-cost conversion)

**Scope**: Report variant only. The **Summary** and **Detail** functions still use the old
unscoped, live-rate pattern (unchanged).

### Fix: Base Currency + Voided Settlements (v1.3, 2026-06-17)

**Problem**: For a company whose base currency is not PKR (e.g. company `1010` with
`EUR → USD`), a foreign cost showed a huge wrong figure (EUR 451.04 appeared as
**146,617**), and the cost row could vanish entirely.

**Root causes**:
1. The exchange-rate lookup was hard-coded to `to_currency = 'PKR'` **with no
   `company_code` filter**, so a foreign cost was multiplied by *another* company's
   `from→PKR` rate instead of this company's `from→base` rate.
2. The "amount settled" sum included **voided** payment settlements, so an over-counted
   settlement pushed the outstanding ≤ 0 and the report skipped the row.

**Fix** (`getAPAgeingAnalysisReport`):
1. Derive the company **base currency** = `currencies.to_currency` for that company
   (default `PKR`), and query rates with `to_currency = <base> AND company_code = <this
   company>`. EUR→USD now uses 1.15 → cost reads **USD 518.70**.
2. Exclude settlements whose `payment_settlement.status === 'Void'` (the cost-query
   include now joins `payment_settlement`). Outstanding = 518.70 − live-settled.
3. The "Supplier Total **PKR**" label now follows the base currency
   (`header.currency` / `apReportBaseCurrency`) → "Supplier Total **USD**".

**Files Changed**:
- `psback/controllers/report.controller.js` — `getAPAgeingAnalysisReport`
- `psback/views/pages/reports/ap_ageing_analysis.ejs` — supplier-total currency label

**Scope**: This (Report) variant. The **Detail** and **Summary** reports got the same
fix (see their docs); the **AR** ageing reports still use the old hard-coded-PKR pattern.

### Fix: Credit Limit formatted as accounting amount (v1.6, 2026-06-19)

**Problem**: The per-supplier **Credit Limit** value printed as a raw number (`60000000.00`),
left-aligned — unlike every other money column.

**Fix** (`ap_ageing_analysis.ejs`, PDF view): the Credit Limit cell now runs the value through
the same `formatAmount()` helper and is `text-align:right`, so it shows as `60,000,000.00`,
right-aligned, matching the amount columns.

**Files Changed**:
- `psback/views/pages/reports/ap_ageing_analysis.ejs` — Credit Limit cell formatting

**Scope**: PDF view only. (The Excel export's Credit Limit cell was not part of this request.)

### Fix: Opening XO numbers now shown instead of `COST-<id>` (v1.5, 2026-06-18)

**Problem**: Opening-balance rows showed placeholder labels like `COST-6752` … `COST-6755`
instead of a real XO number.

**Root cause**: These are **opening-balance imported costs** (`costs.is_opening = 1`,
`import_batch_id` set, no order, no costing document). The XO number was taken only from the
costing `document`, which opening costs don't have — so the code fell back to `COST-<cost.id>`.
The real number was sitting unused in the cost's own `xo_number` column (e.g. `TTOX00501518`).

**Fix** (`getAPAgeingAnalysisReport`):
1. The XO-number resolution now prefers: costing `document.document_number` →
   `cost.xo_number` → `COST-<id>` (last-resort fallback only).
2. Because opening rows now carry a real `TTOX…` number, they also receive **Manual JE
   settlement** subtraction (previously skipped for `COST-*` rows) — correct, since an opening
   XO settled by manual JE should reduce its outstanding.

**Files Changed**:
- `psback/controllers/report.controller.js` — `getAPAgeingAnalysisReport` (`xoNumber` fallback)

**Scope**: Report variant only. The **Summary** and **Detail** functions are unchanged.

### Fix: Manual JE Settlements now reduce outstanding (v1.4, 2026-06-18)

**Problem**: An XO settled through a **Manual Journal Entry** still showed its full amount
outstanding. Example: Q & K `TTXO00000031` (cost 25,120 PKR) was settled 25,000 via Manual
JE `TTJV000111`, but the report kept showing **25,120** instead of the real remaining **120**.

**Root cause**: The report computed outstanding only as `cost − payment_settlement_costs`.
Manual JE settlements are not in that table — they are journal entries tagged to the XO via
`journal_entries.analysis_code1 = <document_number>` inside a live `Manual JE` batch. The
report never read them, even though cost status, AR aging, and the document picker already do
(via `services/manualJeAdjustment.js`).

**Fix** (`getAPAgeingAnalysisReport` only):
1. Import `sumManualJeAdjustment` from `services/manualJeAdjustment.js`.
2. In the XO-grouping loop, subtract `sumManualJeAdjustment(documentNumber)` from each XO
   group's outstanding (skipped for `COST-*` fallback rows that have no real XO number).
3. If the JE fully settles the XO, the row is dropped (same as fully-paid costs).
4. The helper counts **live** Manual JE only — Void batches and `VOID REVERSAL -` rows are
   excluded, so a voided JE nets back to zero.

**Note on duplicate XO numbers**: Because `document_number` is reused across documents/suppliers
in this data, a JE tagged to a number shared by two suppliers applies to both — this mirrors the
existing `recalculateCostStatusByDocNumber` behaviour and is intentionally kept consistent.

**Files Changed**:
- `psback/controllers/report.controller.js` — `getAPAgeingAnalysisReport` (import + JE subtraction)

**Scope**: Report variant only. The **Summary** and **Detail** functions still ignore Manual
JE settlements (unchanged).

### Fix: Voided Settlements actually applied in code (v1.3.1, 2026-06-18)

**Problem**: The v1.3 entry above described excluding voided settlements, but that change
was **not present in the running code** — `getAPAgeingAnalysisReport` still summed every
`payment_settlement_costs` row regardless of status. Example: Q & K cost `TTXO00000031`
(true cost **25,120 PKR**) had 4 voided settlements of 120 each; the report subtracted
480 and showed **24,640** instead of 25,120.

**Fix** (`getAPAgeingAnalysisReport` only):
1. The cost query's `payment_settlement_cost` include now nests `payment_settlement` so
   the parent settlement status is available.
2. The "amount settled" reducer **skips any row whose `payment_settlement.status === 'Void'`**.
   `TTXO00000031` now reads **25,120**.

**Known limitation (not addressed here)**: This report still derives outstanding purely
from costs minus live `payment_settlement_costs`. **Manual JE settlements** (e.g. batch
`TTJV000111` debiting Trade Creditors) post only to the GL with no link back to the cost,
so they are **not** reflected — an XO settled by manual JE still appears outstanding. The
base-currency portion of the v1.3 entry is likewise **not yet** in the code.

**Files Changed**:
- `psback/controllers/report.controller.js` — `getAPAgeingAnalysisReport` (settlement include + void filter)

**Scope**: Report variant only. The **Summary** and **Detail** functions still sum voided
settlements (unchanged).

### Change: Single Supplier Total + Supplier Summary with Grand Total (v1.2, 2026-06-11)

**Request**: Each supplier showed two total rows with identical amounts ("Total in PKR" and "Supplier Total PKR:"). Only one was needed, and the report had no overall total.

**Change**:
1. Removed the "Total in PKR" row from both PDF and Excel — "Supplier Total PKR:" (which also carries the days-overdue sum) is now the single per-supplier total row.
2. Added an end-of-report **Grand Total** row to both outputs, summing all suppliers column-wise, separated from the supplier sections by a blank spacer row. (A per-supplier summary table was initially added here, then removed per client requirement.)
3. Suppliers with no data rows contribute nothing to the grand total.

**Files Changed**:
- `psback/views/pages/reports/ap_ageing_analysis.ejs` — removed duplicate total row; added summary + grand total block after the supplier loop
- `psback/controllers/report.controller.js` — `getAPAgeingAnalysisReport` Excel section: removed duplicate total row; collects per-supplier sums into `supplierSummaries` and renders the summary + grand total block at the end of the sheet

**Scope**: AP Ageing Analysis report only — the AP Summary and AP Detail reports were not touched.

### Fix: Currency Conversion (March 2026)

**Problem**: Amounts were showing in the original currency (SAR, USD, etc.) instead of PKR. For example, LYXO00000005 showed 23,284.20 (only published_rate × exchange rate) instead of 30,223.20 (full amount including extra charges × exchange rate).

**Root Cause**: The report was using `total_costing` from the database, which inconsistently includes/excludes `free_of_cost` (extra charges). Some costs saved `total_costing` as just `net_rate × exchange_rate`, missing extra charges.

**Fix**: Changed all three AP Ageing reports (Report, Summary, Detail) to:
1. Calculate the full cost amount in the original currency (published_rate - commission + free_of_cost + taxes + sst) × quantity
2. Look up the exchange rate from `currency_codes` + `currencies` tables
3. Convert to PKR by multiplying with exchange rate

**Files Changed**:
- `psback/controllers/report.controller.js` — `getAPAgeingAnalysisReport`, `getAPAgeingAnalysisSummaryReport`, `getAPAgeingAnalysisDetailReport`

---

## Code References

| Component | File | Function/Location |
|-----------|------|-------------------|
| Frontend Page | `psfront/src/pages/Report/APAgeingAnalysisReport.jsx` | Full component |
| API Function | `psfront/src/api/report.js` | `getAPAgeingAnalysisReport` (line ~256) |
| Backend Route | `psback/routes/report.route.js` | Line 67 |
| Controller | `psback/controllers/report.controller.js` | `exports.getAPAgeingAnalysisReport` |
| PDF Template | `psback/views/pages/reports/ap_ageing_analysis.ejs` | EJS template |
| Permission | `Ap-Ageing-Analysis-Report` | Required for access |

### Related Reports

| Report | Permission | Route |
|--------|-----------|-------|
| AP Ageing Summary | `Ap-Ageing-Analysis-Summary-Report` | `/report/getAPAgeingAnalysisSummaryReport` |
| AP Ageing Detail | `Ap-Ageing-Analysis-Detail-Report` | `/report/getAPAgeingAnalysisDetailReport` |
| AR Ageing Detail | `Ar-Ageing-Analysis-Detail-Report` | `/report/getARAgeingAnalysisDetailReport` |
| AR Ageing Summary | `Ar-Ageing-Analysis-Summary-Report` | `/report/getARAgeingAnalysisSummaryReport` |
