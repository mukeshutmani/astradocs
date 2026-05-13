# Receipt Settlement - Technical Documentation

**Version**: 1.0
**Date**: March 2026
**Author**: System Analysis
**Status**: Stable

---

## Overview

Receipt Settlement is the process of settling customer invoices by applying deposits, credit notes, G/L account entries, and/or immediate payments. It supports both full and partial settlements, grouped invoice handling, and foreign currency with FX gain/loss calculation. Upon submission, the system creates a receipt settlement document and updates invoice statuses accordingly.

**URL**: `/receipt/settlement`

---

## Technical Architecture

### Frontend

| File | Purpose |
|------|---------|
| `psfront/src/pages/RecieptPage/Reciept.jsx` | Main settlement page (~1910 lines) |
| `psfront/src/components/Receipt/FormOfPayment.jsx` | Payment method selection (currently hidden) |
| `psfront/src/components/Receipt/SettlementGLAccount.jsx` | G/L account entry component |
| `psfront/src/components/Receipt/OverPayForm.jsx` | Overpayment handling |
| `psfront/src/api/invoice.js` | `settleReceipt(data)` - POST `/invoice/settleReceipt` |
| `psfront/src/api/invoice.js` | `voidReceiptSettlement(receipt_number)` - PUT `/invoice/voidReceiptSettlement/{receipt_number}` |
| `psfront/src/api/invoice.js` | `updateReceiptStatus(receipt_number, status)` - PUT `/invoice/receiptStatusUpdate/{receipt_number}` |

### Backend

| File | Purpose |
|------|---------|
| `psback/routes/invoice.route.js` | Route: `POST /settleReceipt` |
| `psback/controllers/invoice.controller.js` | `settleReceipt()` (line ~1492) - main settlement logic |
| `psback/controllers/invoice.controller.js` | `setReceiptSettlementStatus()` - update status |
| `psback/controllers/invoice.controller.js` | `generateReceiptSettlement()` - HTML/PDF generation |
| `psback/views/pages/receiptSettlement.ejs` | Receipt settlement document template |
| `psback/services/journal.js` | `settlementPostingRules()` - journal entry generation |

### Database Models

| Model File | Table | Purpose |
|------------|-------|---------|
| `psback/models/Receipt/receipt_settlement.js` | `receipt_settlement` | Main settlement record |
| `psback/models/Receipt/receipt_settlement_invoice.js` | `receipt_settlement_invoice` | Links settlement to invoices |
| `psback/models/Receipt/receipt_settlement_deposit.js` | `receipt_settlement_deposit` | Links settlement to customer deposits |
| `psback/models/Receipt/receipt_settlement_payment.js` | `receipt_settlement_payment` | G/L account payments within settlement |
| `psback/models/Receipt/receipt_settlement_credit_note.js` | `receipt_settlement_credit_note` | Links settlement to credit notes |

---

## Page Layout & Sections

The settlement page has the following sections (top to bottom):

### 1. Search & Date Filters
- **Search input**: Filters invoices by invoice number, document number, customer name/number, order number, supplier reference, passenger name
- **From / To date**: Filters invoices by `invoice_date` range
- **Clear Filters**: Resets search and date filters

### 2. Settlement Header
- **Customer selector** (`LiveComboBox`): Selects customer, fetches invoices/deposits/credit notes on change
- **Branch Name**: Auto-populated read-only field from selected customer's branch
- **Clear Filters**: Resets customer and all related state
- **Select Date**: Settlement date (defaults to today, limited to `noOfDays` back, cannot be in the future)
- **Adjustment Date**: Optional adjustment date

### 3. Invoice Table
- **Columns**: Checkbox, Document Number, Date, Customer No, Customer Name, Status, Currency (PKR), Invoice Amount, Outstanding Amount, Pay Amount
- **Grouping**: Invoices are grouped by `invoice_number` via `useMemo`. Each group row shows aggregated amounts.
- **Select All**: Checkbox in header selects/deselects all filtered invoices
- **Pay Amount**: Editable when selected; defaults to full outstanding. Changing group pay amount distributes proportionally across invoices in the group.
- **Total row**: Shows sum of all selected invoice pay amounts

### 4. Deposits Section
- Visible only when a customer is selected
- **Date filters**: From/To date to filter deposits
- **Columns**: Checkbox, Deposit Number, Document (link), Your Ref, Date, Remark, Customer No, Customer Name, Currency, Available Amount, Amount (editable), Action button
- **Amount logic**: Typing an amount caps at the deposit's available balance (`current_amount`). Overpayment is validated at submit time.
- **Plus/Trash button**: Toggles between adding full available amount and removing deposit allocation

### 5. Credit Notes Section
- Visible only when a customer is selected
- **Columns**: Checkbox, Credit Note No, Date, Reason, Total Amount, Available Amount, Amount to Use (editable), Actions
- **Amount to Use**: Capped at `available_amount` (= `amount - used_amount`)
- **Toggle button**: Sets amount to full available or removes it

### 6. G/L Account Settlement Section
- **FX Gain/Loss**: Auto-calculated entry (amber box) when foreign currency invoices have exchange rate differences. Uses account `770100` (FX Gain/Loss).
- **Manual entries**: Add/remove G/L account rows via `SettlementGLAccount` component
- **Add G/L Account button**: Adds a new blank G/L account entry
- **Total G/L Account Amount**: Displayed at bottom

### 7. Balance Calculator
- **Left column**: Total Pay Amount, Total Deposit Used, Total Credit Note Used, Total G/L Account Used, Total Payment Amount, Total Amount Paid
- **Right column**: Balance Required, Available Deposits, Available Credit Notes, Status (BALANCED / PARTIAL PAYMENT / OVERPAYMENT)
- **Auto Balance button**: Distributes deposit + credit note amounts across selected invoices sequentially
- **Warnings**: Displayed for deposit overuse, credit note overuse, or overpayment

### 8. Form of Payment (Hidden)
- Currently wrapped in `<div className="hidden">` - not visible to users
- Supports up to 5 payment entries with types: bank, cash, card, cheque, etc.

### 9. Settlement Summary
- Original Invoice Total, Previously Paid, Outstanding Before/After This Payment
- Settlement Status: Full Settlement / Partial Settlement / Overpaid

### 10. Receipt Details & Submit
- **Remarks**: Textarea for settlement remarks
- **Submit button**: Dynamic label based on state:
  - "Select Invoices to Proceed" (no selection)
  - "Add Payment to Proceed" (insufficient payment)
  - "Reduce Payment Amount" (overpayment)
  - "Proceed with Partial Settlement" (underpayment)
  - "Proceed with Full Settlement" (balanced)

---

## Data Flow

### Frontend State

| State Variable | Type | Purpose |
|----------------|------|---------|
| `customerId` | number | Selected customer ID |
| `invoice` | array | All invoices for customer |
| `filteredInvoice` | array | Invoices after search/date filters |
| `selectedReciepts` | array | Selected invoices with computed PKR amounts |
| `deposits` | array | Customer deposits |
| `selectedDeposits` | array | Selected deposits |
| `depositValues` | array | `[{ id, amount, converted_amount }]` |
| `creditNotes` | array | Customer credit notes |
| `selectedCreditNotes` | array | Selected credit notes |
| `creditNoteValues` | array | `[{ id, amount, converted_amount }]` |
| `glAccounts` | array | G/L account entries (including auto FX) |
| `glAccountValues` | array | Validated G/L accounts with amounts |
| `payments` | array | Payment entries (up to 5) |
| `selectedDate` | string | Settlement date (YYYY-MM-DD) |
| `adjustmentDate` | string | Optional adjustment date |
| `data` | object | `{ remarks, internal_remarks, receipt_reference, billing_address }` |

### Invoice Selection Logic

When an invoice is selected (`selectReciept`):
1. `total_price` and `previousAmount` are read from the invoice
2. Exchange rate is determined: `invoice.exchange_rate` or latest currency rate, default 1
3. All amounts are converted to PKR:
   - `paymentAmount` = `(total_price - previousAmount) * exchangeRate` (outstanding in PKR)
   - `originalPrice` = `total_price * exchangeRate`
   - `previousAmount` = `previousAmount * exchangeRate`

### Calculation Functions

| Function | Logic |
|----------|-------|
| `requiredAmount()` | `invoicePayAmounts - deposits - creditNotes - glAccounts(excl. FX) - payments` (min 0) |
| `totalAmountPaid()` | `deposits + creditNotes + glAccounts(excl. FX) + payments` (FX Gain/Loss excluded as it's an accounting adjustment, not actual payment) |
| `getSettlementBalance()` | `totalAmountPaid - totalPayAmount` (positive = overpaid, uses user's chosen pay amounts) |
| `isPartialSettlementValid()` | Validates deposit/credit note usage, ensures `totalPaid > 0` and `totalPaid <= invoiceAmount` |
| `calculateFxGainLoss()` | Sums `(currentRate - invoiceRate) * foreignAmount` for non-PKR invoices |

---

## Backend Settlement Flow

### `settleReceipt()` - Step by Step

The entire operation runs inside a Sequelize transaction with row-level locking.

**Step 1 - Parse Invoice Groups**
- Extracts all invoice IDs from request (handles both single and grouped invoices via `invoice_ids` array)
- Builds `invoiceGroupMap` (maps each invoice ID to its group) and `groupPaymentInfo` (total payment per group)

**Step 2 - Load Invoices**
- Fetches invoices with: Service -> Order -> Branch, existing `receipt_settlement_invoice` records, `invoice_tax`, `currency_code` -> `currency`
- Uses `LOCK.UPDATE` for row-level locking

**Step 3 - Branch/Company Validation**
- Validates `req.user.company_code` matches the branch's `company_code`

**Step 4 - Permission Check**
- Calls `checkDocIssuePermission()` for document type "receipt"
- If immediate payments exist, also checks "deposit" permission

**Step 5 - Generate Receipt Number**
- Pattern: `{branchPrefix}{typeCode}{sequence}` where typeCode = `ST`
- Example: `50ST00000001`
- Gapless numbering: finds all existing settlements for the company, extracts used sequences, picks the next available

**Step 6 - Create Immediate Payment Deposits**
- For each payment with amount > 0, creates a `customer_deposit` record
- Deposit receipt number pattern: `{branchPrefix}DP{sequence}` (e.g., `50DP00000001`)
- Stores payment details: pay_type_id, currency_id, bank_id, card info, etc.

**Step 7 - Process Credit Notes**
- Validates each credit note: exists, not void, belongs to customer (direct or via refund), sufficient balance
- Builds `creditNotesToUse` array with validated amounts

**Step 8 - Process G/L Accounts**
- Validates each G/L account exists via `gl_settle_account` table
- Builds `glAccountsToUse` array with amounts and currency info

**Step 9 - Settlement Balance Validation**
- `totalAmountReceived = deposits + creditNotes + glAccounts`
- Prevents overpayment (balance > 0.01)
- Allows partial settlement (underpayment)
- Requires total > 0

**Step 10 - Create Settlement Record**
- Creates `receipt_settlement` with receipt_number, amount, user_id, adjustment_date, and receiptData fields

**Step 11 - Allocate Payments to Invoices**
- Calculates remaining balance for each invoice (total in PKR - previous settlements excluding voided)
- **Equal Division**: Within each group, payment is divided equally among invoices
  - `equalShare = floor(groupPayment / invoiceCount * 100) / 100`
  - Last invoice gets the rounding remainder
- **Carry-over**: If an invoice's share exceeds its remaining balance, the excess carries to the next invoice
- **Redistribution**: After initial pass, any remaining carry-over is redistributed to invoices that still have outstanding balance (up to 10 rounds)

**Step 12 - Create Invoice Settlement Records & Update Status**
- Creates `receipt_settlement_invoice` for each allocation
- Updates invoice status:
  - `"Settled"` if `newTotalPaid >= invoiceTotalInPKR`
  - `"Partially Settled"` if `newTotalPaid > 0`
  - `"Printed"` otherwise (no change)

**Step 13 - Apply Credit Notes**
- Creates `receipt_settlement_credit_note` records
- Updates credit note: `used_amount += amount`, status to "Settled" or "Partially Settled"
- Sets `settled_invoice_number` to first invoice number

**Step 14 - Apply G/L Accounts**
- Creates `receipt_settlement_payment` records with `customer_deposit_id = null`

**Step 15 - Apply Deposits**
- Validates sufficient balance on each deposit
- Updates `current_amount = current_amount - usedAmount`
- Updates deposit status: "Settled" (balance = 0), "Partially Settled" (balance < original)
- Creates `receipt_settlement_deposit` records
- Does NOT create `receipt_settlement_payment` for existing deposits (avoids duplicate journal entries)

**Step 16 - Return**
- Returns `{ status: "success", settlement }` with the created settlement record
- Frontend navigates to `/documents/{receipt_number}?type=settlement&status=Raised`

---

## Receipt Number Format

| Type | Pattern | Example |
|------|---------|---------|
| Settlement | `{branchPrefix}ST{8-digit seq}` | `50ST00000001` |
| Deposit (immediate payment) | `{branchPrefix}DP{8-digit seq}` | `50DP00000003` |

- `branchPrefix` = `branch.document_prefix` padded to 2 characters
- Sequence is gapless within the company (not just branch)

---

## Foreign Currency Handling

### FX Gain/Loss Calculation (Frontend)
- For each selected non-PKR invoice:
  - `invoiceExchangeRate` = rate stored on the invoice at creation time
  - `currentExchangeRate` = latest rate from currency table
  - `paymentAmountInForeign = paymentAmountInPKR / currentExchangeRate`
  - `fxDiff = (currentRate - invoiceRate) * foreignAmount`
- Positive = Gain, Negative = Loss
- Auto-added to G/L accounts using account `770100` (FX Gain/Loss)

### Exchange Rate in Backend
- Invoice total is stored in original currency (`total_price`)
- Converted to PKR using `invoice.exchange_rate` or latest currency rate
- All settlement amounts in `receipt_settlement_invoice` are stored in PKR

---

## Validation Rules

### Frontend Validations
| Rule | Error Message |
|------|--------------|
| No invoices selected | "Please select at least one invoice" |
| Payment validation errors | "Please fix the payment validation errors before proceeding" |
| Deposit exceeds available | "One or more deposit amounts exceed the available balance" |
| Credit note exceeds available | "One or more credit note amounts exceed the available balance" |
| Total paid > total available | "Total payment amount exceeds total available amount" |
| Overpayment (balance > 0.01) | "Overpayment of {amount}. Please adjust the payment amounts." |
| Total paid <= 0 | "Payment amount must be greater than zero." |
| Date in future | "Date cannot be in the future." |
| Date before min date | "Not match with system Date" |

### Backend Validations
| Rule | Status Code | Error |
|------|-------------|-------|
| Invoices not found | 400 | "Invoices not found" |
| Company code mismatch | 403 | "User company code does not match branch company code" |
| No doc_issue permission | 403 | "DOC_ISSUE_PERMISSION_DENIED" |
| Credit note not found | 400 | "Credit note with ID {id} not found" |
| Credit note is void | 400 | "Credit note {ref} is void and cannot be used" |
| Credit note wrong customer | 400 | "Credit note does not belong to this customer" |
| Credit note insufficient balance | 400 | "Requested amount exceeds available credit note balance" |
| G/L account not found | 400 | "G/L account with ID {id} not found" |
| Overpayment | 400 | "Overpayment not allowed. Excess amount: {amount}" |
| Zero settlement | 400 | "Settlement amount must be greater than zero" |
| Deposit insufficient balance | 400 | "Insufficient deposit balance" |

---

## API Endpoints

### Settlement
| Method | Endpoint | Controller | Purpose |
|--------|----------|------------|---------|
| POST | `/api/invoice/settleReceipt` | `settleReceipt` | Create new receipt settlement |
| PUT | `/api/invoice/receiptStatusUpdate/:receipt_number` | `setReceiptSettlementStatus` | Update settlement status |
| PUT | `/api/invoice/voidReceiptSettlement/:receipt_number` | `voidReceiptSettlement` | Void a settlement |
| GET | `/api/invoice/receiptSettlement/:receipt_number` | `generateReceiptSettlement` | Generate settlement HTML |

### Supporting Data
| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/customer/getCustomers` | Fetch customers for selection |
| GET | `/api/customer/:id/orders` | Fetch customer invoices |
| GET | `/api/customer/:id/credit-notes` | Fetch customer credit notes |
| GET | `/api/deposit/customer/:id` | Fetch customer deposits |
| GET | `/api/data/gl-settlement-account` | Fetch G/L settlement accounts |
| GET | `/api/data/systemDate` | Fetch system date (noOfDays) |

---

## Manual JE Integration (Listing)

The settlement-page invoice listing (`getOrdersByCustomerId` in `psback/controllers/customer.controller.js`) reflects Manual JE settlements as well as receipt-settlement records. This keeps the listing in sync with `invoice.status` (which is already updated by `recalculateInvoiceStatusByNumber` when a Manual JE is created/edited/voided — see `docs/JOURNAL_ENTRIES_PLAN.md` item 12).

1. After invoices are grouped by `invoice_number`/`document_number`, the controller calls `sumManualJeAdjustment(invoice_number)` once per unique invoice number (from `services/manualJeAdjustment.js`).
2. The returned PKR adjustment is divided by the group's `exchangeRate` and added to `previousAmount` (group's paid-so-far, in source currency).
3. Outstanding (`total_price - previousAmount`) then subtracts the JE; status flips to `Partially Settled` when JE > 0 and to `Settled` (filtered out of the listing) when JE fully clears the invoice.
4. Manual JE rows that belong to a `Void` batch or are themselves `VOID REVERSAL -` rows are excluded by `sumManualJeAdjustment`'s filter, so a voided JE nets the invoice back to its prior state automatically.

This complements the create/void paths in `invoice.controller.js`, which already add `sumManualJeAdjustment` to the paid total when computing settlement balance.

---

## Settlement Types

| Type | Condition | Submit Label |
|------|-----------|--------------|
| Full Settlement | `totalPaid == totalOutstanding` | "Proceed with Full Settlement" |
| Partial Settlement | `0 < totalPaid < totalOutstanding` | "Proceed with Partial Settlement" |
| Overpayment | `totalPaid > totalOutstanding` | "Reduce Payment Amount" (blocked) |

---

## Keyboard Shortcuts

- **Ctrl+S**: Triggers submit button click (saves/submits the settlement)

---

## Related Modules

- **Advanced Deposit** (`/receipt/advanced-deposit`): Creates customer deposits independently
- **Daily Settlement Report** (`/dailySettlementReport`): Report of all receipt settlements
- **Payment Settlement** (`/paymentSettlementReport`): Report of payment settlements
- **Documents**: Settlement documents viewable at `/documents/{receipt_number}?type=settlement`
- **Journal Entries**: Settlement posting rules in `psback/services/journal.js` (prefix "RS")
