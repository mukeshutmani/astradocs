# Cash and Bank Balance Report - Technical Documentation

**Version**: 1.1
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
4. **Internal Transfer Section**: All internal-transfer legs touching this account, shown as their own section (between AR Deposits and Payments). Transfer-in legs (`transfer_type = 'account_to'`, money received into this account) show in **Received Amount**; transfer-out legs (`transfer_type = 'account_from'`, money sent out of this account) show in **Paid Amount**. In this section the **Payee/Payer No.** column shows the section account's own **bank account number** (`bank_accounts.account_number`); cash accounts with no bank account leave it blank. The **Payee/Payer Name** column shows the transfer route **"From &lt;source bank&gt; To &lt;destination bank&gt;"** (each side's `chart_of_account.description`) — the same text on both the sending and receiving sides.
5. **Internal Transfer Total Row**: Received Amount = sum of transfer-in legs; Paid Amount = sum of transfer-out legs
6. **Payments Section**: Only normal outflows (Supplier, Expense, Credit Note). Internal transfers are no longer listed here — they live in the Internal Transfer section above.
7. **Payments Total Row**: Paid Amount = sum of outflow payments
8. **Account Balance Row**: Net balance = (Total AR Deposits + transfer-ins received) − transfer-outs − Total Payments (highlighted in yellow in Excel). Rendered after all three sections, so it shows even for accounts whose only outflows were internal transfers.
9. **Summary Section** (when date filter is applied):
   - **Previous Balance**: (deposits + transfer-ins) minus outflow payments before the filter start date (shown with date = 1 day before start)
   - **Period Activity**: (deposits + transfer-ins received) minus payments within the filtered period
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
| Received Amount | `amount * exchange_rate` (deposits + transfer-in payments) | Deposit amount converted to PKR; **also** holds internal transfer-in legs (`transfer_type = 'account_to'`) listed in the Payments section. Empty for outflow payment rows |
| Paid Amount (Base Currency) | `amount * exchange_rate` (outflow payments only) | Payment amount converted to PKR. Empty for AR Deposit rows and for transfer-in (`account_to`) rows |

---

## Calculation Logic

### Per-Account AR Deposit Totals

```
total_ar_deposits.amount = sum(deposit.amount) for all deposits in the account
total_ar_deposits.base_amount = sum(deposit.base_amount) for all deposits in the account
```

### Per-Account Payment Totals

```
total_payments.amount    = sum(payment.amount) for OUTFLOW payments (excludes transfer_type = 'account_to')
total_payments.received  = sum(payment.received_amount) for transfer-in legs (transfer_type = 'account_to')
total_payments.base_amount = sum(payment.base_amount) for all payments in the account
```

### Internal Transfers (`transfer_type`)

An internal transfer between two accounts writes two `payment_settlement_payments` rows:
- `account_from` — money OUT of the source account → shown as a normal outflow (Paid Amount).
- `account_to` — money IN to the destination account → shown in Received Amount and added (not subtracted) in the balance.

Normal supplier/expense/credit-note payments have `transfer_type = NULL` and are always outflows.

### Account Balance

```
account_balance.amount = (total_ar_deposits.amount + total_payments.received) - total_payments.amount
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

## Recent Updates

### Internal Transfer moved to its own section (2026-06-19)

**Request**: Internal-transfer rows were mixed inside the Payments section. They should sit in
their own **Internal Transfer** section, placed after AR Deposits and before Payments — same
shape as the other sections (rows + total).

**Change** (`getCashAccountBalanceReport` + `cash-account-balance.ejs`):
1. Both transfer legs are now detected: `account_to` (money in → Received Amount) and
   `account_from` (money out → Paid Amount).
2. Those rows are pulled out of `payments[]` into a new `internal_transfers[]` list with its own
   totals (`total_internal_transfers.received` / `.amount` / `.base_amount`).
3. The Payments section now holds only normal supplier/expense/credit-note outflows.
4. **Account Balance row** was moved to render after all three sections (PDF and Excel), so an
   account whose only outflows were transfers still shows its balance row.
5. In the Internal Transfer section, the **Payee/Payer No.** column shows the section account's
   own bank account number (looked up from `bank_accounts` by `chart_of_account_id`, scoped by
   company). Cash accounts with no bank account row leave it blank. The **Payee/Payer Name**
   column shows the route "From &lt;source bank&gt; To &lt;destination bank&gt;", resolved from
   both transfer legs by `payment_settlement_id` (each side's `chart_of_account.description`).
6. All balance numbers are unchanged — the calculation only moved its terms around:
   `balance = deposits + transfers_in − transfers_out − payments` (identical net to before).
   Period Activity / Previous Balance / Fund Available are unaffected.

**Scope**: Both PDF and Excel outputs. Affects every account that has internal transfers.

### Internal transfer-ins shown as Received, not Paid (2026-06-16)

**Problem**: An internal transfer received into a bank account (e.g. TPPY00000005 into 181050) was showing in **Paid Amount** and being subtracted as an outflow, understating the balance.

**Root cause**: A transfer writes two `payment_settlement_payments` rows — `account_to` (money in) and `account_from` (money out). The report treated **every** payment-settlement row as an outflow and ignored `transfer_type`.

**Fix** (`getCashAccountBalanceReport`):
1. Rows with `transfer_type = 'account_to'` keep their place in the Payments list but put the amount in **Received Amount** (Paid Amount blank), flagged via `is_transfer_in`.
2. `total_payments.amount` excludes transfer-ins; a new `total_payments.received` subtotal sums them.
3. `account_balance.amount` and **Period Activity** = (AR deposits + transfer-ins received) − payments.
4. **Previous Balance** counts pre-period transfer-ins as money in (added), not out.
5. PDF template `cash-account-balance.ejs`: payment rows now render `received_amount` in the Received column and show Paid only when present; the Total Payments row shows the received subtotal. Excel picks up the new fields and sets the Payments-total Received cell.

**Scope note**: applies to **all** `account_to` rows, not just one document — for account 181050 in May–Jun 2026 that is TPPY00000005, TPPY00000006 and TTPY00000039.

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
