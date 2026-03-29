# Chart of Accounts - Copy from Company

## Overview
Allows copying Chart of Accounts from one company to another. Only missing accounts (matched by `key_account`) are added to the target company. Existing accounts are never modified or deleted.

## Workflow
1. Click **"Copy from Company"** button on Chart of Accounts page (green button, visible when GL Edit Mode is ON)
2. Select **Source Company** (copy FROM)
3. Select **Target Company** (copy TO)
4. Click **Analyze** to see comparison
5. Review the analysis: source count, target count, missing accounts preview
6. Click **Copy X Missing Accounts** to execute
7. Results summary shows what was added and what was skipped

## API Endpoints

### GET `/chartOfAccount/analyze_companies`
Analyzes and compares accounts between two companies.

**Query Parameters:**
- `source_company_code` (required) - Company code to copy from
- `target_company_code` (required) - Company code to copy to

**Response:**
```json
{
  "source_count": 332,
  "target_count": 307,
  "common_count": 296,
  "missing_count": 36,
  "only_in_target_count": 11,
  "missing": [{ "key_account": "421100", "description": "Tour Cost", ... }],
  "only_in_target": [{ "key_account": "999999", "description": "Custom Account" }]
}
```

### POST `/chartOfAccount/copy_from_company`
Copies missing accounts from source to target company.

**Request Body:**
```json
{
  "source_company_code": "9876",
  "target_company_code": "1003"
}
```

**Response:**
```json
{
  "source_company_code": "9876",
  "target_company_code": "1003",
  "source_total": 332,
  "total_added": 36,
  "total_skipped": 296,
  "added_accounts": [{ "key_account": "421100", "description": "Tour Cost" }]
}
```

## Behavior
- **Matching**: Accounts are matched between companies by `key_account` number
- **Missing Only**: Only accounts that exist in source but NOT in target are copied
- **Existing Preserved**: Accounts already in target are never modified or deleted
- **Target-Only Preserved**: Accounts that exist only in target are left untouched
- **Sub Account Mapping**: If source account has a `sub_account_id`, it maps to the target company's sub account by `sub_account_no`
- **User Assignment**: New accounts are assigned to the first user found in the target company
- **Transaction Safety**: All operations run in a single database transaction (all or nothing)

## Fields Copied
- key_account, description, account_type, journal_entry_type_id
- level, type, account_number, status
- sub_account_id (mapped by sub_account_no), ar_ap_control
- disable_sub_account, fx_adjustment, petty_cash_account

## Guards
- Requires `GL_EDIT_MODE=true` environment variable
- Requires `view-account` permission (analyze) / `create-account` permission (copy)
- Requires JWT authentication

## Typical Use Case
1. Company 9876 has 332 accounts (fully set up)
2. Company 1003 is new with 307 accounts (missing some)
3. Run copy: 36 missing accounts are added to 1003
4. Run again later: if 9876 gets new accounts, only the new ones will be copied (existing skip)
