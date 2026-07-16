# Customer Position Report - Technical Documentation

**Version**: 1.0
**Date**: March 2026
**Author**: System Analysis
**Status**: Stable

---

## Overview

The Customer Position Report provides a summary view of each customer's financial position, showing opening balance, sales invoices, refunds (credit notes), receipts, deposits, payments to customer, journal voucher adjustments, and net balance. One row per customer.

---

## Technical Architecture

### Frontend Component

**File**: `psfront/src/pages/Report/CustomerPositionReport.jsx`

**Filters**:
- Document Date: =, <, <=, >, >=, <>, between (on `created_at` or `adjustment_date`)
- Branch: isNotBlank, isBlank, isEqual, between
- Customer: isNotBlank, isBlank, isEqual, between
- Adjustment Date Mode: checkbox (Posted to Ledger)
- Include Raised Invoices: checkbox (`includeRaised`, default OFF) — see [Include Raised Invoices](#include-raised-invoices-2026-06-08)

**Output**: PDF or Excel

**Filter retention & inline preview**:
- Selected filters are retained after generating (module-level `cachedFilter`); returning to the page keeps prior selections. A full page refresh resets to `DEFAULT_FILTER`.
- PDF is rendered inline below the filters (iframe → `{VITE_API_URL}/document/view/{report_number}`) with a **Preview** button (opens the full PDF in a new tab). No navigation away from the page. Excel still downloads and clears the inline PDF.

### Backend Controller

**File**: `psback/controllers/report.controller.js`
**Function**: `getCustomerPositionReport` (line 6474)
**Route**: `POST /api/report/getCustomerPositionReport`
**Permission**: `Customer-Position-Detail-Report`

### PDF Template

**File**: `psback/views/pages/reports/report2.ejs`
**Format**: A3 Landscape, 5mm margins

### Helper Service

**File**: `psback/services/customer_balance_calculator.js`
**Function**: `calculateCustomerBalance` — handles Air vs Hotel vs Other invoice calculations

---

## Report Layout

### Row Structure

One row per customer with summary totals. Customers with **no activity at all**
(every money column is `0.00`) are **excluded** from the report — see
[Hide Zero-Activity Customers](#hide-zero-activity-customers-2026-06-08).

| Column | Key | Description |
|--------|-----|-------------|
| Customer No. | `customer_no` | Customer number |
| Customer Name | `customer_name` | Customer name |
| Opening Balance B/F | `opening_balance` | Balance before report period |
| ADD: Sales Invoice | `sales_invoice` | Period invoice total (ordered invoices) |
| ADD: Opening Invoices | `opening_invoices` | Period imported opening invoice total |
| LESS: Refunds | `refunds` | Period credit note total |
| LESS: Receipts | `receipts` | Period receipts + deposits combined |
| ADD: Payments | `payments` | Payment settlements TO customer (via credit notes) |
| ADD: JV Debit | `jv_debit` | Manual JE debit entries |
| LESS: JV Credit | `jv_credit` | Manual JE credit entries |
| Net Balance | `net_balance` | Final calculated balance |

### Grand Total Row

Sum of all numeric columns across all customers.

---

## Calculation Logic

### Invoice Total (Per Customer)

Uses `calculateCustomerBalance` helper:

**Air Services**:
```
netPerPassenger = farePerPax + taxesPerPax - discountPerPax - rebatePerPax + tFeePerPax + sstPerPax + suppFeePerPax
netPerPassengerPKR = netPerPassenger × exchangeRate
invoiceTotal = netPerPassengerPKR × numberOfPassengers
```
`suppFeePerPax` = `invoice.customer_supplementary_fee` divided by the number of passengers
(the supplementary fee is a per-invoice total, split per passenger like T.Fee and SST).

> **Per-passenger invoices (2026-07-14)**: if the invoice has a `passenger_id` (service invoiced
> passenger-by-passenger), `numberOfPassengers` is the count of matching `service_passengers`
> rows (normally 1), so the invoice is counted once instead of once per passenger. The period
> invoice SQL selects `i.passenger_id` plus an `invoice_passenger_count` subquery for this. If
> the invoice's passenger is no longer on the service, it falls back to the full passenger count.
> Same rule as the Customer Account Statement (its doc, section 10.0.12).

**Hotel Services**:
```
totalAmount = totalPrice + (SST if not already included)
invoiceTotal = totalAmount × exchangeRate
```

**Other Services**:
```
totalAmount = totalPrice + (SST if not included)
invoiceTotal = totalAmount × exchangeRate
```

### Opening Balance

Computed by the **shared helper** `psback/services/customerOpeningBalance.service.js`
(`getCustomerOpeningBalances`) — the **same** helper the Customer Account Statement uses,
so the two reports always show the identical Opening Balance B/F.

```
openingBalance = historicalInvoices
               - historicalCreditNotes
               - historicalReceipts (GL account payments only)
               - historicalDeposits (full values)
               + historicalPayments (payments TO customer)
               + historicalJvDebit
               - historicalJvCredit
```

Historical data = all transactions BEFORE the report's startDate. The helper applies the
Air per-passenger valuation and the foreign-currency credit-note conversion, so opening
balance reconciles with the previous period's Net Balance. Since 2026-07-14 the helper
counts **all** live invoices of a service (previously only the first — passenger-wise
invoices after the first were missed).

Pre-period **opening invoices** are added on top of this (folded into Opening Balance B/F).

### Net Balance

```
netBalance = openingBalance
           + periodInvoiceTotal
           + periodOpeningInvoices (imported opening invoices)
           - periodRefunds (credit notes only)
           - periodReceipts (receipts + deposits combined)
           + periodPayments (payments TO customer)
           + jvDebitTotal
           - jvCreditTotal
```

### Key Rules

- **Service refunds** from refunds table are **completely excluded** — only credit notes count as refunds
- **Deposits** use **full deposit values**, not remaining balance
- **Receipts** only count **GL account payments** (deposit payments excluded to avoid double-counting)
- **Credit notes**: Foreign currency ones use `amount_base` (PKR converted); PKR ones (currency_id = 110) use `amount` directly
- **Journal entries**: Only `Manual JE` batch type, status `Open` or `Posted`, matched by `gl_entity_id` = customer_id
- **Supplementary fee**: `invoice.customer_supplementary_fee` is included in the Sales Invoice total and Net Balance. For Air invoices it is split per passenger and added to the per-passenger Net; for Hotel/Other it is already part of `total_price`. Opening balance (historical) values invoices from `total_price`, which already includes the supplementary fee.

---

## Data Sources

### Primary Tables

| Table | Purpose |
|-------|---------|
| `customers` | Customer info (customer_number, customer_name) |
| `users` | Company scoping |
| `customer_deposits` | Deposit records (amount, currency, status) |
| `receipt_settlements` | Receipt/payment records from customers |
| `receipt_settlement_payments` | Payment method details (GL account vs deposit) |
| `orders` | Links customers to services |
| `services` | Service records (service_type, ticket_issue_date) |
| `invoices` | Invoice records (price, quantity, discount, rebate, fees, taxes) |
| `invoice_taxes` | Per-invoice tax amounts |
| `credit_notes` | Credit note records |
| `payment_settlements` | Payments made TO customer (via credit notes) |
| `journal_entries` | Manual JE entries (gl_entity_id = customer_id) |
| `journal_batches` | JE batch info (batch_type, status) |
| `invoice_settings` | Company setting for ticket_issue_date usage |
| `currencies` / `currency_codes` | Exchange rates (to PKR) |
| `service_types` | Service type (Air, Hotel, etc.) |
| `branches` | Branch filtering |

### Query Strategy (7-Step Approach)

1. **Step 1**: Get customers (basic data with company scope)
2. **Step 2**: Get customer deposits (with currency/exchange rate, scoped by company_code)
3. **Step 3**: Get receipt settlements (with payment details)
4. **Step 4**: Get orders with services (raw SQL, with branch filter)
5. **Step 5**: Get invoices (raw SQL) + invoice taxes
6. **Step 6**: Get credit notes + historical data (invoices, receipts, credit notes, deposits, JEs before startDate)
7. **Step 7**: Get payment settlements (payments TO customer)

---

## Adjustment Date Mode

When `adjustmentDateMode = true`:
- If `adjustment_date` is NULL (unposted): include only if `created_at` is in date range
- If `adjustment_date` exists (posted): include only if `adjustment_date` is in date range

When `adjustmentDateMode = false`:
- Filter by `created_at` only

Applies to: deposits, receipts, and their historical counterparts.

---

## Ticket Issue Date Logic

Controlled by `invoice_settings.ticket_issue_date_in_invoice` per company:
- If enabled AND service is Air AND `ticket_issue_date` exists: use `ticket_issue_date` for date filtering
- Otherwise: use `invoice_date`
- Applied at JavaScript level (not SQL) when enabled
- Affects both period invoices and historical invoices (opening balance)

---

## Company Scoping

Scoped via `customer → user → where: { company_code: req.user.company_code }`.

All currency exchange rate queries (deposits, historical deposits) also filter by `company_code` to ensure correct PKR conversion rates are used per company.

---

## Report Metadata

```javascript
{
  report_number: "TPCP" + timestamp,
  report_type: "customer-position-report",
  file_type: "xlsx" | "pdf"
}
```

---

## Output Formats

### Excel
- Worksheet: "Customer Position Report"
- Frozen rows: 7 (header section)
- Amount columns: right-aligned, `#,##0.00` format
- Column width: 15px fixed
- Grand total row: bold, light gray background

### PDF
- Template: `report2.ejs` (A3 Landscape)
- Font: Times New Roman, 8px
- Number format: `formatNumberWithCommas` (2 decimal places)
- Header repeats on each page

---

## Code References

| File | Purpose |
|------|---------|
| `psfront/src/pages/Report/CustomerPositionReport.jsx` | Frontend UI |
| `psback/controllers/report.controller.js` (line 6474) | Controller logic |
| `psback/services/customer_balance_calculator.js` | Invoice calculation helper |
| `psback/views/pages/reports/report2.ejs` | PDF template |
| `psback/routes/report.route.js` (line 56) | Route definition |
| `psfront/src/api/report.js` | API client |

---

## Recent Updates

### Exclude Void-Leftover Raised Invoices (2026-06-16)

**Problem**: With **Include Raised** ON, a customer's **ADD:Sales Invoice** total doubled (example: KH0152 showed 52,622 instead of 26,311).

**Root cause**: Voiding an invoice (`_voidInvoiceDocumentInTx` in `invoice.controller.js`) auto-creates a **Raised** replacement draft on the **same service**. With Include Raised ON, that leftover draft was counted on top of the service's real Printed invoice, double-counting it. (The Void row itself was already correctly excluded — it is the auto-created Raised replacement that leaked in.)

**Rule**: When `includeRaised` is ON, a **Raised** invoice is dropped if its **service also has a `Void` invoice** (i.e. it is a void replacement). Genuine Raised drafts (services with no void) still count, so the feature is unchanged for them. Discriminating on the missing invoice number was rejected — un-printed invoices legitimately have no number, so that would have dropped genuine drafts too.

**Where**: `psback/controllers/report.controller.js` → `getCustomerPositionReport`, right after the period-invoice fetch/date-filter (Step 5). One extra query gets the set of `service_id`s that have a `Void` invoice (scoped to the report's `serviceIds`); period `invoices` are then filtered to drop `status = 'Raised'` rows on those services.

**Not changed**: The **Opening Balance B/F** path uses the shared `getCustomerOpeningBalances` helper (also used by the Customer Account Statement). A void+Raised that occurred *before* the report period could still double-count there; that shared helper was left untouched to avoid affecting the Account Statement — flagged for a separate decision.

### Single-Date Heading (2026-06-10)

**Goal**: When the report is run for a single day (From = To, e.g. the `=` operator or a between with the same date twice), the PDF heading showed "Document Date: 2026-06-01 To 2026-06-01". It now shows just "Document Date: 2026-06-01". A real range still shows "2026-06-01 To 2026-06-05".

**Where**: `psback/controllers/report.controller.js` → `getCustomerPositionReport`, the `header.documentDate` line — if `startMoment` and `endMoment` are the same day, only the single date is printed.

**Not changed**: the "Document Date:" label itself (it lives in the shared `report2.ejs` template used by 7 reports) and the Excel output (which does not print the document date in its header).

### Include Raised Invoices (2026-06-08)

**Goal**: Mirror the new Customer Account Statement option here (this report is the summary version). When enabled, un-printed **Raised** invoices are counted in the customer's totals.

**Behaviour**:
1. New checkbox **"Include Raised invoices"** (`includeRaised`, default **OFF**) on the filter screen. OFF = today's behaviour (only `Printed`, `Settled`, `Partially Settled`).
2. When ON, `Raised` is added to the invoice status list, so Raised invoices flow into **ADD:Sales Invoice** and **Net Balance**, and (for Raised invoices dated before the period) into **Opening Balance B/F**, keeping reconciliation with the Customer Account Statement.
3. No Status column is shown — this report is one summary row per customer, with no per-document listing.

**Where**:
- `psback/controllers/report.controller.js` → `getCustomerPositionReport`: `includeRaised` read from the request; applied to the period invoice status list (the raw-SQL `i.status IN (...)` clause); passed into the shared `getCustomerOpeningBalances` helper for the opening balance.
- `psback/services/customerOpeningBalance.service.js`: `includeRaised` param (default **false**) — already shared with the Customer Account Statement.
- `psfront/src/pages/Report/CustomerPositionReport.jsx`: checkbox added.

**Caveat**: a Raised invoice has no frozen exchange rate (saved only at print time), so foreign-currency Raised invoices use the **live** rate. PKR invoices are unaffected.

**Not changed**: the early-exit path used when a company has **zero ordered services** still lists all customers without raised invoices (rare scenario, left as-is). The dead historical-invoice query (kept from the 2026-05-19 opening-balance refactor) was left untouched since its result is not used in output.

### Hide Zero-Activity Customers (2026-06-08)

**Goal**: When the report is run without a customer filter, it listed **every** customer in the company — including ones with no invoices, receipts, deposits, or balance (all columns `0.00`). Those empty rows are now hidden.

**Rule**: A customer row is removed only when **every** money column is `0.00` (opening balance, sales invoice, opening invoices, refunds, receipts, payments, JV debit, JV credit, net balance). A customer whose real activity nets to zero (e.g. invoice 1,000 and receipt 1,000) still has non-zero columns and **remains** in the report.

**Where**: `psback/controllers/report.controller.js` → `getCustomerPositionReport`. After the per-customer rows (`data1`) are built and before the Grand Total is computed, `data1` is filtered to keep only rows with at least one non-zero money column. The Grand Total then sums only the visible rows.

**Not changed**: The separate early-exit path used only when a company has **zero ordered services** still lists all customers (rare scenario, left as-is).

### Opening Balance — Shared with Customer Account Statement (2026-05-19)

**Problem**: The Customer Position Report's "Opening Balance B/F" did not match the Customer Account Statement for the same customer and dates (example: TAH690, Apr 2026 — Position Report 3,800,696.42 vs Account Statement 3,782,591.00, off by 18,105.42). Every other line matched; only the opening balance was wrong, which also threw off Net Balance.

**Root cause**: The two reports computed opening balance with **two different code paths**. The Position Report used the older `calculateCustomerBalance` helper, which lacked the Air per-passenger valuation fix and the foreign-currency credit-note conversion fix that the Account Statement has.

**Solution Implemented**:
1. The Account Statement's opening-balance calculation was extracted **verbatim** into a shared helper — `psback/services/customerOpeningBalance.service.js` → `getCustomerOpeningBalances(...)`.
2. The Customer Account Statement now calls this helper (pure code move — no behaviour change).
3. The Customer Position Report now calls the **same** helper for "Opening Balance B/F" instead of `calculateCustomerBalance`'s opening balance. `calculateCustomerBalance` is still used for the period figures (refunds, receipts, deposits).
4. The two reports can no longer drift apart — one source of truth.

**Files Modified**:
- `psback/services/customerOpeningBalance.service.js` — new shared helper.
- `psback/controllers/report.controller.js` — `getCustomerAccountStatementReport` (inline block replaced with a helper call) and `getCustomerPositionReport` (opening balance now from the helper).

**Note**: in the Position Report, the old `historicalPaymentsTotal` / `historicalJvDebitTotal` / `historicalJvCreditTotal` variables are now unused (the helper computes those internally); left in place to keep the change minimal.

### Add Opening Invoices Column (2026-05-19)

**Goal**: Show imported **opening invoices** as their own **"ADD:Opening Invoices"** column, so the Customer Position Report lines up column-by-column with the Customer Account Statement.

**What it was before**:
1. The report builds its invoice list from `orders INNER JOIN services`.
2. Opening invoices have **no order**, so they were **completely excluded** — not in "ADD:Sales Invoice", not in "Opening Balance B/F".
3. Every customer with opening invoices was understated.

**Solution Implemented**:
1. Reuses `getOpeningInvoicesForCustomers(companyCode, customerNumbers)` from `psback/services/openingInvoice.service.js`; opening invoices fetched once and grouped by customer number.
2. A per-customer opening invoices total is built with the report date filter (`dayKey`): pre-period ones fold into Opening Balance B/F, in-period ones go to the new column. PKR = `total_price × exchange_rate`.
3. New `opening_invoices` ("ADD:Opening Invoices") column added right after "ADD:Sales Invoice"; included in the `net_balance` formula.
4. Excel: `_invoices` added to the amount-column regex so the column is right-aligned and number-formatted. PDF (`report2.ejs`) renders the column automatically (dynamic template — no template edit).

**Files Modified**:
- `psback/controllers/report.controller.js` (`getCustomerPositionReport`) — opening-invoice fetch + grouping, per-customer opening invoices total, `opening_invoices` column, opening balance + `net_balance` formula, Excel amount-column regex.

**Known limitation**: the early-exit path used when a company has **zero ordered services** does not include opening invoices (rare scenario).

---

**Document End**
