# AP Ageing Analysis Report - Technical Documentation

**Version**: 1.1
**Date**: March 2026
**Author**: System Analysis
**Status**: Stable - Currency Conversion Fix Applied

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
exchangeRate = currencies[currencyCode → PKR] → e.g., 77.10 for SAR
totalCostPKR = totalCostLocal × exchangeRate
```

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

- Queried by `supplier_name` match and `doc_status = 'Printed'`
- Filtered by company branch IDs to avoid cross-company data
- Amount is **negated** (reduces outstanding)
- Order number fetched from linked refund record

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
- **Total in PKR**: Sum of all items for the supplier
- **Supplier Total PKR**: Same as total, with sum of days overdue

---

## Output Formats

### PDF
- **Template**: `psback/views/pages/reports/ap_ageing_analysis.ejs`
- **Layout**: A4 Landscape
- **Features**: Clickable XO links, supplier grouping, totals per supplier

### Excel
- **Library**: ExcelJS
- **Features**: Frozen header rows, auto-fit columns, supplier grouping, formatted amounts

---

## Known Issues & Fixes

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
