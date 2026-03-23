# Customer Profile Terms Tab - Architecture Analysis

## Overview
The Customer Profile Terms Tab displays financial calculations for a customer including credit terms, deposit information, and financial summaries. This analysis covers the frontend component, API endpoint, and backend business logic.

---

## 1. Frontend Component Structure

### Main Component File
**Location:** `/mnt/c/Codes/Powersuite/psfront/src/pages/CustomerPage/CreateCustomer.jsx`

The Terms tab is implemented as a tab within the tabbed interface at **lines 1073-1097**.

```jsx
<TabsContent value="terms">
  <>
    <div className="space-y-6 rounded-sm bg-white shadow-md py-5 px-5 border">
      <div className="flex items-center gap-3 mb-5 bg-[#09a371] py-3 px-3 rounded-sm shadow-lg text-white">
        <Banknote /> <h1>Terms</h1>
      </div>
      <CustomerTerms data={formValues} setData={setFormValues} />
      
      <h1>Invoice due date calculated by</h1>
      <TermsFormCustomer data={formValues} setData={setFormValues} />
    </div>
    <br />
    <div className="space-y-6 rounded-sm bg-white shadow-md py-5 px-5 border">
      <h3 className="flex items-center gap-3 mb-5 bg-[#09a371] py-3 px-3 rounded-sm shadow-lg text-white">
        <Receipt /> Finance
      </h3>
      <FinanceFields 
        data={formValues} 
        setData={setFormValues} 
        FinancialSummary={financialSummary} 
        TotalDeposit={totalDeposit} 
        CusRefAmount={customerRefundAmount} 
        TotalOutstanding={currentOutstandingBalance}
      />
    </div>
  </>
</TabsContent>
```

**Key Props Passed:**
- `financialSummary`: Summary data from backend
- `totalDeposit`: Total deposit amount
- `customerRefundAmount`: Total customer refund amount
- `currentOutstandingBalance`: Current outstanding balance

### Sub-Component 1: CustomerTerms
**Location:** `/mnt/c/Codes/Powersuite/psfront/src/pages/CustomerPage/CustomerFormFields/CustomerTerms.jsx`

This component displays editable credit term fields:
- Credit Limit (number input)
- Addon Credit Limit (number input)
- Credit From Date (date input)
- Credit To Date (date input)

**Key Features:**
- Fetches currencies and payment types from API
- Manages form values with React hooks
- Default currency: PKR (set automatically if available)

**State Management:**
```javascript
const [values, setValues] = useState({
  payment_type: data?.terms?.payment_type || null,
  deposit_amount: data?.terms?.deposit_amount || null,
  currency_id: data?.terms?.currency_id || "",
  receipt_no: data?.terms?.receipt_no || "",
  credit_limit: data?.terms?.credit_limit || null,
  share_credit_code: data?.terms?.share_credit_code || "",
  addon_credit_limit: data?.terms?.addon_credit_limit || null,
  credit_from_date: data?.terms?.credit_from_date || null,
  credit_to_date: data?.terms?.credit_to_date || null,
});
```

### Sub-Component 2: TermsFormCustomer
**Location:** `/mnt/c/Codes/Powersuite/psfront/src/components/CustomerForm/TermsFormCustomer.jsx`

This component handles invoice due date calculations:

**Key Fields:**
- Credit Days (number)
- Late Payment Charge (number)
- Credit Terms Type (select: 1-5)
- Credit Type (radio buttons):
  - Credit Days
  - BSP Payment Term
  - BSP PAYMENT +15 days
  - Statement Date
  - Weekly
- Weekly (select: Monday-Sunday, hidden by default)
- Last Day of the Month (checkbox, hidden by default)
- Other Due Date (date, hidden by default)

**Dynamic Field Visibility:**
Fields are conditionally shown/disabled based on the selected Credit Type:
- Weekly selector appears when "Weekly" is selected
- Last Day of Month and Other Due Date appear when "Statement Date" is selected

### Sub-Component 3: FinanceFields
**Location:** `/mnt/c/Codes/Powersuite/psfront/src/components/CustomerForm/FinanceFields.jsx`

Displays read-only financial calculation fields:
- **Total Invoices** - Sum of all customer invoices
- **Total Settlements** - Sum of all settlement payments
- **Total Deposit** - Sum of all customer deposits
- **Total Refund** - Sum of all customer refunds
- **Current Outstanding** - Calculated total outstanding balance

**Key Features:**
- All fields are read-only (readOnly attribute)
- Values are formatted with currency formatting (toLocaleString)
- Data updates via useEffect when financial summary changes
- State structure:
```javascript
const [values, setValues] = useState({
  ytd_turnover: parseFloat(data?.terms?.ytd_turnover) || 0,
  ly_turnover: parseFloat(data?.terms?.ly_turnover) || 0,
  current_outstanding: parseFloat(data?.terms?.current_outstanding) || 0,
  overdue_balance: parseFloat(data?.terms?.overdue_balance) || 0,
  credit_note_balance: parseFloat(data?.terms?.credit_note_balance) || 0,
  deposit_balance: parseFloat(data?.terms?.deposit_balance) || 0,
  overpayment_balance: parseFloat(data?.terms?.overpayment_balance) || 0,
  voucher_balance: parseFloat(data?.terms?.voucher_balance) || 0,
  uatp_rebate: parseFloat(data?.terms?.uatp_rebate) || 0,
  net_outstanding: parseFloat(data?.terms?.net_outstanding) || 0,
  total_invoices: parseFloat(data?.terms?.total_invoices) || 0,
  total_settlements: parseFloat(data?.terms?.total_settlements) || 0,
  total_deposit: parseFloat(data?.terms?.total_deposit) || 0,
  total_refund: parseFloat(data?.terms?.total_refund) || 0,
  total_outstanding: parseFloat(data?.terms?.total_outstanding) || 0,
});
```

---

## 2. API Endpoint

### Endpoint Definition
**Route:** `GET /customer/getCustomer/:id`
**Location:** `/mnt/c/Codes/Powersuite/psback/routes/customer.route.js` (line 44)
**Authentication:** JWT (required)
**Method:** GET

### API Call from Frontend
**Location:** `/mnt/c/Codes/Powersuite/psfront/src/api/customer.js` (lines 273-280)

```javascript
const getSingleCustomer = async (id) => {
  try {
    const data = await axios.get(`/customer/getCustomer/${id}`);
    return data;
  } catch (error) {
    throw error.response.data.message;
  }
};
```

### API Response Structure
The endpoint returns a JSON object containing:

```javascript
{
  customer: {
    // Customer data object with all profile information
    id,
    customer_number,
    customer_name,
    // ... all customer fields
    credit_limit,
    addon_credit_limit,
    credit_from_date,
    credit_to_date,
    // ... more fields
  },
  financialSummary: {
    totalInvoicesAmount,
    totalSettlementsAmount,
    customer_refund_amount,
    totalDepositAmount,
    outstandingBalance: (totalInvoices - refunds - deposits),
    invoiceBreakdown: {
      PartiallySettled: { count, amount },
      Printed: { count, amount },
      Settled: { count, amount }
    },
    settlementBreakdown: {
      Printed: { count, amount }
    },
    criteria: {
      invoices: "Description",
      settlements: "Description",
      deposits: "Description",
      refunds: "Description"
    },
    lastUpdated: ISO timestamp
  },
  orders: [ array of order summaries ]
}
```

---

## 3. Backend Controller & Business Logic

### Controller: getCustomer
**Location:** `/mnt/c/Codes/Powersuite/psback/controllers/customer.controller.js` (lines 1630-1898)

**Responsibility:** Orchestrates data collection and calculation of financial summaries

**Process Flow:**

1. **Fetch Customer Profile** (lines 1636-1669)
   - Retrieves customer record with associations:
     - customer_contact
     - customer_membership
     - customer_visa
     - cus_supplementary
     - branch
     - sales_id
     - fee_maintenance

2. **Fetch Orders** (lines 1681-1710)
   - Retrieves all orders for the customer
   - Includes nested data:
     - passengers
     - services with invoices
     - invoice details (discounts, taxes, settlements)
     - costs
     - documents

3. **Calculate Refund Amount** (lines 1712-1752)
   - Queries refund table with status "Printed"
   - Includes credit note associations
   - Filters for credit notes with doc_status "Printed"
   - **Total:** Sum of all credit note amounts

4. **Calculate Deposit Amount** (lines 1758-1770)
   - Queries customer_deposit table with status "Printed"
   - **Total:** Sum of current_amount field for all printed deposits

5. **Calculate Invoices & Settlements** (lines 1772-1858)
   - **Invoice Processing:**
     - Filters invoices with statuses: "Partially Settled", "Printed", "Settled"
     - Converts foreign currency to PKR using exchange rate
     - Maintains breakdown by invoice status
     - Uses total_price column from invoice table

   - **Settlement Processing:**
     - Processes receipt_settlement_invoices
     - Only counts settlements with status "Printed"
     - Tracks amount and count

6. **Build Financial Summary** (lines 1860-1881)
   ```javascript
   const customerFinancialSummary = {
     totalInvoicesAmount,           // Sum of all valid invoices (converted to PKR)
     totalSettlementsAmount,        // Sum of all printed settlements
     customer_refund_amount,        // Sum of printed credit notes
     invoiceBreakdown,              // Breakdown by status
     settlementBreakdown,           // Breakdown of settlements
     totalOrders,                   // Count of orders
     totalDeposits,                 // Count of deposits
     totalDepositAmount,            // Sum of deposit amounts
     FinalAmount,                   // invoices - refunds - deposits
     outstandingBalance,            // invoices - refunds - deposits
     criteria: { ... }              // Documentation of calculation method
   };
   ```

### Key Calculation Formula
**Outstanding Balance = Total Invoices - Customer Refunds - Total Deposits**

---

## 4. Supporting Service: Balance Calculator

### Service File
**Location:** `/mnt/c/Codes/Powersuite/psback/services/customer_balance_calculator.js`

This service provides reusable calculation functions used across multiple reports.

### Key Functions:

#### 1. calculateInvoiceAmount(invoice, invoiceTaxes, exchangeRate, serviceType, numberOfRooms)
Calculates complete invoice amount including:
- Base price
- Discounts (percentage-based)
- Rebates (percentage-based)
- Taxes
- Transaction fees
- SST (Service and Sales Tax)
- Multi-room multiplication for hotels
- Currency conversion

**Formula:**
```
subtotal = (basePrice - discount - rebate + taxes) * quantity
sstAmount = (transactionFee * sstPercent) / 100
totalAmount = (subtotal * roomMultiplier) + transactionFee + sstAmount
finalAmount = totalAmount * exchangeRate
```

#### 2. calculateDepositBalance(deposit, depositUsages, exchangeRate)
- Returns full deposit value (not remaining balance)
- Applies currency exchange rate

#### 3. calculateCreditNoteAmount(creditNote)
- Returns credit note refund_amount or amount field

#### 4. calculateReceiptAmount(receipt)
- Returns G/L account payment amounts only (filtered by controller)

#### 5. calculateCustomerBalance(params)
Main calculation function that:
- Calculates opening balance (if historical data provided)
- Sums period invoices, refunds, receipts, and deposits
- Computes net balance
- Returns detailed breakdown

---

## 5. Data Flow Diagram

### Profile Loading Sequence:
```
Frontend CreateCustomer Page
    ↓
    ├─→ useEffect on mount
    │   └─→ fetchDropdownsValues()
    │       └─→ Promise.all([getCountryCode, getBranches, getSalesId, etc])
    │
    └─→ useEffect when id param changes
        └─→ fetchSingleCustomer()
            └─→ getSingleCustomer(id)
                └─→ GET /customer/getCustomer/:id
                    └─→ Backend Controller: getCustomer()
                        ├─→ Query: customer record
                        ├─→ Query: orders with services/invoices
                        ├─→ Query: refunds (printed with credit notes)
                        ├─→ Query: customer_deposits (printed)
                        ├─→ Calculate: invoice totals (with PKR conversion)
                        ├─→ Calculate: settlement totals
                        └─→ Build: financialSummary
                            └─→ Return JSON response
                                └─→ Frontend receives response
                                    ├─→ setFormValues() with customer data
                                    ├─→ setFinancialSummary()
                                    ├─→ setTotalDeposit()
                                    ├─→ setCustomerRefundAmount()
                                    └─→ setCurrentOutstandingBalance()
```

### Terms Tab Display Sequence:
```
Terms Tab Content Rendered
    ↓
    ├─→ CustomerTerms Component
    │   ├─→ Display: Credit Limit input (editable)
    │   ├─→ Display: Addon Credit Limit input (editable)
    │   ├─→ Display: Credit From Date (editable)
    │   └─→ Display: Credit To Date (editable)
    │
    ├─→ TermsFormCustomer Component
    │   ├─→ Display: Credit Days input
    │   ├─→ Display: Late Payment Charge input
    │   ├─→ Display: Credit Terms Type select
    │   ├─→ Display: Credit Type radio buttons
    │   ├─→ Conditional Display: Weekly select (if "Weekly" selected)
    │   ├─→ Conditional Display: Last Day of Month (if "Statement Date" selected)
    │   └─→ Conditional Display: Other Due Date (if "Statement Date" selected)
    │
    └─→ FinanceFields Component
        └─→ Display (Read-only, formatted):
            ├─→ Total Invoices: {totalInvoicesAmount}
            ├─→ Total Settlements: {totalSettlementsAmount}
            ├─→ Total Deposit: {totalDepositAmount}
            ├─→ Total Refund: {customer_refund_amount}
            └─→ Current Outstanding: {outstandingBalance}
```

---

## 6. Key Files Summary

### Frontend Files:
| File | Purpose | Lines |
|------|---------|-------|
| `/psfront/src/pages/CustomerPage/CreateCustomer.jsx` | Main form component with tabs | 1073-1097 (Terms tab) |
| `/psfront/src/pages/CustomerPage/CustomerFormFields/CustomerTerms.jsx` | Credit terms input fields | 1-152 |
| `/psfront/src/components/CustomerForm/TermsFormCustomer.jsx` | Invoice due date calculation options | 1-181 |
| `/psfront/src/components/CustomerForm/FinanceFields.jsx` | Financial summary display (read-only) | 1-126 |
| `/psfront/src/api/customer.js` | Customer API client functions | 273-280 |

### Backend Files:
| File | Purpose | Lines |
|------|---------|-------|
| `/psback/routes/customer.route.js` | API endpoint definitions | 44 |
| `/psback/controllers/customer.controller.js` | Business logic for getCustomer | 1630-1898 |
| `/psback/services/customer_balance_calculator.js` | Reusable calculation functions | 1-234 |

---

## 7. Financial Calculation Details

### Included Components:
1. **Invoices** (Status: Partially Settled, Printed, Settled)
   - Converted to PKR for foreign currencies
   - Uses total_price column from invoices table
   - Includes tax, discount, rebate, transaction fee, SST

2. **Refunds** (Status: Printed credit notes)
   - Only credit notes counted (service refunds excluded)
   - Amount taken from credit note record

3. **Deposits** (Status: Printed)
   - Full deposit amount (not remaining balance)
   - Sums current_amount field

4. **Settlements** (Status: Printed)
   - Only G/L account payments (excluding deposits/credit notes)
   - Amount from receipt_settlement table

### Outstanding Balance Calculation:
```
Outstanding Balance = Total Invoices - Total Refunds - Total Deposits
                    = Σ(invoices) - Σ(credit_notes) - Σ(deposits)
```

---

## 8. State Management in CreateCustomer

The financial data is stored in parent component state:

```javascript
const [financialSummary, setFinancialSummary] = useState(null);
const [totalDeposit, setTotalDeposit] = useState(0);
const [customerRefundAmount, setCustomerRefundAmount] = useState(0);
const [currentOutstandingBalance, setCurrentOutstandingBalance] = useState(0);

// Populated in fetchSingleCustomer():
const FinancialSummary = res?.data?.financialSummary || {};
const TotalDeposit = res?.data?.financialSummary?.totalDepositAmount || 0;
const CusRefAmount = res?.data?.financialSummary?.customer_refund_amount || 0;
const TotalOutstanding = res?.data?.financialSummary?.outstandingBalance || 0;
```

These values are then passed as props to FinanceFields component for display.

---

## 9. Form Value Structure in CreateCustomer

The terms data in the main form state:

```javascript
terms: {
  addon_credit_limit: null,
  credit_to_date: null,
  credit_from_date: null,
  share_credit_code: null,
  payment_type: null,
  deposit_amount: null,
  credit_limit: null,
  receipt_no: null,
  currency_id: null,
  credit_days: null,
  default_payee: null,
  interface_account_no: null,
  gst_reg_no: null,
  service_provided: null,
  internet_address: null,
  trading_currency: null,
  credit_days_1: null,
  late_payment_charge: null,
  credit_term_type: null,
  date_type: null,
  weekly: null,
  other_due_date: null,
  // Financial summary fields (read-only):
  ytd_turnover: null,
  ly_turnover: null,
  current_outstanding: null,
  overdue_balance: null,
  credit_note_balance: null,
  deposit_balance: null,
  overpayment_balance: null,
  voucher_balance: null,
  uatp_rebate: null,
  net_outstanding: null,
  total_invoices: null,
  total_settlements: null,
  total_deposit: null,
  total_refund: null,
  total_outstanding: null,
}
```

