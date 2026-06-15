# Bank Activity Report - Technical Documentation

**Version**: 1.0
**Date**: March 2026
**Author**: System Analysis
**Status**: Stable - Verified

---

## Overview

The Bank Activity Report shows all journal entry transactions associated with bank accounts, grouped by bank name. Each bank group displays its entries with batch numbers, entry numbers, dates, references, cheque numbers, and DR/CR amounts, followed by a per-bank net total. A grand total summarizes all banks.

---

## Technical Architecture

### Frontend Component

**File**: `psfront/src/pages/Report/BankActivityReport.jsx`

**Filters**:
- Date Range: =, <, <=, >, >=, <>, between (on `transaction_date`)
- Banks: isNotBlank, isBlank, isEqual, between (company-wise bank list; filters on the account's `bank_id`)
- Branch: isNotBlank, isBlank, isEqual
- JE Period: isNotBlank, isBlank, isEqual

**Output**: PDF or Excel

### Backend Controller

**File**: `psback/controllers/report.controller.js`
**Function**: `getBankActivityReport` (line 9366)
**Route**: `POST /api/report/getBankActivityReport`
**Permission**: `Bank-Activity-Report`

### PDF Template

**File**: `psback/views/pages/reports/settlement.ejs` (shared with Daily Settlement and Payment Settlement reports)
**Config**: `totalColsize: 7`

---

## Report Layout

### Grouping Structure

Entries are grouped by bank name. Each bank group contains:

1. **Bank Header Row**: Bank name in Batch No. column, account number in Account No. column
2. **Entry Rows**: One row per journal entry, sorted by date descending within each bank
3. **Net Total Row**: Per-bank DR and CR totals with label "Net for [Bank Name]"
4. **Grand Total Row**: Overall DR and CR totals across all banks

Banks are sorted by their most recent transaction date (newest first).

### Columns

| Column | Source | Description |
|--------|--------|-------------|
| Batch No. | `journal_batch.batch_no` | Journal batch number |
| Entry No. | `journal_entry.entry_no` | Entry number within the batch |
| Date | `journal_entry.transaction_date` | Transaction date (DD MMM YYYY) |
| Account No. | `bank_account.account_number` | Bank account number |
| Line No. | `journal_entry.id` | Journal entry ID (primary key) |
| Reference | `journal_entry.description` + `journal_entry.analysis_code1` | Reference text; "undefined" removed, analysis_code1 appended if not already present |
| Remarks | source document `remarks` | Remarks of the source document behind the entry, matched by `analysis_code1` (see [Remarks Column](#remarks-column-2026-06-08)) |
| Cheque No. | `journal_entry.analysis_code5` | Cheque number |
| DR Amount | `journal_entry.debit` | Debit amount (2 decimal places) |
| CR Amount | `journal_entry.credit` | Credit amount (2 decimal places) |

---

## Calculation Logic

### Per-Bank Totals

```
totalDr = sum(entry.debit) for all entries in the bank group
totalCr = sum(entry.credit) for all entries in the bank group
```

### Grand Totals

```
grandTotalDr = sum(totalDr) across all bank groups
grandTotalCr = sum(totalCr) across all bank groups
```

### Bank Display Name Priority

```
1. bank.name (from banks table)
2. bank_account.bank_account_code
3. bank_account.account_number
4. "Unknown Bank" (fallback)
```

### Reference Text Construction

```javascript
referenceText = entry.description (with "undefined" removed)
if (entry.analysis_code1 && !referenceText.includes(analysis_code1)) {
    referenceText = referenceText + " " + entry.analysis_code1
}
```

---

## Data Sources

### Primary Tables

| Table | Purpose |
|-------|---------|
| `journal_entries` | Main transaction records (debit, credit, description, analysis codes) |
| `journal_batches` | Batch info (batch_no, journal_entry_period) |
| `chart_of_accounts` | GL account descriptions |
| `bank_accounts` | Bank account details (account_number, bank_account_code) |
| `banks` | Bank names |
| `branches` | Company scoping |

### Key Joins

```
journal_entries → journal_batches (journal_batch_id)
journal_entries → chart_of_accounts (chart_of_account_id)
chart_of_accounts → bank_accounts (one-to-many)
bank_accounts → banks (bank_id)
journal_batches → branches (branch_id)
```

### Query Constraints

- `bank_account` join is `required: true` — only entries linked to bank accounts are included
- `journal_batch` join is `required: true`
- `branch` join is `required: true` with company scoping via `company_code`
- Results ordered by `batch_no ASC`, then `entry_no ASC`
- Within each bank group, entries sorted by `transaction_date` descending

---

## Company Scoping

Scoped via `journal_batch → branch → where: { company_code: req.user.company.code }`.

---

## Filter Logic

### Date Filter
Applied on `journal_entry.transaction_date`:
- `=` operator: matches entire day (startOf to endOf day)
- `between`: uses start and end dates
- Other operators: direct comparison

### Banks Filter
Applied on `bank_account.bank_id` (so a selected bank matches every account of that bank):
- `isEqual`: specific bank
- `between`: range of bank IDs (auto-swaps if start > end)

The dropdown is built **company-wise** on the frontend from the already company-scoped bank accounts (`getBankAccountsApi`), de-duplicated by the account's parent `bank`. No new endpoint is used. Manually-typed banks (accounts with `bank_id = NULL`) do not appear as selectable banks; they still show under the default "Is Not Blank" (all) view. The request still uses the `bankAccountFilter` / `bank_account_id` / `bank_account_idStart` / `bank_account_idEnd` fields, which now carry **bank** ids.

### Branch Filter
Applied on `branch.id`.

### JE Period Filter
Applied on `journal_batch.journal_entry_period` (fiscal year).

---

## Report Metadata

```javascript
{
  report_number: "RPT565" + timestamp,
  report_type: "bank-activity-report",
  file_type: "xlsx" | "pdf"
}
```

---

## Excel Output

- Header section frozen (first 7 rows)
- Amount columns right-aligned with `#,##0.00` format
- Bank header rows, entry rows, bank total rows, and grand total row
- Grand total row has bold font with light gray background
- Auto-fit column widths (min 10, max 50)

---

## Verification Results (March 2026)

Verified with batch TTJV000007 (3 entries across 2 banks):
- MCB Bank Limited: Line 8146 (DR 100,000, Cheque 645654) ✓
- MCB Bank Limited: Line 8148 (DR 2,000, Cheque 832667523) ✓
- MCB Bank Net: DR 102,000, CR 0 ✓
- Meezan Bank Limited: Line 8150 (DR 6,750) ✓
- Meezan Bank Net: DR 6,750, CR 0 ✓
- Grand Total: DR 108,750, CR 0 ✓
- All values verified correct against database. No bugs found.

---

## Recent Updates

### "Bank Account" filter renamed to "Banks" (2026-06-15)

**Goal**: Let the user filter the report by a whole **bank** (company-wise) instead of by an individual bank account.

**Changes**:
1. Frontend label changed from "Bank Account" to **"Banks"**.
2. The dropdown now lists the **company's banks** (de-duplicated from the company-scoped bank accounts, each of which carries its parent bank) showing `bank.name` with `bank.id` as the value. No new API call.
3. Backend `getBankActivityReport` now filters on `bank_account.bank_id` (isEqual / between) instead of `bank_account.id`, so selecting a bank includes all of that bank's accounts.
4. The on-report filter label looks up the **bank** name and reads "Banks" / "Banks Range".

**Note**: Request field names were left as-is (`bankAccountFilter`, `bank_account_id`, `bank_account_idStart`, `bank_account_idEnd`); they now carry bank ids.

### Remarks Column (2026-06-08)

**Goal**: Show a **Remarks** column (right after Reference) holding the remarks written on the source document behind each bank entry.

**Why a lookup is needed**: `journal_entries` has no remarks field. The remark lives on the source document, which the entry references through `analysis_code1` (the document number).

**How it works** (`getBankActivityReport` in `report.controller.js`):
1. After loading the journal entries, the distinct document numbers from `analysis_code1` are collected (the trailing ` (Void)` marker is stripped first).
2. Remarks are fetched in bulk from the source tables, **scoped to the current company** (join `users.company_code`) so identical document numbers in other companies cannot match:
   - `DP` → `customer_deposits.receipt_number` → `remarks`
   - `AP` → `supplier_deposits.payment_number` → `remarks`
   - `PY` → `payment_settlements.payment_number` → `remarks` (multiple settlement rows per number; one is taken via `MAX`)
3. A `remarks` cell is added to every entry row, plus empty placeholders on the bank header, per-bank total, and grand-total rows so columns line up.
4. PDF and Excel both build columns dynamically from the row data, so both gain the column. The PDF template (`settlement.ejs`) got a `key === 'remarks'` style so long remarks **wrap** (scoped to the remarks key only; the reports that share this template are unaffected).

**Not covered**: rare `OP` (overpayment) and `OJ` (opening) bank lines have no remarks-bearing source document, so their Remarks show blank.

---

## Code References

| File | Purpose |
|------|---------|
| `psfront/src/pages/Report/BankActivityReport.jsx` | Frontend UI |
| `psback/controllers/report.controller.js` (line 9366) | Controller logic |
| `psback/views/pages/reports/settlement.ejs` | PDF template (shared) |
| `psback/routes/report.route.js` (line 59) | Route definition |
| `psfront/src/api/report.js` | API client |

---

**Document End**
