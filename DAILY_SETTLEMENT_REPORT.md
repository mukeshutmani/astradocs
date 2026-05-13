# Daily Settlement Report - Technical Documentation

**Version**: 1.0
**Date**: March 2026
**Author**: System Analysis
**Status**: Stable - Bug Fixed

---

## Overview

The Daily Settlement Report shows all receipt settlements received from customers, including deposits used, credit notes applied, invoices settled, and GL account payments. Each settlement is displayed with its component rows (invoices, deposits, credit notes) and a per-settlement total.

---

## Technical Architecture

### Frontend Component

**File**: `psfront/src/pages/Report/DailySettlementReport.jsx`

**Filters**:
- Date Range: =, <, <=, >, >=, <>, between (on `created_at`)
- Branch: isNotBlank, isBlank, isEqual, between
- Customer: isNotBlank, isBlank, isEqual, between

**Output**: PDF or Excel

### Backend Controller

**File**: `psback/controllers/reports/dailySettlement.report.controller.js`
**Function**: `getDailySettlementReport`
**Route**: `POST /api/report/getDailySettlementReport`

### PDF Template

**File**: `psback/views/pages/reports/settlement.ejs` (shared with Payment Settlement Report)

---

## Report Layout

### Row Structure Per Settlement

Each settlement generates multiple rows:

1. **Receipt Row** (main row): Date, Staff ID, "Receipt" type, document number, first invoice number, first deposit/credit note/GL account info, first invoice settled amount
2. **Additional Invoice Rows**: One row per additional invoice (2nd, 3rd, etc.)
3. **Additional Deposit Rows**: One row per additional deposit (2nd, 3rd, etc.)
4. **Credit Note Rows**: One row per credit note (skips first if no deposits exist, since it's shown in Receipt row)
5. **Total Row**: Per-settlement totals

### Columns

| Column | Source | Description |
|--------|--------|-------------|
| Date | `receipt_settlement.created_at` | Settlement date (DD-MM-YYYY) |
| Payment Staff ID | `user.username` | User who created the settlement |
| Type No. | "Receipt" or "Credit Note" | Row type identifier |
| Document | `receipt_settlement.receipt_number` | Settlement document number |
| Invoice No. | `invoice.invoice_number` | Invoice being settled |
| TCID | (unused) | Always empty |
| Card No./Cheq.No./Cr.NoteNo./Acc.No. | Varies | Deposit receipt number, credit note reference, or GL account |
| Customer No. | `customer.customer_number` | Customer code |
| Settled Amt. | Deposit/credit note/GL amount | Amount from payment method |
| Invoice Amt. | Calculated | Outstanding amount BEFORE this settlement (`total_amount - sum(prior settlements)`) |
| Balance | Calculated | Outstanding amount AFTER this settlement (`Invoice Amt - settled portion in this receipt`) |
| Overpayment | Calculated | If settled > invoice total |
| Deposit | (unused) | Always empty |
| Refund (Fr.Supplier) | `refund.customer_refund_amount` | Sum of refund amounts |
| Settled Amt.(Local Curr.) | Same as Settled Amt. × exchange rate | Local currency equivalent |
| Exchange Rate | `currency.exchange_rate` | From invoice currency |

### First Row Priority Logic

The first row's "Card No./Cheq.No." and "Settled Amt." show (in priority order):
1. First deposit → `customer_deposit.receipt_number` and deposit settled amount
2. First credit note → `credit_note.reference` and credit note amount
3. First GL payment → `chart_of_account.key_account - description` and GL payment amount

---

## Calculation Logic

### Invoice Amt. and Balance (Per Row)

```
priorSettled = sum(all receipt_settlement_invoices.amount for this invoice
               from settlements created BEFORE the current settlement)
invoiceAmt = invoice.total_amount - priorSettled   // Outstanding before this settlement
balance = invoiceAmt - settledPortionInThisReceipt  // Outstanding after this settlement
```

Prior settlements are calculated chronologically using `created_at ASC, id ASC` ordering, so within the same date, settlements with higher IDs are considered later.

### Overpayment

```
overpayment = invoiceSettledAmount > invoiceTotalAmount
  ? (invoiceSettledAmount - invoiceTotalAmount)
  : ""
```

### Per-Settlement Totals

```
Total Settled Amt = sum(all settled_amount values in settlement rows)
Total Invoice Amt = sum(all invoice_amount values in settlement rows)
Total Balance = sum(all balance values in settlement rows)
Total Settled Amt Local = sum(all settled_amount_local values)
```

---

## Data Sources

### Primary Tables

| Table | Purpose |
|-------|---------|
| `receipt_settlements` | Main settlement records |
| `receipt_settlement_invoices` | Invoice settlement amounts |
| `receipt_settlement_deposits` | Deposit usage records |
| `receipt_settlement_credit_notes` | Credit note usage |
| `receipt_settlement_payments` | GL account payments |
| `invoices` | Invoice details |
| `customers` | Customer info |
| `customer_deposits` | Deposit details |
| `credit_notes` | Credit note details |
| `currencies` + `currency_codes` | Exchange rates |
| `chart_of_accounts` | GL account descriptions |
| `users` | Staff info |

### Query Filters

- Settlement status: `['Printed', 'je']`
- Company scoped via `user.company_code`
- Date filter on `receipt_settlement.created_at`

---

## Company Scoping

Scoped via `receipt_settlement → user → where: { company_code: req.user.company_code }`.

---

## Report Metadata

```javascript
{
  report_number: "DSR" + timestamp,
  report_type: "daily-settlement",
  file_type: "xlsx" | "pdf"
}
```

---

## Bug Fix History

### Change #1: Invoice Amt Column Redesign (March 2026)

**Original behavior**: "Invoice Amt." showed the settled portion per settlement.

**New behavior**: "Invoice Amt." now shows the **outstanding balance before this settlement**, and a new **"Balance"** column shows the outstanding after this settlement.

**Example** (TTIN00000003, total 65,050, three settlements):
- TTST00000001 (first): Invoice Amt = 65,050, settled 13,576.21, Balance = 51,473.79
- TTST00000002 (second): Invoice Amt = 51,473.79, settled 12,000, Balance = 39,473.79
- TTST00000009 (third): Invoice Amt = 39,473.79, settled 5,000, Balance = 34,473.79

### Change #2: Sort Order Fix (March 2026)

**Problem**: Settlements on the same date appeared in inconsistent order.

**Fix**: Added secondary sort `id DESC` so latest settlements (higher ID) appear first within the same date. Query order: `[['created_at', 'DESC'], ['id', 'DESC']]`.

### Change #3: Manual JE Rows (May 2026)

**Behavior**: Manual JE batches that settle invoices (`journal_entries.analysis_code1 = invoice_number`) now appear in the report alongside Receipt / Credit Note rows.

**Source**:
- Reads `journal_entries` joined to `journal_batches` where `batch_type = 'Manual JE'` AND batch `status != 'Void'` AND row description NOT LIKE `'VOID REVERSAL -%'`. Same `liveManualJeWhereClause` used by `services/manualJeAdjustment.js`. → Voided JEs and their reversal rows drop out automatically.
- Date filter is mirrored from the receipt query but applied to `journal_entries.transaction_date`.
- Company scoping: `journal_batch.branch_id → branches.company_code = req.user.company_code`, validated against the invoice's `order.branch_id`.
- Customer / branch filters are applied through `invoice → service → order` (same path as receipts).

**Column mapping** (per JE row tagged to an invoice):

| Column | Source |
|---|---|
| Date | `journal_batches.dated` (first row of the batch only) |
| Payment Staff ID | `journal_batches.created_by → users.username` |
| Type No. | `"JE"` (first row only) |
| Document | `journal_batches.batch_no` (first row only) |
| Invoice No. | `journal_entries.analysis_code1` |
| TCID | empty |
| Card No./Cheq.No./… | `chart_of_account.key_account-description` of the **offset legs** of the JE batch. An offset leg is any row in the same `journal_batch` whose `analysis_code1` is either NULL or NOT one of the customer-invoice numbers being rendered as primary rows (so rows tagged to an XO number, a free-text ref, or no ref at all are all considered offset legs). Labels are deduped and joined with `, ` if multiple. |
| Customer No. | `invoice → service → order.customer → customer_number` |
| Settled Amt. | `je.debit + je.credit` (only one side is populated on any row, so the sum equals the row's absolute amount) |
| Invoice Amt. | `Σ (invoice.total_amount × invoice.exchange_rate) − prior_settled` aggregated across ALL `invoices` rows that share the same `invoice_number` for this customer (multi-line invoices). `total_amount` is stored in source currency, so each line is converted to PKR before summing. `prior_settled` is the chronological union of receipt + JE settlements on this `invoice_number`, also in PKR. PKR invoices (`exchange_rate = 1`) and single-line invoices reduce naturally to the single-row case. |
| Balance | Invoice Amt. − Settled Amt. |
| Overpayment | empty |
| Deposit | empty |
| Refund | empty |
| Settled Amt.(Local Curr.) | same as Settled Amt. (JE rows are stored in PKR) |
| Exchange Rate | `invoice.exchange_rate` of the linked invoice (e.g., `325.92` for an EUR invoice). PKR invoices show `1.00`. JE amounts themselves are always in PKR; this column reports the source invoice's FX rate so the row is consistent with how receipt rows display the rate. |

**Grouping & totals**: One group per `journal_batch`. Date / Type / Document / Card-No / Customer-No are populated only on the first row of the group; subsequent rows for additional invoices in the same JE batch show only the invoice-specific cells. A per-batch total row (`Total Amount (Valid) in PKR :`) is appended after each group, matching the receipt-group total pattern.

**Ordering**: After both receipt and JE groups are built, a post-processing step at the end of the controller slices `data1` into per-group chunks (each ending in a `{total: true}` row), parses the header row's `date.value` (DD-MM-YYYY), sorts ALL groups (receipt + JE together) by date DESC, and flattens back. This means receipts and JE rows interleave by date — the latest date appears first regardless of type. Groups with no parseable date sort to the bottom.

### Change #5: Daily Settlement Report History Card Removed (May 2026)

**Behavior**: The "Daily Settlement Report History" card on the report page (`/reports/daily-settlement` or wherever this page is mounted) has been removed. The page now shows only the filter + generate area; previously generated reports are no longer listed inline.

**File**: `psfront/src/pages/Report/DailySettlementReport.jsx` — the `<Card>` whose `CardTitle` was `Daily Settlement Report History`, along with its table body and pagination footer, was removed.

**Note**: The `fetchHistory` function, the `reports`/`pagination` state, and the mount-time history fetch are still present in the file but are now unused. They were intentionally left in place per the project's Scope Discipline rule (only remove what was explicitly asked for). Clean them up in a separate pass if desired.

### Change #4: Excel ↔ PDF Parity for Total Rows (May 2026)

**Behavior**: Excel and PDF render the per-group total row identically.

**What changed (Excel side only)**: The Excel writer now detects `row.total === true` and renders that row the same way PDF does:
- Merge columns 1 through `splitIndex` (the position of `settled_amount` — typically column 9) into a single right-aligned cell containing `row.grandTotal.value` (e.g., `"Total Amount (Valid) in PKR :"`).
- Populate only the right-side amount columns (`settled_amount`, `invoice_amount`, `balance`, `overpayment`, `deposit`, `settled_amount_local`, `exchange_rate`, `refund`).
- Apply bold font + light-grey fill (`FFD9D9D9`) + thin border to match the visual emphasis used in PDF.

**Why**: previously the Excel renderer ignored `row.total` (it only checked a stale `payment_staff_id.value === "Total"` condition that never matched the current row shape), so total rows in Excel appeared as ordinary blank rows with just the numeric totals — no label, no bold, no merge. After this change Excel mirrors PDF.

**Files**: `psback/controllers/reports/dailySettlement.report.controller.js` (Excel branch only). PDF template (`settlement.ejs`) and data builder are unchanged.

**Empty-header row**: When `settlements.length === 0` the receipt branch pushes a placeholder empty-headers row. If JE groups exist, that placeholder is removed before JE rows are rendered (no need for the placeholder when JE rows already populate the columns).

**Code reference**: `psback/controllers/reports/dailySettlement.report.controller.js` — Manual JE block sits after the receipt-settlement loop and before the report-creation step.

---

## Verification Results (March 2026)

Verified with 4 receipt settlements:
- TTST00000002: Single invoice 12,000 (ahsan) ✓
- TTST00000007: Single invoice 1,000 (qasim) ✓
- TTST00000008: Single invoice 4,000 with deposit (qasim) ✓
- TTST00000001: Two invoices (13,576.21 + 6,523.79) + deposit 10,100 + credit note 10,000 = 20,100 (qasim) ✓
- All totals verified correct after bug fix.

---

## Code References

| File | Purpose |
|------|---------|
| `psfront/src/pages/Report/DailySettlementReport.jsx` | Frontend UI |
| `psback/controllers/reports/dailySettlement.report.controller.js` | Controller logic |
| `psback/views/pages/reports/settlement.ejs` | PDF template (shared) |
| `psback/routes/report.route.js` | Route definition (line 52) |
| `psfront/src/api/report.js` | API client |

---

**Document End**
