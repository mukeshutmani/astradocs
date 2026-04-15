# Customer Volume Report Changes Summary

## Date: November 6, 2025

### Changes Implemented:

## 1. ✅ Refund Handling - Enhanced to Deduct from Specific Service Types
**Previous behavior:**
- Refunds were tracked in a separate `total_refund` column
- Each service type column (air_sales, hotel_sales, etc.) showed gross sales
- Net sales calculation was: `net_sales = total_sales - total_refund`

**New behavior:**
- Refunds are now deducted directly from the specific service type they relate to
- If a refund is for an Air ticket, it reduces the `air_sales` column
- If a refund is for a Hotel booking, it reduces the `hotel_sales` column
- Each service type column now shows NET sales (after refunds)
- The `total_refund` column still shows the total refunded amount for reference
- `total_sales` now represents the sum of all net service type sales

### Example:
```
Before:
- Air ticket sale: PKR 50,000
- Air ticket refund: PKR 10,000
- Report showed: air_sales = 50,000, total_refund = 10,000

After:
- Air ticket sale: PKR 50,000
- Air ticket refund: PKR 10,000
- Report shows: air_sales = 40,000 (50,000 - 10,000), total_refund = 10,000
```

## 2. ✅ Combined Air Sales and Tax Columns
**Previous structure:**
- `air_ticket_sales`: Air sales without taxes
- `air_ticket_taxes`: Air ticket taxes only
- `air_total`: Combined sales + taxes

**New structure:**
- `air_sales`: Single column showing combined air sales including taxes
- Removed separate tax column for cleaner report layout
- Revenue column renamed from `air_ticket_revenue` to `air_revenue`

## Technical Implementation Details:

### File Modified: `psback/controllers/report.controller.js`

1. **Added refund tracking by service type** (lines 567-577):
   - Created `refundsByServiceType` object to track refunds per service category

2. **Categorized refunds by service type** (lines 899-930):
   - When processing credit notes, refunds are now categorized by their service type
   - Each refund amount is added to the appropriate service type tracker

3. **Deducted refunds from specific columns** (lines 1003-1025):
   - Before finalizing the customer data, refunds are deducted from each service type's sales
   - `total_sales` is recalculated as the sum of all net service type sales
   - `net_sales` now equals `total_sales` (since refunds are already deducted)

## Benefits of These Changes:

1. **Clearer reporting**: Each service type column now shows the actual net sales after refunds
2. **Better analysis**: Users can see the true sales performance for each service type
3. **Simplified layout**: Removed redundant air tax columns for cleaner presentation
4. **Maintains audit trail**: Still shows total refund amount for reconciliation purposes

## Testing Recommendations:

1. Generate a report with date range that includes refunded transactions
2. Verify that refunded amounts are correctly deducted from their respective service types
3. Confirm total_sales equals the sum of all individual service type sales
4. Check that the report matches expected values as shown in the screenshot

## Database Query for Verification:

```sql
-- Verify refund allocation by service type
SELECT
    st.type as service_type,
    COUNT(DISTINCT i.id) as invoice_count,
    SUM(i.total_price) as gross_sales,
    COUNT(DISTINCT cn.id) as refund_count,
    SUM(COALESCE(cn.refund_amount_base, 0)) as total_refunds,
    (SUM(i.total_price) - SUM(COALESCE(cn.refund_amount_base, 0))) as net_sales
FROM services s
JOIN service_types st ON s.service_type_id = st.id
JOIN invoices i ON i.service_id = s.id
LEFT JOIN refunds r ON r.invoice_no = i.invoice_number
LEFT JOIN credit_notes cn ON cn.id = r.credit_note_id
    AND cn.doc_status IN ('Printed', 'Settled', 'Partially Settled')
WHERE i.status IN ('Printed', 'Settled', 'Partially Settled')
GROUP BY st.type
ORDER BY st.type;
```

## Notes:
- The report now accurately reflects the business requirement where refunds impact the actual sales figures for each service type
- This aligns with the example shown in the screenshot where the refund amount (PKR 1,000) should be deducted from Air Ticket Sales

---

## Update: April 2026 — Multiple Invoices Per Service

### Problem
- A service could have more than one valid (Printed / Partially Settled / Settled) invoice, for example when a document was re-invoiced without voiding the earlier one.
- Previous code picked only the first Printed invoice per service using `invoices.find(...)`, silently dropping any additional invoices.
- Result: the report under-reported sales for the affected customer/service type.

### Fix
- `psback/controllers/report.controller.js` → `getCustomerVolume` now loops over **all** valid invoices for each service (Printed / Partially Settled / Settled, plus Void for credit-note processing).
- Sales, SST, invoice taxes, and refunds are aggregated per invoice.
- Cost calculation and supplier debit notes remain **once per service** (guarded by a `serviceCostProcessed` flag) to avoid double-counting when a service has multiple invoices.
- Date filter is applied per invoice; if every invoice on a service fails the date range, the service is skipped entirely and its cost is not counted.

### Behavior Change
- Services with multiple valid invoices now contribute every invoice's sales. Previously only one was counted.
- Customers who have duplicate Printed invoices on the same service will show higher totals after this fix.