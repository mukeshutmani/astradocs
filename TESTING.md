# Testing Guide — PowerSuite

How automated testing works in this repository, for any developer (or AI assistant)
picking up the project.

Last updated: 2026-07-25

---

## 1. The four layers

| Layer | Location | Runner | Touches a database? | Speed |
|---|---|---|---|---|
| Unit (backend) | `psback/tests/unit/` | Jest | No — in-memory fake | ~2 s |
| Integration (backend) | `psback/tests/integration/` | Jest | Yes — **`astra_test` only** | ~10 s |
| Reconciliation (reports) | `psback/tests/reconciliation/` | Jest | Yes — **live data, read-only** | ~40 s |
| Frontend | `psfront/tests/` | Vitest + jsdom | No — API calls are mocked | ~5 s |

Reconciliation is a fourth layer for **reports only**. An accounting identity means
nothing on toy data, so it runs against the live database — and safety comes from
`tests/reconciliation/setup.js`, which wraps `sequelize.query` and refuses anything
that is not a read, model calls included. It fails closed.

There are no browser-automation (Playwright/Cypress) tests. Frontend tests render
React components in memory with jsdom and run from the console. Real screen checks
are done manually on staging before a production deploy.

Test folders are named after the **feature as the business sees it**, not after the
controller file. Printing and voiding an invoice live in `invoice-document/` on both
sides, even though the backend code sits in `document.controller.js`.

---

## 2. Commands

One command, one argument — on both sides:

```bash
npm test                 # list every target you can run
npm test <target>        # run one
npm test <target> -t H7  # extra arguments go straight to jest / vitest
```

`package.json` holds a **single** test line on each side. Every target lives in
`tests/registry.js` and is executed by `scripts/test.js`. Adding a module or a
report means adding one line to that registry — `package.json` is never touched
again, so it cannot rot into a wall of scripts as the report count grows.

### Backend (`cd psback`)

| Target | What it runs |
|---|---|
| `unit` | Fake-database tests only. No connection, safe anywhere. |
| `integration` | Real `astra_test` database. |
| `all` | Unit + integration (reconciliation excluded on purpose). |
| `invoice` | Invoice module — unit + integration. |
| `invoice_document` | Invoice Document — printing and voiding. |
| `cus_acc_stt_rep` | Customer Account Statement reconciliation — live data, read-only. |
| `recon_all` | Every report reconciliation — release check only. |

### Frontend (`cd psfront`)

| Target | What it runs |
|---|---|
| `all` | Every frontend test. |
| `watch` | Every frontend test, re-running on save. |
| `invoice_document` | Invoice Document screen. |

Tests are organised **per module / feature**, so a one-line change in the invoice
code only needs `npm test invoice`, not the whole suite. Targets marked live in the
listing warn before they start and cannot write.

### Report targets ask which company

A report is always run for one company, so a report target prompts for it rather
than defaulting silently:

```
$ npm test cus_acc_stt_rep
Company code to reconcile: 1001
Reconciling company 1001.
```

To skip the prompt (CI, or repeat runs) use either form — note the bare `--`,
because npm swallows an unknown flag otherwise:

```bash
npm test cus_acc_stt_rep -- --company=1001
RECON_COMPANY=1001 npm test cus_acc_stt_rep
```

A target asks for a company when its registry entry carries `needsCompany: true`.

---

## 3. Database safety

- Unit tests **never** open a database connection. The fake lives in
  `psback/tests/unit/helpers/mockDb.js`.
- Integration tests read `MYSQL_DATABASE` from `psback/.env.development` and
  **refuse to run** against anything other than `astra_test`
  (`psback/tests/integration/setup.js`). Staging and production are blocked by
  that guard, and a test asserts the guard itself.
- Integration tests are currently **read-only**. Any future create/update/delete
  test must clean up after itself and stay on `astra_test`.

---

## 4. File versioning rule

Every test file carries a version header:

```js
/**
 * @testfile   unit/invoice/edgeCases
 * @module     Invoice
 * @covers     controllers/invoice.controller.js
 * @version    1.1.0
 * @updated    2026-07-25
 * @cases      25 (10 simple / 5 medium / 10 hard)
 */
```

Rules:

1. Any developer who edits a test file **must bump `@version`** and update `@updated`.
2. The registry in section 8 must be updated in the same commit.
3. Version numbers use `MAJOR.MINOR.PATCH`:
   - **MAJOR** — the contract changed (a rule was redefined or removed).
   - **MINOR** — new tests or new cases added.
   - **PATCH** — a fix to the test harness itself, no change in what is asserted.
4. To check which version you have, open the file header or read the registry.

---

## 5. Coverage standard — 25 cases per test file (MANDATORY)

Every test file, backend or frontend, must reach **at least 25 cases**, split by
difficulty into three named `describe` blocks:

| Level | Count | What belongs here |
|---|---|---|
| **Level 1 — Simple** | 10 | One input, one obvious expectation. Does it load, does it render, is the badge right, is the field saved, does an empty or failed response behave. |
| **Level 2 — Medium** | 5 | Derived values and rules involving two or more fields. Totals, date rules, permission and selection rules, filters. |
| **Level 3 — Hard** | 10 | Where clients actually get hurt: money correctness, mixed statuses, rounding and tax edges, company and branch separation, ordering and duplicate keys, missing or malformed data. |

Rules:

1. Name each case with its level and number, e.g. `S1:`, `M3:`, `H7:` — so a
   failure line immediately says how serious it is.
2. Hard cases must be grounded in something real: a production row, a past defect,
   or a rule from `docs/`. Cite it in a comment with the real document number.
3. Do not invent business rules. If the correct behaviour is a business decision,
   **ask Mukesh before asserting it**.
4. A failing case is a defect, not a broken test. Never weaken an assertion to
   turn it green.
5. If a module genuinely has fewer than 25 meaningful cases, split it by feature
   into separate files rather than padding with duplicates.
6. Every module gets its own `npm run test:<module>` command.

---

## 6. Report reconciliation architecture — one report, one folder, one command

PowerSuite has around 40 report endpoints. They are never tested as one block,
because changing one report must never force a run of all of them.

**The rule: one report = one folder = one command.**

```
psback/tests/reconciliation/
  setup.js                                 ← shared read-only guard
  customer-account-statement/
    harness.js                             ← pure Excel readers for this report
    openingBalance.recon.test.js           ← the shared opening-balance engine
    report.recon.test.js                   ← the controller, filters, boundaries
    sections.recon.test.js                 ← money sections tying to the summary
    additivity.recon.test.js               ← splitting relations (metamorphic)
  daily-sale-report/
    harness.js                             ← readers for the 32-column sheet
    report.recon.test.js                   ← structure, nine filters, tenancy
  trial-balance/                           ← next report goes here
    ...
  ar-ageing/
    ...
```

Each report gets its own script, named after the report, so a change to one report
is verified by one command:

| Report | Command |
|---|---|
| Customer Account Statement | `npm test cus_acc_stt_rep` |
| Supplier Account Statement | `npm test sup_acc_stt_rep` |
| Trial Balance | `npm test trial_bal_rep` |
| Balance Sheet | `npm test bal_sheet_rep` |
| Profit & Loss | `npm test pnl_rep` |
| AR Ageing | `npm test ar_age_rep` |
| AP Ageing | `npm test ap_age_rep` |
| Daily Sale Report | `npm test daily_sale_rep` |
| Daily Settlement Report | `npm test daily_stl_rep` |

Customer Account Statement and Daily Sale Report exist today; the rest are the
naming to follow as they are added.

`daily_sale_rep` covers **only** POST `/report/dailySaleReport`. The DSR report,
DSR-01 and Daily Sale Report With Refund are three separate controllers and get
their own folders and commands when their turn comes — never fold them together.
`npm test recon_all` runs every report at once and is meant for a release check,
not for daily work.

### Adding reconciliation for a new report

1. Create `tests/reconciliation/<report-name>/` — kebab-case, named after the
   report as the business calls it, not after the controller function.
2. Add one `*.recon.test.js` per engine or section of that report.
3. Import the guard from `../setup` and call `installReadOnlyGuard()` **before**
   requiring anything that touches the database.
4. Discover fixtures with SQL at runtime. Never hard-code customer or invoice ids
   — they rot, and the file must work against any company or database.
   **Check every column against the schema before writing the query.** A fixture
   query that throws happens inside `beforeAll`, which fails every case in the file
   at once and looks like a catastrophic report bug when it is a typo. Guessing that
   `suppliers` had a `company_code` column turned all 30 Daily Sale Report cases red
   on the first run — it does not; suppliers belong to a company through `user_id`.
   Tenant columns in this database are not uniform: `users` has `company_code`,
   `orders` has `branch_id`, and `suppliers`, `customers` and `invoices` have
   neither. Confirm with `INFORMATION_SCHEMA.COLUMNS`, not from memory.
5. Add one entry to `psback/tests/registry.js` with `serial: true` and
   `live: true`, and add its row to the table above. **Do not touch `package.json`.**
6. Register the file in section 8 and update section 9.

### What a report reconciliation must assert

Not "the number equals 592,425". Numbers change with the data; identities do not.
Assert the accounting truths instead:

1. **Balancing** — debits equal credits; assets equal liabilities plus equity.
2. **Roll-forward** — closing of one period equals opening of the next.
3. **Composition** — a total equals the sum of the rows shown under it, and of its
   own buckets.
4. **Single-document movement** — adding one known simple document moves the total
   by exactly that document's amount.
5. **Quiet period** — a window with no transactions changes nothing.
6. **Completeness and uniqueness** — every document in the window appears exactly
   once, never twice, never missing.
7. **Boundaries** — the from-date and to-date are inclusive; the days either side
   are excluded; PKT and UTC do not shift a document into the wrong day.
8. **Cross-report** — the same figure in two reports must match (AR Ageing total =
   Customer Position total = the AR control account in the Trial Balance).
9. **Tenancy** — every row belongs to the requesting company and branch.
10. **Determinism** — the same inputs give the same answer twice, and the order of
    the inputs changes nothing.
11. **Splitting (additivity)** — cut the window in two and the parts must add back
    up to the whole. Split by customer as well as by date, and by both at once: a
    bug that cancels out along one axis has nowhere to hide along both.
12. **Monotonicity** — widening a window can never lower a total that only ever
    adds up positive amounts.

**Before asserting additivity, separate flows from stocks.** A flow counts only
what happened inside the window, so it adds across a split. A stock is a position
on one date, so it does not: the opening belongs to the first part and the closing
to the last. On the Customer Account Statement the seven middle lines are flows,
while `Opening Balance B/F` and `Net Balance` are stocks
(`report.controller.js:13990`, `:14008`, `:14027`, `:14058`). Asserting additivity
on a stock line produces a red test with no defect behind it.

Splitting is the relation that catches a join fan-out: a duplicated row inflates
the whole and the parts by different amounts, so the sum stops matching while every
individual figure still looks plausible.

## 7. How to design the 25 cases — the senior-tester method

**Read this before writing a single test.** The 25 cases in the existing files are
examples, not a template to copy. Every module has its own ways of hurting a
client. Your job is to find them, the way a senior tester who knows this business
would — not to fill a quota.

### 6.1 Investigate first, write second

Never design cases from the function name. Spend the first pass gathering facts:

1. **Read the whole function or component**, and list every `if`, every early
   return, every fallback default. Each branch is a candidate case; a fallback
   that invents a value (like defaulting a status to `Printed`) is almost always
   a defect.
2. **Read `docs/` for that module.** Business rules live there — invoice dates,
   cost quantity semantics, settlement linkage, tenant isolation.
3. **Query the real data read-only** (`astra_test`, or production with SELECT
   only) and look for the ugly rows: duplicates of a supposedly unique key, NULLs
   in required fields, mixed statuses inside one group, zero and negative amounts,
   extreme values, records with no parent. Those rows *are* your hard cases.
4. **Read the git history of the file** (`git log -p --follow <file>`). Every bug
   that was ever fixed there must have a permanent test, or it will come back.
   This project has a documented history of regressions from exactly that gap.
5. **Check `docs/SECURITY_TENANT_ISOLATION.md`** for the leak classes already
   known, and confirm the module you are testing is not repeating one.

### 6.2 Astra's standing risk areas

Apply this list to **every** module. It encodes what has actually gone wrong in
this system:

1. **Tenancy** — a company is reached through `service → order → user.company_code`;
   a branch through `orders.branch_id` and `req.user.allowed_branch_ids`. Every
   read *and* every write must be scoped. Test with a second company always present.
2. **Per-company uniqueness** — `invoice_number`, `document_number` and `supp_no`
   are unique **per company, not globally**. Never assume a number identifies one
   row. Always seed the same number in two companies.
3. **Duplicates inside one company** — legacy duplicate document numbers exist.
   Any "find by number" code must be tested with two rows sharing that number.
4. **Status transitions** — Void must never return to Printed or Raised; Settled
   must not be downgraded; an already-void record must not be voided again; a
   printed or voided record must never be deletable.
5. **Money** — a voided line must never be added into a total; the exchange rate
   saved at print time must be the one used; watch double counting of SST and
   transaction fees; totals must keep two decimals and their separators.
6. **Quantity semantics** — costs are stored once per service. `quantity = pax`
   means a per-person rate; `quantity = 1` with several passengers means a
   whole-service total. Per-passenger reports must divide, not multiply.
7. **Dates** — an invoice date for an Air service comes from
   `services.ticket_issue_date`, other service types use `invoice_date`; the same
   date applies to every row of one invoice number. Timezone is UTC vs PKT.
8. **Settlement linkage** — Manual JE settles documents through
   `journal_entries.analysis_code1 = document_number`. Anything computing
   `cost − payment_settlement` alone will miss those.
9. **Permissions** — document issue permission, branch permission and user group
   all gate actions. Test a user who lacks each one.
10. **Audit trail** — printed and voided records must survive; nothing may delete
    or renumber them silently.

### 6.3 Derivation checklist

For each function or screen, walk these questions and keep every answer that
produces a meaningful case:

1. Who else can call this? Another company's user, a user with **no**
   `company_code`, a user restricted to one branch, someone calling the API
   directly with no UI involved.
2. What if the id in the request belongs to somebody else?
3. What if two rows share the key being looked up?
4. What if the value is null, empty, zero, negative, or enormous?
5. What if the record is already in the state being requested?
6. What if half the group is void and half is live?
7. What happens to money, and what happens to the audit trail?
8. Is the operation reversible, and does a mid-way failure roll back completely?
9. Does it change any row it was not asked to change?
10. What does the **user actually see** afterwards — the number, the badge, the
    date, the row count?

### 6.4 Quality rules

1. Assert **observable outcomes** — what the row becomes, what the screen shows,
   what status code comes back. Do not assert internal call shapes; those break
   on harmless refactors.
2. **Test the product, not the fake.** If a failure could be caused by the mock,
   fix the mock first and re-check. A wrong fixture shape has already produced
   false findings in this project — for example asserting on `total_amount` when
   the screen reads `total_price`.
3. Keep each case independent — reseed before every test, never rely on order.
4. One reason to fail per case. If a case can fail for two unrelated reasons,
   split it.
5. When a case exists because of a real defect, name the real record in a comment
   (e.g. `Real case KHIN00000978: 592,425.00 live + 2,001.00 void showed as
   594,426.00`).

### 6.5 Worked example — how the Invoice Document cases were derived

1. Reading `InvoiceList.jsx` showed grouping by `document_number` alone and a
   `'Printed'` fallback → two hard cases.
2. Querying production for document numbers holding both a Void and a live
   invoice found exactly one, `KHIN00000978` → the billing-amount case, with the
   real numbers in the comment.
3. Reading `document.controller.js` showed the void guard checks the **document
   row's** status, not the **invoice's** → the "printing revives a voided
   invoice" case, which matches the KHIN00000003 / KHIN00000011 damage.
4. The company-scope fallback `req.user?.company_code ? {...} : {}` showed an
   empty filter for a user with no company → the "no company_code" case.
5. Everything else — badges, dates, selection rules, formatting — filled the
   simple and medium levels.

---

## 8. Test file registry

| File | Module | Version | Cases | Meets 25-case standard |
|---|---|---|---|---|
| `psfront/tests/invoice-document/invoiceList.test.jsx` | Invoice Document | 3.1.0 | 26 | Yes (10/5/11) |
| `psback/tests/unit/invoice-document/printVoid.test.js` | Invoice Document | 2.0.0 | 25 | Yes (10/5/10) |
| `psback/tests/reconciliation/customer-account-statement/openingBalance.recon.test.js` | Customer Account Statement | 1.0.0 | 25 | Yes (10/5/10) |
| `psback/tests/reconciliation/customer-account-statement/report.recon.test.js` | Customer Account Statement | 1.0.0 | 48 | Yes (10/13/25) |
| `psback/tests/reconciliation/customer-account-statement/sections.recon.test.js` | Customer Account Statement | 1.0.0 | 25 | Yes (10/5/10) |
| `psback/tests/reconciliation/customer-account-statement/additivity.recon.test.js` | Customer Account Statement | 1.0.0 | 25 | Yes (10/5/10) |
| `psback/tests/reconciliation/customer-account-statement/harness.js` | Shared helper | 1.0.0 | — | n/a |
| `psback/tests/reconciliation/daily-sale-report/report.recon.test.js` | Daily Sale Report | 1.0.1 | 30 | Yes (10/10/10) |
| `psback/tests/reconciliation/daily-sale-report/harness.js` | Shared helper | 1.0.0 | — | n/a |
| `psback/tests/reconciliation/setup.js` | Shared helper | 1.0.0 | — | n/a |
| `psback/tests/registry.js` | Test target registry | 1.0.0 | — | n/a |
| `psback/scripts/test.js` | Test runner | 1.0.0 | — | n/a |
| `psfront/tests/registry.js` | Test target registry | 1.0.0 | — | n/a |
| `psfront/scripts/test.js` | Test runner | 1.0.0 | — | n/a |
| `psback/tests/unit/invoice/tenantIsolation.test.js` | Invoice | 1.1.0 | 22 | Not yet |
| `psback/tests/unit/invoice/edgeCases.test.js` | Invoice | 1.1.0 | 11 | Not yet |
| `psback/tests/integration/invoice/companyIsolation.test.js` | Invoice | 1.0.0 | 6 | Not yet |
| `psback/tests/unit/helpers/mockDb.js` | Shared helper | 1.3.0 | — | n/a |
| `psback/tests/integration/setup.js` | Shared helper | 1.0.0 | — | n/a |
| `psfront/tests/setup.js` | Shared helper | 1.0.0 | — | n/a |

Backend files are being brought up to the 25-case standard module by module.

---

## 9. Reading the results

A **failing test is not a broken test**. These suites were written to describe how
the system *should* behave, so a red test marks a real defect that is still open.
Never weaken an assertion to make a test pass — fix the code instead.

As of 2026-07-25:

| Suite | Total | Pass | Fail |
|---|---|---|---|
| Backend unit | 58 | 38 | 20 |
| Backend integration | 6 | 3 | 3 |
| Customer Account Statement reconciliation | 98 | 96 | 2 |
| Frontend | 26 | 21 | 5 |

Reconciliation counts are for company 1001; the figures move with the company you
run, but the identities themselves must hold for every one.

`additivity.recon.test.js` adds 25 more cases to the Customer Account Statement
(123 in total) and has not been run yet, so it is deliberately left out of the
table above rather than guessed at. Update this row after the first run.

The Daily Sale Report has 30 cases so far and has not been run yet either. Its two
remaining files — `totals.recon.test.js` (summary identities) and
`additivity.recon.test.js` (splitting relations) — bring it to 80.

The failures are the open defect list, not flakiness.

---

## 10. Writing a new test file

1. Put it under the feature folder: `psback/tests/unit/<feature>/` or
   `psfront/tests/<feature>/`. Name it after the feature, not the controller.
2. Add the version header from section 4.
3. **Do section 6 first** — investigate, then derive the cases.
4. Build the 25 cases per section 5, in three `describe` blocks named
   `Level 1 — Simple`, `Level 2 — Medium`, `Level 3 — Hard`.
5. Backend unit: `jest.mock("../../../models", () => require("../helpers/mockDb").db)`
   and mock every service the controller requires at the top of its file.
6. Frontend: mock the `@/api/*` modules the screen imports, then render the real
   component inside `<MemoryRouter>`.
7. Add a target to `tests/registry.js` if the feature does not have one yet —
   never add a script to `package.json`.
8. Register the file in section 7 and update section 8.

---

## 11. Deployment flow

1. Change code locally.
2. Run the module's test command. All green (or only known failures) → continue.
3. Deploy to staging.
4. Manually check the critical screens on staging.
5. Deploy to production.
