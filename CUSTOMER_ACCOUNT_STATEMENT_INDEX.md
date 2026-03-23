# Customer Account Statement Report - Complete Documentation Index

## Overview
This documentation provides a comprehensive analysis of the Customer Account Statement Report implementation in the PowerSuite travel booking system. The report is located in both backend (Express.js) and frontend (React) components.

## Documentation Files

### 1. CUSTOMER_ACCOUNT_STATEMENT_ANALYSIS.md (Comprehensive - 757 lines)
**The primary detailed reference document**

Contains:
- Executive Summary
- Report Purpose & Overview
- Input Parameters & Filters (all 13 filter options)
- Database Tables & Models (19 models explained with fields)
- Key Calculations & Business Logic (with formulas and pseudocode)
- Data Points Displayed in Report (all 7 sections detailed)
- Report Output Formats (PDF and Excel structure)
- Backend Processing Flow (5-step data flow)
- Related Service Files (customer_balance_calculator.js)
- Frontend Component Integration
- Special Handling & Edge Cases (8 different scenarios)
- Performance Considerations
- Database Query Overview (6 main queries)
- API Endpoint Documentation (POST /report/getCustomerAccountStatementReport)
- Summary Table of All Database Models (23 entries)
- Code Files Reference
- Key Insights & Design Patterns

**Use this document for**: Complete understanding of the implementation, database schema analysis, formula verification, and feature documentation.

### 2. CUSTOMER_ACCOUNT_STATEMENT_QUICK_REFERENCE.md (Quick Lookup - 161 lines)
**Fast reference for developers**

Contains:
- Core Files (with line numbers)
- API Endpoint Details
- Input Filters Summary (3 filter types)
- Key Calculations (formulas in compact format)
- Key Database Models (grouped by category)
- Report Sections (7 sections listed)
- Status Filters
- Special Handling
- Output Structure (PDF/Excel comparison)
- Data Flow Summary
- Performance Notes
- Company Isolation
- File Size Statistics

**Use this document for**: Quick lookups, API integration, field references, and before diving into the full analysis.

### 3. CUSTOMER_ACCOUNT_STATEMENT_DIAGRAMS.md (Visual Architecture - 565 lines)
**ASCII diagrams and architectural visualizations**

Contains:
1. Data Flow Diagram - Complete request/response flow
2. Database Model Relationships - ER-style diagram
3. Invoice Calculation Flow - Step-by-step calculation with decision trees
4. Balance Calculation Architecture - Opening balance formula flow
5. Report Structure Hierarchy - Complete report layout visualization
6. Database Query Sequence - All 7 query steps documented
7. File Generation Architecture - PDF vs Excel generation paths
8. Currency Conversion Points - Where conversions happen
9. Error Handling Flow - Exception handling paths
10. Performance Optimization Points - Optimization table

**Use this document for**: Understanding data flow, architectural decisions, calculation processes, and teaching others about the system.

## Key Information by Topic

### For Implementation / Development
**Start with**: CUSTOMER_ACCOUNT_STATEMENT_QUICK_REFERENCE.md
**Then read**: CUSTOMER_ACCOUNT_STATEMENT_ANALYSIS.md (Sections 2, 7, 15)

Key files to understand:
- `/psback/controllers/report.controller.js` (lines 9298-10605)
- `/psback/services/customer_balance_calculator.js`
- `/psfront/src/pages/Report/CustomerAccountStatement.jsx`

### For Database Queries
**Start with**: CUSTOMER_ACCOUNT_STATEMENT_ANALYSIS.md (Section 3 & 12)
**Reference**: CUSTOMER_ACCOUNT_STATEMENT_DIAGRAMS.md (Section 6)

19 database models used:
- Primary: customer, order, service, invoice
- Financial: receipt_settlement, customer_deposit, credit_note, refund
- Support: service_flight, service_hotel, currency_code, etc.

### For Formula/Calculation Logic
**Start with**: CUSTOMER_ACCOUNT_STATEMENT_DIAGRAMS.md (Sections 3 & 4)
**Details in**: CUSTOMER_ACCOUNT_STATEMENT_ANALYSIS.md (Section 4)

Main formulas:
- Invoice Amount = (subtotal × rooms) + fee + SST
- Opening Balance = Invoices - Receipts - Refunds - Deposits - Credit Notes
- Net Balance = Opening + Period Invoices - Period Refunds - Receipts - Deposits

### For Report Output
**Start with**: CUSTOMER_ACCOUNT_STATEMENT_QUICK_REFERENCE.md (Output Structure)
**Details in**: CUSTOMER_ACCOUNT_STATEMENT_ANALYSIS.md (Section 5 & 6)

Report includes 7 main sections:
1. Ticket Bookings (flights)
2. Hotel Bookings
3. General Bookings
4. Refund Tickets
5. Refund General
6. Receipts (with payment details)
7. Deposits (prepayments)
Plus Summary with 6 balance figures

### For API Integration
**Quick ref**: CUSTOMER_ACCOUNT_STATEMENT_QUICK_REFERENCE.md (API Endpoint)
**Full spec**: CUSTOMER_ACCOUNT_STATEMENT_ANALYSIS.md (Section 13)

Endpoint: POST /report/getCustomerAccountStatementReport
- Timeout: 5 minutes
- Auth: JWT + "customer-account-statement" permission
- Formats: PDF or Excel
- 8 input parameters

### For Performance Tuning
**Reference**: CUSTOMER_ACCOUNT_STATEMENT_QUICK_REFERENCE.md (Performance Notes)
**Details**: CUSTOMER_ACCOUNT_STATEMENT_ANALYSIS.md (Section 11)
**Visualization**: CUSTOMER_ACCOUNT_STATEMENT_DIAGRAMS.md (Section 10)

Optimizations:
- 7 separate queries instead of complex JOINs
- Set-based deduplication of invoices
- Currency conversion at query level
- Transaction management
- Execution time: 100-500ms typical

### For Multi-Currency Support
**Where it's used**: Invoices, Refunds, Deposits, Payment methods
**Implementation**: All amounts converted to PKR using exchange_rate
**Fallback**: 1.0 if exchange rate not found

### For Multi-Tenancy
**Isolation**: Company code on user model
**Implementation**: `user.company_code = req.user.company_code` filter
**Scope**: All customer queries filtered by company

## Quick Navigation

### By Feature
- **Customer Filtering**: ANALYSIS Section 2, DIAGRAMS Section 1
- **Date Filtering**: ANALYSIS Section 2, DIAGRAMS Section 10
- **Balance Calculations**: ANALYSIS Section 4, DIAGRAMS Sections 3-4
- **Service Type Handling**: ANALYSIS Section 10.1, DIAGRAMS Section 3
- **Currency Conversion**: ANALYSIS Section 4.4, DIAGRAMS Section 8
- **Excel Generation**: ANALYSIS Section 6.2, DIAGRAMS Section 7
- **PDF Generation**: ANALYSIS Section 6.1, DIAGRAMS Section 7
- **Receipt Settlements**: ANALYSIS Section 5, DIAGRAMS Section 6
- **Deposit Tracking**: ANALYSIS Section 10.5, QUICK_REF Special Handling

### By File Location
- Backend Controller: ANALYSIS Section 15 (line 9298)
- Backend Service: ANALYSIS Section 8
- Frontend Component: ANALYSIS Section 9
- Database Models: ANALYSIS Section 3 & 14
- API Wrapper: ANALYSIS Section 9
- EJS Template: ANALYSIS Section 6.1
- Routes: ANALYSIS Section 13

### By Database Table
- Customer (19 models total): ANALYSIS Section 3
- Orders & Services: ANALYSIS Section 3
- Invoices & Taxes: ANALYSIS Section 3
- Receipts & Settlements: ANALYSIS Section 3
- Deposits & Usage: ANALYSIS Section 3
- Credit Notes: ANALYSIS Section 3
- Currency & GL: ANALYSIS Section 3

## Statistics

### Code Coverage
- Controller: 1,308 lines (includes other reports)
- Specific function: 307 lines (getCustomerAccountStatementReport)
- Service: 240 lines
- Frontend: 318 lines
- Template: 715 lines
- Total documentation: 1,483 lines (across 3 files)

### Database Complexity
- Models accessed: 19
- Separate queries: 7 (plus historical if date range)
- Status filters: Multiple ("Printed", "Settled", "Void" exclusions)
- Currency fields: 4 different points

### Report Sections
- Sales invoices: 3 types (Ticket, Hotel, General)
- Refunds: 2 types (Ticket, General)
- Payments: 1 type with settlement details
- Deposits: 1 type with remaining balance
- Summary: 6 financial figures

## Common Questions Answered

**Q: Where are invoices calculated?**
A: ANALYSIS Section 4.1, DIAGRAMS Section 3, formula with taxes, discounts, rebates, fees, and currency conversion.

**Q: How are opening balances calculated?**
A: ANALYSIS Section 4.1, DIAGRAMS Section 4, formula with historical data before startDate.

**Q: What's the transaction flow?**
A: DIAGRAMS Section 1, complete data flow from frontend to file upload.

**Q: How many queries are executed?**
A: 7 main queries plus historical queries if date range provided. ANALYSIS Section 12, DIAGRAMS Section 6.

**Q: How are hotels handled differently?**
A: ANALYSIS Section 10.1, subtotal multiplied by number_of_rooms.

**Q: How is currency handled?**
A: ANALYSIS Section 4.4, DIAGRAMS Section 8, all amounts auto-converted to PKR using exchange_rate.

**Q: What formats are supported?**
A: PDF (via EJS + wkhtmltopdf) and Excel (via ExcelJS). ANALYSIS Section 6.

**Q: What are the input filters?**
A: 4 customer filters + 8 date filters + output format. ANALYSIS Section 2, QUICK_REF.

**Q: How is performance optimized?**
A: ANALYSIS Section 11, DIAGRAMS Section 10, separate queries, Set deduplication, index strategy.

**Q: How is data security maintained?**
A: ANALYSIS Section 1, JWT auth + permission check + company_code isolation.

## Version & Last Updated
- Created: November 7, 2025
- Analysis Depth: Medium (comprehensive)
- Code Snapshot: As of latest controller implementation
- Documentation: 1,483 lines across 3 files + this index

## How to Use These Documents

1. **First time learning**: Read QUICK_REFERENCE.md for overview, then ANALYSIS.md for details
2. **Specific questions**: Use the Quick Navigation section above
3. **Understanding architecture**: Start with DIAGRAMS.md
4. **Implementation**: Refer to ANALYSIS.md Sections 15 for file locations
5. **API integration**: QUICK_REF for endpoint, ANALYSIS Section 13 for full spec
6. **Troubleshooting**: ANALYSIS Section 10-11 for edge cases and performance

---

**All documents saved in**: `/mnt/c/Codes/Powersuite/`

Files:
- CUSTOMER_ACCOUNT_STATEMENT_ANALYSIS.md
- CUSTOMER_ACCOUNT_STATEMENT_QUICK_REFERENCE.md
- CUSTOMER_ACCOUNT_STATEMENT_DIAGRAMS.md
- CUSTOMER_ACCOUNT_STATEMENT_INDEX.md (this file)
