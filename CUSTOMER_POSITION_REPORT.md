# Customer Position Report - Technical Documentation

**Version**: 1.0
**Date**: March 2026
**Author**: System Analysis
**Status**: Stable

---

## Overview

The Customer Position Report provides a summary view of each customer's financial position, showing opening balance, sales invoices, refunds (credit notes), receipts, deposits, payments to customer, journal voucher adjustments, and net balance. One row per customer.

---

## Technical Architecture

### Frontend Component

**File**: `psfront/src/pages/Report/CustomerPositionReport.jsx`

**Filters**:
- Document Date: =, <, <=, >, >=, <>, between (on `created_at` or `adjustment_date`)
- Branch: isNotBlank, isBlank, isEqual, between
- Customer: isNotBlank, isBlank, isEqual, between
- Adjustment Date Mode: checkbox (Posted to Ledger)

**Output**: PDF or Excel

### Backend Controller

**File**: `psback/controllers/report.controller.js`
**Function**: `getCustomerPositionReport` (line 6474)
**Route**: `POST /api/report/getCustomerPositionReport`
**Permission**: `Customer-Position-Detail-Report`

### PDF Template

**File**: `psback/views/pages/reports/report2.ejs`
**Format**: A3 Landscape, 5mm margins

### Helper Service

**File**: `psback/services/customer_balance_calculator.js`
**Function**: `calculateCustomerBalance` — handles Air vs Hotel vs Other invoice calculations

---

## Report Layout

### Row Structure

One row per customer with summary totals:

| Column | Key | Description |
|--------|-----|-------------|
| Customer No. | `customer_no` | Customer number |
| Customer Name | `customer_name` | Customer name |
| Opening Balance B/F | `opening_balance` | Balance before report period |
| ADD: Sales Invoice | `sales_invoice` | Period invoice total |
| LESS: Refunds | `refunds` | Period credit note total |
| LESS: Receipts | `receipts` | Period receipts + deposits combined |
| ADD: Payments | `payments` | Payment settlements TO customer (via credit notes) |
| ADD: JV Debit | `jv_debit` | Manual JE debit entries |
| LESS: JV Credit | `jv_credit` | Manual JE credit entries |
| Net Balance | `net_balance` | Final calculated balance |

### Grand Total Row

Sum of all numeric columns across all customers.

---

## Calculation Logic

### Invoice Total (Per Customer)

Uses `calculateCustomerBalance` helper:

**Air Services**:
```
netPerPassenger = farePerPax + taxesPerPax - discountPerPax - rebatePerPax + tFeePerPax + sstPerPax
netPerPassengerPKR = netPerPassenger × exchangeRate
invoiceTotal = netPerPassengerPKR × numberOfPassengers
```

**Hotel Services**:
```
totalAmount = totalPrice + (SST if not already included)
invoiceTotal = totalAmount × exchangeRate
```

**Other Services**:
```
totalAmount = totalPrice + (SST if not included)
invoiceTotal = totalAmount × exchangeRate
```

### Opening Balance

```
openingBalance = historicalInvoices
               - historicalReceipts (GL account payments only)
               - historicalCreditNotes
               - historicalDeposits (full values)
               + historicalPayments (payments TO customer)
               + historicalJvDebit
               - historicalJvCredit
```

Historical data = all transactions BEFORE the report's startDate.

### Net Balance

```
netBalance = openingBalance
           + periodInvoiceTotal
           - periodRefunds (credit notes only)
           - periodReceipts (receipts + deposits combined)
           + periodPayments (payments TO customer)
           + jvDebitTotal
           - jvCreditTotal
```

### Key Rules

- **Service refunds** from refunds table are **completely excluded** — only credit notes count as refunds
- **Deposits** use **full deposit values**, not remaining balance
- **Receipts** only count **GL account payments** (deposit payments excluded to avoid double-counting)
- **Credit notes**: Foreign currency ones use `amount_base` (PKR converted); PKR ones (currency_id = 110) use `amount` directly
- **Journal entries**: Only `Manual JE` batch type, status `Open` or `Posted`, matched by `gl_entity_id` = customer_id

---

## Data Sources

### Primary Tables

| Table | Purpose |
|-------|---------|
| `customers` | Customer info (customer_number, customer_name) |
| `users` | Company scoping |
| `customer_deposits` | Deposit records (amount, currency, status) |
| `receipt_settlements` | Receipt/payment records from customers |
| `receipt_settlement_payments` | Payment method details (GL account vs deposit) |
| `orders` | Links customers to services |
| `services` | Service records (service_type, ticket_issue_date) |
| `invoices` | Invoice records (price, quantity, discount, rebate, fees, taxes) |
| `invoice_taxes` | Per-invoice tax amounts |
| `credit_notes` | Credit note records |
| `payment_settlements` | Payments made TO customer (via credit notes) |
| `journal_entries` | Manual JE entries (gl_entity_id = customer_id) |
| `journal_batches` | JE batch info (batch_type, status) |
| `invoice_settings` | Company setting for ticket_issue_date usage |
| `currencies` / `currency_codes` | Exchange rates (to PKR) |
| `service_types` | Service type (Air, Hotel, etc.) |
| `branches` | Branch filtering |

### Query Strategy (7-Step Approach)

1. **Step 1**: Get customers (basic data with company scope)
2. **Step 2**: Get customer deposits (with currency/exchange rate)
3. **Step 3**: Get receipt settlements (with payment details)
4. **Step 4**: Get orders with services (raw SQL, with branch filter)
5. **Step 5**: Get invoices (raw SQL) + invoice taxes
6. **Step 6**: Get credit notes + historical data (invoices, receipts, credit notes, deposits, JEs before startDate)
7. **Step 7**: Get payment settlements (payments TO customer)

---

## Adjustment Date Mode

When `adjustmentDateMode = true`:
- If `adjustment_date` is NULL (unposted): include only if `created_at` is in date range
- If `adjustment_date` exists (posted): include only if `adjustment_date` is in date range

When `adjustmentDateMode = false`:
- Filter by `created_at` only

Applies to: deposits, receipts, and their historical counterparts.

---

## Ticket Issue Date Logic

Controlled by `invoice_settings.ticket_issue_date_in_invoice` per company:
- If enabled AND service is Air AND `ticket_issue_date` exists: use `ticket_issue_date` for date filtering
- Otherwise: use `invoice_date`
- Applied at JavaScript level (not SQL) when enabled
- Affects both period invoices and historical invoices (opening balance)

---

## Company Scoping

Scoped via `customer → user → where: { company_code: req.user.company_code }`.

---

## Report Metadata

```javascript
{
  report_number: "TPCP" + timestamp,
  report_type: "customer-position-report",
  file_type: "xlsx" | "pdf"
}
```

---

## Output Formats

### Excel
- Worksheet: "Customer Position Report"
- Frozen rows: 7 (header section)
- Amount columns: right-aligned, `#,##0.00` format
- Column width: 15px fixed
- Grand total row: bold, light gray background

### PDF
- Template: `report2.ejs` (A3 Landscape)
- Font: Times New Roman, 8px
- Number format: `formatNumberWithCommas` (2 decimal places)
- Header repeats on each page

---

## Code References

| File | Purpose |
|------|---------|
| `psfront/src/pages/Report/CustomerPositionReport.jsx` | Frontend UI |
| `psback/controllers/report.controller.js` (line 6474) | Controller logic |
| `psback/services/customer_balance_calculator.js` | Invoice calculation helper |
| `psback/views/pages/reports/report2.ejs` | PDF template |
| `psback/routes/report.route.js` (line 56) | Route definition |
| `psfront/src/api/report.js` | API client |

---

**Document End**
