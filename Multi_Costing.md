Multiple Costing for a Service - Summary

Overview

The system supports multiple costs per service, allowing for cost voiding and regeneration while maintaining a complete audit trail.

Database Structure

- Relationship: One Service can have many Costs (1:N relationship)
- Association: Service.hasMany(Cost) with foreign key service_id and alias 'Cost'
- Status Field: Each cost has a status field that can be:
  - Raised - New cost, not yet printed
  - Printed - Cost has been printed/finalized
  - Void - Cost has been voided/cancelled
  - Paid - Cost has been fully paid
  - Partially Paid - Cost has been partially paid

How It Works

1. Initial Cost Creation

- When a service is created, a cost is generated with status Raised
- The cost stores all pricing details including published_rate, commission, net_rate, sst, and calculates total_costing

2. Cost Voiding Process

When a cost needs to be voided (e.g., due to errors or changes):
- The original cost's status is changed to Void
- A new replacement cost is automatically created with:
  - Same pricing details as the voided cost
  - Recalculated total_costing
  - Status set to Raised
  - New cost record ID (auto-generated)

3. Active Cost Selection

The system identifies the "active" cost using:
```javascript
const activeCost = service?.Cost
  ?.filter(cost => cost.status !== 'Void')  // Exclude voided costs
  ?.sort((a, b) => {                        // Get the most recent
    if (a.createdAt && b.createdAt) {
      return new Date(b.createdAt) - new Date(a.createdAt);
    }
    return (b.id || 0) - (a.id || 0);
  })?.[0];
```

4. Display Logic

- Service Tables: Show the total_costing from the active (non-voided) cost
- Cost Report: Lists only costs with Raised status for document generation
- Service Details: Displays pricing information from the latest active cost

5. Document Generation

- Only costs with Raised status can be selected for document generation
- Once printed, the cost status changes to Printed
- Printed costs cannot be edited but can be voided if needed

Key Differences from Invoice Multi-System

1. Status Values: Uses "Paid" and "Partially Paid" instead of "Settled" and "Partially Settled"
2. Field Names: Uses total_costing instead of total_price
3. Cost-specific fields: published_rate, commission, net_rate, sst, staff_comission, gst_inclusive

Business Rules

- Only one active (non-voided) cost per service at a time
- Cannot create a new cost if an active one exists (must void first)
- Voided costs remain in the system for record-keeping
- Cost IDs are never reused
- The total_costing field stores the final calculated amount to avoid recalculation inconsistencies
- Cost status is updated directly (not through documents like invoices)

API Endpoints

- GET /service/:id - Returns service with all costs (Cost array)
- POST /cost/:serviceId - Creates new cost (blocked if active cost exists)
- PUT /cost/void/:document_number - Voids cost and creates replacement
- GET /cost/order/listAllServiceCosts/:orderId - Lists costs for document generation

Important Implementation Notes

1. The Service model association has been updated from:
   ```javascript
   Service.hasOne(models.Cost, { foreignKey: 'service_id' });
   ```
   to:
   ```javascript
   Service.hasMany(models.Cost, { foreignKey: 'service_id', as: 'Cost' });
   ```

2. Helper function getActiveCost() is used to retrieve the current active cost from the array

3. For backward compatibility, the active cost is also exposed as service.Cost in some contexts

This architecture ensures data integrity while providing flexibility for corrections and maintaining a complete financial history for supplier payments and costing analysis.