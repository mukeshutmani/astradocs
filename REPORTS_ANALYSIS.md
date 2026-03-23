# PowerSuite Reports Analysis Documentation

## Executive Summary

This document provides a comprehensive analysis of all reports in the PowerSuite system, detailing their data sources, calculation methods, business logic, and potential areas for verification. The analysis was conducted to understand report generation processes and identify areas requiring validation for data accuracy.

## Table of Contents

1. [Report Categories](#report-categories)
2. [Individual Report Analysis](#individual-report-analysis)
3. [Key Calculations and Formulas](#key-calculations-and-formulas)
4. [Data Flow and Dependencies](#data-flow-and-dependencies)
5. [Potential Data Accuracy Issues](#potential-data-accuracy-issues)
6. [Recommendations for Verification](#recommendations-for-verification)

---

## Report Categories

The PowerSuite system contains **24 distinct report types** organized into 4 main categories:

### 1. **Accounts Payable (AP) Reports**
- Supplier Listing Report
- AP Ageing Analysis Report
- Supplier Position Report
- Supplier Account Statement Report
- Payment Settlement Report

### 2. **Accounts Receivable (AR) Reports**
- AR Ageing Analysis Detail Report
- AR Ageing Analysis Summary Report
- Customer Account Statement Report
- Customer Position Detail Report
- Customer Deposit Movement Report

### 3. **History/Operational Reports**
- Chart of Account Report
- Inventory Report
- Refund Report
- Daily Invoice Report
- Daily Settlement Report
- Trail Balance By Journal Period Report
- Customer Profile Listing Report
- Airline Sales Report
- Ticket By Airline Report
- Bank Activity Report
- Cash Account Balance Report
- Credit Note Report

### 4. **Management Information System (MIS) Reports**
- Customer Volume Report

---

## Individual Report Analysis

### 1. Customer Volume Report
**Purpose**: Analyzes customer sales and revenue across different service types

**Key Data Sources**:
- Tables: `customers`, `orders`, `services`, `invoices`, `costs`, `invoice_taxes`, `cost_taxes`
- Service Types: Air, Visa, Hotel, Insurance, Car Rental

**Calculations**:
```
Total Sales = Invoice.total_price
Total Costs = Cost.total_costing
Revenue = Total Sales - Total Costs
Margin = (Revenue / Total Sales) × 100

For Air Tickets:
- Sales Without Tax = Total Sales - Invoice Tax Amount
- Air Total = Sales + Taxes
```

**Filters Available**:
- Date range (multiple operators: =, <, >, between, etc.)
- Branch
- Product Code/Service Type
- Sales ID
- Airline
- Customer (single or range)

**Output Formats**: PDF, Excel

---

### 2. Daily Sale Report
**Purpose**: Comprehensive daily view of sales and costs broken down by service type with profit/loss per invoice

**Key Data Sources**:
- Tables: `invoices`, `documents`, `services`, `service_types`, `costs`, `cost_taxes`, `invoice_taxes`, `invoice_discounts`, `orders`, `customers`, `service_passengers`, `passengers`, `credit_notes`, `refunds`, `debit_notes`

**Layout**: Landscape orientation, one row per invoice

**Columns**: Invoice No, Inv Date, PNR, Status, Client Name, Receivable by type (Air/Visa/Hotel/Ins/Car/Cruise/Tour/Train/Misc), Total Sales, Payable by type, Total Cost, Profit/Loss, Pax

**Calculations**:
```
Sales = invoice.total_price (includes SST, currency converted if needed)

Cost Per Unit:
  commissionAmt = published_rate × commission% / 100
  netRate = published_rate - commissionAmt
  costPerUnit = netRate + free_of_cost + sum(cost_taxes) + WHT
  WHT = commissionAmt × cost.sst% / 100
  totalCost = costPerUnit × cost.quantity

Profit/Loss = Sales - Total Cost
```

**Business Logic**:
- Default status filter: Printed, Settled, Partially Settled (excludes Void, Raised)
- Air services use `ticket_issue_date` if company setting enabled, otherwise `invoice_date`
- Active cost = most recent non-Void cost per service
- Cost quantity multiplier (typically matches passenger count)
- Multi-currency conversion to PKR
- Credit notes, refunds, debit notes fetched for adjusted calculations
- Summary table with per-category and grand totals

**Verified**: March 2026 - All calculations correct. No bugs found.

**Documentation**: See `docs/DAILY_SALE_REPORT.md` for full technical documentation.

---

### 3. AR Ageing Analysis Detail Report
**Purpose**: Analyzes accounts receivable aging to identify overdue payments with individual invoice detail

**Key Data Sources**:
- Tables: `customers`, `invoices`, `customer_deposits`, `credit_notes`, `receipt_settlement_invoices`, `receipt_settlement_deposits`

**Ageing Buckets**:
- Current (within credit period, days overdue <= 0)
- 1-30 days overdue
- 31-60 days overdue
- 61-90 days overdue
- 91-120 days overdue
- 121+ days overdue

**Calculations**:
```
Due Date = Invoice Date + Credit Terms (credit_days_1, BSP terms, etc.)
Days Overdue = As-Of-Date - Due Date
Outstanding Amount = Invoice Total - Sum(Non-Void Settlements)
Net Outstanding = Invoice Total - Deposits - Credit Notes (allows negative/credit balance)
```

**Special Features**:
- Point-in-time accuracy (all queries filtered by as-of date)
- Multi-currency support with PKR conversion (dual tracking)
- Invoice deduplication by invoice_number
- Deposit deduplication by receipt_number
- Void settlement exclusion
- Credit balance support (negative net outstanding allowed)
- Customer credit term-aware due date calculation

**Documentation**: See `docs/AR_AGEING_ANALYSIS_DETAIL_REPORT.md` for full technical documentation.

---

### 3b. AR Ageing Analysis Summary Report
**Purpose**: Summarized view of AR aging with one row per customer

**Key Data Sources**: Same as Detail Report

**Calculations**:
```
Per Customer: Ageing bucket totals + Net Outstanding (invoices - deposits - credit notes)
Negative values shown in accounting bracket format (60,000) in red
```

**Special Features**:
- Accounting bracket format for negative values (credit balances)
- Matches Detail Report totals exactly
- Average days overdue per customer

**Documentation**: See `docs/AR_AGEING_ANALYSIS_SUMMARY_REPORT.md` for full technical documentation.

---

### 3c. Customer Deposit Movement Report
**Purpose**: Tracks customer deposit lifecycle - creation, settlement usage, and current balance

**Key Data Sources**:
- Tables: `customer_deposits`, `receipt_settlement_deposits`, `receipt_settlements`, `customers`, `branches`, `currency_codes`

**Calculations**:
```
Per Deposit:
  Credit = Deposit Amount × Exchange Rate
  Debit = Sum(All Settlement Amounts × Exchange Rate)
  Sub-Total = Credit - Debit (current balance)

Grand Total:
  Amount = Sum(Sub-Totals)
  Debit = Sum(All Debits)
  Credit = Sum(All Credits)
```

**Special Features**:
- Multi-line rows: Each deposit shows creation + all settlement lines with sub-total
- Adjustment date mode: Filter by ledger posting date or creation date
- Multi-currency with exchange rate conversion
- Settlement details include receipt settlement number, date, and remarks
- Grouped by deposit receipt_number

**Bug Fix (March 2026)**: Fixed missing settlements due to `hasOne` model association. Now queries all settlements separately via `receipt_settlement_deposit.findAll()`.

**Documentation**: See `docs/adjustment-date.md` for adjustment date feature details.

---

### 4. Customer Position Report
**Purpose**: Summary view of each customer's financial position — one row per customer with opening balance, invoices, refunds, receipts, deposits, payments, JV adjustments, and net balance

**Key Data Sources**:
- Tables: `customers`, `orders`, `services`, `invoices`, `invoice_taxes`, `customer_deposits`, `receipt_settlements`, `receipt_settlement_payments`, `credit_notes`, `payment_settlements`, `journal_entries`, `journal_batches`, `invoice_settings`, `currencies`, `branches`

**Columns**: Customer No, Customer Name, Opening Balance B/F, ADD:Sales Invoice, LESS:Refunds, LESS:Receipts, ADD:Payments, ADD:JV Debit, LESS:JV Credit, Net Balance

**Calculations**:
```
Opening Balance = historicalInvoices - historicalReceipts(GL only) - historicalCreditNotes
                - historicalDeposits + historicalPayments + historicalJvDebit - historicalJvCredit

Net Balance = openingBalance + periodInvoices - periodRefunds(credit notes)
            - periodReceipts(GL + deposits) + periodPayments + jvDebit - jvCredit
```

**Key Features**:
- Service refunds excluded — only credit notes count as refunds
- Deposits use full values (not remaining balance)
- Receipts count GL account payments only (deposit payments excluded)
- Adjustment Date Mode support (Posted to Ledger)
- Ticket Issue Date support (for Air services, per company invoice_settings)
- Journal entries: Manual JE only (batch_type = 'Manual JE')
- 7-step query approach for performance

**Documentation**: See `docs/CUSTOMER_POSITION_REPORT.md` for full technical documentation.

---

### 5. Inventory Report
**Purpose**: Tracks ticket inventory with cost analysis

**Key Data Sources**:
- Tables: `service_passengers`, `services`, `costs`, `invoices`, `service_flights`

**Cost Calculations**:
```
Published Rate = cost.published_rate
Commission Amount = (Published Rate × Commission%) / 100
Net Rate = Published Rate - Commission Amount
Regular Tax = Sum of cost_taxes
WHT Amount = (Commission Amount × WHT%) / 100
Total Cost = Net Rate + Regular Tax + WHT Amount
```

---

### 6. Supplier Position Report
**Purpose**: Tracks amounts owed to suppliers

**Key Data Sources**:
- Tables: `suppliers`, `services`, `costs`, `payment_settlements`

**Calculations**:
Similar to Customer Position but for payables:
```
Outstanding = Total Costs - Payments Made
Net Balance = Opening Balance + New Costs - Payments
```

---

### 7. Bank Activity Report
**Purpose**: Shows all journal entry transactions associated with bank accounts, grouped by bank name with per-bank net totals and grand total

**Key Data Sources**:
- Tables: `journal_entries`, `journal_batches`, `chart_of_accounts`, `bank_accounts`, `banks`, `branches`

**Columns**: Batch No, Entry No, Date, Account No, Line No, Reference, Cheque No, DR Amount, CR Amount

**Grouping**: Entries grouped by bank name (from `banks.name`), sorted by most recent transaction date. Within each bank, entries sorted by date descending.

**Key Field Mappings**:
- Line No = `journal_entry.id` (primary key, NOT entry_no)
- Reference = `journal_entry.description` + `journal_entry.analysis_code1` (with "undefined" removed)
- Cheque No = `journal_entry.analysis_code5`
- Account No = `bank_account.account_number`

**Calculations**:
```
Per-Bank:
  totalDr = sum(entry.debit) for all entries in bank group
  totalCr = sum(entry.credit) for all entries in bank group

Grand Total:
  grandTotalDr = sum(totalDr) across all banks
  grandTotalCr = sum(totalCr) across all banks
```

**Filtering**: Only entries linked to bank accounts are included (`bank_account` join is `required: true`). Company scoped via `journal_batch → branch → company_code`.

**Verified**: March 2026 - All values verified correct against database. No bugs found.

**Documentation**: See `docs/BANK_ACTIVITY_REPORT.md` for full technical documentation.

---

### 8. Trail Balance By Journal Period
**Purpose**: Financial trail balance report by accounting period

**Key Data Sources**:
- Tables: `journal_entries`, `journal_periods`, `chart_of_accounts`

**Calculations**:
```
Opening Balance = Previous period closing balance
Period Movement = Sum of period transactions
Closing Balance = Opening Balance + Period Movement
```

---

### 9. Airline Sales Report
**Purpose**: Per-passenger breakdown of airline ticket sales, costs, and profitability

**Key Data Sources**:
- Tables: `services`, `service_types`, `service_passengers`, `service_flights`, `invoices`, `invoice_taxes`, `costs`, `cost_taxes`, `airline_codes`, `city_codes`, `orders`, `customers`

**Row Level**: One row per passenger per service

**Calculations**:
```
Sales Per Passenger:
  sstOnTFee = (invoice.transaction_fee × invoice.sst%) / 100
  totalSales = invoice.total_price - sstOnTFee
  salesPerPax = totalSales / numPassengers

Cost Per Passenger:
  commissionAmt = (cost.published_rate × cost.commission%) / 100
  whtAmt = (commissionAmt × cost.sst%) / 100
  costPerPax = cost.net_rate + cost.free_of_cost + sum(cost_taxes) + whtAmt

Profit = salesPerPax - costPerPax
Margin% = (Profit / salesPerPax) × 100
Net = salesPerPax + (sstOnTFee / numPassengers)
```

**Inclusion Criteria**:
- Service type must be 'Air'
- Invoice must have `invoice_number` (excludes Raised without number)
- Must have at least one passenger
- Must have at least one non-Void cost
- Void costs excluded from query

**Columns**: Product Code, Ticket Issue Date, Customer, Airline, Ticket Number, Routing, First Departure, Invoice Number/Date, Publish, Extra Charges, Commission, WHT, T-Cost, Cost Tax, Sales, Sales Tax, T-Fee, SST, Net, Profit, M%, TCID

**Verified**: March 2026 - All calculations verified correct against database. No bugs found.

**Documentation**: See `docs/AIRLINE_SALES_REPORT.md` for full technical documentation.

---

### 10. Payment Listing Report
**Purpose**: Lists all payment settlements to suppliers with payment method details, bank/cash box, exchange rates, and gain/loss

**Key Data Sources**:
- Tables: `payment_settlements`, `payment_settlement_payments`, `payment_settlement_costs`, `suppliers`, `chart_of_accounts`, `pay_type_forms`, `currency_codes`, `currencies`, `costs`, `services`, `documents`

**Columns**: Payment No, Payee Name, Date/Doc No, Status, Void Amount, Form of Payment, Bank/Cash Box, Cheque Date, Paid Amount, Overpayment, Ex. Rate, Gain/Loss Amt, Base Amt

**Calculations**:
```
Per Row:
  amountLocal = payment.amount × (exchange_rate || 1)
  Printed → Paid Amount = amountLocal, Base Amt = amountLocal
  Void → Void Amount = amountLocal, Base Amt = 0

Totals:
  Total Paid Amount = sum(Printed amounts)
  Total Void Amount = sum(Void amounts)
  Net Amount PKR = Total Paid - Total Void
  Total Paid Amount PKR = Net Amount PKR
```

**Special Features**:
- Adjustment Date mode support (Posted to Ledger)
- Date/Doc No combines payment date with XO costing document number
- Settlements without payment records shown with fallback values
- Multi-currency with exchange rate conversion

**Verified**: March 2026 - All calculations correct. No bugs found.

**Documentation**: See `docs/PAYMENT_LISTING_REPORT.md` for full technical documentation.

### 11. Daily Settlement Report
**Purpose**: Shows all receipt settlements from customers with deposits, credit notes, invoices settled, and GL account payments

**Key Data Sources**:
- Tables: `receipt_settlements`, `receipt_settlement_invoices`, `receipt_settlement_deposits`, `receipt_settlement_credit_notes`, `receipt_settlement_payments`, `invoices`, `customers`, `customer_deposits`, `credit_notes`, `chart_of_accounts`, `currencies`

**Columns**: Date, Payment Staff ID, Type No, Document, Invoice No, TCID, Card/Cheq/CrNote/Acc No, Customer No, Settled Amt, Invoice Amt, Overpayment, Deposit, Refund, Settled Amt Local, Exchange Rate

**Row Structure Per Settlement**:
1. Receipt row (main) - shows first invoice + first deposit/credit note/GL
2. Additional invoice rows (2nd, 3rd, etc.)
3. Additional deposit rows
4. Credit note rows
5. Per-settlement total row

**Calculations**:
```
Invoice Amt = receipt_settlement_invoices.amount (settled portion, NOT full invoice total)
Overpayment = settledAmount > invoiceTotal ? (settledAmount - invoiceTotal) : ""
Totals = sum of all row values per settlement
```

**Bug Fixed**: Invoice Amt column was showing `invoice.total_amount` (full total) for additional invoice rows instead of `receipt_settlement_invoices.amount` (settled portion). Fixed March 2026.

**Verified**: March 2026 - All calculations correct after bug fix.

**Documentation**: See `docs/DAILY_SETTLEMENT_REPORT.md` for full technical documentation.

---

### 12. Refund Report
**Purpose**: Lists all printed refund records with service details, customer/supplier info, flight routing, credit note references, currency, and refund amounts

**Key Data Sources**:
- Tables: `refunds`, `services`, `service_types`, `orders`, `customers`, `suppliers`, `costs`, `credit_notes`, `currency_codes`, `service_flights`, `city_codes`, `branches`

**Columns**: Refund Date, Refund No, Status, Refund Type, Airline Form, Number, Pax Name, Debtor No, Customer Name, Booking Order No, Invoice No, Product Code, Original Segment, Refunded Segment, Document No(Customer), Document Date, Currency Code, Exchange Rate, Supplier No, Document No(Supplier), Document Date(Supplier), Voucher No, Supplier Currency, Customer Airline Charges, Customer Refund Charges, Supplier Airline Charges, Supplier Refund Charges, Supplier Amount, Customer Amount

**Key Logic**:
- `status = 'Printed'` filter — only printed refunds
- Refund Type = first character of `service_type.type` (A=Air, H=Hotel, etc.)
- Document No(Customer) = `credit_note.reference` looked up by `client_refrence = refund_no`
- Document No(Supplier) = `refund.cost_document_number` or `cost.supplier_reference`
- Original Segment built from `service_flights` city codes (e.g., KHI-DXB, DXB-KHI)
- Grand Total sums: customer/supplier amounts and charges

**Verified**: March 2026 - All 27 columns verified correct. No bugs found.

**Documentation**: See `docs/REFUND_REPORT.md` for full technical documentation.

---

## Key Calculations and Formulas

### Revenue Calculation Standard
```javascript
// Standard Revenue Formula across all reports
Revenue = Total Sales - Total Costs
Margin% = (Revenue / Total Sales) × 100
```

### Cost Calculation Standard
```javascript
// Standard Cost Calculation
Base Cost = Published Rate - Commission
Commission Amount = (Published Rate × Commission%) / 100
Tax Amount = Sum of all applicable taxes
WHT (Withholding Tax) = (Commission Amount × WHT Rate) / 100
Total Cost = Base Cost + Tax Amount + WHT
```

### Sales Calculation Standard
```javascript
// Standard Sales Calculation
Base Price = Invoice Price
Discount Amount = (Base Price × Discount%) / 100
Rebate Amount = (Base Price × Rebate%) / 100
Transaction Fee = Fixed or percentage based
SST on Transaction = (Transaction Fee × SST%) / 100
Total Sales = Base Price - Discount - Rebate + Transaction Fee + SST
```

### Currency Conversion
```javascript
// Currency Conversion Logic
if (currency !== 'PKR') {
  PKR_Amount = Original_Amount × Exchange_Rate
}
```

---

## Data Flow and Dependencies

### Primary Data Flow
1. **Orders** → Created with customer reference
2. **Services** → Added to orders (Flight, Hotel, Visa, etc.)
3. **Invoices** → Generated for services
4. **Costs** → Recorded against services
5. **Settlements** → Applied to invoices (receipts) or costs (payments)
6. **Journal Entries** → Created for accounting transactions

### Key Dependencies
- All reports depend on proper **company_code** filtering
- Financial reports require accurate **exchange rates**
- Ageing reports depend on **credit terms** configuration
- Profitability reports require both **invoice** and **cost** data

---

## Potential Data Accuracy Issues

### 1. **Calculation Inconsistencies**
- **Issue**: Different reports may calculate similar metrics differently
- **Example**: Revenue calculation in Customer Volume vs Daily Invoice Report
- **Impact**: Discrepancies in financial reporting

### 2. **Currency Conversion Timing**
- **Issue**: Exchange rates may be applied at different points
- **Risk**: Conversion differences between invoice and settlement dates
- **Affected Reports**: All multi-currency reports

### 3. **Tax Calculation Variations**
- **Issue**: Tax calculations vary between service types
- **Specific Concern**: 
  - Air tickets: Complex tax structure with multiple tax types
  - Other services: Simpler tax application
- **Impact**: Potential tax reporting inaccuracies

### 4. **Date Range Filtering**
- **Issue**: Inconsistent date filtering logic
- **Examples**:
  - Some reports use `created_at`
  - Others use `invoice_date` or `document_date`
- **Impact**: Period-based reporting discrepancies

### 5. **Opening Balance Calculations**
- **Issue**: Complex logic for determining opening balances
- **Risk Areas**:
  - Customer Position Report with date ranges
  - Supplier Position Report
- **Impact**: Incorrect balance forward amounts

### 6. **Deleted/Cancelled Records**
- **Issue**: Handling of cancelled invoices/services
- **Observation**: Status filtering not consistent across reports
- **Impact**: Inflated or deflated totals

### 7. **Rounding Differences**
- **Issue**: Floating-point arithmetic without consistent rounding
- **Location**: Throughout calculation chains
- **Impact**: Small discrepancies that compound

---

## Recommendations for Verification

### Priority 1: Critical Verifications

1. **Revenue Calculation Audit**
   - Compare Customer Volume Report totals with Daily Invoice Report
   - Verify: Total Sales, Total Costs, and Revenue calculations match
   - Test with sample data set

2. **Tax Calculation Verification**
   - Manually calculate taxes for sample invoices
   - Compare with report outputs
   - Special focus on Air ticket taxes

3. **Currency Conversion Testing**
   - Test reports with multi-currency transactions
   - Verify exchange rate application consistency
   - Check conversion at invoice vs settlement time

### Priority 2: Data Integrity Checks

1. **Balance Reconciliation**
   - Customer Position Report opening/closing balances
   - Verify against actual transaction history
   - Cross-check with AR Ageing reports

2. **Date Range Consistency**
   - Run same date range across different reports
   - Verify transaction inclusion/exclusion
   - Document which date field each report uses

3. **Status Filtering Validation**
   - Verify cancelled/voided transaction handling
   - Check invoice status filtering (Printed, Settled, etc.)
   - Document business rules for each status

### Priority 3: Calculation Standardization

1. **Create Calculation Library**
   - Standardize revenue calculation
   - Standardize cost calculation
   - Standardize tax calculation

2. **Implement Rounding Rules**
   - Define decimal places for each calculation
   - Implement consistent rounding (e.g., banker's rounding)
   - Apply throughout all reports

3. **Add Calculation Logging**
   - Log intermediate calculation steps
   - Enable calculation audit trail
   - Facilitate debugging and verification

---

## Testing Checklist

### For Each Report:
- [ ] Verify filter functionality
- [ ] Test with empty data sets
- [ ] Test with large data sets
- [ ] Verify PDF generation
- [ ] Verify Excel generation
- [ ] Check calculation accuracy
- [ ] Validate date range filtering
- [ ] Test currency conversion
- [ ] Verify tax calculations
- [ ] Check status filtering
- [ ] Validate user permissions
- [ ] Test branch filtering
- [ ] Verify totals and subtotals
- [ ] Check sorting and grouping
- [ ] Validate opening/closing balances

---

## Technical Implementation Details

### Report Generation Process
1. User selects filters in frontend
2. API call to `/api/report/{reportType}`
3. Backend controller validates permissions
4. Database query with filters applied
5. Data processing and calculations
6. Format generation (PDF/Excel)
7. File upload to storage (MinIO/S3)
8. Return signed URL for download

### Database Transaction Usage
- All reports use database transactions
- Ensures data consistency during report generation
- Rollback on error

### File Storage
- Reports stored in MinIO/S3
- Unique naming: `{PREFIX}{timestamp}.{format}`
- Signed URLs for secure access

---

## Conclusion

The PowerSuite reporting system is comprehensive but requires systematic verification to ensure data accuracy. Key areas of concern include:

1. Calculation consistency across reports
2. Currency conversion timing and rates
3. Tax calculation variations
4. Date range filtering inconsistencies
5. Opening balance determination logic

Implementing the recommended verifications and standardizations will improve report reliability and accuracy.

---

## Appendix

### Report Prefixes
- TPCV - Customer Volume
- TPDI - Daily Invoice
- TPAR - AR Ageing
- TPAP - AP Ageing
- TPCP - Customer Position
- TPSP - Supplier Position
- TPIR - Inventory Report

### Status Codes Used
- Invoice: Draft, Printed, Partially Settled, Settled, Cancelled
- Order: Active, Completed, Cancelled
- Service: Active, Completed, Voided

### Permission Keys
Each report requires specific permission keys for access:
- customer-volume
- daily-invoice
- ar-ageing-analysis-detail
- supplier-position
- (etc.)

---

*Document Generated: [Current Date]*
*Version: 1.0*
*Last Updated: Analysis of PowerSuite Reports System*