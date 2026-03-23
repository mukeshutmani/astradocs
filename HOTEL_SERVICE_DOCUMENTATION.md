`# PowerSuite Hotel Services Documentation

## Overview
Hotel services in PowerSuite are a core component of the travel booking system, allowing users to manage hotel bookings with comprehensive pricing, cost tracking, and rate breakdown capabilities.

### Recent Updates (January 2025)
- **Lump Sum Pricing**: Fixed persistence issues with lump sum vs night breakdown pricing modes
- **Pricing Type Synchronization**: Cost and Price components now maintain synchronized pricing types
- **Room Count Integration**: Number of rooms now properly factors into all calculations
- **Improved Data Flow**: Enhanced synchronization between AddHotel, HotelCost, and HotelPrice components

## Database Schema

### Main Tables

#### 1. `service_hotel` (Primary Hotel Service Table)
**Location:** `psback/models/service_hotel.js`

| Field | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Primary key, auto-increment |
| `service_id` | INTEGER | FK to services table |
| `city` | STRING | City location |
| `hotel_name` | STRING | Name of the hotel |
| `hotel_chain` | STRING | Hotel chain code |
| `check_in` | DATE | Check-in date |
| `check_out` | DATE | Check-out date |
| `same_day_check_out` | BOOLEAN | Same day checkout flag |
| `no_of_nights` | INTEGER | Number of nights |
| `no_of_rooms` | INTEGER | Number of rooms |
| `room_id` | INTEGER | FK to room_types table |
| `room_category_id` | INTEGER | FK to room_categories table |
| `guest_per_room` | INTEGER | Guests per room |
| `meals` | STRING | Meal plan details |
| `other_services` | STRING | Additional services |
| `remarks` | STRING | General remarks |
| `address` | STRING | Hotel address |
| `rate_description` | STRING | Rate description |
| `tel` | STRING | Telephone |
| `fax` | STRING | Fax number |
| `status` | INTEGER | FK to hotel_status.code |
| `itin_remarks` | STRING | Itinerary remarks |
| `refrence_code` | STRING | Reference code |
| `confirmation_no` | STRING | Confirmation number |
| `hotel_code` | STRING | Hotel code |
| `confirmation_date` | DATE | Confirmation date |
| `confirmation_by` | STRING | Confirmed by |
| `cancellation_policy` | STRING | Cancellation policy |

#### 2. `hotel_rate_breakdown` (Rate Breakdown Table)
**Location:** `psback/models/hotel_rate_breakdown.js`

| Field | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Primary key |
| `service_id` | INTEGER | FK to services table |
| `lumpsum` | BOOLEAN | Lump sum pricing flag (true = lump sum, false = night breakdown) |
| `markup` | DECIMAL(10,2) | Markup amount |
| `cost` | DECIMAL(10,2) | Cost amount (per room for lump sum, per night for breakdown) |
| `price` | DECIMAL(10,2) | Price amount (per room for lump sum, per night for breakdown) |
| `date` | DATE | Date for rate (NULL for lump sum, specific date for night breakdown) |

#### 3. `hotel_status` (Status Reference Table)
**Location:** `psback/models/system_models/hotel_status.js`

| Field | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Primary key |
| `label` | STRING | Status label |
| `code` | STRING | Status code |

#### 4. `hotel_chain_maintenance` (Hotel Chain Reference)
**Location:** `psback/models/system_models/hotel_chain_maintenance.js`

| Field | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Primary key |
| `hotel_chain_code` | STRING(10) | Chain code |
| `hotel_chain_name` | STRING(160) | Chain name |

## Backend Implementation

### Controllers

#### Service Controller (`psback/controllers/service.controller.js`)

**Key Functions:**

1. **Hotel Service Creation:**
   - Function: Part of `createService` (lines 869-897)
   - Creates `service_hotel` record with all hotel details
   - Handles room type and category associations
   - **Updated lump sum handling**: Checks both `hotelRate?.lumpsum` and `hotelRate?.isLumpSum` for backward compatibility
   - **Properly saves lumpsum field**: Sets `lumpsum: true/false` in hotel_rate_breakdown records
   - **Date handling**: Sets date to NULL for lump sum records

2. **Hotel Service Update:**
   - Function: Part of `updateService` (lines 1843-1878)
   - Updates existing hotel service or creates new one if not exists
   - Maintains referential integrity with related tables
   - **Consistent lump sum handling**: Uses same logic as creation for lumpsum field

3. **Hotel Rate Breakdown Management:**
   - `CreateHotelRetailBreakDown` (line 4124-4138)
   - `getAllHotelRateBreakDown` (line 4140-4152)
   - `getSingleHotelRateBreakDown` (line 4154-4168)
   - `UpdateHotelRateBreakDown` (line 4170-4191)
   - `DeleteHotelRateBreakDown` (line 4193-4208)

4. **Currency Handling for Hotels:**
   - Special logic for non-PKR currencies (lines 378-391, 621-634)
   - Converts between PKR and original currency using exchange rates
   - Maintains proper currency formatting

#### Data Controller (`psback/controllers/data.controller.js`)

**Hotel Chain Management:**
- `getHotelChains` (line 2887-2902) - List with pagination
- `createHotelChain` (line 2930-2937)
- `updateHotelChain` (line 2939-2956)
- `deleteHotelChain` (line 2958-2972)

**Hotel Status Management:**
- `getHotelStatus` (line 2974-2999) - List with search
- `createHotelSatus` (line 3001-3009)
- `updateHotelStatus`
- `deleteHoteStatus`

### API Routes

#### Service Routes (`psback/routes/service.route.js`)
```javascript
// Hotel Rate Breakdown endpoints
POST   /service/hotel-rate-breakdown
GET    /service/hotel-rate-breakdown
GET    /service/hotel-rate-breakdown/:id
PUT    /service/hotel-rate-breakdown/:id
DELETE /service/hotel-rate-breakdown/:id
```

#### Data Routes (`psback/routes/data.route.js`)
```javascript
// Hotel Chain Maintenance
GET    /data/hotelChainCodes
GET    /data/hotelChainCodes/:id
POST   /data/hotelChainCodes
PUT    /data/hotelChainCodes/:id
DELETE /data/hotelChainCodes/:id

// Hotel Status Maintenance
GET    /data/hotel-status
GET    /data/hotel-status/:id
POST   /data/hotel-status
PUT    /data/hotel-status/:id
DELETE /data/hotel-status/:id
```

## Frontend Implementation

### Components

#### 1. `AddHotel.jsx` (`psfront/src/components/AddHotel.jsx`)
**Purpose:** Main hotel details form component

**Key Features:**
- City selection with LiveComboBox
- Hotel name and chain selection
- Check-in/out date management with automatic night calculation
- **Number of rooms management** (propagates to cost and price calculations)
- Room type and category selection
- Guest per room configuration
- Meal plans and additional services
- Confirmation details tracking
- Status management

**State Management:**
- Manages comprehensive hotel details state
- Auto-calculates nights based on check-in/out dates
- Handles same-day checkout scenarios

#### 2. `HotelCost.jsx` (`psfront/src/components/service/HotelCost.jsx`)
**Purpose:** Hotel cost management component

**Key Features:**
- Supplier selection and management
- Currency handling with exchange rates
- Two pricing modes:
  - Night breakdown: Individual rate per night × number of rooms
  - Lump sum: Single amount per room × number of rooms
- **Room count synchronization** from hotel details
- **Pricing type selection** (controls both cost and price display)
- Commission calculation (percentage or manual)
- Tax management (WHT - Withholding Tax)
- Published rate and net rate calculations
- Cost locking based on document status
- **Visual calculation summary** showing room multiplication

**Business Rules:**
- Cost cannot be updated if status is "Printed", "Partially Paid", or "Paid"
- Automatic currency conversion for non-PKR currencies
- Commission can be automatic (percentage) or manual entry

#### 3. `HotelPrice.jsx` (`psfront/src/components/service/HotelPrice.jsx`)
**Purpose:** Hotel pricing and invoice management

**Key Features:**
- **Automatically synchronized** currency, pricing type, and room count from cost component
- Two pricing modes (auto-synced with cost):
  - Night breakdown with markup per night × number of rooms
  - Lump sum with total markup × number of rooms
- **Disabled pricing type selector** (follows cost component)
- **Room count display** (synced from cost)
- Markup calculation (amount or percentage)
- GST (tax) handling with inclusive/exclusive options
- Transaction fee management
- Tax addition interface
- Total price calculation
- **Visual calculation summary** showing room multiplication

**Business Rules:**
- Price cannot be updated if invoice status is "Printed", "Partially Settled", or "Settled"
- Automatic price calculation: base price + markup + taxes + transaction fee
- Currency conversion handling for display vs storage

#### 4. `HotelRateBreakDown.jsx` (`psfront/src/components/HotelRateBreakDown.jsx`)
**Purpose:** Detailed rate breakdown management

**Features:**
- Toggle between lump sum and night-by-night breakdown
- Individual cost/markup/price per night
- Automatic price calculation (cost + markup)
- Date assignment for each night rate

### API Services

#### Data API (`psfront/src/api/data.js`)

**Hotel Status Functions:**
- `createHotelStatus` (line 76)
- `getHotelStatus` (line 85)
- `singleHotelStatus` (line 100)
- `updateHotelStatus` (line 109)
- `deleteHotelStatus` (line 118)

**Hotel Chain Functions:**
- `getHotelChainCodes` (line 1567)
- `getHotelChainCodeById` (line 1586)
- `createHotelChainCode` (line 1595)
- `updateHotelChainCode` (line 1604)
- `deleteHotelChainCode` (line 1613)

**Hotel Rate Breakdown Functions:**
- `createHotelRateBreakDown` (line 1623)
- `getAllHotelRateBreakDown` (line 1632)
- `getSingleHotelRateBreakDown` (line 1641)
- `updateHotelRateBreakDown` (line 1649)
- `deleteHotelRateBreakDown` (line 1657)

## Component Synchronization and Data Flow

### Pricing Type Synchronization
1. **HotelCost Component** is the master controller for pricing type
2. When pricing type changes in HotelCost:
   - Updates `service.cost.pricing_type`
   - HotelPrice component automatically detects the change
   - HotelPrice updates its own pricing_type to match
3. **HotelPrice Component** has a disabled pricing type selector
   - Always follows the cost component's pricing type
   - Ensures consistency in calculations

### Room Count Synchronization
1. **Data Flow**: AddHotel → HotelCost → HotelPrice
2. **AddHotel Component**:
   - User enters number of rooms
   - Updates `hotelDetail.no_of_rooms`
3. **HotelCost Component**:
   - Watches `service.hotel.no_of_rooms` for changes
   - Updates its internal `number_of_rooms` state
   - Saves to `service.cost.number_of_rooms`
4. **HotelPrice Component**:
   - Watches `service.cost.number_of_rooms`
   - Automatically syncs when cost changes
   - Uses same room count in price calculations

### Calculation Formula Updates
**Night Breakdown Mode**:
- Cost: `(sum of all night rates) × number_of_rooms`
- Price: `((sum of all night rates) × number_of_rooms) + markup`

**Lump Sum Mode**:
- Cost: `lump_sum_amount × number_of_rooms`
- Price: `(lump_sum_amount × number_of_rooms) + markup`

## Hotel Service Workflow

### 1. Creating a Hotel Service

**Step 1: Service Type Selection**
- User selects "Hotel" or "Hotel Domestic" service type in AddService.jsx

**Step 2: Hotel Details Entry**
- AddHotel component renders
- User enters:
  - City (filtered for domestic if Hotel Domestic)
  - Hotel name and chain
  - Check-in/out dates
  - Room details (type, category, number)
  - Guest information
  - Additional services and meals

**Step 3: Cost Configuration (HotelCost)**
- Number of rooms automatically populated from hotel details
- Select supplier
- Choose currency and set exchange rate
- Configure pricing type:
  - Night breakdown: Enter rate per night (system multiplies by rooms)
  - Lump sum: Enter amount per room (system multiplies by rooms)
- Set commission (automatic % or manual)
- Add taxes if applicable
- View calculation summary showing room multiplication

**Step 4: Price Configuration (HotelPrice)**
- Number of rooms automatically synced from cost
- Currency automatically synced from cost
- Pricing type automatically synced from cost (selector is disabled)
- Set markup (amount or percentage)
- Configure GST if applicable
- Add transaction fees
- System calculates total price with room multiplication
- View calculation summary showing room multiplication

**Step 5: Service Creation**
- Frontend sends complete service data to backend
- Backend creates:
  - Service record
  - service_hotel record
  - Cost records
  - Price records
  - hotel_rate_breakdown records (if applicable)

### 2. Updating a Hotel Service

**Process:**
1. Load existing service data including hotel details
2. Populate all forms with existing data
3. User modifies required fields
4. System checks document status:
   - Cost locked if "Printed", "Partially Paid", or "Paid"
   - Price locked if invoice "Printed", "Partially Settled", or "Settled"
5. Update request sent to backend
6. Backend updates all related records in transaction

### 3. Special Features

**Currency Handling:**
- System stores amounts in original currency
- Frontend may display in PKR for consistency
- Automatic conversion using exchange rates
- Special handling for hotel services to maintain accuracy

**Rate Breakdown:**
- Allows detailed per-night pricing
- Useful for varying rates across stay period
- Can switch between breakdown and lump sum
- Maintains cost/markup/price relationship

**Status Tracking:**
- Hotel status (Confirmed, Pending, etc.)
- Confirmation details (number, date, by whom)
- Cancellation policy tracking
- Reference codes for supplier systems

## Key Business Rules

1. **Pricing Hierarchy:**
   - Cost forms the base
   - Markup added to cost creates base price
   - Taxes and fees added to create final price

2. **Document Locking:**
   - Costs locked when printed or paid
   - Prices locked when invoice printed or settled
   - Prevents accidental modifications to finalized bookings

3. **Currency Consistency:**
   - Cost and price must use same currency
   - Exchange rates tracked for conversion
   - PKR used as base currency for internal calculations

4. **Night Calculation:**
   - Automatic calculation based on check-in/out dates
   - Same-day checkout supported
   - Number of nights drives rate breakdown structure

5. **Commission Handling:**
   - Can be percentage-based or manual
   - Affects net rate calculations
   - Tracked separately from markup

## Integration Points

1. **With Order System:**
   - Hotels are services within orders
   - Share common service infrastructure
   - Integrated with passenger management

2. **With Invoice System:**
   - Price feeds into invoice generation
   - Status synchronization for locking
   - Tax calculations flow through

3. **With Supplier System:**
   - Supplier selection for cost tracking
   - Payment tracking through cost status
   - Reference code management

4. **With Reporting:**
   - Cost/price data for profitability reports
   - Booking statistics by hotel/chain
   - Commission tracking for supplier payments

## File Locations Summary

### Backend Files
- **Models:**
  - `psback/models/service_hotel.js` - Main hotel service model
  - `psback/models/hotel_rate_breakdown.js` - Rate breakdown model
  - `psback/models/system_models/hotel_status.js` - Status reference
  - `psback/models/system_models/hotel_chain_maintenance.js` - Chain reference

- **Controllers:**
  - `psback/controllers/service.controller.js` - Main service operations
  - `psback/controllers/data.controller.js` - Reference data management

- **Routes:**
  - `psback/routes/service.route.js` - Service endpoints
  - `psback/routes/data.route.js` - Reference data endpoints

### Frontend Files
- **Components:**
  - `psfront/src/components/AddHotel.jsx` - Hotel details form
  - `psfront/src/components/service/HotelCost.jsx` - Cost management
  - `psfront/src/components/service/HotelPrice.jsx` - Price management
  - `psfront/src/components/HotelRateBreakDown.jsx` - Rate breakdown

- **Pages:**
  - `psfront/src/pages/AddService.jsx` - Main service creation/edit page
  - `psfront/src/pages/HotelStatus/` - Status management pages
  - `psfront/src/pages/HotelChainMaintanance/` - Chain management pages

- **API Services:**
  - `psfront/src/api/data.js` - Reference data API calls
  - `psfront/src/api/booking.js` - Service creation API calls

## Troubleshooting Guide

### Common Issues and Solutions

1. **Pricing Type Reverts to Night Breakdown After Refresh**
   - **Cause**: Lumpsum field not being saved correctly to database
   - **Solution**: Ensure backend saves `lumpsum: true/false` in hotel_rate_breakdown
   - **Fixed in**: Backend controller now properly handles both `lumpsum` and `isLumpSum` fields

2. **Number of Rooms Not Syncing Between Cost and Price**
   - **Cause**: Missing synchronization logic or blocked useEffect conditions
   - **Solution**: HotelPrice now watches `service.cost.number_of_rooms` for changes
   - **Fixed in**: Added dedicated useEffect for room count synchronization

3. **Room Count Resets to 1 on Page Refresh**
   - **Cause**: Not checking saved cost/price data for number_of_rooms
   - **Solution**: Components now check saved data first before falling back to hotel details
   - **Fixed in**: Initialization logic updated to prioritize saved data

4. **Changes in Hotel Details Not Updating Cost/Price**
   - **Cause**: useEffect conditions preventing updates after initial load
   - **Solution**: Removed blocking conditions that prevented updates
   - **Fixed in**: Simplified useEffect logic to always update when source changes

5. **Calculation Totals Don't Match Expected Values**
   - **Cause**: Number of rooms not factored into calculations
   - **Solution**: Updated calculation functions to multiply by room count
   - **Display**: Added visual breakdown showing room multiplication in summary

## Maintenance Operations

### Adding New Hotel Chains
1. Navigate to Hotel Chain Maintenance page
2. Use `createHotelChainCode` API
3. Provide chain code and name
4. Available in hotel selection dropdown

### Managing Hotel Statuses
1. Access Hotel Status management
2. Create custom status codes
3. Map to business workflow stages
4. Used for tracking booking lifecycle

### Rate Breakdown Management
1. Can be managed per service
2. Supports both creation and updates
3. Allows historical rate tracking
4. Useful for audit and reporting

## Practical Implementation Findings

### Hotel Service Creation Flow (Order 546 Case Study)

#### 1. Prerequisites
- **Passenger Requirement**: At least one passenger must be added to the order before creating any service
- This is enforced at the UI level to ensure proper booking association

#### 2. Database Record Structure

When a hotel service is created through the UI, the following records are generated:

**services table (id: 1522):**
```sql
- order_id: 546
- service_type_id: 6 (Hotel International)
- price: 64500.00 (includes 10000 transaction fee)
- cost: 45500.00
- gross_profit: 9000.00
- passengers: "John Smith"
- created_date: 2025-01-31
```

**service_hotels table:**
```sql
- service_id: 1522
- city: "Dubai"
- hotel_name: "Marriott Downtown Dubai"
- check_in: 2025-02-05
- check_out: 2025-02-08
- no_of_nights: 3
- no_of_rooms: 1
- guest_per_room: 2
- status: 1 (Confirmed)
```

**hotel_rate_breakdowns table (3 records for 3 nights):**
```sql
Night 1: date: 2025-02-05, cost: 15000.00, price: 18000.00
Night 2: date: 2025-02-06, cost: 16000.00, price: 19000.00
Night 3: date: 2025-02-07, cost: 14500.00, price: 17500.00
```

**costs table:**
```sql
- service_id: 1522
- supplier_id: 109
- commission: 0.00
- net_rate: 45500.00
- payable: 45500.00
- currency: "PKR"
- exchange_rate: 1.00
```

#### 3. Key Observations

1. **Automatic Transaction Fee**: 
   - System automatically adds PKR 10,000 transaction fee
   - User enters subtotal (54,500), system calculates final (64,500)

2. **Table Name Variations**:
   - Model files use singular (service_hotel)
   - Database uses plural (service_hotels)
   - Important for direct SQL queries

3. **Date Handling**:
   - Night breakdown dates are automatically generated
   - Each night gets its own record starting from check-in date
   - Last record date is check-out minus one day

4. **Status Management**:
   - Default status is set to 1 (Confirmed) in the example
   - Status affects cost/price editability

5. **Currency Considerations**:
   - PKR services store values directly
   - Exchange rate is 1.00 for PKR
   - Non-PKR currencies would show different exchange rates

#### 4. UI-Database Mapping

**UI Input → Database Storage:**
- Hotel Details Form → service_hotels table
- Cost Night Breakdown → costs table + hotel_rate_breakdowns (cost column)
- Price Night Breakdown → services.price + hotel_rate_breakdowns (price column)
- Supplier Selection → costs.supplier_id
- Passenger Assignment → services.passengers (comma-separated names)

#### 5. Calculation Flow (Updated with Room Count)

**Example: 2 rooms, 3 nights**

1. **Cost Calculation (Night Breakdown)**:
   - Per night rates: Night 1: 15000, Night 2: 16000, Night 3: 14500
   - Subtotal (3 nights): 15000 + 16000 + 14500 = 45500
   - Total with rooms: 45500 × 2 rooms = 91000
   - Stored in services.cost and costs.net_rate

2. **Cost Calculation (Lump Sum)**:
   - Lump sum per room: 45500
   - Total with rooms: 45500 × 2 rooms = 91000
   - Stored in services.cost and costs.net_rate

3. **Price Calculation (Night Breakdown)**:
   - Per night rates: Night 1: 18000, Night 2: 19000, Night 3: 17500
   - Subtotal (3 nights): 18000 + 19000 + 17500 = 54500
   - Total with rooms: 54500 × 2 rooms = 109000
   - Plus markup and transaction fees as configured
   - Stored in services.price

4. **Price Calculation (Lump Sum)**:
   - Lump sum per room: 54500
   - Total with rooms: 54500 × 2 rooms = 109000
   - Plus markup and transaction fees as configured
   - Stored in services.price

5. **Gross Profit**:
   - Price minus cost minus transaction fee
   - Calculated after room multiplication
   - Stored in services.gross_profit

#### 6. Data Integrity Patterns

1. **Referential Integrity**:
   - service_hotels.service_id → services.id
   - hotel_rate_breakdowns.service_id → services.id
   - costs.service_id → services.id
   - All related records use the same service_id

2. **Data Consistency**:
   - Night breakdown totals match header records
   - Cost totals align between services and costs tables
   - Price calculations are consistent across UI and database

3. **Business Rule Enforcement**:
   - Passenger requirement validated before service creation
   - Status-based locking prevents unauthorized edits
   - Currency consistency maintained between cost and price

#### 7. Important Implementation Notes

1. **Night Breakdown State Management**:
   - Frontend maintains array of objects for breakdown
   - Each object has date, cost, and price properties
   - Lumpsum toggle properly persists to database
   - Frontend correctly loads lumpsum state from saved data

2. **Save Operation**:
   - Single API call creates all related records
   - Transaction ensures all-or-nothing creation
   - Service ID generated first, then used for related tables
   - `lumpsum` field properly saved as boolean in hotel_rate_breakdown
   - `number_of_rooms` saved in cost and price data for persistence

3. **Update Considerations**:
   - Updates check document status first
   - Locked records return error messages
   - Partial updates supported for non-locked fields
   - Room count changes propagate through all components

4. **Data Loading and Persistence**:
   - Frontend checks for saved `pricing_type` in cost/price data
   - Falls back to hotel_rate_breakdown.lumpsum field
   - Ensures pricing type persists across page refreshes
   - Room count persists in cost.number_of_rooms and price.number_of_rooms

5. **Component State Synchronization**:
   - HotelCost is the master for pricing_type and number_of_rooms
   - HotelPrice watches service.cost for changes
   - useEffect hooks ensure real-time synchronization
   - Proper dependency arrays prevent infinite loops