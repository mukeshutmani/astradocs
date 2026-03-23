# PowerSuite Cost Calculation System - Complete Analysis

## Executive Summary

This document provides a comprehensive analysis of how costs are calculated in the PowerSuite service page, covering both frontend and backend implementations. The analysis reveals several critical discrepancies in cost saving and calculation logic that may lead to data inconsistencies.

## Table of Contents
1. [System Architecture Overview](#system-architecture-overview)
2. [Frontend Cost Calculation](#frontend-cost-calculation)
3. [Backend Cost Processing](#backend-cost-processing)
4. [Database Schema](#database-schema)
5. [Calculation Formulas](#calculation-formulas)
6. [Critical Discrepancies Found](#critical-discrepancies-found)
7. [Data Flow Diagram](#data-flow-diagram)
8. [Recommendations](#recommendations)

---

## 1. System Architecture Overview

### Key Components
- **Frontend**: React with Redux state management
- **Backend**: Express.js with Sequelize ORM
- **Database**: MySQL
- **Base Currency**: PKR (Pakistani Rupee)

### Two-Part Calculation System
1. **Cost Side**: Supplier costs (what the business pays)
2. **Price Side**: Customer prices (what customers pay)

---

## 2. Frontend Cost Calculation

### 2.1 Core Calculation Services

#### CostCalculationService.js (`psfront/src/services/CostCalculationService.js`)
Handles all cost-side calculations:

```javascript
// Cost Calculation Formula (lines 34-91):
1. commission_amount = (published_rate * commission_percent) / 100
2. net_rate = published_rate - commission_amount
3. net_rate_with_extra = net_rate + extra_charges
4. tax_amount = SUM(all tax amounts from taxes array)
5. sst_amount = (commission_amount * sst_percent) / 100  // WHT on commission
6. total_tax_per_unit = tax_amount + sst_amount
7. cost_per_unit = net_rate_with_extra + total_tax_per_unit
8. total_cost_local = cost_per_unit * quantity
9. total_cost = convertToBaseCurrency(total_cost_local, exchange_rate)
```

**Key Fields Returned:**
- `published_rate`: Supplier's quoted rate
- `commission_amount`: Calculated commission
- `net_rate`: Rate after commission deduction
- `extra_charges`: Additional charges (free_of_cost field)
- `total_unit`: Cost per unit (net_rate_with_extra)
- `total_cost`: Final cost in PKR
- `total_costing`: Same as total_cost (for backend compatibility)

#### PriceCalculationService.js (`psfront/src/services/PriceCalculationService.js`)
Handles all price-side calculations:

```javascript
// Price Calculation Formula (lines 40-101):
1. base_price = (if not set, use published_rate with currency conversion)
2. price_with_markup = base_price + markup
3. discount_amount = (price_with_markup * discount_percent) / 100
4. rebate_amount = (price_with_markup * rebate_percent) / 100
5. total_unit_before_tax = price_with_markup - discount_amount - rebate_amount
6. tax_amount = SUM(all tax amounts)
7. total_unit = total_unit_before_tax + tax_amount
8. transaction_fee_sst = (transaction_fee * sst_percent) / 100
9. subtotal_local = (total_unit * quantity) + transaction_fee + transaction_fee_sst
10. total_sales = convertToBaseCurrency(subtotal_local, exchange_rate)
```

### 2.2 Component Structure

#### Main Components
- `Cost.jsx`: Orchestrates cost calculations using hooks
- `Price.jsx`: Orchestrates price calculations using hooks
- `HotelCost.jsx`: Hotel-specific cost handling
- `HotelPrice.jsx`: Hotel-specific price handling

#### Custom Hooks
- `useServiceCalculations.js`: Manages debounced calculations
- `useCurrencySelection.js`: Currency and exchange rate management
- `useExchangeRate.js`: Exchange rate validation
- `useTaxManagement.js`: Tax addition/removal
- `useTransactionFee.js`: Transaction fee calculations

### 2.3 Currency Handling

**Special PKR Handling:**
- When currency is PKR (code '110'), exchange rate is forced to 1.0
- All amounts are ultimately converted to PKR for storage
- Currency conversion uses: `convertToBaseCurrency(amount, exchangeRate)`

---

## 3. Backend Cost Processing

### 3.1 Service Controller (`psback/controllers/service.controller.js`)

#### Cost Creation/Update (lines 2464-2663)

**Critical Hotel Service Exception (lines 2522-2536, 2610-2623):**
```javascript
// For Hotel services with non-PKR currency and exchange_rate > 1:
if (serviceType.type === 'Hotel' && cost.currency !== '110' && exchangeRate > 1) {
    // Convert PKR value BACK to original currency before saving
    costDataForUpdate.total_costing = (parseFloat(cost.total_costing) / exchangeRate).toFixed(2);
}
```

**⚠️ DISCREPANCY #1**: Hotel services store `total_costing` in original currency, not PKR!

### 3.2 Cost Controller (`psback/controllers/cost.controller.js`)

#### Cost Retrieval (lines 125-147)

**Dynamic Recalculation:**
```javascript
// Backend RECALCULATES costs instead of using stored total_costing:
const published_rate = parseFloat(activeCost?.published_rate || 0);
const commission_percent = parseFloat(activeCost?.commission || 0);
const commission_amount = (published_rate * commission_percent) / 100;
const net_rate = published_rate - commission_amount;
const extra_charges = parseFloat(activeCost?.free_of_cost || 0);
const net_rate_with_extra = net_rate + extra_charges;
const sst_percent = parseFloat(activeCost?.sst || 0);
const wht_amount = (commission_amount * sst_percent) / 100;
const tax_amount = SUM(cost_taxes);
const cost_per_unit = net_rate_with_extra + tax_amount + wht_amount;
const totalCost = cost_per_unit * quantity;
```

**⚠️ DISCREPANCY #2**: Backend recalculates costs dynamically, ignoring stored `total_costing`!

### 3.3 Refund Handlers

All refund handlers (`psback/services/refund/handlers/`) use priority logic:

```javascript
// Priority for cost determination:
if (total_costing > 0) {
    use total_costing;
} else if (net_rate > 0) {
    use net_rate * quantity;
} else if (published_rate > 0 && commission > 0) {
    use published_rate * quantity * (1 - commission);
} else {
    use published_rate * quantity;
}
```

**⚠️ DISCREPANCY #3**: Refund handlers trust stored `total_costing`, while cost controller doesn't!

---

## 4. Database Schema

### 4.1 Costs Table
```sql
CREATE TABLE `costs` (
    `id` int PRIMARY KEY AUTO_INCREMENT,
    `published_rate` decimal(10,2),         -- Supplier's rate
    `commission` decimal(10,5),             -- Commission percentage
    `net_rate` decimal(10,2),               -- After commission
    `sst` decimal(10,2),                    -- WHT percentage
    `free_of_cost` decimal(10,2),           -- Extra charges
    `quantity` int,
    `currency` varchar(3),
    `total_costing` decimal(10,2),          -- Calculated total
    `amount` decimal(10,2) DEFAULT '0.00',  -- Legacy/unused field
    `service_id` int NOT NULL,
    `status` enum('Paid','Partially Paid','Raised','Void','Printed')
)
```

### 4.2 Cost_Taxes Table
```sql
CREATE TABLE `cost_taxes` (
    `id` int PRIMARY KEY AUTO_INCREMENT,
    `cost_id` int NOT NULL,
    `tax_code` varchar(255) DEFAULT 'tax',
    `tax_amount` decimal(10,2)
)
```

### 4.3 Invoices Table (Price Side)
```sql
CREATE TABLE `invoices` (
    `id` int PRIMARY KEY AUTO_INCREMENT,
    `price` decimal(10,2),                  -- Base price
    `markup` decimal(10,2),
    `discount` decimal(10,6),
    `rebate` decimal(10,6),
    `transaction_fee` float,
    `sst` varchar(45) DEFAULT '0',          -- SST amount
    `total_price` decimal(10,2),            -- Final price
    `quantity` int,
    `service_id` int NOT NULL
)
```

---

## 5. Calculation Formulas

### 5.1 Cost Side Formula
```
Published Rate (from supplier)
├── minus Commission (% or manual amount)
│   └── = Net Rate
├── plus Extra Charges (free_of_cost)
│   └── = Net Rate with Extra
├── plus Regular Taxes (from cost_taxes)
├── plus WHT/SST (on commission amount)
│   └── = Cost Per Unit
└── multiply by Quantity
    └── = Total Cost (in local currency)
        └── Convert to PKR = Total Costing
```

### 5.2 Price Side Formula
```
Base Price (or converted published rate)
├── plus Markup
│   └── = Price with Markup
├── minus Discount (%)
├── minus Rebate (%)
│   └── = Unit Price Before Tax
├── plus Taxes
│   └── = Total Unit Price
├── multiply by Quantity
├── plus Transaction Fee
├── plus SST on Transaction Fee
│   └── = Total Price (in local currency)
        └── Convert to PKR = Total Sales
```

---

## 6. Critical Discrepancies Found

### 🚨 DISCREPANCY #1: Hotel Service Currency Handling
**Location**: `service.controller.js` lines 2522-2536, 2610-2623
**Issue**: Hotel services with non-PKR currency store `total_costing` in ORIGINAL currency, not PKR
**Impact**: Inconsistent data storage - all other services store in PKR

### 🚨 DISCREPANCY #2: Cost Retrieval Recalculation
**Location**: `cost.controller.js` lines 125-147
**Issue**: Backend recalculates costs dynamically instead of using stored `total_costing`
**Impact**:
- Stored `total_costing` field is effectively ignored
- Potential for calculation differences if formulas change
- Performance impact from constant recalculation

### 🚨 DISCREPANCY #3: Inconsistent Total Cost Usage
**Location**: Various refund handlers vs cost controller
**Issue**: Refund handlers trust `total_costing`, cost controller doesn't
**Impact**: Different parts of system may show different cost values

### 🚨 DISCREPANCY #4: Missing Field Validation
**Issue**: No validation that calculated values match stored values
**Impact**: Data integrity cannot be guaranteed

### 🚨 DISCREPANCY #5: Amount Field Confusion
**Location**: Database schema
**Issue**: `amount` field in costs table appears unused (default 0.00) but still exists
**Impact**: Potential confusion about which field contains the actual cost

### 🚨 DISCREPANCY #6: Multiple Calculation Methods in Backend
**Location**: `psback/utils/transactionFeeCalculator.js` lines 243-265
**Issue**: Yet another cost calculation method exists with different formulas
**Details**:
- Basic mode: `(net_rate + taxes) * quantity` (missing extra_charges and SST!)
- Full mode: SST calculated on `(publishedRate - commission + taxes) * quantity` instead of just commission
**Impact**: Three different calculation methods in backend alone, each producing different results

---

## 7. Data Flow Diagram

```
Frontend Entry
    ↓
[User enters cost data in form]
    ↓
CostCalculationService.calculateCostValues()
    ↓
[Calculate all values including total_costing in PKR]
    ↓
Send to Backend API
    ↓
service.controller.js → updateService/createService
    ↓
[Special Case: Hotel?]
    ├─Yes→ [Convert total_costing back to original currency]
    └─No→  [Keep total_costing in PKR]
    ↓
Store in Database
    ↓
[Retrieval Paths]
    ├─cost.controller.js → [Recalculate, ignore stored total_costing]
    ├─Refund handlers → [Use stored total_costing with priority]
    └─Reports/Documents → [Variable usage]
```

---

## 8. Recommendations

### 8.1 Immediate Actions Required

1. **Standardize Currency Storage**
   - Decision needed: Store all `total_costing` in PKR or original currency
   - Currently inconsistent between hotel and other services

2. **Fix Cost Retrieval Logic**
   - Either trust stored `total_costing` OR always recalculate
   - Current mixed approach causes inconsistencies

3. **Add Validation Layer**
   ```javascript
   // Suggested validation on save
   const calculatedTotal = calculateTotalCost(costData);
   if (Math.abs(calculatedTotal - costData.total_costing) > 0.01) {
       log.warn('Total cost mismatch detected');
   }
   ```

4. **Database Cleanup**
   - Remove or document the unused `amount` field
   - Add comments to clarify field purposes

### 8.2 Long-term Improvements

1. **Single Source of Truth**
   - Move all calculation logic to a shared service
   - Ensure frontend and backend use identical formulas

2. **Audit Trail**
   - Log when recalculation differs from stored values
   - Track calculation formula versions

3. **Testing Suite**
   - Add comprehensive tests for cost calculations
   - Test currency conversion edge cases
   - Validate hotel service special handling

4. **Documentation**
   - Document why hotel services have special handling
   - Clear specification of which fields are authoritative

### 8.3 Data Integrity Checks

Run these queries to identify existing inconsistencies:

```sql
-- Find hotel services with potential currency issues
SELECT
    c.id,
    c.total_costing,
    c.currency,
    s.id as service_id,
    st.type
FROM costs c
JOIN services s ON c.service_id = s.id
JOIN service_types st ON s.service_type_id = st.id
WHERE st.type = 'Hotel'
  AND c.currency != '110'
  AND c.total_costing IS NOT NULL;

-- Compare stored vs calculated totals
SELECT
    c.id,
    c.published_rate,
    c.commission,
    c.quantity,
    c.total_costing as stored_total,
    ((c.published_rate - (c.published_rate * c.commission / 100)) * c.quantity) as calculated_base
FROM costs c
WHERE c.status != 'Void'
  AND ABS(c.total_costing - ((c.published_rate - (c.published_rate * c.commission / 100)) * c.quantity)) > 1;
```

---

## Conclusion

The PowerSuite cost calculation system has evolved with multiple calculation paths that have diverged over time. The most critical issues are:

1. **Hotel services storing costs in original currency while all others use PKR**
2. **Backend recalculating costs instead of trusting stored values**
3. **No validation between calculated and stored values**
4. **Three different calculation formulas in backend producing different results**

These discrepancies can lead to:
- Financial reporting errors
- Incorrect refund calculations
- User confusion when costs appear different in various screens
- Difficulty in auditing and troubleshooting

Immediate action is recommended to standardize the calculation and storage approach across all service types and system components.