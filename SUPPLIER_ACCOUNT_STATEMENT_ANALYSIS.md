# Supplier Account Statement Report - Complete Implementation Analysis

## Overview
The Supplier Account Statement report is a comprehensive system that tracks supplier transactions, including bookings (flights, hotels, general services), refunds, and payments. The implementation spans both frontend and backend with data fetching, calculation, and multi-format reporting capabilities.

---

## 1. File Structure and Locations

### Frontend Files
- **Main Component**: `/mnt/c/Codes/Powersuite/psfront/src/pages/Report/SupplierAccountStatement.jsx`
- **API Integration**: `/mnt/c/Codes/Powersuite/psfront/src/api/report.js`
  - Function: `getSupplierAccountStatementReport(filterData)`

### Backend Files
- **Route Handler**: `/mnt/c/Codes/Powersuite/psback/routes/report.route.js`
  - Endpoint: `POST /report/getSupplierAccountStatementReport`
  - Middleware: `authenticate`, permission check (`supplier-account-statement`)
  
- **Controller Logic**: `/mnt/c/Codes/Powersuite/psback/controllers/report.controller.js`
  - Function: `exports.getSupplierAccountStatementReport()` (line 9940)

- **EJS Template**: `/mnt/c/Codes/Powersuite/psback/views/pages/reports/supplier-account-statement.ejs`
  - PDF rendering template for supplier statements

- **Database Models**:
  - `/mnt/c/Codes/Powersuite/psback/models/cost.js` - Cost information
  - `/mnt/c/Codes/Powersuite/psback/models/cost_tax.js` - Tax details per cost

---

## 2. Data Flow Architecture

### 2.1 Frontend Request Flow

```
SupplierAccountStatement.jsx (Component)
    ↓
Form Submission (handleSubmit)
    ↓
getSupplierAccountStatementReport(filterData) [API Call]
    ↓
POST /report/getSupplierAccountStatementReport
    ↓
Backend Processing
    ↓
Response with:
    - report object (report_number, file_type, created_at)
    - downloadLink (for Excel files)
    - PDF rendering (for PDF exports)
```

### 2.2 Backend Processing Flow

```
Request Received (filters)
    ↓
Build Supplier Where Clause (supp_no filters)
    ↓
Build Date Where Clauses (invoice_date, created_at)
    ↓
Query Database for Suppliers with Relations
    ↓
For Each Supplier:
    - Calculate Opening Balance (historical transactions before start date)
    - Process Services:
        - Get XO Document Number
        - Calculate Costs (published_rate - commission + taxes)
        - Categorize by Type (Flight, Hotel, General)
        - Generate Booking Rows
    - Process Refunds:
        - Subtract from addSaleInvoices
        - Create Refund Rows
    - Process Payments:
        - Aggregate from payment_settlement_payments
        - Create Voucher Rows
    - Calculate Net Balance
    ↓
Generate Excel or PDF
    ↓
Upload to Storage
    ↓
Return Response with Download Link
```

---

## 3. Filter and Query Parameters

### Filter Input Structure (Frontend)

```javascript
{
  supp_noFilter: "isNotBlank" | "isBlank" | "isEqual" | "between",
  supp_no: string | null,              // Single supplier number
  supp_noStart: string | null,         // Range start
  supp_noEnd: string | null,           // Range end
  dateFilter: "blank" | "=" | "<" | "<=" | ">" | ">=" | "<>" | "between",
  startDate: string | null,            // ISO date format
  endDate: string | null,              // ISO date format
  adjustmentDateMode: boolean,         // When true, use adjustment_date for posted docs
  type: "pdf" | "excel"                // Output format
}
```

### Supplier Query Conditions

| Filter Type | SQL WHERE Clause |
|---|---|
| `isNotBlank` | `supp_no IS NOT NULL AND supp_no != ''` |
| `isBlank` | `supp_no IS NULL OR supp_no = ''` |
| `isEqual` | `supp_no = [value]` |
| `between` | `supp_no BETWEEN [start] AND [end]` |

### Date Query Conditions

Date filters apply to:
- `invoice.invoice_date` (for invoiced services)
- `service.created_at` (fallback for services)
- `debit_note.doc_date` (for debit notes)

Operators:
- `=`: Exact date match
- `<`: Before date
- `<=`: On or before date
- `>`: After date
- `>=`: On or after date
- `<>`: Not equal to date
- `between`: Between two dates (inclusive)

---

## 4. XO (Costing Document) References

XO stands for **Exchange Order** or **Costing Document**. It's the core mechanism for cost tracking.

### XO Data Structure

```javascript
{
  xoNumber: service.Cost?.Document?.document_number || "",
  xoDate: service.Cost?.created_at || service.created_at,
  // XO Document must exist for service to appear in report
  // Referenced in all booking tables as 'xo' column
}
```

### Usage in Report

- **Ticket Bookings**: `date, xo, ticketNo, passenger, sector, class, fare, taxes, net`
- **Hotel Bookings**: `date, xo, hotel, passenger, checkin, checkout, roomType, rooms, nights, net`
- **General Bookings**: `date, xo, pax, refNo, remarks, vendor, transactionDate, net`
- **Refund Bookings**: Same structure with refund amounts

**Important**: Services without Cost data are skipped and logged with warning.

---

## 5. Amount Calculations and Formatting

### 5.1 Cost Calculation Formula

```javascript
// Base Components from Cost Model
published_rate = Cost.published_rate         // Original fare/rate (in cost currency)
commission = Cost.commission                 // Percentage (e.g., 5.5)
net_rate = Cost.net_rate                     // Calculated: published_rate - (published_rate * commission / 100)
currency = Cost.currency                     // Currency ID (e.g., 110 for PKR, other for foreign currencies)

// Tax Components
cost_taxes = Cost.cost_taxes[] (array)       // Individual tax records
tax_amount = SUM(cost_tax.tax_amount)        // Total taxes per cost

// Currency Conversion (NEW)
if (currency && currency !== 'PKR' && currency !== 110) {
    // Convert to PKR using convertToPKR service
    basePriceInPKR = await convertToPKR(published_rate, currency, currency_code).convertedAmount
    netRateInPKR = await convertToPKR(net_rate, currency, currency_code).convertedAmount
    taxesInPKR = await convertToPKR(taxes, currency, currency_code).convertedAmount
} else {
    // Already in PKR, no conversion needed
    basePriceInPKR = published_rate
    netRateInPKR = net_rate
    taxesInPKR = taxes
}

// Final Amount for Supplier Statement (in PKR)
fare = basePriceInPKR
taxes = taxesInPKR
net = (netRateInPKR + taxesInPKR) * quantity

// Cost Total (used in addSaleInvoices)
costTotal = netRateInPKR + taxesInPKR
totalAmount = costTotal * quantity
```

### 5.2 Amount Formatting

Numbers are formatted as currency (2 decimal places):
```javascript
Number(value).toLocaleString('en-US', {
  minimumFractionDigits: 2,
  maximumFractionDigits: 2
})
// Example: 1234.50 displays as "1,234.50"
```

### 5.3 Refund Amount Calculation

```javascript
// Refunds use base currency amounts (PKR) if available
refundAmount = parseFloat(refund.supplier_refund_amount_base || refund.supplier_refund_amount || 0)

// If supplier_refund_amount_base is not set, the system falls back to supplier_refund_amount
// which should be converted to PKR if in foreign currency
```

### 5.4 Summary Calculation

```javascript
netBalance = openingBalance
           + addSaleInvoices      // Add costs incurred
           - lessRefundInvoices   // Subtract refunds (debit notes)
           - lessPayments         // Subtract payments made
           - lessAdvancePayments  // Subtract advance payments (supplier deposits)
           + jvCreditTotal        // Add JV credits (for suppliers, increases payable)
           - jvDebitTotal         // Subtract JV debits (decreases payable)

Where:
  openingBalance = Historical costs - historical refunds - historical payments
                 + historical debit notes + (historical jvCredit - historical jvDebit)
                 - historical advance payments
  addSaleInvoices = Sum of costs (in PKR) for services in date range
  lessRefundInvoices = Sum of refund amounts (in PKR) in date range
  lessPayments = Sum of payment_settlement_payment.amount records (in PKR)
  lessAdvancePayments = Sum of supplier deposit amounts in date range
  jvCreditTotal = Sum of manual JE credit entries (gl_entity_id = supplier_id)
  jvDebitTotal = Sum of manual JE debit entries (gl_entity_id = supplier_id)

Note: For suppliers (liability accounts), JV Credits INCREASE balance, JV Debits DECREASE balance
      (opposite of customer accounts)
Note: All amounts are converted to PKR using the convertToPKR service from currencyConverter.js
```

### 5.5 Voucher (Payment) Columns

```javascript
{
  date: payment.created_at (formatted as DD-MM-YYYY),
  adjustmentDate: settlement.adjustment_date (DD-MM-YYYY), // Only if adjustmentDateMode
  voucherNo: settlement.payment_number || payment.voucher_number,
  reference: payment.voucher_number || settlement.payment_number,
  description: payment.remarks || "Payment to [SupplierName]",
  chequeNo: payment.check_number,
  debit: payment.amount,      // Payment to supplier
  credit: 0                   // Not used in supplier statements
}
```

**Adjustment Date Mode for Vouchers**:
- When `adjustmentDateMode = true`:
  - If `settlement.adjustment_date` is NULL (unposted): include if `created_at` is in range
  - If `settlement.adjustment_date` exists (posted): include if `adjustment_date` is in range
- When `false`: always filter by `created_at`

### 5.6 Advance Payment Columns (NEW)

```javascript
{
  date: deposit.created_at (DD-MM-YYYY),
  adjustmentDate: deposit.adjustment_date (DD-MM-YYYY), // Only if adjustmentDateMode
  paymentNo: deposit.payment_number,
  supplierNo: supplier.supp_no,
  supplierName: supplier.supp_name,
  originalAmount: deposit.amount,
  availableAmount: deposit.current_amount
}
```

Source: `supplier_deposits` table (status != 'Void')

### 5.7 Journal Voucher (JV) Columns (NEW)

```javascript
{
  date: je.transaction_date (DD-MM-YYYY),
  voucherNo: batch.batch_no,
  reference: je.analysis_code1,
  description: je.description,
  paymentMethod: `${account.key_account} - ${account.description}`,
  debit: parseFloat(je.debit),
  credit: parseFloat(je.credit)
}
```

Source: `journal_entries` + `journal_batches` where `batch_type = 'Manual JE'`, `status IN ['Open', 'Posted']`, `gl_entity_id = supplier_id`

---

## 6. Database Relations and Includes

### Main Query Structure

```javascript
db.supplier.findAll({
  where: supplierWhere,
  include: [
    {
      model: db.user,              // For company_code filter
      include: {
        model: db.company,         // Filter by user's company
        where: { code: req?.user?.company_code }
      }
    },
    {
      model: db.debit_note,        // Supplier debit notes
      required: false,
      where: {
        doc_date: invoiceDateWhere.invoice_date || {}
      }
    },
    {
      model: db.service,           // Main service details
      required: false,
      where: Object.keys(serviceDateWhere).length ? serviceDateWhere : undefined,
      include: [
        {
          model: db.service_type   // Flight, Hotel, General, etc.
        },
        {
          model: db.invoice,       // Invoicing info
          where: {
            status: ["Printed", "Settled", "Partially Settled"],
            ...invoiceDateWhere
          },
          required: false
        },
        {
          model: db.cost,          // Main cost/costing document
          where: {
            status: ["Printed", "Settled", "Partially Paid", "Paid"]
          },
          include: [
            {
              model: db.cost_tax   // Tax breakdown per cost
            },
            {
              model: db.document,  // XO document (costing document)
              where: {
                document_type: "costing"
              }
            },
            {
              model: db.currency_code,  // Currency information for conversion
              required: false,
              include: {
                model: db.currency,
                where: { to_currency: 'PKR' },
                required: false
              }
            },
            {
              model: db.payment_settlement_cost,
              include: {
                model: db.payment_settlement,
                where: { status: ["Printed"] },
                include: {
                  model: db.payment_settlement_payment
                }
              }
            }
          ]
        },
        {
          model: db.service_flight,    // Flight specifics
          include: [
            { model: db.city_code, as: "city_from_code" },
            { model: db.city_code, as: "city_to_code" },
            { model: db.airline_class_code, as: "flightClass" }
          ]
        },
        {
          model: db.service_hotel,     // Hotel specifics
          include: [
            { model: db.room_type, as: "room_type" }
          ]
        },
        {
          model: db.service_passenger, // Passenger info
          include: {
            model: db.passenger
          }
        },
        {
          model: db.refund,            // Refund records
          as: 'refunds',
          attributes: ['id', 'refund_no', 'supplier_refund_amount', 'supplier_refund_amount_base',
                      'passenger_name', 'created_at', 'debit_note_generated', 'debit_note_id',
                      'currency_code', 'exchange_rate'],
          include: {
            model: db.debit_note,      // Debit note associated with refund
            as: 'debit_note',
            attributes: ['id', 'doc_no', 'reference', 'doc_date', 'amount'],
            required: false
          }
        }
      ]
    }
  ]
})

// Additional queries (NEW):

// Supplier Deposits (Advance Payments)
db.supplier_deposit.findAll({
  where: { supplier_id, status: { [Op.ne]: 'Void' } }
  // Date filter respects adjustmentDateMode
})

// Journal Entries (Manual JE)
db.journal_entry.findAll({
  where: { gl_entity_id: supplierIds },
  include: [{
    model: db.journal_batch,
    where: { batch_type: 'Manual JE', status: ['Open', 'Posted'] }
  }, {
    model: db.chart_of_account
  }]
  // Date filter on transaction_date
})
```

---

## 7. Report Data Structure

### Output Data Array

```javascript
[
  {
    supplier: {
      name: "Supplier Name",
      address: "Full Address",
      taxNumber: "Tax ID"
    },
    summary: {
      openingBalance: 1000.00,
      addSaleInvoices: 5000.00,
      lessRefundInvoices: 500.00,
      lessPayments: 2000.00,
      lessAdvancePayments: 500.00,    // NEW
      jvDebitTotal: 100.00,           // NEW - LESS: JV Debits
      jvCreditTotal: 200.00,          // NEW - ADD: JV Credits
      netBalance: 3100.00
    },
    ticketBookings: [
      {
        date: "DD-MM-YYYY",
        xo: "XO-DOC-NUMBER",
        ticketNo: "TICKET123",
        passenger: "John Doe",
        sector: "JFK/LHR/CDG",  // Origin/Destination chain
        class: "Y",              // Economy, Business, etc.
        fare: 500.00,
        taxes: 50.00,
        net: 550.00
      }
    ],
    hotelBookings: [
      {
        date: "DD-MM-YYYY",
        xo: "XO-DOC-NUMBER",
        hotel: "Hotel Name",
        passenger: "John Doe",
        checkin: "DD-MM-YYYY",
        checkout: "DD-MM-YYYY",
        roomType: "Double",
        rooms: 1,
        nights: 3,
        net: 300.00
      }
    ],
    generalBookings: [
      {
        date: "DD-MM-YYYY",
        xo: "XO-DOC-NUMBER",
        pax: "Passenger Name",
        refNo: "Reference",
        remarks: "Remarks",
        vendor: "Vendor Name",
        transactionDate: "DD-MM-YYYY",
        net: 100.00
      }
    ],
    refundTicketBookings: [
      {
        date: "DD-MM-YYYY",
        xo: "XO-DOC-NUMBER",
        ticketNo: "TICKET123",
        passenger: "John Doe",
        sector: "JFK/LHR/CDG",
        class: "Y",
        debitNote: "BSDN00000001",    // Debit note document reference
        fare: 500.00,                  // Supplier override gross amount
        taxes: 50.00,                  // Supplier override tax amount
        commission: 25.00,             // Calculated from cost
        wht: 1.25,                     // (commission × sst%)
        airlineCharges: 10.00,         // supplier_airline_charges
        refundCharges: 5.00,           // supplier_refund_charges
        extraCharges: 0,               // free_of_cost from cost
        net: 500.00                    // supplier_refund_amount_base (PKR)
      }
    ],
    refundGeneralBookings: [
      {
        date: "DD-MM-YYYY",
        xo: "XO-DOC-NUMBER",
        pax: "Passenger Name",
        refNo: "REF-123",
        remarks: "Debit Note",
        vendor: "Vendor Name",
        transactionDate: "DD-MM-YYYY",
        debitNote: "BSDN00000001",  // Debit note document reference (NEW)
        net: 100.00  // Refund/debit note amount in PKR
      }
    ],
    vouchers: [
      {
        date: "DD-MM-YYYY",
        adjustmentDate: "DD-MM-YYYY",  // Only if adjustmentDateMode (NEW)
        voucherNo: "VOUCHER-001",
        reference: "REF-123",
        description: "Payment to Supplier",
        chequeNo: "CHQ-456",
        debit: 2000.00,
        credit: 0
      }
    ],
    advancePayments: [               // NEW SECTION
      {
        date: "DD-MM-YYYY",
        adjustmentDate: "DD-MM-YYYY",  // Only if adjustmentDateMode
        paymentNo: "PAYMENT-001",
        supplierNo: "SUPP001",
        supplierName: "Supplier Name",
        originalAmount: 5000.00,
        availableAmount: 3000.00
      }
    ],
    journalVouchers: [               // NEW SECTION
      {
        date: "DD-MM-YYYY",
        voucherNo: "TTJV000001",
        reference: "REF-123",
        description: "Description",
        paymentMethod: "1234 - Cash Account",
        debit: 1000.00,
        credit: 0
      }
    ]
  }
]
```

---

## 8. Opening Balance Calculation

For date-filtered reports, opening balance includes all transactions BEFORE startDate:

```javascript
openingBalance = 0

// 1. Historical costs (services before startDate)
for each historicalService {
  if (service.Cost) {
    openingBalance += totalCost (netRate + taxes + wht + freeOfCost) × quantity × exchangeRate

    // Subtract historical payments
    for each payment_settlement_cost {
      for each payment_settlement_payment {
        // Respects adjustmentDateMode for date check
        openingBalance -= payment.amount
      }
    }
  }

  // Subtract historical refunds (only if debit_note_generated && not Void/Voided)
  for each refund {
    openingBalance -= refund.supplier_refund_amount_base || supplier_refund_amount
  }
}

// 2. Add historical debit notes (not linked to refunds, not Void)
for each historicalDebitNote {
  openingBalance += debitNote.amount
}

// 3. Historical journal entries (Manual JE only, before startDate)
// For suppliers (liability): credits INCREASE balance, debits DECREASE
openingBalance += (historicalJvCreditTotal - historicalJvDebitTotal)

// 4. Historical advance payments (supplier deposits, before startDate)
// Respects adjustmentDateMode for date check
openingBalance -= totalHistoricalAdvancePayments
```

---

## 9. Output Formats

### 9.1 Excel (XLSX) Output

Generated using **ExcelJS** library:

```javascript
workbook = new ExcelJS.Workbook()
worksheet = workbook.addWorksheet('Supplier Account Statement')

// Frozen rows: First 6 rows (header section)
worksheet.views = [
  { state: 'frozen', ySplit: 6, activeCell: 'A7' }
]

// Structure:
// Rows 1-6: Company header info
// Row 7: Blank
// For each supplier:
//   - Supplier name (merged A:J)
//   - Address
//   - Tax Number
//   - Blank row
//   - Section tables:
//     1. Ticket Bookings
//     2. Hotel Bookings
//     3. General Bookings
//     4. Refund Ticket Bookings (with debitNote column)
//     5. Refund General Bookings (with debitNote column)
//     6. Vouchers (with adjustmentDate column if adjustmentDateMode)
//     7. Advance Payments (with adjustmentDate column if adjustmentDateMode) — NEW
//     8. Journal Vouchers (debit/credit) — NEW
//   - Summary table (Opening Balance → Net Balance, includes Advance Payments & JVs)
//   - Blank row separator
```

Features:
- Column widths auto-adjusted (min 10, max 50)
- Amount columns right-aligned
- Text columns left-aligned
- Bold headers with gray background (#D9D9D9)
- Borders on all cells
- Number formatting: 2 decimal places with thousands separator

### 9.2 PDF Output

Generated via EJS template and **wkhtmltopdf**:

Template: `/mnt/c/Codes/Powersuite/psback/views/pages/reports/supplier-account-statement.ejs`

Features:
- Print-friendly landscape layout
- Dynamic table generation with renderTable() function
- Font: Times New Roman, 12px (body), 10px (table)
- Page break settings for multi-page documents
- Summary table centered on right side
- Company header with filters displayed

---

## 10. Error Handling

### Validation Errors

```javascript
// No suppliers found
Status: 400
{
  error: "No suppliers found matching the criteria. 
          Please check your filters and ensure suppliers 
          exist for your company."
}

// No transaction data after processing
Status: 400
{
  error: "No transaction data found for the selected suppliers. 
          Suppliers must have at least one invoice, refund, 
          credit note, or debit note to appear in the account statement."
}

// Server error
Status: 500
{
  error: "Internal Server Error"
}
```

### Data Validation

- Supplier filter applied (isBlank/isNotBlank required at minimum)
- Date filters must have valid ISO date format
- Range filters (between) automatically swap if start > end
- Services without Cost data are logged and skipped
- Only invoiced services with proper status are included

---

## 11. Performance Considerations

### Database Query Optimization

1. **Selective Includes**: 
   - Uses `required: false` for most relations to avoid INNER JOINs
   - Only services within date range are queried with WHERE clause
   - Invoices filtered by status

2. **Pagination**:
   - Report history uses pagination (page, limit)
   - Individual report generation processes all suppliers in one query

3. **Historical Calculations**:
   - When date filter exists, opening balance calculated separately
   - Uses `Op.lt` to get only pre-startDate transactions

### File Size Considerations

- Excel reports can be large with many suppliers/transactions
- PDF generation uses landscape layout to accommodate wide tables
- Buffer-based upload for Excel files

---

## 12. Integration Points

### Authentication & Authorization

```javascript
router.post("/getSupplierAccountStatementReport", 
  authenticate,                                      // JWT verification
  permission("supplier-account-statement"),         // Role-based permission
  getSupplierAccountStatementReport
)
```

### Company Isolation

```javascript
// All suppliers filtered by company_code
where: {
  include: {
    model: db.company,
    where: { code: req?.user?.company_code },
    required: true
  }
}
```

### Report Tracking

```javascript
db.report.create({
  user_id: req?.user?.id,
  file_type: type === "excel" ? "xlsx" : "pdf",
  report_number: documentNumber,              // TPSAS + timestamp
  report_type: "supplier-account-statement",
  created_at: new Date(),
  updated_at: new Date()
})
```

---

## 13. Key Business Logic Insights

### TTX (Transaction/Costing) References

- **XO** (eXchange Order/Costing Document) is the primary transaction reference
- Generated via `service.Cost?.Document?.document_number`
- Every booking row includes XO number for traceability
- Cost document type must be "costing" for inclusion

### Service Type Categorization

```javascript
type = service.service_type?.type?.toLowerCase()

if (type === "air" || type === "ticket" || type === "flight") {
  // Ticket Booking with: fare, taxes, net
  // Passenger details from service_passengers
  // Flight segments from service_flights
  // Class from airline_class_code
}
else if (type === "hotel") {
  // Hotel Booking with: checkin, checkout, rooms, nights, net
  // Hotel details from service_hotel
  // Room type from room_type model
}
else {
  // General Booking
  // Used for miscellaneous services and refunds
}
```

### Payment Tracking

- **Vouchers** created from payment_settlement_payment records
- Each payment creates one voucher row
- Payments must be from settlements with status "Printed"
- Cheque numbers (check_number) included if available

### Refund Processing

```javascript
refund.supplier_refund_amount subtracted from addSaleInvoices
// Creates separate refund booking rows
// Maintains same structure as original bookings for clarity
```

---

## 14. Summary Tables

### Frontend Component Functions

| Function | Purpose |
|---|---|
| `fetchHistory(page, limit)` | Load report generation history |
| `handlePageChange(newPage)` | Paginate through history |
| `handleSubmit(e, type)` | Submit filters and generate report |

### Backend Controller Functions

| Section | Logic |
|---|---|
| Supplier Filtering | Build WHERE clause based on supp_no filters |
| Date Filtering | Build WHERE clause for invoice_date and created_at |
| Opening Balance | Calculate historical transactions before startDate |
| Service Processing | Extract costs, categorize by type, format data |
| Refund Processing | Identify refunds, subtract from totals |
| Payment Processing | Extract payments, create voucher rows |
| Balance Calculation | Final net balance = opening + sales - refunds - payments - advances + jvCredits - jvDebits |
| Excel Generation | Build workbook with formatted sections |
| PDF Generation | Render EJS template and convert to PDF |

---

## 15. Files to Review for Implementation

### Priority Order
1. **`psfront/src/pages/Report/SupplierAccountStatement.jsx`** - Frontend UI
2. **`psback/controllers/report.controller.js`** (line 9940) - Core logic
3. **`psback/views/pages/reports/supplier-account-statement.ejs`** - PDF template
4. **`psfront/src/api/report.js`** - API integration
5. **`psback/routes/report.route.js`** (line 60) - Route definition
6. **`psback/models/cost.js`** - Cost data model
7. **`psback/models/cost_tax.js`** - Cost tax model

---

## 16. Recent Updates and Fixes (2025-01-19)

### 16.1 Currency Conversion Implementation

**Problem**: Cost documents in foreign currencies were not being converted to PKR, causing incorrect amounts in the supplier account statement.

**Solution Implemented**:
1. Added `currency_code` include to Cost query with nested `currency` relation for exchange rate lookup
2. Implemented currency conversion using the `convertToPKR` service from `currencyConverter.js`
3. All cost amounts (published_rate, net_rate, taxes) are now converted to PKR if in foreign currency
4. Currency conversion applied to:
   - Main service processing loop
   - Historical opening balance calculations
   - All booking types (ticket, hotel, general)

**Files Modified**:
- `/mnt/c/Codes/Powersuite/psback/controllers/report.controller.js` (lines 8826-8833, 8974-8982, 9118-9146)

### 16.2 Debit Note Reference in Refund Section

**Problem**: Debit notes (like BSDN00000001) were missing from the refund section, making verification difficult.

**Solution Implemented**:
1. Added `debit_note` include to refund query with attributes `[doc_no, reference, doc_date, amount]`
2. Added `debitNote` field to refund ticket bookings and refund general bookings
3. Moved standalone supplier debit notes from general bookings to refund general bookings
4. Updated Excel column definitions to include `debitNote` column in refund sections
5. Updated PDF template to display `debitNote` column in refund tables

**Files Modified**:
- `/mnt/c/Codes/Powersuite/psback/controllers/report.controller.js` (lines 8889-8894, 9085-9103, 9214-9257, 9505-9506)
- `/mnt/c/Codes/Powersuite/psback/views/pages/reports/supplier-account-statement.ejs` (lines 214, 218)

### 16.3 Refund Amount Base Currency

**Problem**: Refunds were using `supplier_refund_amount` (original currency) instead of `supplier_refund_amount_base` (PKR).

**Solution Implemented**:
1. Updated refund attributes to include `supplier_refund_amount_base`, `currency_code`, and `exchange_rate`
2. Changed refund amount calculation to use `supplier_refund_amount_base` with fallback to `supplier_refund_amount`
3. Applied to both current period refunds and historical refunds for opening balance

**Files Modified**:
- `/mnt/c/Codes/Powersuite/psback/controllers/report.controller.js` (lines 8886-8888, 9003-9004, 9210-9212, 9050-9053)

### 16.5 Duplicate Debit Notes Per Refund — Opening Balance Fix (2026-04-28)

**Problem**: Opening Balance B/F in the next-period report did not match the previous-period Net Balance. Example: March net = 290,663.92, but April Opening Balance = 371,314.00, off by exactly 80,650.08 (one refund's PKR amount).

**Root cause**:
1. A single refund (e.g. refund_id 434) had **multiple `debit_notes` rows** linked via `debit_note.refund_id`: one with `doc_status='Printed'` (the valid one) and one with `doc_status='Void'` (a stale/cancelled one).
2. The Sequelize association `refund.hasOne(debit_note, { foreignKey: 'refund_id' })` is non-deterministic when multiple rows match — without an `ORDER BY`, MySQL may return either row first.
3. In the **current period** main query, Sequelize happened to pick the Printed debit note → refund counted in `Less Refund Invoices` ✓.
4. In the **opening balance** historical query, Sequelize picked the Void debit note → refund skipped by `if (debitNoteStatus === 'Void') return;` → 80,650.08 not subtracted from opening → opening inflated.

**Solution Implemented**:
1. Added `where: { doc_status: { [Op.notIn]: ['Void', 'Voided'] } }` to both `debit_note` includes (main query and historical query). With `required: false`, Sequelize emits a `LEFT JOIN ... ON dn.refund_id = r.id AND dn.doc_status NOT IN ('Void','Voided')`, so only valid debit notes are joined and `refund.debit_note` is always either the Printed one or `null`.
2. Added a guard in both refund loops: `if (refund.debit_note_generated && !refund.debit_note) return;` — skips refunds where a DN was generated but no valid (non-Void) DN exists. Preserves the prior behavior of skipping all-Void refunds (e.g. refund 433/450) without falling through to the date-only path.
3. Both periods now use the same deterministic logic, so opening balance and current-period totals stay consistent across reports.

**Files Modified**:
- `psback/controllers/report.controller.js`
  - Main query refund→debit_note include (~line 10406): added `where` filter
  - Historical query refund→debit_note include (~line 10610): added `where` filter
  - Opening balance refund loop (~line 10717): added `!refund.debit_note` guard
  - Current period refund loop (~line 11117): added `!refund.debit_note` guard

**Edge cases handled**:
- Refund with one Printed + one Void DN → joins only Printed → counted ✓
- Refund with all Void DNs → join yields null → skipped ✓
- Refund with no DN (debit_note_generated=0) → caught by existing first guard ✓
- Refund with one Printed DN only → unchanged behavior ✓

**Note**: A separate Customer Account Statement query (~line 16435) has the same `refund→debit_note` include pattern without the Void filter. If similar discrepancies appear in customer statements, the same fix should be applied there.

### 16.6 Shared-Settlement Payment Double-Counting Fix (2026-04-28)

**Problem**: Vouchers section in the Supplier Account Statement was showing the same payment row multiple times when one settlement settled multiple costs of the same supplier, and the **Less Payments** total was inflated by the duplicates. The same vulnerability existed in the historical (opening balance) payment loop.

**Root cause**: Payments are reached via `service.Cost.payment_settlement_costs → payment_settlement → payment_settlement_payments`. When a single `payment_settlement` row settles multiple costs of the same supplier, each cost's `payment_settlement_costs` array points to that same settlement, so the same `payment_settlement_payment` row is reachable through every one of those costs. The previous loops did not dedupe by `payment.id`, so the payment was summed once per cost it touched and pushed into the voucher table once per cost.

**Solution Implemented**:
1. **Opening balance payment loop (~line 10688)**: Declared a `Set<payment.id>` outside the per-service loop and added an early-return inside the inner `payments.forEach` so each payment is subtracted from `openingBalance` (and added to `debugPaymentTotal`) at most once for the supplier.
2. **Current-period vouchers loop (~line 11229)**: Same `Set<payment.id>` declared before the services loop, with a guard inside the inner `payments.forEach` so each payment is added to `lessPayments` and pushed into `vouchers` at most once.

**Files Modified**:
- `psback/controllers/report.controller.js`
  - Opening balance: declared `historicalCountedPaymentIds` (~line 10645) and added dedup guard in the `payments.forEach` (~line 10700)
  - Current period: declared `countedVoucherPaymentIds` (~line 10915) and added dedup guard in the voucher `payments.forEach` (~line 11275)

**Edge cases handled**:
- One payment settles N costs of the same supplier → counted once, single voucher row ✓
- Payment with NULL id (defensive) → still counted, dedup only triggers on non-null ids ✓
- Different payments with the same amount but different ids → both counted (dedup is on id, not amount) ✓
- A second supplier's report run reuses fresh Sets (declared per-supplier scope) ✓

**Note**: The matching fix was applied to the Supplier Position Report at the same time (see `SUPPLIER_POSITION_REPORT_ANALYSIS.md` section 10.2), which also had a missing date filter on its historical payment loop.

### 16.7 Opening XO Section (2026-05-19)

**Goal**: Show imported **opening XOs** (supplier cost documents brought in via the Opening XO Import feature) inside the Supplier Account Statement so the supplier's balance reflects them.

**Why it was needed**:
1. Opening XOs are rows in `costs` with `is_opening = 1`; they have **no `documents` row** and **no order**.
2. The report's main cost query joins a costing `document` (INNER JOIN, `document_type: "costing"`), so every opening XO was silently skipped.
3. Opening XOs therefore never appeared and never affected the supplier Net Balance.

**Solution Implemented**:
1. New helper `getOpeningCostsForSuppliers(companyCode, supplierIds)` in `psback/services/openingXo.service.js` — returns opening XO costs (batch type `XO`) for the report's suppliers, matched by `service.supplier_id`.
2. `getSupplierAccountStatementReport` fetches the opening XOs once (after `supplierIds`) and groups them by supplier id; the existing normal-XO query is left untouched.
3. Per supplier, an `openingXo` array is built (columns `date, xo, pax, branch, remarks, net`). PKR amount = `total_costing × exchange_rate`. XO date = `costs.created_at`.
4. The report date filter is applied with `dayKey` (calendar-date compare). Opening XOs dated **before** the period fold into **Opening Balance B/F**; XOs **inside** the period show in the section.
5. New `Opening XO` section renders **immediately after Refund Ticket Bookings** in both Excel and the EJS/PDF template.
6. Summary gets an **"Add Opening XO"** line (`summary.openingXoTotal`); it **increases** the payable, so Net Balance = `opening + sales + openingXO - refunds - payments - advances + jvCredits - jvDebits`. Added to the controller `netBalance`, the Excel `calculatedNetBalance`, and the EJS `calculatedNetBalance`.

**Files Modified**:
- `psback/services/openingXo.service.js` — added `getOpeningCostsForSuppliers`.
- `psback/controllers/report.controller.js` — import, opening-XO fetch + grouping, per-supplier `openingXo` build, `openingXoTotal` in summary, `Opening XO` Excel section, `Add Opening XO` summary row.
- `psback/views/pages/reports/supplier-account-statement.ejs` — `Opening XO` table after Refund Ticket Bookings, `Add Opening XO` summary line, `openingXoTotal` in `calculatedNetBalance`.

**Double-count fix (historical loop)**:
- The historical opening-balance loop scans **every cost of the supplier before the start date** and has **no costing-document filter**, so it also picked up opening XO costs (which have a `Printed` cost row but no `documents` row).
- That made pre-period opening XOs count **twice** in the next period's Opening Balance B/F — once by the historical loop, once by the Opening XO section's `openingXoBeforeStartTotal`.
- Fix: the historical cost loop now skips `is_opening` costs (`if (service.Cost && !service.Cost.is_opening)`). Opening XOs are counted in **one place only** — the Opening XO section.

**Edge cases handled**:
- Opening XO before the period → folded into Opening Balance B/F exactly once (historical loop skips `is_opening` costs).
- Foreign-currency XO → converted to PKR via `total_costing × exchange_rate`.
- Only XO statuses `Printed / Partially Paid / Paid` are included.
- Supplier with no opening XOs → empty section is skipped, `openingXoTotal = 0`.

### 16.8 Ticket/Passenger Line Alignment in Ticket & Refund Ticket Sections (2026-06-08)

**Problem**: When one XO had many tickets, the Ticket Bookings (and Refund Ticket Bookings) row crammed all ticket numbers into one cell and all passenger names into another, each comma-joined and wrapping independently. Ticket #1 didn't line up with passenger #1, and the tall cell overflowed the table header (broken layout).

**Root cause**: ticket numbers and passenger names were built as two *separately* `.filter(Boolean)`-ed arrays and joined with `", "`. A passenger missing a ticket number shifted the two lists out of sync, and the comma-joined strings wrapped at different points.

**Solution Implemented** (one real table row per ticket, internal horizontal borders removed so the group reads as one clean block — same look as the printed XO/Costing document):
1. Build ticket/passenger from a single filtered `service_passengers` list so each ticket number stays paired with its own passenger (placeholder `-` when one side is missing).
2. Per XO, keep the paired `ticketNos`/`passengers` arrays plus date/sector/class/amounts on the booking row through the existing date sort (so dates still sort one-row-per-XO).
3. After the sorts, `expandPaxRows` flattens each XO row into **one row per ticket**: the first ticket row carries XO/date/sector/class/amounts; continuation rows carry only that ticket's number + passenger (other cells blank). Each row also gets `__cont` (is continuation) and `__lastInGroup` flags. Applied to `ticketBookings` and `refundTicketBookings` (supplier function only).
4. Because each ticket number and its passenger physically share a table row, they can never drift out of alignment — even when a long name wraps.
5. PDF (`supplier-account-statement.ejs`): `td` cells `vertical-align: top`; internal horizontal borders suppressed within a group via `border-top:none` when `__cont` and `border-bottom:none` when not `__lastInGroup`, so the per-ticket rows render as one continuous list rather than a boxed grid.
6. **Header overlap fix**: `thead` changed from `table-header-group` (repeated per page) to `table-row-group` (rendered once), so the long ticket list crossing a page no longer collides with a repeated header.
7. Section totals stay correct (amounts only on each XO's first row; continuation rows blank = 0). Excel shows the same per-ticket rows.

**Files Modified**:
- `psback/controllers/report.controller.js` (supplier function) — ticket & refund-ticket build paired `ticketNos`/`passengers` arrays; `expandPaxRows` after the booking sorts.
- `psback/views/pages/reports/supplier-account-statement.ejs` — `td` top-align + group border suppression (`__cont`/`__lastInGroup`); `thead` → `table-row-group`.

**Scope**: Ticket Bookings + Refund Ticket Bookings, supplier function only; Hotel/General and the Customer Account Statement unchanged.

### 16.4 Summary of Changes

**Before**:
- Foreign currency costs shown in original currency
- Debit notes appeared in general bookings without document reference
- Refunds used original currency amounts
- Inconsistent PKR reporting

**After**:
- All costs automatically converted to PKR using exchange rates
- Debit notes appear in refund section with clear document references (e.g., "BSDN00000001")
- Refunds use PKR base amounts for accurate reporting
- Consistent PKR-based supplier account statements

---

## Key Takeaways

1. **XO/Costing Documents** are central to supplier statements - each transaction must have an XO number
2. **Amount Calculation** follows: `net = (published_rate - commission + taxes + wht + freeOfCost) * quantity` (all in PKR after conversion)
3. **Currency Conversion**: All foreign currency costs automatically converted to PKR using exchange rates
4. **Three Transaction Types**: Ticket/Flight, Hotel, General (with corresponding refund types)
5. **Balance Formula**: `netBalance = opening + sales - refunds - payments - advances + jvCredits - jvDebits` (all amounts in PKR)
6. **Debit Note Tracking**: Debit notes appear in refund section with document references for easy verification
7. **Refund Amounts**: Uses `supplier_refund_amount_base` (PKR) for accurate reporting; only refunds with generated debit notes (non-Void) included
8. **Report Generation** supports both Excel (XLSX) and PDF formats with identical data
9. **Opening Balance** calculated separately for date-filtered reports including historical costs, payments, refunds, debit notes, JVs, and advance payments
10. **Payment Tracking** via vouchers tied to payment_settlement records
11. **Company Isolation** enforced at database query level for multi-tenant security
12. **Adjustment Date Mode**: Supports posted-to-ledger filtering using adjustment_date for posted documents, created_at for unposted
13. **Journal Vouchers**: Manual JE entries affecting supplier accounts — credits INCREASE balance, debits DECREASE (opposite of customer accounts)
14. **Advance Payments**: Supplier deposits tracked separately with original and available amounts
15. **Ticket Issue Date**: For Air services uses `ticket_issue_date`, for others uses `Cost.created_at`

---

**Last Updated**: 2026-05-19 — Section 16.7 (Opening XO section added after Refund Ticket Bookings, with "Add Opening XO" summary line)

