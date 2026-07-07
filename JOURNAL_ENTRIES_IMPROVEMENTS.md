# Journal Entries System Improvements

## Overview
The journal entries system has been significantly improved to provide a better user experience with enhanced visual feedback, validation, and intuitive controls.

## Key Improvements

### 1. Visual Design Enhancements
- **Modern Card Layout**: Clean, professional card design with gradient header
- **Color-Coded Balance Status**: Green for balanced, red for unbalanced entries
- **Unbalanced Batch Indicator (Listing)**: On the Journal Entries listing, batch numbers turn red with an `AlertCircle` icon when total Debit ≠ total Credit. Hovering shows a tooltip with debit, credit, the exact difference, AND the offending document number(s) (entries grouped by `analysis_code1` whose per-doc debit ≠ credit). Up to 5 docs are listed; remaining counts shown as "+N more". Voided batches and empty batches are excluded from the check.
- **Status Badges**: Visual indicators for batch type and edit mode
- **Hover Effects**: Improved interactivity with hover states on rows
- **Professional Typography**: Better font hierarchy and spacing

### 2. User Feedback & Loading States
- **Loading Indicators**: Animated spinner while data loads
- **Save Progress Feedback**: Clear indication when entries are being saved
- **Toast Notifications**: Success/error messages with descriptive icons
- **Real-time Validation**: Immediate feedback on entry errors
- **Balance Status Alert**: Prominent display of debit/credit totals and difference

### 3. Improved Data Entry
- **Auto-Balance Feature**: One-click button to automatically balance entries
- **Duplicate Entry**: Copy button to quickly duplicate similar entries
- **Delete Entry**: Easy removal of unwanted entries with tracking
- **Smart Field Behavior**:
  - Entering debit automatically clears credit (and vice versa)
  - Null values properly handled for optional fields
  - Tab navigation between fields

### 4. Validation & Error Handling
- **Required Field Indicators**: Red asterisk on mandatory fields
- **Field-Level Validation**: Visual highlighting of problematic entries
- **Balance Validation**: Prevents saving unbalanced entries
- **Error Messages**: Clear display of all validation issues
- **Smart Error Recovery**: Errors clear when fixed

### 5. Enhanced Filtering & Search
- **Improved Filter UI**: Dedicated search input with clear button
- **Multi-field Search**: Searches both invoice numbers and descriptions
- **Visual Filter Indicator**: Shows when filtering is active

### 6. Helper Features
- **Tooltips**: Helpful hints on buttons and complex features
- **Keyboard Shortcuts**: Tab navigation for efficiency
- **Entry Tips**: Quick reference guide for new users
- **Formatted Numbers**: Proper number formatting with thousand separators

### 7. Mobile Responsiveness
- **Responsive Layout**: Adapts to smaller screens
- **Touch-Friendly Controls**: Larger tap targets on mobile
- **Flexible Grid**: Columns adjust based on screen size

## Technical Improvements

### Frontend Changes
1. **JournalEntriesImproved.jsx**: New enhanced component with all improvements
2. **Better State Management**: Optimized React hooks and memoization
3. **Proper Null Handling**: Fixed issue with sub_account_id being sent as empty string

### Backend Changes
1. **journal_entry.controller.js**:
   - Improved data handling for both array and object formats
   - Better null value processing
   - Enhanced error logging
   - Support for deleted entries tracking

## Usage Guide

### Creating Journal Entries
1. Click "Edit Entries" to enable edit mode
2. Use "Add Rows" to add new entry lines (adds 2 at a time for balance)
3. Fill in required fields (Account is mandatory)
4. Enter either debit OR credit amount (not both)
5. Use Tab key to navigate between fields
6. Click "Auto Balance" if entries don't balance
7. Click "Save Changes" when complete

### Tips for Users
- **Quick Duplication**: Use the copy icon to duplicate similar entries
- **Bulk Changes**: Edit multiple entries before saving
- **Validation**: Address red-highlighted fields before saving
- **Filtering**: Use the search box to find specific entries quickly
- **Links**: Click invoice/customer/product codes to navigate to details

## Files Modified
1. `/psfront/src/pages/GeneralEntries/JournalEntryBatch/JournalEntriesImproved.jsx` - New improved component
2. `/psfront/src/pages/GeneralEntries/JournalEntryBatch/AddJournalEntryBatch.jsx` - Updated to use new component
3. `/psfront/src/pages/GeneralEntries/JournalEntryBatch/JournalEntryBatches.jsx` - Listing now shows unbalanced batches with red Batch No. + AlertCircle icon + tooltip
4. `/psback/controllers/journal_entry.controller.js` - Enhanced data handling; `journalEntryBatches` now also returns `total_debit`, `total_credit`, and `is_balanced` per batch

## Benefits
- **Reduced Errors**: Better validation prevents incorrect entries
- **Faster Data Entry**: Auto-balance and duplication features save time
- **Better User Experience**: Clear feedback and intuitive controls
- **Professional Appearance**: Modern design inspires confidence
- **Improved Accuracy**: Visual cues help users spot issues quickly

## Manual JE ↔ Document Linkage Is Company-Scoped (2026-07-07)

**Problem found**: Manual JEs are linked to invoices/XOs by storing the document **number text** in `journal_entries.analysis_code1`. Document numbers are **unique per company only** — five companies had a "KHXO00000025". Every lookup matched by number alone, so company 9876's settlement JE (KHJV000214) showed on 5th Pillar's (1007) same-numbered XO, was subtracted from its balance ("Paid by JE"), and — worst — the JE post/void status recalc could **flip other companies' same-numbered invoice/XO statuses**.

**Fix**: every Manual-JE-by-document-number lookup is now scoped to the calling company via the JE batch's creator (`journal_batches.created_by → users.company_code`):
1. `psback/services/manualJeAdjustment.js` — all helpers accept a `companyCode` argument (`liveManualJeWhereClause` adds a required `createdBy` user include); `recalculateInvoiceStatusByNumber` / `recalculateCostStatusByDocNumber` additionally scope their invoice fetch (via service→order→user) and document fetch (via documents.user_id→user).
2. Callers now pass the company: `journal_entry.controller.js` (related-documents picker, `manualJeByOrder` order tab, JE create/update/void recalcs), `invoice.controller.js` (receipt settle/void), `payment.controller.js` (payment settle/delete, settlement print — derives company from the settlement's user since that route is unauthenticated), `customer.controller.js` (finance tab), `service.controller.js` (supplier XO listing), `report.controller.js` (supplier position/AP summary + AR Ageing Detail & Summary inline queries), `reports/dailySettlement.report.controller.js`.
3. `document.controller.js` `getDocument` (printed XO/invoice "Paid by JE" footer) — route is unauthenticated, so the company is derived from the rendered document's owner (`documents.user_id → users.company_code`).

**Behavior**: a company only ever sees/subtracts its **own** JEs on its documents. If `companyCode` is not passed (none of the current callers), the helpers fall back to the old unscoped behavior.