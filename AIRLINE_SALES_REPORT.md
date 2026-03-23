# Airline Sales Report - Technical Documentation

**Version**: 1.0
**Date**: March 2026
**Author**: System Analysis
**Status**: Stable - Verified

---

## Overview

The Airline Sales Report provides a per-passenger breakdown of airline ticket sales, costs, and profitability. Each row represents one passenger on one service, showing published rates, commissions, taxes, and profit margins.

---

## Technical Architecture

### Frontend Component

**File**: `psfront/src/pages/Report/AirlineSalesReport.jsx`

**Filters**:
- Date Range: Ticket Issue Date (=, <, <=, >, >=, <>, between)
- Branch: isNotBlank, isBlank, isEqual
- Airline: isNotBlank, isBlank, isEqual (via `service.airline_form`)
- Product Code: isNotBlank, isBlank, isEqual, in (multi-select)
- Ticket Number: isNotBlank, isBlank, isEqual
- Customer: isNotBlank, isBlank, isEqual, between
- Invoice Statuses: Multi-select array filter

**Output**: PDF or Excel

### Backend Controller

**File**: `psback/controllers/report.controller.js`
**Function**: `getAirlineSalesReport` (line ~8678)

### PDF Template

**File**: `psback/views/pages/reports/airline_sales.ejs`

---

## Report Layout

### Row Level

One row per **passenger** per **service**. If a service has 2 passengers, it generates 2 rows with identical financial values.

### Columns

| Column | Source | Description |
|--------|--------|-------------|
| Product Code | `service_type.product_code` | Air product code (12, 13, 14, 15) |
| Ticket Issue Date | `service.ticket_issue_date` | Date ticket was issued |
| Customer No. | `customer.customer_number` | Customer code |
| Customer Name | `customer.customer_name` | Customer name |
| Airline | `airline_code.airline_name` | Airline name (via `service.airline_form`) |
| Ticket Number | `service_passenger.ticket_number` | Passenger ticket number |
| Routing | `service_flights` → `city_code.code` | Flight routing (e.g., KHI / DXB / DXB / KHI) |
| First Departure | `service_flights[0].departure_date` | First flight departure date |
| Invoice Number | `invoice.invoice_number` | Invoice document number |
| Invoice Date | `invoice.createdAt` | Invoice creation date (NOT invoice_date) |
| Publish | `cost.published_rate` | Published fare per passenger |
| Extra Charges | `cost.free_of_cost` | Extra charges from cost |
| Commission | `(published_rate * commission%) / 100` | Commission amount |
| WHT | `(commission_amount * cost.sst%) / 100` | Withholding tax on commission |
| T-Cost | `net_rate + extra + cost_taxes + wht` | Total cost per passenger |
| Cost Tax | `sum(cost_taxes.tax_amount)` | Sum of cost tax amounts |
| Sales | `(total_price - sst_on_tfee) / num_passengers` | Sales per passenger |
| Sales Tax | `sum(invoice_taxes.tax_amount)` | Sum of invoice tax amounts |
| T-Fee | `transaction_fee / num_passengers` | Transaction fee per passenger |
| SST | `sst_on_transaction_fee / num_passengers` | SST on transaction fee per passenger |
| Net | `sales + sst_per_passenger` | Net amount per passenger |
| Profit | `sales - t_cost` | Profit per passenger |
| M% | `(profit / sales) * 100` | Margin percentage |
| TCID | `order.tcid.name` | Team/TCID name |

---

## Calculation Logic

### Sales Calculation

```
sstOnTransactionFee = (invoice.transaction_fee * invoice.sst%) / 100
totalSales = invoice.total_price - sstOnTransactionFee
salesPerPassenger = totalSales / numPassengers
```

### Cost Calculation

```
commissionAmount = (cost.published_rate * cost.commission%) / 100
netRate = cost.net_rate
  // If netRate === publishedRate AND commission > 0:
  //   netRate = publishedRate - commissionAmount
extraCharges = cost.free_of_cost
costTaxes = sum(cost_taxes.tax_amount)
whtAmount = (commissionAmount * cost.sst%) / 100

costPerPassenger = netRate + extraCharges + costTaxes + whtAmount
```

### Profit Calculation

```
profitPerPassenger = salesPerPassenger - costPerPassenger
marginPerPassenger = (profitPerPassenger / salesPerPassenger) * 100
```

### Net Calculation

```
netPerPassenger = salesPerPassenger + (sstOnTransactionFee / numPassengers)
```

---

## Data Filtering

### Service Inclusion Criteria

A service is included in the report only if ALL of these conditions are met:
1. `service_type.type = 'Air'`
2. `invoice.invoice_number` exists (non-null) - excludes Raised invoices without numbers
3. `service_passengers.length > 0` - has at least one passenger
4. `cost` exists - has at least one non-Void cost record

### Cost Filtering

- Costs with `status = 'Void'` are excluded from the query
- Only the first cost record (`service.Costs[0]`) is used per service

### Invoice Filtering

- Optional `invoice_statuses` array filter can restrict which invoice statuses are included
- Invoices without `invoice_number` are automatically excluded by the inclusion criteria

---

## Company Scoping

The report is scoped to the user's company via:
```
order → user → where: { company_code: req.user.company_code }
```

---

## Report Metadata

```javascript
{
  report_number: "ASR" + timestamp,
  report_type: "airline-sales-report",
  file_type: "xlsx" | "pdf"
}
```

---

## Verification Results (March 2026)

Verified with 7 rows across 2 customers (RM0011, QKCOMP), 4 services, 2 airlines (EMIRATES, Air Sial). All 12 numeric columns matched database calculations exactly. No bugs found.

---

## Code References

| File | Purpose |
|------|---------|
| `psfront/src/pages/Report/AirlineSalesReport.jsx` | Frontend UI |
| `psback/controllers/report.controller.js` | Controller logic (~line 8678) |
| `psback/views/pages/reports/airline_sales.ejs` | PDF template |
| `psfront/src/api/report.js` | API client |

---

**Document End**
