# Supplier Payment Settlement - Technical Documentation

**Version**: 1.0
**Date**: March 2026
**Author**: System Analysis
**Status**: Stable

---

## Overview

The Supplier Payment Settlement module allows users to settle outstanding cost documents (XO documents) against suppliers. Users select a supplier, choose which cost documents to pay, and apply payment through multiple methods: direct payment (Form of Payment), supplier deposits (advance payments), overpayments from previous settlements, and debit notes. The system generates a payment settlement document (KHPY) upon successful submission.

---

## Technical Architecture

### Frontend Component

**File**: `psfront/src/pages/PaymentsPage/Payments.jsx`
**Route**: `/payments` (Sidebar: Payments → Supplier Payments)

### Backend Controller

**File**: `psback/controllers/payment.controller.js`
**Function**: `settlePayment` (line ~250)
**Route**: `POST /api/payment/settle`

### Payment Document Template

**File**: `psback/views/pages/paymentSettlement.ejs`

---

## Page Layout & Sections

### 1. Supplier Selection

- Combobox dropdown to select a supplier by number/name
- On selection, fetches all cost documents, deposits, overpayments, and debit notes for that supplier
- Changing supplier resets all selections and payment data

### 2. Date Controls

- **Select Date**: Payment date (default: today). Limited to last N days (configurable via system_dates). Cannot be in the future.
- **Adjustment Date**: Optional. Used for journal entry posting date adjustments.

### 3. Cost Documents Table (XO Documents)

Displays outstanding cost documents grouped by document number.

**Columns**: Document No., Date, Supplier No., Order No., Status, Type, Total Amount, Outstanding Amount, Payable

**Key behaviors**:
- Costs are fetched via `getPaymentCostsBySupplierId` API
- Only "Printed" and "Partially Paid" costs are shown (not "Paid" or "Void")
- **Grouping**: Multiple costs under the same document number (e.g., KHXO00000024 with 3 items) are displayed as a single row
- **Payable column**: Editable input. When user enters an amount, it distributes proportionally among costs in the group. Last cost gets the remainder to avoid floating point rounding issues.
- **Checkbox**: Select/deselect cost documents for settlement
- **Outstanding Amount**: Total Cost Amount (PKR) minus previous non-voided payments

**Cost Amount Calculation (PKR)**:
```
publishedRate = cost.published_rate
commissionAmount = publishedRate × (commission% / 100)
netRate = publishedRate - commissionAmount
netRateWithExtra = netRate + free_of_cost (extra charges)
taxAmount = sum(cost_taxes.tax_amount)
whtAmount = commissionAmount × (sst% / 100)
costPerUnit = netRateWithExtra + taxAmount + whtAmount
totalCostLocal = costPerUnit × quantity
totalCostPKR = totalCostLocal × exchange_rate
outstandingAmount = totalCostPKR - sum(previous non-voided payments)
```

### 4. Supplier Deposits (Advance Payments)

Shows advance payments available for the selected supplier.

**Columns**: Payment Number, Date, Supplier No., Supplier Name, Available Amount, Amount to Use, Status

**Key behaviors**:
- Fetched via `getSupplierDepositBySupplierId` API with optional date filter (From Date / To Date)
- Selecting a deposit auto-populates Amount to Use with available amount
- `converted_amount = amount × exchange_rate` (for foreign currency deposits)
- Only "Printed" status deposits are usable

### 5. Overpayments

Shows previous payment settlements that resulted in overpayments for this supplier.

**Columns**: Payment Number, Date, Supplier No., Supplier Name, Available Amount, Amount to Use, Status

**Key behaviors**:
- Fetched from `getDocuments("payment")` API, filtered by supplier and `overpay_amount > 0`
- **Void documents are excluded** from the list (`status !== 'Void'`)
- Available amount = sum of `payment_settlement_payment.overpay_amount` for the settlement

### 6. Debit Notes

Shows available debit notes for the selected supplier.

**Columns**: Debit Note No., Date, Reason, Currency, Total Amount, Available Amount, Amount to Use, Actions

**Key behaviors**:
- Fetched via `listDebitNotes` API with `status: 'Printed'` filter
- **Currency column**: Shows conversion direction (e.g., "USD to PKR", "AUD to PKR", or just "PKR")
- **Total Amount & Available Amount**: Displayed in PKR (converted using `debitNote.exchange_rate`)
- **Amount to Use**: Input in PKR. Auto-populated with PKR value on selection. Backend stores both foreign currency (`amount`) and PKR (`converted_amount`).
- **Available Amount**: `(debitNote.amount - debitNote.used_amount) × exchange_rate`
- **Select All**: Checkbox in header selects all debit notes with PKR converted amounts

### 7. Remarks

Free-text input for payment remarks. Stored in `payment_settlement.remarks`.

### 8. Form of Payment

Up to 5 payment methods can be added.

**Fields per payment**:
- Payment Method (pay_type_id): Cash, Cheque, Card, Wire Transfer, etc.
- Currency: Default PKR
- Amount: Payment amount in selected currency
- Additional fields based on payment method: Bank, Cheque No., Card Type, Card No., Voucher No., GL Account, Account No., Cash Box, Client Reference

### 9. Overpayment Details (Conditional)

Appears when total paid > total outstanding.

**Fields**:
- Overpayment Amount (auto-calculated)
- GL Account for overpayment (required)

### 10. Settlement Summary

Shows real-time calculation:
- Total Settlement Amount (sum of payable amounts)
- Total Deposits Used
- Total Overpayments Used
- Total Debit Notes Used
- Total Form of Payment
- Total Payment (all methods combined)
- Remaining Balance / Overpayment status
- Settlement Status: Full Settlement ✓, Partial Settlement, Overpayment

---

## Auto-Sync Payable Distribution

When payment methods (Form of Payment, Deposits, Overpayments, Debit Notes) change, the Payable column auto-distributes the total across selected costs proportionally based on outstanding amounts.

```
totalPayable = totalFormOfPayment + totalDeposits + totalOverpayments + totalDebitNotes

For each selected cost:
  ratio = cost.outstandingAmount / totalOutstanding
  payableAmount = totalPayable × ratio (rounded to 2 decimals)
  
Last cost gets remainder to prevent rounding errors.
```

**Dependencies**: `[totalPaymentAmount, depositValues, overpaymentValues, debitNoteValues, selectedCostIds]`

---

## Payment Submission

### Frontend Validations
1. At least one cost document must be selected
2. At least one payment method (direct payment, deposit, overpayment, or debit note) must be used
3. Total payment must equal total settlement amount (with overpayment tolerance)
4. Active payment methods must have required fields (pay_type_id, currency_id)
5. Debit note amounts cannot exceed available balance
6. If overpayment: GL account must be selected and overpay amount must match calculated amount

### Payload (`POST /api/payment/settle`)
```javascript
{
  cost_ids: [5524, 5525, 5526, 5527],
  payments: [{ pay_type_id, currency_id, amount, bank_id, ... }],
  supplier_id: 123,
  created_at: "2026-03-31",
  remarks: "...",
  adjustment_date: null,
  payableAmounts: [{ cost_id, amount, original_amount }],
  deposits: [{ id, amount, converted_amount }],
  debitNotes: [{ id, amount, converted_amount }],
  overpayments_used: [{ id, amount, payment_settlement_id }],
  overpay_amount: 0,
  overpay_gl_account_id: null,
}
```

### Backend Processing (`settlePayment`)
1. Generate payment number (branch code + "PY" + sequential number)
2. Create `payment_settlement` record
3. Create `payment_settlement_cost` records (one per cost, with payable amounts)
4. Create `payment_settlement_payment` records (Form of Payment entries)
5. Process supplier deposits: create `payment_settlement_deposit`, update deposit `current_amount`
6. Process debit notes: create `payment_settlement_debit_note` (with `exchange_rate` and `base_amount`), update debit note `used_amount` and `doc_status`
7. Process overpayments: create `payment_settlement_overpayment`, link source payment
8. Handle new overpayment: create overpayment payment record with GL account
9. Update each cost's status:
   - `totalPaid >= totalCostAmount (including exchange_rate)` → "Paid"
   - `totalPaid > 0` → "Partially Paid"
   - Otherwise → "Printed"
10. Render payment settlement document (EJS template) and upload PDF

---

## Payment Document (EJS Template)

**File**: `psback/views/pages/paymentSettlement.ejs`

### Sections:
1. **Header**: Company name, address, pay to details, voucher number, date, branch, prepared by
2. **Payment Method Details**: Each form of payment with method type, bank, cheque details
3. **Amount**: Total settlement amount in PKR with words
4. **Debit Notes Used** (if any): Table showing debit note no., description, previous balance (PKR), amount used (PKR), remaining balance (PKR). All amounts converted to PKR using exchange rate.
5. **Overpayments Used** (if any): Similar table for overpayment sources
6. **Cost Documents Table**: Grouped by document number. Columns: Document No., Doc Ref., Supp Inv No., Date, Ref., Previous Balance, Payment Amount, Remaining Amount. Grand total row with amount in words.
7. **Overpayment Section** (if applicable): GL account details for new overpayment

---

## Currency Handling (Important)

### Foreign Currency Costs
- Cost amounts are stored in local currency and converted to PKR using `cost.exchange_rate`
- All payable amounts are in PKR
- `totalCostPKR = costPerUnit × quantity × exchange_rate`

### Foreign Currency Debit Notes
- Debit note `amount` is in foreign currency, `amount_base` is in PKR
- `exchange_rate` stored on the debit note record
- Display: "USD to PKR" (or "PKR" if already PKR)
- Amount to Use input is in PKR; backend stores both foreign (`amount`) and PKR (`converted_amount`)
- `payment_settlement_debit_note` stores `exchange_rate` and `base_amount` for proper display
- Older records without `base_amount` fall back to `amount × debitNote.exchange_rate`

### Cost Status Calculation (All 3 Places)
When determining Paid/Partially Paid status, the total cost must include exchange rate:
```
totalCostAmount = costPerUnit × quantity × exchange_rate
```
This applies in: settlement creation, void reversal, and general status recalculation.

---

## Known Bugs Fixed (March 2026)

1. **Debit note amounts showed in foreign currency instead of PKR** — Fixed display and conversion logic
2. **Void overpayment documents showed in overpayments list** — Added `status !== 'Void'` filter
3. **Cost status incorrectly set to "Paid" for foreign currency costs** — Exchange rate was missing from cost total calculation in void logic (3 places fixed)
4. **Floating point rounding (.01 errors) in payable distribution** — Fixed with `Math.round(x * 100) / 100` and last-item-gets-remainder pattern
5. **Payment document showed individual costs instead of grouped by document** — Added grouping by document_number in EJS template

---

## Order Page: Receipt/Payment Tab

**File**: `psfront/src/pages/OrderDetail.jsx` (line ~3665)

Shows all payment and receipt history for an order's services.

**Key behaviors**:
- Collects settlements from all services, then groups payments by `payment_number`
- **Void settlements are excluded** from the display
- Non-payment items (receipts, deposits, overpayments) also exclude void status
- Columns: Type, Document No., Paid By/Paid To, Status, Date, Amount, Remaining, FOP, Issued By

---

## Database Tables

| Table | Purpose |
|-------|---------|
| `payment_settlements` | Main settlement record (payment_number, supplier_id, amount, status, remarks) |
| `payment_settlement_costs` | Links settlement to costs (cost_id, amount per cost) |
| `payment_settlement_payments` | Form of payment details (pay_type, currency, amount, bank, cheque, card, etc.) |
| `payment_settlement_deposits` | Supplier deposit usage (deposit_id, amount) |
| `payment_settlement_debit_notes` | Debit note usage (debit_note_id, amount, exchange_rate, base_amount, currency_code) |
| `payment_settlement_overpayments` | Overpayment usage from previous settlements |
| `costs` | Cost documents with status tracking (Printed, Partially Paid, Paid) |
| `debit_notes` | Debit notes (amount, used_amount, exchange_rate, doc_status) |
| `supplier_deposits` | Advance payments (current_amount tracks available balance) |

---

## Code References

| File | Purpose |
|------|---------|
| `psfront/src/pages/PaymentsPage/Payments.jsx` | Frontend UI |
| `psback/controllers/payment.controller.js` | Backend settlement, void, status logic |
| `psback/views/pages/paymentSettlement.ejs` | Payment document PDF template |
| `psback/routes/payment.route.js` | Route definitions |
| `psfront/src/api/payment.js` | API client |
| `psfront/src/pages/OrderDetail.jsx` (line ~3665) | Order Receipt/Payment tab |
| `psback/models/Payment/payment_settlement.js` | Settlement model |
| `psback/models/Payment/payment_settlement_payment.js` | Payment method model |
| `psback/models/Payment/payment_settlement_debit_note.js` | Debit note usage model |
| `psback/models/Payment/payment_settlement_overpayment.js` | Overpayment usage model |

---

**Document End**
