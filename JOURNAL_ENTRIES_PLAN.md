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
- ⚠️ Regenerate functionality for existing batches (UI button hidden — underlying regen logic produces incorrect entries; backend endpoint still exists but not user-accessible until fixed)
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
7. **Void entries use void date, not original document date** - All void posting-rules functions set `transaction_date = <doc>.updated_at` so reversal entries land in the period the void actually happened. Each respective void controller explicitly writes `updated_at = NOW()` when voiding, making this reliable as long as the voided document is not subsequently edited (which is the normal case).

   **Date filter on the `from`/`to` range** uses `created_at OR updated_at` (Sequelize `Op.or`) across all 7 void posting-rules functions that accept the range (`voidCreditNotePostingRules`, `voidDebitNotePostingRules`, `voidDepositPostingRules`, `voidSupplierDepositPostingRules`, `voidSettlementPostingRules`, `voidCreditNotePaymentPostingRules`, `voidInternalTransferPostingRules`). Reason: a document voided today but created weeks ago must still be picked up when the user runs "Generate System JE" with today's date — filtering on `created_at` alone misses these. Duplicate-protection still holds because each function checks `journal_entry.analysis_code1 LIKE '<doc> (Void)'` before posting. Applied to **all 11 void paths**:
   - `voidInvoicePostingRules` → `invoice.updated_at`
   - `voidCostingPostingRules` (XO) → `costing.updated_at`
   - `voidCreditNotePostingRules` → `creditNote.updated_at`
   - `voidDebitNotePostingRules` → `debitNote.updated_at`
   - `voidDepositPostingRules` → `deposit.updated_at`
   - `voidSupplierDepositPostingRules` → `deposit.updated_at`
   - `voidSettlementPostingRules` (5 entry types) → `settlement.updated_at`
   - `voidCreditNotePaymentPostingRules` → `payment.updated_at`
   - `voidExpensePaymentPostingRules` → `payment.updated_at`
   - `voidSupplierPaymentPostingRules` (5 entry types) → `payment.updated_at`
   - `voidInternalTransferPostingRules` (ACTO + ACFR) → `payment.updated_at`

8. **Manual JE voiding creates a NEW reversal batch with the next sequential number** - When a Manual JE batch is voided via `voidBatch` (`journal_entry.controller.js`), the controller:
   - Accepts an optional `void_date` (YYYY-MM-DD) from the request body. Validates it's not before the original batch's `dated` value AND not in the future. Anchors the parsed date at noon (`<date>T12:00:00`) to avoid timezone-shift edge cases when computing the period. Falls back to `new Date()` if not provided.
   - Marks the original batch `status = 'Void'` (entries untouched — audit trail preserved).
   - Generates the next sequential `JV` batch number across the company (same logic as `createJournalEntryBatch`).
   - Creates a NEW Manual JE batch with that number: **`status = 'Void'`** (both original and reversal share the same status — they represent one cancelled JE event together), `dated = <voidDate>`, `journal_entry_period = MMYYYY of voidDate`, narrative `"Reversal of voided batch <original_batch_no>"`.
   - Inserts reversal rows into the NEW batch — same accounts, analysis codes, and gl_entity_id; **debit/credit swapped**; `transaction_date = <voidDate>`; description prefixed `VOID REVERSAL - <original_description>`.
   - All steps run inside a single DB transaction so partial failures roll back cleanly.
   - GL trial balance shows both batches netting to zero.
   - API response includes both `batch_no` (original) and `reversal_batch_no` (new) so the UI can surface the new number to the user via toast.

   **Filtering convention**: with both batches now `Void`, the status filter alone (`status != 'Void'`) is sufficient to keep voided JEs out of reports. The `description LIKE 'VOID REVERSAL -%'` filter is still applied across the codebase as a defensive belt-and-braces measure (and as a human-readable audit marker), but it is functionally redundant after this change.

   **Frontend (`JournalEntryBatches.jsx`):** clicking the red `Void` button opens an AlertDialog with a `DateInput` field. The picker enforces an allowed range of `[original batch's dated, today]` — `minDate` is the original JE's `dated` value (no back-dating before the JE existed); `maxDate="today"` (no future-dating). Selected date is sent as `void_date` in the API request body. The dialog button shows a spinning `Loader2` icon while the request is in flight; `e.preventDefault()` on the AlertDialogAction stops Radix from auto-closing the dialog so the spinner remains visible until the call completes.

11. **Editing a Manual JE batch can renumber it when the Document Type (branch) changes** (`updateJournalEntryBatch`):
    - When `branch_id` in the update payload differs from the current batch's branch, the controller treats this as a "move to another branch" operation:
      - Blocked if the batch is `Void` (voided records should never mutate).
      - Blocked if any reversal batch references this batch via narrative `"Reversal of voided batch <old_batch_no>"` — the reversal's narrative would otherwise become stale.
      - Otherwise generates the next sequential `JV` batch number for the new branch's company (same logic as `createJournalEntryBatch` / `voidBatch`) and updates both `batch_no` and `branch_id` atomically.
    - Response now returns `{ id, batch_no, renumbered, previous_batch_no }` so the UI can react to a renumber.
    - **Frontend (`AddJournalEntryBatchImproved.jsx`)**: on save, if the response indicates a renumber, the page navigates (`replace: true`) to the new URL `/gl/JournalEntries/batch/<new_batch_no>` and shows a toast `"Batch renumbered: <previous> → <new>"`. The journal_entries themselves do not move (they reference batch by FK `journal_batch_id`, not by `batch_no`), so audit trail is preserved.

12. **Manual JE settlement updates invoice / cost status (Level 2 integration)**
    Shared service `psback/services/manualJeAdjustment.js` exposes:
    - `sumManualJeAdjustment(ref, t)` — sums live (`status != 'Void'` AND description `notLike 'VOID REVERSAL -%'`) Manual JE row debit+credit for a doc reference.
    - `sumManualJeAdjustmentByCostId(costId, t)` — same but resolves doc number(s) for a cost id via `documents`.
    - `recalculateInvoiceStatusByNumber(invoiceNumber, t)` — applies receipt-settlements + Manual JE adjustments and updates `invoice.status` (`Settled` / `Partially Settled` / `Printed`); never overrides `Void`/`Raised`; locks rows.
    - `recalculateCostStatusByDocNumber(documentNumber, t)` — same idea for costs (`Paid` / `Partially Paid` / `Printed`); reuses `payment.controller.js`'s cost-total formula.
    - `recalculateStatusesForRefs(refs, t)` — fan-out helper; ignores non-IN/XO refs silently.

    Wired into:
    - `journal_entry.controller.js` → `createJournalEntries`, `updateJournalEntries`, `voidBatch`, `deleteBatch`. `updateJournalEntries` recalcs both pre-update and post-update refs so unlinking/deleting reverts status.
    - `invoice.controller.js` → receipt-settlement create AND void paths now add `sumManualJeAdjustment(invoice.invoice_number, t)` to the paid total.
    - `payment.controller.js` → supplier-payment create AND void paths now add `sumManualJeAdjustmentByCostId(cost.id, t)` to the paid total.

    Result: any ordering of receipt/payment-settlement and Manual JE actions produces consistent `invoice.status` / `cost.status`. Reports, listings, and the Manual JE document picker (which filters by status) stay accurate without further changes. No new DB columns — the "balance/remaining" is always derived live from `total - settlements - manualJeAdjustments` per standard accounting practice.

13. **Customer/Supplier-driven Document picker in Manual JE editor** (`JournalEntriesImproved.jsx`)
    - Backend: `GET /journalEntry/related-documents?customer_id=X` returns invoices via `documents → service → order.customer_id`; `?supplier_id=Y` returns costing/XO documents via `documents → service.supplier_id`. Both queries:
      - Apply **company isolation**: `order.branch_id` must belong to the requesting user's company (`req.user.company_code`).
      - Apply **status filter**: skips closed/cancelled docs (Invoices: exclude `Void`, `Raised`, `Settled`; Costs: exclude `Void`, `Raised`, `Paid`).
      - Return deduped `{ document_number, type, amount }`.
    - Frontend: when a row's Customer/Supplier is selected, the Document column switches from a free-form text input to a `SearchableSelect` whose options are that entity's invoices (customer) or XOs (supplier). Options are cached per `${entityType}-${entityId}` key. The dropdown's loading vs empty state is distinguished by checking `Object.prototype.hasOwnProperty.call(documentOptionsByEntity, key)` so an entity with zero docs shows "No invoices/XOs found" instead of staying on "Loading...".
    - A new "Amount" column appears right of Document, looking up the linked doc's amount from the cached options for instant display.

14. **Manual JE settlement reflects on the printed invoice / cost document footer**
    - `document.controller.js` (invoice and cost render paths) sums `sumManualJeAdjustment(...)` for the document's references.
    - `invoiceDocument.ejs` adds a "Received by JE" line between Received and Balance (only when `manualJEAdjustmentAmount > 0`); Balance = Grand Total − Received − Received by JE.
    - `costDocument.ejs` adds "Paid by JE" + "Balance" rows under Grand Total.
    - Multi-currency: PKR amounts converted to invoice/cost currency using `exchangeRateInfo.rate`.

**Customer/Supplier-driven Document picker in Manual JE editor** (`JournalEntriesImproved.jsx`)
- Backend: `GET /journalEntry/related-documents?customer_id=X` returns invoices via `documents → service → order.customer_id`; `?supplier_id=Y` returns costing/XO documents via `documents → service.supplier_id`. Both queries:
  - Apply **company isolation**: `order.branch_id` must belong to the requesting user's company (`req.user.company_code`).
  - Apply **status filter**: skips closed/cancelled docs.
    - Invoices: exclude `Void`, `Raised`, `Settled` (keep `Printed`, `Partially Settled`)
    - Costs: exclude `Void`, `Raised`, `Paid` (keep `Printed`, `Partially Paid`)
  - Return deduped `{ document_number, type }`.
- Frontend: when a row's Customer/Supplier is selected, the Document column switches from a free-form text input to a `SearchableSelect` whose options are that entity's invoices (customer) or XOs (supplier). Options are cached per `${entityType}-${entityId}` key so the same entity isn't fetched twice across rows. Clearing or changing the entity also clears the previously linked `analysis_code1`. If no entity is selected, the column falls back to the original free-form text input.

9. **Display behavior for voided/reversal Manual JE batches** (frontend `JournalEntriesImproved.jsx`):
   - The view-mode table only hides rows when the batch contains any row whose description starts with `VOID REVERSAL -`. With the current "new batch" model:
     - Original voided batch contains only originals → all shown normally.
     - Reversal batch contains only `VOID REVERSAL -` rows → all shown.
   - The `VOID REVERSAL` token in the description is rendered in red (`text-red-600 font-semibold`) so the void state is visually obvious.
   - EditMode keeps the full list to avoid index mismatches in edit handlers.

10. **Listing-page indicators for voided + reversal batches** (frontend `JournalEntryBatches.jsx`):
    - A `reversalByOriginal` map is built from the batch list — for every batch whose narrative starts with `Reversal of voided batch <original>`, it maps `<original> → <reversal_batch_no>` so the UI can surface the link in both directions.
    - **Voided original (Manual JE, `status === 'Void'`, narrative does NOT start with `Reversal of voided batch `)**:
      - Actions cell: empty (no pill, no button — the Voided pill lives on the reversal batch instead).
      - Status indicator: visual override displays it in the **Posted** color so it stays distinguishable from the reversal batch (DB status is `Void`, but on the listing the original visually retains its identity as "the original posted record").
      - Batch No cell: blue link with a red `Ban` icon and a red hover tooltip: `This JE is already voided — Reversed in batch: <reversal_batch_no>` (or `Reversal batch not found in current view` if the reversal isn't in the loaded list).
    - **Reversal/void JE (narrative starts with `Reversal of voided batch <original>`)**:
      - Actions cell: red `Voided` pill (this is the JE that represents the void operation).
      - Status indicator: shows the **Void** color naturally — DB status IS `Void`, no override needed.
      - Batch No cell: blue link with an amber `AlertCircle` icon and an amber hover tooltip: `Reversal batch — This JE is the reversal of voided batch: <original_batch_no>`.
    - **Unbalanced batches** (any type): red link with red `AlertCircle` icon and red tooltip showing debit/credit/difference and offending docs.
    - Other Manual JE batches: standard `Void` button + simple blue Batch No link.

    **Status color palette** (`statusColorCoding()` in `JournalEntryBatches.jsx`):
    - `Open` → purple
    - `Drafted` → green
    - `Posted` → blue
    - `Void` → red (changed from black)
    All status overrides go through this function so future palette tweaks stay in one place.

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

### 4. Regenerate Existing Batches (TEMPORARILY DISABLED)
- The "Regenerate" UI button and batch-selection checkboxes are currently hidden
- Reason: regeneration was producing incorrect entries (e.g., duplicate posting types, missing service-type filter on debit notes)
- Workaround: null `je_generated` on the source documents and delete the batch from the UI, then run "Generate System JE" fresh
- The backend `regenerateBatches` endpoint still exists for when the UI is restored after the underlying logic is fixed

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

### 6. Credit-Note JE unbalanced by exactly the supplementary fee
- **Symptom**: A System-JE batch for a customer Credit Note (CN) is unbalanced; the difference equals `refund.calculator_overrides.customer.supplementaryFee`.
- **Root cause**: `RFAREC` posts the full `refund.customer_refund_amount` (which already includes the supplementary fee), but the supplementary-fee component had no offsetting JE row because no entry-type code existed for it.
- **Fix (applied)**: Added entry type **`RFSUP`** (`Refund - Supplementary Fee`). The CN posting paths in `psback/services/journal.js` now post `calcOverrides.customer.supplementaryFee` via RFSUP. Updated in all 5 paths: `voidCreditNotePostingRules`, `regenerateCreditNoteEntries`, `creditNotePostingRules`, `refundPostingRules`, `regenerateRefundEntries`.
- **Required setup per branch**: Add a Posting Rule mapping `RFSUP` → an income/admin-fee Chart of Account (e.g. `410135 - Supplementary Fee`) on the **debit** side, prefix `<branch_doc_prefix>CN`. Without this rule the imbalance persists for any CN whose refund has a supplementary fee.
- **Existing unbalanced batches**: Need to be regenerated (null `je_generated` on the source CN, delete the batch, re-run "Generate System JE") after the posting rule is in place.

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