# Customer Account Statement Report - Quick Reference

## Core Files
- **Backend Controller**: `/psback/controllers/report.controller.js` (lines 9298-10605)
- **Backend Service**: `/psback/services/customer_balance_calculator.js`
- **Frontend Component**: `/psfront/src/pages/Report/CustomerAccountStatement.jsx`
- **PDF Template**: `/psback/views/pages/reports/customer-account-statement.ejs`
- **API Wrapper**: `/psfront/src/api/report.js`

## API Endpoint
```
POST /report/getCustomerAccountStatementReport
Timeout: 5 minutes
Auth: JWT + "customer-account-statement" permission
```

## Input Filters

### Customer Filter
- `isNotBlank` - All customers (default)
- `isBlank` - No customers
- `isEqual` - Single customer (requires customer_id)
- `between` - Range (requires customer_idStart, customer_idEnd)

### Date Filter
- `blank` - No filtering (default)
- `=`, `<`, `<=`, `>`, `>=`, `<>` - Single date comparisons
- `between` - Date range (requires startDate, endDate)

### Output
- `pdf` - PDF format via EJS template
- `excel` - XLSX format via ExcelJS

## Key Calculations

```
Opening Balance = Historical Invoices - Historical Receipts 
                - Historical Credit Notes - Historical Refunds 
                - Historical Deposits (remaining balance)

Net Balance = Opening Balance + Period Invoices 
            - Period Refunds - Period Receipts 
            - Period Deposits (remaining balance)
```

## Invoice Amount Formula
```
basePrice = invoice.price
discount = (basePrice * invoice.discount%) 
rebate = (basePrice * invoice.rebate%)
unitPriceBeforeTax = basePrice - discount - rebate
taxes = SUM(invoice_taxes)
subtotal = (unitPriceBeforeTax + taxes) * invoice.quantity

// For hotels, multiply by rooms
totalAmount = (subtotal * numberOfRooms) + transactionFee + sstAmount

// Convert currency
amountInPKR = totalAmount * exchangeRate
```

## Key Database Models (19 total)

**Primary Entities**:
- customer, user, order, service, invoice, refund

**Payment/Receipt Models**:
- receipt_settlement, receipt_settlement_invoice, receipt_settlement_payment, 
- receipt_settlement_credit_note, receipt_settlement_deposit

**Deposit Models**:
- customer_deposit, pay_type_form, chart_of_account

**Service Details**:
- service_flight, service_hotel, service_passenger, service_type

**Financial**:
- invoice_tax, credit_note, document

**Currency**:
- currency_code, currency

**Reporting**:
- report

## Report Sections

1. **Ticket Bookings** - Flight invoices (by passenger)
2. **Hotel Bookings** - Hotel invoices (with room multiplier)
3. **General Bookings** - Other service invoices
4. **Refund Ticket Bookings** - Flight refunds
5. **Refund General Bookings** - Other refunds + credit notes
6. **Receipts** - Payment records with settlement details
7. **Deposits** - Prepayments (showing remaining balance only)
8. **Summary** - Financial totals

## Status Filters (Void Records Excluded)

- Invoice: Only "Printed", "Settled", "Partially Settled"
- Refund: Excludes "Void"
- Receipt: Excludes "Void"
- Deposit: Excludes "Void"
- Credit Note: Excludes "Void"

## Special Handling

- **Hotels**: Subtotal multiplied by number of rooms
- **Flights**: One row per passenger
- **Currency**: Auto-converted to PKR using exchange_rate
- **Deposits**: Only shown if remaining balance > 0
- **Deduplication**: Uses Set to track processed invoice IDs

## Output Structure

### PDF
- EJS template rendering
- wkhtmltopdf conversion
- Page breaks between customers
- Professional formatting with borders and shading

### Excel
- Frozen header rows (first 6)
- Multiple worksheets (one per report)
- Auto-width columns (10-50px)
- Formatted numbers with 2 decimals
- Section headers with gray background
- Settlement details with indentation

## Data Flow

1. Frontend sends POST with filters
2. Backend validates auth & permissions
3. Fetches customers (with company code check)
4. Fetches transactions (orders, deposits, receipts, credit notes)
5. Calculates opening balance (if date range provided)
6. Transforms data by service type
7. Generates PDF or Excel
8. Uploads to S3/MinIO
9. Returns download link + report record

## Performance Notes

- Uses separate queries instead of complex JOINs
- Deduplicates invoices with Set
- Tracks query execution time in logs
- Typical execution: 100-500ms for medium datasets
- 5-minute timeout for long-running reports

## Company Isolation

All customer queries filtered by:
```
user.company_code = req.user.company_code
```

## File Size
- Controller: ~1300 lines of code
- Service: 240 lines of code
- Template: 700+ lines of EJS
- Frontend: 318 lines of React

