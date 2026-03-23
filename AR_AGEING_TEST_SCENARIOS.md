# AR Ageing Analysis Detail Report - Test Scenarios

## Overview
This document provides test scenarios to verify the corrected AR Ageing Analysis Detail Report calculations.

## Key Fixes Implemented

### 1. Due Date Calculation
- **Fixed:** Now calculates invoice due date based on customer credit terms
- **Formula:** `Due Date = Invoice Date + Credit Terms`
- **Credit Terms Support:**
  - `credit days`: Adds specified number of days
  - `BSP Payment Term`: Adds 7 days
  - `BSP PAYMENT +15 days`: Adds 15 days
  - `Weekly`: Adds 7 days
  - `Statement Date`: End of month + 30 days

### 2. Days Overdue Calculation
- **Fixed:** Calculates from due date, not invoice date
- **Formula:** `Days Overdue = MAX(0, As-Of-Date - Due Date)`
- **Note:** Shows 0 if invoice is not yet due

### 3. Outstanding Amount
- **Fixed:** Uses actual settlement data from `receipt_settlement_invoice` table
- **Formula:** `Outstanding = Invoice Total - Sum(Settled Amounts)`
- **Excludes:** Fully settled invoices (Outstanding <= 0)

### 4. Ageing Buckets
- **Fixed:** Based on days overdue, not invoice age
- **Buckets:**
  - Current: Not yet due (Days Overdue <= 0)
  - 1-30 Days: 1-30 days overdue
  - 31-60 Days: 31-60 days overdue
  - 61-90 Days: 61-90 days overdue
  - 91-120 Days: 91-120 days overdue
  - 120+ Days: Over 120 days overdue

### 5. Deposit Handling
- **Fixed:** Deposits shown as credit reduction, not in ageing buckets
- **Display:** Shows as "Less: Deposit Credit" after subtotal

## Test Scenarios

### Scenario 1: Basic Credit Terms Test
**Setup:**
- Customer: ABC Corp
- Credit Terms: 30 credit days
- Invoice Date: January 1, 2024
- Invoice Amount: $10,000
- As-Of Date: February 15, 2024

**Expected Results:**
- Due Date: January 31, 2024
- Days Overdue: 15 days
- Ageing Bucket: 1-30 Days
- Outstanding in 1-30 Days bucket: $10,000

### Scenario 2: Partially Settled Invoice
**Setup:**
- Customer: XYZ Ltd
- Credit Terms: 15 credit days
- Invoice Date: January 10, 2024
- Invoice Amount: $5,000
- Settled Amount: $3,000
- As-Of Date: March 1, 2024

**Expected Results:**
- Due Date: January 25, 2024
- Days Overdue: 35 days
- Outstanding Amount: $2,000
- Ageing Bucket: 31-60 Days
- Outstanding in 31-60 Days bucket: $2,000

### Scenario 3: Not Yet Due Invoice
**Setup:**
- Customer: DEF Inc
- Credit Terms: 60 credit days
- Invoice Date: February 1, 2024
- Invoice Amount: $8,000
- As-Of Date: February 28, 2024

**Expected Results:**
- Due Date: April 1, 2024
- Days Overdue: 0 (not yet due)
- Ageing Bucket: Current
- Outstanding in Current bucket: $8,000

### Scenario 4: Statement Date Credit Terms
**Setup:**
- Customer: GHI Company
- Credit Terms: Statement Date
- Invoice Date: January 15, 2024
- Invoice Amount: $12,000
- As-Of Date: March 15, 2024

**Expected Results:**
- Due Date: March 2, 2024 (End of Jan + 30 days)
- Days Overdue: 13 days
- Ageing Bucket: 1-30 Days
- Outstanding in 1-30 Days bucket: $12,000

### Scenario 5: Multiple Invoices with Different Ages
**Setup:**
- Customer: Multi Corp
- Credit Terms: 30 credit days
- Invoices:
  1. Date: Dec 1, 2023, Amount: $5,000
  2. Date: Jan 1, 2024, Amount: $3,000
  3. Date: Feb 1, 2024, Amount: $7,000
- As-Of Date: March 1, 2024

**Expected Results:**
Invoice 1:
- Due Date: Dec 31, 2023
- Days Overdue: 60 days
- Bucket: 31-60 Days

Invoice 2:
- Due Date: Jan 31, 2024
- Days Overdue: 29 days
- Bucket: 1-30 Days

Invoice 3:
- Due Date: March 2, 2024
- Days Overdue: 0 (not yet due)
- Bucket: Current

**Customer Totals:**
- Current: $7,000
- 1-30 Days: $3,000
- 31-60 Days: $5,000
- Total Outstanding: $15,000

### Scenario 6: Deposit Credit Application
**Setup:**
- Customer: JKL Services
- Total Outstanding Invoices: $20,000
- Customer Deposit Credit: $5,000
- As-Of Date: March 1, 2024

**Expected Results:**
- Total Outstanding: $20,000
- Less: Deposit Credit: -$5,000
- Net Outstanding: $15,000

## SQL Queries for Verification

### 1. Check Invoice Outstanding Amount
```sql
SELECT
    i.invoice_number,
    i.total_price as invoice_total,
    COALESCE(SUM(rsi.amount), 0) as total_settled,
    (i.total_price - COALESCE(SUM(rsi.amount), 0)) as outstanding
FROM invoices i
LEFT JOIN receipt_settlement_invoices rsi ON rsi.invoice_id = i.id
WHERE i.id = [INVOICE_ID]
GROUP BY i.id, i.invoice_number, i.total_price;
```

### 2. Check Customer Deposit Credit
```sql
SELECT
    cd.customer_id,
    cd.amount as deposit_amount,
    COALESCE(SUM(rsd.amount), 0) as settled_amount,
    (cd.amount - COALESCE(SUM(rsd.amount), 0)) as available_credit
FROM customer_deposits cd
LEFT JOIN receipt_settlement_deposits rsd ON rsd.customer_deposit_id = cd.id
WHERE cd.customer_id = [CUSTOMER_ID]
    AND cd.status IN ('Printed', 'Partially Settled')
GROUP BY cd.id, cd.customer_id, cd.amount;
```

### 3. Verify Credit Terms
```sql
SELECT
    c.customer_name,
    c.customer_number,
    c.date_type as credit_term_type,
    c.credit_term_days
FROM customers c
WHERE c.id = [CUSTOMER_ID];
```

## Testing Checklist

### Pre-Test Setup
- [ ] Ensure test database has sample data
- [ ] Create customers with different credit terms
- [ ] Create invoices with various dates
- [ ] Create partial settlements
- [ ] Create customer deposits

### Calculation Tests
- [ ] Verify due date calculation for each credit term type
- [ ] Verify days overdue calculation
- [ ] Verify outstanding amount calculation
- [ ] Verify ageing bucket categorization
- [ ] Verify deposit credit application
- [ ] Verify grand totals

### Edge Cases
- [ ] Test with no credit terms (defaults to immediate payment)
- [ ] Test with future-dated invoices
- [ ] Test with fully settled invoices (should not appear)
- [ ] Test with zero-amount invoices
- [ ] Test with negative settlements (credit notes)

### Report Output Tests
- [ ] Excel generation with correct columns
- [ ] PDF generation with correct formatting
- [ ] Subtotals per customer
- [ ] Grand totals across all customers
- [ ] Number formatting (decimals, commas)
- [ ] Date formatting

## Expected vs Actual Comparison

Create a test spreadsheet with columns:
1. Test Case ID
2. Customer Name
3. Invoice Number
4. Invoice Date
5. Credit Terms
6. Expected Due Date
7. Actual Due Date
8. Expected Days Overdue
9. Actual Days Overdue
10. Expected Bucket
11. Actual Bucket
12. Pass/Fail

## Common Issues to Watch For

1. **Time Zone Issues:** Ensure dates are calculated consistently
2. **Rounding Errors:** Check decimal precision in calculations
3. **Null Handling:** Verify handling of missing credit terms
4. **Status Filtering:** Ensure only correct invoice statuses included
5. **Currency Conversion:** If multi-currency, verify conversion rates

## Regression Testing

After fixes are deployed:
1. Run report for previous month and compare with old version
2. Identify and document any discrepancies
3. Verify discrepancies are due to fixes, not new bugs
4. Get finance team approval on new calculations

## Sign-off Criteria

- [ ] All test scenarios pass
- [ ] Finance team approves calculation logic
- [ ] No regression in other reports
- [ ] Performance acceptable (< 30 seconds for 1000 invoices)
- [ ] Documentation updated