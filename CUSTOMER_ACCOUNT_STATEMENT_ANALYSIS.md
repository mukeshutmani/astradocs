# Customer Account Statement Report - Comprehensive Analysis

## Executive Summary

The **Customer Account Statement Report** is a comprehensive financial report in the PowerSuite travel booking system that provides a detailed breakdown of each customer's account activity, including invoices, refunds, payments, deposits, and overall account balance. The report can be generated in both PDF and Excel formats.

---

## 1. Report Purpose & Overview

### What the Report Does
The Customer Account Statement Report:
- Displays complete financial transaction history for one or more customers
- Shows all invoices (Flight, Hotel, General/Other services)
- Shows ONLY credit notes (not service refunds) in the refund section
- Shows all payments/receipts received from customers
- Shows all customer deposits and remaining balances
- Calculates opening balance (transactions before the report date range)
- Calculates net balance (opening + period transactions)
- Provides summary calculations for account reconciliation
- Supports currency conversion to PKR (Pakistani Rupee)

**IMPORTANT**: Service refunds (from the refunds table) are completely excluded from this report. Only credit notes are included in refund calculations and displays.

### Business Use Case
- **Customer reconciliation**: Verify customer account balances with payment history
- **Financial reporting**: Track customer receivables
- **Deposit management**: Monitor customer deposit usage
- **Multi-period analysis**: Compare account activity across different time periods

---

## 2. Input Parameters & Filters

### Frontend Filter Interface (React Component)
**Location**: `/mnt/c/Codes/Powersuite/psfront/src/pages/Report/CustomerAccountStatement.jsx`

#### Available Filters

1. **Customer Filter** (Dropdown + Conditional Input)
   - `isNotBlank`: All customers with records (default)
   - `isBlank`: Customers with no records
   - `isEqual`: Single specific customer (requires customer selection via LiveComboBox)
   - `between`: Range of customers by ID (requires start and end customer selection)

2. **Service Date Filter** (Dropdown + Conditional Input)
   - `blank`: No date filtering (default)
   - `=`: Equal to a specific date
   - `<`: Less than (before) a date
   - `<=`: Less than or equal to a date
   - `>`: Greater than (after) a date
   - `>=`: Greater than or equal to a date
   - `<>`: Not equal to a date
   - `between`: Date range (requires start and end dates)

3. **Output Format**
   - `pdf`: Generate PDF report
   - `excel`: Generate Excel spreadsheet (XLSX format)

4. **Adjustment Date Mode (Posted to Ledger)** — NEW
   - Checkbox: `adjustmentDateMode`
   - When enabled: uses `adjustment_date` for posted documents, `created_at` for unposted
   - Affects: Deposits, Receipts, Credit Notes, Payment Settlements, and historical data
   - Displayed in header as "Adjustment Date: Posted to Ledger"

6. **Hide Column** — NEW (2026-06-09)
   - Single multi-select dropdown labelled **Hide Column** (`hideColumns` array)
   - Options: **Discount, Rebate, T.Fee, SST** (internal keys: `discount`, `rebate`, `transactionFee`, `sst`)
   - The user can tick one or more; those columns are **hidden** from the Ticket Booking and Refund Ticket Booking tables (PDF + Excel). Nothing selected = report unchanged.
   - **Display-only**: hiding a column does NOT change the per-row **Net** or any total — the hidden amounts are still included in the math (Net is precomputed and the section totals sum `Net`). The "Total" row colspan shrinks by the number of hidden columns so the table stays aligned.
   - **T.Fee folds into Taxes when hidden (2026-06-17)**: when **T.Fee** (`transactionFee`) is hidden, its amount is added into the **Taxes** column for each Ticket Booking row (e.g. Taxes 200 + T.Fee 50 → Taxes shows 250). This is display-only — **Net**, **Total Sales**, and **Net Balance** are unchanged. Applied at the ticket row build in `getCustomerAccountStatementReport` (`taxes = (taxesPerPassenger + (hidden ? transactionFeePerPassenger : 0)) * exchangeRate`), so both PDF and Excel (including the Excel Taxes total) reflect it. Only affects the Ticket Bookings table (the Refund-Ticket table is not populated in this report).
   - UI: the trigger is an **auto-width** dropdown button (`w-auto`, `min-w-[160px]`, `max-w-full`) that grows with the number of selected labels and stays within the card; built with `DropdownMenuCheckboxItem` (menu stays open on toggle via `onSelect` preventDefault).

7. **Hide Opening Balance** — NEW (2026-06-30)
   - Checkbox labelled **Opening Balance** (`hideOpeningBalance`, default OFF), placed to the **right of the Hide Column** dropdown. Available for **all companies**.
   - When ON: the **Opening Balance B/F** line is removed from the Summary (PDF + Excel) **and** removed from the **Net Balance** math (in the controller `openingBalance` is set to `0` before the Net Balance is computed), so the visible summary lines still reconcile to the Net Balance.
   - When OFF (default): report unchanged.
   - Only **Opening Balance B/F** is affected — the separate **Add Opening Invoices** line (in-period imported opening invoices) is untouched.

5. **Include Raised Invoices** — NEW
   - Checkbox: `includeRaised` (default OFF)
   - When OFF (default): only `Printed`, `Settled`, `Partially Settled` invoices appear (unchanged behaviour)
   - When ON: un-printed `Raised` invoices are also pulled in and counted in **Total Sales** and **Net Balance** (and in **Opening Balance B/F** for Raised invoices dated before the period, so reconciliation holds)
   - **Un-numbered Raised drafts are skipped** (2026-06-16): a Raised invoice whose `invoice_number` is still blank (NULL or empty) is an unfinished draft, so it is excluded from the report and from the Opening Balance. Numbered Raised invoices still appear. Applied in both the period invoice query and `customerOpeningBalance.service.js`.
   - Each booking row shows a **Status** column (after Invoice) with the document's actual status (Raised / Printed / Settled / Partially Settled), in both PDF and Excel
   - Caveat: a Raised invoice has no frozen exchange rate (saved only at print time), so foreign-currency Raised invoices use the **live** rate. PKR invoices are unaffected.

### Request Body Structure
```javascript
{
  customerFilter: "isNotBlank" | "isBlank" | "isEqual" | "between",
  customer_id: number,              // For isEqual filter
  customer_idStart: number,          // For between filter
  customer_idEnd: number,            // For between filter
  dateFilter: "blank" | "=" | "<" | "<=" | ">" | ">=" | "<>" | "between",
  startDate: "YYYY-MM-DD",           // Required if dateFilter is not "blank"
  endDate: "YYYY-MM-DD",             // Required if dateFilter is "between"
  adjustmentDateMode: boolean,       // When true, use adjustment_date for posted docs
  includeRaised: boolean,            // When true, also include un-printed "Raised" invoices
  hideColumns: string[],             // Columns to hide on Ticket tables: any of 'discount','rebate','transactionFee','sst' (display only)
  hideOpeningBalance: boolean,       // When true, hide Opening Balance B/F and remove it from Net Balance
  type: "pdf" | "excel"              // Output format
}
```

---

## 3. Database Tables & Models Queried

### Core Models Involved

1. **customer** (Primary)
   - Fields: `id`, `customer_name`, `customer_number`, `address`, `sales_tax_number`
   - Relationship: One-to-many with Orders, Receipts, Deposits, Credit Notes

2. **user** (Filter by company)
   - Fields: `id`, `company_code`
   - Purpose: Multi-tenancy - ensures only company's customers are included

3. **order**
   - Fields: `id`, `order_number`, `customer_id`
   - Purpose: Groups services/invoices

4. **service** (Multiple types)
   - Fields: `id`, `service_type_id`, `order_id`
   - Related models:
     - `service_flight`: Flight details (departure city, arrival city, class)
     - `service_hotel`: Hotel details (hotel name, check-in, check-out, rooms, nights)
     - `service_type`: Service category (Flight, Hotel, Tour, Cruise, etc.)
     - `service_passenger`: Passenger information

5. **invoice**
   - Fields: `id`, `invoice_number`, `invoice_date`, `price`, `quantity`, `discount`, `rebate`, `transaction_fee`, `sst`, `status`
   - Status filter: Only "Printed", "Settled", "Partially Settled"
   - Relationships: Has `invoice_tax`, `invoice_discount`, `receipt_settlement_invoice`

6. **invoice_tax**
   - Fields: `tax_amount`
   - Purpose: Additional tax charges per invoice

7. **invoice_discount** (Referenced but not directly queried)
   - Fields: Related to discount information

8. **refund** (NO LONGER USED IN THIS REPORT)
   - Service refunds are completely excluded from Customer Account Statement
   - Only credit notes are used for refund tracking

9. **receipt_settlement** (Payment Receipts)
   - Fields: `id`, `receipt_number`, `receipt_reference`, `amount`, `payment_method`, `remarks`, `created_at`, `status`
   - Status filter: Excludes "Void" receipts
   - Relationships:
     - `receipt_settlement_invoice`: Links receipts to specific invoices
     - `receipt_settlement_payment`: Payment method details
     - `receipt_settlement_credit_note`: Credit notes applied to receipt
     - `receipt_settlement_deposit`: Deposits used in receipt settlement

10. **receipt_settlement_invoice**
    - Links receipts to invoices with amount applied

11. **receipt_settlement_payment**
    - Fields: `amount`, `currency_code`, `exchange_rate`, `base_amount`, `remarks`
    - Relationships: `gl_settle_account` (GL account info), `customer_deposit` (if deposit used)

12. **receipt_settlement_credit_note**
    - Fields: `amount`, `currency_code`, `exchange_rate`, `base_amount`, `remarks`
    - Purpose: Track credit note application to receipts

13. **receipt_settlement_deposit**
    - Fields: `amount`
    - Purpose: Track which deposits were used in receipt settlement

14. **customer_deposit**
    - Fields: `id`, `receipt_number`, `amount`, `description`, `reference`, `check_number`, `created_at`, `status`
    - Status filter: Excludes "Void" deposits
    - Relationships:
      - `pay_type_form`: Payment type (Check, Cash, Transfer, etc.)
      - `chart_of_account`: GL account for the deposit
      - `currency_code`: Currency information

15. **chart_of_account**
    - Fields: `key_account`, `description`
    - Purpose: GL account details for payment methods

16. **currency_code** & **currency**
    - Purpose: Currency conversion to PKR
    - Filter: Only includes exchanges to "PKR"
    - Fields: `exchange_rate`

17. **credit_note**
    - Fields: `id`, `doc_no`, `doc_date`, `doc_status`, `refund_amount`, `amount`, `to`, `reference`, `remarks`, `customer_id`
    - Status filter: Excludes "Void" credit notes
    - Purpose: Track credit memos against customer accounts

18. **report**
    - Fields: `user_id`, `file_type`, `report_number`, `report_type`, `created_at`, `updated_at`
    - Purpose: Store report generation record

19. **document**
    - Fields: `document_type`, `status`
    - Filter: Only "invoice" type documents

---

## 4. Key Calculations & Business Logic

### 4.1 Opening Balance Calculation

**Purpose**: Calculate customer's account balance as of the start date (before the report period)

> **Note (2026-05-19)**: This calculation now lives in the shared helper
> `psback/services/customerOpeningBalance.service.js` (`getCustomerOpeningBalances`).
> It was extracted verbatim from this controller — no behaviour change — so the
> **Customer Position Report** can call the same helper and produce the identical
> Opening Balance B/F. Both reports now share one source of truth.
>
> **Fix (2026-05-20) — frozen exchange rate missing in historical attributes**:
> The historical invoice include was loading invoice attributes but `exchange_rate`
> was NOT in the list, so `inv.exchange_rate` was always `undefined`, the
> `storedInvRate > 1` check failed, and the helper fell back to the **live**
> currency-table rate instead of the **frozen** rate from print time. Any
> foreign-currency invoice whose live rate differed from its frozen rate was
> valued incorrectly in the opening balance — so the previous period's Net
> Balance didn't carry forward to the next period's Opening Balance B/F.
>
> Example: QKCOMP, invoice TTIN00000025 (Hotel, 1,650 AED). Frozen rate = 78.42
> → period values it at 129,393 PKR. Live rate = 76.45 → helper valued it at
> 126,142.50 PKR. Gap = 3,250.50, which surfaced as the missing 3,250.50 between
> March's Net Balance and April's Opening Balance B/F.
>
> **Fix**: added `'exchange_rate'` to the `attributes` list of the historical
> `db.invoice` include in `customerOpeningBalance.service.js`. Same logic and
> formula — only the missing attribute was added.

**Formula**:
```
Opening Balance = Historical Invoices
               - Historical Receipts (GL account only)
               - Historical Credit Notes
               - Historical Deposits (full values)
               + Historical Payments (payments TO customer via credit notes)
               + Historical JV Debit (manual JE debits)
               - Historical JV Credit (manual JE credits)
```

**Components**:

#### Historical Invoices (transactions before startDate):
```javascript
invoiceAmount = {
    basePrice = invoice.price
    discount = (basePrice * invoice.discount%) 
    rebate = (basePrice * invoice.rebate%)
    unitPriceBeforeTax = basePrice - discount - rebate
    taxes = SUM(invoice_taxes[].tax_amount)
    unitPriceWithTax = unitPriceBeforeTax + taxes
    quantity = invoice.quantity || 1
    subtotal = unitPriceWithTax * quantity
    transactionFee = invoice.transaction_fee || 0
    sstPercent = invoice.sst || 0
    sstAmount = (transactionFee * sstPercent) / 100
    
    // Special handling for hotels
    numberOfRooms = (serviceType === "hotel") ? service_hotel.no_of_rooms : 1
    
    totalAmount = (subtotal * numberOfRooms) + transactionFee + sstAmount
    
    // Apply currency conversion
    return totalAmount * exchangeRate
}
```

#### Historical Refunds (Credit Notes Only):
```javascript
// ONLY credit notes are counted - service refunds are completely excluded
creditNoteAmount = credit_note.refund_amount || credit_note.amount
```

#### Historical Deposits:
```javascript
// Use full deposit values
depositTotal = deposit.amount * exchangeRate
```

#### Historical Receipts (G/L Account Payments Only):
```javascript
// Only count G/L account payment portions
receiptsTotal = 0
for each receipt_settlement:
    for each receipt_settlement_payment:
        if (has_gl_account && !is_deposit):
            receiptsTotal += payment.base_amount
```

#### Historical Credit Notes:
```javascript
creditNoteAmount = credit_note.refund_amount || credit_note.amount
```

### 4.2 Period Totals Calculation

For transactions within the report date range:

#### Period Invoices:
- Same calculation as historical invoices, but filtered by invoice_date between startDate and endDate

#### Period Refunds (Credit Notes Only):
- Only credit notes are calculated, filtered by doc_date between startDate and endDate
- Service refunds are completely excluded from this report

#### Period Receipts (G/L Account Payments Only):
- Only count G/L account payment portions from receipt_settlement_payments
- Exclude deposits and credit note portions

#### Period Deposits:
- Use full deposit values (not remaining balance)

### 4.3 Net Balance Calculation

```javascript
netBalance = openingBalance
           + periodInvoices (totalSales)
           - periodRefunds (credit notes only)
           - periodReceipts (GL account payments + deposits combined)
           + periodPayments (payments TO customer via credit notes)
           + periodJvDebit (manual JE debit entries)
           - periodJvCredit (manual JE credit entries)
```

Where `periodReceipts = lessReceipts + lessDeposits` (receipts and deposits are combined in the "LESS: Receipts" line).

### 4.4 Currency Conversion

All amounts are converted to PKR using:
```javascript
amountInPKR = amount * exchangeRate
```

**Invoice Exchange Rate Priority (Frozen Rate)**:
For invoices, the exchange rate is determined in this order:
1. **Stored rate on invoice** (`invoice.exchange_rate`) — frozen at the time the invoice was printed. This ensures that changing the exchange rate later does NOT retroactively affect already-printed invoices.
2. **Live rate from currency table** (`invoice.currency_code.currencies[0].exchange_rate`) — used as fallback for old invoices that don't have a stored rate.

```javascript
const storedInvRate = parseFloat(inv.exchange_rate || 0);
const exchangeRate = (storedInvRate > 1) ? storedInvRate : (inv.currency_code?.currencies?.[0]?.exchange_rate || 1);
```

**When is the exchange rate saved on the invoice?**
When an invoice status is set to "Printed" (via document printing in `document.controller.js`), the current exchange rate from the currency table is saved to the invoice's `exchange_rate` column. This "freezes" the rate at print time.

Exchange rates are retrieved from:
- `invoice.exchange_rate` (preferred — frozen at print time) with fallback to `invoice.currency_code.currencies[0].exchange_rate` (for invoices)
- `refund.currency_code_refund.currencies[0].exchange_rate` (for refunds, using alias)
- `deposit.currency_code.currencies[0].exchange_rate` (for deposits)
- `receipt_settlement_payment.exchange_rate` (for payment methods)

### 4.5 Duplicate Prevention

The code uses `processedInvoiceIds` Set to prevent counting the same invoice multiple times:
```javascript
if (!inv.id || !processedInvoiceIds.has(inv.id)) {
    totalInvoices += invoiceAmount;
    if (inv.id) {
        processedInvoiceIds.add(inv.id);
    }
}
```

---

## 5. Data Points Displayed in Report

### Customer Information Section
- Customer Name
- Customer Address
- Sales Tax Number
- Customer Number (in header)

### Report Header Information
- Report ID (format: "CUSTSTMT" + timestamp)
- Report Title: "Customer Account Statement"
- Company Name
- Company Address
- Printed By (username)
- Print Date/Time (formatted date and time)
- Document Date Range (startDate to endDate)
- Page Number

### Transaction Sections

#### 1. **Ticket Bookings** (Flight/Air Services)
Columns:
- Date (invoice_date formatted as DD-MM-YYYY)
- Invoice (invoice_number)
- Status (invoice.status — Raised / Printed / Settled / Partially Settled)
- Ticket No. (service_passenger.ticket_number)
- Passenger (service_passenger.passenger_name)
- Sector (flight routes from city codes, e.g., "ISB/DXB/JFK")
- Class (airline_class_code.class_code)
- Fare (invoice.price)
- Taxes (invoice_tax.tax_amount)
- Discount (calculated as basePrice * invoice.discount%)
- Rebate (calculated as basePrice * invoice.rebate%)
- T.Fee (transaction_fee)
- SST (calculated as transactionFee * invoice.sst%)
- Supp Fee (invoice.customer_supplementary_fee, split per passenger)
- Net (total calculated amount, including the supplementary fee)

Each passenger creates a row. The supplementary fee, transaction fee, and SST are per-invoice
totals, so each is divided by the number of passengers for the per-passenger row values.

> **Hide Column (2026-06-09)**: The **Discount, Rebate, T.Fee, and SST** columns can be hidden
> via the **Hide Column** filter (`hideColumns`). Hiding is display-only — the values remain in
> the **Net** and in all totals. Applies to both this table and the Refund Ticket Booking table,
> in PDF and Excel.

#### 2. **Hotel Bookings**
Columns:
- Date (invoice_date formatted as DD-MM-YYYY)
- Invoice (invoice_number)
- Hotel (service_hotel.hotel_name)
- Passenger (service_passengers list)
- Chk-in (service_hotel.check_in formatted as DD-MM-YYYY)
- Chk-out (service_hotel.check_out formatted as DD-MM-YYYY)
- Room Type (room_type.room_type)
- Rooms (service_hotel.no_of_rooms)
- Nights (service_hotel.no_of_nights)
- Net (total calculated amount including room multiplier)

#### 3. **General/Other Bookings** (Non-flight, non-hotel services)
Columns:
- Date
- Invoice
- Pax (passenger names)
- Ref No (reference number - empty)
- Remarks (the word **Umrah**/**Hajj** when the service type is Umrah/Hajj, otherwise empty — see [Hajj/Umrah Word in Remarks](#1007-hajjumrah-word-in-remarks-2026-06-17))
- Vendor (vendor name - empty)
- Transaction Date (invoice_date)
- Net (total amount)

#### 4. **Refund Ticket Bookings**
Currently not used - only credit notes are shown in refund sections

#### 5. **Refund General Bookings**
Same structure as General Bookings, showing ONLY credit notes (non-void)

#### 6. **Receipts/Vouchers** (G/L Account Payments Only)
**IMPORTANT**: Only receipts with G/L account payments are shown. Deposits and credit notes used in settlements are excluded.

Main Columns:
- Date (created_at formatted as DD-MM-YYYY)
- Receipt No. (receipt_number)
- Reference (receipt_reference)
- Description (remarks)
- Payment Method (payment_method)
- Amount (ONLY G/L account payment portions)

**Settlement Details** (Sub-sections per receipt):
- **Invoices Settled**: List of invoice numbers with amounts applied
  - Format: "INVOICE_NUMBER (Amount)"
- **Payment Methods**: Breakdown by GL account
  - Format: "GL_ACCOUNT: Amount CURRENCY (Rate: X, Base: Y PKR) [Deposit: Z] - Remarks"
- **Credit Notes Applied**: Credit notes used in settlement
  - Format: "TTCN00000XXX (Amount CURRENCY)"
- **Deposits Used**: Deposits applied to receipt
  - Format: "DEPOSIT_NO (Amount)"

#### 7. **Deposits** (Customer Prepayments)
Columns:
- Date (created_at)
- Deposit No. (receipt_number)
- Reference (reference || description)
- Description (description)
- Cheque No (composed from: pay_type_form.label - chart_of_account.key_account - check_number)
- Amount (FULL deposit value converted to PKR, not remaining balance)

**Note**: All deposits are shown with their full values converted to PKR using the deposit's currency exchange rate

#### 8. **Payments** (Payment Settlements TO Customer — NEW)
Payments made TO the customer via credit notes (refund payments).
Columns:
- Date (created_at formatted as DD-MM-YYYY)
- Payment No. (payment_number)
- Credit Note No. (credit_note.doc_no)
- Reference (reference)
- Amount (payment amount)

#### 9. **Journal Vouchers (JV)** (Manual JE Entries — NEW)
Manual journal entry adjustments for the customer account.
Columns:
- Date (transaction_date formatted as DD-MM-YYYY)
- Voucher No. (journal_batch.batch_no)
- Reference (description)
- Description (narration)
- Payment Method (chart_of_account description)
- Debit (debit amount)
- Credit (credit amount)

**Filtering**: Only `batch_type = 'Manual JE'`, status `Open` or `Posted`, matched by `gl_entity_id` = customer_id. Rows whose `description` starts with `VOID REVERSAL -` are excluded so a voided Manual JE nets back to zero in the report (the original entries are already excluded via the parent batch's Void status filter).

### Summary Section
```
Opening Balance B/F:      XXX,XXX.XX
Add Sale Invoices:        XXX,XXX.XX
Less Refund Invoices:     XXX,XXX.XX
Less Receipts:            XXX,XXX.XX  (GL account payments + deposits combined)
Add Payments:             XXX,XXX.XX  (payments TO customer via credit notes)
Add JV Debit:             XXX,XXX.XX  (manual JE debit entries)
Less JV Credit:           XXX,XXX.XX  (manual JE credit entries)
Net Balance:              XXX,XXX.XX
```

---

## 6. Report Output Formats

### 6.1 PDF Format
**Technology**: EJS template + wkhtmltopdf
**Template Location**: `/mnt/c/Codes/Powersuite/psback/views/pages/reports/customer-account-statement.ejs`
**Features**:
- Page breaks between customers
- Formatted tables with borders
- Proper spacing and alignment
- Number formatting with thousands separator
- Professional styling with gray section headers
- Summary box with bold totals

**PDF Generation Process**:
1. Render EJS template with customer data
2. Convert HTML to PDF using `createPdf()` function
3. Upload to S3/MinIO
4. Return download link

### 6.2 Excel Format
**Technology**: ExcelJS library
**Features**:
- Frozen header rows (first 6 rows)
- Professional styling:
  - Bold headers with gray background
  - Borders on all cells
  - Right-aligned numeric columns
  - Proper font sizing (11px, 10px, 9px depending on content)
  - Indented settlement details (size 9px, italic, gray)
- Dynamic column width adjustment
- Separate sections with section headers (size 13px, bold, gray background)
- Summary table with bold values
- Auto-width columns (min 10, max 50)

**Excel Structure per Customer**:
1. Company Name (size 28, bold, centered)
2. Company Address (size 12, centered)
3. Report ID, Printed By, Print Date (size 11)
4. Customer Info (Name, Address, Sales Tax Number)
5. Transaction Sections
6. Summary Section
7. Page break before next customer

---

## 7. Backend Processing Flow

### Step 1: Request Validation
- Authenticate user (JWT token)
- Check permission ("customer-account-statement")
- Validate request parameters

### Step 2: Data Retrieval (Optimized for Performance)
```
Start Database Transaction
    ├─ Fetch customers matching filters + company code check
    ├─ Fetch orders with services + invoices for those customers
    ├─ Fetch customer deposits for those customers
    ├─ Fetch deposit usage (for remaining balance calculation)
    ├─ Fetch receipt settlements with all settlement details
    ├─ Fetch credit notes
    └─ If date filter applied:
        └─ Fetch historical data (before startDate) for opening balance
```

### Step 3: Data Transformation
- Calculate invoice amounts with currency conversion
- Calculate refund amounts
- Categorize by service type (Flight, Hotel, General)
- Extract passenger information
- Build settlement details from related records
- Calculate opening and period balances

### Step 4: Report Generation
- Create report record in database
- If Excel: Build workbook with formatting
- If PDF: Render EJS template + convert to PDF
- Upload file to S3/MinIO

### Step 5: Response
Return to frontend:
```javascript
{
  status: 200,
  message: "success",
  link: "S3/MinIO URL",
  downloadLink: "signed URL for downloads",
  report: {
    id, user_id, file_type, report_number, 
    report_type, created_at, updated_at
  }
}
```

---

## 8. Related Service Files

### customer_balance_calculator.js
**Location**: `/mnt/c/Codes/Powersuite/psback/services/customer_balance_calculator.js`

Provides shared calculation functions:
- `calculateInvoiceAmount()`: Invoice total with all components
- `calculateRefundAmount()`: Refund with currency conversion
- `calculateDepositBalance()`: Remaining deposit balance
- `calculateReceiptAmount()`: Receipt amount
- `calculateCreditNoteAmount()`: Credit note amount
- `calculateCustomerBalance()`: Main calculation function with opening/period/net balances

**Purpose**: Ensures consistent calculations across Customer Account Statement and Customer Position Report

---

## 9. Frontend Component Integration

### Component: CustomerAccountStatement.jsx
**Location**: `/mnt/c/Codes/Powersuite/psfront/src/pages/Report/CustomerAccountStatement.jsx`

**Key Features**:
- Filter state management (customer, date range, filter operators)
- Dynamic filter input fields based on selected filters
- Dual output format support (PDF/Excel)
- Report history table with pagination
- Download functionality for Excel
- Report viewing for PDF

**API Call**:
```javascript
POST /report/getCustomerAccountStatementReport
Timeout: 300000ms (5 minutes)
```

**Report Display**:
- PDF: Navigates to `/reports/{report_number}?type=report`
- Excel: Downloads file with format `Customer_Account_Statement_{report_number}.xlsx`

---

## 10. New Features (March 2026 Updates)

### 10.0.1 Adjustment Date Mode (Posted to Ledger)
When `adjustmentDateMode = true`:
- If `adjustment_date` is NULL (unposted): include only if `created_at` is in date range
- If `adjustment_date` exists (posted): include only if `adjustment_date` is in date range

When `adjustmentDateMode = false`:
- Filter by `created_at` only

Applies to: Deposits, Receipts, Credit Notes, Payment Settlements, and all historical data queries.

### 10.0.2 Ticket Issue Date Logic
Controlled by `invoice_settings.ticket_issue_date_in_invoice` per company:
- If enabled AND service is "Air" AND `ticket_issue_date` exists: use `ticket_issue_date` for date filtering
- Otherwise: use `invoice_date`
- Applied at JavaScript level (not SQL) when enabled
- Affects both period invoices and historical invoices (opening balance calculation)

### 10.0.3 Payment Settlements Section (NEW)
Shows refund payments made TO customers via credit notes:
- Fetched from `payment_settlement` table where `credit_note_id` is not null
- Status filter: excludes 'Void'
- Date filtering respects adjustmentDateMode
- Columns: Date, Payment No, Credit Note No, Reference, Amount
- Added to both opening balance (historical) and period calculations

### 10.0.4 Journal Voucher (JV) Section (NEW)
Shows manual journal entry adjustments for customer accounts:
- Fetched from `journal_entry` + `journal_batch` tables
- Only `batch_type = 'Manual JE'`, status `Open` or `Posted`
- Matched by `gl_entity_id` = customer_id (string comparison)
- Date filter uses `transaction_date`
- Columns: Date, Voucher No, Reference, Description, Payment Method, Debit, Credit
- Added to both opening balance (historical) and period calculations
- Debit increases balance, Credit decreases balance
- **Void reversal rows excluded**: rows whose `description` starts with `VOID REVERSAL -` are filtered out. Combined with the parent batch's `status NOT IN ('Void')` filter, this means a voided Manual JE contributes zero to the report (original entries excluded via voided batch status; reversal entries excluded via description). Applies to both PDF and Excel output since both render from the same `journalEntries` data.

### 10.0.5 Credit Note Currency Handling (UPDATED)
Foreign currency detection uses the invoice's exchange rate via the refund→invoice chain as the source of truth:
- Credit note query includes `refund → Invoice → currency_code` to get the actual invoice currency
- If invoice currency is not PKR and exchange rate > 1: convert `cn.amount * invoiceExRate` to get PKR
- If `amount_base` is available and differs from `amount`: use `amount_base` directly (already PKR)
- If PKR invoice: use original `amount` or `billing_amount`
- Credit note creation now saves correct `currency_id` and `exchange_rate` from the invoice

### 10.0.6 Receipt Display Gate (UPDATED)
Only receipts with GL account payments totaling > 0 are displayed:
- If a receipt uses ONLY deposits or credit notes → excluded entirely
- Mixed payment methods → only GL account portion shown in amount

---

## 10.0.7 Hajj/Umrah Word + Miscellaneous Description in Remarks (2026-06-17)

For **General/Other Bookings**, the **Remarks** column shows:
- the word **`Umrah`** or **`Hajj`** when the service type is Umrah/Hajj (read from `service.service_type.type`); and
- the **service description** (`service_miscellaneous.description`, e.g. "extra luggage") when the service type is **Miscellaneous** (2026-06-19);
- for **company 1007 only**, the **service name** (`service.service_type.type`, e.g. Tour / Visa / Cruise / Train / Insurance) for all other service types (2026-06-30); and
- for every **other company**, all remaining service types keep a blank Remarks.

Details:
- The General booking row sets `remarks = (type === 'umrah' || type === 'hajj') ? service.service_type.type : (type === 'miscellaneous' && service.description ? \`Misc/${service.description}\` : (isCompany1007 ? service.service_type.type : ""))`, where `isCompany1007 = String(req.user.company_code) === '1007'`.
- **Company 1007 (2026-06-30)**: the final "else" shows the service name instead of blank, so Tour/Visa/Cruise/Train/Insurance rows display their type in Remarks. All other companies are unchanged.
- The customer orders query now includes `service_miscellaneous` (attribute `description` only) so the value is available.
- Display-only — no effect on amounts, Total Sales, or Net Balance. PDF and Excel both already render the Remarks column.
- (Umrah/Hajj briefly used `package_name`, simplified to the type word — Hajj records often have a blank package name.)

**Files Modified**: `psback/controllers/report.controller.js` (`getCustomerAccountStatementReport`) — `service_miscellaneous` include + `remarks` value on General bookings.

## 10.0.8 Per-Passenger Serial Number (S.NO) on Ticket Bookings (2026-06-17)

The **Ticket Bookings** section now has an **S.NO** column as the **first column**, numbering passenger rows **1..N per invoice** (restarts each invoice). Mirrors the Supplier Account Statement.
- Controller: the per-passenger `forEach` index is used — `sno: String(idx + 1)` on each ticket row.
- PDF (`customer-account-statement.ejs`): S.NO `th`/`td` added as the first column, narrowed with `width:1%; white-space:nowrap;` (the customer table has no clip rule, so the header shows in full). Total-row colspan bumped `15 → 16` for the extra column.
- Excel: `'sno'` added first to the Ticket `addSection` columns; header label special-cased to `S.NO`.
- Display-only; totals and balances unchanged. Scope: Ticket (Air) section only.

## 10.0.9 General/Other Bookings — One Row Per Passenger (2026-06-17)

The **General/Other Bookings** section now renders **one row per passenger** (it previously showed a single row per invoice with passenger names comma-joined).
- For each invoice, the build loops the service passengers and pushes one row each; the invoice total is **split across passengers** with a cent-accurate split, so the section total still equals the invoice total.
- `remarks` (the word Umrah/Hajj, see 10.0.7) and other fields repeat on each passenger row.
- Services with no passengers still produce a single row carrying the full amount.
- "Add Sale Invoices" / Net Balance unaffected (summary adds the invoice total once per invoice, independent of the row split).
- **S.NO column (2026-06-18)**: an **S.NO** column was added as the **first column**, numbering passenger rows **1..N per invoice** (restarts each invoice) — `sno: String(gi + 1)` on each row; added to the PDF table (first `th`/`td`, narrowed via `width:1%; nowrap`, Total-row colspan bumped 8→9) and to the Excel `addSection` columns (header `S.NO`).

## 11. Special Handling & Edge Cases

### 11.1 Hotel Room Multiplier
Hotel invoices multiply the calculated subtotal by number of rooms:
```javascript
if (serviceType === "hotel") {
    numberOfRooms = service_hotel.no_of_rooms || 1
    totalAmount = (subtotal * numberOfRooms) + transactionFee + sstAmount
}
```

### 11.2 Multiple Services per Order
Orders can have multiple services. Each service generates its own invoice entry.

### 11.3 Void Records Excluded
- Refunds with `status = "Void"` are excluded
- Deposits with `status = "Void"` are excluded
- Credit notes with `doc_status = "Void"` are excluded
- Receipts with `status = "Void"` are excluded

### 11.4 Invoice Status Filter
By default only invoices with status in ["Printed", "Settled", "Partially Settled"] are included.
When the **Include Raised Invoices** option (`includeRaised`) is ON, "Raised" is added to that list
for both the period sections and the Opening Balance B/F (the shared helper receives `includeRaised`).
The same status list is applied in the two invoice queries in the controller and in
`customerOpeningBalance.service.js` (where `includeRaised` defaults to false, so the Customer
Position Report, which does not pass the flag, is unaffected).

**Un-numbered Raised drafts excluded (2026-06-16)**: when `includeRaised` is ON, the invoice
query additionally excludes Raised invoices whose `invoice_number` is blank (NULL or empty) via
`NOT (status = 'Raised' AND invoice_number IS NULL/'')`. These are unfinished drafts (an invoice
number is assigned later), so they would otherwise show as a row with no Invoice number and 0.00
amounts. The same exclusion is mirrored in `customerOpeningBalance.service.js` so a draft cannot
affect the Opening Balance B/F.

### 11.5 Deposit Calculation
All deposits are shown with their FULL values converted to PKR
- Full deposit amount is converted using the deposit's currency exchange rate for both display and calculations
- No longer tracking remaining balance or usage

### 11.6 Receipt/Voucher Filtering
Only receipts with G/L account payments are included:
- If a receipt uses only deposits or credit notes, it's excluded entirely
- If a receipt uses mixed payment methods, only the G/L account portion is counted
- The amount shown is the sum of G/L account payments only

### 11.7 Single-Day Date Range Handling
When startDate equals endDate (single day):
- Uses `>=` operator instead of `BETWEEN` to include full day

### 11.8 Currency Conversion
All amounts automatically converted to PKR. For invoices, the exchange rate frozen at print time is used (stored in `invoice.exchange_rate`). If no stored rate exists (legacy invoices), falls back to the live rate from the currency table. If no exchange rate found at all, defaults to 1.0

### 11.9 Multiple Passengers
Flight refunds and bookings create one row per passenger per invoice.

### 11.9.1 Supplementary Fee on Ticket Bookings
The `invoice.customer_supplementary_fee` (set when the supplementary checkbox is ticked on the
invoice) is included in the Customer Account Statement for ticket/Air bookings:
- Shown in its own **Supp Fee** column in the Ticket Booking table (PDF and Excel).
- Added into the per-passenger **Net** (split by passenger count, like T.Fee and SST).
- Counted in **Total Sales** and in the **Opening Balance B/F** (historical Air calc), so the
  report value matches the invoice's stored `total_price`.
- Hotel and General bookings already include it implicitly because they value invoices from
  `total_price`, which is saved with the supplementary fee at invoice create/update time.

### 11.10 Unified Date Boundary (Opening Balance vs Period) — IMPORTANT
The report values invoices in two places: the **period sections** ("Total Sales") and the
**Opening Balance B/F** (historical calc). Both MUST use the **same** start-of-period date
boundary, otherwise an invoice dated near a month boundary is classified into the period by
one calc and into the opening balance by the other — so one report's closing balance no
longer equals the next report's opening balance (a reconciliation gap).

**Rule:** the historical/opening-balance invoice cutoff uses a single shared constant
`historicalCutoff = moment(startDate).startOf('day').toDate()`, which is the **exact same**
boundary the period sections use via `dateFilterInfo.start` / `dateFilterInfo.date`.

- ❌ Do NOT use a raw `new Date(startDate)` for the historical invoice cutoff — that is UTC
  midnight, while the period calc uses moment local-midnight; on a non-UTC server they differ
  by the timezone offset and month-boundary invoices get mis-classified.
- ✅ Always compare the invoice date against `historicalCutoff` so an invoice is counted in
  the opening balance **if and only if** it is before the period — never in both, never in
  neither.

This guarantees: previous period's Net Balance == next period's Opening Balance B/F.

---

## 12. Performance Considerations

### Optimization Strategies
1. **Separate Queries**: Uses separate queries instead of deep nesting to avoid timeout
2. **Set-based Deduplication**: Uses Set to track processed invoice IDs
3. **Index Requirements**: 
   - customer_id on orders, invoice, receipt_settlement, customer_deposit, credit_note
   - invoice_date on invoice
   - created_at on deposit, receipt_settlement, refund, credit_note
   - status on invoice, refund, receipt_settlement, deposit, credit_note

### Query Complexity
- Main query: Single customer fetch with user validation
- Supporting queries: 7+ separate queries for orders, deposits, receipts, credit notes, and historical data
- Total execution time tracked in logs: Typically 100-500ms for medium-sized datasets

---

## 13. Database Query Overview

### Main Queries Executed

1. **Customer Query**
```sql
SELECT * FROM customer 
JOIN user ON customer.user_id = user.id
WHERE {customerFilter conditions}
AND user.company_code = '{current_company_code}'
```

2. **Orders with Services & Invoices**
```sql
SELECT o.*, s.*, i.*, it.*, sf.*, sh.*, st.*
FROM orders o
JOIN services s ON o.id = s.order_id
LEFT JOIN invoices i ON s.id = i.service_id
LEFT JOIN invoice_taxes it ON i.id = it.invoice_id
LEFT JOIN service_flights sf ON s.id = sf.service_id
LEFT JOIN service_hotels sh ON s.id = sh.service_id
LEFT JOIN service_types st ON s.service_type_id = st.id
WHERE o.customer_id IN ({customerIds})
AND i.status IN ['Printed', 'Settled', 'Partially Settled']
AND {date filters on invoice_date}
```

3. **Customer Deposits**
```sql
SELECT cd.*, ptf.*, coa.*, cc.*, c.*
FROM customer_deposits cd
LEFT JOIN pay_type_forms ptf ON cd.pay_type_form_id = ptf.id
LEFT JOIN chart_of_accounts coa ON cd.chart_of_account_id = coa.id
LEFT JOIN currency_codes cc ON cd.currency_code_id = cc.id
LEFT JOIN currencies c ON cc.id = c.currency_code_id
WHERE cd.customer_id IN ({customerIds})
AND cd.status != 'Void'
AND {date filters}
```

4. **Receipt Settlements with Details**
```sql
SELECT rs.*, rsi.*, rsp.*, rscn.*, rsd.*
FROM receipt_settlements rs
LEFT JOIN receipt_settlement_invoices rsi ON rs.id = rsi.receipt_settlement_id
LEFT JOIN receipt_settlement_payments rsp ON rs.id = rsp.receipt_settlement_id
LEFT JOIN receipt_settlement_credit_notes rscn ON rs.id = rscn.receipt_settlement_id
LEFT JOIN receipt_settlement_deposits rsd ON rs.id = rsd.receipt_settlement_id
WHERE rs.customer_id IN ({customerIds})
AND rs.status != 'Void'
```

5. **Credit Notes**
```sql
SELECT * FROM credit_notes
WHERE customer_id IN ({customerIds})
AND doc_status != 'Void'
AND {date filters on doc_date}
```

6. **Deposit Usage**
```sql
SELECT * FROM receipt_settlement_deposits
WHERE customer_deposit_id IN ({depositIds})
```

---

## 14. API Endpoint

### POST /report/getCustomerAccountStatementReport

**Authentication**: Required (JWT token via cookie)
**Authorization**: Permission check for "customer-account-statement"
**Timeout**: 5 minutes (300000ms)

**Request Body**:
```javascript
{
  customerFilter: string,     // Filter operator
  customer_id: number,        // Optional: for single customer
  customer_idStart: number,   // Optional: for range
  customer_idEnd: number,     // Optional: for range
  dateFilter: string,         // Filter operator or "blank"
  startDate: string,          // ISO date string
  endDate: string,            // ISO date string (for range filter)
  adjustmentDateMode: boolean, // When true, use adjustment_date for posted docs
  type: string                // "pdf" or "excel"
}
```

**Response**:
```javascript
{
  status: 200,
  message: "success",
  link: string,               // Direct S3/MinIO link
  downloadLink: string,       // Signed URL for downloads
  report: {
    id: number,
    user_id: number,
    file_type: string,        // "pdf" or "xlsx"
    report_number: string,    // "CUSTSTMT" + timestamp
    report_type: string,      // "customer-account-statement"
    created_at: timestamp,
    updated_at: timestamp
  }
}
```

---

## 15. Summary Table: All Database Models Used

| Model | Tables | Purpose | Key Relationship |
|-------|--------|---------|------------------|
| customer | customers | Account owner | Parent for orders, deposits, receipts |
| user | users | Multi-tenancy | Parent for customers |
| order | orders | Service grouping | Parent for services |
| service | services | Transaction unit | Parent for invoices, refunds |
| service_flight | service_flights | Flight details | Child of service |
| service_hotel | service_hotels | Hotel details | Child of service |
| service_type | service_types | Category | Referenced by service |
| service_passenger | service_passengers | Passenger info | Child of service |
| passenger | passengers | Passenger details | Referenced by service_passenger |
| invoice | invoices | Billing document | Child of service |
| invoice_tax | invoice_taxes | Tax breakdown | Child of invoice |
| ~~refund~~ | ~~refunds~~ | ~~Refund record~~ | ~~Not used - excluded from report~~ |
| receipt_settlement | receipt_settlements | Payment receipt | Child of customer |
| receipt_settlement_invoice | receipt_settlement_invoices | Invoice paid | Junction |
| receipt_settlement_payment | receipt_settlement_payments | Payment method | Child of receipt |
| receipt_settlement_credit_note | receipt_settlement_credit_notes | CN applied | Junction |
| receipt_settlement_deposit | receipt_settlement_deposits | Deposit used | Junction |
| customer_deposit | customer_deposits | Prepayment | Child of customer |
| pay_type_form | pay_type_forms | Payment type | Referenced by deposit |
| chart_of_account | chart_of_accounts | GL account | Referenced by deposit, payment |
| currency_code | currency_codes | Currency | Referenced by invoice, deposit, payment |
| currency | currencies | Exchange rates | Child of currency_code |
| credit_note | credit_notes | Credit memo | Child of customer |
| payment_settlement | payment_settlements | Payments TO customer | Via credit notes (NEW) |
| journal_entry | journal_entries | Manual JE entries | Via gl_entity_id = customer_id (NEW) |
| journal_batch | journal_batches | JE batch info | Parent of journal_entry (NEW) |
| invoice_setting | invoice_settings | Company settings | ticket_issue_date flag (NEW) |
| report | reports | Report record | Created by function |

---

## 16. Code Files Reference

| File | Type | Purpose |
|------|------|---------|
| `/psback/controllers/report.controller.js` | Controller | Main function `getCustomerAccountStatementReport` (line 11589) |
| `/psback/services/customer_balance_calculator.js` | Service | Shared calculation functions |
| `/psback/routes/report.route.js` | Routes | API endpoint definition |
| `/psfront/src/pages/Report/CustomerAccountStatement.jsx` | React | Frontend UI and filter management |
| `/psfront/src/api/report.js` | API | Frontend API call wrapper |
| `/psback/views/pages/reports/customer-account-statement.ejs` | Template | EJS HTML template for PDF |

---

## 17. Key Insights & Design Patterns

1. **Separation of Concerns**: Calculation logic in service file, controller orchestrates data flow
2. **Reusability**: Balance calculator can be used by multiple reports
3. **Multi-format Support**: Same data used for both PDF and Excel
4. **Currency Handling**: Automatic conversion at point of display
5. **Performance**: Multiple small queries instead of complex JOINs
6. **Audit Trail**: Report records stored for tracking
7. **Multi-tenancy**: Company code ensures data isolation
8. **Flexible Filtering**: Customer and date filters support multiple operators
9. **Refund Handling**: Service refunds are completely excluded; only credit notes are used for refund tracking and calculations
10. **Deposit Handling**: Full deposit values are shown and calculated (not remaining balances)
11. **Receipt/Voucher Handling**: Only G/L account payment portions are shown and counted; deposits and credit notes used in settlements are excluded from receipt amounts
12. **Adjustment Date Mode**: Supports posted-to-ledger filtering using adjustment_date for posted documents
13. **Ticket Issue Date**: For Air services, can use ticket_issue_date instead of invoice_date (per company setting)
14. **Payment Settlements**: Tracks refund payments made TO customers via credit notes
15. **Journal Vouchers**: Manual JE entries affecting customer accounts (debit increases, credit decreases balance)
16. **Frozen Exchange Rate**: Invoice exchange rates are saved at print time and used in reports. Changing the exchange rate later does NOT retroactively affect already-printed invoices. Legacy invoices without a stored rate fall back to the live currency table rate.

---

**Last Updated**: June 2026 — Include Raised Invoices now skips un-numbered Raised drafts (blank invoice_number); earlier: added "Include Raised Invoices" option (optional toggle, default OFF) with a Status column on Ticket/Hotel/General bookings (PDF + Excel); Adjustment Date Mode, Ticket Issue Date, Payments section, JV section, updated formulas, frozen exchange rate on invoice print

