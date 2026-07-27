# Fee Maintenance Module

Automatic pricing engine for customer charges. It decides how much **Transaction Fee** and
**Markup** to add on top of a service when a booking is created — from rules defined once
per customer fee group, instead of typing fees by hand on every booking.

Last updated: 2026-07-20

---

## 1. Building Blocks (Database)

1. `fee_maintenances` — the **Fee Group** (name + company code). One group per plan, e.g.
   "Pak Suzuki Official". Company-scoped.
2. `maintenance_fee_groups` — the **rules inside a group**. Each rule = matching conditions
   (product codes, airline, class, routing, ticket type, validity, segments) + fee amounts
   (method / percent / fixed / min / max).
3. `is_markup_fee_rule` flag on a rule separates **Markup rules** from **Transaction Fee
   rules**. The two are independent rule types (matched, validated, and applied separately).
4. Side tables per rule: `fee_main_product_codes` (service types covered), `fee_airlines`
   (airlines), `fee_class_airlines` (flight classes).
5. `customers.group_id` links a customer to ONE fee group. A sub-customer (with
   `customer.company` set) inherits the **parent company's** fee group on every fee path.
6. `fee_type_maintenance` is a separate small module: maps a fee code to a GL settle
   account for accounting. It is not part of the matching engine.

## 2. Key Files

| Layer | File | Role |
|---|---|---|
| Engine | `psback/utils/feeRuleValidator.js` | Rule matching (scoring + exclusion), duplicate-rule check, `isAirTicketType` |
| Engine | `psback/utils/transactionFeeCalculator.js` | Fee/markup amount calculation (methods, min/max) |
| Engine | `psback/utils/iurFeeResolver.js` | Shared resolver for import/transfer paths (routing base, class letters, parent fallback) |
| API | `psback/controllers/service.controller.js` → `getTransactionFee` | `GET /api/service/transaction-fee` (order pages) |
| CRUD | `psback/controllers/data.controller.js` | Fee group + rule CRUD, class-codes distinct list, routing↔product guard |
| Import | `psback/controllers/lccImport.controller.js` | LCC import fee application (via shared resolver) |
| UI | `psfront/src/pages/MaintenanceGroups/AddFeeRule.jsx`, `EditFreeRule.jsx` | Rule forms |
| UI | `psfront/src/components/service/Price.jsx`, `HotelPrice.jsx` | Order-side fee display/refetch |
| UI | `psfront/src/hooks/service/useTransactionFee.js`, `services/TransactionFeeService.js` | Fetch + currency handling |

## 3. How Matching Works (core principle)

1. Every criteria field on a rule is a **hard filter**: if the field is set and the service
   does not match it, the rule is completely excluded for that service.
2. If the field is set and the service matches, the rule earns **points** (product +3,
   airline +2, class +2, segments +2, ticket type +2, specific routing +2,
   international/domestic routing +1, origin-base +1, validity +1).
3. An empty field means "don't care" — no block, no points.
4. Highest score wins ("most specific rule wins"). On a tie, the newest rule (highest id)
   wins.
5. Transaction-fee rules and markup rules are matched **separately**. Markup falls back to
   the matched transaction rule's markup fields for legacy combined rules.

## 4. Rule Criteria Reference

1. **Product Code** — rule applies only to the selected service types. Empty = all types.
2. **Airline Codes + Airline mode** — air tickets only. Modes: Include (only listed),
   Exclude (all except listed), BSP (only `airline_codes.bsp_airline = true`), LCC (only
   `lcc = true`). Codes with no mode behave as Include.
3. **Airline Class Codes** — air tickets only. Matched by class **LETTER** (`class_code`,
   e.g. "Y"), NOT by row id — the `airline_class_codes` table stores one row per airline
   for the same letter, so letter matching is the only correct comparison. The rule-form
   dropdown shows one entry per letter ("Y - ECONOMY") using each letter's most common
   description (endpoint `distinct` mode).
4. **Segments** — air tickets only. The rule applies when the entered flight-row count
   EQUALS the rule's Segment To (or From when To is empty). It is an exact match, NOT a
   range ("1 to 3" fires only on exactly 3 segments).
5. **Routing** — air tickets only. All = no restriction; International = at least one
   endpoint outside PK; Domestic = departure country is PK; Specific Routing = departure
   AND arrival city must both match exactly.
6. **Origin As Base + Origin Country** — air tickets only. Active only when BOTH are set:
   flight departure country must equal the selected country.
7. **Ticket Type** — Normal / Reissue are hard filters; "Both" is a wildcard. Manual
   added services and LCC imports count as "normal"; IUR-created reissues are detected from
   the service description prefix; EMD lines are unrestricted.
8. **Validity From/To** — checked against today's date (server clock), inclusive both ends.
   Expired or future rules never apply. Both dates are required on the form.
9. "Air tickets only" = service types 1–4 (Int'l/Domestic × Supplier/BSP,
   `service_types.type = 'Air'`; EMD id 37 is deliberately excluded). Non-air services
   (hotel, car transfer, tours, visa, …) IGNORE criteria 2–6 — their Int'l/Domestic nature
   is already carried by the product code itself.

## 5. Fee Amount Calculation

1. **Percentage** — percent × subtotal (subtotal in the service currency).
2. **Flat / Fixed** — set amount, defined in base PKR; the order form divides by the
   exchange rate for foreign-currency services.
3. **Highest Amount** — computes both percentage and fixed, takes the higher.
4. **Per Quantity** — base fee (percent or fixed) × passenger count (client-side, eligible
   service types only; hotels split by rooms × nights).
5. **Min/Max caps** apply after the method (0 = no limit): raised to min, capped at max.
6. **Markup** uses the same methods but on the **base price** (not the full subtotal), and
   has no min/max caps.

## 6. Rule Form Behaviour (Add / Edit)

1. **Routing auto-sync with Product Code**: all selected products International → routing
   auto-sets to "International"; all Domestic → "Domestic". The contradicting radio is
   disabled. Mixed or neutral products (Visa, EMD, Umrah…) force nothing. A manually chosen
   "Specific Routing" is never overwritten.
2. Contradictions (e.g. Int'l products + Domestic routing) are blocked in three layers:
   disabled radio → submit validation → backend check in create AND update endpoints.
3. **Duplicate protection**: saving a rule identical on EVERY scope field to an existing
   rule of the same type (transaction vs markup) is rejected with a detailed message.
   Overlapping-but-not-identical rules are allowed (score/tie-break resolves them).
4. Specific Routing requires different departure/arrival cities.

## 7. How Fees Reach Orders (all paths use the SAME engine)

1. **Order pages (create)** — `GET /api/service/transaction-fee` computes fee + markup;
   the form auto-fills. Refetch triggers: subtotal change, currency change, airline change,
   **service-type switch** (e.g. Hotel Domestic → Hotel International; a non-matching type
   correctly drops the fee to 0), segments/cities/class changes, quantity (per-quantity).
2. **Order pages (edit)** — the saved fee is never overwritten silently; a "value changed →
   refresh" chip/message appears and the user applies it explicitly.
3. **Manual override** — the user can switch a fee to manual; auto-recalculation then skips
   that field until manual mode is turned off.
4. **Pending PNR → order / IUR auto-BILL** — server-side via `iurFeeResolver`
   (`applyFeeToServiceInvoice` / `computeFeeGroupCharges`).
5. **LCC import** — server-side via the same resolver: full rule criteria, routing/classes
   from the imported flights, parent-company fallback, ticket type "normal". Applies
   transaction fee only (no markup), amounts in PKR.

## 8. Currency

1. Matching/calculation happens in the service currency; a PKR equivalent is returned for
   reporting (PKR = currency id 110).
2. Percentage fees are already in the selected currency (percent of the converted subtotal);
   fixed fees are PKR and get divided by the exchange rate on the form.

## 9. Known Behaviours / Watch-outs

1. Segment From/To is an **exact match** on the To value, not a range (see 4.4). The UI
   labels still say From/To.
2. Two overlapping (non-identical) rules matching the same booking resolve silently by
   score, then newest id. No warning is shown when saving such rules.
3. Validity is evaluated on the **server clock**; if the server runs UTC, rule boundaries
   shift ~5 hours vs Pakistan time on the first/last day.
4. The manual order form saves the client-computed fee; the backend recomputes only on the
   import/transfer paths.
5. Legacy rule columns exist that the form no longer exposes (IATA, region, haul, fare
   type, charge type, comm, booking channel, currency, discount, Calculate On). They are
   inert — rules cannot carry values there via the UI.

## 10. Changelog

### 2026-07-20 — Criteria gating overhaul (fee rules now apply correctly on orders)
1. **All air ticket types gated** — the matcher's "is flight" check covered only type 1
   (Int'l Supplier); airline/class filters were silently skipped for types 2/3/4
   (Domestic Supplier, Int'l/Domestic BSP). Now all four air types are gated via the shared
   `isAirTicketType` helper (EMD unchanged). Fixed a live wrong fee at Pak Suzuki (Economy
   rule 195 beating Business rule 194 on BSP tickets).
2. **Class matching by letter** — rules match the class letter ("Y") instead of the
   per-airline row id, across all paths; service class ids are resolved to letters
   server-side.
3. **Class dropdown sync** — rule forms show one entry per letter with its most common
   description, matching the order page's Class field (`distinct` mode on the
   airline-class-codes endpoint). Fixed Edit-form chips never showing descriptions.
4. **Air-only criteria skip non-air services** — routing, segments, and origin-base no
   longer exclude hotels/car transfers/tours etc. (e.g. "Hotel Domestic + routing Domestic"
   now applies to a domestic hotel booking).
5. **Routing ↔ product sync** on the rule forms + backend contradiction guard (create and
   update).
6. **Order-side type switch** — switching the service Type (Hotel/Air/Car Rental
   Domestic ↔ International) in create mode now refetches the fee; previously the stale fee
   from the old type remained and was saved.
7. **LCC import parity** — full rule criteria loaded, routing/classes from imported
   flights, parent-company fallback, ticket type "normal" (was: only airline criteria
   visible; product/class rules invisible).
