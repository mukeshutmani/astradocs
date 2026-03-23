# Balance Sheet Report

## Overview
The Balance Sheet report shows the financial position of the company at a specific point in time (journal period). It displays Assets, Liabilities, and Shareholders' Equity with both Current Period activity and Year-To-Date cumulative values.

## Files

| File | Purpose |
|------|---------|
| `psfront/src/pages/Report/BalanceSheet.jsx` | Frontend form with filters |
| `psback/controllers/reports/balanceSheet.report.controller.js` | Backend controller - data queries and report generation |
| `psback/views/pages/reports/balance-sheet.ejs` | EJS template for PDF rendering |
| `psback/routes/report.route.js` (line ~71) | Route: `POST /api/report/getBalanceSheet` |
| `psfront/src/api/report.js` | Frontend API call `getBalanceSheet()` |

## Filters

| Filter | Required | Description |
|--------|----------|-------------|
| As of Period (JE Period) | Yes | MMYYYY format (e.g., 012026 = January 2026) |
| Branch | No | All Branches (default) or Specific Branch |
| Sub A/C No. | No | Filter by sub-account (e.g., BMC, BTT, ETB) |

## Report Columns

| Column | Description |
|--------|-------------|
| Account No. | `key_account` + `sub_account_no` (e.g., 151110BMC) |
| Account Name | Chart of account description |
| Opening Balance | Balance before current period activity (`YTD - Current Period`) |
| Current Period | Activity for the selected period only |
| Year To Date | Cumulative from fiscal year start to selected period |

## Data Logic

### Fiscal Year Determination
- Fiscal year is determined from `journal_periods` table using the selected period's month/year
- The `fiscal_year` field (e.g., "072025") indicates the fiscal year start (July 2025)
- YTD periods = all months from fiscal year start to selected period

### Account Types
- **Assets** (`A/Assets`): Amount = Debit - Credit (debit positive)
- **Liabilities** (`L/Liabilities`): Amount = Credit - Debit (credit positive)
- **Shareholders' Equity** (`S/Shareholders' Equity`): Amount = Credit - Debit (credit positive)

### Opening Balance Calculation
- Opening Balance = Year To Date - Current Period
- Represents the account balance at the start of the selected period (before current period transactions)
- Calculated per account row and for all totals/subtotals (Total Assets, Total Liabilities, Total Shareholders' Equity, Retained Earnings, Grand Total)

### Retained Earnings
- Calculated from P&L accounts: Revenue (`R/Revenue/Sales`) minus Expenses (`C/Cost of Sales`, `E/Expense/COGS`, `E/Expenses`)
- Shown as a separate line under Shareholders' Equity when non-zero
- Opening Balance for Retained Earnings = YTD Net Income - Current Period Net Income

### Report Structure
```
ASSETS (centered header)
  Assets (category header)
    Account rows...
    --- subtotal ---

LIABILITIES & SHAREHOLDERS' EQUITY (centered header)
  Liabilities (category header)
    Account rows...
    --- subtotal ---
  Shareholders' Equity (category header)
    Account rows...
    --- subtotal ---
    === GRAND TOTAL ===
```

### Balance Rule
- Total Assets = Total Liabilities + Total Shareholders' Equity (including Retained Earnings)

## Formatting
- Grid borders: light gray (`#ccc`) borders on all table cells and outer border
- Column header row: `#f0f0f0` background
- Section headers (ASSETS, LIABILITIES & SHAREHOLDERS' EQUITY): `#e8e8e8` background, centered
- Subtotal and final total rows: `#f5f5f5` background, dashed top border
- Negative values displayed in parentheses: `(85,695.00)`
- Currency: PKR (Pakistan Rupee)

## Database Tables Used
- `journal_entries` - Financial transactions (debit/credit amounts)
- `journal_batches` - Batch info with branch_id and journal_entry_period
- `chart_of_accounts` - Account definitions (key_account, description, account_type)
- `sub_accounts` - Sub-account codes (sub_account_no like BMC, BTT)
- `journal_periods` - Period definitions with fiscal_year mapping
- `branches` - Branch info (code, name, document_prefix)
- `companies` - Company info
- `reports` - Report history records

## Sub-Account (Sub A/C) Concept
- Sub-accounts are stored in `sub_accounts` table with fields: `id`, `sub_account_no`, `description`
- Journal entries reference sub-accounts via `sub_account_id` foreign key
- In the report, Account No. = key_account + sub_account_no (e.g., `151110` + `BMC` = `151110BMC`)
- Examples: BMC (Bukhari Travel - Malir Cant.), BTT (Bukhari Travel & Tourism), ETB (ET-GSA), UMB (BTTS UMRAH)

## Reference
- Based on RPT501 - Balance Sheet from PowerSuite Cloud
- Report permission: `Balance-Sheet-Report`
