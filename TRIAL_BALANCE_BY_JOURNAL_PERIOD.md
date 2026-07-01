# Trial Balance by Journal Period - Implementation Plan

## Overview
Create a new Trial Balance report that allows selection of a journal period (from-to) with separate debit/credit column display.

## Key Differences from Current Trial Balance
1. **Period Range Selection**: Instead of single period, users select start and end journal periods
2. **Debit/Credit Columns**: Instead of single "Period Activities" column showing net amount, display separate Debit and Credit columns
3. **Opening Balance Calculation**: Calculate based on all entries BEFORE the start period
4. **Closing Balance Calculation**: Calculate based on opening + all activities within the period range

## Implementation Steps

### 1. Frontend Changes

#### A. Add Menu Item (Report.jsx)
```javascript
// Add to GL section after existing Trial Balance
{
  label: "Trial Balance by Journal Period",
  onClick: () => navigate("trailBalanceByJournalPeriod")
}
```

#### B. Create New Component (TrailBalanceByJournalPeriod.jsx)
- Copy existing TrailBalance.jsx as base
- Modify to include:
  - Start Journal Period field
  - End Journal Period field
  - Both fields required for generation
  - Pass both periods to backend API

#### C. Filter retention & inline preview
- Selected filters are retained after generating (module-level `cachedFilter` in `TrailBalanceByJournalPeriod.jsx`); returning to the page keeps prior selections. A full page refresh resets to `DEFAULT_FILTER`.
- PDF is rendered inline below the filters (iframe → `{VITE_API_URL}/document/view/{report_number}`) with a **Preview** button (opens the full PDF in a new tab). No navigation away from the page. Excel still downloads and clears the inline PDF.

### 2. Backend Implementation

#### A. Add New Route
```javascript
// In routes/report.route.js
router.post("/trail-balance-by-je-range", getTrailBalanceByJERange);
```

#### B. Controller Logic (report.controller.js)

```javascript
exports.getTrailBalanceByJERange = async (req, res) => {
  const {
    startJournalPeriod,  // e.g., "052025"
    endJournalPeriod,     // e.g., "072025"
    branch_id,
    show_zero = false
  } = req.body;

  // Key Logic Changes:

  // 1. OPENING BALANCE CALCULATION
  // Get all entries BEFORE startJournalPeriod
  openingBatchWhere = {
    journal_entry_period: {
      [Sequelize.Op.lt]: startJournalPeriod
    }
  };

  // 2. PERIOD ACTIVITIES
  // Get all entries BETWEEN start and end periods (inclusive)
  periodBatchWhere = {
    journal_entry_period: {
      [Sequelize.Op.between]: [startJournalPeriod, endJournalPeriod]
    }
  };

  // 3. DATA STRUCTURE
  // For each account/branch combination:
  const rowData = {
    je_period_from: startJournalPeriod,
    je_period_to: endJournalPeriod,
    fiscal_year: extractFiscalYear(startJournalPeriod),
    document_prefix: branch.document_prefix,
    key_account: account.key_account,
    account_name: account.description,

    // Opening Balance (before start period)
    opening_debit: openingBalance > 0 ? openingBalance : 0,
    opening_credit: openingBalance < 0 ? Math.abs(openingBalance) : 0,

    // Period Activities (during period range)
    period_debit: periodDebits,
    period_credit: periodCredits,

    // Closing Balance (opening + activities)
    closing_debit: closingBalance > 0 ? closingBalance : 0,
    closing_credit: closingBalance < 0 ? Math.abs(closingBalance) : 0,

    account_type: accountTypeCode
  };
}
```

### 3. Report Column Structure

| Column | Description |
|--------|-------------|
| JE Period From | Start period (e.g., 052025) |
| JE Period To | End period (e.g., 072025) |
| Fiscal Year | Extracted from periods |
| Doc Prefix | Branch document prefix |
| Key Account | Account number |
| Account Name | Account description |
| Opening Debit | Opening balance if debit |
| Opening Credit | Opening balance if credit |
| Period Debit | Total debits in period range |
| Period Credit | Total credits in period range |
| Closing Debit | Closing balance if debit |
| Closing Credit | Closing balance if credit |
| Account Type | A/L/R/E etc. |

### 4. Balance Calculation Logic

```javascript
// For each account:
openingBalance = sum(debits before start) - sum(credits before start)
periodDebits = sum(debits in range)
periodCredits = sum(credits in range)
closingBalance = openingBalance + periodDebits - periodCredits

// Display logic:
if (balance > 0) -> show in Debit column
if (balance < 0) -> show in Credit column (as positive)
if (balance = 0) -> show 0 in both or hide based on show_zero flag
```

### 5. SQL Queries Required

1. **Opening Balance Query**:
```sql
SELECT
  chart_of_account_id,
  branch_id,
  SUM(debit) as total_debit,
  SUM(credit) as total_credit
FROM journal_entries je
JOIN journal_batches jb ON je.journal_batch_id = jb.id
WHERE jb.journal_entry_period < :startPeriod
GROUP BY chart_of_account_id, branch_id
```

2. **Period Activities Query**:
```sql
SELECT
  chart_of_account_id,
  branch_id,
  SUM(debit) as period_debit,
  SUM(credit) as period_credit
FROM journal_entries je
JOIN journal_batches jb ON je.journal_batch_id = jb.id
WHERE jb.journal_entry_period BETWEEN :startPeriod AND :endPeriod
GROUP BY chart_of_account_id, branch_id
```

### 6. Validation Rules
- Start period must be <= End period
- Both periods required
- Validate period format (MMYYYY)
- Handle cases where no data exists in range

### 6a. Period Dropdown Behaviour (Start / End Period)
- The Start/End Period dropdowns are built in `fetchJournalPeriods` (`TrailBalanceByJournalPeriod.jsx`) from `GET /jvPeriod`.
- **Open periods only**: periods with `status = 1` are shown; closed periods (`status = 0`) are hidden.
- **Latest first**: periods are sorted newest year/month at the top for quicker selection.

### 7. Testing Scenarios
1. Single period (start = end)
2. Multiple periods span
3. No data in range
4. Negative balances display
5. Zero balance handling
6. Cross-fiscal year periods

### 8. PDF Template Updates
- Adjust column widths for additional columns
- Ensure debit/credit alignment
- Format negative numbers appropriately
- Add period range in header

## File List to Create/Modify

### New Files:
1. `/psfront/src/pages/Report/TrailBalanceByJournalPeriod.jsx`

### Modified Files:
1. `/psfront/src/pages/Report/Report.jsx` - Add menu item
2. `/psback/routes/report.route.js` - Add route
3. `/psback/controllers/report.controller.js` - Add getTrailBalanceByJERange
4. `/psback/views/pages/reports/report1.ejs` - Ensure template handles new columns

## Notes
- Uses permission "Trial-Balance-By-Journal-Period-Report"
- Consider performance for large date ranges
- Maintain consistency with existing Trial Balance format where applicable
- Ensure proper branch filtering and company code validation