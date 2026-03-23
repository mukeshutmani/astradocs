# PowerSuite Posting Rules Setup Guide

## Overview
This guide provides the complete configuration for journal entry posting rules based on the financial consultant's requirements (implemented January 2025). The structure separates revenue components, includes matching cost of sales entries, and properly handles discounts/rebates when present. Journal entries are now organized into period-wise batches for better period management and reporting.

## Critical Setup Requirements

### 1. Prefix Configuration
**IMPORTANT**: The prefix must match your branch's document prefix exactly!
- For branch 'TT': Use prefix 'TTIN' for invoices (NOT 'TTINV')
- Format: `${branch_prefix}${document_type}`
- Examples: 'TTIN', 'TTXO', 'TTDP', 'TTRS', 'TTPS'

### 2. Understanding Invoice Fields
- **invoice.price**: Already includes markup (e.g., 67,000 = 64,000 cost + 3,000 markup)
- **invoice.markup**: Separate field for tracking markup amount
- **Discounts/Rebates**: Apply to invoice.price (not price + markup)

## Invoice (INV) Type Posting Rules

### Required Rules for ALL Invoices

| Entry Type | Code | DR/CR | Customer Category | Product Code | Chart of Account | Description |
|------------|------|-------|-------------------|--------------|------------------|-------------|
| **Account Receivable** | AREC | Debit | All | All | 151110 - Trade Debtors | Total customer receivable |
| **Sales Tax** | STAX | Credit | All | All | 261000 - Tax Payable | SST on transaction fees |
| **Air Sales Revenue** | ASLE | Credit | All | All | 410100 - Air Sales Revenue | Base ticket revenue |
| **Air Sales Tax Revenue** | ATAX | Credit | All | All | 410105 - Air Sales Tax Revenue | Airline taxes as revenue |
| **PSF Markup Revenue** | PSFM | Credit | All | All | 410115 - PSF Markup Revenue | Markup on tickets |
| **PSF Transaction Fee** | PSFT | Credit | All | All | 410120 - PSF Transaction Fee Revenue | Service fees |
| **Cost of Sales** | CSAL | Debit | All | All | 520110 - Cost of Sales - Air | Direct ticket cost |
| **Cost of Sales Tax** | CSTX | Debit | All | All | 520115 - Cost of Sales - Air Tax | Tax costs |

### BSP-Specific Rules

| Entry Type | Code | DR/CR | Customer Category | Product Code | Chart of Account | Description |
|------------|------|-------|-------------------|--------------|------------------|-------------|
| **IATA Clearing (Int'l)** | IATA | Credit | All | 14/Int'l Air Ticket - BSP | 231120 - IATA-BSP Clearing | BSP payable for international |
| **IATA Clearing (Domestic)** | IATA | Credit | All | 13/Domestic Air Ticket | 231120 - IATA-BSP Clearing | BSP payable for domestic |

### Conditional Rules (When Discounts/Rebates Exist)

| Entry Type | Code | DR/CR | Customer Category | Product Code | Chart of Account | Description |
|------------|------|-------|-------------------|--------------|------------------|-------------|
| **Discount** | DISC | Debit | All | All | 530000 - Sales Discount* | Reduces revenue |
| **Rebate** | RBTE | Debit | All | All | 530000 - Sales Rebate* | Reduces revenue |

*Create appropriate discount/rebate accounts if not existing

## Cost/Exchange Order (XO) Type Posting Rules

| Entry Type | Code | DR/CR | Description |
|------------|------|-------|-------------|
| **IATA Clearing** | IATA | Debit | BSP cost expense |
| **Advance Tax** | ATAX | Debit | Advance tax on purchases |
| **Account Payable** | APAY | Credit | Total payable to suppliers |

## Customer Deposit (DP) Type Posting Rules

| Entry Type | Code | DR/CR | Description |
|------------|------|-------|-------------|
| **Cash/Bank** | CASH | Debit | Cash/bank receipt |
| **Customer Deposit** | DEPO | Credit | Liability to customer |

## Receipt Settlement (RS) Type Posting Rules

| Entry Type | Code | DR/CR | Description |
|------------|------|-------|-------------|
| **Settlement** | SETL | Debit | Reverse deposit liability |
| **Account Receivable** | AREC | Credit | Clear invoice receivable |

## Payment Settlement (PS) Type Posting Rules

| Entry Type | Code | DR/CR | Description |
|------------|------|-------|-------------|
| **Account Payable** | APAY | Debit | Clear supplier payable |
| **Cash/Bank** | CASH | Credit | Cash outflow |

## Setup Instructions

### Step 1: Set Up Journal Periods (Required First)
1. Go to GL > Journal Periods
2. Generate periods for your fiscal year
3. Verify period dates are correct
4. Ensure periods are active (status = 1)

### Step 2: Navigate to Posting Rules
1. Go to GL > Posting Rules Maintenance
2. Select Posting Number Type: INV (Sales Invoice)
3. Select your Branch

### Step 3: Add Rules via UI
For each rule in the tables above:
1. Click "Add Rule"
2. Select Entry Type
3. Set DR/CR
4. Set Customer Category (usually "All")
5. Set Product Code (usually "All" except for IATA)
6. Select Chart of Account
7. Set Type as "Custom" or "Additional"
8. Save

### Step 4: Verify Configuration
```sql
-- Check all posting rules for your branch
SELECT 
    pr.prefix_code,
    jet.code as entry_type,
    pr.debit_credit,
    pr.chart_of_account_id,
    coa.key_account,
    coa.description
FROM posting_rules pr
JOIN journal_entry_types jet ON pr.journal_entry_type_id = jet.id
LEFT JOIN chart_of_accounts coa ON pr.chart_of_account_id = coa.id
WHERE pr.prefix_code = 'TTIN'  -- Replace with your prefix
ORDER BY jet.code;
```

## Common Configuration Errors

### 1. Wrong Prefix
- **Error**: Rules not applying to documents
- **Fix**: Ensure prefix matches exactly (e.g., 'TTIN' not 'TTINV')

### 2. Missing Discount/Rebate Rules
- **Error**: Journal batch unbalanced when invoices have discounts
- **Fix**: Add DISC and RBTE posting rules with appropriate accounts

### 3. Duplicate Rules
- **Error**: Multiple rules for same entry type
- **Fix**: Use "Additional" type to override defaults, remove duplicates

### 4. Wrong Account Assignments
- **Error**: Accounts don't match business requirements
- **Fix**: Verify with chart of accounts and financial consultant

## Testing Your Configuration

### 1. Create Test Invoice
- Include base price, markup, taxes
- Add discount and rebate percentages
- Include transaction fees

### 2. Generate Journal Entries
- Go to GL > Journal Entries
- Click "Generate System JE"
- Select date range including test invoice
- System will automatically create separate batches for each period

### 3. Verify Balance
```sql
-- Check if journal batch is balanced
SELECT
    batch_no,
    journal_entry_period,
    SUM(debit) as total_debit,
    SUM(credit) as total_credit,
    SUM(debit) - SUM(credit) as imbalance
FROM journal_entries je
JOIN journal_batches jb ON je.journal_batch_id = jb.id
WHERE jb.batch_no = 'YOUR_BATCH_NO'
GROUP BY jb.id, jb.batch_no, jb.journal_entry_period;
```

## Example Configuration for Branch 'TT'

```sql
-- Complete set of posting rules for branch TT
INSERT INTO posting_rules (prefix_code, journal_entry_type_id, debit_credit, chart_of_account_id, record_type) 
SELECT 'TTIN', jet.id, dr_cr, coa.id, 'Additional'
FROM (VALUES
    ('AREC', 'debit', '151110'),
    ('STAX', 'credit', '261000'),
    ('ASLE', 'credit', '410100'),
    ('ATAX', 'credit', '410105'),
    ('PSFM', 'credit', '410115'),
    ('PSFT', 'credit', '410120'),
    ('CSAL', 'debit', '520110'),
    ('CSTX', 'debit', '520115'),
    ('IATA', 'credit', '231120'),
    ('DISC', 'debit', '530000'),
    ('RBTE', 'debit', '530000')
) AS rules(entry_code, dr_cr, account_key)
JOIN journal_entry_types jet ON jet.code = rules.entry_code
JOIN chart_of_accounts coa ON coa.key_account = rules.account_key;
```

## Credit Note (CN) Type Posting Rules

### Required Rules for ALL Credit Notes

| Entry Type | Code | DR/CR | Customer Category | Product Code | Chart of Account | Description |
|------------|------|-------|-------------------|--------------|------------------|-------------|
| **Revenue Reversal** | CNRV | Debit | All | All | 410100 - Air Sales Revenue | Reverse revenue |
| **AR Reduction** | CNAR | Credit | All | All | 151110 - Trade Debtors | Reduce receivables |

### Optional Rules (When Refunds Issued)

| Entry Type | Code | DR/CR | Customer Category | Product Code | Chart of Account | Description |
|------------|------|-------|-------------------|--------------|------------------|-------------|
| **Refund Liability** | CNRF | Debit | All | All | 245200 - Refunds Payable | Create refund liability |
| **Handling Fee** | CNHF | Debit | All | All | 410130 - Handling Fee Revenue | Handling fee on refund |
| **Cash/Bank** | CASH | Credit | All | All | varies | Refund payment |

## Debit Note (DN) Type Posting Rules

### Required Rules for ALL Debit Notes

| Entry Type | Code | DR/CR | Description |
|------------|------|-------|-------------|
| **AP Reduction** | DNAP | Debit | Reduce accounts payable |
| **Expense Recovery** | DNEX | Credit | Recovery/other income |

### Optional Rules (When Refunds Received)

| Entry Type | Code | DR/CR | Description |
|------------|------|-------|-------------|
| **Refund Recovery** | DNRF | Debit | Supplier refund receivable |
| **Cash/Bank** | CASH | Credit | Refund received |

## Enhanced Receipt Settlement (RS) Rules

### Updated Settlement Rules (with Credit Notes)

| Entry Type | Code | DR/CR | Description |
|------------|------|-------|-------------|
| **AR Clearance** | AREC | Credit | Clear invoice receivables |
| **Deposit Application** | SETL | Debit | Apply customer deposits |
| **Credit Note Application** | CNAR | Debit | Apply credit notes |
| **Cash Payment** | CASH | Debit | Direct cash/bank payments |

**Note**: Settlement rules now support multiple payment sources:
- Previous customer deposits
- Credit notes
- New payments (cash, cheque, card, etc.)

Each component creates separate journal entries for proper tracking.

## Troubleshooting

### Journal Entries Not Generating
1. Check document status (must be 'Printed')
2. Verify je_generated is NULL
3. Ensure posting rules exist for the branch
4. Check date range includes document dates
5. **Verify journal periods are created and active**
6. **Check document transaction dates fall within period ranges**

### Unbalanced Entries
1. Verify all posting rules are configured
2. Check for missing DISC/RBTE when discounts exist
3. Ensure calculations in journal.js are correct
4. Verify invoice.total_amount matches calculation
5. **For credit notes**: Ensure CNRV (debit) and CNAR (credit) amounts match
6. **For settlements**: Verify all payment sources sum to invoice totals

### Accounts Not Displaying
1. Check chart_of_account_id is set in posting rules
2. Verify accounts exist and are active (status = 1)
3. Check frontend display logic

### Period-Related Issues
1. **No batches created**: Check if journal periods exist for the selected date range
2. **Wrong period assignment**: Verify document transaction_date is correct
3. **Multiple batches unexpected**: This is normal - system creates one batch per period

### Credit Note Issues
1. **No entries generated**: Check branch has CN posting rules (prefix: TTCN)
2. **Missing refund entries**: Verify refund_amount > 0 and CNRF/CASH rules exist
3. **je_generated column missing**: Run SQL_UPDATES_CN_DN_SETTLEMENTS.sql script

### Debit Note Issues
1. **No entries generated**: Check branch has DN posting rules (prefix: TTDN)
2. **je_generated column missing**: Run SQL_UPDATES_CN_DN_SETTLEMENTS.sql script

### Settlement Issues
1. **Credit notes not appearing**: Check receipt_settlement_credit_notes table has entries
2. **Multiple payments not tracked**: Verify receipt_settlement_payments table
3. **Old implementation**: If using hardcoded accounts, update to posting rules system

## Refund (RF) Type Posting Rules

### Overview
Refund journal entries **reverse the original invoice and cost document entries**. The system automatically calculates proportional reversals based on customer and supplier refund amounts.

### Required Rules for Customer Refunds

| Entry Type | Code | DR/CR | Chart of Account | Description |
|------------|------|-------|------------------|-------------|
| **AR Reduction** | RFAREC | Credit | 151110 - Trade Debtors | Reduce customer receivable |
| **Sales Revenue Reversal** | RFASLE | Debit | 410100 - Air Sales Revenue | Reverse base sales revenue |
| **Sales Tax Reversal** | RFATAX | Debit | 410105 - Air Sales Tax Revenue | Reverse airline tax revenue |
| **Markup Reversal** | RFPSFM | Debit | 410115 - PSF Markup Revenue | Reverse markup revenue |
| **Transaction Fee Reversal** | RFPSFT | Debit | 410120 - PSF Transaction Fee Revenue | Reverse fee revenue |
| **SST Reversal** | RFSTAX | Debit | 261000 - Tax Payable | Reverse sales tax liability |

### Optional Customer Refund Rules

| Entry Type | Code | DR/CR | Chart of Account | Description |
|------------|------|-------|------------------|-------------|
| **Discount Reversal** | RFDISC | Credit | 530000 - Sales Discount | Reverse discount expense |
| **Rebate Reversal** | RFRBTE | Credit | 530000 - Sales Rebate | Reverse rebate expense |

### Required Rules for Supplier Refunds

| Entry Type | Code | DR/CR | Chart of Account | Description |
|------------|------|-------|------------------|-------------|
| **AP Reduction** | RFAPAY | Debit | 231000 - Accounts Payable | Reduce supplier payable |
| **Cost Reversal** | RFCSAL | Credit | 520110 - Cost of Sales - Air | Reverse cost of sales |
| **Cost Tax Reversal** | RFCSTX | Credit | 520115 - Cost of Sales - Air Tax | Reverse cost tax |

### Optional Supplier Refund Rules

| Entry Type | Code | DR/CR | Chart of Account | Description |
|------------|------|-------|------------------|-------------|
| **IATA Clearing Reversal** | RFIATA | Debit | 231120 - IATA-BSP Clearing | Reverse BSP liability (BSP airlines only) |
| **WHT Reversal** | RFATXC | Credit | 151200 - Advance Tax Receivable | Reverse withholding tax asset |
| **Commission Reversal** | RFCOMM | Debit | 420100 - Commission Income | Reverse commission income |

### Setup Example for Branch 'TT'

```sql
-- Customer Refund Entries
INSERT INTO posting_rules (branch_id, company_code, prefix_code,
    journal_entry_type_id, chart_of_account_id, debit_credit, record_type)
SELECT 1, 'COMP01', 'TTRF', jet.id, coa.id, dr_cr, 'Default'
FROM (VALUES
    ('RFAREC', 'credit', '151110'),
    ('RFASLE', 'debit', '410100'),
    ('RFATAX', 'debit', '410105'),
    ('RFPSFM', 'debit', '410115'),
    ('RFPSFT', 'debit', '410120'),
    ('RFSTAX', 'debit', '261000'),
    ('RFDISC', 'credit', '530000'),
    ('RFRBTE', 'credit', '530000'),
    -- Supplier Refund Entries
    ('RFAPAY', 'debit', '231000'),
    ('RFCSAL', 'credit', '520110'),
    ('RFCSTX', 'credit', '520115'),
    ('RFIATA', 'debit', '231120'),
    ('RFATXC', 'credit', '151200'),
    ('RFCOMM', 'debit', '420100')
) AS rules(entry_code, dr_cr, account_key)
JOIN journal_entry_types jet ON jet.code = rules.entry_code
JOIN chart_of_accounts coa ON coa.key_account = rules.account_key;
```

### Important Notes

1. **Reversal Logic**:
   - Refunds reverse original invoice/cost entries
   - Amounts are calculated proportionally based on refund percentage
   - Example: 50% refund reverses 50% of original revenue

2. **Customer vs Supplier Refunds**:
   - Customer refund entries only created if `customer_refund_amount > 0`
   - Supplier refund entries only created if `supplier_refund_amount > 0`
   - Both can exist for the same refund

3. **Prerequisites**:
   - Refund must link to original invoice (`invoice_id`)
   - Refund must link to original cost (`cost_id`)
   - Refund status must be "Printed"
   - `je_generated` must be NULL

4. **Verification**:
```sql
-- Check refund posting rules
SELECT pr.prefix_code, jet.code, pr.debit_credit, coa.key_account
FROM posting_rules pr
JOIN journal_entry_types jet ON pr.journal_entry_type_id = jet.id
LEFT JOIN chart_of_accounts coa ON pr.chart_of_account_id = coa.id
WHERE pr.prefix_code = 'TTRF'  -- Replace with your branch prefix
ORDER BY jet.code;
```

### Refund Issues

1. **No entries generated**:
   - Check branch has RF posting rules (prefix: TTRF, SSRF, etc.)
   - Verify refund status is "Printed"
   - Ensure customer_refund_amount or supplier_refund_amount > 0
   - Check invoice_id and cost_id are not null

2. **Entries not balanced**:
   - Verify all required posting rules are configured
   - Check both customer AND supplier entries if both refund amounts exist
   - Ensure discount/rebate rules exist if original invoice had discounts

3. **Wrong amounts calculated**:
   - Verify refund percentage calculation
   - Check original invoice total_amount is correct
   - Ensure invoice and cost data properly linked

## Support Contacts
For assistance with posting rules configuration:
- Technical Support: Check journal.js calculations
- Financial Consultant: Verify account mappings
- System Administrator: Database and permission issues