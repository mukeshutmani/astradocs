# Posting Rules Replica Configuration

## Overview

The Posting Rules Replica feature allows administrators to copy an entire set of posting rules from one branch (source) to one or more target branches in a single operation. This eliminates the manual, time-consuming process of re-creating identical posting rules branch by branch.

**Implementation Date**: 2026-03-29
**Status**: Complete
**UI Location**: Posting Rule Maintenance page (GL > Posting Rule) — "Copy to Branches" button

## Business Problem

When a new branch is created or posting rules are updated, administrators must manually configure every posting rule (AREC, ASLE, ATAX, PSFM, PSFT, STAX, CSAL, CSTX, IATA, DISC, RBTE, and all refund types) for each branch individually. With 10+ branches and 15+ rules per posting type, this means hundreds of manual entries — error-prone and time-consuming.

## Solution

A **"Copy to Branches"** button on the existing Posting Rule Maintenance page that opens a dialog allowing:

1. **Source Branch**: The currently selected branch (already loaded with its rules)
2. **Target Branches**: Multi-select checkboxes to pick one or more destination branches
3. **Scope**: Copy rules for the currently selected posting number type, or all posting types at once
4. **Overwrite Option**: Choose whether to overwrite existing rules on target branches or skip branches that already have rules

## How It Works

### Prefix Code Adjustment

Each branch has a unique `document_prefix` (e.g., TT, LHE, SS). When rules are copied, the `prefix_code` is automatically adjusted:

```
Source Branch: TT (document_prefix = "TT")
Target Branch: LHE (document_prefix = "LHE")

Source Rule prefix_code: TTIN  →  Target Rule prefix_code: LHEIN
Source Rule prefix_code: TTXO  →  Target Rule prefix_code: LHEXO
Source Rule prefix_code: TTRF  →  Target Rule prefix_code: LHERF
```

### What Gets Copied

For each source posting rule, the following fields are replicated to target branches:

| Field | Behavior |
|-------|----------|
| `journal_entry_type_id` | Copied as-is |
| `posting_number_type_id` | Copied as-is |
| `branch_id` | Changed to target branch ID |
| `debit_credit` | Copied as-is |
| `service_type_id` | Copied as-is |
| `sub_account_id` | Copied as-is |
| `chart_of_account_id` | Copied as-is |
| `customer_type_id` | Copied as-is |
| `prefix_code` | Rebuilt with target branch document_prefix |
| `record_type` | Copied as-is (Default/Additional) |
| `company_code` | Set to current user's company_code |

### What Does NOT Change

- Chart of Account mappings remain identical (same GL accounts across branches)
- Entry type configurations (DR/CR) remain identical
- Service type and customer type filters remain identical

## Usage Workflow

### Step 1: Configure Source Branch
1. Navigate to **GL > Posting Rule**
2. Select the **Posting Number Type** (e.g., INV/Sales Invoice)
3. Select the **Source Branch** (e.g., TTINV/Sales Invoice (TT))
4. Verify the rules are correctly configured

### Step 2: Copy to Target Branches
1. Click the **"Copy to Branches"** button (next to Add Rule / Save Changes)
2. In the dialog:
   - Source branch is displayed (read-only)
   - Check the **target branches** you want to copy to
   - Toggle **"All Posting Types"** to copy across INV, XO, DP, RS, PS, RF types
   - Toggle **"Overwrite existing rules"** if you want to replace rules on branches that already have them
3. Click **"Copy Rules"**
4. Review the summary showing how many rules were created per branch

### Step 3: Verify
1. Switch to a target branch in the dropdown
2. Confirm the rules appear correctly
3. Verify prefix codes match the target branch

## Overwrite Behavior

| Overwrite OFF (default) | Overwrite ON |
|------------------------|--------------|
| Skips branches that already have rules for the selected posting type | Deletes existing rules on target branches first, then creates new copies |
| Safe for incremental setup | Use when source rules have changed and targets need full refresh |
| Reports skipped branches in summary | Reports overwritten count in summary |

## API Endpoint

### POST `/posting-rule/copy_to_branches`

**Authentication**: Required
**Permission**: Requires `GL_EDIT_MODE = true`

**Request Body:**
```json
{
    "source_branch_id": 1,
    "target_branch_ids": [2, 3, 5],
    "posting_number_type_id": 4,
    "overwrite": false
}
```

- `posting_number_type_id`: Optional. If omitted, copies rules for ALL posting types.
- `overwrite`: Optional (default: false). If true, deletes existing target rules before copying.

**Response (201):**
```json
{
    "total_created": 30,
    "total_skipped": 0,
    "total_overwritten": 0,
    "source_rules_count": 10,
    "branches": [
        {
            "branch_id": 2,
            "branch_name": "Lahore",
            "status": "copied",
            "rules_created": 10
        },
        {
            "branch_id": 3,
            "branch_name": "Islamabad",
            "status": "copied",
            "rules_created": 10
        },
        {
            "branch_id": 5,
            "branch_name": "Karachi",
            "status": "copied",
            "rules_created": 10
        }
    ]
}
```

**Possible statuses per branch:**
- `copied` — Rules successfully created
- `skipped` — Branch skipped (already has rules and overwrite is off, or same as source)

## Troubleshooting

### 1. Copy Button Disabled
- Ensure a Posting Number Type and Branch are selected
- Ensure `GL_EDIT_MODE` is enabled in the backend `.env` file

### 2. Target Branch Shows "Skipped"
- The target branch already has posting rules for that posting type
- Enable the **"Overwrite existing rules"** toggle to replace them

### 3. Wrong Prefix Codes After Copy
- Check the target branch's `document_prefix` in the branch configuration
- The prefix code is built as: `{branch.document_prefix}{postingType.code.substring(0,2)}`

### 4. Rules Not Appearing After Copy
- Refresh the page
- Switch the branch dropdown to the target branch
- Verify the correct posting number type is selected

## Example Scenario

**Setup**: TT branch has 11 invoice posting rules configured correctly. Need to replicate to LHE, ISB, and KHI branches.

1. Go to Posting Rule page, select INV + TT branch — see 11 rules
2. Click "Copy to Branches"
3. Check LHE, ISB, KHI — toggle "All Posting Types" ON
4. Click "Copy Rules"
5. Result: 11 rules x 6 posting types x 3 branches = 198 rules created
6. Each branch now has correct prefix codes (LHEIN, ISBN, KHIIN, etc.)

---

## Cross-Company Copy (Copy from Company)

### Overview
Copies posting rules from another company's branch into your company's branches. Automatically maps `chart_of_account_id` and `sub_account_id` across companies by matching `key_account` and `sub_account_no`.

### Prerequisites
- Chart of Accounts must be synced between companies first (use Chart of Accounts "Copy from Company" feature)
- `GL_EDIT_MODE` must be enabled

### UI Location
Green **"Copy from Company"** button on Posting Rule Maintenance page.

### Workflow
1. Click **"Copy from Company"**
2. Select **Source Company** (e.g., 9876)
3. Select **Source Branch** (e.g., TT)
4. Your company is automatically set as target (read-only)
5. Select **Target Branches** in your company (multi-select)
6. Toggle options (All posting types / Overwrite)
7. Click **Copy Rules**
8. Review results including any unmapped accounts

### Cross-Company Field Mapping

| Field | Behavior |
|-------|----------|
| `chart_of_account_id` | Mapped by `key_account` number across companies |
| `sub_account_id` | Mapped by `sub_account_no` across companies |
| `branch_id` | Changed to target branch ID |
| `prefix_code` | Rebuilt with target branch `document_prefix` |
| `company_code` | Set to logged-in user's company_code |
| All other fields | Copied as-is |

### Unmapped Accounts Warning
If a source posting rule references a chart of account that doesn't exist in the target company (no matching `key_account`), the rule is still created but with `chart_of_account_id = null`. These are reported in the results as "Unmapped Accounts".

### API Endpoint

#### GET `/posting-rule/branches_by_company`
Returns branches for a specific company.

**Query Parameters:** `company_code` (required)

#### POST `/posting-rule/copy_from_company`

**Request Body:**
```json
{
    "source_company_code": "9876",
    "source_branch_id": 1,
    "target_branch_ids": [10, 11, 12],
    "posting_number_type_id": 4,
    "overwrite": false
}
```

**Response (201):**
```json
{
    "total_created": 30,
    "total_skipped": 0,
    "total_overwritten": 0,
    "source_rules_count": 10,
    "unmapped_accounts": [],
    "branches": [
        { "branch_id": 10, "branch_name": "TT", "status": "copied", "rules_created": 10 }
    ]
}
```
