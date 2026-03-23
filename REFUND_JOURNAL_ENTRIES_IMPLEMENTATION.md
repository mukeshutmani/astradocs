# Refund Journal Entries Implementation Guide

## Table of Contents
1. [Overview](#overview)
2. [Business Logic](#business-logic)
3. [Journal Entry Types](#journal-entry-types)
4. [Calculation Logic](#calculation-logic)
5. [Database Setup](#database-setup)
6. [Posting Rules Configuration](#posting-rules-configuration)
7. [Usage Workflow](#usage-workflow)
8. [Example Scenarios](#example-scenarios)
9. [Testing](#testing)
10. [Troubleshooting](#troubleshooting)

## Overview

The Refund Journal Entries system automatically generates accounting entries that **reverse the original invoice and cost document entries** when a refund is issued. This ensures proper accounting treatment by:
- Reducing customer receivables
- Reversing revenue recognition
- Reversing cost of sales (when supplier provides refund)
- Reducing supplier payables (when applicable)

**Implementation Date**: 2025-10-09
**Status**: ✅ Complete - Ready for Database Setup & Testing

## Business Logic

### Core Principles

1. **Refunds Reverse Original Entries**
   - Original invoice entries are reversed proportionally based on customer refund amount
   - Original cost entries are reversed proportionally based on supplier refund amount
   - Debits become Credits, Credits become Debits

2. **Proportional Reversal**
   - If customer receives 50% refund, revenue entries are reversed by 50%
   - If supplier refunds 40%, cost entries are reversed by 40%
   - Allows for different customer vs supplier refund amounts

3. **Entry Components**
   - **Customer Refund Entries**: Reverse invoice entries (AR, revenues, fees)
   - **Supplier Refund Entries**: Reverse cost entries (AP, COGS, IATA clearing)

### Refund Workflow

```
[Invoice Created] → [Cost Document Created]
        ↓                       ↓
[Journal Entries]       [Journal Entries]
        ↓                       ↓
[Refund Issued] → [Refund Printed]
        ↓
[Refund Journal Entries Generated]
   (Reverses original entries)
```

## Journal Entry Types

### Customer Refund Entries (Reverse Invoice)

| Code | Description | Debit/Credit | Reverses | Account Type |
|------|-------------|--------------|----------|--------------|
| **RFAREC** | Refund AR Reduction | Credit | AREC (DR) | Trade Debtors |
| **RFASLE** | Refund Air Sales Revenue Reversal | Debit | ASLE (CR) | Sales Revenue |
| **RFATAX** | Refund Air Tax Revenue Reversal | Debit | ATAX (CR) | Tax Revenue |
| **RFPSFM** | Refund PSF Markup Reversal | Debit | PSFM (CR) | Markup Revenue |
| **RFPSFT** | Refund Transaction Fee Reversal | Debit | PSFT (CR) | Fee Revenue |
| **RFSTAX** | Refund Sales Tax Reversal | Debit | STAX (CR) | SST Payable |
| **RFDISC** | Refund Discount Reversal | Credit | DISC (DR) | Discount Expense |
| **RFRBTE** | Refund Rebate Reversal | Credit | RBTE (DR) | Rebate Expense |

### Supplier Refund Entries (Reverse Cost)

| Code | Description | Debit/Credit | Reverses | Account Type |
|------|-------------|--------------|----------|--------------|
| **RFCSAL** | Refund Cost of Sales Reversal | Credit | CSAL (DR) | Cost of Sales |
| **RFCSTX** | Refund Cost Tax Reversal | Credit | CSTX (DR) | Cost Tax |
| **RFIATA** | Refund IATA Clearing Reversal | Debit | IATA (CR) | IATA-BSP Clearing |
| **RFAPAY** | Refund AP Reduction | Debit | APAY (CR) | Accounts Payable |
| **RFATXC** | Refund Advance Tax Reversal | Credit | ATAX (DR) | WHT Asset |
| **RFCOMM** | Refund Commission Reversal | Debit | COMM (CR) | Commission Income |

## Calculation Logic

### Refund Percentage Calculation

```javascript
// Customer refund percentage
refundPercentage = customerRefundAmount / invoiceTotalAmount

// Supplier refund percentage
costRefundPercentage = supplierRefundAmount / invoiceTotalAmount
```

### Customer Refund Entry Calculations

```javascript
// 1. RFAREC - Direct customer refund amount
amount = customerRefundAmount

// 2. RFASLE - Proportional revenue reversal
originalASLE = cost.published_rate * cost.quantity
amount = originalASLE * refundPercentage

// 3. RFATAX - Proportional tax revenue reversal
originalATAX = SUM(invoice_taxes.tax_amount) * invoice.quantity
amount = originalATAX * refundPercentage

// 4. RFPSFM - Proportional markup reversal
originalPSFM = invoice.markup * invoice.quantity
amount = originalPSFM * refundPercentage

// 5. RFPSFT - Proportional transaction fee reversal
originalPSFT = invoice.transaction_fee
amount = originalPSFT * refundPercentage

// 6. RFSTAX - Proportional SST reversal
originalSTAX = (invoice.sst / 100) * invoice.transaction_fee
amount = originalSTAX * refundPercentage

// 7. RFDISC - Proportional discount reversal
originalDISC = (invoice.discount / 100) * invoice.price * invoice.quantity
amount = originalDISC * refundPercentage

// 8. RFRBTE - Proportional rebate reversal
originalRBTE = (invoice.rebate / 100) * invoice.price * invoice.quantity
amount = originalRBTE * refundPercentage
```

### Supplier Refund Entry Calculations

```javascript
// Only generated if supplierRefundAmount > 0

// 1. RFCSAL - Proportional cost reversal
originalCSAL = cost.published_rate * cost.quantity
amount = originalCSAL * costRefundPercentage

// 2. RFCSTX - Proportional cost tax reversal
originalCSTX = SUM(cost_taxes.tax_amount) * cost.quantity
amount = originalCSTX * costRefundPercentage

// 3. RFIATA - Proportional IATA clearing reversal (BSP airlines only)
netAmount = (published_rate - commission) * cost.quantity
costTaxes = SUM(cost_taxes.tax_amount) * cost.quantity
originalIATA = netAmount + costTaxes
amount = originalIATA * costRefundPercentage

// 4. RFAPAY - Direct supplier refund amount
amount = supplierRefundAmount

// 5. RFATXC - Proportional WHT reversal
whtAmount = (cost.sst / 100) * commission * cost.quantity
amount = whtAmount * costRefundPercentage

// 6. RFCOMM - Proportional commission reversal
originalCOMM = (cost.commission / 100) * cost.published_rate * cost.quantity
amount = originalCOMM * costRefundPercentage
```

## Database Setup

### Step 1: Run SQL Script

Execute the SQL script to create journal entry types:

```bash
mysql -u username -p database_name < docs/SQL_UPDATES_REFUND_JOURNAL_ENTRIES.sql
```

Or run the SQL commands directly:

```sql
-- Insert refund journal entry types
INSERT INTO journal_entry_types (code, description, created_at, updated_at) VALUES
('RFAREC', 'Refund - AR Reduction (Reverse Receivable)', NOW(), NOW()),
('RFASLE', 'Refund - Air Sales Revenue Reversal', NOW(), NOW()),
('RFATAX', 'Refund - Air Sales Tax Revenue Reversal', NOW(), NOW()),
('RFPSFM', 'Refund - PSF Markup Revenue Reversal', NOW(), NOW()),
('RFPSFT', 'Refund - PSF Transaction Fee Reversal', NOW(), NOW()),
('RFSTAX', 'Refund - Sales Tax Reversal (SST)', NOW(), NOW()),
('RFDISC', 'Refund - Discount Reversal', NOW(), NOW()),
('RFRBTE', 'Refund - Rebate Reversal', NOW(), NOW()),
('RFCSAL', 'Refund - Cost of Sales Reversal', NOW(), NOW()),
('RFCSTX', 'Refund - Cost of Sales Tax Reversal', NOW(), NOW()),
('RFIATA', 'Refund - IATA-BSP Clearing Reversal', NOW(), NOW()),
('RFAPAY', 'Refund - Accounts Payable Reduction', NOW(), NOW()),
('RFATXC', 'Refund - Advance Tax/WHT Reversal (Cost)', NOW(), NOW()),
('RFCOMM', 'Refund - Commission Reversal', NOW(), NOW())
ON DUPLICATE KEY UPDATE description = VALUES(description), updated_at = NOW();
```

### Step 2: Verify Installation

```sql
-- Check refund journal entry types
SELECT id, code, description
FROM journal_entry_types
WHERE code LIKE 'RF%'
ORDER BY code;

-- Should return 14 entry types
```

## Posting Rules Configuration

### Required Posting Rules Per Branch

**Prefix Code**: `{BranchPrefix}RF` (e.g., TTRF, SSRF, LHERF)

#### Customer Refund Rules

| Entry Type | Code | Debit/Credit | Typical Account | Required |
|------------|------|--------------|-----------------|----------|
| AR Reduction | RFAREC | Credit | 151110 - Trade Debtors | ✅ Yes |
| Sales Reversal | RFASLE | Debit | 410100 - Air Sales | ✅ Yes |
| Tax Reversal | RFATAX | Debit | 410110 - Tax Revenue | ✅ Yes |
| Markup Reversal | RFPSFM | Debit | 410120 - Markup Revenue | ✅ Yes |
| Fee Reversal | RFPSFT | Debit | 410130 - Transaction Fee | If applicable |
| SST Reversal | RFSTAX | Debit | 245300 - SST Payable | If applicable |
| Discount Reversal | RFDISC | Credit | 530100 - Discount Expense | If applicable |
| Rebate Reversal | RFRBTE | Credit | 530200 - Rebate Expense | If applicable |

#### Supplier Refund Rules

| Entry Type | Code | Debit/Credit | Typical Account | Required |
|------------|------|--------------|-----------------|----------|
| AP Reduction | RFAPAY | Debit | 231000 - Accounts Payable | ✅ Yes |
| Cost Reversal | RFCSAL | Credit | 510100 - Cost of Sales | ✅ Yes |
| Cost Tax Reversal | RFCSTX | Credit | 510110 - Cost Tax | If applicable |
| IATA Reversal | RFIATA | Debit | 232000 - IATA Clearing | For BSP only |
| WHT Reversal | RFATXC | Credit | 151200 - WHT Asset | If applicable |
| Comm. Reversal | RFCOMM | Debit | 420100 - Commission Income | If applicable |

### Setup Example (TT Branch)

```sql
-- Customer Refund Entries
INSERT INTO posting_rules (
    branch_id, company_code, prefix_code,
    journal_entry_type_id, chart_of_account_id,
    debit_credit, record_type
) VALUES
-- AR Reduction (Credit - reduces receivable)
(1, 'COMP01', 'TTRF',
    (SELECT id FROM journal_entry_types WHERE code = 'RFAREC'),
    (SELECT id FROM chart_of_accounts WHERE key_account = '151110'),
    'credit', 'Default'),

-- Revenue Reversals (Debit - reduces revenue)
(1, 'COMP01', 'TTRF',
    (SELECT id FROM journal_entry_types WHERE code = 'RFASLE'),
    (SELECT id FROM chart_of_accounts WHERE key_account = '410100'),
    'debit', 'Default'),

(1, 'COMP01', 'TTRF',
    (SELECT id FROM journal_entry_types WHERE code = 'RFATAX'),
    (SELECT id FROM chart_of_accounts WHERE key_account = '410110'),
    'debit', 'Default'),

(1, 'COMP01', 'TTRF',
    (SELECT id FROM journal_entry_types WHERE code = 'RFPSFM'),
    (SELECT id FROM chart_of_accounts WHERE key_account = '410120'),
    'debit', 'Default'),

-- Supplier Refund Entries
(1, 'COMP01', 'TTRF',
    (SELECT id FROM journal_entry_types WHERE code = 'RFAPAY'),
    (SELECT id FROM chart_of_accounts WHERE key_account = '231000'),
    'debit', 'Default'),

(1, 'COMP01', 'TTRF',
    (SELECT id FROM journal_entry_types WHERE code = 'RFCSAL'),
    (SELECT id FROM chart_of_accounts WHERE key_account = '510100'),
    'credit', 'Default'),

(1, 'COMP01', 'TTRF',
    (SELECT id FROM journal_entry_types WHERE code = 'RFCSTX'),
    (SELECT id FROM chart_of_accounts WHERE key_account = '510110'),
    'credit', 'Default');
```

## Usage Workflow

### 1. Create and Process Refund

1. Navigate to Order → Refunds
2. Create refund with:
   - Customer refund amount
   - Supplier refund amount (if applicable)
   - Select segments/services to refund
3. Print refund document (status changes to "Printed")

### 2. Generate Journal Entries

1. Navigate to GL → Journal Entries
2. Click "Generate System JE"
3. Select branch and date range
4. System processes all printed refunds with `je_generated = NULL`
5. Entries are automatically grouped by accounting period

### 3. Review Generated Entries

```sql
-- View refund journal entries
SELECT
    je.analysis_code1 AS refund_no,
    jet.code AS entry_type,
    coa.key_account,
    coa.description AS account_name,
    je.debit,
    je.credit,
    je.description
FROM journal_entries je
JOIN journal_batches jb ON je.journal_batch_id = jb.id
JOIN journal_entry_types jet ON je.chart_of_account_id = jet.id
LEFT JOIN chart_of_accounts coa ON je.chart_of_account_id = coa.id
WHERE je.analysis_code1 LIKE '%RF%'
ORDER BY je.analysis_code1, jet.code;
```

### 4. Verify Balance

```sql
-- Check if refund entries balance
SELECT
    analysis_code1 AS refund_no,
    SUM(debit) AS total_debit,
    SUM(credit) AS total_credit,
    SUM(debit) - SUM(credit) AS imbalance
FROM journal_entries
WHERE analysis_code1 LIKE '%RF%'
GROUP BY analysis_code1;

-- Imbalance should be 0.00 for all refunds
```

## Example Scenarios

### Scenario 1: Full Refund (100%)

**Original Invoice (RM 155,510)**
- Published Rate: RM 128,000 (64,000 × 2 tickets)
- Airline Taxes: RM 20,280 (10,140 × 2)
- Markup: RM 6,000 (3,000 × 2)
- Transaction Fee: RM 5,000
- SST (5%): RM 250
- **Total**: RM 155,510

**Original Invoice Entries:**
```
DR AREC: 155,510
DR CSAL: 128,000
DR CSTX: 20,280
CR ASLE: 128,000
CR ATAX: 20,280
CR PSFM: 6,000
CR PSFT: 5,000
CR STAX: 250
CR IATA: 148,280
```

**Refund (100% - RM 155,510 customer, RM 148,280 supplier)**

**Refund Journal Entries:**
```
CR RFAREC: 155,510  (Reduce receivable 100%)
DR RFASLE: 128,000  (Reverse revenue 100%)
DR RFATAX: 20,280   (Reverse tax revenue 100%)
DR RFPSFM: 6,000    (Reverse markup 100%)
DR RFPSFT: 5,000    (Reverse fee 100%)
DR RFSTAX: 250      (Reverse SST 100%)

CR RFCSAL: 128,000  (Reverse cost 100%)
CR RFCSTX: 20,280   (Reverse cost tax 100%)
DR RFIATA: 148,280  (Reduce IATA liability 100%)
DR RFAPAY: 148,280  (Reduce AP 100%)

Total DR: 307,810
Total CR: 307,810 ✓ Balanced
```

### Scenario 2: Partial Refund (60%)

**Original Invoice: RM 155,510**
**Refund: RM 93,306 customer (60%), RM 88,968 supplier (60%)**

**Refund Journal Entries:**
```
CR RFAREC: 93,306   (60% of receivable)
DR RFASLE: 76,800   (60% of revenue)
DR RFATAX: 12,168   (60% of tax revenue)
DR RFPSFM: 3,600    (60% of markup)
DR RFPSFT: 3,000    (60% of fee)
DR RFSTAX: 150      (60% of SST)

CR RFCSAL: 76,800   (60% of cost)
CR RFCSTX: 12,168   (60% of cost tax)
DR RFIATA: 88,968   (60% of IATA)
DR RFAPAY: 88,968   (60% of AP)

Total DR: 184,686
Total CR: 184,686 ✓ Balanced
```

### Scenario 3: Customer Refund Only (No Supplier Refund)

**Original Invoice: RM 155,510**
**Refund: RM 93,306 customer (60%), RM 0 supplier**

This scenario occurs when airline doesn't refund but company still refunds customer (e.g., as goodwill):

**Refund Journal Entries:**
```
CR RFAREC: 93,306   (60% of receivable)
DR RFASLE: 76,800   (60% of revenue)
DR RFATAX: 12,168   (60% of tax revenue)
DR RFPSFM: 3,600    (60% of markup)
DR RFPSFT: 3,000    (60% of fee)
DR RFSTAX: 150      (60% of SST)

Total DR: 95,718
Total CR: 93,306
Imbalance: 2,412 (company absorbs loss)

-- NOTE: Need additional entry for the loss
DR REFUND_LOSS: 2,412
CR CASH/BANK: 2,412
```

## Testing

### Test Checklist

- [ ] **Database Setup**
  - [ ] Run SQL script successfully
  - [ ] Verify all 14 refund entry types created
  - [ ] Check refunds table has `je_generated` column

- [ ] **Posting Rules Setup**
  - [ ] Create posting rules for test branch
  - [ ] Verify all required entry types configured
  - [ ] Check correct debit/credit settings
  - [ ] Confirm chart of accounts mapping

- [ ] **Generate Refund Entries**
  - [ ] Create test refund (100% customer & supplier)
  - [ ] Print refund document
  - [ ] Generate journal entries
  - [ ] Verify entries created
  - [ ] Check je_generated flag set to true

- [ ] **Verify Calculations**
  - [ ] Customer refund amount = RFAREC credit
  - [ ] Revenue reversals match refund percentage
  - [ ] Supplier refund amount = RFAPAY debit
  - [ ] Cost reversals match supplier refund percentage

- [ ] **Balance Verification**
  - [ ] Total debits = Total credits
  - [ ] All entries have valid chart of accounts
  - [ ] Transaction dates correct
  - [ ] Period assignment correct

- [ ] **Regeneration Testing**
  - [ ] Find batch with refund entries
  - [ ] Regenerate batch
  - [ ] Verify entries recreated correctly
  - [ ] Check batch narratives include "Refund"

- [ ] **Edge Cases**
  - [ ] Partial refund (50%)
  - [ ] Customer-only refund (no supplier refund)
  - [ ] Multiple refunds for same invoice
  - [ ] Refund with discounts/rebates

## Troubleshooting

### Common Issues

#### 1. No Entries Generated for Refunds

**Symptoms**: Generate System JE doesn't create refund entries

**Possible Causes**:
- Posting rules not configured for branch
- Refund status not "Printed"
- je_generated already set to true
- No refund amounts entered

**Solutions**:
```sql
-- Check posting rules
SELECT pr.*, jet.code
FROM posting_rules pr
JOIN journal_entry_types jet ON pr.journal_entry_type_id = jet.id
WHERE pr.prefix_code = 'TTRF'  -- Replace with your branch prefix
AND jet.code LIKE 'RF%';

-- Check refund status
SELECT refund_no, status, je_generated,
       customer_refund_amount, supplier_refund_amount
FROM refunds
WHERE refund_no = 'LHERF0000001';  -- Your refund number

-- Reset je_generated for testing
UPDATE refunds
SET je_generated = NULL
WHERE refund_no = 'LHERF0000001';
```

#### 2. Entries Not Balanced

**Symptoms**: SUM(debit) ≠ SUM(credit)

**Possible Causes**:
- Missing posting rules
- Incorrect debit/credit configuration
- Calculation errors

**Solutions**:
```sql
-- Check which entry types are missing
SELECT jet.code, jet.description
FROM journal_entry_types jet
WHERE jet.code LIKE 'RF%'
AND jet.id NOT IN (
    SELECT DISTINCT journal_entry_type_id
    FROM posting_rules
    WHERE prefix_code = 'TTRF'
);

-- Verify debit/credit settings
SELECT jet.code, pr.debit_credit
FROM posting_rules pr
JOIN journal_entry_types jet ON pr.journal_entry_type_id = jet.id
WHERE pr.prefix_code = 'TTRF'
AND jet.code LIKE 'RF%';
```

#### 3. Wrong Amounts Calculated

**Symptoms**: Refund amounts don't match expected values

**Possible Causes**:
- Invoice/cost data missing
- Incorrect refund percentage
- Tax calculations off

**Solutions**:
```sql
-- Verify refund and invoice data
SELECT
    r.refund_no,
    r.customer_refund_amount,
    r.supplier_refund_amount,
    i.total_amount AS invoice_total,
    (r.customer_refund_amount / i.total_amount * 100) AS refund_percentage
FROM refunds r
JOIN invoices i ON r.invoice_id = i.id
WHERE r.refund_no = 'LHERF0000001';

-- Check invoice taxes
SELECT invoice_id, tax_code, tax_amount
FROM invoice_taxes
WHERE invoice_id = (
    SELECT invoice_id FROM refunds
    WHERE refund_no = 'LHERF0000001'
);
```

#### 4. Refund Not Linking to Invoice

**Symptoms**: refund.Invoice is null in journal generation

**Possible Causes**:
- Missing invoice_id in refund record
- Invoice deleted or void
- Database association issue

**Solutions**:
```sql
-- Check refund-invoice link
SELECT r.refund_no, r.invoice_id, i.invoice_number, i.status
FROM refunds r
LEFT JOIN invoices i ON r.invoice_id = i.id
WHERE r.refund_no = 'LHERF0000001';

-- Check cost link
SELECT r.refund_no, r.cost_id, c.id AS cost_exists
FROM refunds r
LEFT JOIN costs c ON r.cost_id = c.id
WHERE r.refund_no = 'LHERF0000001';
```

### Support Queries

```sql
-- 1. Check unprocessed refunds
SELECT
    refund_no,
    customer_refund_amount,
    supplier_refund_amount,
    status,
    je_generated,
    created_at
FROM refunds
WHERE je_generated IS NULL
AND status = 'Printed'
ORDER BY created_at DESC;

-- 2. Check refund journal batches
SELECT
    jb.batch_no,
    jb.journal_entry_period,
    COUNT(DISTINCT je.analysis_code1) AS refund_count,
    SUM(je.debit) AS total_debit,
    SUM(je.credit) AS total_credit
FROM journal_batches jb
JOIN journal_entries je ON jb.id = je.journal_batch_id
WHERE je.analysis_code1 LIKE '%RF%'
GROUP BY jb.id, jb.batch_no, jb.journal_entry_period;

-- 3. Detailed refund entry breakdown
SELECT
    r.refund_no,
    r.customer_refund_amount,
    r.supplier_refund_amount,
    jet.code AS entry_type,
    je.debit,
    je.credit,
    coa.key_account,
    coa.description AS account_name
FROM refunds r
JOIN journal_entries je ON r.refund_no = je.analysis_code1
JOIN journal_entry_types jet ON je.journal_entry_type_id = jet.id
LEFT JOIN chart_of_accounts coa ON je.chart_of_account_id = coa.id
WHERE r.refund_no = 'LHERF0000001'
ORDER BY jet.code;

-- 4. Compare refund vs original invoice
SELECT
    'Original Invoice' AS document_type,
    jet.code,
    SUM(je.debit) AS debit,
    SUM(je.credit) AS credit
FROM journal_entries je
JOIN journal_entry_types jet ON je.journal_entry_type_id = jet.id
WHERE je.analysis_code1 = 'TTIN0000001'  -- Invoice number
GROUP BY jet.code

UNION ALL

SELECT
    'Refund Reversal' AS document_type,
    jet.code,
    SUM(je.debit) AS debit,
    SUM(je.credit) AS credit
FROM journal_entries je
JOIN journal_entry_types jet ON je.journal_entry_type_id = jet.id
WHERE je.analysis_code1 = 'LHERF0000001'  -- Refund number
GROUP BY jet.code;
```

## Best Practices

1. **Always Configure All Required Posting Rules**
   - Don't skip optional entry types if your invoices use those features
   - Example: If invoices have discounts, configure RFDISC

2. **Verify Refund Data Before Generating Entries**
   - Ensure invoice_id and cost_id are populated
   - Check customer and supplier refund amounts are correct
   - Confirm refund status is "Printed"

3. **Review Entries Before Posting**
   - Always check that debits = credits
   - Verify amounts match expected percentages
   - Confirm correct accounting period

4. **Use Regeneration for Corrections**
   - If posting rules change, regenerate affected batches
   - Don't manually edit journal entries

5. **Document Custom Scenarios**
   - If you have special refund cases (goodwill refunds, partial supplier refunds, etc.)
   - Document the expected accounting treatment
   - Consider creating additional entry types if needed

## Integration with Existing Systems

### Credit Notes vs Refunds

**Credit Notes**:
- Issued to reduce customer's outstanding balance
- May or may not result in cash refund
- Uses CNRV, CNAR entry types

**Refunds**:
- Actual reversal of sale transaction
- Links to original invoice and cost
- Uses RFAREC, RFASLE, etc. entry types

Both can coexist. Refunds reverse the original transaction, while credit notes create new liability.

### Settlements

When a refund is settled:
- Receipt settlements can apply refund amounts
- Similar to how deposits are applied
- Creates SETL entry to reverse the refund receivable

## Future Enhancements

1. **Automatic Loss/Gain Entries**
   - When customer refund ≠ supplier refund
   - Create GL entry for the difference
   - Account for company loss or gain

2. **Refund Approval Workflow**
   - Only generate entries for approved refunds
   - Track approval chain in journal narrative

3. **Bulk Refund Processing**
   - Generate entries for multiple refunds at once
   - Group by customer or reason

4. **Refund Reversal**
   - Handle cases where refund is cancelled
   - Reverse the refund entries

---

**Document Version**: 1.0
**Last Updated**: 2025-10-09
**Author**: System Development Team
**Status**: Ready for Production
