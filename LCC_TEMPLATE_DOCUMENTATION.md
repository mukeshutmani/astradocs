# LCC Template Functionality Documentation

## Overview
The LCC (Low Cost Carrier) Template system in PowerSuite is a configurable data import mechanism designed to process flight booking data from various low-cost carriers in Excel format and convert it into standardized service records within the PowerSuite system. The system now features real-time progress tracking and improved error handling.

## Recent Updates (October 2025)
- **PNR + Customer Grouping**: Rows with the same PNR and customer code are automatically grouped into a single order
- **Real-time Progress Tracking**: Row-by-row processing with Server-Sent Events (SSE)
- **Improved Tax Handling**: Support for multiple tax columns with individual tracking
- **Enhanced Error Recovery**: Failed rows don't stop the entire import
- **Better Timeout Management**: Extended timeouts and streaming for large files
- **Progress Modal**: Visual feedback during import with detailed statistics

## Architecture Components

### 1. Database Structure

#### lcc_profile_templates Table
- **Purpose**: Stores template definitions for different airlines/carriers
- **Key Fields**:
  - `id`: Primary key
  - `label`: Template name (e.g., "AirSial", "IndiGo", "Air Blue", etc.)
  - `company_id`: Links template to a specific company (null for global templates)
  - `created_at`, `updated_at`: Timestamps

#### templates Table
- **Purpose**: Stores field mapping configurations for each template
- **Key Fields**:
  - `id`: Primary key
  - `template_id`: Foreign key to lcc_profile_templates
  - `document_column`: Column name in the Excel file
  - `service_column`: Target field in the service system
  - `mandatory`: Whether the field is required ('mandatory'/'optional')
  - `handling_method`: How to process the data ('direct_mapping'/'extraction')
  - `separator`: For splitting multi-value fields
  - `start_position`, `length`: For substring extraction
  - `description`: Field description
  - `row_identifier`: Row identification marker

### 2. Backend Components

#### Models
- **lcc_profile_template.js**: Sequelize model for template definitions
- **template.js**: Sequelize model for field mappings
- **Note**: Database column for supplier is `supp_no`, not `code`

#### Controller (lccImport.controller.js)
Core functions:
- `importLccData()`: Traditional batch import function
- `importLccDataSSE()`: **NEW** - Real-time progress import with SSE
- `processLccRow()`: Internal helper to process individual rows (not exported)
- `previewImport()`: Preview without saving
- `getTemplateWithMappings()`: Retrieve template configuration
- `createAirSialTemplate()`: Create airline-specific template

**Key Improvements**:
- Empty row filtering to improve performance
- Individual tax column processing
- Better error messages with row context
- Support for airline form numbers
- Enhanced date/time handling for flights

#### Routes (lccImport.route.js)
- `POST /lcc-import/import`: Traditional batch import
- `POST /lcc-import/import-sse`: **NEW** - SSE-based real-time import
- `POST /lcc-import/preview`: Preview import
- `GET /lcc-import/template/:id`: Get template details
- `POST /lcc-import/template/airsial/create`: Create AirSial template

#### Axios Configuration
- Default timeout: **120 seconds (2 minutes)** for regular requests
- Import timeout: **300 seconds (5 minutes)** for file uploads
- Auto-detection of import requests for extended timeout

### 3. Frontend Components

#### CreateTemplateWithUpload.jsx
- Main component for template configuration and import
- Located at route: `/iur/upload-lcc/:id`
- Features:
  - Template field mapping configuration
  - File upload interface
  - Real-time progress modal
  - SSE-based import option

#### ImportProgressModal.jsx
**NEW Component** - Real-time import progress display
- Uses shadcn/ui components (Dialog, Progress, Button)
- Features:
  - Progress bar with percentage
  - Row-by-row status updates
  - Success/error statistics
  - Detailed error list with row numbers
  - Status indicators (processing, completed, error)

#### LccImport.jsx (Legacy)
- Alternative import interface
- Not currently used in main routing

## Data Flow Process

### 1. Template Configuration Phase
1. Admin creates an LCC template with a label (airline name)
2. Defines field mappings between Excel columns and service fields
3. Specifies handling methods for each field
4. Sets mandatory/optional flags

### 2. Import Execution Phase

#### Option A: Traditional Batch Import
All rows processed in a single transaction (all-or-nothing approach)

#### Option B: SSE Real-time Import (Recommended)
Row-by-row processing with live updates and automatic PNR grouping

##### Step 1: File Upload & Validation
- User uploads Excel file (.xlsx/.xls)
- File size limit: 10MB
- MIME type validation
- Empty row filtering applied

##### Step 1.5: PNR + Customer Grouping (when createOrder is enabled)
- Rows are grouped by **PNR + Customer Code** before processing
- One order is created per unique PNR + Customer combination
- If the same PNR has 3 different customers, 3 separate orders are created
- All services with the same PNR and same customer are added to that order
- Grouping information sent via SSE event

##### Step 2: SSE Connection Setup
```javascript
// Frontend establishes SSE connection using dynamic base URL
const baseURL = import.meta.env.VITE_API_URL || '/api';
const response = await fetch(`${baseURL}/lcc-import/import-sse`, {
  method: 'POST',
  headers: { 'Authorization': `Bearer ${token}` },
  body: formData
});

// Backend SSE headers
res.writeHead(200, {
  'Content-Type': 'text/event-stream',
  'Cache-Control': 'no-cache',
  'Connection': 'keep-alive',
  'X-Accel-Buffering': 'no'
});
```

##### Step 3: Real-time Processing
For each row:
1. Send 'processing' event with row details
2. Process in individual transaction
3. Send 'row_complete' or 'row_error' event
4. Update statistics in real-time

##### Step 4: Field Mapping & Processing
- **Direct Mapping**: Copy value as-is
- **Extraction**: Apply separator/position rules
- **Tax Handling**: Process multiple tax columns separately

##### Step 5: Data Transformation
- Parse routes (e.g., "KHI-ISB" → segments)
- Convert numeric values (handle commas, spaces)
- Process multiple tax columns (OT, RG, YQ, SP, XZ, N9, XT, YI, PK, PB, YR)
- Lookup reference data:
  - Cities (create if missing) — global
  - Airlines — global
  - Suppliers (using `supp_no` column) — **scoped to the logged-in user's company** (via `supplier.user_id → user.company_code`)
  - Flight classes — global
  - Customers (using `customer_number`) — **scoped to the logged-in user's company** (via `customer.branch.company_code`)
  - Duplicate ticket check — **scoped to the logged-in user's company** (via `service.user_id → user.company_code`)
  - Order numbering — scoped to branches of the logged-in user's company
  - Invoice & Cost document numbering — scoped to branches of the logged-in user's company

##### Step 6: Service Creation
Creates interconnected records with enhanced data:

###### A. Main Service Record
```javascript
service = {
  service_type_id: 1, // Flight
  user_id: current_user,
  description: "LCC Import - route - PNR",
  pnr: extracted_pnr,
  supplier_reference: ticket_number,
  status: 'Booked',
  processed_iur: 0, // Pending PNR
  airline_form: airline_code, // Now stores actual form number
  supplier_id: looked_up_supplier,
  // ... other fields
}
```

###### B. Service Flights with Times
```javascript
service_flight = {
  service_id: service.id,
  city_from: lookup_city_id,
  city_to: lookup_city_id,
  airline_id: lookup_airline_id,
  departure_date: parsed_with_time,
  arrival_date: parsed_with_time,
  flight_number: actual_flight_number,
  flight_class: lookup_class_id
}
```

###### C. Enhanced Passengers
```javascript
passenger = {
  passenger_name: name,
  passenger_type: from_excel_or_default,
  ticket_number: ticket,
  seat_number: from_excel,
  order_id: optional_order
}
```

###### D. Individual Tax Records
```javascript
// Each tax column creates separate record
cost_tax = {
  cost_id: cost.id,
  tax_name: 'OT', // Actual column name
  tax_code: 'OT',
  tax_amount: individual_tax_value,
  is_percentage: false
}
```

## SSE Event Types

### Events Sent During Import

#### Import Lifecycle Events

1. **start**: Initial event with total row count
```json
{ "type": "start", "totalRows": 100, "message": "Starting import of 100 rows with automatic order creation" }
```

2. **grouping_complete**: PNR grouping completed (when createOrder is enabled)
```json
{ "type": "grouping_complete", "totalPnrs": 25, "totalRows": 100, "message": "Grouped 100 rows into 25 PNR groups" }
```

#### Order Events

3. **order_creation**: Order being created for PNR group
```json
{ "type": "order_creation", "currentPnr": "ABC123", "rowCount": 4, "message": "Creating order for PNR ABC123 (4 rows)..." }
```

4. **order_created**: Order successfully created for PNR group
```json
{ "type": "order_created", "orderId": 5678, "orderNumber": "HQSO00001234", "customerCode": "CUST001", "pnr": "ABC123", "rowCount": 4, "message": "Order HQSO00001234 created for PNR ABC123 with 4 services" }
```

5. **order_rollback**: Order rolled back due to no successful services
```json
{ "type": "order_rollback", "orderId": 5678, "message": "Order rolled back - no services were created successfully" }
```

#### Row Processing Events

6. **processing**: Row being processed
```json
{ "type": "processing", "currentRow": 5, "totalRows": 100, "pnr": "ABC123", "orderId": 5678, "message": "Processing row 5 (1/4 in PNR ABC123)" }
```

7. **row_complete**: Row successfully imported
```json
{ "type": "row_complete", "currentRow": 5, "totalRows": 100, "successCount": 5, "errorCount": 0, "ordersCreated": 2 }
```

8. **row_duplicate**: Duplicate ticket detected (same airline + ticket number, scoped to the logged-in user's company_code)
```json
{ "type": "row_duplicate", "currentRow": 5, "duplicate": { "row": 5, "pnr": "ABC123", "ticketNumber": "1234567890", "message": "Duplicate ticket" } }
```

9. **row_error**: Row failed to import (all rows in PNR group fail together)
```json
{ "type": "row_error", "currentRow": 5, "error": {"row": 5, "error": "Supplier not found", "pnr": "ABC123"} }
```

#### Document Generation Events

10. **generating_documents**: Starting invoice document generation for auto-invoice orders
```json
{ "type": "generating_documents", "message": "Generating invoice documents for 3 orders with auto-invoice enabled..." }
```

11. **documents_generated**: Invoice documents successfully created
```json
{ "type": "documents_generated", "message": "Generated 3 invoice documents...", "invoiceDocuments": [{ "orderId": 1, "orderNumber": "HQSO00001234", "documentNumber": "HQIN00000001", "invoiceCount": 4 }] }
```

12. **document_warning**: Invoice document generation failed for an order
```json
{ "type": "document_warning", "message": "Failed to generate document for order 123: error details" }
```

13. **generating_cost_documents**: Starting cost document generation for services with suppliers
```json
{ "type": "generating_cost_documents", "message": "Checking for services with suppliers to generate Cost documents..." }
```

14. **cost_documents_generated**: Cost documents successfully created
```json
{ "type": "cost_documents_generated", "message": "Generated 5 Cost documents for services with suppliers", "costDocuments": [{ "pnr": "ABC123", "supplierName": "Air Blue", "documentNumber": "HQCO00000001", "costCount": 2 }] }
```

15. **cost_document_warning**: Cost document generation failed
```json
{ "type": "cost_document_warning", "message": "Failed to generate Cost document for PNR ABC123: error details" }
```

#### Completion Events

16. **complete**: Import finished with full summary
```json
{ "type": "complete", "totalRows": 100, "successCount": 95, "duplicateCount": 3, "errorCount": 2, "errors": [...], "duplicates": [...], "services": [...], "ordersCreated": 25, "orders": [...], "invoiceDocuments": [...], "costDocuments": [...] }
```

17. **error**: Fatal error that stops the entire import
```json
{ "type": "error", "message": "Unexpected error description" }
```

## Progress Modal Features

### Visual Components
- **Progress Bar**: Shows percentage complete with animation
- **Statistics Grid**:
  - Rows Processed (current/total)
  - Successful imports (green)
  - Errors (red)
- **Status Messages**: Real-time updates on current operation
- **Error List**: Scrollable list of errors with row numbers

### User Experience
- Modal cannot be closed during processing
- Real-time updates without page refresh
- Clear error messages with context
- Final summary with actionable results

## Error Handling

### Row-Level Errors (SSE Mode)
- Each row processed independently
- Failed rows don't affect others
- Detailed error context provided
- Processing continues after errors

### Enhanced Error Messages
```javascript
{
  row: 5,
  error: "Supplier not found: SUPPNBSP0006. Please create the supplier first.",
  details: {
    pnr: "ABC123",
    ticket: "1234567890",
    passenger: "John Doe",
    route: "KHI-ISB"
  }
}
```

### Timeout Handling
- Default: 120 seconds (2 minutes) for regular requests
- Imports: 5 minutes timeout
- SSE: No timeout issues (streaming)
- Helpful messages for timeout scenarios

## Response Formats

### SSE Stream Response
Continuous stream of events as described above

### Traditional Batch Response
```javascript
{
  success: true,
  message: "Successfully imported X services as pending PNRs",
  template: "Template Name",
  totalRows: total_count,
  successCount: imported_count,
  errorCount: error_count,
  errors: [...], // Detailed error array
  services: [...]
}
```

## Pre-configured Templates

### Air Blue Template
Enhanced mappings with multiple tax columns:
- Customer No → customer_code (mandatory)
- Supplier No → supplier_code (optional)
- PAX_NAME → passenger_name (mandatory)
- Airline Form → airline_code (mandatory)
- Tkt No. → ticket_number (mandatory)
- PNR → pnr (mandatory)
- Sale Date → sale_date (optional)
- Sector → route (mandatory)
- Flight → flight_number (optional)
- DepDate → departure_date (optional)
- DepTime → departure_time (optional)
- ArvDate → arrival_date (optional)
- ArvTime → arrival_time (optional)
- Basic Fare → price (mandatory)
- OT → tax_amount (individual tax)
- RG → tax_amount (individual tax)
- YQ → tax_amount (individual tax)
- SP → tax_amount (individual tax)
- XZ → tax_amount (individual tax)
- N9 → tax_amount (individual tax)
- XT → tax_amount (individual tax)
- YI → tax_amount (individual tax)
- PK → tax_amount (individual tax)
- PB → tax_amount (individual tax)
- YR → tax_amount (individual tax)
- Seat No. → seat_number (optional)
- PAX_TYPE → passenger_type (optional)
- Class → flight_class (optional)

## Multi-Sector Flight Support

### Overview
The LCC import system fully supports multi-sector (connecting/multi-leg) flights with individual dates and times for each flight segment. When your Excel file contains routes like "KHI-ISB-LHE" (3 cities = 2 sectors), the system can handle specific departure/arrival dates and times for each sector.

### How Multi-Sector Works

#### Automatic Route Parsing
When you provide a route like "KHI-DXB-LON", the system automatically:
1. Parses it into segments: KHI→DXB (sector 1), DXB→LON (sector 2)
2. Creates separate `service_flight` records for each segment
3. Looks up or creates city records for each airport code
4. Assigns dates/times to each sector based on your configuration

#### Date/Time Assignment Logic

**For Sector 1 (First Leg)**:
- Uses primary columns: `departure_date`, `departure_time`, `arrival_date`, `arrival_time`
- Example: KHI→DXB uses DepDate, DepTime, ArvDate, ArvTime

**For Sector 2 (Second Leg)**:
- Looks for numbered columns: `departure_date2`, `departure_time2`, `arrival_date2`, `arrival_time2`
- Also supports alternate names: `DepDate2`, `DepTime2`, `ArrvDate2`, `ArrvTime2`
- Example: DXB→LON uses DepDate2, DepTime2, ArvDate2, ArrvTime2

**For Sector 3 (Third Leg)**:
- Uses: `departure_date3`, `departure_time3`, `arrival_date3`, `arrival_time3`
- Alternate: `DepDate3`, `DepTime3`, `ArrvDate3`, `ArrvTime3`

**Fallback Behavior** (if sector-specific dates not provided):
- System adds +1 day per sector to the first sector's dates
- Example: If first sector departs Jan 1, second sector defaults to Jan 2

### Excel Template Structure for Multi-Sector

#### Example 1: 2-Sector Route (KHI-ISB-LHE)

| PNR | Route | DepDate | DepTime | ArvDate | ArvTime | DepDate2 | DepTime2 | ArvDate2 | ArvTime2 |
|-----|-------|---------|---------|---------|---------|----------|----------|----------|----------|
| ABC123 | KHI-ISB-LHE | 2025-10-25 | 08:00 | 2025-10-25 | 09:30 | 2025-10-25 | 11:00 | 2025-10-25 | 12:15 |

**Result**:
- Sector 1: KHI→ISB departs 08:00, arrives 09:30
- Sector 2: ISB→LHE departs 11:00, arrives 12:15

#### Example 2: 3-Sector Route (KHI-DXB-LHR-CDG)

| PNR | Route | DepDate | DepTime | DepDate2 | DepTime2 | DepDate3 | DepTime3 |
|-----|-------|---------|---------|----------|----------|----------|----------|
| XYZ789 | KHI-DXB-LHR-CDG | 2025-10-25 | 02:00 | 2025-10-25 | 09:00 | 2025-10-26 | 14:00 |

**Result**:
- Sector 1: KHI→DXB departs Oct 25 02:00
- Sector 2: DXB→LHR departs Oct 25 09:00
- Sector 3: LHR→CDG departs Oct 26 14:00

### Template Configuration for Multi-Sector

#### Required Service Columns (Standard)
Always configure these for the first sector:
- `departure_date` - Primary departure date
- `departure_time` - Primary departure time
- `arrival_date` - Primary arrival date
- `arrival_time` - Primary arrival time
- `flight_number` - Primary flight number

#### Additional Service Columns (For Sector 2+)
Add these mappings to support multiple sectors:

```javascript
// For Sector 2
{
  document_column: 'DepDate2',
  service_column: 'departure_date2',
  mandatory: 'optional',
  handling_method: 'direct_mapping'
}
{
  document_column: 'DepTime2',
  service_column: 'departure_time2',
  mandatory: 'optional',
  handling_method: 'direct_mapping'
}
{
  document_column: 'ArvDate2',
  service_column: 'arrival_date2',
  mandatory: 'optional',
  handling_method: 'direct_mapping'
}
{
  document_column: 'ArvTime2',
  service_column: 'arrival_time2',
  mandatory: 'optional',
  handling_method: 'direct_mapping'
}
{
  document_column: 'FlightNo2',
  service_column: 'flight_number2',
  mandatory: 'optional',
  handling_method: 'direct_mapping'
}

// For Sector 3 (if needed)
// Use departure_date3, departure_time3, arrival_date3, arrival_time3, flight_number3
```

### Supported Service Column Names

The system supports the following mappable fields:

**Core Service Fields**:
- `pnr` - Booking reference
- `ticket_number` - Ticket number
- `sale_date` - Sale/issue date
- `status` - Booking status

**Passenger Information**:
- `passenger_name` - Passenger name(s)
- `passenger_type` - Passenger type (ADT/CHD/INF)
- `passport_number` - Passport number

**Flight Details - Sector 1 (Primary)**:
- `route` - Route/Sector (e.g., KHI-ISB-LHE)
- `flight_number` - Flight number
- `flight_class` - Flight class code
- `airline_code` - Airline code
- `departure_date`, `departure_time` - Departure date/time
- `arrival_date`, `arrival_time` - Arrival date/time
- `departure_city`, `arrival_city` - Departure/arrival city

**Flight Details - Sectors 2-5**:
- `departure_date2` through `departure_date5` (also accepts `DepDate2`, etc.)
- `departure_time2` through `departure_time5` (also accepts `DepTime2`, etc.)
- `arrival_date2` through `arrival_date5` (also accepts `ArrvDate2`, `ArvDate2`, etc.)
- `arrival_time2` through `arrival_time5` (also accepts `ArrvTime2`, `ArvTime2`, etc.)
- `flight_number2` through `flight_number5` (also accepts `FlightNo2`, etc.)

**Financial Fields**:
- `price` - Base fare/price
- `tax_amount` - Tax amount
- `total_amount` - Total amount
- `commission` - Commission (%)
- `discount` - Discount
- `extra_charges` - Extra charges
- `currency` - Currency code
- `sst` - Sales and Service Tax
- `markup` - Markup amount

**Reference Fields**:
- `customer_code` - Customer reference (REQUIRED). The customer must exist in the logged-in user's company (resolved via `customer.branch.company_code = user.company_code`). A customer with the same `customer_number` in another company will NOT match, and import will fail with "Customer not found in your company".
- `supplier_code` - Supplier reference
- `booking_reference` - External booking ref
- `invoice_number` - Invoice number

**Additional Fields**:
- `remarks` - Remarks/notes
- `baggage_allowance` - Baggage allowance
- `meal_type` - Meal type
- `seat_number` - Seat number
- `adult_count` - Number of adults
- `child_count` - Number of children
- `infant_count` - Number of infants

### Best Practices for Multi-Sector

1. **Always provide route**: The route field (e.g., "KHI-ISB-LHE") determines how many sectors to create

2. **Optional dates**: If you don't provide sector-specific dates, the system will estimate them by adding days

3. **Consistent data**: Ensure all rows with the same PNR have the same route structure

4. **Flight numbers**: Different flight numbers per sector help with tracking

5. **Time zones**: Be aware of time zone differences in your arrival/departure times

6. **Testing**: Use the preview function to verify multi-sector parsing before importing

### Database Records Created

For a 2-sector route, the system creates:
- **1 service record** (parent)
- **2 service_flight records** (one per sector)
  - Each has its own city_from, city_to, departure_date, arrival_date
- **N passenger records** (linked to service)
- **1 cost record** with breakdown
- **Multiple cost_tax records** (one per tax column)

### Example Template: Air Blue with Multi-Sector Support

```javascript
// Primary flight info
{ document_column: 'Sector', service_column: 'route', mandatory: 'mandatory' }
{ document_column: 'DepDate', service_column: 'departure_date', mandatory: 'optional' }
{ document_column: 'DepTime', service_column: 'departure_time', mandatory: 'optional' }
{ document_column: 'ArvDate', service_column: 'arrival_date', mandatory: 'optional' }
{ document_column: 'ArvTime', service_column: 'arrival_time', mandatory: 'optional' }
{ document_column: 'Flight', service_column: 'flight_number', mandatory: 'optional' }

// Sector 2 support
{ document_column: 'DepDate2', service_column: 'departure_date2', mandatory: 'optional' }
{ document_column: 'DepTime2', service_column: 'departure_time2', mandatory: 'optional' }
{ document_column: 'ArvDate2', service_column: 'arrival_date2', mandatory: 'optional' }
{ document_column: 'ArvTime2', service_column: 'arrival_time2', mandatory: 'optional' }
{ document_column: 'Flight2', service_column: 'flight_number2', mandatory: 'optional' }

// Sector 3 support (if needed)
{ document_column: 'DepDate3', service_column: 'departure_date3', mandatory: 'optional' }
{ document_column: 'DepTime3', service_column: 'departure_time3', mandatory: 'optional' }
{ document_column: 'ArvDate3', service_column: 'arrival_date3', mandatory: 'optional' }
{ document_column: 'ArvTime3', service_column: 'arrival_time3', mandatory: 'optional' }
{ document_column: 'Flight3', service_column: 'flight_number3', mandatory: 'optional' }
```

### Troubleshooting Multi-Sector Imports

**Issue**: Sectors getting wrong dates
- **Solution**: Check that your Excel column names exactly match the template's `document_column`
- **Tip**: Use preview mode to verify date assignment

**Issue**: System only creating one flight record for multi-sector route
- **Solution**: Verify route format uses hyphens (KHI-ISB-LHE) not spaces or commas
- **Check**: Ensure route parsing is detecting segments correctly

**Issue**: Second sector using same date as first
- **Solution**: Add explicit `DepDate2`, `DepTime2` columns to your template configuration

**Issue**: Dates in wrong format
- **Solution**: Excel dates should be in YYYY-MM-DD or DD/MM/YYYY format
- **Tip**: Times can be HH:MM or HH:MM:SS format

## Security & Permissions

### Template Access
- Templates can be company-specific or global
- Users can only access templates for their company
- Global templates (company_id = null) accessible to all

### Authentication
- JWT authentication required for all endpoints
- User company verified before operations
- Token passed in Authorization header for SSE

### File Security
- File type validation (Excel only)
- Size limit enforcement (10MB)
- Memory storage (no disk persistence)
- Empty row filtering for performance

## Best Practices

### Template Design
1. Map all critical fields as mandatory
2. Use extraction for complex fields
3. Map individual tax columns separately
4. Provide clear descriptions
5. Test with sample data first

### Import Process
1. Use SSE import for real-time feedback
2. Enable automatic order creation to group services by PNR
3. Preview before importing
4. Validate data quality in Excel
5. Rows with the same PNR but different customer codes will create separate orders
6. Remove empty rows from Excel
7. Ensure reference data exists (airlines, suppliers)

### Performance Considerations
- SSE mode processes rows grouped by PNR (no timeout issues)
- Empty rows automatically filtered during grouping
- One transaction per PNR group (all services in same PNR succeed/fail together)
- Efficient lookup caching
- Proper indexing on lookup tables
- 5-minute timeout for batch imports
- PNR grouping happens in-memory before processing starts

### Error Recovery
- In SSE mode with PNR grouping, failed PNR groups don't stop import
- All rows in a PNR group fail together (transactional consistency)
- Review error details in modal
- Fix issues and re-import failed PNR groups only
- Check for missing reference data (suppliers, airlines)
- Rows with the same PNR but different customer codes will be split into separate orders

## Integration Points

### With Order Management
- Optional order_id links services to orders
- Passenger matching within orders

### With Service System
- Creates service records as "pending PNR"
- Integrates with existing service workflows
- Compatible with IUR processing

### With Financial Systems
- Creates individual tax records for accounting
- Generates invoices for billing
- Supports multi-currency operations
- Tax breakdown preserved

## Troubleshooting Guide

# Common Issues and Solutions

1. **Timeout Errors**
   - Use SSE import mode (default)
   - Split large files into batches
   - Remove empty rows from Excel

2. **Supplier Not Found**
   - Check supplier exists with correct `supp_no` **in the logged-in user's company** (scoped via `supplier.user_id → user.company_code`)
   - A supplier with the same `supp_no` in another company will NOT match — create the supplier under the current company first
   - Verify supplier code spelling

3. **Missing Data**
   - Check template mappings
   - Verify column names match Excel
   - Ensure mandatory fields have data

4. **Tax Calculations**
   - All tax columns are summed automatically
   - Individual taxes preserved in database
   - Check for numeric format in Excel

## Future Enhancements Potential
1. WebSocket support for bidirectional communication
2. Batch size configuration for SSE processing
3. Pause/resume functionality for large imports
4. Export error rows to Excel for correction
5. Automated template detection from Excel structure
6. Validation rules engine
7. Import history tracking with rollback
8. API-based imports from external systems
9. Duplicate detection and handling
10. Multi-sheet Excel support