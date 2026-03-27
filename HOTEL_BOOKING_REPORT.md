# Hotel Booking Report

## Overview
The Hotel Booking Report provides a detailed listing of all hotel bookings, grouped by city and hotel name. It supports PDF and Excel output formats.

## API Endpoint
- **Route**: `POST /api/report/getHotelBookingReport`
- **Authentication**: Required (JWT)
- **Permission**: `Hotel-Booking-Report`

## Request Body Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| checkinDateFilter | string | Date filter operator: `=`, `<`, `<=`, `>`, `>=`, `<>`, `between` |
| checkinStartDate | date | Start date for check-in filter |
| checkinEndDate | date | End date (required when filter is `between`) |
| hotelNameFilter | string | `isEqual` to filter by specific hotel name |
| hotel_name | string | Hotel name value when hotelNameFilter is `isEqual` |
| customerFilter | string | `isEqual`, `isNotBlank`, or `isBlank` |
| customer_id | integer | Customer ID when customerFilter is `isEqual` |
| supplierFilter | string | `isEqual`, `isNotBlank`, or `isBlank` |
| supplier_id | integer | Supplier ID when supplierFilter is `isEqual` |
| type | string | Output type: `pdf` (default) or `excel` |

## Data Source
- Queries `service` table where `service_type_id = 3` (Hotel)
- Includes: `service_hotel`, `order` (with `branch` and `customer`), `supplier`, `invoice`, `cost` (with `cost_tax`), `user` (with `company` filtered by user's company_code)

## Report Columns

| Column | Source | Description |
|--------|--------|-------------|
| booking_no | order.order_number | Booking/order number |
| customer_name | order.customer.customer_name | Customer name |
| hotel_code | service_hotel.hotel_code | Hotel code |
| charge_type | Hardcoded | Always "Chargeable" |
| supplier_name | supplier.supplier_name | Supplier name |
| checkin_date | service_hotel.check_in | Check-in date (DD-MM-YYYY) |
| checkout_date | service_hotel.check_out | Check-out date (DD-MM-YYYY) |
| invoice_no | invoice.invoice_number | Invoice number |
| no_of_rooms | service_hotel.no_of_rooms | Number of rooms |
| no_of_nights | service_hotel.no_of_nights | Number of nights |
| total_room_nights | Calculated | no_of_rooms * no_of_nights |
| commission | Calculated | published_rate * commission% / 100 |
| sales_gst_incl | invoice.total_amount | Sales amount GST inclusive |
| cost_gst_incl | Calculated | published_rate - commission + cost_taxes + WHT |

## Cost Calculation
- **Commission**: `published_rate * commission_percent / 100`
- **Nett Rate**: `published_rate - commission`
- **Regular Tax**: Sum of all `cost_tax.tax_amount`
- **WHT**: `commission * sst_percent / 100`
- **Total Cost (GST Incl)**: `nett_rate + regular_tax + WHT`

## Grouping
Data is grouped hierarchically:
1. **City** - From `service_hotel.city`
2. **Hotel Name** - From `service_hotel.hotel_name`

Each group includes:
- A city heading row: `City: <city_name>`
- A hotel heading row: `Hotel: <hotel_name>`
- Individual booking rows
- A subtotal row after each hotel group with summed numeric fields

## Output
- **PDF**: Generated via `report1.ejs` template and `createPdf`, uploaded to MinIO
- **Excel**: Generated via `createExcel` with auto-transformed column headers, uploaded to MinIO
- **Report Record**: Stored in `report` table with type `hotel-booking-report` and number prefix `THBR`

## Files
- **Controller**: `psback/controllers/report.controller.js` (`exports.getHotelBookingReport`)
- **Route**: `psback/routes/report.route.js`
- **Template**: `psback/views/pages/reports/report1.ejs`
