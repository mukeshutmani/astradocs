# Supplier Advance Payment System

## Overview

The Supplier Advance Payment System in PowerSuite allows users to make advance payments to suppliers that can later be applied against future invoices and cost settlements. This system maintains a running balance of advances per supplier and ensures proper financial tracking through integration with the accounting module.

## Table of Contents

1. [Database Schema](#database-schema)
2. [Business Workflow](#business-workflow)
3. [API Endpoints](#api-endpoints)
4. [User Interface](#user-interface)
5. [Accounting Integration](#accounting-integration)
6. [Key Business Rules](#key-business-rules)
7. [Technical Implementation](#technical-implementation)

---

## Database Schema

### Core Tables

#### supplier_deposits
**Location**: `psback/models/supplier_deposit.js`

Stores all advance payments made to suppliers.

| Column | Type | Description |
|--------|------|-------------|
| `id` | INT | Primary key |
| `payment_number` | VARCHAR | Unique document number (format: `{BranchCode}AP{00000001}`) |
| `supplier_id` | INT | Foreign key to suppliers table |
| `user_id` | INT | User who created the advance |
| `branch_id` | INT | Branch where advance was created |
| `currency_id` | INT | Currency of the advance payment |
| `amount` | DECIMAL | Original advance amount |
| `current_amount` | DECIMAL | Remaining available balance |
| `status` | ENUM | "Printed", "Void", or "Raised" |
| `pay_type_id` | INT | Payment method (Cash, Cheque, Card, Bank Transfer, GL Account) |
| `bank_id` | INT | Associated bank (for cheque/card payments) |
| `chart_of_account_id` | INT | GL account for cash accounting |
| `gl_account_id` | INT | GL settlement account (for GL payments) |
| `check_number` | VARCHAR | Cheque number (if applicable) |
| `card_number` | VARCHAR | Card number (if applicable) |
| `voucher_number` | VARCHAR | Voucher reference |
| `je_generated` | BOOLEAN | Journal entry created flag |
| `settled_invoice_number` | VARCHAR | References related settlements |
| `remarks` | TEXT | Additional notes |
| `created_at` | DATETIME | Creation timestamp |
| `updated_at` | DATETIME | Last update timestamp |

**Key Behavior**:
- `current_amount` starts equal to `amount`
- `current_amount` decreases as advance is used in settlements
- Only advances with `current_amount > 0` can be applied to new settlements

#### payment_settlement_deposit
**Location**: `psback/models/Payment/payment_settlement_deposit.js`

Junction table linking advances to payment settlements.

| Column | Type | Description |
|--------|------|-------------|
| `id` | INT | Primary key |
| `payment_settlement_id` | INT | Foreign key to payment_settlement |
| `supplier_deposit_id` | INT | Foreign key to supplier_deposit |
| `amount` | DECIMAL | How much of the advance was used |

#### payment_settlement
**Location**: `psback/models/Payment/payment_settlement.js`

Records when supplier costs are settled (paid to suppliers).

| Column | Type | Description |
|--------|------|-------------|
| `id` | INT | Primary key |
| `payment_number` | VARCHAR | Document number (format: `{BranchCode}PY{00000001}`) |
| `supplier_id` | INT | Supplier being paid |
| `amount` | DECIMAL | Total settlement amount |
| `status` | ENUM | "Printed" or "Void" |
| `je_generated` | BOOLEAN | Journal entry created flag |
| `created_at` | DATETIME | Creation timestamp |
| `updated_at` | DATETIME | Last update timestamp |

#### payment_settlement_cost
Links specific service costs to a payment settlement.

| Column | Type | Description |
|--------|------|-------------|
| `cost_id` | INT | Service cost being settled |
| `payment_settlement_id` | INT | Associated settlement |
| `amount` | DECIMAL | Amount being paid for this cost |

---

## Business Workflow

### 1. Create Supplier Advance Payment

**User Action**: Navigate to Advance Payment page and create new advance

**System Process**:
1. User selects supplier and payment method
2. User enters amount and currency
3. System generates unique payment number: `{BranchCode}AP{00000001}`
4. Creates `supplier_deposit` record:
   ```
   amount = 100,000 PKR
   current_amount = 100,000 PKR (full balance available)
   status = "Printed"
   ```
5. Generates receipt document for printing

**Example**:
```
Payment Number: 01AP00000001
Supplier: ABC Travel Services
Amount: PKR 100,000
Payment Method: Bank Transfer
Current Available: PKR 100,000
```

### 2. Create Service and Costs

**User Action**: Create service booking with associated supplier costs

**System Process**:
1. Service created (flight, hotel, etc.)
2. Cost record created with:
   - Supplier reference
   - Amount due
   - Status = "Raised" (awaiting payment)

### 3. Settle Supplier Costs Using Advance

**User Action**: Create payment settlement and apply advance payments

**System Process**:
1. User selects supplier and outstanding costs
2. System retrieves available advances for that supplier:
   ```sql
   WHERE supplier_id = ?
   AND status NOT IN ('Void', 'Raised')
   AND current_amount > 0
   ```
3. User selects which advances to use and amount from each
4. System validates: `used_amount ≤ current_amount`
5. Creates settlement records:

**payment_settlement**:
```
payment_number: 01PY00000001
supplier_id: 123
amount: 50,000
status: "Printed"
```

**payment_settlement_deposit** (advance application):
```
payment_settlement_id: 1
supplier_deposit_id: 1
amount: 50,000
```

**payment_settlement_cost** (cost linkage):
```
cost_id: 456
payment_settlement_id: 1
amount: 50,000
```

6. Updates `supplier_deposit.current_amount`:
   ```
   current_amount = 100,000 - 50,000 = 50,000 PKR
   ```

7. Updates cost status to "Paid" or "Partially Paid"

### 4. Remaining Balance Tracking

**After Settlement**:
```
Payment Number: 01AP00000001
Original Amount: PKR 100,000
Used in Settlements: PKR 50,000
Current Available: PKR 50,000
```

The remaining PKR 50,000 can be used in future settlements.

### 5. Multiple Settlements with Same Advance

**Scenario**: Use the same advance across multiple settlements

**Example Timeline**:
- **Day 1**: Create advance of PKR 100,000
- **Day 5**: Settle costs worth PKR 30,000 → Remaining: PKR 70,000
- **Day 10**: Settle costs worth PKR 40,000 → Remaining: PKR 30,000
- **Day 15**: Settle costs worth PKR 30,000 → Remaining: PKR 0

Each settlement creates a new `payment_settlement_deposit` record.

### 6. Voiding Advances

**User Action**: Void an unused or partially used advance

**System Validations**:
1. Checks if advance is linked to non-voided settlements
2. If linked, prevents voiding (displays error)
3. If not linked or only partially used:
   - Sets `status = "Void"`
   - Sets `current_amount = 0`
   - Prevents future usage

**Business Rule**: Cannot void an advance that has been applied to active settlements.

---

## API Endpoints

### Supplier Deposit Endpoints
**Base Route**: `/api/supplier_deposit`

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/supplier_deposit` | Create new advance payment |
| GET | `/supplier_deposit` | List all advances (paginated, searchable) |
| GET | `/supplier_deposit/:id` | Get specific advance by ID |
| GET | `/supplier_deposit/supplier/:supplierId` | Get advances for specific supplier (with date filters) |
| GET | `/supplier_deposit/receipt/:paymentNumber` | Get advance receipt document |
| PUT | `/supplier_deposit/:id` | Update advance details |
| PUT | `/supplier_deposit/:paymentNumber/void` | Void advance payment |
| DELETE | `/supplier_deposit/:id` | Delete advance |

### Payment Settlement Endpoints
**Base Route**: `/api/payment`

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/payment/settle` | Create payment settlement |
| GET | `/payment/settlement/:paymentNumber` | Get settlement details |
| GET | `/payment/getPaymentSettlementPDF/:paymentNumber` | Generate PDF document |
| PUT | `/payment/settlement/:paymentNumber/void` | Void settlement |
| DELETE | `/payment/settlement/:paymentNumber` | Delete settlement |

### Example API Calls

**Create Advance Payment**:
```javascript
POST /api/supplier_deposit
{
  "supplier_id": 123,
  "amount": 100000,
  "currency_id": 1,
  "pay_type_id": 6, // Bank Transfer
  "bank_id": 5,
  "remarks": "Advance for winter season bookings"
}
```

**Get Available Advances for Supplier**:
```javascript
GET /api/supplier_deposit/supplier/123
// Returns all non-void advances with current_amount > 0
```

**Settle Payment with Advance**:
```javascript
POST /api/payment/settle
{
  "supplier_id": 123,
  "costs": [
    { "cost_id": 456, "amount": 50000 }
  ],
  "deposits": [
    { "supplier_deposit_id": 1, "amount": 50000 }
  ]
}
```

---

## User Interface

### 1. Advance Payment Creation Page
**Location**: `psfront/src/pages/PaymentsPage/AdvancePayment.jsx`

**Features**:
- **Supplier Selection**: Live search combo box
- **Branch Selection**: Defaults to user's company branch
- **Payment Method Form**: Dynamic fields based on selected method
  - Cash: GL account selection
  - Cheque: Cheque number input
  - Credit Card: Bank, card type, card number
  - Bank Transfer: Bank account selection
  - GL Account: GL settlement account
- **Currency Selection**: Defaults to PKR, supports multi-currency
- **Amount Input**: Formatted number input
- **Date Selection**: System date validation (up to N days back)
- **Remarks**: Additional notes textarea
- **Keyboard Shortcuts**: Ctrl+S to submit

**Validation**:
- Required fields based on payment method
- Cheque number format validation
- Amount must be positive
- Automatic chart_of_account assignment based on pay_type

**After Submission**:
- Redirects to PDF receipt view
- Receipt can be printed/saved for records

### 2. Settlement Page with Advance Application
**Location**: `psfront/src/pages/RecieptPage/Reciept.jsx`

**Features**:
- Select supplier and view outstanding costs
- View available advances for selected supplier
- Apply advances to settlement:
  - Shows `current_amount` (available balance)
  - Input field for amount to use
  - Multi-currency conversion display
  - Real-time validation
- Prevent overpayment: `used_amount ≤ current_amount`
- Shows total available from all deposits and credit notes

**Validation Logic**:
```javascript
const isDepositUsageValid = () => {
  return depositValues.every(depositValue => {
    const maxUsable = parseFloat(deposit.current_amount || 0);
    const usedAmount = parseFloat(depositValue.amount || 0);
    return usedAmount <= maxUsable;
  });
}
```

### 3. Deposit List Page
**Location**: `psfront/src/pages/Document/DepositList.jsx`

**Features**:
- Date range filtering
- Status filtering (Printed, Partially Settled, Settled, Void)
- Search by document number, supplier, amount
- Sortable columns
- Pagination
- Click to view receipt document

---

## Accounting Integration

### Chart of Accounts (COA)

**Payment Method to COA Mapping**:
Each payment method references specific GL accounts via `pay_type_form` table:

| Payment Method | GL Account Type | Example Account |
|----------------|-----------------|-----------------|
| Cash | Cash Box Account | 1000 - Cash in Hand |
| Cheque | Bank Account | 1010 - Bank Account ABC |
| Credit Card | Bank Account | 1020 - Card Settlement Account |
| Bank Transfer | Bank Account | 1010 - Bank Account XYZ |
| GL Account | Custom GL Account | User-selected settlement account |

**supplier_deposit.chart_of_account_id**: References the COA for this payment

### Journal Entry Generation

**Fields**:
- `je_generated`: Boolean flag indicating journal entry creation status
- `coa_id`: Chart of account ID for the settlement

**Journal Entry Flow** (when je_generated = false):
1. Debit: Supplier Advance Account (Asset)
2. Credit: Payment Method Account (Cash/Bank)

**Example Journal Entry**:
```
Date: 2025-01-15
Reference: 01AP00000001

Debit:  Supplier Advances - ABC Travel    PKR 100,000
Credit: Bank Account - XYZ Bank                      PKR 100,000

Description: Advance payment to ABC Travel Services
```

**When Advance is Applied to Settlement**:
1. Debit: Accounts Payable - Supplier
2. Credit: Supplier Advances (reduces asset)

**Example**:
```
Date: 2025-01-20
Reference: 01PY00000001

Debit:  Accounts Payable - ABC Travel     PKR 50,000
Credit: Supplier Advances - ABC Travel               PKR 50,000

Description: Applied advance to settlement 01PY00000001
```

### GL Settlement Accounts

**Special Account Type**: `gl_settle_account`
- Used when `payment_type_id = 5` (GL Account payment)
- Linked via `gl_account_id` in `supplier_deposit`
- Allows direct settlement against custom GL accounts

---

## Key Business Rules

### 1. Advance Creation Rules
- Supplier must exist in the system
- Amount must be greater than zero
- Currency must be valid (defaults to PKR)
- Payment method determines required fields:
  - Cheque: Must provide cheque number
  - Credit Card: Must provide bank and card details
  - GL Account: Must select GL settlement account
- Branch determines document number prefix
- Status automatically set to "Printed"
- `current_amount` initialized to equal `amount`

### 2. Advance Usage Rules
- Only advances with status "Printed" can be used
- Cannot use voided advances (status = "Void")
- Cannot use more than `current_amount` (available balance)
- Supplier in advance must match supplier in settlement
- Multi-currency advances converted using exchange rate
- Can use multiple advances in single settlement
- Can use single advance across multiple settlements

### 3. Advance Voiding Rules
- Cannot void if linked to non-voided settlements
- Voiding sets:
  - `status = "Void"`
  - `current_amount = 0`
- Voided advances excluded from available list
- Prevents future usage

### 4. Settlement Rules
- Payment settlement can include:
  - Multiple costs
  - Multiple advances (deposits)
  - Multiple payment methods
  - Credit notes
- Total settlement amount = sum of all payment sources
- Cost status updated based on total paid:
  - "Paid": total_paid ≥ cost_amount
  - "Partially Paid": total_paid < cost_amount
- Settlement can be voided (reverses journal entries)

### 5. Balance Tracking Rules
- `current_amount` tracks remaining balance
- Decremented when used in settlement:
  ```
  new_current_amount = old_current_amount - used_amount
  ```
- Cannot go negative (validated before settlement)
- When `current_amount = 0`, advance no longer appears in available list

### 6. Multi-Currency Rules
- Each advance stored in specific currency
- Exchange rate applied during settlement:
  ```
  base_currency_amount = advance_amount × exchange_rate
  ```
- Conversion displayed in UI for transparency
- Settlement can mix different currencies
- Each currency converted independently

---

## Technical Implementation

### Document Numbering System

**Format**: `{BranchCode}{Type}{SequenceNumber}`

**Components**:
- **BranchCode**: 2-digit branch identifier (e.g., "01")
- **Type**: 2-letter document type code:
  - "AP" = Advance Payment
  - "PY" = Payment Settlement
  - "IN" = Invoice
- **SequenceNumber**: 8-digit sequential number (e.g., "00000001")

**Examples**:
- `01AP00000001` - First advance payment in branch 01
- `02AP00000125` - 125th advance payment in branch 02
- `01PY00000050` - 50th payment settlement in branch 01

**Generation Logic**:
```javascript
// Find highest existing number for this branch and type
const lastPayment = await SupplierDeposit.findOne({
  where: { branch_id: branchId },
  order: [['payment_number', 'DESC']]
});

// Extract sequence number and increment
const lastNumber = parseInt(lastPayment.payment_number.substring(4)) || 0;
const nextNumber = (lastNumber + 1).toString().padStart(8, '0');

// Generate new payment number
const paymentNumber = `${branchCode}AP${nextNumber}`;
```

**Characteristics**:
- Branch-specific (not company-wide)
- Sequential and gapless
- Unique per branch and document type
- Sortable chronologically

### Database Relationships

**ER Diagram**:
```
Supplier (1) ←→ (Many) Supplier_Deposit
                           ↓
                           | (1)
                           ↓
                  Payment_Settlement_Deposit
                           ↓
                           | (Many)
                           ↓
              Payment_Settlement (1) ←→ (Many) Payment_Settlement_Cost
                                                        ↓
                                                        | (Many)
                                                        ↓
                                                      Cost
```

**Sequelize Associations** (`psback/models/index.js:629-653`):
```javascript
// Supplier to Advances
Supplier.hasMany(SupplierDeposit, { foreignKey: 'supplier_id' });
SupplierDeposit.belongsTo(Supplier, { foreignKey: 'supplier_id' });

// Advance to Junction Table
SupplierDeposit.hasOne(PaymentSettlementDeposit, {
  foreignKey: 'supplier_deposit_id'
});
PaymentSettlementDeposit.belongsTo(SupplierDeposit, {
  foreignKey: 'supplier_deposit_id'
});

// Settlement to Junction Table
PaymentSettlement.hasMany(PaymentSettlementDeposit, {
  foreignKey: 'payment_settlement_id'
});
PaymentSettlementDeposit.belongsTo(PaymentSettlement, {
  foreignKey: 'payment_settlement_id'
});

// Settlement to Costs
PaymentSettlement.hasMany(PaymentSettlementCost, {
  foreignKey: 'payment_settlement_id'
});
PaymentSettlementCost.belongsTo(PaymentSettlement, {
  foreignKey: 'payment_settlement_id'
});
```

### Key Controller Methods

#### createSupplierDeposit()
**Location**: `psback/controllers/supplier_deposit.controller.js`

**Process**:
1. Validate branch (or use default company branch)
2. Generate unique payment_number
3. Set `current_amount = amount`
4. Set `status = "Printed"`
5. Capture user, branch, payment method details
6. Store payment-specific info (cheque #, card #, GL account)
7. Return created record with receipt link

#### getSupplierDepositBySupplierId()
**Location**: `psback/controllers/supplier_deposit.controller.js`

**Query Logic**:
```javascript
where: {
  supplier_id: supplierId,
  status: { [Op.notIn]: ['Void', 'Raised'] },
  current_amount: { [Op.gt]: 0 }
}
```

**Returns**:
- Deposits with available balance only
- Includes currency and exchange rate info
- Supports date range filtering
- Includes document link for viewing

#### settlePayment()
**Location**: `psback/controllers/payment.controller.js`

**Transaction Flow**:
1. Generate payment_settlement number
2. Create `payment_settlement` record
3. Create `payment_settlement_cost` records (link costs)
4. Create `payment_settlement_deposit` records (link advances)
5. Update cost status based on total paid:
   ```javascript
   const totalPaid = calculateTotalPaid(cost_id);
   const costAmount = (publishedRate - commission) + extraCharges + taxes + wht;

   if (totalPaid >= costAmount) {
     cost.status = 'Paid';
   } else {
     cost.status = 'Partially Paid';
   }
   ```
6. Decrement `supplier_deposit.current_amount`
7. Commit transaction
8. Return settlement document

### Cost Amount Calculation

**Formula**:
```
Total Cost = (Published Rate - Commission) + Extra Charges + Taxes + WHT

Where:
- Published Rate: Base cost from supplier
- Commission: Discount/commission received
- Extra Charges: Additional fees
- Taxes: Applicable taxes
- WHT: Withholding tax
```

**Example**:
```
Published Rate: PKR 50,000
Commission: PKR 2,000
Extra Charges: PKR 1,000
Taxes: PKR 5,000
WHT: PKR 500

Total Cost = (50,000 - 2,000) + 1,000 + 5,000 + 500
           = 48,000 + 1,000 + 5,000 + 500
           = PKR 54,500
```

### Partial Settlement Tracking

**Scenario**: Cost of PKR 100,000 paid in 3 installments

**Settlement 1** (PKR 30,000):
```javascript
payment_settlement_cost {
  cost_id: 456,
  payment_settlement_id: 1,
  amount: 30000
}

// Cost status updated
cost.status = 'Partially Paid'
cost.total_paid = 30000
```

**Settlement 2** (PKR 40,000):
```javascript
payment_settlement_cost {
  cost_id: 456,
  payment_settlement_id: 2,
  amount: 40000
}

// Cost status still partial
cost.status = 'Partially Paid'
cost.total_paid = 70000
```

**Settlement 3** (PKR 30,000):
```javascript
payment_settlement_cost {
  cost_id: 456,
  payment_settlement_id: 3,
  amount: 30000
}

// Cost status updated to paid
cost.status = 'Paid'
cost.total_paid = 100000
```

**Cumulative Calculation**:
```javascript
const totalPaid = await PaymentSettlementCost.sum('amount', {
  where: {
    cost_id: costId,
    '$payment_settlement.status$': { [Op.ne]: 'Void' }
  },
  include: [{
    model: PaymentSettlement,
    attributes: []
  }]
});
```

### Exchange Rate Handling

**Multi-Currency Example**:

**Advance in USD**:
```
amount: 1,000 USD
currency_id: 2 (USD)
exchange_rate: 280 PKR/USD
```

**During Settlement** (base currency = PKR):
```javascript
const advanceInBaseCurrency = advance.amount * exchangeRate;
// 1,000 USD × 280 = 280,000 PKR

// User can use up to PKR 280,000 equivalent from this advance
```

**Validation**:
```javascript
// Convert used amount back to advance currency for validation
const usedInAdvanceCurrency = usedAmount / exchangeRate;
// 140,000 PKR / 280 = 500 USD

// Check against current_amount in advance currency
if (usedInAdvanceCurrency <= advance.current_amount) {
  // Valid usage
  advance.current_amount -= usedInAdvanceCurrency;
  // 1,000 - 500 = 500 USD remaining
}
```

---

## Complete Example Workflow

### Scenario
ABC Travel Services is a supplier. We will make a PKR 200,000 advance payment and use it across multiple settlements.

### Step 1: Create Advance Payment

**User Action**:
1. Navigate to "Advance Payment to Supplier"
2. Select: ABC Travel Services (supplier_id: 123)
3. Payment Method: Bank Transfer
4. Bank Account: XYZ Bank (bank_id: 5)
5. Amount: PKR 200,000
6. Currency: PKR (currency_id: 1)
7. Remarks: "Advance for Q1 bookings"
8. Submit (Ctrl+S)

**System Response**:
```sql
INSERT INTO supplier_deposits (
  payment_number, supplier_id, user_id, branch_id,
  currency_id, amount, current_amount, status,
  pay_type_id, bank_id, remarks, created_at
) VALUES (
  '01AP00000042', 123, 10, 1,
  1, 200000, 200000, 'Printed',
  6, 5, 'Advance for Q1 bookings', NOW()
);
```

**Result**:
- Payment Number: `01AP00000042`
- Available Balance: PKR 200,000
- Status: Printed
- Receipt PDF generated

### Step 2: Create Services and Costs

**User creates 3 flight bookings**:

**Cost 1** (Flight to Dubai):
```sql
INSERT INTO cost (
  supplier_id, published_rate, commission,
  extra_charges, taxes, wht, status
) VALUES (
  123, 80000, 5000, 2000, 8000, 800, 'Raised'
);
-- cost_id: 501
-- Total: (80000-5000) + 2000 + 8000 + 800 = 85,800
```

**Cost 2** (Flight to London):
```sql
INSERT INTO cost (
  supplier_id, published_rate, commission,
  extra_charges, taxes, wht, status
) VALUES (
  123, 120000, 8000, 3000, 12000, 1200, 'Raised'
);
-- cost_id: 502
-- Total: (120000-8000) + 3000 + 12000 + 1200 = 128,200
```

**Cost 3** (Flight to Istanbul):
```sql
INSERT INTO cost (
  supplier_id, published_rate, commission,
  extra_charges, taxes, wht, status
) VALUES (
  123, 60000, 4000, 1500, 6000, 600, 'Raised'
);
-- cost_id: 503
-- Total: (60000-4000) + 1500 + 6000 + 600 = 64,100
```

**Total Outstanding**: PKR 278,100

### Step 3: First Settlement (Pay Dubai Flight)

**User Action**:
1. Navigate to Payment Settlement
2. Select Supplier: ABC Travel Services
3. Select Cost: Dubai Flight (PKR 85,800)
4. Click "Apply Advances"
5. Select Advance: 01AP00000042 (Available: PKR 200,000)
6. Enter Amount: PKR 85,800
7. Submit

**System Response**:
```sql
-- Create settlement
INSERT INTO payment_settlement (
  payment_number, supplier_id, amount, status, created_at
) VALUES (
  '01PY00000120', 123, 85800, 'Printed', NOW()
);
-- payment_settlement_id: 1

-- Link cost
INSERT INTO payment_settlement_cost (
  payment_settlement_id, cost_id, amount
) VALUES (
  1, 501, 85800
);

-- Link advance
INSERT INTO payment_settlement_deposit (
  payment_settlement_id, supplier_deposit_id, amount
) VALUES (
  1, 42, 85800
);

-- Update advance balance
UPDATE supplier_deposits
SET current_amount = 200000 - 85800
WHERE id = 42;
-- current_amount now: 114,200

-- Update cost status
UPDATE cost
SET status = 'Paid'
WHERE id = 501;
```

**Result**:
- Settlement: `01PY00000120` created
- Advance Balance: PKR 114,200 remaining
- Cost 501: Paid

### Step 4: Second Settlement (Partial Payment for London)

**User Action**:
1. Select Cost: London Flight (PKR 128,200)
2. Select Advance: 01AP00000042 (Available: PKR 114,200)
3. Enter Amount: PKR 100,000 (partial payment)
4. Submit

**System Response**:
```sql
-- Create settlement
INSERT INTO payment_settlement (
  payment_number, supplier_id, amount, status, created_at
) VALUES (
  '01PY00000121', 123, 100000, 'Printed', NOW()
);

-- Link cost
INSERT INTO payment_settlement_cost (
  payment_settlement_id, cost_id, amount
) VALUES (
  2, 502, 100000
);

-- Link advance
INSERT INTO payment_settlement_deposit (
  payment_settlement_id, supplier_deposit_id, amount
) VALUES (
  2, 42, 100000
);

-- Update advance balance
UPDATE supplier_deposits
SET current_amount = 114200 - 100000
WHERE id = 42;
-- current_amount now: 14,200

-- Update cost status (partial payment)
UPDATE cost
SET status = 'Partially Paid'
WHERE id = 502;
```

**Result**:
- Settlement: `01PY00000121` created
- Advance Balance: PKR 14,200 remaining
- Cost 502: Partially Paid (PKR 100,000 of PKR 128,200)
- Outstanding on Cost 502: PKR 28,200

### Step 5: Third Settlement (Use Remaining Advance for Istanbul)

**User Action**:
1. Select Cost: Istanbul Flight (PKR 64,100)
2. Select Advance: 01AP00000042 (Available: PKR 14,200)
3. Enter Amount: PKR 14,200 (use all remaining)
4. Add Bank Transfer: PKR 49,900 (to cover balance)
5. Submit

**System Response**:
```sql
-- Create settlement
INSERT INTO payment_settlement (
  payment_number, supplier_id, amount, status, created_at
) VALUES (
  '01PY00000122', 123, 64100, 'Printed', NOW()
);

-- Link cost
INSERT INTO payment_settlement_cost (
  payment_settlement_id, cost_id, amount
) VALUES (
  3, 503, 64100
);

-- Link advance
INSERT INTO payment_settlement_deposit (
  payment_settlement_id, supplier_deposit_id, amount
) VALUES (
  3, 42, 14200
);

-- Link bank payment
INSERT INTO payment_settlement_payment (
  payment_settlement_id, pay_type_id, bank_id, amount
) VALUES (
  3, 6, 5, 49900
);

-- Update advance balance
UPDATE supplier_deposits
SET current_amount = 14200 - 14200
WHERE id = 42;
-- current_amount now: 0 (fully used)

-- Update cost status
UPDATE cost
SET status = 'Paid'
WHERE id = 503;
```

**Result**:
- Settlement: `01PY00000122` created
- Advance Balance: PKR 0 (fully utilized)
- Cost 503: Paid
- Advance 01AP00000042 will no longer appear in available list

### Summary

**Original Advance**: PKR 200,000

**Settlements**:
1. `01PY00000120`: Used PKR 85,800 → Remaining: PKR 114,200
2. `01PY00000121`: Used PKR 100,000 → Remaining: PKR 14,200
3. `01PY00000122`: Used PKR 14,200 → Remaining: PKR 0

**Total Utilized**: PKR 200,000 across 3 separate settlements

**Outstanding Costs**:
- Cost 501: Fully Paid
- Cost 502: Partially Paid (PKR 28,200 outstanding)
- Cost 503: Fully Paid

---

## Key Files Reference

### Backend Models
| File | Purpose |
|------|---------|
| `psback/models/supplier_deposit.js` | Advance payment model |
| `psback/models/Payment/payment_settlement.js` | Settlement model |
| `psback/models/Payment/payment_settlement_cost.js` | Cost linkage |
| `psback/models/Payment/payment_settlement_deposit.js` | Advance-settlement junction |

### Backend Controllers
| File | Purpose |
|------|---------|
| `psback/controllers/supplier_deposit.controller.js` | Advance CRUD operations |
| `psback/controllers/payment.controller.js` | Settlement operations |

### Backend Routes
| File | Purpose |
|------|---------|
| `psback/routes/supplier_deposit.route.js` | Advance endpoints |
| `psback/routes/payment.route.js` | Settlement endpoints |

### Frontend Components
| File | Purpose |
|------|---------|
| `psfront/src/pages/PaymentsPage/AdvancePayment.jsx` | Create advance UI |
| `psfront/src/pages/RecieptPage/Reciept.jsx` | Apply advances to settlements |
| `psfront/src/pages/Document/DepositList.jsx` | List/search advances |

### Frontend API
| File | Purpose |
|------|---------|
| `psfront/src/api/supplier_deposit.js` | Advance API client |
| `psfront/src/api/payment.js` | Settlement API client |

---

## Best Practices

1. **Always verify available balance** before applying advance to settlement
2. **Use proper payment methods** with required details (cheque numbers, GL accounts)
3. **Track multi-currency advances** carefully with correct exchange rates
4. **Generate journal entries** promptly after creating advances and settlements
5. **Void advances only when necessary** and ensure no active settlements exist
6. **Document remarks** for audit trail and clarity
7. **Monitor partial settlements** to ensure full payment tracking
8. **Reconcile regularly** between supplier deposits and accounts payable

---

## Troubleshooting

### Issue: Cannot void advance
**Cause**: Advance is linked to non-voided settlement
**Solution**: Void the settlement first, then void the advance

### Issue: Advance not appearing in available list
**Causes**:
- Status is "Void" or "Raised"
- `current_amount = 0` (fully used)
- Different supplier selected
**Solution**: Check advance status and balance, ensure correct supplier

### Issue: Settlement amount exceeds available balance
**Cause**: Trying to use more than `current_amount`
**Solution**: Use partial amount from advance or add additional payment method

### Issue: Multi-currency conversion error
**Cause**: Missing or incorrect exchange rate
**Solution**: Verify exchange rate is set for the advance currency

### Issue: Cost status not updating after settlement
**Cause**: Settlement voided or total paid calculation error
**Solution**: Check all settlements for the cost are non-voided and recalculate total

---

## Future Enhancements

1. **Auto-apply advances**: Automatically apply oldest advances when settling costs
2. **Advance expiry**: Set expiration dates for unused advances
3. **Advance approval workflow**: Require approval for advances above threshold
4. **Refund mechanism**: Handle unused advance refunds
5. **Advanced reporting**: Advance aging, utilization reports
6. **Bulk settlements**: Apply multiple advances to multiple costs in one transaction
7. **Advance transfer**: Transfer unused advance from one supplier to another
8. **Interest calculation**: Calculate interest on advance amounts

---

*Document Version: 1.0*
*Last Updated: 2025-11-20*
*System: PowerSuite - Travel Booking Management*
