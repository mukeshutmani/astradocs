# Umrah Service Documentation

## Overview

The Umrah Service module manages Umrah pilgrimage package bookings within the PowerSuite travel system. It supports complete package management including flights, hotel accommodations, ground transportation, tours, and emergency contacts. Each Umrah service is linked to a parent `service` record and supports per-passenger pricing (Adult/Child/Infant) with multi-currency display (PKR + SAR). The Umrah module is structurally similar to Hajj but has key differences in hotel management (category, qty, status fields) and transport/tour handling (sector_route-based instead of pickup/drop).

## Service Type

- **Type**: `"Umrah"`
- **Product Code**: Configured in `service_types` table (e.g., `101`)
- **Service Type ID**: Foreign key in the `services` table linking to `service_types`
- The system identifies a service as Umrah when `serviceType.type === "Umrah"`

## Database Schema

### Parent Table: `service_umrahs`

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `id` | INTEGER | No | Primary key, auto-increment |
| `service_id` | INTEGER | Yes | FK to `services` table (1:1 relationship) |
| `package_name` | STRING | Yes | Name of the Umrah package |
| `status` | STRING | Yes | Status as plain string (unlike Hajj which uses FK) |
| `departure_date` | DATEONLY | Yes | Package departure date |
| `arrival_date` | DATEONLY | Yes | Package arrival/return date |
| `guest_name` | STRING | Yes | Primary guest name (unique to Umrah) |
| `airline` | STRING | Yes | Airline information |
| `emergency_notes` | TEXT | Yes | Notes for emergency situations |
| `other_services_notes` | TEXT | Yes | Notes for additional services |
| `disclaimer` | TEXT | Yes | Disclaimer text for the package |
| `terms_and_conditions` | TEXT | Yes | Terms and conditions text |

### Child Table: `service_umrah_hotels`

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `id` | INTEGER | No | Primary key |
| `service_umrah_id` | INTEGER | No | FK to `service_umrahs` |
| `hotel_name` | STRING | Yes | Name of the hotel |
| `distance` | STRING | Yes | Distance from Haram |
| `category` | STRING | Yes | Hotel category/star rating (instead of `standard` in Hajj) |
| `city` | STRING | Yes | City (e.g., Makkah, Madinah) |
| `check_in` | DATEONLY | Yes | Check-in date |
| `check_out` | DATEONLY | Yes | Check-out date |
| `nights` | INTEGER | Yes | Number of nights (auto-calculated from check_in/check_out) |
| `room_type` | INTEGER | Yes | FK to `room_types` table |
| `view` | STRING | Yes | Room view type |
| `qty` | INTEGER | Yes | Number of rooms (unique to Umrah) |
| `meal_plan` | STRING | Yes | Meal plan type |
| `status` | STRING | Yes | Hotel booking status (unique to Umrah) |

### Child Table: `service_umrah_flights`

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `id` | INTEGER | No | Primary key |
| `service_umrah_id` | INTEGER | No | FK to `service_umrahs` |
| `flight` | STRING | Yes | Flight number/identifier |
| `flight_airline_id` | INTEGER | Yes | FK to `airline_codes` table |
| `flight_airline_code` | STRING | Yes | Airline IATA code |
| `date` | DATEONLY | Yes | Flight date |
| `dep_port` | STRING | Yes | Departure airport/city code |
| `arr_port` | STRING | Yes | Arrival airport/city code |
| `dep_time` | STRING | Yes | Departure time |
| `arr_time` | STRING | Yes | Arrival time |

### Child Table: `service_umrah_transports`

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `id` | INTEGER | No | Primary key |
| `service_umrah_id` | INTEGER | No | FK to `service_umrahs` |
| `sector_route` | STRING | Yes | Route description (e.g., "Jeddah → Makkah") — replaces `pick_up`/`drop` in Hajj |
| `date` | DATEONLY | Yes | Transport date |
| `vehicle` | STRING | Yes | Vehicle type |
| `type` | STRING | Yes | Transport type |
| `quantity` | INTEGER | Yes | Number of vehicles — replaces `booking_ref` in Hajj |

### Child Table: `service_umrah_tours`

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `id` | INTEGER | No | Primary key |
| `service_umrah_id` | INTEGER | No | FK to `service_umrahs` |
| `sector_route` | STRING | Yes | Tour route description |
| `date` | DATEONLY | Yes | Tour date |
| `vehicle` | STRING | Yes | Vehicle type |
| `type` | STRING | Yes | Tour type |
| `quantity` | INTEGER | Yes | Number of vehicles/units |

### Child Table: `service_umrah_contacts`

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `id` | INTEGER | No | Primary key |
| `service_umrah_id` | INTEGER | No | FK to `service_umrahs` |
| `location` | STRING | Yes | Staff location (e.g., "Makkah Staff") |
| `name` | STRING | Yes | Contact person name |
| `phone` | STRING | Yes | Primary phone number |
| `title` | STRING | Yes | Role/title |
| `phone2` | STRING | Yes | Alternate phone number |

## Database Associations

```
service (1) ──── (1) service_umrah
                        │
                        ├── (many) service_umrah_hotels ──── room_type (FK, alias: umrahRoomType)
                        ├── (many) service_umrah_flights ──── airline_code (FK)
                        ├── (many) service_umrah_transports
                        ├── (many) service_umrah_tours
                        └── (many) service_umrah_contacts
```

- `service_umrah.belongsTo(service)` via `service_id`
- `service.hasOne(service_umrah)` via `service_id`
- All child tables use alias names: `hotels`, `flights`, `transports`, `tours`, `emergency_contacts`
- `service_umrah_hotel.belongsTo(room_type)` as `umrahRoomType` (different alias from Hajj's `roomType`)
- `service_umrah_flight.belongsTo(airline_code)` via `flight_airline_id`

## Key Differences from Hajj

| Feature | Hajj | Umrah |
|---------|------|-------|
| Status field | INTEGER FK to `car_rental_statuses` | Plain STRING |
| Guest name | Not available | `guest_name` field on parent |
| Hotel rating | `standard` field | `category` field |
| Hotel confirmation | `conf` field | Not available |
| Hotel quantity | Not available | `qty` field |
| Hotel status | Not available | `status` field (string) |
| Transport route | `pick_up` + `drop` fields | Single `sector_route` field |
| Transport ref | `booking_ref` field | Not available |
| Transport count | Not available | `quantity` field |
| Tour location | `location` field | `sector_route` field |
| Tour ref | `booking_ref` field | Not available |
| Tour count | Not available | `quantity` field |
| Room type alias | `roomType` | `umrahRoomType` |

## API Endpoints

### Generic Service CRUD (handles Umrah via service type)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/service/` | JWT | Create service (include `umrah` object in body) |
| PUT | `/api/service/:id` | JWT | Update service |
| GET | `/api/service/:id` | JWT | Get service with all Umrah nested data |
| DELETE | `/api/service/:id` | JWT | Delete service |

### Umrah-Specific Endpoint

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/api/service/umrah-quotation/:serviceId` | JWT | Download Umrah quotation as Excel file |

## Request/Response Flow

### Create Flow

1. **Frontend** (`AddService.jsx`): User selects Umrah service type, fills `AddUmrah` component
2. **State**: `umrahDetail` object built with nested arrays (hotels, flights, transports, tours, emergency_contacts)
3. **API Call**: `POST /api/service/` with `umrah` object in request body
4. **Controller** (`service.controller.js`, lines ~1024-1060):
   - Creates parent `Service` record
   - Destructures `umrah` into child arrays and parent data
   - Creates `ServiceUmrah` record with `service_id`
   - Bulk creates all child records (hotels, transports, tours, flights, contacts)
   - All operations run within a database transaction
5. **Response**: Returns complete service object

### Update Flow

1. **Frontend**: Loads existing service via `GET /api/service/:id`
2. **User edits**: Modifies data in `AddUmrah` component
3. **API Call**: `PUT /api/service/:id` with updated `umrah` object
4. **Controller** (`service.controller.js`, lines ~3633-3682):
   - Finds existing `ServiceUmrah` or creates new one
   - Updates parent umrah data
   - **Destroys all existing child records** (hotels, transports, tours, flights, contacts)
   - **Recreates child records** from submitted data (with `id: undefined` to ensure fresh IDs)
   - All operations within transaction
5. **Response**: Returns updated service

### Get By ID Flow

1. **API Call**: `GET /api/service/:id`
2. **Controller** (`service.controller.js`, lines ~3058-3096):
   - Fetches service with eager loading of `ServiceUmrah` and all children
   - Includes airline code details for flights
   - Returns complete nested structure
3. **Frontend**: Populates `AddUmrah` component with received data

## Frontend Component: AddUmrah.jsx

**Location**: `/psfront/src/components/AddUmrah.jsx` (630 lines)

### Props

| Prop | Type | Description |
|------|------|-------------|
| `umrahDetail` | Object | Current umrah data (or empty object for new) |
| `setUmrahDetail` | Function | State setter to update parent |
| `type` | String | Service type description text |

### Component Tabs

The component renders 6 tabs:

#### Tab 1: Flight Information
- Table with columns: Flight, Airline (LiveComboBox), Date, Dep Port, Arr Port, Dep Time, Arr Time
- Add/remove flight rows dynamically
- Airline selection uses `LiveComboBox` component with airline code lookup

#### Tab 2: Hotel / Accommodation
- Table with columns: Hotel Name, Distance, Category, City, Check-in, Check-out, Nights (read-only), Room Type (dropdown), View, Qty, Meal Plan, Status
- **Nights auto-calculated**: When check-in or check-out changes, nights are computed as `(check_out - check_in) / (1000 * 60 * 60 * 24)`
- Additional fields vs Hajj: `category` (instead of `standard`), `qty`, `status`

#### Tab 3: Transportation
- Table with columns: Sector/Route, Date, Vehicle, Type, Quantity
- Uses `sector_route` instead of separate pick_up/drop fields
- `quantity` field instead of `booking_ref`

#### Tab 4: Tours
- Table with columns: Sector/Route, Date, Vehicle, Type, Quantity
- Same structure as transportation

#### Tab 5: Other Services
- Single textarea for `other_services_notes`

#### Tab 6: Emergency Contact
- Table with columns: Location, Name, Phone, Title, Phone 2
- Add/remove contact rows
- Additional fields: Emergency Notes (textarea), Disclaimer (textarea), Terms & Conditions (textarea)

### Header Fields (above tabs)
- Package Name (text input)
- Status (text input — plain string, not dropdown)
- Departure Date (date input)
- Arrival Date (date input)
- Guest Name (text input — unique to Umrah)
- Airline (text input)

### Visual Design
- Green header bar with Kaaba icon (`KaabaIcon` component)
- Consistent with Hajj component styling

## Excel Quotation Export

**Location**: `/psback/controllers/umrahExport.controller.js` (851 lines)

### Function: `downloadUmrahQuotation(req, res)`

Generates an Excel workbook with the following sections:

1. **Company Header**: Company name, branch location, office contact, email
2. **Booking Details**: Booking number, date, invoice number, status
3. **Customer Info**: Customer name, package name, client type, service status
4. **Contact Details**: Contact person, guest name, sales staff, team ID
5. **Pilgrims Section**: Adult/Child/Infant counts (deduplicated across all Umrah services in the order)
6. **Flight Details**: Flight info from umrah flights data
7. **Passenger List**: Numbered list with name, type, passport, ticket number
8. **Hotel Details**: Hotel name, distance, category, city, check-in/out, nights, room type, view, qty, meal plan, status, total nights summary
9. **Transportation**: Sector/route, vehicle type, quantity
10. **Tours**: Sector/route, date, vehicle, type, quantity
11. **Pricing Section**:
    - Per-pax pricing breakdown (Adult/Child/Infant)
    - Separate ticket and package pricing per passenger type
    - Total cost with service charges/discounts
    - Net total in PKR
12. **Packages Template**: Pre-formatted rows for QUINT, QUAD, TRIPLE, DOUBLE, CHILD, INFANT pricing
13. **Terms and Conditions**: Minimum 5 rows for manual entry

### Pricing Calculation
- Collects all Umrah services in the order
- Deduplicates passengers across services
- Calculates per-pax cost: `total_cost / count_of_passenger_type`
- Separates ticket pricing from package pricing
- Displays both PKR and SAR amounts

## Files Reference

| File | Purpose |
|------|---------|
| `psback/models/service_umrah.js` | Parent Umrah model definition |
| `psback/models/service_umrah_hotel.js` | Hotel child model |
| `psback/models/service_umrah_flight.js` | Flight child model |
| `psback/models/service_umrah_transport.js` | Transport child model |
| `psback/models/service_umrah_tour.js` | Tour child model |
| `psback/models/service_umrah_contact.js` | Emergency contact child model |
| `psback/models/index.js` | Association definitions (lines ~851-865) |
| `psback/controllers/service.controller.js` | Create (~1024-1060), Update (~3633-3682), GetById (~3058-3096) |
| `psback/controllers/umrahExport.controller.js` | Excel quotation export (851 lines) |
| `psback/routes/service.route.js` | Route definitions |
| `psfront/src/components/AddUmrah.jsx` | Frontend form component (630 lines) |
| `psfront/src/pages/AddService.jsx` | Parent page (conditional render) |
| `psfront/src/api/service.js` | API calls (`downloadUmrahQuotation`) |
