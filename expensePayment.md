# Expense Payment - Technical Documentation

**Version**: 1.0
**Date**: April 2026
**Author**: System Analysis
**Status**: Stable

---

## Overview

The Expense Payment module allows users to record non-order-based expense payments made to a supplier. Unlike Supplier Payment Settlement (which settles outstanding XO cost documents), Expense Payment posts an amount directly against one or more expense GL accounts (chart-of-account rows with `account_type = "E/Expenses"`). It is used for office / operational / overhead payments (rent, utilities, courier, fuel, etc.) paid to a supplier.

On submission the system generates a payment document (KHPY) with multiple debit/credit expense rows plus one or more forms of payment (cash / bank / cheque, etc.).

---

## Technical Architecture

### Frontend Component

**File**: `psfront/src/pages/PaymentsPage/PaymentExpense.jsx`
**Route**: `/payment-expenses` (Sidebar: Payments → Expense Payment)
**Route Registration**: `psfront/src/App.jsx` line ~1522

### Backend Controller

**File**: `psback/controllers/payment.controller.js`
**Function**: `settleExpense` (line ~1570)
**Route**: `POST /api/payment/settle-expense`
**Route file**: `psback/routes/payment.route.js` line 33

### API Client

**File**: `psfront/src/api/payment.js`
**Function**: `settleExpense(data)` → `POST /payment/settle-expense`

### Payment Document Template

**File**: `psback/views/pages/paymentSettlement.ejs` (expense branch starts at line ~1044, rendered when `settlement.coa_id` is truthy)

### Journal Entry Posting Rules

**File**: `psback/services/journal.js`
**Functions**:
- `expensePaymentPostingRules` (line ~1962) — creates journal entries
- `voidExpensePaymentPostingRules` (line ~2799) — reverses journal entries on void

---

## Page Layout & Sections

### 1. Supplier Selection

- Combobox dropdown populated from `fetchSupplier()` API
- Format: `{supp_no} / {supp_name}`
- Required for submission

### 2. Branch Selection

- Combobox dropdown populated from `getBranches()` API
- Value used is `document_prefix` (e.g. `"01"`)
- Selecting a branch auto-fills `branch_id` into every expense row and disables the per-row Branch dropdown

### 3. Expense Accounts Table

Multiple expense lines can be added via `+ Add Account` button.

**Columns**: GL Account, Branch, Remarks, Debit, Credit, (delete)

**Key behaviors**:
- **GL Account**: Combobox filtered to `account_type === "E/Expenses"` and `type === "Detail"` (only detail-level expense accounts, not parent groups)
- **Branch**: Auto-set to top-level branch; disabled when top Branch is chosen
- **Debit** / **Credit**: Numeric inputs (default 0)
- **Net Expense Amount** (auto-calculated):
  ```
  expenseAmount = Σ(row.debit) − Σ(row.credit)
  ```
- Rows can be removed (except the last remaining row)
- Each row **must** have a `coa_id` before submit

### 4. Form of Payment

#### Date Controls

- **Select Date** (`created_at`): Payment date. Default today. Limited to last `no_of_days` days back (from `getSystemDate()` API). Future dates rejected.
- **Adjustment Date**: Optional. Used for journal entry posting date adjustments.

#### Payment Methods

- Up to **5** payment methods can be added (`+` button; 6th add shows error)
- Each payment row is rendered by the shared `FormOfPayment` component
- **Fields per payment**: `pay_type_id`, `currency_id`, `amount`, `check_number`, `chart_of_account_id` (bank COA), `cash_box`, `client_refrence`, `bank_id`, `card_type_id`, `card_number`, `voucher_number`, `gl_account_box`, `account_number`, `gl_account_id`, `remarks`
- First payment row is synced to `expenseAmount` (auto-suggest). Subsequent rows auto-suggest the remaining balance.

### 5. Remarks

Free-text input stored in `payment_settlement.remarks`.

### 6. Submit Button

Label is dynamic:
- `"Select Expense to Proceed"` → when no `coa_id` is selected in any row
- `"Proceed with Payment"` → once at least one expense row has a `coa_id`

Disabled until both a supplier is selected AND at least one expense row has a `coa_id`.

---

## Payment Submission

### Frontend Validations (order of checks in `submitPayment`)

1. No payment validation errors from `FormOfPayment` components
2. Supplier must be selected
3. Every expense row must have `coa_id`
4. Total amount (Σdebit − Σcredit) must be > 0
5. For each payment method: `pay_type_id`, `currency_id`, and `amount > 0` are required

### Payload (`POST /api/payment/settle-expense`)

```javascript
{
  supplier_id: 123,
  coa_id: <first expense row's coa_id>,      // stored on parent settlement
  document_prefix: "01",                      // from Branch selection
  amount: 5000.00,                            // Σdebit − Σcredit
  remarks: "Office rent April 2026",
  created_at: "2026-04-20",
  updated_at: "2026-04-20",
  adjustment_date: null,
  expense_rows: [
    {
      coa_id: 402,
      branch_id: "01",
      remarks: "Rent",
      debit: 5000,
      credit: 0
    }
  ],
  payments: [
    {
      pay_type_id: 1,
      currency_id: 1,
      amount: 5000,
      remarks: "",
      check_number: null,
      chart_of_account_id: null,
      cash_box: null,
      client_refrence: null,
      bank_id: null,
      card_type_id: null,
      card_number: null,
      voucher_number: null,
      gl_account_box: null,
      account_number: null,
      gl_account_id: null
    }
  ]
}
```

### Backend Processing (`settleExpense`)

1. **Branch lookup**: Load branch by `document_prefix`; reject if not found.
2. **Permission check**: `checkDocIssuePermission` for `document_type: "payment"` using the user's `user_group_id`, `company_code`, and branch's `document_prefix`. Returns 403 if not allowed.
3. **Generate payment number**:
   - Format: `{branchCode}PY{8-digit zero-padded sequential}` (e.g. `01PY00000001`)
   - Finds the first unused number in the sequence, scoped to **users of the same `company_code`** (company-wise numbering)
4. **Create `payment_settlement` record** with `coa_id`, `supplier_id`, `amount`, `remarks`, `status: "Printed"`, `user_id`, `created_at`, `updated_at`, `adjustment_date`.
5. **Create `documents` row** with `document_type: "payment"` and link to the settlement.
6. **Create `payment_settlement_expense` rows** — one per expense line (`coa_id`, `branch_id`, `remarks`, `debit`, `credit`).
7. **Create `payment_settlement_payment` rows** — one per payment method. For `pay_type_id` 2 (Cheque) or 11: checks duplicate `check_number` across all settlements and throws if found. Looks up `bank_account` by `chart_of_account_id` to resolve `bank_id`.
8. **Optional `payments_from`** (Internal Transfer flow): creates additional payment rows with `transfer_type: 'account_from'` — not used by the Expense Payment UI but supported by the same controller.
9. Wrapped in a single Sequelize transaction — all-or-nothing.
10. Response: `{ status: "success", message, settlement }`. Frontend navigates to `/documents/{payment_number}?type=payment&status=Printed`.

---

## Journal Entry Generation

Handled by `expensePaymentPostingRules` in `psback/services/journal.js`. Runs as part of the scheduled / on-demand journal batch process.

### Selection criteria

Expense payments are identified by:
- `payment_number LIKE '{branch.document_prefix}PY%'`
- `coa_id IS NOT NULL` (this distinguishes expense payments from supplier payments and internal transfers)
- `status NOT IN ['Raised', 'Void', 'Canceled', 'Cancelled']`
- `je_generated IS NULL` (not yet posted)

### Posting Rules (from `posting_rules` table)

Looked up by `branch_id`, `company_code`, and `prefix_code = '{document_prefix}PY'`. The rule `journal_entry_type.code` determines the entry type:

| Code | Side   | Purpose                       |
|------|--------|-------------------------------|
| EXPE | Debit  | Expense account debit         |
| CASH | Credit | Cash / bank account credit    |

### Debit Entries (EXPE rule)

**When `payment_settlement_expense` rows exist** (current behavior):
- One journal entry per expense row
- `netAmount = row.debit − row.credit`
- `netAmount > 0` → Debit entry with `chart_of_account_id = row.coa_id`, amount = `netAmount`
- `netAmount < 0` → Credit entry (reversed), amount = `|netAmount|`
- `netAmount == 0` → no entry for that row

**Fallback (backward compatibility)** for old records without expense rows:
- Single debit entry using `payment_settlement.coa_id`
- Amount = sum of payment methods (converted to base currency via `getExchangeRateForCurrencyId`), or `payment.amount` if no methods

### Credit Entries (CASH rule)

- One credit entry per payment method
- `chart_of_account_id` = payment method's `chart_of_account_id` (the bank/cash COA), falling back to `cashRule.chart_of_account_id`
- Amount = `payment.amount × exchange_rate` (only multiplied when `rate > 1`)
- Description: `"Payment via {payment_method}"`

### Analysis Codes on entries

- `analysis_code1` = `payment_number`
- `analysis_code2` = `supplier_id`
- `analysis_code3` = `payment_settlement.id`
- `analysis_code4` = `branch_id`
- `analysis_code5` = `check_number` or `card_number` (credit side only)

### After generation

`payment_settlement.je_generated = true` is set for all processed records, so they are not posted again.

### Void Handling

`voidExpensePaymentPostingRules` generates reversal entries when a settlement is voided via `POST /api/payment/settlement/:paymentNumber/void`.

---

## Payment Document (EJS Template)

**File**: `psback/views/pages/paymentSettlement.ejs`

The template has **three rendering branches** based on settlement type:
1. `settlement.supplier && !coa_id && !credit_note_id` → Supplier Payment layout (cost documents table)
2. `settlement.coa_id` → **Expense Payment layout**
3. `!coa_id && !credit_note_id && !supplier` → Internal Transfer layout

### Expense Payment Layout (line ~1044)

**Header**: Company name/address, pay-to details, voucher number, date, branch, prepared by.

**Payment Method Details**: Per-FOP block with method type, bank, cheque details.

**Remarks block** (if any).

**Key Account Payment Items Table**:
- Columns: Key Account, Account Name, Remarks, Debit, Credit
- One row per `payment_settlement_expense` (with `chart_of_account.key_account` and `description`)
- **Fallback**: if no expense rows exist, shows a single row using `settlement.chart_of_account` (legacy records)
- **Grand Total** row showing total payment amount (PKR)
- **Amount in Words** row

---

## Currency Handling

- Expense amounts (debit/credit) are stored as-is (assumed PKR — no multi-currency on expense rows)
- Payment methods CAN be in foreign currency; `getExchangeRateForCurrencyId` converts to base currency (PKR) during journal posting when `rate > 1`
- Template displays amounts with `PKR` prefix and `en-PK` locale formatting

---

## Database Tables

| Table | Purpose |
|-------|---------|
| `payment_settlements` | Parent settlement (fields used: `payment_number`, `user_id`, `coa_id`, `supplier_id`, `amount`, `status`, `remarks`, `created_at`, `updated_at`, `adjustment_date`, `je_generated`) |
| `payment_settlement_expenses` | Expense line rows (`coa_id`, `branch_id`, `remarks`, `debit`, `credit`) |
| `payment_settlement_payments` | Form of payment entries (pay_type, currency, amount, bank, cheque, card, chart_of_account_id, etc.) |
| `documents` | Document numbering registry (`document_number`, `document_type='payment'`, `payment_settlement_id`, `user_id`) |
| `chart_of_accounts` | GL accounts — filtered to `account_type='E/Expenses'` + `type='Detail'` for dropdown |
| `branches` | Branch list (used for `document_prefix`) |
| `bank_accounts` | Lookup of `bank_id` from `chart_of_account_id` during payment save |
| `posting_rules` | Defines EXPE / CASH rules per `prefix_code='{pref}PY'` |
| `journal_entries` | Generated entries (debit/credit) |

### `payment_settlement_expense` Model

```javascript
{
  id: INT PK,
  payment_settlement_id: INT (FK → payment_settlements.id, NOT NULL),
  coa_id: INT (FK → chart_of_accounts.id, NOT NULL),
  branch_id: STRING (nullable),
  remarks: STRING (nullable),
  debit: DECIMAL(10,2) default 0,
  credit: DECIMAL(10,2) default 0,
  created_at, updated_at
}
```

Associations:
- `belongsTo payment_settlement` (FK `payment_settlement_id`)
- `belongsTo chart_of_account` (FK `coa_id`)

---

## Differences vs Supplier Payment Settlement

| Aspect                     | Supplier Payment                                   | Expense Payment                              |
|----------------------------|----------------------------------------------------|----------------------------------------------|
| Target                     | Settles outstanding XO cost documents              | Posts to expense GL accounts directly        |
| Cost selection             | Required (grouped by document_number)              | Not applicable                               |
| Deposits / Overpayments / DN | Supported                                        | Not supported                                |
| Parent `coa_id`            | NULL                                               | Populated (first expense row's coa_id)       |
| Child table                | `payment_settlement_costs`                         | `payment_settlement_expenses`                |
| Status transitions         | Printed → Paid / Partially Paid                    | Always `Printed`                             |
| Posting rule codes         | Supplier-payment specific                          | `EXPE` (debit), `CASH` (credit)              |
| Frontend page              | `Payments.jsx`                                     | `PaymentExpense.jsx`                         |
| API endpoint               | `POST /api/payment/settle`                         | `POST /api/payment/settle-expense`           |
| Controller function        | `settlePayment`                                    | `settleExpense`                              |
| EJS branch                 | `settlement.supplier && !coa_id`                   | `settlement.coa_id`                          |

Shared infrastructure:
- Same `payment_settlements` parent table
- Same `payment_settlement_payments` child for Form of Payment
- Same document prefix scheme (`{branch}PY{padded}`)
- Same `paymentSettlement.ejs` template (different rendering branches)
- Same void endpoint (`POST /api/payment/settlement/:paymentNumber/void`)

---

## Security & Permissions

1. **Authentication**: `authenticate` middleware on route (JWT required)
2. **Route permission**: `POST /settle-expense` does **not** use the `permission("create-payment")` middleware that is applied to the generic `POST /` payment route — access control relies on `checkDocIssuePermission` inside the controller
3. **Document issue permission**: `checkDocIssuePermission` verifies the user's `user_group` can issue `document_type='payment'` for the selected `document_prefix`
4. **Company-wise numbering**: Payment number sequence is isolated per `company_code` (users from different companies have independent `PY` counters)
5. **Duplicate cheque protection**: For `pay_type_id` 2 (Cheque) and 11, the controller throws if the same `check_number` already exists with that `pay_type_id`

---

## Code References

| File | Purpose |
|------|---------|
| `psfront/src/pages/PaymentsPage/PaymentExpense.jsx` | Frontend page |
| `psfront/src/api/payment.js` (`settleExpense`) | API client function |
| `psfront/src/App.jsx` (line ~1522) | Route registration |
| `psfront/src/components/Sidebar.jsx` | Sidebar "Expense Payment" link |
| `psback/routes/payment.route.js` (line 33) | Route definition |
| `psback/controllers/payment.controller.js` (`settleExpense`, line ~1570) | Submission handler |
| `psback/models/Payment/payment_settlement_expense.js` | Expense row model |
| `psback/models/Payment/payment_settlement.js` | Parent settlement model |
| `psback/services/journal.js` (`expensePaymentPostingRules`, line ~1962) | JE generation |
| `psback/services/journal.js` (`voidExpensePaymentPostingRules`, line ~2799) | JE reversal on void |
| `psback/views/pages/paymentSettlement.ejs` (line ~1044) | Document template (expense branch) |
| `psback/services/doc_issue_permission.service.js` | Document-issue permission check |

---

**Document End**
