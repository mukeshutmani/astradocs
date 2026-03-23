# Date Format Migration Guide - PowerSuite

## Overview
This guide explains how to migrate PowerSuite to use consistent DD-MM-YYYY date formatting throughout the application without breaking existing functionality.

## Key Principles
1. **Database remains unchanged**: Dates continue to be stored in ISO format (YYYY-MM-DD) in MySQL
2. **Display format standardized**: All user-facing dates use DD-MM-YYYY format
3. **API consistency**: API responses return dates in DD-MM-YYYY format
4. **Backward compatibility**: Existing data and functionality remain intact

## Implementation Files Created

### 1. Frontend Date Utility
**File**: `psfront/src/utils/dateFormatter.js`
- Centralized date formatting functions
- Handles conversion between display (DD-MM-YYYY) and API/database formats (YYYY-MM-DD)
- Includes validation, parsing, and formatting helpers

### 2. Backend Date Utility
**File**: `psback/utils/dateFormatter.js`
- Server-side date formatting
- Request/response processing helpers
- Middleware for automatic date conversion
- Timezone-aware formatting

### 3. React Hook
**File**: `psfront/src/hooks/useDateFormatter.js`
- Custom hook for React components
- Memoized formatting functions
- Form field helpers

## Migration Steps

### Phase 1: Setup (No Breaking Changes)
1. Install the utility files (already done)
2. No immediate changes to existing code
3. Test utilities independently

### Phase 2: Backend Migration

#### Update Controllers
```javascript
// Before (in any controller)
const moment = require('moment');
invoice_date: moment().format("YYYY-MM-DD HH:mm:ss")

// After
const { formatDateForDatabase, processResponseDates } = require('../utils/dateFormatter');
invoice_date: formatDateForDatabase(new Date(), true)
```

#### Add Middleware for Automatic Conversion
```javascript
// In psback/app.js or specific routes
const { dateProcessingMiddleware } = require('./utils/dateFormatter');

// For specific routes with date fields
app.use('/api/order', dateProcessingMiddleware(['trip_date', 'trip_deadline']));
app.use('/api/service/flight', dateProcessingMiddleware(['departure_date', 'arrival_date']));
```

#### Update API Responses
```javascript
// In controllers, wrap responses
const { processResponseDates } = require('../utils/dateFormatter');

// Before
res.json({ success: true, data: orders });

// After
const processedOrders = processResponseDates(orders, ['trip_date', 'trip_deadline']);
res.json({ success: true, data: processedOrders });
```

### Phase 3: Frontend Migration

#### Update Components Using Dates
```javascript
// Before (in AddHotel.jsx)
import moment from "moment";

check_in: hotelDetail?.check_in
  ? moment(hotelDetail.check_in).format("YYYY-MM-DD")
  : "",

// After
import { formatDateForAPI, formatDateForDisplay } from '@/utils/dateFormatter';

check_in: hotelDetail?.check_in
  ? formatDateForAPI(hotelDetail.check_in)
  : "",
```

#### Use the Custom Hook
```javascript
// In React components
import { useDateFormatter } from '@/hooks/useDateFormatter';

const MyComponent = () => {
  const dateFormatter = useDateFormatter();

  return (
    <div>
      {/* Display formatted date */}
      <span>{dateFormatter.toDisplay(order.trip_date)}</span>

      {/* Date input field */}
      <input
        {...dateFormatter.getDateFieldProps(selectedDate, setSelectedDate)}
      />
    </div>
  );
};
```

### Phase 4: Update EJS Templates

```javascript
// Before (in invoiceDocument.ejs)
<%= moment(invoice.invoice_date).format('DD-MMM-YY') %>

// After
<%= moment(invoice.invoice_date).format('DD-MM-YYYY') %>
```

## Component-Specific Migration Examples

### 1. AddHotel Component
```javascript
// Updated AddHotel.jsx
import { formatDateForAPI, formatDateForDisplay, getDateDifference } from '@/utils/dateFormatter';

// In the component
const [values, setValues] = useState({
  check_in: hotelDetail?.check_in
    ? formatDateForAPI(hotelDetail.check_in)
    : "",
  check_out: hotelDetail?.check_out
    ? formatDateForAPI(hotelDetail.check_out)
    : "",
  // ... other fields
});

// Calculate nights
const calculateNights = (checkIn, checkOut) => {
  return getDateDifference(checkIn, checkOut, 'days');
};

// Display date
<span>Check-in: {formatDateForDisplay(values.check_in)}</span>
```

### 2. Order List Component
```javascript
// In OrderList.jsx
import { useDateFormatter } from '@/hooks/useDateFormatter';

const OrderList = () => {
  const dateFormatter = useDateFormatter();
  const [orders, setOrders] = useState([]);

  useEffect(() => {
    fetchOrders().then(data => {
      // Dates are already formatted by backend
      setOrders(data);
    });
  }, []);

  return (
    <table>
      {orders.map(order => (
        <tr key={order.id}>
          <td>{order.trip_date}</td> {/* Already in DD-MM-YYYY from API */}
        </tr>
      ))}
    </table>
  );
};
```

### 3. DateTimePicker Component
```javascript
// Update DateTimePicker.jsx to use the formatter
import { formatDateForDisplay, parseDateFromDisplay } from '@/utils/dateFormatter';

// When displaying selected date
const displayDate = formatDateForDisplay(selectedDate, includeTime);

// When parsing user input
const parsedDate = parseDateFromDisplay(inputValue);
```

## Database Queries

No changes needed for database queries. Continue using ISO format:

```javascript
// Sequelize queries remain unchanged
where: {
  trip_date: {
    [Op.between]: [startDate, endDate] // Still use Date objects or ISO strings
  }
}
```

## API Request/Response Examples

### Request (Frontend to Backend)
```javascript
// Frontend sends DD-MM-YYYY (user input)
{
  trip_date: "25-12-2024",
  customer_name: "John Doe"
}

// Backend middleware converts to YYYY-MM-DD for database
{
  trip_date: "2024-12-25",
  customer_name: "John Doe"
}
```

### Response (Backend to Frontend)
```javascript
// Database returns YYYY-MM-DD
{
  trip_date: "2024-12-25",
  created_at: "2024-12-25 10:30:00"
}

// Backend converts to DD-MM-YYYY for response
{
  trip_date: "25-12-2024",
  created_at: "25-12-2024 10:30"
}
```

## Testing Strategy

### 1. Unit Tests for Date Utilities
```javascript
// Test date formatter functions
describe('Date Formatter', () => {
  test('formats date for display', () => {
    const date = new Date('2024-12-25');
    expect(formatDateForDisplay(date)).toBe('25-12-2024');
  });

  test('parses display format', () => {
    const dateString = '25-12-2024';
    const parsed = parseDateFromDisplay(dateString);
    expect(parsed.getFullYear()).toBe(2024);
    expect(parsed.getMonth()).toBe(11); // 0-indexed
    expect(parsed.getDate()).toBe(25);
  });
});
```

### 2. Integration Tests
- Test date conversion in API endpoints
- Verify database storage remains in ISO format
- Check display formatting in UI components

### 3. Manual Testing Checklist
- [ ] Create new order with date fields
- [ ] Edit existing order dates
- [ ] Filter orders by date range
- [ ] Generate invoices with correct date format
- [ ] Export reports with formatted dates
- [ ] Check all service types (Flight, Hotel, etc.)

## Rollback Plan

If issues arise during migration:

1. **Utility files are isolated**: Can be removed without affecting existing code
2. **Phased migration**: Revert individual components if needed
3. **Database unchanged**: No data migration required
4. **Keep both formats temporarily**: Use feature flags to switch between old and new formatting

```javascript
// Feature flag approach
const USE_NEW_DATE_FORMAT = process.env.USE_NEW_DATE_FORMAT === 'true';

const formatDate = (date) => {
  if (USE_NEW_DATE_FORMAT) {
    return formatDateForDisplay(date);
  }
  return moment(date).format('DD-MMM-YY'); // Old format
};
```

## Common Issues and Solutions

### Issue 1: Timezone Differences
**Solution**: Use the timezone parameter in backend formatter:
```javascript
formatDateForDisplay(date, false, 'Asia/Kolkata');
```

### Issue 2: Date Picker Compatibility
**Solution**: Use formatDateForAPI when setting input values:
```javascript
<input
  type="date"
  value={formatDateForAPI(date)} // Converts DD-MM-YYYY to YYYY-MM-DD for HTML input
/>
```

### Issue 3: Legacy Data Format
**Solution**: Handle multiple formats in parsing:
```javascript
// The utilities already handle multiple formats
const date = parseDateFromDisplay(dateString) || parseDateFromAPI(dateString);
```

## Performance Considerations

1. **Memoization**: The React hook uses useMemo to prevent re-renders
2. **Batch Processing**: Process arrays of dates efficiently with helper functions
3. **Lazy Loading**: Import date utilities only where needed
4. **Caching**: Consider caching formatted dates for frequently accessed data

## Next Steps

1. **Phase 1** (Immediate): Deploy utility files and test independently
2. **Phase 2** (Week 1): Update backend controllers and add middleware
3. **Phase 3** (Week 2): Migrate frontend components gradually
4. **Phase 4** (Week 3): Update templates and reports
5. **Phase 5** (Week 4): Full testing and validation
6. **Phase 6** (Week 5): Production deployment with monitoring

## Monitoring

After deployment, monitor:
- API response times
- Date parsing errors in logs
- User feedback on date display
- Database query performance

## Support

For issues during migration:
1. Check this guide first
2. Review the utility function documentation
3. Test with the provided examples
4. Use the rollback plan if needed