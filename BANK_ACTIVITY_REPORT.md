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
- Banks: isNotBlank, isBlank, isEqual, between (company-wise GL-account list shown as `key_account / description`; filters on `bank_account.id`)
- Branch: isNotBlank, isBlank, isEqual
- JE Period: isNotBlank, isBlank, isEqual

**Output**: PDF or Excel

**Filter retention & inline preview**:
- Selected filters are retained after generating (module-level `cachedFilter`); returning to the page keeps prior selections. A full page refresh resets to `DEFAULT_FILTER`.
- PDF is rendered inline below the filters (iframe → `{VITE_API_URL}/document/view/{report_number}`) with a **Preview** button (opens the full PDF in a new tab). No navigation away from the page. Excel still downloads and clears the inline PDF.

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
4. **Net Ledger Balance Row**: Per-bank DR − CR balance with label "Net Ledger Balance"; value shown in the DR Amount column (CR Amount left blank)
5. **Grand Total Row**: Overall DR and CR totals across all banks
6. **Grand Net Ledger Balance Row**: Overall DR − CR balance across all banks, shown in the DR Amount column

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
netLedgerBalance = totalDr - totalCr   // shown in the "Net Ledger Balance" row
```

### Grand Totals

```
grandTotalDr = sum(totalDr) across all bank groups
grandTotalCr = sum(totalCr) across all bank groups
grandNetLedgerBalance = grandTotalDr - grandTotalCr   // shown in the "Grand Net Ledger Balance" row
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
Applied on `bank_account.id` (so a selection narrows to one specific GL account):
- `isEqual`: specific GL account
- `between`: range of bank-account IDs (auto-swaps if start > end)

The dropdown is built **company-wise** on the frontend from the already company-scoped bank accounts (`getBankAccountsApi`). Each option is shown by its **GL account** as `key_account / description` (e.g. `181020 / MEEZAN BANK E-SAFAR`), the same label as the Bank Accounts page; the option value is the `bank_account.id`. No new endpoint is used. The request uses the `bankAccountFilter` / `bank_account_id` / `bank_account_idStart` / `bank_account_idEnd` fields, which carry **bank-account** ids. The on-report filter caption also shows the GL account label.

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

### Banks filter now lists GL accounts (2026-06-15)

**Goal**: Show each Banks-filter option by its **GL account** (`key_account / description`, e.g. `181020 / MEEZAN BANK E-SAFAR`) and filter the report to that single GL account. This supersedes the earlier same-day "whole-bank" filtering.

**Changes**:
1. Frontend `BankActivityReport.jsx`: the dropdown is built directly from the company's bank accounts; each option's label is its GL account and its value is the `bank_account.id`. All three pickers (Is Equal, Between start/end) use this list.
2. Backend `getBankActivityReport`: the filter now matches `bank_account.id` (isEqual / between) instead of `bank_account.bank_id`.
3. Backend filter caption: the "Banks" / "Banks Range" label now reads the GL account (`bank_account → chart_of_account`) instead of the bank name.

### "Net Ledger Balance" row added per bank (2026-06-15)

**Goal**: Under each bank's "Net for [Bank]" totals, show one more row giving that bank's **DR − CR balance**.

**Changes**:
1. Controller `getBankActivityReport` now pushes a second row (`isBankBalance: true`) right after each bank's "Net for" total row, labelled **"Net Ledger Balance"**, with `totalDr - totalCr` placed in the **DR Amount** column (CR Amount blank).
2. PDF template `settlement.ejs` gained an `isBankBalance` branch that renders this row in bold (same style as the total row); the blank spacer that separates banks was moved to follow this new row.
3. Excel needs no change — its writer loops over all rows, so the new row appears automatically.

**Note**: A negative balance (more credit than debit) displays as a negative number, e.g. `-6,750.00`.

### "Grand Net Ledger Balance" row added (2026-06-15)

**Goal**: After the Grand Total row, show one more row giving the overall **DR − CR balance** across all banks.

**Changes**:
1. Controller `getBankActivityReport` builds a `data1GrandBalance` object (`reference` = "Grand Net Ledger Balance", `dr_amount` = `grandTotalDr - grandTotalCr`, `cr_amount` blank) and passes it to both the Excel writer and the PDF render.
2. PDF template `settlement.ejs` renders `data1GrandBalance` (bold) right after the Grand Total row, guarded by `typeof data1GrandBalance !== 'undefined'` so other reports sharing the template are unaffected.
3. Excel writer adds the same row right after the grand total row.

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
