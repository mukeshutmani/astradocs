# PowerSuite Refund Module Multi-Service Implementation Plan

## Executive Summary
This document outlines the comprehensive plan to extend the PowerSuite refund module from supporting only Flight services to accommodating all service types: Hotel, Insurance, Tour, Car Rental, VISA, Train, and Cruise.

## Current State Analysis

### What Works Well
- Robust refund workflow (Raised → Printed → Void)
- Credit note generation system
- Customer/Supplier refund amount tracking
- Integration with invoices and costs
- Sequential processing and validation

### What Needs Enhancement
- Service-specific data handling (currently Flight-only)
- Segment/ticket selection logic (Flight-specific)
- UI components hardcoded for Flight patterns
- Validation rules tied to Flight segments

## Service-Specific Requirements

### 1. Flight (Existing) ✅
- **Refund Units**: Segments, Tickets, Passengers
- **Special Logic**: Multi-segment journeys, passenger-segment linking
- **Status**: Fully implemented

### 2. Hotel 🏨
- **Refund Units**: Rooms, Nights, Guests
- **Special Logic**:
  - Partial night refunds
  - Room-wise cancellation
  - Check-in/check-out date modifications
- **Data Structure**:
  ```json
  {
    "selected_rooms": [{"room_number": "101", "nights": [1,2,3]}],
    "selected_guests": [{"name": "John Doe", "room": "101"}],
    "refund_type": "full|partial|nights"
  }
  ```

### 3. Insurance 📋
- **Refund Units**: Policy, Coverage Items, Beneficiaries
- **Special Logic**:
  - Premium refund calculations
  - Partial coverage cancellation
  - Pro-rata refunds based on usage period
- **Data Structure**:
  ```json
  {
    "policy_number": "INS-2024-001",
    "coverage_items": ["medical", "baggage"],
    "refund_basis": "unused_period|cancellation|claim_rejection"
  }
  ```

### 4. Tour 🚌
- **Refund Units**: Participants, Components (hotel, transport, activities)
- **Special Logic**:
  - Component-wise refunds
  - Group booking considerations
  - Partial tour refunds
- **Data Structure**:
  ```json
  {
    "selected_participants": [{"name": "Jane Doe", "id": 1}],
    "selected_components": ["hotel", "transport", "activity_1"],
    "tour_segments": ["day1", "day2"]
  }
  ```

### 5. Car Rental 🚗
- **Refund Units**: Rental Days, Extras (GPS, child seat)
- **Special Logic**:
  - Early return refunds
  - Unused day calculations
  - Extra services refunds
- **Data Structure**:
  ```json
  {
    "rental_period": {"from": "2024-01-01", "to": "2024-01-07"},
    "actual_return": "2024-01-05",
    "selected_extras": ["gps", "insurance"],
    "refund_days": 2
  }
  ```

### 6. VISA 📄
- **Refund Units**: Application, Applicants
- **Special Logic**:
  - Simple refund (usually full amount)
  - Application rejection refunds
  - Processing fee considerations
- **Data Structure**:
  ```json
  {
    "application_number": "VISA-2024-001",
    "applicants": [{"name": "John Doe", "passport": "A123456"}],
    "refund_reason": "rejection|cancellation|duplicate"
  }
  ```

### 7. Train 🚂
- **Refund Units**: Segments, Seats, Passengers (similar to Flight)
- **Special Logic**:
  - Journey segment refunds
  - Seat-wise cancellation
  - Class-based refund rules
- **Data Structure**:
  ```json
  {
    "selected_segments": [{"from": "NYC", "to": "BOS"}],
    "selected_seats": ["A1", "A2"],
    "selected_passengers": [{"name": "John", "seat": "A1"}]
  }
  ```

### 8. Cruise 🚢
- **Refund Units**: Cabins, Passengers, Shore Excursions
- **Special Logic**:
  - Cabin-wise refunds
  - Shore excursion cancellations
  - Dining/beverage package refunds
- **Data Structure**:
  ```json
  {
    "selected_cabins": ["C101", "C102"],
    "selected_passengers": [{"name": "John", "cabin": "C101"}],
    "selected_packages": ["dining", "excursion_1"]
  }
  ```

## Technical Architecture

### 1. Database Schema Modifications

#### Option A: Polymorphic Approach (Recommended)
```sql
-- Add service_type to distinguish
ALTER TABLE refunds
ADD COLUMN service_type ENUM('Flight','Hotel','Insurance','Tour','CarRental','Visa','Train','Cruise') NOT NULL;

-- Create service-specific detail tables
CREATE TABLE refund_details_hotel (
    id INT PRIMARY KEY AUTO_INCREMENT,
    refund_id INT REFERENCES refunds(id),
    selected_rooms JSON,
    selected_nights JSON,
    selected_guests JSON,
    refund_type VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE refund_details_insurance (
    id INT PRIMARY KEY AUTO_INCREMENT,
    refund_id INT REFERENCES refunds(id),
    policy_number VARCHAR(255),
    coverage_items JSON,
    refund_basis VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Similar tables for other services...
```

#### Option B: JSON Extension (Quick Implementation)
```sql
ALTER TABLE refunds
ADD COLUMN service_type ENUM('Flight','Hotel','Insurance','Tour','CarRental','Visa','Train','Cruise') NOT NULL,
ADD COLUMN service_details JSON COMMENT 'Service-specific refund details';

-- Deprecate flight-specific columns gradually
-- selected_segments, selected_tickets, selected_passengers remain for backward compatibility
```

### 2. Backend Architecture

#### Service Handler Pattern
```javascript
// /psback/services/refund/RefundHandlerFactory.js
class RefundHandlerFactory {
  static getHandler(serviceType) {
    switch(serviceType) {
      case 'Flight': return new FlightRefundHandler();
      case 'Hotel': return new HotelRefundHandler();
      case 'Insurance': return new InsuranceRefundHandler();
      // ... other services
    }
  }
}

// /psback/services/refund/handlers/BaseRefundHandler.js
class BaseRefundHandler {
  async validate(refundData) { /* Common validation */ }
  async calculate(service, refundData) { /* Override in subclass */ }
  async extractServiceData(service) { /* Override in subclass */ }
  async generateRefundDetails(refundData) { /* Override in subclass */ }
}

// /psback/services/refund/handlers/HotelRefundHandler.js
class HotelRefundHandler extends BaseRefundHandler {
  async calculate(service, refundData) {
    // Hotel-specific refund calculation
    const { selected_rooms, selected_nights } = refundData;
    // Calculate based on rooms and nights
  }

  async extractServiceData(service) {
    return {
      rooms: service.hotel_rooms,
      nights: service.hotel_nights,
      guests: service.hotel_guests
    };
  }
}
```

### 3. Controller Modifications

```javascript
// /psback/controllers/invoice.controller.js modifications

// Modified refund creation
exports.createRefund = async (req, res) => {
  try {
    const refundData = req.body.data.data;

    for (const refund of refundData) {
      // Get service type
      const service = await Service.findByPk(refund.service_id);
      const serviceType = service.service_type;

      // Get appropriate handler
      const handler = RefundHandlerFactory.getHandler(serviceType);

      // Validate service-specific requirements
      await handler.validate(refund);

      // Calculate refund amounts if needed
      if (!refund.customer_refund_amount) {
        refund.customer_refund_amount = await handler.calculate(service, refund);
      }

      // Save refund with service-specific details
      const refundRecord = await Refund.create({
        ...refund,
        service_type: serviceType,
        service_details: await handler.generateRefundDetails(refund)
      });

      // Create service-specific detail record if using Option A
      if (serviceType === 'Hotel') {
        await RefundDetailsHotel.create({
          refund_id: refundRecord.id,
          ...handler.extractHotelDetails(refund)
        });
      }
    }

    res.json({ success: true });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

### 4. Frontend Architecture

#### Component Structure
```
/psfront/src/
├── pages/Order/
│   ├── Refund.jsx (Main refund page - minimal changes)
│   └── RefundDetail.jsx (Service-aware detail page)
├── components/refund/
│   ├── ServiceRefundSelector.jsx (Dynamic service selector)
│   ├── flight/
│   │   └── FlightRefundForm.jsx (Existing)
│   ├── hotel/
│   │   ├── HotelRefundForm.jsx
│   │   └── RoomNightSelector.jsx
│   ├── insurance/
│   │   ├── InsuranceRefundForm.jsx
│   │   └── CoverageSelector.jsx
│   ├── tour/
│   │   ├── TourRefundForm.jsx
│   │   └── ComponentSelector.jsx
│   └── ... (other services)
```

#### Dynamic Service Component Loading
```jsx
// /psfront/src/components/refund/ServiceRefundSelector.jsx
import FlightRefundForm from './flight/FlightRefundForm';
import HotelRefundForm from './hotel/HotelRefundForm';
// ... other imports

const SERVICE_COMPONENTS = {
  Flight: FlightRefundForm,
  Hotel: HotelRefundForm,
  Insurance: InsuranceRefundForm,
  Tour: TourRefundForm,
  CarRental: CarRentalRefundForm,
  VISA: VisaRefundForm,
  Train: TrainRefundForm,
  Cruise: CruiseRefundForm
};

export default function ServiceRefundSelector({ service, refundData, onChange }) {
  const ServiceComponent = SERVICE_COMPONENTS[service.service_type];

  if (!ServiceComponent) {
    return <div>Refund not available for this service type</div>;
  }

  return <ServiceComponent
    service={service}
    refundData={refundData}
    onChange={onChange}
  />;
}
```

## Implementation Phases

### Phase 1: Foundation (Week 1-2)
1. **Database Schema Updates**
   - Add service_type column
   - Create service detail tables/JSON structure
   - Migration scripts

2. **Backend Handler Architecture**
   - Create BaseRefundHandler
   - Implement RefundHandlerFactory
   - Create FlightRefundHandler (refactor existing)

3. **API Modifications**
   - Update refund endpoints to be service-aware
   - Add service type validation

### Phase 2: Service Implementation (Week 3-4)
Implement handlers and UI for each service in priority order:

1. **Hotel & Car Rental** (Similar patterns)
   - Handler implementation
   - UI components
   - Validation rules

2. **Insurance & VISA** (Simple refunds)
   - Handler implementation
   - Basic UI forms
   - Simple validation

3. **Tour** (Complex components)
   - Handler with component logic
   - Multi-select UI
   - Component-wise calculations

4. **Train & Cruise** (Flight-like)
   - Adapt Flight handler pattern
   - Segment/cabin selection UI
   - Passenger management

### Phase 3: Integration & Testing (Week 5-6)
1. **Integration Testing**
   - End-to-end tests for each service
   - Credit note generation verification
   - Refund calculation accuracy

2. **UI Polish**
   - Consistent styling across service forms
   - Loading states and error handling
   - Service-specific help text

3. **Migration & Deployment**
   - Data migration for existing refunds
   - Staged rollout plan
   - Documentation update

## Implementation Priority Order

Based on business value and complexity:

1. **Hotel** (High usage, medium complexity)
2. **Car Rental** (High usage, low complexity)
3. **Insurance** (Medium usage, low complexity)
4. **VISA** (Medium usage, lowest complexity)
5. **Tour** (Medium usage, high complexity)
6. **Train** (Low usage, medium complexity)
7. **Cruise** (Low usage, high complexity)

## Validation Rules by Service

### Hotel
- At least one room or night must be selected
- Refund date must be before check-in or within cancellation policy
- Guest count must match room capacity

### Insurance
- Policy must be active
- Refund amount cannot exceed premium paid
- Claim status must be checked

### Tour
- At least one participant or component selected
- Tour start date consideration
- Group booking minimums respected

### Car Rental
- Return date validation
- Minimum rental period checks
- Extra services dependency validation

### VISA
- Application status must be appropriate for refund
- Embassy fee vs service fee distinction
- Document submission status check

### Train
- Similar to Flight segment validation
- Seat selection required
- Journey date validation

### Cruise
- Cabin selection required
- Sailing date consideration
- Package dependency validation

## API Endpoints Structure

### Generic Refund Endpoints
```
POST   /api/refund                 # Create refund (service-aware)
GET    /api/refund/:refundNo       # Get refund details
PUT    /api/refund/:refundNo       # Update refund
DELETE /api/refund/:refundNo       # Delete refund

# Service-specific validation
POST   /api/refund/validate/:serviceType

# Service-specific calculations
POST   /api/refund/calculate/:serviceType
```

### Service Data Extraction
```
GET /api/service/:serviceId/refund-data
Returns service-specific refundable items based on service type
```

## Risk Mitigation

### Backward Compatibility
- Keep existing Flight logic intact during migration
- Use feature flags for gradual rollout
- Maintain old API endpoints with deprecation notices

### Data Integrity
- Comprehensive validation at both frontend and backend
- Transaction-based refund creation
- Audit logging for all refund operations

### Performance
- Lazy load service-specific components
- Optimize service detail queries
- Cache service-specific validation rules

## Success Metrics

1. **Technical Metrics**
   - All services supported with <2% error rate
   - Refund creation time <3 seconds
   - 100% backward compatibility maintained

2. **Business Metrics**
   - Refund processing time reduced by 30%
   - Support tickets for refunds reduced by 40%
   - Customer satisfaction score improved

3. **Quality Metrics**
   - >80% test coverage for new code
   - Zero critical bugs in production
   - All service types documented

## Next Steps

### Immediate Actions (This Week)
1. Review and approve this plan
2. Set up development branch
3. Create database migration scripts
4. Begin Phase 1 implementation

### Week 1 Deliverables
1. Database schema updated
2. Base handler architecture in place
3. Flight refund migrated to new pattern
4. Hotel refund handler implemented

### Communication Plan
- Daily standup updates
- Weekly progress reports
- Bi-weekly stakeholder demos
- Continuous documentation updates

## Appendix

### A. Database Migration Script Sample
```sql
-- Add service_type with default for existing records
ALTER TABLE refunds
ADD COLUMN service_type VARCHAR(50) DEFAULT 'Flight' NOT NULL AFTER service_id;

-- Add service_details for flexible storage
ALTER TABLE refunds
ADD COLUMN service_details JSON AFTER service_type;

-- Migrate existing flight-specific data
UPDATE refunds
SET service_details = JSON_OBJECT(
  'segments', selected_segments,
  'tickets', selected_tickets,
  'passengers', selected_passengers
)
WHERE service_type = 'Flight';
```

### B. Service Detection Logic
```javascript
// Utility to detect service type from service record
function detectServiceType(service) {
  if (service.flight_id) return 'Flight';
  if (service.hotel_id) return 'Hotel';
  if (service.insurance_id) return 'Insurance';
  if (service.tour_id) return 'Tour';
  if (service.car_id) return 'CarRental';
  if (service.visa_id) return 'VISA';
  if (service.train_id) return 'Train';
  if (service.cruise_id) return 'Cruise';
  return 'Unknown';
}
```

### C. Testing Checklist Template
```markdown
## [Service Type] Refund Testing

### Functional Tests
- [ ] Create refund with minimal data
- [ ] Create refund with all options
- [ ] Validate service-specific rules
- [ ] Calculate refund amount
- [ ] Generate credit note
- [ ] Update refund
- [ ] Delete refund (Raised status)

### Edge Cases
- [ ] Invalid service data
- [ ] Exceeded amounts
- [ ] Missing required fields
- [ ] Concurrent operations

### Integration Tests
- [ ] Invoice linkage
- [ ] Cost document validation
- [ ] Journal entry generation
- [ ] Credit note PDF generation
```

---

**Document Version**: 1.0
**Created**: January 2025
**Status**: Ready for Review and Implementation