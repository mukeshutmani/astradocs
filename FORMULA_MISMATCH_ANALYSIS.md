# Customer Terms Tab - Formula Mismatch Analysis

## Problem Summary
The financial calculations displayed in the Customer Profile Terms Tab do not match the formulas used in the Customer Position Report. This causes discrepancies when users view the same customer data in different places.

---

## Current Implementation

### Terms Tab (FinanceFields Component)
**File:** `/psfront/src/components/CustomerForm/FinanceFields.jsx`

Currently displays 5 separate fields:
1. **Total Invoices** - Sum of all invoices
2. **Total Settlements** - Sum of settlement amounts
3. **Total Deposit** - Sum of deposit amounts
4. **Total Refund** - Sum of refunds (credit notes)
5. **Current Outstanding** - Calculated balance

**Current Formula (Backend - getCustomer):**
```
Outstanding Balance = Total Invoices - Total Refunds - Total Deposits
```

**Issue:** This formula is INCOMPLETE and does NOT match the Customer Position Report.

---

## Customer Position Report Implementation

### Report Formula (Correct Formula)
**File:** `/psback/controllers/report.controller.js` (lines 6684-6710)

Uses the shared calculator: `calculateCustomerBalance()`

**Formula Components:**
1. **Opening Balance** (if date filter applied)
   - Historical Invoices - Historical Receipts - Historical Credit Notes - Historical Deposits

2. **Period Calculations:**
   - Period Invoices: All invoices in date range (converted to PKR)
   - Period Refunds: All credit notes in date range
   - Period Receipts: Only G/L account payments (excluding deposit payments)
   - Period Deposits: Full deposit amounts

3. **Final Balance:**
```
Net Balance = Opening Balance + Period Invoices - Period Refunds - (Period Receipts + Period Deposits) + Payments
```

**Key Difference:** Receipts and Deposits are COMBINED in the report

---

## Detailed Comparison

### What's Included

| Component | Terms Tab | Position Report |
|-----------|-----------|-----------------|
| Invoices | All (Printed, Partially Settled, Settled) | All (Printed, Partially Settled, Settled) |
| Refunds | Credit notes only | Credit notes only (service refunds excluded) |
| Deposits | Full amount, status=Printed | Full amount, status!=Void |
| Receipts | All amounts | Only G/L account payments (excludes deposit payments) |
| Settlements | Included separately | Combined with Receipts |
| Currency Conversion | Some invoices to PKR | All foreign invoices to PKR |
| Service Refunds | Excluded (correct) | Excluded (correct) |

### What's Different

#### 1. Receipt Filtering
**Terms Tab (Current - INCORRECT):**
- Shows all receipt amounts without filtering

**Position Report (CORRECT):**
- Only counts G/L account payments
- Excludes deposit-related payments
- Logic at line 6617-6626:
```javascript
receipt.receipt_settlement_payments.forEach(payment => {
    const hasGLAccount = payment.gl_settle_account?.chart_of_account;
    const isDeposit = payment.customer_deposit;
    
    // Only count G/L account payments (not deposits)
    if (hasGLAccount && !isDeposit) {
        glAccountTotal += parseFloat(payment.base_amount || payment.amount || 0);
    }
});
```

#### 2. Receipts & Deposits Handling
**Terms Tab (Current - INCORRECT):**
- Displays separately as `total_settlements` and `total_deposit`
- Outstanding Balance = Invoices - Refunds - Deposits (MISSING receipts)

**Position Report (CORRECT):**
- Combines both: `combinedReceipts = receipts + deposits` (line 6697)
- Shows as "LESS:Receipts" which includes both
- Formula: `Net Balance = Opening + Invoices - Refunds - (Receipts + Deposits)`

#### 3. Opening Balance
**Terms Tab (Current):**
- Does NOT calculate opening balance
- Shows only period/current data

**Position Report (CORRECT):**
- Calculates opening balance when date filter applied
- Uses historical data before start date
- Affects final net balance calculation

#### 4. Payment Settlements
**Terms Tab (Current):**
- Not displayed at all

**Position Report (CORRECT):**
- Always 0 as requested (line 6706)
- Included in formula: `+ Payments`

---

## What Needs to Be Fixed

### Frontend Changes Required

#### 1. Update FinanceFields Component
**File:** `/psfront/src/components/CustomerForm/FinanceFields.jsx`

**Changes:**
- Remove `total_settlements` as separate field
- **Combine** receipts and deposits into single "Total Receipts & Deposits" field
- Update outstanding balance calculation
- Add opening balance if applicable
- Update calculation formula

**New Display Fields:**
```
1. Total Invoices (read-only)
2. Total Refunds (read-only)
3. Total Receipts & Deposits (read-only) ← COMBINED
4. Current Outstanding (read-only) ← Updated formula
```

#### 2. Update Outstanding Balance Calculation
**Current (INCORRECT):**
```javascript
outstandingBalance = totalInvoicesAmount - customer_refund_amount - totalDepositAmount
```

**New (CORRECT):**
```javascript
// Combined receipts and deposits
const combinedReceipts = totalSettlementsAmount + totalDepositAmount;

// Outstanding balance
outstandingBalance = totalInvoicesAmount - customer_refund_amount - combinedReceipts
```

### Backend Changes Required

#### 1. Update getCustomer Controller
**File:** `/psback/controllers/customer.controller.js` (lines 1630-1898)

**Changes Required:**

1. **Filter Receipts to G/L Accounts Only** (currently missing)
   - Add logic to exclude deposit payments from receipt totals
   - Match the logic in report.controller.js (lines 6617-6626)

2. **Update Financial Summary** (lines 1860-1881)
   - Separate `periodReceipts` (G/L only) from total receipts
   - Combine in response for clarity

3. **Add Receipt Filtering Logic:**
```javascript
// Process receipts to only count G/L account payments (like Position Report)
const processedReceipts = receipt_settlement_invoices.map(receipt => {
    let glAccountTotal = 0;
    
    if (receipt.receipt_settlement_payments && receipt.receipt_settlement_payments.length > 0) {
        receipt.receipt_settlement_payments.forEach(payment => {
            const hasGLAccount = payment.gl_settle_account?.chart_of_account;
            const isDeposit = payment.customer_deposit;
            
            // Only count G/L account payments (not deposits)
            if (hasGLAccount && !isDeposit) {
                glAccountTotal += parseFloat(payment.base_amount || payment.amount || 0);
            }
        });
    }
    
    return glAccountTotal;
});

totalSettlementsAmount = processedReceipts.reduce((sum, amount) => sum + amount, 0);
```

4. **Update Outstanding Balance Calculation** (line 1869-1870):
```javascript
// OLD (INCORRECT):
// outstandingBalance: totalInvoicesAmount - customer_refund_amount - totalDepositAmount

// NEW (CORRECT):
combinedReceipts: totalSettlementsAmount + totalDepositAmount,
outstandingBalance: totalInvoicesAmount - customer_refund_amount - (totalSettlementsAmount + totalDepositAmount)
```

#### 2. Use Shared Calculator (Optional but Recommended)
Use the existing `customer_balance_calculator.js` service to match Position Report logic exactly:

```javascript
// Import at top
const { calculateCustomerBalance } = require('../services/customer_balance_calculator');

// In getCustomer function
const balances = calculateCustomerBalance({
    invoices: invoices,
    invoiceTaxes: invoiceTaxes,
    refunds: [], // Service refunds excluded
    receipts: glFilteredReceipts,
    deposits: customerDeposits,
    creditNotes: creditNotes,
    calculateOpeningBalance: false // Or true if date range provided
});

// Use calculated values
totalInvoicesAmount = balances.periodInvoices;
totalSettlementsAmount = balances.periodReceipts;
customer_refund_amount = balances.periodRefunds;
totalDepositAmount = balances.periodDeposits;
const combinedReceipts = balances.periodReceipts + balances.periodDeposits;
outstandingBalance = totalInvoicesAmount - customer_refund_amount - combinedReceipts;
```

---

## Implementation Steps

### Phase 1: Backend Fix (Priority 1)
1. Update `getCustomer()` in customer.controller.js
   - Add G/L account filtering for receipts
   - Update financial summary calculation
   - Return `combinedReceipts` in response

### Phase 2: Frontend Fix (Priority 1)
1. Update FinanceFields component
   - Remove separate settlements field
   - Add combined receipts & deposits field
   - Update outstanding balance formula
   - Update displayed values

### Phase 3: Verification (Priority 1)
1. Test with same customer in:
   - Customer Profile → Terms Tab
   - Customer Position Report (with no date filter)
2. Verify matching values
3. Test with date filters

### Phase 4: Complete Integration (Priority 2)
1. Consider using shared calculator for consistency
2. Update tests to reflect new formula
3. Document new formula in code comments

---

## API Response Changes

### Current Response
```javascript
{
  financialSummary: {
    totalInvoicesAmount: 1000,
    totalSettlementsAmount: 200,
    totalDepositAmount: 300,
    customer_refund_amount: 100,
    outstandingBalance: 400 // 1000 - 100 - 300 (INCORRECT - missing receipts)
  }
}
```

### Fixed Response
```javascript
{
  financialSummary: {
    totalInvoicesAmount: 1000,
    totalSettlementsAmount: 200,      // G/L account payments only (filtered)
    totalDepositAmount: 300,
    customer_refund_amount: 100,      // Credit notes only
    combinedReceipts: 500,             // 200 + 300
    outstandingBalance: 400            // 1000 - 100 - 500 (CORRECT)
  }
}
```

---

## Testing Strategy

### Test Case 1: Simple Customer (No Receipts/Deposits)
- Expected: outstandingBalance = totalInvoices - refunds

### Test Case 2: Customer with Receipts
- Expected: outstandingBalance = totalInvoices - refunds - receipts

### Test Case 3: Customer with Deposits
- Expected: outstandingBalance = totalInvoices - refunds - deposits

### Test Case 4: Customer with Both Receipts & Deposits
- Expected: outstandingBalance = totalInvoices - refunds - (receipts + deposits)

### Test Case 5: Matching Reports
- Load same customer in Terms Tab and Position Report (no date filter)
- All financial totals should match exactly

---

## Key Files to Modify

| File | Change | Priority |
|------|--------|----------|
| `/psback/controllers/customer.controller.js` | Add G/L filtering, update calculation | P1 |
| `/psfront/src/components/CustomerForm/FinanceFields.jsx` | Combine receipts+deposits, update display | P1 |
| `/psfront/src/pages/CustomerPage/CreateCustomer.jsx` | Pass new props to FinanceFields | P1 |
| `/psback/services/customer_balance_calculator.js` | Use in controller (optional) | P2 |

---

## Summary of Formula Changes

| Aspect | Current (WRONG) | Correct (Position Report) |
|--------|-----------------|-------------------------|
| **Formula** | Invoices - Refunds - Deposits | Invoices - Refunds - (Receipts + Deposits) |
| **Receipts** | Not filtered | G/L accounts only (no deposits) |
| **Deposits** | Included separately | Combined with Receipts |
| **Settlements** | Shown as separate field | Combined with Deposits |
| **Opening Balance** | Not calculated | Calculated when date filter applied |
| **Display Fields** | 5 fields | 4 fields (combined receipts) |

