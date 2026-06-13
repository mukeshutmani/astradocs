# Order Service Section

**Version**: 1.0
**Date**: 2026-06-13
**Status**: Stable
**Scope**: The "Add Service(s) / Fee" + service table area of the Order Detail page.

---

## Overview

The Order Service section is where the actual things being sold on an order are
added and managed — air tickets, hotels, tours, visas, etc. — plus their pricing,
documents, settlements, and per-service actions (transfer / delete).

**Main file**: `psfront/src/pages/Order/.. ` → rendered inside
`psfront/src/pages/OrderDetail.jsx` (service table ~lines 3150–3700, summary ~4834).

---

## 1. Add Service(s) / Fee

A row of buttons, one per service type. Each opens the add-service form for that
type at `/order/:id/add-service` (filtered by type):

- Air, Hotel, Tour Package, Car Transfer, Car Rental, Train, Insurance, Cruise,
  Visa, Miscellaneous, Umrah, Hajj, and a **Fee** button.
- A separate **IUR** button navigates to `/order/:id/iur`.

Component: `ServicesDropdown` (one per type), driven by the `serviceTypes` list.

---

## 2. Tabs

`Tabs` with: **Service** (the list), **Itinerary**, **Document**,
**Receipt / Payment**, **Order History**.

---

## 3. Service Table — Columns

| Column | Source / meaning |
|--------|------------------|
| Checkbox | Select for bulk actions (e.g. bulk transfer / delete) |
| Description / Itinerary | `service_type.product_code - product_description`; "Group" badge if grouped; passenger-type filter shown when not "ALL". Click → service detail page |
| Service Description | `service.description` (else "N/A") |
| PNR | `service.pnr` |
| Status | `service.status` |
| Supplier / Reference | `supplier.supp_name` + optional `supplier_reference` |
| Qty | Passenger/unit count (Visa uses passenger count) |
| Total Price | **What the customer is charged**, in PKR (see §4) |
| Total Cost | **What is paid to the supplier**, in PKR (see §4) |
| Documents | `ServiceDocumentsModal` — badge with count, opens documents pop-up |
| Actions | Transfer (move service to another order) + Delete (red bin) |

---

## 4. Price & Cost Calculation (important)

Both **Total Price** and **Total Cost** are **recomputed in the browser** on each
load (including currency conversion to PKR) — they are not read blindly from the
database. If a number ever looks off, it is usually an exchange-rate/calculation
issue, not a stored-value issue.

**Total Price (per service, from the active invoice):**
1. `price_with_markup = base_price + markup`
2. `discount_amount = round2(price_with_markup × discount% / 100)`
3. `rebate_amount = round2(price_with_markup × rebate% / 100)`
4. `total_unit_before_tax = price_with_markup − discount_amount − rebate_amount`
5. `+ taxes`, then `× quantity`, `+ transaction_fee + SST + customer_supplementary_fee`
6. `× priceExchangeRate` → PKR.

(For Hotels, `invoice.total_price` is used directly, since `invoice.price` is per-room.)

**Total Cost (per service, from the active cost):**
1. `commission_amount = published_rate × commission% / 100`
2. `net_rate = published_rate − commission_amount`, `+ extra_charges (free_of_cost)`
3. `+ cost taxes`, `+ WHT (commission × sst%)` → `cost_per_unit`
4. `× quantity`, then `× costExchangeRate` → PKR.

Exchange rates: the **saved** rate on the cost/invoice (including voided) is used
first, falling back to the live currency table.

> **Discount precision (2026-06-12)**: typing a discount *amount* derives the
> percent at 6-decimal precision so the exact typed amount sticks. See
> `COST_CALCULATION_SYSTEM_ANALYSIS.md`.

---

## 5. Summary (All values in PKR)

A single totals row across all services: Charge Type, Price, Cost, SST, Profit,
Margin %, Payment, Receipt, Pending Refund, Debtor Balance.

---

## 6. Actions — Transfer & Delete

- **Transfer**: `ServiceTransfer` (single) / `ServiceTransferBulk` (selected) — moves
  service(s) to another order.
- **Delete**: opens `ServiceDeletionPreview`, which calls the backend preview, shows
  what blocks deletion, and only enables the Delete button when allowed. See §7.

---

## 7. Service Deletion Rule

Decided **per real document** (a service's invoice(s) and its cost that have a
**number**). The whole service is deletable only when **every** numbered document is
deletable **and** no non-void settlement exists.

Only **numbered** documents are considered. A **numberless draft** — the Raised
invoice/cost with no number that every active service carries as its live working
copy — is **ignored** (it has no void action and is discarded with the service).

| # | Document state | Deletable? |
|---|----------------|-----------|
| 1 | Service has **no** numbered invoice/cost (only numberless drafts, or nothing) | ✅ Yes |
| 2 | Numbered invoice/cost in any **live** status (Raised-with-number / Printed / Settled / Partially Settled) | ❌ No — void it first |
| 3 | Numbered invoice/cost **Void** and **never posted to a JE** (`je_generated` is null) | ✅ Yes |
| 4 | Numbered invoice/cost **Void** but **posted to a JE** (`je_generated` is set) | ❌ No |

Plus: any **non-void** payment/receipt settlement blocks.

### The two signals
1. **Document number** = "is this a real document?" A document gets its number when
   printed/raised-as-a-document. The numberless live draft is not a real document and
   never blocks (this is what makes a service with only drafts + voided invoices
   deletable, and avoids a dead-end since the draft cannot be voided in the UI).
2. **`je_generated`** = "is there a permanent ledger record?" JE posting is triggered
   manually. A **live** (non-void) document blocks regardless of JE (it is still
   active). A **voided** document blocks only if it was actually posted to a JE — a
   voided document with no JE left no ledger trail and is safe to delete.
   `je_generated` is `null` until posted, `true` once posted, and `false` after its
   void-reversal is posted; so **`je_generated != null` = has a JE record**.

### Implementation
**Backend** — `psback/controllers/service.controller.js`:
1. `deleteService` — per numbered document: block if status ≠ Void; if Void, block
   only when `je_generated != null`. Invoices keyed on `invoice_number`; cost keyed on
   its `costing` document `document_number`. Numberless drafts skipped. Non-void
   settlements blocked by the checks below.
2. `previewServiceDeletion` — `canDelete` uses the same rule (Invoice/Cost items carry
   `hasDocument` + `je_generated`; block if `hasDocument` and (`status != void` or
   `je_generated != null`); settlement items block when `status != void`). Sets
   `blockReason` and `nonVoidItems`.
3. `bulkDeleteServices` — mirrors the same per-document block.

**Frontend** — `psfront/src/components/ServiceDeletionPreview.jsx`:
- Per-service block and the Delete button are driven by backend `canDelete` /
  `blockReason` / `nonVoidItems` (not `hasDocument` alone), so a blocked document
  no longer shows a misleading "you may proceed" message.
- The blocking table shows Type / Number / **Status** so the user sees why.

### History / rationale
- An earlier version blocked on *any* document number (even voided), which made a
  service with only voided invoices permanently undeletable.
- A revision blocked on *any* Raised document — but the numberless live draft cannot
  be voided in the UI, creating a dead-end.
- Final rule (this version): numberless drafts never block, and a voided numbered
  document blocks only when it carries a posted JE. Verified against service 6525
  (order KHSO00000060): 5 voided invoices with no JE + numberless draft invoice/cost →
  now deletable.
