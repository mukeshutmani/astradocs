# Cash and Bank Balance Report - Technical Documentation

**Version**: 1.0
**Date**: March 2026
**Author**: System Analysis
**Status**: Stable

---

## Overview

The Cash and Bank Balance Report shows the balance of cash/bank accounts by comparing AR Deposits (inflows) against Payments (outflows) for each chart of account. Each account group displays its deposits and payments with document numbers, transaction dates, payee/payer details, currency amounts, and base currency amounts, followed by per-account totals and a net account balance (Deposits - Payments). The report has no history section — it generates directly as PDF or Excel on demand.

---

## Technical Architecture

### Frontend Component

**File**: `psfront/src/pages/Report/CashAccountBalanceReport.jsx`

**Filters**:
- Document Date: blank, =, <, <=, >, >=, <>, between (on `created_at`)
- Bank Account: isNotBlank, isBlank, isEqual, in, between (on `chart_of_account.id` directly). Dropdown shows bank accounts + cash accounts (excludes Rollup type), fetched via `GET /api/report/getCashBankAccountsForFilter`
- Document: checkboxes for Deposit, Payment, Overpayment (all selected by default)
- Document No.: text search with real-time autocomplete suggestions (deposits + payments), fetched via `GET /api/report/searchCashBalanceDocuments`. Filters by `receipt_number` (LIKE) for deposits and `payment_number` (LIKE) for payments

**Output**: PDF or Excel

### Backend Controller

**File**: `psback/controllers/report.controller.js`
**Function**: `getCashAccountBalanceReport` (line 14191)
**Route**: `POST /api/report/getCashAccountBalanceReport`
**Permission**: `Cash-Account-Balance-Report`

### PDF Template

**File**: `psback/views/pages/reports/cash-account-balance.ejs`
**Config**: A4 landscape orientation

### API Client

**File**: `psfront/src/api/report.js`
**Function**: `getCashAccountBalanceReport` (line 236)

---

## Report Layout

### Grouping Structure

Data is grouped by chart of account. Each account group contains:

1. **Account Header Row**: Account number and description (e.g., "1001-Cash in Hand")
2. **AR Deposits Section**: All customer deposits received into the account
3. **AR Deposits Total Row**: Sum of deposit amounts
4. **Payments Section**: All payments made from the account (Supplier, Expense, or Credit Note)
5. **Payments Total Row**: Sum of payment amounts
6. **Account Balance Row**: Net balance = Total AR Deposits - Total Payments (highlighted in yellow in Excel)
7. **Summary Section** (when date filter is applied):
   - **Previous Balance**: Total deposits minus payments before the filter start date (shown with date = 1 day before start)
   - **Period Activity**: Deposits minus payments within the filtered period
   - **Fund Available**: Previous Balance + Period Activity (highlighted in green)

### Columns

| Column | Source | Description |
|--------|--------|-------------|
| Document No. | `customer_deposit.receipt_number` / `payment_settlement.payment_number` | Document identifier |
| Status | `customer_deposit.status` / `payment_settlement.status` | Document status (excludes Void) |
| Transaction Date | `created_at` | Transaction date (DD MMM YYYY) |
| Payee/Payer No. | `customer.customer_number` / `supplier.supp_no` / `credit_note.customer.customer_number` | Payee or payer identifier |
| Payee/Payer Name | `customer.customer_name` / `supplier.supp_name` + payment type label | Name with type annotation (Supplier/Expense/Credit Note). Credit Note payments show the customer name. |
| Remarks | `remarks` | Remarks from the deposit/payment document |
| Staff ID | `user.username` | Staff who created the transaction |
| Currency | `currency_code.currency.currency_code` | Transaction currency (default: PKR) |
| Received Amount | `amount * exchange_rate` (deposits only) | Deposit amount converted to PKR using company-specific exchange rate; empty for Payment rows |
| Paid Amount (Base Currency) | `amount * exchange_rate` (payments only) | Payment amount converted to PKR using company-specific exchange rate; empty for AR Deposit rows |

---

## Calculation Logic

### Per-Account AR Deposit Totals

```
total_ar_deposits.amount = sum(deposit.amount) for all deposits in the account
total_ar_deposits.base_amount = sum(deposit.base_amount) for all deposits in the account
```

### Per-Account Payment Totals

```
total_payments.amount = sum(payment.amount) for all payments in the account
total_payments.base_amount = sum(payment.base_amount) for all payments in the account
```

### Account Balance

```
account_balance.amount = total_ar_deposits.amount - total_payments.amount
account_balance.base_amount = total_ar_deposits.base_amount - total_payments.base_amount
```

### Base Amount Conversion

```
base_amount = amount * exchange_rate (from currency_code -> currency where to_currency = "PKR" AND company_code = user.company_code)
```

If no exchange rate is found, defaults to rate of 1 (i.e., base_amount = amount). Exchange rates are company-specific to ensure correct conversion.

---

## Data Sources

### Primary Tables

| Table | Purpose |
|-------|---------|
| `chart_of_account` | Cash/bank accounts being reported |
| `customer_deposit` | AR deposits (inflows) to accounts |
| `payment_settlement_payment` | Payment allocations to accounts |
| `payment_settlement` | Parent payment records |
| `customer` | Customer details for deposits |
| `supplier` | Supplier details for payments |
| `currency_code` / `currency` | Currency conversion to PKR |
| `pay_type_form` | Payment type information |
| `credit_note` | Credit note details for CN payments |
| `user` | Staff information |
| `bank_account` | Bank account details (used for filter resolution) |
| `bank` | Bank names (used for filter display) |

### Key Joins

```
chart_of_account → customer_deposit (account deposits, required: true)
chart_of_account → payment_settlement_payment (account payments, required: false)
payment_settlement_payment → payment_settlement (required: true)
payment_settlement → supplier (required: false)
payment_settlement → chart_of_account as coa_id (expense account, required: false)
payment_settlement → credit_note (required: false)
customer_deposit → currency_code → currency (where to_currency = "PKR")
payment_settlement_payment → currency_code → currency (where to_currency = "PKR")
chart_of_account → user (company scoping via company_code)
```

### Query Constraints

- `chart_of_account → user` join is `required: true` with `company_code` filter for company scoping
- `customer_deposit` join is `required: true` — accounts without deposits are excluded
- `payment_settlement_payment` join is `required: false` — accounts with only deposits are included
- Void records are excluded: `status != 'Void'` on both deposits and payments
- Post-query filter removes accounts with zero deposits AND zero payments

---

## Company Scoping

Scoped via `chart_of_account → user → where: { company_code: req.user.company_code }`.

---

## Filter Logic

### Date Filter
Applied on `created_at` for both customer deposits and payment settlements:
- `blank`: No date filter (shows all records)
- `=` operator: matches entire day (startOf to endOf day using moment.js)
- `between`: uses startDate and endDate
- Other operators (`<`, `<=`, `>`, `>=`, `<>`): direct comparison on startDate (day range)

### Bank/Cash Account Filter
Applied directly on `chart_of_account.id`. The dropdown fetches cash (185xxx) and bank (181xxx) chart of accounts via `GET /api/report/getCashBankAccountsForFilter`:
- `isNotBlank`: All accounts (default)
- `isBlank`: No account filter
- `isEqual`: Specific account by `chart_of_account.id`
- `in`: Multi-select — user picks multiple accounts → sends `bank_account_ids` array → filters by `chart_of_account.id IN (...)`. Selected accounts are displayed as removable badge/tags below the dropdown.
- `between`: Range of `chart_of_account.id` values

The frontend dropdown displays accounts as `"key_account - description"` (e.g., "185001 - Cash Box", "181080 - MCB Bank").

### Document Filter
Checkbox-based filter to include/exclude document types (all selected by default):
- **Deposit**: `customer_deposit` records (AR Deposits section). When unchecked, deposits are excluded from the report.
- **Payment**: `payment_settlement_payment` records where `overpay_amount = 0` (regular payments). When unchecked, regular payments are excluded.
- **Overpayment**: `payment_settlement_payment` records where `overpay_amount > 0`. When unchecked, overpayments are excluded. Overpayment records use `overpay_amount` instead of `amount` for the transaction value and are labeled with `[Overpayment]` in the payee name.

The query dynamically adjusts includes based on selected types — `customer_deposit` and `payment_settlement_payment` are only included when their corresponding types are selected. Accounts with no matching data after filtering are excluded from the output.

When not all 3 types are selected, a "Document" filter line appears in the report header showing the active types.

---

## Payment Type Classification

Payments are classified by type based on linked entities:

| Type | Condition | Payee No. Source | Payee Name Format |
|------|-----------|-----------------|-------------------|
| Supplier | `settlement.supplier` exists | `supplier.supplier_number` | `supplier_name (Supplier)` |
| Expense | `settlement.chart_of_account` exists | `chart_of_account.key_account` | `description (Expense)` |
| Credit Note | `settlement.credit_note` exists | `credit_note.doc_no` or `reference` | `Credit Note - doc_no (Credit Note)` |

---

## Report Metadata

```javascript
{
  report_number: "CABRPT" + timestamp,
  report_type: "cash-account-balance-report",
  file_type: "xlsx" | "pdf"
}
```

---

## Excel Output

- Header section frozen (first 7 rows)
- Company name (size 24, bold, centered), address, report title, report ID, printed by, print date/time
- Date range shown if date filter is applied
- Column headers with gray background (`#D9D9D9`)
- Account header rows with light gray background (`#E0E0E0`, size 14 bold)
- Section labels ("AR Deposits", "Payments") with lighter gray background (`#F0F0F0`)
- Amount columns right-aligned with `#,##0.00` format
- Total rows in bold
- Account Balance row highlighted in yellow (`#FFEB3B`)
- Auto-fit column widths (min 10, max 50)

---

## PDF Output

- A4 landscape orientation
- Times New Roman font
- Report header: Report ID, Print Date/Time, Printed By
- Company name centered (18px bold), report title centered (14px bold)
- Applied filters displayed below title
- Table with alternating row colors (even rows `#f9f9f9`)
- Hover highlight `#e9f5e9`
- Total rows with bold font and `#f0f0f0` background
- All amounts formatted with comma separators and 2 decimal places

---

## Code References

| File | Purpose |
|------|---------|
| `psfront/src/pages/Report/CashAccountBalanceReport.jsx` | Frontend UI |
| `psback/controllers/report.controller.js` (line 14191) | Controller logic |
| `psback/views/pages/reports/cash-account-balance.ejs` | PDF template |
| `psback/routes/report.route.js` (line 64) | Route definition |
| `psfront/src/api/report.js` (line 236) | API client |
| `psfront/src/pages/Report/Report.jsx` (line 72) | Report menu navigation |

---

**Document End**
