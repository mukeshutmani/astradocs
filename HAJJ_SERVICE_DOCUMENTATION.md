# Hajj Service Documentation

## Overview

The Hajj Service module manages Hajj pilgrimage package bookings within the PowerSuite travel system. It supports complete package management including flights, hotel accommodations, ground transportation, tours, and emergency contacts. Each Hajj service is linked to a parent `service` record and supports per-passenger pricing (Adult/Child/Infant) with multi-currency display (PKR + SAR).

## Service Type

- **Type**: `"Hajj"`
- **Product Code**: Configured in `service_types` table (e.g., `103`)
- **Service Type ID**: Foreign key in the `services` table linking to `service_types`
- The system identifies a service as Hajj when `serviceType.type === "Hajj"`

## Database Schema

### Parent Table: `service_hajjs`

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `id` | INTEGER | No | Primary key, auto-increment |
| `service_id` | INTEGER | Yes | FK to `services` table (1:1 relationship) |
| `package_name` | STRING | Yes | Name of the Hajj package |
| `status` | INTEGER | Yes | FK to `car_rental_statuses` table |
| `departure_date` | DATEONLY | Yes | Package departure date |
| `arrival_date` | DATEONLY | Yes | Package arrival/return date |
| `airline` | STRING | Yes | Airline information |
| `emergency_notes` | TEXT | Yes | Notes for emergency situations |
| `other_services_notes` | TEXT | Yes | Notes for additional services |
| `disclaimer` | TEXT | Yes | Disclaimer text for the package |
| `terms_and_conditions` | TEXT | Yes | Terms and conditions text |

### Child Table: `service_hajj_hotels`

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `id` | INTEGER | No | Primary key |
| `service_hajj_id` | INTEGER | No | FK to `service_hajjs` |
| `hotel_name` | STRING | Yes | Name of the hotel |
| `city` | STRING | Yes | City (e.g., Makkah, Madinah) |
| `distance` | STRING | Yes | Distance from Haram |
| `standard` | STRING | Yes | Hotel standard/rating |
| `check_in` | DATEONLY | Yes | Check-in date |
| `check_out` | DATEONLY | Yes | Check-out date |
| `nights` | INTEGER | Yes | Number of nights (auto-calculated from check_in/check_out) |
| `meal_plan` | STRING | Yes | Meal plan type |
| `view` | STRING | Yes | Room view type |
| `room_type` | INTEGER | Yes | FK to `room_types` table |
| `booking_ref` | STRING | Yes | Hotel booking reference |
| `conf` | STRING | Yes | Confirmation number |

### Child Table: `service_hajj_flights`

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `id` | INTEGER | No | Primary key |
| `service_hajj_id` | INTEGER | No | FK to `service_hajjs` |
| `flight` | STRING | Yes | Flight number/identifier |
| `flight_airline_id` | INTEGER | Yes | FK to `airline_codes` table |
| `flight_airline_code` | STRING | Yes | Airline IATA code |
| `date` | DATEONLY | Yes | Flight date |
| `dep_port` | STRING | Yes | Departure airport/city code |
| `arr_port` | STRING | Yes | Arrival airport/city code |
| `dep_time` | STRING | Yes | Departure time |
| `arr_time` | STRING | Yes | Arrival time |

### Child Table: `service_hajj_transports`

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `id` | INTEGER | No | Primary key |
| `service_hajj_id` | INTEGER | No | FK to `service_hajjs` |
| `date` | DATEONLY | Yes | Transport date |
| `booking_ref` | STRING | Yes | Booking reference |
| `vehicle` | STRING | Yes | Vehicle type |
| `pick_up` | STRING | Yes | Pick-up location |
| `drop` | STRING | Yes | Drop-off location |
| `type` | STRING | Yes | Transport type (e.g., Private, Shared) |

### Child Table: `service_hajj_tours`

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `id` | INTEGER | No | Primary key |
| `service_hajj_id` | INTEGER | No | FK to `service_hajjs` |
| `date` | DATEONLY | Yes | Tour date |
| `booking_ref` | STRING | Yes | Booking reference |
| `vehicle` | STRING | Yes | Vehicle type |
| `location` | STRING | Yes | Tour location |
| `type` | STRING | Yes | Tour type |

### Child Table: `service_hajj_contacts`

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `id` | INTEGER | No | Primary key |
| `service_hajj_id` | INTEGER | No | FK to `service_hajjs` |
| `location` | STRING | Yes | Staff location (e.g., "Makkah Staff") |
| `name` | STRING | Yes | Contact person name |
| `phone` | STRING | Yes | Primary phone number |
| `title` | STRING | Yes | Role/title |
| `phone2` | STRING | Yes | Alternate phone number |

## Database Associations

```
service (1) ──── (1) service_hajj
                        │
                        ├── (many) service_hajj_hotels ──── room_type (FK)
                        ├── (many) service_hajj_flights ──── airline_code (FK)
                        ├── (many) service_hajj_transports
                        ├── (many) service_hajj_tours
                        └── (many) service_hajj_contacts
```

- `service_hajj.belongsTo(service)` via `service_id`
- `service.hasOne(service_hajj)` via `service_id`
- `service_hajj.belongsTo(car_rental_status)` via `status`
- All child tables use alias names: `hotels`, `flights`, `transports`, `tours`, `emergency_contacts`
- `service_hajj_hotel.belongsTo(room_type)` as `roomType`
- `service_hajj_flight.belongsTo(airline_code)` via `flight_airline_id`

## API Endpoints

### Generic Service CRUD (handles Hajj via service type)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/service/` | JWT | Create service (include `hajj` object in body) |
| PUT | `/api/service/:id` | JWT | Update service |
| GET | `/api/service/:id` | JWT | Get service with all Hajj nested data |
| DELETE | `/api/service/:id` | JWT | Delete service |

### Hajj-Specific Endpoint

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/api/service/hajj-quotation/:serviceId` | JWT | Download Hajj quotation as Excel file |

## Request/Response Flow

### Create Flow

1. **Frontend** (`AddService.jsx`): User selects Hajj service type, fills `AddHajj` component
2. **State**: `hajjDetail` object built with nested arrays (hotels, flights, transports, tours, emergency_contacts)
3. **API Call**: `POST /api/service/` with `hajj` object in request body
4. **Controller** (`service.controller.js`, lines ~986-1022):
   - Creates parent `Service` record
   - Destructures `hajj` into child arrays and parent data
   - Creates `ServiceHajj` record with `service_id`
   - Bulk creates all child records (hotels, transports, tours, flights, contacts)
   - All operations run within a database transaction
5. **Response**: Returns complete service object

### Update Flow

1. **Frontend**: Loads existing service via `GET /api/service/:id`
2. **User edits**: Modifies data in `AddHajj` component
3. **API Call**: `PUT /api/service/:id` with updated `hajj` object
4. **Controller** (`service.controller.js`, lines ~3581-3631):
   - Finds existing `ServiceHajj` or creates new one
   - Updates parent hajj data
   - **Destroys all existing child records** (hotels, transports, tours, flights, contacts)
   - **Recreates child records** from submitted data (with `id: undefined` to ensure fresh IDs)
   - All operations within transaction
5. **Response**: Returns updated service

### Get By ID Flow

1. **API Call**: `GET /api/service/:id`
2. **Controller** (`service.controller.js`, lines ~3017-3050):
   - Fetches service with eager loading of `ServiceHajj` and all children
   - Includes airline code details for flights
   - Returns complete nested structure
3. **Frontend**: Populates `AddHajj` component with received data

## Frontend Component: AddHajj.jsx

**Location**: `/psfront/src/components/AddHajj.jsx` (617 lines)

### Props

| Prop | Type | Description |
|------|------|-------------|
| `hajjDetail` | Object | Current hajj data (or empty object for new) |
| `setHajjDetail` | Function | State setter to update parent |
| `type` | String | Service type description text |

### Component Tabs

The component renders 6 tabs:

#### Tab 1: Flight Information
- Table with columns: Flight, Airline (LiveComboBox), Date, Dep Port, Arr Port, Dep Time, Arr Time
- Add/remove flight rows dynamically
- Airline selection uses `LiveComboBox` component with airline code lookup

#### Tab 2: Accommodation
- Table with columns: Hotel Name, City, Distance, Standard, Check-in, Check-out, Nights (read-only), Meal Plan, View, Room Type (dropdown), Booking Ref, Conf
- **Nights auto-calculated**: When check-in or check-out changes, nights are computed as `(check_out - check_in) / (1000 * 60 * 60 * 24)`

#### Tab 3: Transport
- Table with columns: Date, Booking Ref, Vehicle, Pick Up, Drop, Type
- Add/remove transport rows

#### Tab 4: Tours
- Table with columns: Date, Booking Ref, Vehicle, Location, Type
- Add/remove tour rows

#### Tab 5: Other Services
- Single textarea for `other_services_notes`

#### Tab 6: Emergency Contact
- Table with columns: Location, Name, Phone, Title, Phone 2
- Add/remove contact rows
- Additional fields: Emergency Notes (textarea), Disclaimer (textarea), Terms & Conditions (textarea)

### Header Fields (above tabs)
- Package Name (text input)
- Status (dropdown from `car_rental_statuses`)
- Departure Date (date input)
- Arrival Date (date input)
- Airline (text input)

### Visual Design
- Green header bar with Kaaba icon (`KaabaIcon` component)
- Consistent with other service type components

## Excel Quotation Export

**Location**: `/psback/controllers/hajjExport.controller.js`

### Function: `downloadHajjQuotation(req, res)`

Generates an Excel workbook with the following sections:

1. **Company Header**: Company name, address, contact info, email
2. **Booking Details**: Booking number, date, invoice number, status
3. **Customer Info**: Customer name, package name, client type, service status
4. **Contact Details**: Contact person, sales staff, team info
5. **Pilgrims Section**: Adult/Child/Infant counts (deduplicated across all Hajj services in the order)
6. **Flight Details**: Flight info from hajj flights data
7. **Passenger List**: Numbered list with name, type, passport, ticket number
8. **Service Breakdown**: Per-service pricing details
9. **Pricing Summary**: Per-pax pricing (ADT/CHD/INF), total cost breakdown

### Pricing Calculation
- Collects all Hajj services in the order
- Deduplicates passengers across services
- Calculates per-pax cost: `total_cost / count_of_passenger_type`
- Displays both PKR and SAR amounts

## Files Reference

| File | Purpose |
|------|---------|
| `psback/models/service_hajj.js` | Parent Hajj model definition |
| `psback/models/service_hajj_hotel.js` | Hotel child model |
| `psback/models/service_hajj_flight.js` | Flight child model |
| `psback/models/service_hajj_transport.js` | Transport child model |
| `psback/models/service_hajj_tour.js` | Tour child model |
| `psback/models/service_hajj_contact.js` | Emergency contact child model |
| `psback/models/index.js` | Association definitions (lines ~833-849) |
| `psback/controllers/service.controller.js` | Create (~986-1022), Update (~3581-3631), GetById (~3017-3050) |
| `psback/controllers/hajjExport.controller.js` | Excel quotation export |
| `psback/routes/service.route.js` | Route definitions |
| `psfront/src/components/AddHajj.jsx` | Frontend form component (617 lines) |
| `psfront/src/pages/AddService.jsx` | Parent page (conditional render) |
| `psfront/src/api/service.js` | API calls (`downloadHajjQuotation`) |
