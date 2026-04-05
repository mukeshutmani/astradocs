# Customer Deposit Movement Report - Technical Documentation

**Version**: 1.0
**Date**: April 2026
**Author**: System Analysis
**Status**: Stable

---

## Overview

The Customer Deposit Movement Report tracks outstanding customer deposits and their settlement activity. It shows each deposit with its original amount (converted to PKR), linked receipt settlements (debits), and the remaining balance. Deposits are grouped by receipt number with settlement lines shown as nested rows. Only deposits with status "Printed" or "Partially Settled" are included.

---

## Technical Architecture

### Frontend Component

**File**: `psfront/src/pages/Report/CustomerDepositMovement.jsx`

**Filters**:
- Document Date: blank, =, <, <=, >, >=, <>, between (on `created_at` or `adjustment_date`)
- Branch: isNotBlank, isBlank, isEqual
- Deposit Number: isNotBlank, isBlank, isEqual (with live search combobox)
- Customer: isNotBlank, isBlank, isEqual (with live search combobox)
- Posted to Ledger (adjustmentDateMode): checkbox toggle

**Output**: PDF or Excel

### Backend Controller

**File**: `psback/controllers/report.controller.js`
**Function**: `getCustomerDepositMovementReport` (line 4366)
**Route**: `POST /api/report/getCustomerDepositMovementReport`
**Permission**: `Customer-Deposit-Movement-Report`

### PDF Template

**File**: `psback/views/pages/reports/customer-deposit-movement.ejs`
**Config**: A4 landscape orientation

### API Client

**File**: `psfront/src/api/report.js`
**Function**: `getCustomerDepositMovementReport`

---

## Report Layout

### Row Structure

Each deposit generates ONE row with multi-line HTML content:
1. **Line 1**: Deposit's own data (amount, date, reference, etc.)
2. **Lines 2-N**: One per receipt settlement linked to this deposit
3. **Sub-Total line**: Balance = Deposit Amount - Total Settled

### Columns

| Column | Source | Description |
|--------|--------|-------------|
| Deposit No. | `customer_deposit.receipt_number` | Deposit document identifier |
| Date | `created_at` | Document creation date (DD-MM-YYYY) |
| Adj. Date | `adjustment_date` | Posting date (DD-MM-YYYY), only shown when "Posted to Ledger" is enabled |
| Reference No. | `receipt_number` / `receipt_settlement.receipt_number` | Deposit reference + settlement references |
| Cheque/Card No. | `check_number` / `card_number` | Payment method identifier |
| Currency | `currency_code.currency_code` | Transaction currency code |
| Rate | `exchange_rate` | Exchange rate to PKR |
| Amount | Computed | Deposit amount and settlement amounts in PKR (multi-line with sub-total) |
| Debit | `receipt_settlement` totals | Total settled amount in PKR |
| Credit | `customer_deposit.amount × exchangeRate` | Total deposit amount in PKR |
| Remarks | `remarks` | Deposit and settlement remarks |
| TCID | `user.username` | Staff who created the deposit |

---

## Calculation Logic

### Exchange Rate Resolution

```
currency = currency_code.currencies.find(
    from_currency = deposit.currency_code AND
    to_currency = "PKR" AND
    company_code = user.company_code
)
exchangeRate = currency.exchange_rate || 1
```

Exchange rates are company-specific to ensure correct conversion across multi-company setup.

### Deposit Amount (PKR)

```
depositAmount = customer_deposit.amount × exchangeRate
```

### Settlement Amount (PKR)

```
settlementAmount = receipt_settlement_deposit.amount × exchangeRate
```

### Total Settled

```
totalSettled = Σ(settlement.amount × exchangeRate) for all settlements of a deposit
```

### Balance (Sub-Total per deposit)

```
balance = depositAmount - totalSettled
```

### Grand Totals

```
totalCredit = Σ(deposit.amount × exchangeRate) for all deposits
totalDebit = Σ(Σ(settlement.amount × exchangeRate)) for all deposits
totalBalance = totalCredit - totalDebit
```

---

## Data Sources

### Primary Tables

| Table | Purpose |
|-------|---------|
| `customer_deposit` | Deposit records (status: Printed, Partially Settled) |
| `receipt_settlement_deposit` | Links deposits to receipt settlements |
| `receipt_settlement` | Receipt settlement parent records |
| `receipt_settlement_payment` | Payment links from settlements |
| `customer` | Customer details |
| `branch` | Branch information |
| `currency_code` / `currency` | Currency conversion to PKR |
| `chart_of_account` | GL account details |
| `user` / `company` | Company scoping |

### Settlement Loading

Settlements are batch-loaded separately (not via hasOne association) to capture ALL settlements per deposit:
1. Collect all deposit IDs
2. Fetch all `receipt_settlement_deposit` records for those IDs
3. Group by `customer_deposit_id`

---

## Filter Logic

### Date Filter

When **adjustmentDateMode is disabled**: filters by `created_at` only.

When **adjustmentDateMode is enabled**: dual-condition filter:
```
(adjustment_date IS NULL AND created_at in range)
OR (adjustment_date IS NOT NULL AND adjustment_date in range)
```
This ensures unposted deposits (no adjustment_date) use `created_at`, while posted deposits use `adjustment_date`.

For non-"between" operators, single dates are expanded to full day range (`startOf('day')` to `endOf('day')`).

### Branch Filter

- `isNotBlank`: branch_id IS NOT NULL
- `isBlank`: branch_id IS NULL
- `isEqual`: branch_id = selected value

### Deposit Number Filter

- `isNotBlank`: receipt_number IS NOT NULL
- `isBlank`: receipt_number IS NULL
- `isEqual`: receipt_number = selected value (live search from `/deposit/customer_deposit`)

### Customer Filter

- `isNotBlank`: customer_id IS NOT NULL
- `isBlank`: customer_id IS NULL
- `isEqual`: customer_id = selected value (live search from `/customer/getCustomers`)

---

## Company Scoping

Scoped via `customer_deposit → user → company` where `company.code = req.user.company_code` (required: true).

---

## Report Metadata

```javascript
{
  report_number: "TPCDM" + timestamp,
  report_type: "customer-deposit-movement",
  file_type: "xlsx" | "pdf"
}
```

---

## Excel Output

- Header section frozen (first 7 rows)
- Company name (24pt bold, centered), address, report title, report ID, printed by, print date/time
- Column headers with gray background (#D9D9D9), 12pt bold
- Dynamic column count based on adjustmentDateMode (9 or 10 columns)
- Amount columns right-aligned with thousand separators
- Grand total row with light gray background (#F5F5F5)
- Auto-fit column widths (min 10, max 50)

---

## PDF Output

- A4 landscape orientation
- Multi-line HTML cells for deposit + settlement rows
- Sub-total per deposit with border separator
- Grand total row at bottom
- Alternating row colors, hover effects
- Conditional `adj_date` column based on adjustmentDateMode

---

## Code References

| File | Purpose |
|------|---------|
| `psfront/src/pages/Report/CustomerDepositMovement.jsx` | Frontend UI |
| `psback/controllers/report.controller.js` (line 4366) | Controller logic |
| `psback/views/pages/reports/customer-deposit-movement.ejs` | PDF template |
| `psback/routes/report.route.js` (line 54) | Route definition |
| `psfront/src/api/report.js` | API client |

---

**Document End**
