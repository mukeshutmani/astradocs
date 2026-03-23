# Customer Account Statement Report - Visual Diagrams & Architecture

## 1. Data Flow Diagram

```
Frontend (React)
    |
    v
CustomerAccountStatement.jsx
    |
    | (User enters filters & clicks Generate)
    |
    v
POST /report/getCustomerAccountStatementReport
    |
    v
Backend Express Router
    |
    ├─ Authenticate (JWT)
    ├─ Check Permission ("customer-account-statement")
    |
    v
report.controller.js::getCustomerAccountStatementReport()
    |
    ├─ Start DB Transaction
    |
    ├─ Parse Filters
    |   ├─ customerFilter (isEqual/between/isNotBlank/isBlank)
    |   ├─ dateFilter (=/</<=/>/>=/<>/between)
    |   └─ type (pdf/excel)
    |
    ├─ Build WHERE clauses
    |
    ├─ Query Customers
    |   └─ Filter by company_code (multi-tenancy)
    |
    ├─ Query Orders + Services + Invoices
    |   └─ Include: flight, hotel, passenger, tax details
    |
    ├─ Query Customer Deposits
    |   └─ Include: payment type, GL account, currency
    |
    ├─ Query Deposit Usage
    |   └─ For remaining balance calculation
    |
    ├─ Query Receipt Settlements
    |   └─ Include: invoices, payments, credit notes, deposits used
    |
    ├─ Query Credit Notes
    |
    └─ IF date range provided:
        ├─ Query Historical Orders (before startDate)
        ├─ Query Historical Receipts
        ├─ Query Historical Deposits
        └─ Query Historical Credit Notes
            |
            v
            Calculate Opening Balance
                = Invoices - Receipts - Credit Notes - Refunds - Deposits
    
    |
    ├─ Transform Data by Service Type
    |   ├─ Ticket Bookings (flight)
    |   ├─ Hotel Bookings
    |   └─ General Bookings (other)
    |
    ├─ Calculate Period Totals
    |   ├─ Period Invoices
    |   ├─ Period Refunds
    |   ├─ Period Receipts
    |   └─ Period Deposits
    |
    ├─ Calculate Net Balance
    |   = Opening + Invoices - Refunds - Receipts - Deposits
    |
    ├─ Create Report Record in DB
    |
    ├─ IF type === "excel"
    |   └─ Generate with ExcelJS
    |       └─ Format: headers, sections, summary
    |
    ├─ ELSE IF type === "pdf"
    |   ├─ Render EJS Template
    |   └─ Convert to PDF with wkhtmltopdf
    |
    ├─ Upload to S3/MinIO
    |
    └─ Return Response
        {
          status: 200,
          link: "S3 URL",
          downloadLink: "signed URL",
          report: { id, number, type, etc }
        }
    
    v
Frontend receives response
    |
    ├─ IF excel: Download file
    ├─ IF pdf: Navigate to viewer
    └─ Refresh report history
```

## 2. Database Model Relationships

```
┌─────────────────────────────────────────────────────────────────┐
│                      CUSTOMER ACCOUNT STATEMENT FLOW             │
└─────────────────────────────────────────────────────────────────┘

                           ┌──────────┐
                           │ customer │
                           │ ├─ name  │
                           │ ├─ addr  │
                           │ └─ STN   │
                           └────┬─────┘
                                │
                ┌───────────────┼───────────────┐
                │               │               │
                v               v               v
            ┌─────────┐   ┌─────────┐   ┌────────────┐
            │  order  │   │ deposit  │   │ receipt    │
            └────┬────┘   │          │   │ settlement │
                 │        └──────────┘   └────┬───────┘
                 │                            │
                 v                            │
            ┌─────────┐                       │
            │ service │                       │
            └────┬────┘                       │
                 │                            │
    ┌────────────┼────────────┐               │
    │            │            │               │
    v            v            v               │
 ┌───────┐ ┌──────────┐ ┌──────────┐        │
 │flight │ │  hotel   │ │passenger │        │
 └───────┘ └──────────┘ └──────────┘        │
    │            │                          │
    v            v                          v
 ┌─────────┐ ┌─────────┐ ┌──────────────┐ ┌──────────────────┐
 │ invoice │ │ invoice │ │ refund       │ │ receipt_settlement
 │ ├─ tax  │ │ ├─ tax  │ └──────────────┘ │ ├─ invoice link
 │ └─ amt  │ │ └─ amt  │                  │ ├─ payment method
 └─────────┘ └─────────┘                  │ ├─ credit note link
                                          │ └─ deposit link
                                          └──────────────────┘

        ┌─────────────────────────────────┐
        │      Currency Conversion        │
        │  (All to PKR via exchange_rate) │
        └─────────────────────────────────┘
```

## 3. Invoice Calculation Flow

```
START: invoice + service details
    |
    v
┌──────────────────────────────────────┐
│  Extract Base Components             │
│  ├─ basePrice = invoice.price        │
│  ├─ discount% = invoice.discount     │
│  ├─ rebate% = invoice.rebate         │
│  ├─ quantity = invoice.quantity      │
│  ├─ taxes = SUM(invoice_taxes)       │
│  ├─ transactionFee = invoice.fee     │
│  ├─ sstPercent = invoice.sst         │
│  └─ exchangeRate = currency rate     │
└──────────┬───────────────────────────┘
           |
           v
    ┌─────────────────────────────┐
    │ Calculate Unit Price        │
    │ discount_amt = basePrice *  │
    │              discount% / 100│
    │ rebate_amt = basePrice *    │
    │            rebate% / 100    │
    │                             │
    │ unitPriceBeforeTax =        │
    │   basePrice - discount_amt  │
    │           - rebate_amt      │
    │                             │
    │ unitPriceWithTax =          │
    │   unitPriceBeforeTax + taxes│
    └────────┬────────────────────┘
             |
             v
    ┌──────────────────────────┐
    │ Calculate Subtotal       │
    │ subtotal =               │
    │   unitPriceWithTax *     │
    │   quantity               │
    └────────┬─────────────────┘
             |
             v
    ┌──────────────────────────┐
    │ Check Service Type       │
    └────────┬─────────────────┘
             |
    ┌────────┴─────────┐
    |                  |
    v                  v
 HOTEL?            OTHER?
    |                |
    v                v
 ┌─────────────┐  ┌──────────┐
 │ Multiply by │  │ Room     │
 │ # of rooms  │  │ multiplier=1
 │             │  └──────────┘
 │ roomTotal = │
 │  subtotal * │
 │  no_of_rooms│
 └────────┬────┘
          |
          └──────────┬───────────┘
                     |
                     v
         ┌───────────────────────────┐
         │ Add Transaction Fee & SST │
         │                           │
         │ sstAmount =               │
         │   transactionFee *        │
         │   sstPercent / 100        │
         │                           │
         │ totalAmount =             │
         │   (subtotal * rooms) +    │
         │   transactionFee +        │
         │   sstAmount               │
         └───────────┬───────────────┘
                     |
                     v
         ┌───────────────────────────┐
         │ Apply Currency Conversion │
         │                           │
         │ amountInPKR =             │
         │   totalAmount *           │
         │   exchangeRate            │
         └───────────┬───────────────┘
                     |
                     v
               FINAL AMOUNT (PKR)
```

## 4. Balance Calculation Architecture

```
                    OPENING BALANCE CALCULATION
                    (if date range provided)
                            |
                ┌───────────┬───────────┬───────────┬───────────┐
                |           |           |           |           |
                v           v           v           v           v
         Historical   Historical   Historical Historical  Historical
         Invoices     Refunds      Receipts   Deposits   Credit Notes
           |             |           |          |            |
           v             v           v          v            v
        SUM()         SUM()       SUM()      BALANCE()     SUM()
        (Using        (Using      (Direct)   (Remaining    (Using
         formula)     formula)             minus used)    refund_amt)
           |             |           |          |            |
           └─────┬───────┴───────────┴──────────┴────────────┘
                 |
                 v
         Opening Balance Formula:
         ┌───────────────────────────────────┐
         │ OB = Invoices - Receipts          │
         │       - Refunds - Deposits        │
         │       - Credit Notes              │
         └───────────────────────────────────┘
                 |
                 v
    ┌──────────────────────────────┐
    │   Period Totals (Same Logic) │
    │   ├─ Period Invoices         │
    │   ├─ Period Refunds          │
    │   ├─ Period Receipts         │
    │   ├─ Period Deposits         │
    │   └─ Period Credit Notes     │
    └────────────┬─────────────────┘
                 |
                 v
         ┌─────────────────────────┐
         │  Net Balance Formula     │
         │                         │
         │ NB = Opening +          │
         │      Period Invoices -  │
         │      Period Refunds -   │
         │      Period Receipts -  │
         │      Period Deposits    │
         └─────────────────────────┘
```

## 5. Report Structure Hierarchy

```
REPORT (customer-account-statement)
│
├─ HEADER
│  ├─ Report ID (CUSTSTMT + timestamp)
│  ├─ Company Name & Address
│  ├─ Printed By & Date/Time
│  ├─ Customer No.
│  ├─ Document Date Range
│  └─ Page Number
│
├─ FOR EACH CUSTOMER:
│  │
│  ├─ CUSTOMER INFO
│  │  ├─ Name
│  │  ├─ Address
│  │  └─ Sales Tax Number
│  │
│  ├─ SALES INVOICES SECTION
│  │  ├─ TICKET BOOKINGS (if any flights)
│  │  │  ├─ Date | Invoice | Ticket# | Passenger | Sector | Class
│  │  │  ├─ Fare | Taxes | Discount | Rebate | T.Fee | SST | Net
│  │  │  └─ [One row per passenger]
│  │  │
│  │  ├─ HOTEL BOOKINGS (if any hotels)
│  │  │  ├─ Date | Invoice | Hotel | Passenger | Chk-in | Chk-out
│  │  │  ├─ Room Type | Rooms | Nights | Net
│  │  │  └─ [One row per invoice]
│  │  │
│  │  └─ GENERAL BOOKINGS (if any other)
│  │     ├─ Date | Invoice | Pax | Ref No | Remarks | Vendor | Net
│  │     └─ [One row per invoice]
│  │
│  ├─ REFUND INVOICES SECTION
│  │  ├─ REFUND TICKET BOOKINGS (if any refunds)
│  │  │  └─ [Same structure as sales tickets]
│  │  │
│  │  └─ REFUND GENERAL BOOKINGS
│  │     ├─ Refund records
│  │     └─ Credit note records
│  │
│  ├─ RECEIPTS SECTION
│  │  ├─ Main Receipt Row
│  │  │  ├─ Date | Receipt No. | Reference | Description
│  │  │  ├─ Payment Method | Amount
│  │  │
│  │  └─ FOR EACH RECEIPT:
│  │     ├─ [Optional] Invoices Settled: INV_NUM (Amt), ...
│  │     ├─ [Optional] Payment Methods: GL_ACC (Amt) [Deposit] ...
│  │     ├─ [Optional] Credit Notes Applied: CN_NUM (Amt), ...
│  │     └─ [Optional] Deposits Used: DEP_NUM (Amt), ...
│  │
│  ├─ DEPOSITS SECTION
│  │  ├─ Date | Deposit No. | Reference | Description | Cheque No | Amount
│  │  └─ [Only showing remaining balance > 0]
│  │
│  └─ SUMMARY SECTION
│     ├─ Opening Balance B/F
│     ├─ Add Sale Invoices
│     ├─ Less Refund Invoices
│     ├─ Less Receipts
│     ├─ Less Deposits
│     └─ Net Balance
│
└─ [PAGE BREAK]
   [Next Customer if multiple]
```

## 6. Database Query Sequence

```
STEP 1: Get Customers
────────────────────
SELECT c.* FROM customers c
JOIN users u ON c.user_id = u.id
WHERE {filter} AND u.company_code = ?
     → customers[] 

STEP 2: Get Orders + Services + Invoices
──────────────────────────────────────────
SELECT o.*, s.*, i.*, it.*
FROM orders o
  JOIN services s ON o.id = s.order_id
  LEFT JOIN invoices i ON s.id = i.service_id
  LEFT JOIN invoice_taxes it ON i.id = it.invoice_id
  LEFT JOIN service_flights sf ON s.id = sf.service_id
  LEFT JOIN service_hotels sh ON s.id = sh.service_id
WHERE o.customer_id IN (customer_ids)
  AND i.status IN ('Printed', 'Settled', 'Partially Settled')
  AND {date filter}
     → orders[] with nested services, invoices, taxes

STEP 3: Get Customer Deposits
──────────────────────────────
SELECT cd.*, ptf.*, coa.*, cc.*, c.*
FROM customer_deposits cd
  LEFT JOIN pay_type_forms ptf ON cd.pay_type_form_id = ptf.id
  LEFT JOIN chart_of_accounts coa ON cd.chart_of_account_id = coa.id
  LEFT JOIN currency_codes cc ON cd.currency_code_id = cc.id
  LEFT JOIN currencies c ON cc.id = c.currency_code_id AND c.to_currency = 'PKR'
WHERE cd.customer_id IN (customer_ids)
  AND cd.status != 'Void'
  AND {date filter}
     → deposits[] with payment type, GL account, currency

STEP 4: Get Deposit Usage
──────────────────────────
SELECT * FROM receipt_settlement_deposits
WHERE customer_deposit_id IN (deposit_ids)
     → depositUsages[] for balance calculation

STEP 5: Get Receipt Settlements
────────────────────────────────
SELECT rs.*, rsi.*, rsp.*, rscn.*, rsd.*
FROM receipt_settlements rs
  LEFT JOIN receipt_settlement_invoices rsi
  LEFT JOIN receipt_settlement_payments rsp
  LEFT JOIN receipt_settlement_credit_notes rscn
  LEFT JOIN receipt_settlement_deposits rsd
WHERE rs.customer_id IN (customer_ids)
  AND rs.status != 'Void'
  AND {date filter}
     → receiptSettlements[] with all settlement details

STEP 6: Get Credit Notes
─────────────────────────
SELECT * FROM credit_notes
WHERE customer_id IN (customer_ids)
  AND doc_status != 'Void'
  AND {date filter on doc_date}
     → creditNotes[]

STEP 7 (Optional): Get Historical Data (if date range)
───────────────────────────────────────────────────────
Repeat steps 2-6 but with condition: date < startDate
     → historical[Orders, Deposits, Receipts, Credit Notes]

All steps executed within database transaction
```

## 7. File Generation Architecture

```
Report Data Structure
       │
       ├─ PDF Generation
       │  │
       │  ├─ Render EJS Template
       │  │  ├─ Pass: header, customerStatements
       │  │  └─ Generate HTML
       │  │
       │  └─ Convert HTML to PDF
       │     ├─ Use wkhtmltopdf
       │     ├─ Apply CSS styling
       │     ├─ Add page breaks
       │     └─ Generate binary PDF
       │
       └─ Excel Generation
          │
          ├─ Create Workbook (ExcelJS)
          │
          ├─ Add Worksheet (frozen headers)
          │
          ├─ For Each Customer:
          │  ├─ Add header section (rows 1-6)
          │  ├─ Add customer info (rows 7-9)
          │  ├─ For each transaction section:
          │  │  ├─ Add section title (bold, gray bg)
          │  │  ├─ Add column headers
          │  │  ├─ Add data rows
          │  │  ├─ Format cells (borders, alignment)
          │  │  └─ Format numbers (2 decimals, right-aligned)
          │  ├─ Add summary section
          │  └─ Add page break
          │
          └─ Write to buffer & upload
```

## 8. Currency Conversion Points

```
Input Data (Multiple Currencies)
    │
    ├─ Invoices
    │  └─ invoice.currency_code.currencies[0].exchange_rate
    │
    ├─ Refunds
    │  └─ refund.currency_code_refund.currencies[0].exchange_rate
    │
    ├─ Deposits
    │  └─ deposit.currency_code.currencies[0].exchange_rate
    │
    ├─ Receipt Payments
    │  └─ receipt_settlement_payment.exchange_rate
    │
    └─ Credit Notes
       └─ receipt_settlement_credit_note.exchange_rate

All values × exchange_rate = PKR Amount

[Fallback: exchange_rate = 1 if not found]

Output: All amounts in PKR
```

## 9. Error Handling Flow

```
Request Received
    │
    v
Authentication Check
    ├─ Valid JWT? → Continue
    └─ Invalid → 401 Unauthorized
    
Permission Check ("customer-account-statement")
    ├─ Permitted? → Continue
    └─ Not permitted → 403 Forbidden

Database Query Execution
    ├─ Success → Continue
    └─ Error → Log & 500 Internal Server Error

Data Transformation
    ├─ Success → Continue
    └─ Error → Log & 500 Internal Server Error

File Generation
    ├─ Success → Continue
    └─ Error → Log & 500 Internal Server Error

File Upload
    ├─ Success → 200 with response
    └─ Error → Log & 500 Internal Server Error

All errors logged with:
├─ Error message
├─ Stack trace
└─ Request details
```

## 10. Performance Optimization Points

```
Optimization Strategy          Implementation
────────────────────────────── ──────────────────────────────
Avoid N+1 queries             Use include() for related data
                              Batch separate queries

Prevent complex JOINs         Execute 7 separate queries
                              Let application layer assemble

Deduplicate data              Track processed invoice IDs
                              Use Set for O(1) lookup

Lazy filter evaluation        Build WHERE clauses
                              Pass to Sequelize

Date filter efficiency        Create indexes on:
                              - invoice.invoice_date
                              - deposit.created_at
                              - receipt.created_at
                              - credit_note.doc_date

Currency conversion           Done at query level
                              Not in reporting loop

Report caching               Store in database
                             Retrieved with pagination
```

