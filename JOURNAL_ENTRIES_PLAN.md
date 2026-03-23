# PowerSuite Journal Entries Implementation

## Overview
Implementation of automated journal entries generation based on financial consultant's requirements (Updated January 2025). The new structure provides cleaner P&L reporting with proper revenue recognition and cost matching. Journal entry batches are now created period-wise, with separate batches for each accounting period.

## Current Status ✅

### Completed Implementation
- ✅ Complete journal entry infrastructure (models, tables, service)
- ✅ Chart of accounts with all required accounts
- ✅ Posting rules system with configurable debit/credit mappings
- ✅ Invoice posting function with all entry types
- ✅ Cost document processing (Exchange Orders)
- ✅ Deposit and settlement processing
- ✅ Manual trigger via "Generate System JE" function
- ✅ Regenerate functionality for existing batches
- ✅ je_generated tracking to prevent duplicates
- ✅ All journal entry types (AREC, ASLE, ATAX, PSFM, PSFT, STAX, CSAL, CSTX, IATA, DISC, RBTE)
- ✅ Updated journal.js with correct calculations for all scenarios
- ✅ Period-wise batch creation - separate journal batches for each accounting period
- ✅ Journal Period management system for defining fiscal periods

## New Journal Entry Structure (January 2025)

### Key Changes from Financial Consultant
1. **Revenue Separation**: Air sales and taxes are now separate revenue accounts
2. **Cost Matching**: Direct cost of sales entries match revenue entries
3. **Cleaner Structure**: Simplified discount/rebate handling
4. **Better Reporting**: P&L statements now show proper gross margins

### Journal Entry Types

#### Invoice Entries (INV)
- **AREC**: Account Receivable - Uses invoice.total_amount directly (Debit)
- **ASLE**: Air Sales Revenue - Base cost price (cost.published_rate × quantity) (Credit)
- **ATAX**: Air Sales Tax Revenue - Airline taxes (invoice_taxes sum × quantity) (Credit)
- **PSFM**: PSF Markup Revenue - Markup amount (invoice.markup × quantity) (Credit)
- **PSFT**: PSF Transaction Fee Revenue - Transaction fee amount (Credit)
- **STAX**: Sales Tax - SST on transaction fee (sst% × transaction_fee) (Credit)
- **CSAL**: Cost of Sales - Direct ticket cost (cost.published_rate × quantity) (Debit)
- **CSTX**: Cost of Sales Tax - Tax costs (cost_taxes sum × quantity) (Debit)
- **IATA**: IATA-BSP Clearing - BSP payable ((published_rate - commission + taxes) × quantity) (Credit)
- **DISC**: Discount - When discount exists (discount% × invoice.price × quantity) (Debit)
- **RBTE**: Rebate - When rebate exists (rebate% × invoice.price × quantity) (Debit)

#### Cost Entries (XO)
- **IATA**: IATA-BSP Clearing (Debit)
- **ATAX**: Advance Tax/WHT on commission (Debit)
- **APAY**: Account Payable (Credit)

#### Deposit Entries (DP)
- **CASH**: Cash/Bank Receipt (Debit)
- **DEPO**: Customer Deposit Liability (Credit)

#### Receipt Settlement (RS)
- **SETL**: Reverse Deposit Liability (Debit)
- **AREC**: Clear Invoice Receivable (Credit)

#### Payment Settlement (PS)
- **APAY**: Clear Supplier Payable (Debit)
- **CASH**: Cash/Bank Payment (Credit)

## Implementation Details

### Database Schema

#### Journal Period Table
The `journal_period` table defines accounting periods for the system:
```sql
- id: Primary key
- company_code: Link to company
- fiscal_year: e.g., "2024-2025"
- month: Period month (1-12)
- year: Calendar year
- status: Active/Inactive
- start_date: Period start date
- end_date: Period end date
- report_date: Reporting cutoff date
```

#### Journal Batch Table
The `journal_batch` table now includes period information:
```sql
- journal_entry_period: Period identifier (e.g., "2024-2025-01")
```

#### je_generated Tracking
All document tables have `je_generated` column:
```sql
- invoices.je_generated
- costs.je_generated
- customer_deposits.je_generated
- receipt_settlements.je_generated
- payment_settlements.je_generated
```

### Key Calculation Logic

Located in `/psback/services/journal.js`:

#### Period-wise Batch Creation Process:
1. **Entry Collection**: All journal entries are collected first without creating batches
2. **Period Grouping**: Entries are grouped by their transaction dates into respective periods
3. **Batch Creation**: Separate batches are created for each period with unique batch numbers
4. **Period Validation**: System validates that transaction dates fall within active period ranges

#### Important Notes:
1. **invoice.price already includes markup** - Don't add markup again
2. **Associations are lowercase**: Use `invoice_taxes` not `Invoice_taxes`
3. **AREC uses invoice.total_amount directly** - Ensures balance with actual invoice
4. **DISC/RBTE calculate on invoice.price only** - Not price + markup
5. **Period Assignment**: Entries are assigned to periods based on their transaction_date
6. **Batch Narrative**: Each batch includes the period identifier in its narrative

```javascript
// Key calculations
case 'AREC': 
    // Use invoice total directly to ensure balance
    amount = Number(invoice?.total_amount || 0);
    break;

case 'ASLE': 
    // Base revenue matches cost for margin calculation
    amount = Number(cost?.published_rate || 0) * Number(cost?.quantity || 1);
    break;

case 'ATAX': 
    // Airline taxes as revenue
    amount = (invoice?.invoice_taxes?.reduce((sum, tax) => 
        sum + Number(tax.tax_amount || 0), 0) || 0) * Number(invoice?.quantity || 1);
    break;

case 'DISC':
    // Discount on invoice price (which includes markup)
    const discountPercent = Number(invoice?.discount || 0) / 100;
    const invoicePrice = Number(invoice?.price || 0) * Number(invoice?.quantity || 1);
    amount = discountPercent * invoicePrice;
    break;

case 'RBTE':
    // Rebate on invoice price (which includes markup)
    const rebatePercent = Number(invoice?.rebate || 0) / 100;
    const invoicePriceRebate = Number(invoice?.price || 0) * Number(invoice?.quantity || 1);
    amount = rebatePercent * invoicePriceRebate;
    break;
```

### Posting Rules Configuration

#### Rule Precedence
- "Additional" type rules override "Default" rules
- Rules are filtered by service_type_id when applicable
- Branch-specific rules take precedence

#### Important Note on ATAX
- **In Invoices**: ATAX = Air Sales Tax Revenue (from invoice_taxes table)
- **In Costs**: ATAX = Advance Tax/WHT (calculated on commission)
- Same code, different purpose based on document type

## Usage Workflow

### 1. Set Up Journal Periods
- Navigate to GL > Journal Periods
- Generate periods for fiscal year (e.g., July to June)
- Set period start/end dates
- Ensure periods are active (status = 1)

### 2. Configure Posting Rules
- Navigate to GL > Posting Rules Maintenance
- Select branch and document type
- Add all required rules with correct prefix (e.g., 'TTIN' not 'TTINV')
- Ensure proper chart of accounts are selected

### 3. Generate Journal Entries
- Navigate to GL > Journal Entries
- Click "Generate System JE"
- Select branch and date range
- System processes all eligible documents
- **Automatic Period Assignment**: Entries are grouped into periods based on transaction dates
- **Multiple Batches**: One batch created per period within the selected date range

### 4. Regenerate Existing Batches
- Select an existing journal batch
- Click "Regenerate" button
- System will recalculate all entries with current rules
- Period assignment is preserved during regeneration

### 5. Verification
- Check journal batch for balanced entries
- Verify debit = credit totals
- Review individual entries for accuracy
- Confirm period assignment is correct in batch narrative

## Example Calculation

### Invoice Example with Discounts
```
Invoice Details:
- Invoice Price: 67,000 × 2 = 134,000 (includes 3,000 markup per ticket)
- Cost Price: 64,000 × 2 = 128,000
- Airline Taxes: 10,140 × 2 = 20,280
- Discount: 2% = 2,680 (on invoice price)
- Rebate: 1% = 1,340 (on invoice price)
- Transaction Fee: 5,000
- SST (5% on fee): 250
- Total Invoice: 155,510

Journal Entries:
DR AREC (Trade Debtors): 155,510
DR CSAL (Cost of Sales): 128,000
DR CSTX (Cost of Sales Tax): 20,280
DR DISC (Discount): 2,680
DR RBTE (Rebate): 1,340

CR STAX (Tax Payable): 250
CR IATA (BSP Clearing): 148,280
CR ASLE (Air Sales Revenue): 128,000
CR ATAX (Air Sales Tax Revenue): 20,280
CR PSFM (PSF Markup Revenue): 6,000
CR PSFT (Transaction Fee Revenue): 5,000

Total Debits: 307,810
Total Credits: 307,810 ✓ Balanced
```

## Implementation Checklist

- [x] Add all journal entry types to database
- [x] Add complete chart of accounts
- [x] Update journal.js with correct calculations
- [x] Fix association names (invoice_taxes, cost_taxes)
- [x] Implement AREC to use invoice.total_amount
- [x] Fix DISC/RBTE to calculate on invoice.price only
- [x] Test with sample invoices including discounts
- [x] Verify balanced journal entries
- [x] Implement regenerate functionality

## Common Issues and Solutions

### 1. Unbalanced Entries
- Check all posting rules are configured with correct prefix (e.g., 'TTIN' not 'TTINV')
- Ensure DISC/RBTE posting rules exist when invoices have discounts
- Verify invoice.price understanding (already includes markup)

### 2. Missing Account Numbers in UI
- Accounts exist in database but may not display
- Check chart_of_account_id is properly set in posting rules
- Verify accounts are active (status = 1)

### 3. Wrong Amounts
- AREC should use invoice.total_amount directly
- DISC/RBTE calculate on invoice.price, not price + markup
- Association names are lowercase (invoice_taxes, not Invoice_taxes)

### 4. Regenerate Not Working
- Ensure backend is restarted after code changes
- Both generation and regenerate functions must be updated

### 5. Period-Related Issues
- **No Periods Found**: Ensure journal periods are created and active for the company
- **Entries Not Assigned**: Check transaction dates fall within period date ranges
- **Missing Entries**: Verify documents have valid transaction_date values
- **Wrong Period**: Confirm period start/end dates are correctly configured

## Support Queries

```sql
-- Check unprocessed invoices
SELECT COUNT(*) FROM invoices
WHERE je_generated IS NULL
AND status NOT IN ('Raised', 'Void');

-- Check active journal periods
SELECT * FROM journal_periods
WHERE company_code = ?
AND status = 1
ORDER BY start_date;

-- Verify posting rules with correct prefix
SELECT pr.*, jet.code, coa.key_account
FROM posting_rules pr
JOIN journal_entry_types jet ON pr.journal_entry_type_id = jet.id
LEFT JOIN chart_of_accounts coa ON pr.chart_of_account_id = coa.id
WHERE pr.branch_id = ?
AND pr.prefix_code = 'TTIN'; -- Use correct prefix

-- Check journal batches by period
SELECT
    journal_entry_period,
    COUNT(*) as batch_count,
    SUM(CASE WHEN status = 'Posted' THEN 1 ELSE 0 END) as posted_count
FROM journal_batches
WHERE branch_id = ?
GROUP BY journal_entry_period
ORDER BY journal_entry_period;

-- Check journal batch balance
SELECT
    batch_no,
    journal_entry_period,
    SUM(debit) as total_debit,
    SUM(credit) as total_credit,
    SUM(debit) - SUM(credit) as imbalance
FROM journal_entries je
JOIN journal_batches jb ON je.journal_batch_id = jb.id
WHERE jb.batch_no = ?
GROUP BY jb.id, jb.batch_no, jb.journal_entry_period;
```