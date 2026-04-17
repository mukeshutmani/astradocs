# Trial Balance by Journal Entry Report

## Overview
The Trial Balance by Journal Entry report shows all account balances **as on** a single journal period (month). For every account it displays:

1. **Opening** (Debit / Credit) — cumulative balance from all entries **before** the selected period.
2. **Activities** (Debit / Credit) — net movement **inside** the selected period.
3. **Closing** (Debit / Credit) — Opening + Activities.

Rows are broken down **per branch** and grouped by key account. Rollup (parent) accounts are auto-summed from their children.

## Files

| File | Purpose |
|------|---------|
| `psfront/src/pages/Report/TrailBalance.jsx` | Frontend form with filters |
| `psback/controllers/report.controller.js` (`getTrailBalanceByJE`, line ~2383) | Backend controller |
| `psback/views/pages/reports/trial-balance.ejs` | EJS template for PDF rendering |
| `psback/routes/report.route.js` (line 51) | Route: `POST /api/report/trailBalanceByJE` |
| `psfront/src/api/report.js` | Frontend API call `getTrailBalanceByJE()` |

## Permission
`Trial-Balance-Report`

## Filters

| Filter | Required | Notes |
|--------|----------|-------|
| Journal Period Filter (`journalPeriodFilter`) | Yes | Supported values: `=`, `<`, `<=`, `>`, `>=`, `<>`, `between`, `blank` |
| Start Journal Period (`startJournalPeriod`) | Yes (unless filter is `blank`) | `MMYYYY` format, e.g. `042026` = Apr 2026 |
| End Journal Period (`endJournalPeriod`) | Only when filter is `between` | `MMYYYY` format |
| Branch Filter (`branchFilter`) | No | `isBlank` / `isEqual` / `isNotBlank` (default) |
| Branch ID (`branch_id`) | Only when `branchFilter = isEqual` | Integer FK to `branch.id` |
| Key Account Filter (`keyAccountFilter`) | No | `isBlank` / `isEqual` / `isNotBlank` (default) |
| Key Account (`key_account`) | Only when `keyAccountFilter = isEqual` | Chart of account key |
| Show Zero (`show_zero`) | No | If false, rows with 0 opening & 0 activity are hidden |
| Type (`type`) | No | `pdf` (default) or `excel` |

## Report Columns

| Column | Description |
|--------|-------------|
| Je Period | Formatted selected period (e.g. `Apr2026`) |
| Fiscal Year | Year portion of the period (e.g. `2026`) |
| Document Prefix | Branch document prefix |
| Key Account | Chart of account key |
| Account Name | Chart of account description |
| Opening Debit | Opening balance if positive (debit side) |
| Opening Credit | Opening balance if negative (credit side) |
| Activities Debit | Period activity if positive |
| Activities Credit | Period activity if negative |
| Closing Debit | Closing balance if positive |
| Closing Credit | Closing balance if negative |
| Rollup / Detail | `R` for rollup parent, `D` for detail account |
| Account Type | First character of `account_type` (e.g. `A`, `L`, `R`, `E`, `S`) |

## Data Logic

### 1. Journal Entries Query
1. Pulls from `journal_entry` joined to `journal_batch` and `branch`.
2. Filtered by the current user's `company_code`.
3. Grouped by `branch_id`, `branch.code`, `branch.name`, `branch.document_prefix`, and `chart_of_account_id`.
4. Returns `SUM(debit)` and `SUM(credit)` per group.

### 2. Opening Balance
1. If `startJournalPeriod` is set, a second query runs with `journal_entry_period < startJournalPeriod`.
2. Opening balance per (branch, account) = `SUM(debit) - SUM(credit)`.
3. Stored in-memory by key `${branchId}_${accountId}`.

### 3. Period Activities
1. For each (branch, account) from the current-period query: `activities = debit - credit`.
2. `closing = opening + activities`.

### 4. Debit / Credit Split
1. Positive value → placed in the **Debit** column.
2. Negative value → absolute value placed in the **Credit** column.
3. Applied independently to Opening, Activities, and Closing.

### 5. Zero Balance Handling
1. When `show_zero` is false, rows where **both** opening balance AND period activity are exactly 0 are skipped.
2. Otherwise every account that appears in the chart of accounts is shown.

### 6. Rollup Accounts
1. Rollup accounts (`type` starts with `roll`) that have no direct entries but whose children do appear are added automatically.
2. The rollup's prefix is derived by stripping trailing zeros from its `key_account`.
3. Any detail row whose `key_account` starts with that prefix is treated as a child.
4. The rollup row sums Opening, Activities, and Closing values of its children.

### 7. Branch Handling
1. Entries with no branch are grouped under a `NO_BRANCH` key with code `N/A` and name `No Branch`.
2. When `branch_id` is supplied, only that branch is included (enforced in the Sequelize `include` on `db.branch`).
3. Within each key account, branches are sorted by branch `code` ascending.

### 8. Ordering
1. Primary: `key_account` ascending (so rollup parents appear before detail children alphabetically).
2. Secondary: branch `code` ascending inside each key account group.

### 9. Grand Total
1. One `GRAND TOTAL` row is appended summing every displayed row's six debit/credit amounts.
2. Not account-type aware — it is a simple column-wise sum of what is shown.

## Output

### PDF
1. Rendered by `psback/views/pages/reports/trial-balance.ejs`.
2. Uploaded to S3/MinIO via `uploadFile()`; a signed link is returned.

### Excel
1. Generated by `createTrialBalanceExcel()` with the column schema listed above.
2. Numeric columns (opening/activities/closing debit & credit) are formatted as numbers.

### Report Record
1. A row is created in the `report` table with:
   - `report_number` = `TPJE` + `Date.now()`
   - `report_type` = `trail-balance-by-journal-entry`
   - `file_type` = `pdf` or `xlsx`
2. Returned in the API response for later retrieval.

## API

### Request — `POST /api/report/trailBalanceByJE`
```json
{
  "journalPeriodFilter": "=",
  "startJournalPeriod": "042026",
  "endJournalPeriod": null,
  "branchFilter": "isEqual",
  "branch_id": 3,
  "keyAccountFilter": "isNotBlank",
  "key_account": null,
  "show_zero": false,
  "type": "pdf"
}
```

### Response (success)
```json
{
  "status": 200,
  "message": "success",
  "link": "<s3-key>",
  "downloadLink": "<signed-url-if-excel>",
  "report": { "id": 123, "report_number": "TPJE1713350000000", ... }
}
```

### Response (error)
```json
{
  "status": 500,
  "message": "Internal Server Error",
  "error": "<sequelize/db message>"
}
```

## Known Edge Cases / Notes
1. **Branch column naming** — the `branch` model uses `code` / `name`, not `branch_code` / `branch_name`. The filter-label lookup (controller line ~2811) uses `attributes: ['code', 'name']`.
2. **`NO_BRANCH` bucket** — entries whose `journal_batch.branch_id` is null are aggregated under a synthetic `NO_BRANCH` key rather than dropped.
3. **Rollup sorting** — rollup rows are merged into `data1` and then the whole array is sorted by `key_account`, so a rollup like `1511100` will sit near its children `1511101`, `1511102`, etc.
4. **Grand Total is not type-aware** — it sums debit and credit columns directly. For a traditional trial balance balance-check (Total Debits == Total Credits), read the Closing Debit vs Closing Credit totals.
5. **`show_zero = false`** hides only rows where both opening and activity are 0. An account with a non-zero opening but no activity this period will still appear.
