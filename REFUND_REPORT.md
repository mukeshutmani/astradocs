# Refund Report - Technical Documentation

**Version**: 1.0
**Date**: March 2026
**Author**: System Analysis
**Status**: Stable - Verified

---

## Overview

The Refund Report lists all printed refund records with their associated service details, customer/supplier information, flight routing, credit note references, currency details, and refund amounts (both customer and supplier side). One row per refund.

---

## Technical Architecture

### Frontend Component

**File**: `psfront/src/pages/Report/RefundReport.jsx`

**Filters**:
- Refund Date: =, <, <=, >, >=, <>, between (on `refund.created_at`)
- Branch: isNotBlank, isBlank, isEqual
- Service Type: isNotBlank, isBlank, isEqual
- Customer: isNotBlank, isBlank, isEqual, between

**Output**: PDF or Excel

**Filter retention & inline preview**:
- Selected filters are retained after generating (module-level `cachedFilter`); returning to the page keeps prior selections. A full page refresh resets to `DEFAULT_FILTER`.
- PDF is rendered inline below the filters (iframe → `{VITE_API_URL}/document/view/{report_number}`) with a **Preview** button (opens the full PDF in a new tab). No navigation away from the page. Excel still downloads and clears the inline PDF.

### Backend Controller

**File**: `psback/controllers/report.controller.js`
**Function**: `getRefundReport` (line 5689)
**Route**: `POST /api/report/getRefundReport`
**Permission**: `Refund-Report`

### PDF Template

**File**: `psback/views/pages/reports/report2.ejs` (shared generic report template)
**Format**: A3 Landscape

---

## Report Columns

| # | Column | Key | Source | Description |
|---|--------|-----|--------|-------------|
| 1 | Refund Date | `refund_date` | `refund.created_at` | Formatted DD MMM YYYY |
| 2 | Refund No. | `refund_no` | `refund.refund_no` | Refund document number |
| 3 | Status | `status` | `refund.status` | Always "Printed" (filter) |
| 4 | Refund Type | `refund_type` | `service_type.type[0]` | First character of service type (A=Air, H=Hotel, etc.) |
| 5 | Airline Form | `airline_form` | `service.airline_form` | Airline form number |
| 6 | Number | `number` | Varies by service type | Service-specific number (see below) |
| 7 | Pax Name | `pax_name` | `refund.passenger_name` | Passenger name(s) |
| 8 | Debtor No. | `customer_number` | `customer.customer_number` | Customer code |
| 9 | Customer Name | `customer_name` | `customer.customer_name` | Customer name |
| 10 | Booking Order No. | `booking_order_number` | `refund.order_no` | Order number |
| 11 | Invoice No. | `invoice_number` | `refund.invoice_no` | Invoice number |
| 12 | Product Code | `product_code` | `service_type.product_code` | Service type product code |
| 13 | Original Segment | `original_segment` | `service_flights` → city codes | Flight routing (e.g., KHI-DXB, DXB-KHI) |
| 14 | Refunded Segment | `refunded_segment` | `refund.refund` | Refunded route segment |
| 15 | Document No(Customer) | `document_number` | `credit_note.reference` | Credit note reference (looked up by `client_refrence = refund_no`) |
| 16 | Document Date | `document_date` | `credit_note.created_at` | Credit note creation date |
| 17 | Currency Code | `currency_code` | `currency_code_refund.currency_code` | Currency code (e.g., PKR) |
| 18 | Exchange Rate | `exchange_rate` | `refund.exchange_rate` | Exchange rate (5 decimal places) |
| 19 | Supplier No. | `supplier_number` | `supplier.supp_no` | Supplier code |
| 20 | Document No. (Supplier) | `document_number_supplier` | `refund.cost_document_number` or `cost.supplier_reference` | Supplier document (XO number) |
| 21 | Document Date (Supplier) | `document_date_supplier` | `cost.created_at` | Cost creation date |
| 22 | Voucher No. | `voucher_no` | `refund.voucher_no` | Voucher number(s) from JSON array |
| 23 | Supplier Currency | `supplier_currency` | `currency_code_refund.currency_code` | Supplier currency |
| 24 | Customer Airline Charges | `customer_airline_charges` | `refund.customer_airline_charges` | Customer airline charges |
| 25 | Customer Refund Charges | `customer_refund_charges` | `refund.customer_refund_charges` | Customer refund charges |
| 26 | Supplier Airline Charges | `supplier_airline_charges` | `refund.supplier_airline_charges` | Supplier airline charges |
| 27 | Supplier Refund Charges | `supplier_refund_charges` | `refund.supplier_refund_charges` | Supplier refund charges |
| 28 | Supplier Amount | `supplier_amount` | `refund.supplier_refund_amount` | Supplier refund amount |
| 29 | Customer Amount | `customer_amount` | `refund.customer_refund_amount` | Customer refund amount |

---

## Service Number Logic

The "Number" column value depends on service type:

| Service Type | Source Field |
|-------------|-------------|
| Flight / Air | `service_flights[0].ticket_number` |
| Hotel | `service_hotel.confirmation_no` |
| Car Transfer | `service_car_transfer.conf_code` |
| Rental Car | `service_rental_car.conf_code` |
| Visa | `service_visas[0].visa_code` or `passport_number` |
| Insurance | `service_insurance.policy_number` |
| Tour | `service_tour.package_code` |
| Train / Rail | `service_trains[0].company` |
| Miscellaneous | `service_miscellaneous.description` |

---

## Original Segment (Routing) Construction

Built from `service_flights` for flight services:
```
segments = service_flights
  .filter(f => f.city_from_code.code && f.city_to_code.code)
  .map(f => `${city_from_code.code}-${city_to_code.code}`)
routing = segments.join(", ")
```

Example: `KHI-DXB, DXB-KHI`

---

## Credit Note Lookup

The Document No(Customer) and Document Date columns are populated by looking up a credit note:
```javascript
credit_note = db.credit_note.findOne({ where: { client_refrence: refund.refund_no } })
document_number = credit_note.reference   // e.g., TTCN00000001
document_date = credit_note.created_at    // formatted DD MMM YYYY
```

---

## Grand Total Row

Sums the following numeric columns across all rows:
- Customer Airline Charges
- Customer Refund Charges
- Supplier Airline Charges
- Supplier Refund Charges
- Supplier Amount
- Customer Amount

**Currency Conversion**: All amounts (both individual rows and Grand Total) are converted to PKR using the refund's stored `exchange_rate`. The Currency Code and Exchange Rate columns show the original currency for reference.

All other columns are empty except `refund_no` which shows "Total".

---

## Data Sources

### Primary Tables

| Table | Purpose |
|-------|---------|
| `refunds` | Main refund records |
| `services` | Service details (airline_form, supplier_id, order_id) |
| `service_types` | Service type name and product code |
| `orders` | Links to customer and branch |
| `customers` | Customer info (customer_number, customer_name) |
| `branches` | Company scoping |
| `suppliers` | Supplier info (supp_no) |
| `service_flights` | Flight segments with city codes |
| `city_codes` | Airport/city codes (KHI, DXB, etc.) |
| `costs` | Cost document (supplier_reference, created_at) |
| `credit_notes` | Credit note lookup (by client_refrence = refund_no) |
| `currency_codes` | Currency code lookup |
| `service_hotel` | Hotel service details |
| `service_car_transfer` | Car transfer details |
| `service_visa` | Visa details |
| `service_insurance` | Insurance details |
| `service_tour` | Tour details |
| `service_train` | Train details |
| `service_rental_car` | Rental car details |
| `service_miscellaneous` | Miscellaneous service details |

### Query Joins

```
refunds
  → currency_codes (as currency_code_refund)
  → costs (cost_id)
  → services (service_id)
    → suppliers (supplier_id)
    → service_types (service_type_id)
    → orders (order_id)
      → customers (customer_id) [with customerWhere filter]
      → branches (branch_id) [with company_code filter]
    → service_flights → city_codes (city_from, city_to)
    → service_hotel
    → service_car_transfer
    → service_visa
    → service_insurance
    → service_tour
    → service_train
    → service_rental_car
    → service_miscellaneous
```

### Query Filters

- `refund.status = 'Printed'` (only printed refunds)
- Date filter on `refund.created_at`
- Company scoped via `order → branch → company_code`
- Optional: customer filter, branch filter, service type filter

---

## Company Scoping

Scoped via `refund → service → order → branch → where: { company_code: req.user.company_code }`.

---

## Report Metadata

```javascript
{
  report_number: "TPRR" + timestamp,
  report_type: "refund-report",
  file_type: "xlsx" | "pdf"
}
```

---

## Output Formats

### PDF
- Template: `report2.ejs` (A3 Landscape)
- Filter display header showing date range, customer, branch
- Data table with Grand Total row

### Excel
- Uses `createExcel` + `autoTransformForExcel` helper functions
- Worksheet: "Refund Report"
- Column headers from `key` property of each data field
- Numeric columns formatted with commas and 2 decimal places
- Grand Total row appended

---

## Number Formatting

Numeric columns are formatted with commas and 2 decimal places:
```javascript
// Fields formatted: customer_amount, supplier_amount,
//   customer_airline_charges, customer_refund_charges,
//   supplier_airline_charges, supplier_refund_charges
parseFloat(value).toFixed(2).replace(/\B(?=(\d{3})+(?!\d))/g, ",")
// Example: 34692.00 → "34,692.00"
```

---

## Verification Results (March 2026)

Verified with refund TTRF00000001 (date range: 20 Feb 2026 - 01 Mar 2026):
- All 27 columns verified correct against database
- Refund Date: 26 Feb 2026 ✓
- Pax: AZFAR KARIM CHD. ✓
- Customer: QKCOMP / Q & K Company PVT Ltd. ✓
- Original Segment: KHI-DXB, DXB-KHI ✓
- Refunded Segment: KHI-DXB, DXB-KHI ✓
- Document No(Customer): TTCN00000001 (credit note lookup) ✓
- Supplier: Q K SUPPLIER ✓
- Document No(Supplier): TTXO00000002 ✓
- Supplier Amount: 34,692.00 ✓
- Customer Amount: 36,050.00 ✓
- Grand Total matches ✓
- Company scoping correctly excludes other company refunds (ASRF*) ✓

### Bug Fix: Foreign Currency Grand Total (March 18, 2026)
- **Issue**: Foreign currency amounts (e.g., AUD 411.48) were displayed raw without PKR conversion, mixing currencies in both rows and Grand Total
- **Fix**: All numeric amounts are now converted to PKR at the row level using `refund.exchange_rate`. The display values themselves are in PKR, so Grand Total sums correctly
- **Example**: TTRF00000011 in AUD (rate 196): Supplier 411.48 AUD → 80,650.08 PKR, Customer 408.10 AUD → 79,987.60 PKR
- **Affects**: Both PDF and Excel output (individual rows and Grand Total)

---

## Code References

| File | Purpose |
|------|---------|
| `psfront/src/pages/Report/RefundReport.jsx` | Frontend UI |
| `psback/controllers/report.controller.js` (line 5689) | Controller logic |
| `psback/views/pages/reports/report2.ejs` | PDF template (shared) |
| `psback/routes/report.route.js` (line 53) | Route definition |
| `psfront/src/api/report.js` | API client |

---

**Document End**
