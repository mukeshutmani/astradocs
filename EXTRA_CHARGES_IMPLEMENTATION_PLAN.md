# Extra Charges Implementation Plan

## Overview
This document outlines the implementation plan for adding an "Extra Charges" field to the cost calculation system in PowerSuite. The extra charges will be stored using the existing `free_of_cost` column and will be added to the cost after subtracting commission.

## Current Cost Calculation Formula

```
1. Commission Amount = Published Rate × Commission %
2. Net Rate = Published Rate - Commission Amount
3. WHT Amount = Commission Amount × SST %
4. Tax Amount = Sum of all cost taxes
5. Cost Per Unit = Net Rate + Tax Amount + WHT Amount
6. Total Cost = Cost Per Unit × Quantity
```

## New Cost Calculation Formula (With Extra Charges)

```
1. Commission Amount = Published Rate × Commission %
2. Net Rate = Published Rate - Commission Amount
3. Extra Charges = free_of_cost field (repurposed)
4. Net Rate After Extra = Net Rate + Extra Charges  // NEW STEP
5. WHT Amount = Commission Amount × SST %
6. Tax Amount = Sum of all cost taxes
7. Cost Per Unit = Net Rate After Extra + Tax Amount + WHT Amount
8. Total Cost = Cost Per Unit × Quantity
```

## Implementation Checklist

### Phase 1: Database & Model Updates

- [x] **Migration Script**
  - [x] Create migration to add column alias or comment for `free_of_cost` → `extra_charges`
  - [x] Update any existing seed data or test data

- [x] **Model Updates**
  - [x] Update `psback/models/cost.js` to document the new purpose of `free_of_cost`
  - [x] Add validation rules for extra charges (non-negative)

### Phase 2: Backend Controller Updates

- [x] **cost.controller.js** (`psback/controllers/cost.controller.js`)
  - [x] Line 128: Add extra charges to calculation
    ```javascript
    const extra_charges = parseFloat(activeCost?.free_of_cost || 0);
    const net_rate_with_extra = net_rate + extra_charges;
    ```
  - [x] Line 139: Update cost per unit calculation
    ```javascript
    const cost_per_unit = net_rate_with_extra + tax_amount + wht_amount;
    ```
  - [x] Lines 358-377: Update void cost replacement logic
  - [x] Lines 465-484: Update previously voided cost replacement logic

- [x] **payment.controller.js** (`psback/controllers/payment.controller.js`)
  - [x] Line 362-367: Include extra charges in total cost calculation
  - [x] Line 523-531: Update cost calculation for payment status
  - [x] Line 784-800: Update settlement status calculation

- [x] **report.controller.js** (`psback/controllers/report.controller.js`)
  - [x] Line 1085: Add extra charges to cost calculation
  - [x] Line 1752: Include extra charges in invoice cost comparison
  - [x] Line 3009: Update total cost of sales calculation
  - [x] Line 7373: Include extra charges in net rate calculation
  - [x] Line 9519: Update cost aggregation logic
  - [x] Line 10419: Include extra charges in total cost
  - [x] Line 11792: Update service cost calculation
  - [x] All other cost calculation sections

- [x] **service.controller.js** (`psback/controllers/service.controller.js`)
  - [x] Update any cost-related calculations to include extra charges
  - [x] Ensure extra charges are preserved when updating services

### Phase 3: Document Generation Updates

- [x] **costDocument.ejs** (`psback/views/pages/costDocument.ejs`)
  - [x] Line 375-377: Update cost calculation
    ```javascript
    const extra_charges = parseFloat(cost?.free_of_cost || 0);
    const netRateWithExtra = netRate + extra_charges;
    let totalUnit = netRateWithExtra + taxAmount;
    ```
  - [x] Lines 528-532: Add display of extra charges in the document (if non-zero)
    ```javascript
    <% if (extraCharges && extraCharges > 0) { %>
    <div class="price-breakdown">
      Extra Charges: <%= currency?.currency_code || 'PKR' %> <%= extraCharges.toLocaleString('en-US', {minimumFractionDigits: 2, maximumFractionDigits: 2}) %>
    </div>
    <% } %>
    ```

- [x] **paymentSettlement.ejs** (`psback/views/pages/paymentSettlement.ejs`)
  - [x] Lines 589-591: Add extra charges to net rate calculation
    ```javascript
    const extraCharges = parseFloat(cost?.free_of_cost || 0);
    const netRateWithExtra = netRate + extraCharges;
    const costPerUnit = netRateWithExtra + taxAmount + whtAmount;
    ```

### Phase 4: Frontend Service Updates

- [x] **CostCalculationService.js** (`psfront/src/services/CostCalculationService.js`)
  - [x] Line 33: Add extraCharges parameter
    ```javascript
    static calculateCostValues(params) {
      const {
        // ... existing params
        extraCharges = 0,  // NEW
      } = params;
    ```
  - [x] Line 68: Calculate net rate with extra charges
    ```javascript
    const nettRate = safePublishedRate - finalCommissionAmount;
    const nettRateWithExtra = nettRate + this._parseFloat(extraCharges);
    ```
  - [x] Line 79: Update cost per unit calculation
    ```javascript
    const costPerUnit = nettRateWithExtra + totalTaxPerUnit;
    ```

### Phase 5: Frontend Component Updates

- [x] **Cost.jsx** (`psfront/src/components/service/Cost.jsx`)
  - [x] Line 36: Add extra_charges to initial state
    ```javascript
    extra_charges: cost?.free_of_cost || 0,
    ```
  - [x] Add extra_charges to values that trigger recalculation
  - [x] Pass extra_charges to calculation service

- [x] **CostPricing.jsx** (`psfront/src/components/service/Cost/CostPricing.jsx`)
  - [x] Add UI field for Extra Charges input
  - [x] Place field after commission section
  - [x] Add appropriate validation and formatting

- [x] **CostSummary.jsx** (`psfront/src/components/service/Cost/CostSummary.jsx`)
  - [x] Display extra charges in the cost summary
  - [x] Show calculation breakdown including extra charges

- [x] **useServiceCalculations.js** (`psfront/src/hooks/service/useServiceCalculations.js`)
  - [x] Line 33: Add extraCharges to transformed params
    ```javascript
    extraCharges: valuesRef.current.extra_charges || valuesRef.current.free_of_cost,
    ```

### Phase 6: Frontend Display Components

- [x] **RefundCalculator.jsx** (`psfront/src/components/RefundCalculator.jsx`)
  - [x] Include extra charges in refund calculations
  - [x] Add supplierExtraCharges state variable
  - [x] Integrate extra charges into supplier net calculation: `nett = grossSupplier - commissionAmount + extraCharges`
  - [x] Add extra charges to handleApply and handleClear functions
  - [x] Add UI field for Extra Charges in supplier section

- [x] **OrderDetail.jsx** (`psfront/src/pages/OrderDetail.jsx`)
  - [x] Update summary calculation section (Lines 1592-1610): Add extra charges to cost per unit calculation
  - [x] Update services table display section (Lines 3248-3263): Add extra charges to cost calculation for table display
  - [x] Update first order summary section (Lines 4235-4258): Add extra charges to order totals calculation
  - [x] Update second order summary section (Lines 4350-4373): Add extra charges to alternative order summary calculation
  - [x] All sections now include: `extra_charges = parseFloat(cost?.free_of_cost || 0)` and `net_rate_with_extra = net_rate + extra_charges`

- [x] **CostList.jsx** (`psfront/src/pages/Document/CostList.jsx`)
  - [x] Show extra charges in cost lists

- [x] **CostReport.jsx** (`psfront/src/pages/Order/CostReport.jsx`)
  - [x] Include extra charges in cost reports

### Phase 7: Additional Components

- [x] **VisaCost.jsx** (`psfront/src/components/service/VisaCost.jsx`)
  - [x] Add extra charges support for visa services

- [x] **HotelCost.jsx** (`psfront/src/components/service/HotelCost.jsx`)
  - [x] Add `extra_charges` to formData state initialization (Line 66)
  - [x] Update `calculateValues` function to include extra charges in calculation (Lines 160-178)
  - [x] Update `calculateWithData` function to include extra charges (Lines 239-257)
  - [x] Add `free_of_cost` mapping in `updateFormData` for backend (Line 297)
  - [x] Add UI input field for Extra Charges after Net Rate (Lines 858-884)
  - [x] Add extra charges display in summary section (Lines 1183-1188)
  - [x] Update cost per unit calculation to include extra charges (Line 1199)

- [x] **Payments.jsx** (`psfront/src/pages/PaymentsPage/Payments.jsx`)
  - [x] Ensure extra charges are included in payment calculations

### Phase 7.5: Journal Entries Integration

- [x] **journal.js** (`psback/services/journal.js`)
  - [x] Section 1 (Lines 283-387): Invoice/Cost posting journal entries
    - [x] Add `extraCharges` and `netRateWithExtra` variables (Lines 284-285)
    - [x] Update COST entry type to use `netRateWithExtra` (Line 291)
    - [x] Update APAY entry type to use `netRateWithExtra` (Line 304)
    - [x] Update IATA entry type to include extra charges (Line 335)
    - [x] Update CSAL entry type to use `netRateWithExtra` (Line 382)
  - [x] Section 2 (Lines 515-580): Cost document journal entries
    - [x] Add `extraCharges` and `netRateWithExtra` variables (Lines 533-534)
    - [x] Update APAY entry type to use `netRateWithExtra` (Line 540)
    - [x] Update IATA entry type to use `netRateWithExtra` (Line 549)
    - [x] Update CSAL entry type to use `netRateWithExtra` (Line 569)
  - [x] Section 3 (Lines 1130-1180): Payment journal entries - No changes needed (uses pre-calculated amounts)
  - [x] Section 4 (Lines 1912-1985): Batch invoice journal entries
    - [x] Add `extraCharges` variable (Line 1916)
    - [x] Update IATA entry type to include extra charges (Line 1955)
    - [x] Update CSAL entry type to include extra charges (Lines 1978-1980)
  - [x] Section 5 (Lines 2078-2120): Regenerate journal entries
    - [x] Add `extraCharges` and `netRateWithExtra` variables (Lines 2085-2086)
    - [x] Update APAY entry type to use `netRateWithExtra` (Line 2092)
    - [x] Update IATA entry type to use `netRateWithExtra` (Line 2100)
    - [x] Update CSAL entry type to use `netRateWithExtra` (Line 2116)

### Phase 8: Testing Requirements

#### Unit Tests
- [ ] Test cost calculation with zero extra charges
- [ ] Test cost calculation with positive extra charges
- [ ] Test cost calculation with extra charges and multiple currencies
- [ ] Test cost calculation with extra charges and taxes
- [ ] Test cost calculation with extra charges and quantity > 1

#### Integration Tests
- [ ] Test creating a cost with extra charges via API
- [ ] Test updating a cost with extra charges
- [ ] Test voiding a cost with extra charges
- [ ] Test payment settlement with extra charges
- [ ] Test report generation with extra charges
- [ ] Test journal entry generation includes extra charges in all entry types (COST, APAY, IATA, CSAL)

#### Manual Testing Scenarios
- [ ] Create a new service with extra charges
- [ ] Edit existing service to add extra charges
- [ ] Generate cost document with extra charges
- [ ] Create payment settlement including extra charges
- [ ] Generate reports showing extra charges
- [ ] Test with different service types (Air, Hotel, Visa, etc.)
- [ ] Test with multi-currency scenarios
- [ ] Test with bulk operations

### Phase 9: Data Migration

- [ ] **Migration Script for Existing Data**
  - [ ] Set default value of 0 for all existing free_of_cost fields if NULL
  - [ ] Document any special cases or exceptions

### Phase 10: Documentation

- [ ] Update API documentation for cost endpoints
- [ ] Update user guide for extra charges feature
- [ ] Create training material for staff
- [ ] Update system architecture documentation

## Rollback Plan

If issues arise during implementation:
1. The `free_of_cost` field remains backward compatible
2. Setting extra charges to 0 reverts to original calculation
3. Database changes are minimal (using existing column)
4. Frontend changes can be feature-flagged if needed

## Success Criteria

1. Extra charges can be added to any service cost
2. Extra charges are correctly added after commission subtraction
3. All reports accurately reflect extra charges
4. Documents display extra charges when present
5. Payment calculations include extra charges
6. No regression in existing cost calculations

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Calculation errors | High | Comprehensive unit testing |
| Report inconsistencies | Medium | Validate all report calculations |
| UI/UX confusion | Low | Clear labeling and help text |
| Performance impact | Low | Minimal calculation overhead |

## Timeline Estimate

- Phase 1-2: 1 day (Backend core changes)
- Phase 3-4: 1 day (Documents and frontend services)
- Phase 5-7: 2 days (Frontend components)
- Phase 8: 2 days (Testing)
- Phase 9-10: 1 day (Migration and documentation)

**Total Estimated Time: 7 days**

## Notes

- The `free_of_cost` column is repurposed to store extra charges
- Extra charges are always added (never subtracted)
- Currency conversion applies to extra charges
- Extra charges are included in all totals and reports
- The field should be clearly labeled in the UI to avoid confusion

## Approval

- [ ] Technical Lead Review
- [ ] Business Stakeholder Approval
- [ ] Testing Team Sign-off
- [ ] Documentation Complete

---

*Last Updated: [Current Date]*
*Version: 1.0*