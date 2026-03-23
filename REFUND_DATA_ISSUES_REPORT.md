# Refund Calculator Data Issues Report
**Date:** 2025-10-21
**Issue:** Incorrect supplier refund amounts and net_rate data

## Executive Summary

Two critical issues were identified in the refund system:
1. **Code Issue**: TourRefundHandler was incorrectly calculating supplier refunds as 90% of customer refunds
2. **Data Issue**: Net rates are being saved incorrectly (appears to be 10% of actual value in many cases)

## Issues Identified

### 1. Incorrect Refund Calculation Logic (ALL FIXED)
- **Problem**: Multiple handlers used arbitrary percentages instead of actual cost data
- **Impact**: ALL service type refunds had incorrect supplier amounts
- **Status**: ✅ **ALL HANDLERS FIXED** (2025-10-21)
- **Fixed Handlers**:
  - ✅ **TourRefundHandler.js**: Was using 90%, now uses actual cost data
  - ✅ **HotelRefundHandler.js**: Was using 90%, now uses actual cost data
  - ✅ **CarRentalRefundHandler.js**: Was using 90%, now uses actual cost data
  - ✅ **TrainRefundHandler.js**: Was using 90%, now uses actual cost data
  - ✅ **CarTransferRefundHandler.js**: Was using 95%, now uses actual cost data
  - ✅ **VisaRefundHandler.js**: Was using 95%, now uses actual cost data
  - ✅ **InsuranceRefundHandler.js**: Was using 95%, now uses actual cost data

### 2. Incorrect Net Rate Data
- **Location**: Database `costs` table
- **Problem**: Net rates appear to be saved as 10% of actual value
- **Example**: Service 2062 shows net_rate=4,000 when it should be 40,000

## Affected Records

### Tour Refunds with Incorrect Supplier Amounts

| Refund No | Customer Amount | Supplier Amount | Should Be | Issue |
|-----------|-----------------|-----------------|-----------|-------|
| TTRF00000012 | 42,000 | 37,800 | Based on actual cost | 90% of customer |
| TTRF00000011 | 42,000 | 37,800 | Based on actual cost | 90% of customer |
| ETRF00000053 | 795,000 | 800,000 | 80,000 | Exceeds customer |
| TTRF00000005 | 27,500 | 30,000 | 3,000 | Exceeds customer |

### Services with Suspicious Net Rates (90% margin pattern)

| Service ID | Package | Net Rate | Published Rate | Margin % | Likely Issue |
|------------|---------|----------|----------------|----------|--------------|
| 2062 | MALAYSIA PKG | 4,000 | 40,000 | 90% | Net rate too low |
| 2126 | Ultra Tour | 900 | 10,000 | 91% | Net rate too low |
| 2107 | 15 Days Vietnam | 20,000 | 200,000 | 90% | Net rate too low |
| 1855 | MALAYSIA TOUR PKG | 3,000 | 30,000 | 90% | Net rate too low |

## SQL Queries for Analysis

### 1. Find Tour refunds with supplier amounts that are 90% of customer amounts:
```sql
SELECT
    refund_no,
    service_type,
    customer_refund_amount,
    supplier_refund_amount,
    ROUND(supplier_refund_amount / customer_refund_amount * 100, 2) as percentage
FROM refunds
WHERE service_type = 'Tour'
    AND ABS(supplier_refund_amount - (customer_refund_amount * 0.9)) < 100
ORDER BY created_at DESC;
```

### 2. Find services with suspiciously high margins (>85%):
```sql
SELECT
    s.id as service_id,
    st.type as service_type,
    c.net_rate,
    c.published_rate,
    c.total_costing,
    ROUND(((c.published_rate - c.net_rate) / c.published_rate * 100), 2) as margin_percentage
FROM costs c
JOIN services s ON c.service_id = s.id
JOIN service_types st ON s.service_type_id = st.id
WHERE c.published_rate > 0
    AND ((c.published_rate - c.net_rate) / c.published_rate) > 0.85
ORDER BY margin_percentage DESC;
```

### 3. Check if net_rate is consistently 10% of published_rate:
```sql
SELECT
    COUNT(*) as total_records,
    SUM(CASE WHEN ABS(net_rate - (published_rate * 0.1)) < 1 THEN 1 ELSE 0 END) as likely_errors,
    ROUND(SUM(CASE WHEN ABS(net_rate - (published_rate * 0.1)) < 1 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) as error_percentage
FROM costs
WHERE published_rate > 0;
```

## Recommended Actions

### Immediate Actions
1. ✅ **Fix Code**: Remove 90% calculation from TourRefundHandler (COMPLETED)
2. **Data Audit**: Run analysis queries to identify all affected records
3. **Manual Review**: Have finance team review the identified records

### Data Correction Plan
1. **Identify Pattern**: Determine if net_rate is consistently 10% of what it should be
2. **Correct Net Rates**: Update costs table with correct net_rate values
3. **Recalculate Refunds**: Update supplier_refund_amount for affected refunds
4. **Validation**: Cross-check with original invoices/purchase orders

### Prevention
1. **Add Validation**: Implement checks to prevent margins > 85%
2. **Audit Trail**: Log all cost entries with user and timestamp
3. **Review Process**: Require approval for high-margin entries
4. **Testing**: Add unit tests for refund calculations

## Sample Correction Queries

**Note: These queries should be reviewed and approved before execution**

### Fix specific refund (example for TTRF00000012):
```sql
-- First verify the correct amounts
SELECT
    r.refund_no,
    r.supplier_refund_amount as current_amount,
    c.total_costing as actual_cost,
    c.net_rate,
    c.published_rate
FROM refunds r
JOIN costs c ON r.cost_id = c.id
WHERE r.refund_no = 'TTRF00000012';

-- Update if confirmed (DO NOT RUN without verification)
-- UPDATE refunds
-- SET supplier_refund_amount = [VERIFIED_AMOUNT],
--     supplier_refund_amount_base = [VERIFIED_AMOUNT]
-- WHERE refund_no = 'TTRF00000012';
```

### Fix net_rate if pattern is confirmed:
```sql
-- If net_rate is consistently 10% of what it should be
-- UPDATE costs
-- SET net_rate = net_rate * 10
-- WHERE [CONDITIONS_AFTER_VERIFICATION];
```

## Code Changes Made

### TourRefundHandler.js
- **Removed**: Hardcoded 90% calculation
- **Added**: Proper cost data retrieval
- **Added**: Error handling for missing cost data
- **Priority**: total_costing > net_rate > published_rate-commission

### invoice.controller.js
- **Added**: Include cost model when fetching service for refunds
- **Ensures**: Cost data is available for calculation

### RefundCalculator.jsx
- **Added**: Validation for unreasonable supplier refund amounts
- **Added**: Warning logs for suspicious data
- **Improved**: Handling of stored vs calculated values

## Next Steps

1. **Executive Review**: Present findings to management
2. **Data Team**: Audit and correct net_rate values
3. **Finance Team**: Validate corrected refund amounts
4. **Testing**: Comprehensive testing of refund calculations
5. **Monitoring**: Set up alerts for suspicious patterns

## Contact
For questions about this report or the fixes implemented, please contact the development team.