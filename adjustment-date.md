# Adjustment Date Feature

## Overview

The **Adjustment Date** feature allows filtering reports based on when transactions were **posted to the ledger** rather than when they were **created**.

- **Created Date (`created_at`)**: When the document was initially created in the system
- **Adjustment Date (`adjustment_date`)**: When the document was posted/adjusted to the ledger

---

## How It Works

### When "Posted to Ledger" Checkbox is **UNCHECKED**

Simple filtering by `created_at` date only.

```
Filter by: created_at IN date_range
```

### When "Posted to Ledger" Checkbox is **CHECKED**

The system checks the `adjustment_date` field:

| adjustment_date | Filter Applied |
|-----------------|----------------|
| **NULL** (unposted) | Include only if `created_at` is in date range |
| **EXISTS** (posted) | Include only if `adjustment_date` is in date range |

```
Filter by:
  IF adjustment_date IS NULL:
    → created_at IN date_range
  ELSE:
    → adjustment_date IN date_range
```

---

## Example

**Filter Range**: 01 Nov 2025 to 30 Nov 2025
**Checkbox**: Checked (Posted to Ledger)

| Record | Created Date | Adjustment Date | Result |
|--------|-------------|-----------------|--------|
| A | 19-12-2025 | NULL | **Excluded** (created not in Nov, unposted) |
| B | 01-01-2026 | 15-11-2025 | **Included** (adj date in Nov) |
| C | 15-11-2025 | NULL | **Included** (created in Nov, unposted) |
| D | 01-01-2026 | 05-12-2025 | **Excluded** (adj date not in Nov) |
| E | 20-11-2025 | 25-11-2025 | **Included** (adj date in Nov) |

---

## Historical/Opening Balance Calculation

For opening balance (records before the filter start date), the same logic applies:

| adjustment_date | Filter Applied |
|-----------------|----------------|
| **NULL** (unposted) | Include only if `created_at < startDate` |
| **EXISTS** (posted) | Include only if `adjustment_date < startDate` |

---

## SQL Query Logic

### Current Period (Between Filter)
```sql
-- When adjustmentDateMode = true
WHERE (
    (adjustment_date IS NULL AND created_at BETWEEN startDate AND endDate)
    OR
    (adjustment_date IS NOT NULL AND adjustment_date BETWEEN startDate AND endDate)
)

-- When adjustmentDateMode = false
WHERE created_at BETWEEN startDate AND endDate
```

### Historical/Opening Balance
```sql
-- When adjustmentDateMode = true
WHERE (
    (adjustment_date IS NULL AND created_at < startDate)
    OR
    (adjustment_date IS NOT NULL AND adjustment_date < startDate)
)

-- When adjustmentDateMode = false
WHERE created_at < startDate
```

---

## Reports with Adjustment Date Feature

| Report | File Location |
|--------|---------------|
| Customer Account Statement | `psback/controllers/report.controller.js` |
| Customer Position Report | `psback/controllers/report.controller.js` |
| Supplier Account Statement | `psback/controllers/report.controller.js` |
| Supplier Position Report | `psback/controllers/report.controller.js` |
| Payment Listing Report | `psback/controllers/report.controller.js` |
| Customer Deposit Movement Report | `psback/controllers/report.controller.js` |

---

## Frontend Implementation

Each report has a checkbox in the filter form:

```jsx
<div className="grid grid-cols-9 gap-4">
  <label className="col-span-1 text-sm font-medium">
    Adjustment Date
  </label>
  <div className="col-span-3 flex items-center space-x-2">
    <input
      type="checkbox"
      id="adjustmentDateMode"
      checked={filter.adjustmentDateMode}
      onChange={(e) => setFilter({ ...filter, adjustmentDateMode: e.target.checked })}
      className="h-4 w-4 rounded border-gray-300"
    />
    <label htmlFor="adjustmentDateMode" className="text-sm">
      Posted to Ledger
    </label>
  </div>
</div>
```

---

## Backend Implementation (Sequelize)

```javascript
// When adjustmentDateMode is true
if (adjustmentDateMode) {
    w[Sequelize.Op.or] = [
        {
            [Sequelize.Op.and]: [
                { adjustment_date: null },
                { created_at: { [Sequelize.Op.between]: dateRange } }
            ]
        },
        {
            [Sequelize.Op.and]: [
                { adjustment_date: { [Sequelize.Op.ne]: null } },
                { adjustment_date: { [Sequelize.Op.between]: dateRange } }
            ]
        }
    ];
} else {
    w.created_at = { [Sequelize.Op.between]: dateRange };
}
```

---

## PDF/Report Column Display

When the "Posted to Ledger" checkbox is **checked**, an additional **Adjustment Date** column will appear in the generated report (PDF/Excel).

### Example: Supplier Account Statement - Vouchers Table

**Without Adjustment Date Checkbox:**
| Date | Voucher No. | Reference | Description | Cheque No. | Debit | Credit |

**With Adjustment Date Checkbox:**
| Date | Adj Date | Voucher No. | Reference | Description | Cheque No. | Debit | Credit |

### Implementation in EJS Template

```ejs
<%
  const voucherColumns = header.adjustmentDateMode
    ? ['date','adjustmentDate','voucherNo','reference','description','chequeNo','debit','credit']
    : ['date','voucherNo','reference','description','chequeNo','debit','credit'];
%>
<% renderTable('Vouchers', supplierData.vouchers, voucherColumns); %>
```

### Reports with Adjustment Date Column

| Report | Section | Column Added |
|--------|---------|--------------|
| Supplier Account Statement | Vouchers Table | Adj Date |
| Customer Account Statement | Vouchers Table | Adj Date |
| Customer Deposit Movement | Main Table | Adj Date |

---

## Business Logic Summary

1. **Unposted documents** (no adjustment_date): Filter by when they were created
2. **Posted documents** (has adjustment_date): Filter by when they were posted to ledger
3. This ensures accurate financial reporting based on ledger posting dates while still showing unposted documents within the creation date range
4. **Report Display**: When checkbox is checked, an "Adjustment Date" column is shown in the generated PDF/Excel report

---

## Bug Fixes

### Version 1.1 (March 2026) - Settlement Query Fix

**Bug**: Customer Deposit Movement Report was missing settlements for deposits with multiple receipt settlements.

**Root Cause**: The model association `customer_deposit.hasOne(receipt_settlement_deposit)` only returns ONE settlement per deposit. When a deposit had multiple settlements (e.g., TTDP00000001 had TTST00000001 for 10,100 AND TTST00000008 for 4,000), only one was shown.

**Impact**: Sub-Total balance for affected deposits was incorrect (showed 96,000 instead of correct 85,900 for TTDP00000001). Grand Total Amount and Debit columns were also wrong.

**Fix**: Instead of relying on the `hasOne` include, the report now queries `receipt_settlement_deposit.findAll()` separately for all deposit IDs and groups them by `customer_deposit_id`. This ensures ALL settlements are fetched regardless of the model association type.

**Files Changed**: `psback/controllers/report.controller.js` - `getCustomerDepositMovementReport()` (line ~4324)

---

## Customer Deposit Movement Report - Column Styling

### Column Width Configuration

The following columns have custom width settings in `customer-deposit-movement.ejs`:

| Column | Position | Min Width | Style |
|--------|----------|-----------|-------|
| Date | 2nd | 85px | `white-space: nowrap` |
| Amount | 7th | 130px | `white-space: nowrap` |
| Debit | 8th | 100px | `white-space: nowrap` |

### Amount Column Sub-Total Format

The Amount column displays settlement amounts with a Sub-Total line:

```
     10,000.00        (deposit amount)
      5,000.00        (settlement amount)
Sub-Total: 5,000.00   (with label and border-top)
```

### CSS Implementation

```css
/* Date column */
td:nth-child(2), th:nth-child(2) {
  min-width: 85px;
  white-space: nowrap;
}

/* Amount column */
td:nth-child(7), th:nth-child(7) {
  min-width: 130px;
  white-space: nowrap;
}

/* Debit column */
td:nth-child(8), th:nth-child(8) {
  min-width: 100px;
  white-space: nowrap;
}
```
