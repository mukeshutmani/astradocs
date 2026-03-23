# Customer Volume Report - Debit Notes Implementation

## Date: November 6, 2025

## Overview
Extended the Customer Volume Report to handle debit notes on the cost side, mirroring the refund (credit note) handling on the sales side. The report now provides accurate profit/revenue calculations based on adjusted sales and costs.

## Important Accounting Principles:
- **Credit Notes**: Money we owe customers (reduces our sales)
- **Debit Notes**: Money suppliers owe us (reduces our costs)

## Changes Implemented:

### 1. ✅ Debit Note Tracking by Service Type
**Previous behavior:**
- Costs were tracked as a single total per service
- Debit notes existed in the system but weren't included in cost calculations
- Revenue calculation: `revenue = sales - base_costs`

**New behavior:**
- Debit notes are now tracked by service type (similar to refunds)
- Debit notes REDUCE the costs of their respective service types
- Each service type's net cost: `base_cost - debit_notes`
- Revenue calculation: `revenue = net_sales (after refunds) - net_costs (after debit notes)`

### 2. ✅ Cost Adjustment Formula
```
For each service type:
- Base Cost = Original cost from cost records
- Debit Notes = Sum of debit notes from suppliers (money owed back to us)
- Net Cost = Base Cost - Debit Notes
- Revenue = Net Sales (after refunds) - Net Cost
```

### 3. ✅ New Report Columns
- `total_debit_notes`: Shows total debit note amounts that reduced costs
- Revenue columns now reflect accurate profit after both sales and cost adjustments

## Technical Implementation:

### Database Relationships:
- Services have a `supplier_id`
- Debit notes also have a `supplier_id`
- Debit notes are linked to services through the supplier relationship

### Code Changes in `psback/controllers/report.controller.js`:

1. **Added debit note inclusion in query** (lines 439-451):
   - Joined supplier with debit_notes to fetch valid debit notes
   - Only includes debit notes with status: Printed, Settled, or Partially Settled

2. **Added tracking objects** (lines 592-616):
   - `debitNotesByServiceType`: Tracks debit notes per service category
   - `costsByServiceType`: Tracks base costs per service category

3. **Process costs and debit notes** (lines 816-905):
   - Track base costs by service type
   - Process debit notes from suppliers
   - Allocate debit notes to appropriate service types
   - Handle currency conversion for foreign currency debit notes

4. **Revenue calculation** (lines 1173-1206):
   - Calculate net cost for each service type: `base_cost - debit_notes`
   - Calculate revenue for each service type: `net_sales - net_cost`
   - Sum all revenues for total revenue

## Example Scenario:

### Before Implementation:
```
Air ticket sale: PKR 50,000
Air ticket refund (credit note): PKR 5,000
Air ticket cost: PKR 30,000
Supplier debit note: PKR 2,000

Old calculation:
- Air Sales shown: 50,000
- Air Revenue: 50,000 - 30,000 = 20,000
- Debit note not considered
```

### After Implementation:
```
Air ticket sale: PKR 50,000
Air ticket refund (credit note): PKR 5,000 (customer gets money back)
Air ticket cost: PKR 30,000
Supplier debit note: PKR 2,000 (supplier owes us money back)

New calculation:
- Air Sales shown: 45,000 (50,000 - 5,000 refund)
- Air Net Cost: 28,000 (30,000 - 2,000 debit note)
- Air Revenue: 45,000 - 28,000 = 17,000
```

## Benefits:

1. **Accurate Profit Calculation**: Revenue now reflects true profit after all adjustments
2. **Symmetrical Handling**: Credit notes reduce sales, debit notes increase costs
3. **Service-Level Accuracy**: Each service type shows its true profitability
4. **Complete Financial Picture**: Report shows:
   - Net sales (after refunds)
   - Total refunds
   - Total debit notes
   - Accurate revenue/profit

## Testing Checklist:

- [x] Debit notes are fetched from the database
- [x] Debit notes are allocated to correct service types
- [x] Foreign currency debit notes are converted to PKR
- [x] Costs include debit note amounts
- [x] Revenue calculations use adjusted costs
- [x] Report shows total_debit_notes column
- [x] Each service type's revenue reflects adjusted calculations

## SQL Query for Verification:

```sql
-- Verify debit note impact on costs and revenue
SELECT
    st.type as service_type,
    COUNT(DISTINCT s.id) as service_count,
    SUM(c.total_costing) as base_costs,
    COUNT(DISTINCT dn.id) as debit_note_count,
    SUM(COALESCE(dn.base_amount, dn.amount, 0)) as total_debit_notes,
    (SUM(c.total_costing) - SUM(COALESCE(dn.base_amount, dn.amount, 0))) as net_costs_after_debit,
    SUM(i.total_price) as gross_sales,
    SUM(COALESCE(cn.refund_amount_base, 0)) as total_refunds,
    (SUM(i.total_price) - SUM(COALESCE(cn.refund_amount_base, 0))) as net_sales,
    ((SUM(i.total_price) - SUM(COALESCE(cn.refund_amount_base, 0))) -
     (SUM(c.total_costing) - SUM(COALESCE(dn.base_amount, dn.amount, 0)))) as revenue
FROM services s
JOIN service_types st ON s.service_type_id = st.id
LEFT JOIN costs c ON c.service_id = s.id AND c.status != 'Void'
LEFT JOIN invoices i ON i.service_id = s.id
LEFT JOIN suppliers sup ON s.supplier_id = sup.id
LEFT JOIN debit_notes dn ON dn.supplier_id = sup.id
    AND dn.doc_status IN ('Printed', 'Settled', 'Partially Settled')
LEFT JOIN refunds r ON r.invoice_no = i.invoice_number
LEFT JOIN credit_notes cn ON cn.id = r.credit_note_id
    AND cn.doc_status IN ('Printed', 'Settled', 'Partially Settled')
WHERE i.status IN ('Printed', 'Settled', 'Partially Settled')
GROUP BY st.type
ORDER BY st.type;
```

## Summary:

The Customer Volume Report now provides a complete financial picture:
- **Sales side**: Refunds (credit notes) are deducted from sales for each service type
- **Cost side**: Debit notes are deducted from costs for each service type
- **Revenue**: Calculated as net sales minus net costs, providing true profitability

This ensures that the report accurately reflects the business reality where:
- Credit notes = money we owe customers (reduces our sales)
- Debit notes = money suppliers owe us (reduces our costs)
- Revenue/profit = net sales (after refunds) - net costs (after debit notes)