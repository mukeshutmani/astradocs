# Multi-Sector Flight Import - Setup Guide

## Quick Start: 3-Step Setup

This guide shows you how to configure your LCC template to handle multi-sector flights (connecting flights with multiple legs).

---

## Step 1: Prepare Your Excel File

### Example: 2-Sector Flight Data

Create an Excel file with these columns for a route like **KHI-ISB-LHE** (2 sectors):

| Column Name | Description | Example Value |
|-------------|-------------|---------------|
| PNR | Booking reference | ABC123 |
| PAX_NAME | Passenger name | John Doe |
| Tkt No. | Ticket number | 0841234567890 |
| Customer No | Customer code | CUST001 |
| Supplier No | Supplier code | SUPP0001 |
| Sector | Full route | KHI-ISB-LHE |
| Basic Fare | Base price | 15000 |
| **DepDate** | **Sector 1 departure date** | **2025-10-25** |
| **DepTime** | **Sector 1 departure time** | **08:00** |
| **ArvDate** | **Sector 1 arrival date** | **2025-10-25** |
| **ArvTime** | **Sector 1 arrival time** | **09:30** |
| **Flight** | **Sector 1 flight number** | **PK301** |
| **DepDate2** | **Sector 2 departure date** | **2025-10-25** |
| **DepTime2** | **Sector 2 departure time** | **11:00** |
| **ArvDate2** | **Sector 2 arrival date** | **2025-10-25** |
| **ArvTime2** | **Sector 2 arrival time** | **12:15** |
| **Flight2** | **Sector 2 flight number** | **PK650** |

### Sample Excel Data

```
PNR     | PAX_NAME  | Sector        | DepDate    | DepTime | ArvDate    | ArvTime | DepDate2   | DepTime2 | ArvDate2   | ArvTime2 | Flight | Flight2
--------|-----------|---------------|------------|---------|------------|---------|------------|----------|------------|----------|--------|--------
ABC123  | John Doe  | KHI-ISB-LHE   | 2025-10-25 | 08:00   | 2025-10-25 | 09:30   | 2025-10-25 | 11:00    | 2025-10-25 | 12:15    | PK301  | PK650
```

### Key Points:
- **Route format**: Use hyphens between city codes (KHI-ISB-LHE)
- **Number of sectors**: 3 cities = 2 flight sectors (2 legs)
- **Numbered columns**: Add `2` suffix for second sector, `3` for third, etc.
- **Date format**: YYYY-MM-DD or DD/MM/YYYY
- **Time format**: HH:MM or HH:MM:SS

---

## Step 2: Configure Template Mappings

### Option A: Using the UI

1. Navigate to: **IUR → Upload LCC → Create/Edit Template**
2. For your template, add these field mappings:

#### Base Mappings (Always Required)

| Excel Column | Maps To (service_column) | Mandatory | Handling |
|--------------|-------------------------|-----------|----------|
| PNR | pnr | Yes | direct_mapping |
| PAX_NAME | passenger_name | Yes | direct_mapping |
| Tkt No. | ticket_number | Yes | direct_mapping |
| Customer No | customer_code | Yes | direct_mapping |
| Supplier No | supplier_code | No | direct_mapping |
| Sector | route | Yes | direct_mapping |
| Basic Fare | price | Yes | direct_mapping |

#### Sector 1 Date/Time Mappings

| Excel Column | Maps To (service_column) | Mandatory | Handling |
|--------------|-------------------------|-----------|----------|
| DepDate | departure_date | No | direct_mapping |
| DepTime | departure_time | No | direct_mapping |
| ArvDate | arrival_date | No | direct_mapping |
| ArvTime | arrival_time | No | direct_mapping |
| Flight | flight_number | No | direct_mapping |

#### Sector 2 Date/Time Mappings (ADD THESE!)

| Excel Column | Maps To (service_column) | Mandatory | Handling |
|--------------|-------------------------|-----------|----------|
| **DepDate2** | **departure_date2** | **No** | **direct_mapping** |
| **DepTime2** | **departure_time2** | **No** | **direct_mapping** |
| **ArvDate2** | **arrival_date2** | **No** | **direct_mapping** |
| **ArvTime2** | **arrival_time2** | **No** | **direct_mapping** |
| **Flight2** | **flight_number2** | **No** | **direct_mapping** |

#### Sector 3 Date/Time Mappings (If Needed)

| Excel Column | Maps To (service_column) | Mandatory | Handling |
|--------------|-------------------------|-----------|----------|
| DepDate3 | departure_date3 | No | direct_mapping |
| DepTime3 | departure_time3 | No | direct_mapping |
| ArvDate3 | arrival_date3 | No | direct_mapping |
| ArvTime3 | arrival_time3 | No | direct_mapping |
| Flight3 | flight_number3 | No | direct_mapping |

### Option B: Using API/Code

```javascript
// Example: Creating template mappings programmatically
const mappings = [
    // ... other mappings ...

    // Sector 1 (Primary)
    { document_column: 'DepDate', service_column: 'departure_date', mandatory: 'optional' },
    { document_column: 'DepTime', service_column: 'departure_time', mandatory: 'optional' },
    { document_column: 'ArvDate', service_column: 'arrival_date', mandatory: 'optional' },
    { document_column: 'ArvTime', service_column: 'arrival_time', mandatory: 'optional' },
    { document_column: 'Flight', service_column: 'flight_number', mandatory: 'optional' },

    // Sector 2
    { document_column: 'DepDate2', service_column: 'departure_date2', mandatory: 'optional' },
    { document_column: 'DepTime2', service_column: 'departure_time2', mandatory: 'optional' },
    { document_column: 'ArvDate2', service_column: 'arrival_date2', mandatory: 'optional' },
    { document_column: 'ArvTime2', service_column: 'arrival_time2', mandatory: 'optional' },
    { document_column: 'Flight2', service_column: 'flight_number2', mandatory: 'optional' },

    // Sector 3 (if needed)
    { document_column: 'DepDate3', service_column: 'departure_date3', mandatory: 'optional' },
    { document_column: 'DepTime3', service_column: 'departure_time3', mandatory: 'optional' },
    { document_column: 'ArvDate3', service_column: 'arrival_date3', mandatory: 'optional' },
    { document_column: 'ArvTime3', service_column: 'arrival_time3', mandatory: 'optional' },
    { document_column: 'Flight3', service_column: 'flight_number3', mandatory: 'optional' },
];

// Create mappings in database
await models.template.bulkCreate(mappings, { transaction });
```

---

## Step 3: Import and Verify

### Import Process

1. Go to: **IUR → Upload LCC**
2. Select your configured template
3. Upload your Excel file
4. Click **Preview** to verify parsing (recommended!)
5. Review the preview - you should see:
   - Multiple flight segments per service
   - Correct dates/times for each sector
6. Click **Import** (with SSE for real-time progress)

### What Happens During Import

For route **KHI-ISB-LHE** with proper date/time mappings:

```
📦 Service Created: ID 1001, PNR ABC123, Description "LCC Import - KHI-ISB-LHE - ABC123"

✈️ Flight Sector 1 Created:
   - Route: KHI → ISB
   - Departure: 2025-10-25 08:00
   - Arrival: 2025-10-25 09:30
   - Flight: PK301

✈️ Flight Sector 2 Created:
   - Route: ISB → LHE
   - Departure: 2025-10-25 11:00
   - Arrival: 2025-10-25 12:15
   - Flight: PK650

👤 Passenger Created: John Doe, Ticket 0841234567890
```

### Verification in Database

After import, check the `service_flight` table:

```sql
SELECT
    sf.id,
    cf.city_name as from_city,
    ct.city_name as to_city,
    sf.departure_date,
    sf.arrival_date,
    sf.flight_number
FROM service_flight sf
JOIN city cf ON sf.city_from = cf.id
JOIN city ct ON sf.city_to = ct.id
WHERE sf.service_id = 1001
ORDER BY sf.id;
```

Expected result:
```
id  | from_city | to_city | departure_date      | arrival_date        | flight_number
----|-----------|---------|---------------------|---------------------|-------------
1   | Karachi   | Islamabad | 2025-10-25 08:00:00 | 2025-10-25 09:30:00 | PK301
2   | Islamabad | Lahore    | 2025-10-25 11:00:00 | 2025-10-25 12:15:00 | PK650
```

---

## Common Scenarios

### Scenario 1: International Multi-Sector (3 Sectors)

**Route**: KHI-DXB-LHR-CDG (Karachi → Dubai → London → Paris)

**Excel Setup**:
```
Sector: KHI-DXB-LHR-CDG
DepDate: 2025-10-25  | DepTime: 02:00  | ArvDate: 2025-10-25  | ArvTime: 05:00
DepDate2: 2025-10-25 | DepTime2: 09:00 | ArvDate2: 2025-10-25 | ArvTime2: 14:00
DepDate3: 2025-10-26 | DepTime3: 08:00 | ArvDate3: 2025-10-26 | ArvTime3: 09:30
```

**Result**: 3 separate flight records with accurate times for each leg

### Scenario 2: Domestic Multi-City (2 Sectors)

**Route**: KHI-ISB-LHE

**Excel Setup**:
```
Sector: KHI-ISB-LHE
DepDate: 2025-10-25  | DepTime: 08:00  | ArvDate: 2025-10-25  | ArvTime: 09:30
DepDate2: 2025-10-25 | DepTime2: 11:00 | ArvDate2: 2025-10-25 | ArvTime2: 12:15
```

**Result**: 2 separate flight records for domestic connection

### Scenario 3: No Specific Dates for Sector 2+ (Fallback Mode)

**Route**: KHI-DXB-LHR

**Excel Setup** (only sector 1 dates):
```
Sector: KHI-DXB-LHR
DepDate: 2025-10-25  | DepTime: 02:00  | ArvDate: 2025-10-25  | ArvTime: 05:00
(No DepDate2, DepTime2, etc.)
```

**Result**:
- Sector 1: Oct 25 02:00 → Oct 25 05:00 ✓
- Sector 2: Oct 26 02:00 → Oct 26 05:00 (auto-incremented by 1 day)

---

## Troubleshooting

### Problem: Only 1 Flight Record Created (Expected 2+)

**Symptoms**: Route is "KHI-ISB-LHE" but only KHI→ISB flight is created

**Causes & Solutions**:
1. **Route format issue**
   - ❌ Wrong: "KHI ISB LHE" (spaces)
   - ❌ Wrong: "KHI,ISB,LHE" (commas)
   - ✅ Correct: "KHI-ISB-LHE" (hyphens)

2. **Database constraint**
   - Check for errors in import logs
   - Verify all cities exist or can be created

### Problem: Sector 2 Has Same Date as Sector 1

**Symptoms**: Both sectors show Oct 25 08:00 departure

**Causes & Solutions**:
1. **Missing DepDate2 mapping**
   - Verify template has `departure_date2` mapping
   - Check Excel column name matches exactly: "DepDate2"

2. **Empty values in Excel**
   - Ensure DepDate2 cell is not empty
   - Check for hidden characters or formatting issues

3. **Column name mismatch**
   - Template says: `DepDate2`
   - Excel says: `Dep Date 2` (space) ❌
   - Must match exactly!

### Problem: Date Format Not Recognized

**Symptoms**: Dates appear as "Invalid Date" or use current date

**Causes & Solutions**:
1. **Supported formats**:
   - ✅ YYYY-MM-DD (2025-10-25)
   - ✅ DD/MM/YYYY (25/10/2025)
   - ✅ MM/DD/YYYY (10/25/2025)
   - ❌ DD-MM-YYYY (25-10-2025) - May not work

2. **Excel date as number**
   - Format cells as "Date" not "General"
   - Or use text format with explicit date string

### Problem: Times Not Parsing

**Symptoms**: All flights show 00:00 or current time

**Causes & Solutions**:
1. **Time format in Excel**:
   - ✅ HH:MM (08:00, 14:30)
   - ✅ HH:MM:SS (08:00:00)
   - ❌ "8am" or "2:30 pm" - Text format won't parse

2. **Excel time as number**:
   - Format cells as "Time" not "General"
   - Use 24-hour format for consistency

---

## Advanced: 4+ Sectors

For very complex itineraries with 4+ sectors, simply continue the pattern:

```javascript
// Sector 4
{ document_column: 'DepDate4', service_column: 'departure_date4', mandatory: 'optional' },
{ document_column: 'DepTime4', service_column: 'departure_time4', mandatory: 'optional' },
{ document_column: 'ArvDate4', service_column: 'arrival_date4', mandatory: 'optional' },
{ document_column: 'ArvTime4', service_column: 'arrival_time4', mandatory: 'optional' },
{ document_column: 'Flight4', service_column: 'flight_number4', mandatory: 'optional' },

// Sector 5
{ document_column: 'DepDate5', service_column: 'departure_date5', mandatory: 'optional' },
// ... and so on
```

**Practical Limit**: Most airlines don't have itineraries with more than 4-5 sectors, but the system supports unlimited sectors.

---

## Testing Checklist

Before going to production, test your template:

- [ ] Single-sector route works correctly
- [ ] 2-sector route creates 2 flight records
- [ ] 3-sector route creates 3 flight records
- [ ] Dates/times are correctly assigned to each sector
- [ ] Different flight numbers per sector
- [ ] Preview mode shows correct parsing
- [ ] Import succeeds without errors
- [ ] Database has correct number of `service_flight` records
- [ ] Dates don't overlap incorrectly
- [ ] PNR grouping works (all sectors in same order)

---

## Quick Reference: Service Column Names

| Sector | Departure Date | Departure Time | Arrival Date | Arrival Time | Flight Number |
|--------|---------------|---------------|--------------|--------------|---------------|
| 1 | `departure_date` | `departure_time` | `arrival_date` | `arrival_time` | `flight_number` |
| 2 | `departure_date2` | `departure_time2` | `arrival_date2` | `arrival_time2` | `flight_number2` |
| 3 | `departure_date3` | `departure_time3` | `arrival_date3` | `arrival_time3` | `flight_number3` |
| 4 | `departure_date4` | `departure_time4` | `arrival_date4` | `arrival_time4` | `flight_number4` |
| N | `departure_dateN` | `departure_timeN` | `arrival_dateN` | `arrival_timeN` | `flight_numberN` |

**Alternative names also supported**:
- `DepDate2`, `DepTime2`, `ArrvDate2`, `ArvDate2`, `ArrvTime2`, `ArvTime2`, `FlightNo2`

---

## Need Help?

- Review the main documentation: `LCC_TEMPLATE_DOCUMENTATION.md`
- Check import logs for detailed error messages
- Use preview mode before importing
- Contact system administrator if issues persist

---

**Last Updated**: October 2025
**Version**: 1.0
