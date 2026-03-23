# Customer Deposit Posting Rules Setup Guide

## Overview
This guide provides the complete configuration for customer deposit journal entry posting rules. Customer deposits create a liability that is later cleared when receipt settlements are processed. Each deposit generates exactly two journal entries based on the payment method selected.

## Company 9876 (New Testing Co.) - Chart of Accounts
For company code 9876, the following chart of accounts have been created by user **wajahat** (user_id: 54):

### Cash & Bank Accounts (Assets)
- **185000** - Cash
- **185001** - Cash Box
- **185002** - Cash Box - Petty Cash
- **181010** - General Bank
- **181030** - Standard Chartered Bank-Current
- **181040** - Testing Bank
- **181050** - Habib Bank Testing
- **181060** - Faysal Bank
- **181070** - NIB Bank
- **181080** - MCB Bank
- **181090** - BTTS New Bank

### Customer Deposit Accounts (Liabilities)
- **245100** - Customer Deposit / Overpayment (Primary deposit liability account)
- **245900** - Customer Deposit Suspense (Alternative for special cases)

## Journal Entry Structure

### Entry Types for Customer Deposits (DP)
Customer deposits always generate two balanced entries:

1. **CASH (Debit)**: Records the receipt of payment
   - Account depends on payment method selected
   - Amount: Full deposit amount

2. **DEPO (Credit)**: Records the customer deposit liability
   - Account: Customer Deposit Liability account (from posting rule)
   - Amount: Full deposit amount

## Payment Method to GL Account Mapping

### 1. Cash (ID: 1)
- **GL Account Source**: Selected from chart_of_account children dropdown
- **Field Used**: `customer_deposit.chart_of_account_id`
- **Example Accounts**: Cash on Hand, Petty Cash
- **Journal Description**: "Cash receipt [receipt_number]"

### 2. Cheque (ID: 2)
- **GL Account Source**: Bank account's chart_of_account_id
- **Field Used**: `customer_deposit.chart_of_account_id` (from selected bank)
- **Required Fields**: `bank_id`, `check_number`
- **Example Accounts**: Bank accounts linked to banks
- **Journal Description**: "Cheque receipt [receipt_number]"

### 3. Credit Card (ID: 3)
- **GL Account Source**: Bank account's chart_of_account_id
- **Field Used**: `customer_deposit.chart_of_account_id` (from selected bank)
- **Required Fields**: `bank_id`, `card_type_id`, `card_number`
- **Example Accounts**: Credit card merchant accounts
- **Journal Description**: "Credit Card receipt [receipt_number]"

### 5. GL Account (ID: 5)
- **GL Account Source**: Selected from GL Settlement Accounts
- **Field Used**: `customer_deposit.gl_account_id`
- **Example Accounts**: Any GL account configured in gl_settle_accounts
- **Journal Description**: "GL Account receipt [receipt_number]"

### 6. Direct Deposit / Bank Transfer (ID: 6)
- **GL Account Source**: Bank account's chart_of_account_id
- **Field Used**: `customer_deposit.chart_of_account_id` (from selected bank)
- **Required Fields**: `bank_id`
- **Example Accounts**: Bank accounts
- **Journal Description**: "Bank Transfer receipt [receipt_number]"

### 7. Debit Card (ID: 7)
- **GL Account Source**: Bank account's chart_of_account_id
- **Field Used**: `customer_deposit.chart_of_account_id` (from selected bank)
- **Required Fields**: `bank_id`, `card_type_id`, `card_number`
- **Example Accounts**: Debit card merchant accounts
- **Journal Description**: "Debit Card receipt [receipt_number]"

## Required Posting Rules Configuration

### Step 1: Set Up Journal Entry Types
Ensure these journal entry types exist in the `journal_entry_types` table:

| Code | Description | Type |
|------|-------------|------|
| CASH | Cash/Bank Receipt | Debit |
| DEPO | Customer Deposit Liability | Credit |

### Step 2: Configure Posting Rules
Add these posting rules for your branch. For Company 9876 branches:

**Example for TESTING BRANCH (TT):**
| Prefix | Entry Type | DR/CR | Chart of Account | Description |
|--------|------------|-------|------------------|-------------|
| TTDP | CASH | Debit | 185001 - Cash Box (Fallback) | Payment receipt |
| TTDP | DEPO | Credit | 245100 - Customer Deposit / Overpayment | Customer liability |

**Important**: The CASH entry will use the account from the deposit record based on payment method. The posting rule account is only a fallback.

### SQL to Add Posting Rules for Company 9876
```sql
-- Add Customer Deposit posting rules for TESTING BRANCH (branch_id: 22)
-- Company: 9876 (New Testing Co.)
INSERT INTO posting_rules (
    branch_id,
    company_code,
    prefix_code,
    journal_entry_type_id,
    debit_credit,
    chart_of_account_id,
    record_type,
    created_at,
    updated_at
)
SELECT
    22, -- TESTING BRANCH ID
    '9876', -- Company code for New Testing Co.
    'TTDP', -- TT (branch prefix) + DP
    jet.id,
    CASE
        WHEN jet.code = 'CASH' THEN 'debit'
        WHEN jet.code = 'DEPO' THEN 'credit'
    END,
    CASE
        WHEN jet.code = 'CASH' THEN 309 -- Cash Box (185001) - user wajahat
        WHEN jet.code = 'DEPO' THEN 321 -- Customer Deposit / Overpayment (245100) - user wajahat
    END,
    'Default',
    NOW(),
    NOW()
FROM journal_entry_types jet
WHERE jet.code IN ('CASH', 'DEPO');

-- For other branches in Company 9876, use their respective prefixes:
-- Malir Cant. Branch (MC): MCDP
-- Sialkot (BS): BSDP
-- UMRAH (UB): UBDP
-- Karachi Clifton (KC): KCDP
```

## Example Journal Entries

### Example 1: Cash Payment (Company 9876)
```
Customer Deposit: PKR 50,000
Payment Method: Cash
Selected Account: Cash Box (185001)

Journal Entries:
DR 185001 - Cash Box: 50,000
CR 245100 - Customer Deposit / Overpayment: 50,000
```

### Example 2: Bank Transfer (Company 9876)
```
Customer Deposit: PKR 100,000
Payment Method: Bank Transfer
Bank Account: Standard Chartered Bank-Current (181030)

Journal Entries:
DR 181030 - Standard Chartered Bank-Current: 100,000
CR 245100 - Customer Deposit / Overpayment: 100,000
```

### Example 3: Cheque Payment (Company 9876)
```
Customer Deposit: PKR 75,000
Payment Method: Cheque
Bank Account: Faysal Bank (181060)
Check Number: 12345

Journal Entries:
DR 181060 - Faysal Bank: 75,000
CR 245100 - Customer Deposit / Overpayment: 75,000
```

## Implementation Details

### Database Fields Used
- `customer_deposit.chart_of_account_id`: Primary GL account (for most payment methods)
- `customer_deposit.gl_account_id`: GL account for payment method type 5
- `customer_deposit.pay_type_id`: Payment method identifier
- `customer_deposit.amount`: Deposit amount
- `customer_deposit.receipt_number`: Unique deposit identifier
- `customer_deposit.je_generated`: Tracks if journal entries were created

### Journal Entry Analysis Codes
- `analysis_code1`: Receipt number
- `analysis_code2`: Customer ID
- `analysis_code3`: Bank ID (when applicable)
- `analysis_code4`: Branch ID
- `analysis_code5`: Check/Card number (when applicable)

## Testing Checklist

1. **Create Test Deposits for Each Payment Method**:
   - [ ] Cash payment with account selection
   - [ ] Cheque payment with bank and check number
   - [ ] Credit card payment with bank and card details
   - [ ] GL Account payment with GL account selection
   - [ ] Bank transfer with bank selection
   - [ ] Debit card payment with bank and card details

2. **Generate Journal Entries**:
   - [ ] Navigate to GL > Journal Entries
   - [ ] Click "Generate System JE"
   - [ ] Select date range including test deposits
   - [ ] Verify batch creation

3. **Verify Each Entry**:
   - [ ] Correct debit account based on payment method
   - [ ] Correct credit to Customer Deposit liability
   - [ ] Balanced entries (debit = credit)
   - [ ] Proper descriptions with payment method
   - [ ] Correct analysis codes

## Common Issues and Solutions

### Issue 1: Wrong GL Account Used
**Problem**: Journal entry uses wrong GL account for payment
**Solution**: Verify that:
- For GL Account method: `gl_account_id` is populated
- For other methods: `chart_of_account_id` is populated
- Bank accounts have correct `chart_of_account_id` linked

### Issue 2: Entries Not Generating
**Problem**: Customer deposits not creating journal entries
**Solution**: Check that:
- Posting rules exist with correct prefix (e.g., 'TTDP')
- Deposit status is 'Printed' (not 'Void')
- `je_generated` is NULL
- Date range includes deposit creation date

### Issue 3: Unbalanced Entries
**Problem**: Debit and credit don't match
**Solution**: Ensure:
- Both CASH and DEPO posting rules are configured
- Amount parsing is correct (no NULL values)
- No duplicate posting rules

## SQL Queries for Verification

```sql
-- Check unprocessed deposits
SELECT
    cd.receipt_number,
    cd.amount,
    cd.pay_type_id,
    ptf.label as payment_method,
    cd.chart_of_account_id,
    cd.gl_account_id,
    cd.je_generated
FROM customer_deposits cd
LEFT JOIN pay_type_forms ptf ON cd.pay_type_id = ptf.id
WHERE cd.je_generated IS NULL
  AND cd.status = 'Printed';

-- Verify posting rules
SELECT
    pr.prefix_code,
    jet.code as entry_type,
    pr.debit_credit,
    coa.key_account,
    coa.description
FROM posting_rules pr
JOIN journal_entry_types jet ON pr.journal_entry_type_id = jet.id
LEFT JOIN chart_of_accounts coa ON pr.chart_of_account_id = coa.id
WHERE pr.prefix_code LIKE '%DP';

-- Check generated journal entries
SELECT
    je.analysis_code1 as receipt_no,
    je.transaction_date,
    coa.key_account,
    coa.description,
    je.debit,
    je.credit,
    je.description as entry_description
FROM journal_entries je
JOIN journal_batches jb ON je.journal_batch_id = jb.id
LEFT JOIN chart_of_accounts coa ON je.chart_of_account_id = coa.id
WHERE je.analysis_code1 IN (
    SELECT receipt_number FROM customer_deposits
)
ORDER BY je.analysis_code1, je.debit DESC;
```

## Notes
- Voucher payment method (ID: 4) has been removed from the system
- GL Account payment method provides maximum flexibility for unusual transactions
- All deposits must balance (one debit, one credit of equal amounts)
- Customer deposit liability is cleared when receipt settlements are processed

## Company 9876 Branch Reference
For setting up posting rules in Company 9876 (New Testing Co.), use the following branch prefixes:

| Branch ID | Branch Name | Prefix | Deposit Rule Prefix |
|-----------|-------------|--------|-------------------|
| 12 | Malir Cant. Branch | MC | MCDP |
| 13 | Sialkot | BS | BSDP |
| 14 | UMRAH | UB | UBDP |
| 22 | TESTING BRANCH | TT | TTDP |
| 24 | TESTING BRANCH PK | TP | TPDP |
| 34 | Testing Tourism and services | ET | ETDP |
| 35 | Waseem branch | WB | WBDP |
| 45 | Karachi Clifton Branch | KC | KCDP |
| 46 | Sialkot Branch | SK | SKDP |

All branches should use:
- **Debit Account (CASH)**: 185001 (Cash Box) or appropriate bank account
- **Credit Account (DEPO)**: 245100 (Customer Deposit / Overpayment)

Chart of accounts created by user **wajahat** (user_id: 54) for Company 9876.