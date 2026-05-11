# Hotel Booking Report

## Overview
The Hotel Booking Report provides a detailed listing of all hotel bookings, grouped by hotel name. It supports PDF and Excel output formats.

## API Endpoint
- **Route**: `POST /api/report/getHotelBookingReport`
- **Authentication**: Required (JWT)
- **Permission**: `Hotel-Booking-Report`

## Request Body Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| orderDateFilter | string | Date filter operator: `=`, `<`, `<=`, `>`, `>=`, `<>`, `between` |
| orderStartDate | date | Start date for order date filter (order.created_at) |
| orderEndDate | date | End date (required when filter is `between`) |
| checkInDateFilter | string | Date filter operator for check-in date: `=`, `<`, `<=`, `>`, `>=`, `<>`, `between` |
| checkInStartDate | date | Start date for check-in date filter (service_hotel.check_in) |
| checkInEndDate | date | End date for check-in date filter (required when filter is `between`) |
| checkOutDateFilter | string | Date filter operator for check-out date: `=`, `<`, `<=`, `>`, `>=`, `<>`, `between` |
| checkOutStartDate | date | Start date for check-out date filter (service_hotel.check_out) |
| checkOutEndDate | date | End date for check-out date filter (required when filter is `between`) |
| hotelNameFilter | string | `isEqual` to filter by specific hotel name |
| hotel_name | string | Hotel name value when hotelNameFilter is `isEqual` |
| customerFilter | string | `isEqual`, `isNotBlank`, or `isBlank` |
| customer_id | integer | Customer ID when customerFilter is `isEqual` |
| supplierFilter | string | `isEqual`, `isNotBlank`, or `isBlank` |
| supplier_id | integer | Supplier ID when supplierFilter is `isEqual` |
| type | string | Output type: `pdf` (default) or `excel` |

## Data Source
- Queries `service` table where `service_type_id = 3` (Hotel)
- Includes: `service_hotel`, `order` (with `branch` and `customer`), `supplier`, `invoice`, `cost` (with `cost_tax` and `document`), `user` (with `company` filtered by user's company_code)


## Report Columns

| Column | Source | Description |
|--------|--------|-------------|
| booking_no | order.order_number | Booking/order number |
| customer_name | order.customer.customer_name | Customer name |
| supplier_name | supplier.supplier_name | Supplier name |
| checkin_date | service_hotel.check_in | Check-in date (DD-MM-YYYY) |
| checkout_date | service_hotel.check_out | Check-out date (DD-MM-YYYY) |
| invoice_no | invoice.invoice_number | Invoice number |
| inv_status | invoice.status | Invoice status (Raised, Printed, Void, Settled, Partially Settled) |
| xo_no | cost.Document.document_number | XO (costing) document number |
| xo_status | cost.status | XO/Cost status (Raised, Printed, Void, Paid, Partially Paid) |
| no_of_rooms | service_hotel.no_of_rooms | Number of rooms |
| no_of_nights | service_hotel.no_of_nights | Number of nights |
| total_room_nights | Calculated | no_of_rooms * no_of_nights |
| commission | Calculated | published_rate * commission% / 100 |
| sales_gst_incl | invoice.total_amount | Sales amount GST inclusive |
| cost_gst_incl | Calculated | published_rate - commission + cost_taxes + WHT |
| profit | Calculated | sales_gst_incl - cost_gst_incl |
| currency | cost.currency_code.currency_code | Foreign currency code (e.g., AUD, USD). Empty if PKR |
| used_rate | cost.exchange_rate | Exchange rate stored at booking time (not live rate). Empty if PKR |

## Cost Calculation
- **Commission**: `published_rate * commission_percent / 100`
- **Nett Rate**: `published_rate - commission`
- **Regular Tax**: Sum of all `cost_tax.tax_amount`
- **WHT**: `commission * sst_percent / 100`
- **Total Cost (GST Incl)**: `nett_rate + regular_tax + WHT`

## Void Handling
- Each service can have multiple invoices and multiple costs (XOs).
- **Void invoices and void costs are completely excluded** — they are filtered out before display.
- The first non-void invoice and first non-void cost are used for each service row.
- If all invoices are void but a valid cost exists: row shows with empty invoice fields.
- If all costs are void but a valid invoice exists: row shows with empty cost fields.
- If both all invoices and all costs are void: the service row is skipped entirely.

## Grouping
Data is sorted by city then hotel name. Grouped by hotel name only (city heading removed).

Each group includes:
- A hotel heading row: `Hotel: <hotel_name>` (displayed in semi-bold/font-weight 600)
- Individual booking rows
- A subtotal row after each hotel group with summed numeric fields

## Column Widths
All columns have explicit widths: Booking No (7%), Customer Name (10%), Supplier Name (10%), Checkin Date (5%), Checkout Date (5%), Invoice No (6%), Inv Status (4%), Xo No (6%), XO Status (4%), No Of Rooms (4%), No Of Nights (4%), Total Room Nights (4%), Commission (5%), Sales Gst Incl (6%), Cost Gst Incl (6%), Profit (6%), Currency (3%), Used Rate (4%). Configured via `columnWidths` passed to the report template.

## Output
- **PDF**: Generated via `report1.ejs` template and `createPdf` (A4 Landscape), uploaded to MinIO. Print font size: 9px for data, 12px for subtitle, 14px for title.
- **Excel**: Generated via ExcelJS with full formatting matching PDF layout — company name, report title, report ID, print info, filters in header section; column headers with gray background; data rows with thin borders; hotel group heading rows (merged, bold, light gray fill); subtotal rows (bold); grand total row; empty separator rows between hotel sections. Numeric columns use `#,##0.00` format. Header rows are frozen.
- **Report Record**: Stored in `report` table with type `hotel-booking-report` and number prefix `THBR`

## Files
- **Controller**: `psback/controllers/report.controller.js` (`exports.getHotelBookingReport`)
- **Route**: `psback/routes/report.route.js`
- **Template**: `psback/views/pages/reports/report1.ejs`
