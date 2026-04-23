# Payment Listing Report - Technical Documentation

**Version**: 1.0
**Date**: March 2026
**Author**: System Analysis
**Status**: Stable - Verified

---

## Overview

The Payment Listing Report shows all **non-void** payment settlements made to suppliers, including payment method details, bank/cash box information, exchange rates, and gain/loss amounts. Each row represents a payment method entry within a settlement. Settlements without payment method records are also shown with fallback values. A secondary **Advance Payments** section (from `supplier_deposits`) is shown below the settlements when matching records exist.

**Void settlements are excluded** from the report (April 2026). Internal-transfer settlements (with `transfer_type` set on `payment_settlement_payments`) collapse to a single row representing the source bank (the `account_from` leg) — the `account_to` leg is suppressed so transfers are not double-counted.

---

## Technical Architecture

### Frontend Component

**File**: `psfront/src/pages/Report/PaymentListingReport.jsx`

**Filters**:
- Date Range: =, <, <=, >, >=, <>, between (with adjustment date mode support)
- Branch: isNotBlank, isBlank, isEqual (by payment_number prefix)
- Supplier: isNotBlank, isBlank, isEqual, between (by supp_no; dropdown displays supplier name only)
- Payment Document Number: isEqual, between
- Order Number: isNotBlank, isBlank, isEqual, between
- Card/Cheque Number: isNotBlank, isBlank, isEqual, between
- Currency: isNotBlank, isBlank, isEqual, between
- Adjustment Date Mode: Checkbox (Posted to Ledger)

**Output**: PDF or Excel

### Backend Controller

**File**: `psback/controllers/report.controller.js`
**Function**: `getPaymentSettlementReport` (line ~7962)

### PDF Template

**File**: `psback/views/pages/reports/settlement.ejs`

---

## Report Layout

### Columns

| Column | Source | Description |
|--------|--------|-------------|
| Payment No. | `payment_settlement.payment_number` | Payment document number |
| Payee Name | `supplier.supp_name` | Supplier name (blank if no supplier) |
| Date/Doc No. | `payment_settlement.created_at` + XO number | Voucher date of the settlement (not the payment row's `created_at`) / costing document number |
| Adj Date | `payment_settlement.adjustment_date` | Only shown when adjustment date mode enabled |
| Status | `payment_settlement.status` | Always "Printed" (void is excluded) |
| Void Amount | — | Always blank (void is excluded) |
| Form of Payment | `pay_type_form.label` | Cheque, Cash, Card, etc. |
| Bank/Cash Box | `chart_of_account.description` + `key_account` | Account description and number. For internal transfers this is the **source** bank (`account_from` leg). |
| Cheque Date | `payment_settlement.created_at` | Voucher date of the settlement (same as Date/Doc No.) |
| Paid Amount | Amount | Amount of this payment row (PKR) |
| Overpayment | `payment_settlement.overpayment` | Overpayment amount |
| Ex. Rate | `currency_code.exchange_rate` (via currency) | Exchange rate (1.00 for PKR) |
| Gain/Loss Amt | `payment.gain_loss_amount` | Currency gain/loss |
| Base Amt | `amount × exchange_rate` | Amount in base currency (PKR) |

### Date/Doc No. Format

- If both settlement date and XO number exist: `{settlementDate}/{xoNumber}`
- Otherwise: whichever is available
- **Date** = `payment_settlement.created_at` (the voucher date, same as what prints on the settlement document). Previously used `payment_settlement_payment.created_at`, which drifted whenever a payment row was re-inserted on edit — fixed April 2026.
- XO Number sourced from: `payment_settlement_costs[0] → cost → service → document (type='costing')`

### Fallback for Settlements Without Payment Records

When a settlement has no `payment_settlement_payment` entries:
- Amount = `payment_settlement.amount`
- Date = `payment_settlement.created_at`
- Form of Payment = blank
- Bank/Cash Box = blank
- Ex. Rate = 1 (default)

---

## Calculation Logic

### Settlement-level filter (applied in the Sequelize query)

```
payment_settlement.status != 'Void'
```

### Payment-row filter (applied in JS after fetch)

```
keep row IF payment_settlement_payments.transfer_type !== 'account_to'
```
Regular supplier payments have `transfer_type = NULL` so they pass through.
Internal transfers contribute only the `account_from` leg.

### Per-Row Amounts

```
amountLocal = payment.amount × (currency_code.exchange_rate || 1)

Paid Amount = amountLocal
Base Amt   = amountLocal
Void Amount = ""   (always — void is excluded)
```

### Summary Totals

```
Total Paid Amount     = sum(all paid_amount values)
Total Paid Amount PKR = Total Paid Amount
```

(Previous "Total Void Amount" and "Net Amount PKR" rows were removed because void is excluded.)

---

## Adjustment Date Mode

When enabled, date filtering uses:
- If `adjustment_date` is NULL: filter by `created_at`
- If `adjustment_date` exists: filter by `adjustment_date`
- An "Adj Date" column is added to the report

See `docs/adjustment-date.md` for full details.

---

## Data Sources

### Primary Tables

| Table | Purpose |
|-------|---------|
| `payment_settlements` | Main settlement records |
| `payment_settlement_payments` | Payment method details (Cheque/Cash/Card) |
| `payment_settlement_costs` | Links settlements to costs |
| `suppliers` | Payee information |
| `chart_of_accounts` | Bank/Cash box details |
| `pay_type_forms` | Payment type labels |
| `currency_codes` + `currencies` | Currency and exchange rate |
| `documents` (type='costing') | XO document numbers |

---

## Company Scoping

Scoped via `payment_settlement → user → where: { company_code: req.user.company_code }`.

---

## Report Metadata

```javascript
{
  report_number: "PMSR" + timestamp,
  report_type: "payment-settlement-report",
  file_type: "xlsx" | "pdf"
}
```

---

## Advance Payments Section

Below the payment settlements table, an optional **Advance Payments** section is rendered when supplier advance payment records exist matching the same filters.

### Data Source

**Table**: `supplier_deposits`

Filtered by:
- Same date range, branch, and supplier filters as the main settlement section
- `status != 'Void'` (only Printed deposits)
- Company scoped via `supplier_deposit → user → where: { company_code }`

### Columns

| Column | Source | Description |
|--------|--------|-------------|
| Date | `supplier_deposit.created_at` | Payment creation date (DD MMM YYYY) |
| Voucher No | `supplier_deposit.payment_number` | Advance payment document number |
| Transaction | `supplier.supp_name` / `bank.name` | Supplier name and bank name joined with " / " |
| Description | `supplier_deposit.remarks` | Free-text remarks |
| Debit | `supplier_deposit.amount` | Original deposit amount (money paid out) |
| Credit | `amount - current_amount` | Utilized/settled portion |

### Totals

- **Total Debit**: Sum of all debit values
- **Total Credit**: Sum of all credit values

### Display Behavior

- Section only appears when matching advance payment records exist
- Description column uses word-wrap for long text (max-width: 25%)
- Both PDF and Excel exports include this section

---

## Verification Results (March 2026)

Verified with 3 payment settlements (TTPY00000001-03):
- TTPY00000001: Cheque payment 5,050 (no supplier) ✓
- TTPY00000002: Cash payment 3,000 (no supplier) ✓
- TTPY00000003: Direct settlement 5,000 to Q & K SUPPLIER LTD. with XO doc TTXO00000002 ✓
- All totals (13,050) verified correct. No bugs found.

---

## Code References

| File | Purpose |
|------|---------|
| `psfront/src/pages/Report/PaymentListingReport.jsx` | Frontend UI |
| `psback/controllers/report.controller.js` | Controller logic (~line 7962) |
| `psback/views/pages/reports/settlement.ejs` | PDF template |
| `psfront/src/api/report.js` | API client |

---

**Document End**
