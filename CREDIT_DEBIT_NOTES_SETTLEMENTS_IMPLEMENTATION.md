# Credit Notes, Debit Notes & Enhanced Settlements Implementation

## Overview
This document outlines the implementation of journal entries for Credit Notes, Debit Notes, and enhanced settlement processing with multiple payment methods. This extends the existing journal entry system documented in JOURNAL_ENTRIES_PLAN.md.

## Database Schema Changes Required

### 1. Add je_generated Column
Both `credit_notes` and `debit_notes` tables need the je_generated tracking column:

```sql
-- Credit Notes
ALTER TABLE credit_notes
ADD COLUMN je_generated TINYINT(1) DEFAULT NULL AFTER updated_at,
ADD INDEX idx_je_generated (je_generated);

-- Debit Notes
ALTER TABLE debit_notes
ADD COLUMN je_generated TINYINT(1) DEFAULT NULL AFTER updated_at,
ADD INDEX idx_je_generated (je_generated);
```

### 2. Add New Journal Entry Types

```sql
-- Credit Note Entry Types (prefix: CN)
INSERT INTO journal_entry_types (code, description) VALUES
('CNAR', 'Credit Note - Accounts Receivable Reversal'),
('CNRV', 'Credit Note - Revenue Reversal'),
('CNCS', 'Credit Note - Cost of Sales Reversal'),
('CNIA', 'Credit Note - IATA Reversal'),
('CNRF', 'Credit Note - Refund Liability');

-- Debit Note Entry Types (prefix: DN)
INSERT INTO journal_entry_types (code, description) VALUES
('DNAP', 'Debit Note - Accounts Payable Adjustment'),
('DNEX', 'Debit Note - Additional Expense'),
('DNIA', 'Debit Note - IATA Adjustment'),
('DNRF', 'Debit Note - Refund Recovery');

-- Payment Method Entry Types
INSERT INTO journal_entry_types (code, description) VALUES
('PMCASH', 'Payment Method - Cash'),
('PMCHQ', 'Payment Method - Cheque'),
('PMCC', 'Payment Method - Credit Card'),
('PMDC', 'Payment Method - Debit Card'),
('PMBT', 'Payment Method - Bank Transfer'),
('PMGL', 'Payment Method - GL Account');
```

## Credit Note Journal Entries (CN)

### Business Logic
Credit notes reduce customer receivables and reverse revenue. They may be issued for:
- Refunds/cancellations
- Price adjustments
- Service corrections
- Goodwill gestures

### Document Type: Credit Note (CN)
**Prefix**: `${branch_prefix}CN` (e.g., TTCN)

### Entry Types

#### Customer Credit Notes (Issued to customers)

| Entry Code | Description | DR/CR | Calculation | Account |
|------------|-------------|-------|-------------|---------|
| **CNRV** | Revenue Reversal | Debit | credit_note.billing_amount | 410100 - Air Sales Revenue |
| **CNAR** | AR Reduction | Credit | credit_note.billing_amount | 151110 - Trade Debtors |

#### For Refundable Credit Notes

| Entry Code | Description | DR/CR | Calculation | Account |
|------------|-------------|-------|-------------|---------|
| **CNRF** | Refund Liability | Debit | credit_note.refund_amount | 245200 - Customer Refunds Payable |
| **CASH** | Cash/Bank Payment | Credit | credit_note.refund_amount | Based on payment method |

### Example Credit Note Entry
```
Customer requests refund for cancelled ticket:
Original Invoice: 50,000
Credit Note Amount: 50,000
Less Handling Fee: 2,000
Refund Amount: 48,000

Journal Entries:
DR CNRV (Revenue Reversal): 50,000
CR CNAR (AR Reduction): 50,000

DR CNRF (Refund Liability): 48,000
DR HAND (Handling Fee Revenue): 2,000
CR CASH (Bank Payment): 50,000
```

## Debit Note Journal Entries (DN)

### Business Logic
Debit notes increase amounts owed by suppliers or adjust payables. They may be issued for:
- Supplier overcharges
- Service failures
- Price corrections
- Penalty charges

### Document Type: Debit Note (DN)
**Prefix**: `${branch_prefix}DN` (e.g., TTDN)

### Entry Types

#### Supplier Debit Notes (Issued to suppliers)

| Entry Code | Description | DR/CR | Calculation | Account |
|------------|-------------|-------|-------------|---------|
| **DNAP** | AP Reduction | Debit | debit_note.billing_amount | 231000 - Accounts Payable |
| **DNEX** | Expense/Recovery | Credit | debit_note.billing_amount | Varies by type |

### Example Debit Note Entry
```
Supplier error requires adjustment:
Original Cost: 100,000
Debit Note: 10,000 (overcharge)

Journal Entries:
DR DNAP (AP Reduction): 10,000
CR DNEX (Expense Recovery): 10,000
```

## Enhanced Receipt Settlements

### Current Implementation Issues
The current `settlementPostingRules` function has limitations:
1. Doesn't handle credit notes properly
2. Doesn't track payment methods used
3. Uses hardcoded accounts instead of posting rules

### Enhanced Implementation

#### Entry Types for Receipt Settlements (RS)

| Entry Code | Description | DR/CR | Calculation | Account |
|------------|-------------|-------|-------------|---------|
| **SETL** | Settlement Application | Debit | Applied from deposits | 245100 - Customer Deposits |
| **AREC** | AR Clearance | Credit | Total invoices settled | 151110 - Trade Debtors |
| **CNAR** | CN Application | Debit | Credit notes applied | 151110 - Trade Debtors |
| **PMCASH** | Cash Payment | Debit | Cash received | Based on payment method |
| **PMCHQ** | Cheque Payment | Debit | Cheque received | Based on bank account |
| **PMCC** | Card Payment | Debit | Card payment | Based on merchant account |

### Settlement Components
A receipt settlement can include:
1. **Customer Deposits** - Previously received deposits applied
2. **Credit Notes** - Credit notes offset against invoices
3. **Direct Payments** - New payments received (cash, cheque, card, etc.)
4. **Invoices** - Outstanding invoices being settled

### Example Receipt Settlement
```
Settlement for invoice 100,000:
- Apply Deposit: 40,000
- Apply Credit Note: 10,000
- Cash Payment: 30,000
- Card Payment: 20,000

Journal Entries:
DR SETL (Deposit Application): 40,000
DR CNAR (Credit Note Application): 10,000
DR PMCASH (Cash Received): 30,000
DR PMCC (Card Received): 20,000
CR AREC (Invoice Clearance): 100,000
```

## Enhanced Payment Settlements

### Current Implementation Issues
Current `paymentPostingRules` function:
1. Only handles single payment method per settlement
2. Doesn't track individual payment details
3. Limited support for multiple cost allocations

### Enhanced Implementation

#### Entry Types for Payment Settlements (PS)

| Entry Code | Description | DR/CR | Calculation | Account |
|------------|-------------|-------|-------------|---------|
| **APAY** | AP Clearance | Debit | Total costs paid | 231000 - Accounts Payable |
| **DNAP** | DN Application | Credit | Debit notes applied | 231000 - Accounts Payable |
| **PMCASH** | Cash Payment | Credit | Cash paid | Based on payment method |
| **PMCHQ** | Cheque Payment | Credit | Cheque issued | Based on bank account |
| **PMBT** | Bank Transfer | Credit | Transfer made | Based on bank account |

### Payment Components
A payment settlement can include:
1. **Cost Documents** - Exchange orders being paid
2. **Debit Notes** - Debit notes offset against payables
3. **Payment Methods** - Multiple payment methods used

### Example Payment Settlement
```
Settlement for supplier invoice 200,000:
- Apply Debit Note: 20,000
- Cheque Payment: 100,000
- Bank Transfer: 80,000

Journal Entries:
DR APAY (AP Clearance): 200,000
CR DNAP (Debit Note Applied): 20,000
CR PMCHQ (Cheque Issued): 100,000
CR PMBT (Bank Transfer): 80,000
```

## Payment Method Handling

### Payment Types (pay_type_id)
1. **Cash** (1) - Use chart_of_account_id from cash box selection
2. **Cheque** (2) - Use bank_account.chart_of_account_id
3. **Credit Card** (3) - Use bank_account.chart_of_account_id (merchant account)
4. **GL Account** (5) - Use gl_account_id directly
5. **Bank Transfer** (6) - Use bank_account.chart_of_account_id
6. **Debit Card** (7) - Use bank_account.chart_of_account_id

### Payment Method Account Resolution
```javascript
function getPaymentMethodAccount(payment) {
    // GL Account type - use gl_account_id directly
    if (payment.pay_type_id === 5 && payment.gl_account_id) {
        return payment.gl_account_id;
    }

    // All other types use chart_of_account_id
    // This is set based on payment method:
    // - Cash: Selected cash box account
    // - Bank methods: Bank account's chart_of_account_id
    return payment.chart_of_account_id;
}
```

## Implementation Checklist

### Database Changes
- [ ] Add je_generated to credit_notes table
- [ ] Add je_generated to debit_notes table
- [ ] Insert new journal entry types for CN
- [ ] Insert new journal entry types for DN
- [ ] Insert payment method entry types

### Code Implementation
- [ ] Create creditNotePostingRules function
- [ ] Create debitNotePostingRules function
- [ ] Enhance settlementPostingRules for credit notes
- [ ] Enhance settlementPostingRules for payment methods
- [ ] Enhance paymentPostingRules for debit notes
- [ ] Enhance paymentPostingRules for multiple payments
- [ ] Create regenerateCreditNoteEntries function
- [ ] Create regenerateDebitNoteEntries function
- [ ] Update regenerateSettlementEntries function
- [ ] Update regeneratePaymentEntries function
- [ ] Add CN/DN processing to generateJournal
- [ ] Update je_generated flags for CN/DN

### Posting Rules Setup
- [ ] Create posting rules for CN entry types
- [ ] Create posting rules for DN entry types
- [ ] Create posting rules for payment methods
- [ ] Document posting rules configuration

### Testing
- [ ] Test credit note journal generation
- [ ] Test debit note journal generation
- [ ] Test settlement with credit notes
- [ ] Test settlement with multiple payments
- [ ] Test payment with debit notes
- [ ] Test regeneration for CN/DN
- [ ] Verify all entries balance
- [ ] Test period assignment for CN/DN

## Posting Rules Configuration

### Credit Note Rules (Prefix: TTCN)

| Entry Type | Code | DR/CR | Account | Description |
|------------|------|-------|---------|-------------|
| Revenue Reversal | CNRV | Debit | 410100 | Reverse air sales revenue |
| AR Reduction | CNAR | Credit | 151110 | Reduce trade debtors |
| Refund Liability | CNRF | Debit | 245200 | Create refund liability |
| Cash Payment | CASH | Credit | varies | Bank/cash account |

### Debit Note Rules (Prefix: TTDN)

| Entry Type | Code | DR/CR | Account | Description |
|------------|------|-------|---------|-------------|
| AP Reduction | DNAP | Debit | 231000 | Reduce accounts payable |
| Expense Recovery | DNEX | Credit | varies | Recovery income account |

### Settlement Rules (Enhanced)

| Entry Type | Code | DR/CR | Account | Description |
|------------|------|-------|---------|-------------|
| Deposit Application | SETL | Debit | 245100 | Apply customer deposit |
| Invoice Clearance | AREC | Credit | 151110 | Clear receivables |
| Credit Note Application | CNAR | Debit | 151110 | Apply credit note |
| Cash Received | PMCASH | Debit | varies | Cash receipt |
| Cheque Received | PMCHQ | Debit | varies | Cheque receipt |
| Card Received | PMCC | Debit | varies | Card receipt |

## SQL Queries for Support

```sql
-- Check unprocessed credit notes
SELECT COUNT(*) FROM credit_notes
WHERE je_generated IS NULL
AND doc_status = 'Printed';

-- Check unprocessed debit notes
SELECT COUNT(*) FROM debit_notes
WHERE je_generated IS NULL
AND doc_status NOT IN ('Void');

-- Verify credit note batch balance
SELECT
    batch_no,
    SUM(CASE WHEN description LIKE '%CNRV%' THEN debit ELSE 0 END) as revenue_reversal,
    SUM(CASE WHEN description LIKE '%CNAR%' THEN credit ELSE 0 END) as ar_reduction,
    SUM(debit) - SUM(credit) as imbalance
FROM journal_entries je
JOIN journal_batches jb ON je.journal_batch_id = jb.id
WHERE jb.batch_no LIKE '%CN%'
GROUP BY jb.id, jb.batch_no;

-- Check settlement with credit notes
SELECT
    rs.receipt_number,
    rs.amount as settlement_amount,
    COUNT(DISTINCT rsi.invoice_id) as invoices_count,
    COUNT(DISTINCT rscn.credit_note_id) as credit_notes_count,
    SUM(rscn.amount) as cn_amount_applied
FROM receipt_settlements rs
LEFT JOIN receipt_settlement_invoices rsi ON rs.id = rsi.receipt_settlement_id
LEFT JOIN receipt_settlement_credit_notes rscn ON rs.id = rscn.receipt_settlement_id
WHERE rs.je_generated IS NULL
GROUP BY rs.id;

-- Check payment methods in settlements
SELECT
    pay_type_id,
    COUNT(*) as count,
    SUM(amount) as total_amount
FROM receipt_settlement_payments
WHERE receipt_settlement_id IS NOT NULL
GROUP BY pay_type_id;
```

## Notes

### Credit Note vs Refund
- **Credit Note**: Reduces AR, may or may not result in cash refund
- **Refund**: Actual cash/bank payment to customer
- Credit notes can be:
  1. Applied to future invoices (no cash movement)
  2. Refunded immediately (creates refund liability + cash payment)
  3. Applied in settlement with other payments

### Debit Note vs Credit Memo
- **Debit Note**: Issued TO supplier (reduces what we owe them)
- **Credit Note**: Issued TO customer (reduces what they owe us)
- Both reduce liabilities/receivables but in opposite directions

### Settlement Complexity
Settlements can combine multiple payment sources:
- Previous deposits (already recorded, just applying)
- Credit notes (reducing AR without cash)
- New payments (various methods, creating cash/bank entries)
- Each component needs proper journal entry

### Period Assignment
All new entry types follow the same period assignment logic:
- Transaction date determines the period
- Entries grouped by period into separate batches
- Period code format: MMYYYY (e.g., 012025 for January 2025)
