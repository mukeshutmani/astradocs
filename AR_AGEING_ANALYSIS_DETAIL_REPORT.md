# AR Ageing Analysis Detail Report - Technical Documentation

**Version**: 1.5
**Date**: March 2026
**Author**: System Analysis
**Status**: Stable - All Critical Issues Resolved

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Report Purpose](#report-purpose)
3. [Technical Architecture](#technical-architecture)
4. [Data Flow](#data-flow)
5. [Calculation Logic](#calculation-logic)
6. [User Interface](#user-interface)
7. [Output Formats](#output-formats)
8. [Data Integrity Features](#data-integrity-features)
9. [Remaining Moderate Issues](#remaining-moderate-issues)
10. [Recommendations](#recommendations)
11. [Code References](#code-references)

---

## Executive Summary

The AR Ageing Analysis Detail Report is a critical financial report that provides detailed visibility into outstanding accounts receivable. It categorizes unpaid invoices into aging buckets (Current, 1-30, 31-60, 61-90, 91-120, 120+ days) based on days past due date. The report also accounts for customer deposits and credit notes as credit balances.

The report supports **point-in-time accuracy** - all queries are filtered by the as-of date, allowing historical aging snapshots. It handles **multi-currency** with dual tracking (original + PKR converted amounts) and includes deduplication logic for invoices, deposits, and settlements.

---

## Report Purpose

### Business Objectives

1. **Cash Flow Management**: Monitor expected incoming payments and identify collection priorities
2. **Credit Risk Assessment**: Identify customers with significant overdue balances
3. **Collections Strategy**: Prioritize collection efforts based on aging severity
4. **Credit Limit Monitoring**: Track customer exposure against approved credit limits
5. **Financial Reporting**: Support month-end closing and financial statement preparation
6. **Bad Debt Provisioning**: Identify potentially uncollectible receivables

### Key Metrics Provided

- Outstanding invoice amounts per customer
- Aging breakdown by time buckets
- Days overdue for each invoice
- Customer credit terms and limits
- Net outstanding after credits (deposits + credit notes)
- Sales representative assignment
- Original currency amounts and exchange rates alongside PKR values

---

## Technical Architecture

### Frontend Components

**File**: `psfront/src/pages/Report/ARAgeingAnalysisDetailReport.jsx`

```
React Component Structure:
├── Filter Form
│   ├── Customer Number Filter (isNotBlank, isBlank, isEqual, between)
│   │   └── LiveComboBox (server-side search via /customer/getCustomers)
│   ├── As Of Date (default: today, single DateInput)
│   ├── Invoice Date Filter (isNotBlank, isBlank, =, <, <=, >, >=, <>, between)
│   │   └── DateInput (single or range based on operator)
│   └── Branch Filter (isNotBlank, isBlank, isEqual, between)
│       └── Combobox (client-side, pre-loaded up to 1000 branches)
├── Generate Button (dropdown: PDF / Excel)
└── Report History Table (pagination)
```

### Backend Controller

**File**: `psback/controllers/report.controller.js`
**Function**: `getARAgeingAnalysisDetailReport` (lines 14318-15359)

### Supporting Services

- **Currency Converter**: `psback/services/currencyConverter.js` - `convertToPKR()`
- **PDF Generator**: wkhtmltopdf via `createPdf(html, true)` (landscape mode)
- **File Storage**: AWS S3/MinIO via `uploadFile()` and `getSignedUrl()`

### Database Models Involved

- `customer` - Customer master data (includes `date_type`, `credit_days_1`, `credit_limit`)
- `invoice` - Invoice records (status: `Printed` or `Partially Settled` only)
- `receipt_settlement_invoice` - Payment settlements (with void status checking)
- `customer_deposit` - Advance deposits
- `credit_note` - Credit adjustments (uses `doc_date` and `doc_status`)
- `currency_code` - Currency reference (lookup by ID)
- `currency` - Exchange rates
- `branch` - Branch information (includes `document_prefix`)
- `sales_id` - Sales representative
- `user` - Company scoping via `company_code`
- `service` / `order` - Used to link invoices to customers
- `receipt_settlement` - Parent settlement record (for void status checking)
- `receipt_settlement_deposit` - Deposit settlement allocations
- `receipt_settlement_credit_note` - Credit note settlement allocations
- `report` - Report metadata storage

---

## Data Flow

### Request Flow

```
1. User Submits Filter Form (Frontend)
   ↓
2. API Call: POST /api/report/ar-ageing-analysis-detail
   ↓
3. Controller: getARAgeingAnalysisDetailReport()
   ↓
4. Database Queries (within single Sequelize transaction):
   a. Fetch Customers (filtered by customer ID, branch)
   b. For each customer:
      - Fetch Outstanding Invoices (grouped & deduplicated by invoice_number)
      - Fetch Settlements (voided excluded, temporally filtered)
      - Fetch Deposits (deduplicated by receipt_number)
      - Fetch Credit Notes (temporally filtered by doc_date)
   ↓
5. Process & Calculate:
   - Calculate due dates (credit-term-aware)
   - Determine aging buckets
   - Convert currencies to PKR (dual tracking)
   - Deduplicate records
   - Aggregate totals (net outstanding floored at zero)
   ↓
6. Generate Output (PDF or Excel)
   ↓
7. Upload to Storage & Return Link
```

### Data Transformation

```javascript
Input (Database) → Processing → Output (Report)

Invoice Record:
{
  invoice_number: "INV001",
  invoice_date: "2025-10-01",
  total_price: 10000,
  currency: "PKR",
  status: "Partially Settled"
}

+ Settlement Records (non-void only):
[{ amount: 2000 }, { amount: 1500 }]

= Calculated Output:
{
  invoice_number: "INV001",
  invoice_date: "01 Oct 2025",
  due_date: "31 Oct 2025",
  invoice_amount: 10000.00,
  amount_settled: 3500.00,
  outstanding: 6500.00,
  original_outstanding: 6500.00,
  original_currency: "PKR",
  exchange_rate: 1,
  days_overdue: 17,
  ageing_bucket: "1-30 days"
}
```

---

## Calculation Logic

### 1. Due Date Calculation

**Function**: `calculateDueDate()` (lines 14512-14536)

```javascript
Credit Term Types:
┌─────────────────────────┬───────────────────────────────┐
│ Credit Term Type        │ Due Date Calculation          │
├─────────────────────────┼───────────────────────────────┤
│ BSP Payment Term        │ Invoice Date + 7 days         │
│ BSP PAYMENT +15 days    │ Invoice Date + 22 days        │
│ Weekly                  │ Invoice Date + 7 days         │
│ Statement Date          │ End of Month + 30 days        │
│ Any Other / Default     │ Invoice Date + credit_days_1  │
└─────────────────────────┴───────────────────────────────┘
```

The default case parses `customer.credit_days_1` as an integer (defaults to 0 if not set).

### 2. Days Overdue Calculation

```javascript
Days Overdue = moment(asOfDate).diff(moment(dueDate), 'days')

If Days Overdue <= 0: Invoice is CURRENT (not yet due)
If Days Overdue > 0: Invoice is OVERDUE
```

### 3. Aging Bucket Assignment

```javascript
Bucket Ranges:
┌─────────────┬─────────────────┬──────────────────┐
│ Bucket      │ Days Range      │ Risk Level       │
├─────────────┼─────────────────┼──────────────────┤
│ Current     │ ≤ 0             │ Low              │
│ 1-30 Days   │ 1-30            │ Low-Medium       │
│ 31-60 Days  │ 31-60           │ Medium           │
│ 61-90 Days  │ 61-90           │ High             │
│ 91-120 Days │ 91-120          │ Very High        │
│ 120+ Days   │ > 120           │ Critical         │
└─────────────┴─────────────────┴──────────────────┘
```

Deposits and credit notes are also assigned to aging buckets based on their creation/document date (not due date).

### 4. Outstanding Amount Calculation

```javascript
For each invoice (grouped by invoice_number, deduplicated):
  Invoice Outstanding = Sum(unique invoice amounts) - Sum(non-void settlements)
  // Invoices with outstanding <= 0 are skipped

For each customer:
  Total Outstanding = Sum(Invoice Outstanding amounts in PKR)
  Deposit Credit = Sum(deposit.current_amount for non-zero deposits)
  Credit Note Credit = Sum(Credit Note amount - used_amount)
  Net Outstanding = Total Outstanding - Deposit Credit - Credit Note Credit
  // Can be negative (indicates customer has credit balance)

// Credit note outstanding uses used_amount (source of truth for all settlement types):
creditNoteOutstanding = creditNote.amount - creditNote.used_amount
```

Net outstanding **allows negative values** to indicate customer credit balances (e.g., when deposits exceed invoice totals).

### 5. Currency Conversion

All amounts are converted to PKR (Pakistani Rupee) using exchange rates from `currency_code` table.

```javascript
Converted Amount = Original Amount × Exchange Rate
// Math.ceil() is applied for rounding up via convertToPKR()

// Both original and converted values are preserved in output:
{
  outstanding: convertedAmount,       // PKR value used in aging buckets
  original_outstanding: rawAmount,    // Original currency value
  original_currency: "USD",           // Currency code
  exchange_rate: 278.50               // Rate used
}
```

Currency conversion errors are caught and logged - the unconverted amount is used as fallback.

---

## User Interface

### Filter Options

| Filter | Type | Input Component | Options | Default |
|--------|------|-----------------|---------|---------|
| Customer Number | Select | LiveComboBox (server-side search) | Is Not Blank, Is Blank, Is Equal, Between | Is Not Blank |
| As Of Date | Date | DateInput | Single date value | Today |
| Invoice Date | Select | DateInput | Is Not Blank, Is Blank, =, <, <=, >, >=, <>, Between | Is Not Blank |
| Branch | Select | Combobox (client-side, up to 1000) | Is Not Blank, Is Blank, Is Equal, Between | Is Not Blank |

### Filter Input Details

- **Customer Number**: When `isEqual` → single LiveComboBox. When `between` → two LiveComboBoxes (Start/End).
- **Invoice Date**: When single operator (=, <, etc.) → one DateInput. When `between` → two DateInputs (Start/End).
- **Branch**: When `isEqual` → single Combobox. When `between` → two Comboboxes (Start/End). Display format: `{document_prefix}/{name}`.

### Output Options

- **PDF Report**: Formatted landscape A4 document. Navigates to `/reports/${report_number}?type=report` for viewing.
- **Excel Export**: Downloads as blob with filename `AR_Ageing_Analysis_Detail_${report_number}.xlsx`. Two-step process: generate report → fetch XLSX blob → trigger download.

---

## Output Formats

### PDF Template Structure

**File**: `psback/views/pages/reports/ar_ageing_analysis_detail_fixed.ejs`

```
┌─────────────────────────────────────────────────────────┐
│ Report Header                                           │
│ - Report ID, Print Date/Time, Printed By               │
│ - Company Name & Address                                │
│ - Report Title & Filters Applied                        │
├─────────────────────────────────────────────────────────┤
│ Column Headers                                          │
│ Customer/Doc No | TC Code | Sales ID | Customer Ref |  │
│ Doc Date | Due Date | Days Overdue | Current |         │
│ 1-30 Days | 31-60 Days | 61-90 Days | 91-120 Days |    │
│ 121+ Days | Total Outstanding                           │
├─────────────────────────────────────────────────────────┤
│ Customer Section (repeats for each customer)           │
│ ├─ Customer Name & Number                              │
│ ├─ Invoice Lines (each invoice row)                    │
│ ├─ Customer Total Row                                  │
│ ├─ Deposits Section (if any, shown as negative)        │
│ ├─ Credit Notes Section (if any, shown as negative)    │
│ └─ Net Outstanding Row (if credits exist)              │
├─────────────────────────────────────────────────────────┤
│ Grand Total Section                                     │
│ - Grand Total (all customers)                           │
│ - Less: Total Deposits                                  │
│ - Less: Total Credit Notes                              │
│ - Net Grand Total                                       │
└─────────────────────────────────────────────────────────┘
```

### Excel Structure

**Columns** (13 total):
Invoice No | Invoice Date | Due Date | Days Overdue | Invoice Amount | Amount Settled | Outstanding | Current | 1-30 Days | 31-60 Days | 61-90 Days | 91-120 Days | 120+ Days

**Layout**:
- **Rows 1-6**: Header section (company name size 24, address, report title, metadata). All merged across columns A-K.
- **Row 7**: Frozen header row (column headers)
- **Frozen rows**: First 7 rows are frozen for scrolling

**Color-coded sections**:
- Gray (`FFD9D9D9`): Column headers (bold, bordered)
- Light gray (`FFF0F0F0`): Customer headers (`{customer_number} - {customer_name} (Credit Terms: {credit_terms})`)
- Orange (`FFFFE0B2`): Customer subtotals (bold)
- Green font: Deposit credits (negative amounts), with "Less: Deposit Credit" label
- Dark gray (`FFD3D3D3`): Grand totals (bold size 12, double-line borders)

**Formatting**: Numeric cells use `#,##0.00` format, right-aligned. Auto-fit columns (width 10-50).

---

## Data Integrity Features

### Invoice Deduplication
Invoices are **grouped by `invoice_number`** using a `Map`. Each group tracks unique `invoice.id` values via a `Set` to avoid double-counting when Sequelize JOINs produce duplicate rows. Settlements are also deduplicated by `settlement.id`.

### Deposit Deduplication
Deposits are deduplicated using a `processedDepositIds` Set keyed by `receipt_number || id`. This handles cases where the JOIN with `receipt_settlement_deposit` produces duplicate deposit rows. Applied in two passes: once for total credit calculation and once for line item generation.

### Voided Settlement Exclusion
Receipt settlements with `receipt_settlement.status === 'Void'` are explicitly excluded from the settled amount calculation for both invoices and credit notes, ensuring voided payments don't affect the outstanding balance.

### Credit Note Settlement Aggregation
Credit note outstanding is calculated using the `used_amount` field directly: `amount - used_amount`. The `used_amount` field is the single source of truth, updated by both receipt settlements (via `receipt_settlement_credit_notes`) and payment refunds (via payment controller). This ensures the report captures all settlement mechanisms accurately.

### Point-in-Time Accuracy
All queries are temporally filtered by the as-of date:
- **Invoices**: `invoice_date <= asOfDate`
- **Invoice settlements**: `created_at <= asOfDate`
- **Deposits**: `created_at <= asOfDate`
- **Deposit settlements**: `created_at <= asOfDate`
- **Credit notes**: `doc_date <= asOfDate`
- **Credit note settlements**: `created_at <= asOfDate`

### As-Of Date Validation
Future dates are rejected with a 400 error to prevent generating speculative aging reports.

### Null Invoice Date Handling
Invoices with null `invoice_date` are placed in the **Current** bucket with a console warning, rather than causing errors.

### Customer Inclusion Criterion
A customer is only included in the output if they have at least one outstanding invoice, deposit, or credit note. Customers with no activity are excluded entirely.

### Invoice Status Filtering
Only invoices with status `'Printed'` or `'Partially Settled'` are included. Fully settled and draft invoices are excluded.

---

## Remaining Moderate Issues

### Currency Rounding Discrepancy

`currencyConverter.js` uses `Math.ceil()` for conversion, but the controller uses `.toFixed(2)` for display. This can cause small discrepancies between individual invoice amounts and displayed totals.

### Template Aging Bucket Label Mismatch

| Backend (Excel) | Template (PDF) |
|-----------------|----------------|
| 120+ Days       | 121+ Days      |

Minor confusion for users comparing PDF and Excel outputs.

---

## Recommendations

### Completed Fixes (Version 1.1)

1. Invoice Date Filter - Now uses `invoice_date` field
2. Branch Between Filter - Filter properly applied to query
3. Settlement Temporal Consistency - All settlements respect as-of date
4. BSP Payment Term Logic - `BSP PAYMENT +15 days` now uses 22 days (7 + 15)
5. As-Of Date Validation - Prevents future-dated reports
6. Null Invoice Date Handling - Safely handles missing dates
7. Document Date Filtering - All documents filtered by as-of date

### Completed Fixes (Version 1.3)

8. **Credit Note Settlement Aggregation** - Fixed bug where credit note settlements were accessed as a singular object (`receipt_settlement_credit_note?.amount`) instead of summing the hasMany array (`receipt_settlement_credit_notes`). This caused partially settled credit notes to show the full amount instead of the outstanding balance.
9. **Credit Note Void Settlement Exclusion** - Added `receipt_settlement` include to credit note queries to check void status, matching the existing invoice settlement void checking behavior.
10. **NET GRAND TOTAL Floor** - ~~Added `Math.max(0, ...)` to the PDF template's NET GRAND TOTAL calculation~~ (Reverted in v1.5 - negative values now allowed to show credit balances).

### Completed Fixes (Version 1.4)

11. **Credit Note Settlement Source of Truth** - Replaced `receipt_settlement_credit_notes` array summing with direct `used_amount` field usage. The previous approach only captured receipt-based settlements, missing payment refund settlements. The `used_amount` field is updated by all settlement mechanisms (receipt settlements + payment refunds), making it the authoritative source for credit note utilization.

### Completed Fixes (Version 1.5)

12. **Net Outstanding Allows Negative (Credit Balance)** - Removed `Math.max(0, ...)` from both controller (`net_outstanding` calculation) and template (`NET GRAND TOTAL`). Customers with deposits exceeding their invoice totals now correctly show negative net outstanding values, indicating a credit balance. Previously these were capped at 0, hiding the customer's credit position.

### Remaining Improvements

#### Short-Term

1. **Standardize Aging Labels** - Align "120+ Days" vs "121+ Days" between PDF and Excel
2. **Currency Rounding Consistency** - Standardize rounding approach across conversions

#### Long-Term Enhancements

3. **BSP Calendar Integration** - Integrate actual BSP settlement calendar for precise due dates
4. **Performance Optimization** - Replace N+1 queries with batch loading, add database indexes
5. **Advanced Filtering** - Add sales representative, customer category, and minimum outstanding filters

---

## Code References

### Key Files

| File | Purpose | Key Lines |
|------|---------|-----------|
| `psfront/src/pages/Report/ARAgeingAnalysisDetailReport.jsx` | Frontend UI | All |
| `psback/controllers/report.controller.js` | Main logic | 14318-15359 |
| `psback/views/pages/reports/ar_ageing_analysis_detail_fixed.ejs` | PDF template | All |
| `psback/services/currencyConverter.js` | Currency conversion | `convertToPKR()` |
| `psfront/src/api/report.js` | API client | `getARAgeingAnalysisDetailReport` |
| `psback/routes/report.route.js` | API routing | AR ageing route |

### Database Tables

```sql
-- Primary tables
customer                        -- Customer master (date_type, credit_days_1, credit_limit)
invoice                         -- Invoice headers (status: Printed/Partially Settled)
receipt_settlement_invoice      -- Payment allocations (void status checked)
customer_deposit                -- Advance deposits (current_amount used)
credit_note                     -- Credit adjustments (doc_date, doc_status)

-- Settlement tables
receipt_settlement              -- Parent settlement (status for void checking)
receipt_settlement_deposit      -- Deposit settlement allocations
receipt_settlement_credit_note  -- Credit note settlement allocations

-- Supporting tables
branch                          -- Branch information (document_prefix)
sales_id                        -- Sales representatives
currency_code                   -- Currency definitions (lookup by ID)
currency                        -- Exchange rates
user                            -- Company scoping (company_code)
service / order                 -- Invoice-to-customer linking
report                          -- Report metadata storage
```

### API Endpoint

```
POST /api/report/ar-ageing-analysis-detail
Content-Type: application/json
Authorization: Bearer <token>

Request Body:
{
  "customerFilter": "isNotBlank|isBlank|isEqual|between",
  "customer_id": number (if isEqual),
  "customer_idStart": number (if between),
  "customer_idEnd": number (if between),
  "asOfDate": "YYYY-MM-DD",
  "dateFilter": "isNotBlank|isBlank|=|<|<=|>|>=|<>|between",
  "startDate": "YYYY-MM-DD",
  "endDate": "YYYY-MM-DD",
  "branchFilter": "isNotBlank|isBlank|isEqual|between",
  "branch_id": number,
  "branch_idStart": number,
  "branch_idEnd": number,
  "type": "pdf|excel"
}

Response:
{
  "status": 200,
  "message": "success",
  "link": "S3/MinIO URL",
  "downloadLink": "Signed URL (for Excel)",
  "report": { report_number, ... }
}
```

### Report Metadata

```javascript
{
  report_number: "ARAGE" + timestamp,
  report_type: "ar-ageing-analysis-detail-report",
  file_type: "xlsx" | "pdf"
}
```

---

## Testing Checklist

### Functional Tests

- [x] Filter by single customer - correct invoices shown
- [x] Filter by customer range - all customers in range included
- [x] Filter by branch (single) - only branch invoices shown
- [x] Filter by branch (range) - FIXED in v1.1
- [x] Filter by invoice date - FIXED in v1.1 (uses `invoice_date` field)
- [x] As-of date affects aging buckets correctly
- [x] As-of date validation rejects future dates
- [x] Partially settled invoices show correct outstanding
- [x] Voided settlements excluded from calculations
- [x] Deposits reduce net outstanding
- [x] Credit notes reduce net outstanding
- [x] Currency conversion applied correctly
- [x] Totals match sum of line items

### Edge Cases

- [x] Invoice with no due date (null invoice_date) - placed in Current bucket
- [x] Invoice fully settled (outstanding = 0) - skipped
- [x] Invoice in foreign currency - dual tracking (original + PKR)
- [x] Multiple invoices same invoice_number - deduplicated via Map/Set
- [x] Deposit with partial settlement - uses `current_amount`
- [x] Future as-of date - rejected with 400 error
- [x] Customer with no outstanding activity - excluded from report

---

## Appendix: Sample Aging Calculation

**Scenario**:
- Customer: ABC Corp
- Credit Terms: 30 days (credit_days_1 = 30)
- As Of Date: 2025-11-17

| Invoice | Invoice Date | Due Date | Days Overdue | Bucket | Outstanding |
|---------|-------------|----------|--------------|--------|-------------|
| INV001  | 2025-11-01  | 2025-12-01 | -14 | Current | 5,000 |
| INV002  | 2025-10-15  | 2025-11-14 | 3 | 1-30 Days | 3,000 |
| INV003  | 2025-09-01  | 2025-10-01 | 47 | 31-60 Days | 7,500 |
| INV004  | 2025-07-15  | 2025-08-14 | 95 | 91-120 Days | 2,000 |
| INV005  | 2025-05-01  | 2025-05-31 | 170 | 120+ Days | 10,000 |

**Customer Totals**:
- Current: 5,000
- 1-30 Days: 3,000
- 31-60 Days: 7,500
- 61-90 Days: 0
- 91-120 Days: 2,000
- 120+ Days: 10,000
- **Total Outstanding: 27,500**
- Less Deposits: (1,500)
- Less Credit Notes: (500)
- **Net Outstanding: 25,500**

---

**Document End**
