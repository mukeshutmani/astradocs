# AP Ageing Analysis Summary Report - Technical Documentation

**Version**: 1.1
**Date**: 2026-06-17
**Author**: System Analysis
**Status**: Stable — base-currency + voided-settlement fix and 2-decimal rounding (v1.1); aligned with the AP Ageing Analysis Report

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

1. **Payables overview**: one row per supplier summarising outstanding amounts owed.
2. **Cash-flow planning**: see, at a glance, which suppliers carry the largest overdue
   balances without scrolling line-by-line documents.
3. **Aging visibility**: outstanding split into buckets (Current, 1‑30, 31‑60, 61‑90,
   91‑120, 121+ days) per supplier and overall.

This is the **summary** sibling of the **AP Ageing Analysis Report** (line items) and the
**AP Ageing Analysis Detail Report**. The numbers reconcile with both.

### Key Metrics Provided

- Outstanding payable per supplier, in the company **base currency**
- **Average Days** overdue per supplier
- Aging breakdown: Current, 1‑30, 31‑60, 61‑90, 91‑120, 121+ days
- A final **Total** row summing every column across all suppliers

---

## Technical Architecture

### Frontend Component

**File**: `psfront/src/pages/Report/APAgeingAnalysisSummaryReport.jsx`

```
React Component Structure:
├── Filter Form
│   ├── Supplier Number Filter (isNotBlank, isBlank, isEqual, between)
│   ├── As Of Date (default: today)
│   ├── Cost Date Filter (=, <, <=, >, >=, <>, between, blank)
│   └── Branch Filter (isNotBlank, isBlank, isEqual, between)
├── Generate Button (PDF / Excel)
└── Report Viewer (navigates to /reports/:report_number)
```

### Backend Controller

**File**: `psback/controllers/report.controller.js`
**Function**: `exports.getAPAgeingAnalysisSummaryReport`

### API Endpoint

- **Route**: `POST /report/getAPAgeingAnalysisSummaryReport`
- **Authentication**: JWT required
- **Permission**: `Ap-Ageing-Analysis-Summary-Report`

### Database Tables Used

| Table | Purpose |
|-------|---------|
| `suppliers` | Supplier info, credit limit, credit days, date type |
| `costs` | Cost records (published_rate, commission, currency, etc.) |
| `cost_taxes` | Tax amounts per cost |
| `services` | Links costs to orders and suppliers |
| `orders` | Branch filtering, order numbers |
| `payment_settlement_costs` | Settlement amounts (joined to `payment_settlements` for status) |
| `payment_settlements` | Settlement status — used to ignore **Void** settlements |
| `debit_notes` | Debit note adjustments (reduce payable) |
| `supplier_deposits` | Advance payments (reduce payable) |
| `currency_codes` | Maps currency ID → currency code (e.g., 159 → EUR) |
| `currencies` | Exchange rates (`from_currency → to_currency`, per company) |

---

## Data Flow

```
1. Frontend sends filter params → POST /report/getAPAgeingAnalysisSummaryReport
2. Backend fetches suppliers matching filter (company_code scoped)
3. Derives the company BASE currency and pre-fetches exchange rates
   (to_currency = base AND company_code = this company)
4. For each supplier:
   a. Fetch costs (status Printed/Partially Paid) + cost_taxes + payment settlements
   b. Apply date filter (cost created_at)
   c. Rebuild cost amount in original currency, convert to base currency
   d. Subtract settled amounts (EXCLUDING voided settlements) → outstanding
   e. Skip fully settled costs; accumulate into this supplier's aging buckets
   f. Add debit notes and supplier deposits (negative, reduce outstanding)
   g. Compute Average Days = round(sum of days-overdue / number of costs)
   h. Format the supplier's bucket amounts (2 decimals)
5. Build a Grand Total across all suppliers
6. Generate PDF (EJS) or Excel (ExcelJS), upload to S3/MinIO
7. Return the report record with report_number
```

---

## Calculation Logic

### Cost Amount (per cost record)

Rebuilt in the **original currency** (matching the XO document), never read from
`total_costing` (that field is unreliable):

```
commissionAmount = published_rate × (commission% / 100)
netRate          = published_rate − commissionAmount
sstAmount        = commissionAmount × (sst% / 100)
taxAmount        = SUM(cost_taxes.tax_amount)
costPerUnit      = netRate + free_of_cost + taxAmount + sstAmount
totalCostLocal   = costPerUnit × quantity
```

### Convert to Base Currency

```
currencyCode = currency_codes[cost.currency]              // e.g., "EUR"
exchangeRate = currencies[from_currency=EUR, to_currency=BASE, company]   // e.g., 1.15
costAmount   = totalCostLocal × exchangeRate              // base-currency value
```

Costs already in the base currency use rate 1.

### Outstanding (voided settlements ignored)

```
totalSettled     = SUM(payment_settlement_costs.amount
                       WHERE payment_settlement.status != 'Void')
totalOutstanding = costAmount − totalSettled
```

If `totalOutstanding <= 0`, the cost is fully settled and skipped.

### Aging Bucket + Average Days

```
dueDate     = documentDate + creditDays      // from supplier credit terms
daysOverdue = asOfDate − dueDate
  ≤ 0 → Current | 1-30 | 31-60 | 61-90 | 91-120 | >120 → 121+

Average Days (per supplier) = round( Σ daysOverdue / number of costs )
```

### Debit Notes & Supplier Deposits

- **Debit notes**: matched by `supplier_name`, `doc_status = Printed`, company branches;
  amount **negated**.
- **Supplier deposits (advances)**: `supplier_id`, status Printed/Partially Paid,
  `current_amount > 0`; `current_amount` (already base currency) **negated**.

---

## Currency Conversion

1. The company **base currency** is the `to_currency` configured for that company in
   `currencies` (e.g. company `1010` = `EUR → USD` ⇒ base **USD**); default `PKR`.
2. Exchange rates are pre-fetched **scoped to this company and its base currency**:
   `SELECT from_currency, exchange_rate FROM currencies WHERE to_currency = :base AND
   company_code = :company`.
3. The summary shows **no per-amount currency label** (column is "Current", total row is
   "Total") — so there is nothing to relabel (unlike the main report's "Supplier Total").

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
| Supplier | `supp_no - supp_name` |
| Average Days | Mean days overdue across the supplier's costs |
| Current | Amount within the credit period |
| 1-30 / 31-60 / 61-90 / 91-120 / 121+ Days | Amount in each overdue bucket |
| Total Outstanding | Supplier's total outstanding payable |

A bold **Total** row sums every amount column across all suppliers.

---

## Output Formats

### PDF
- **Template**: `psback/views/pages/reports/ap_ageing_analysis_summary.ejs`
- **Layout**: A4 Landscape; one row per supplier + a Total row.
- Amounts are formatted to **2 decimals** (supplier rows and Total row match).

### Excel
- **Library**: ExcelJS
- **Columns**: Supplier, Avg Days, Current, 1-30, 31-60, 61-90, 91-120, 121+, Total
  Outstanding.
- Data rows reuse the same 2-decimal formatted amounts; a bold **GRAND TOTAL** row at the
  end.

---

## Known Issues & Fixes

### Fix: Base Currency + Voided Settlements + 2-decimal rounding (v1.1, 2026-06-17)

**Problem**: For a non-PKR-base company (e.g. company `1010` with `EUR → USD`), a EUR cost
of 451.04 showed as **146,617** (another company's EUR→PKR rate), and the supplier row
showed 3-decimal values (`218.696`) while the Total row showed 2 (`218.70`).

**Root causes**:
1. Exchange-rate lookup hard-coded to `to_currency = 'PKR'` with **no company filter**.
2. "Amount settled" summed **voided** settlements, over-counting and skipping rows.
3. Per-supplier amounts were formatted with `minimumFractionDigits: 2` only (no maximum),
   so they kept up to 3 decimals.

**Fix** (`getAPAgeingAnalysisSummaryReport`):
1. Derive base currency = `currencies.to_currency` for the company (default `PKR`); query
   rates with `to_currency = :base AND company_code = :company`. EUR→USD uses 1.15 →
   **USD 518.70**.
2. Exclude settlements whose `payment_settlement.status === 'Void'` (the cost query now
   joins `payment_settlement`). Outstanding = 518.70 − 300 (live) = **218.70**.
3. Added `maximumFractionDigits: 2` to the per-supplier amount formatting → values round
   to 2 decimals, matching the Total row.

This aligns the Summary with the **AP Ageing Analysis Report** and **Detail** report.

**Files Changed**:
- `psback/controllers/report.controller.js` — `getAPAgeingAnalysisSummaryReport`

---

## Code References

| Component | File | Function/Location |
|-----------|------|-------------------|
| Frontend Page | `psfront/src/pages/Report/APAgeingAnalysisSummaryReport.jsx` | Full component |
| Backend Route | `psback/routes/report.route.js` | `getAPAgeingAnalysisSummaryReport` |
| Controller | `psback/controllers/report.controller.js` | `exports.getAPAgeingAnalysisSummaryReport` |
| PDF Template | `psback/views/pages/reports/ap_ageing_analysis_summary.ejs` | EJS template |
| Permission | `Ap-Ageing-Analysis-Summary-Report` | Required for access |

### Related Reports

| Report | Doc |
|--------|-----|
| AP Ageing Analysis Report | `AP_AGEING_ANALYSIS_REPORT.md` |
| AP Ageing Analysis Detail Report | `AP_AGEING_ANALYSIS_DETAIL_REPORT.md` |
| Base-currency rules (app-wide) | `INVOICE_DOCUMENT_CURRENCY.md` |
