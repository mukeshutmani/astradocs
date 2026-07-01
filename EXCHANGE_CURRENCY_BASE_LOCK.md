# Exchange Currencies — Base Currency Lock

How the **Exchange Currencies** maintenance page (System Tables) establishes and locks
the company's base currency.

## 1. The rule (2026-06-30)
1. Each exchange rate is stored as **From Currency → To Currency** in the `currencies`
   table (scoped by `company_code`).
2. The **"To Currency" is the company's base currency**. Every company uses one single
   base — all of its rows share the same `to_currency`.
3. The **first** exchange rate a company adds lets the user pick **both** From and To.
   That first **To** becomes the locked base.
4. After that, the **To Currency is locked** — on every new/edited rate the user only
   picks **From**, and To is forced to the existing base.

## 2. How the base is determined
1. The base is the `to_currency` of the company's **earliest** `currencies` row
   (`ORDER BY id ASC LIMIT 1`), matched by `company_code`.
2. Helper: `getCompanyBaseCurrency(company_code)` in
   `psback/controllers/data.controller.js`.
3. Endpoint: `GET /api/data/currency-base` → `{ base_currency }` (or `null` if the
   company has no rate yet). Reuses the `Exchange-Currencies` permission.

## 3. Server-side enforcement
1. `createCurrency`: if a base exists, the new row's `to_currency` is **forced** to the
   base — the client value is ignored. The first-ever row uses the client's To.
2. `updateCurrency`: `to_currency` is forced back to the base, so an edit cannot change
   it.
3. This holds even if the request bypasses the UI.

## 4. Frontend behaviour
1. `AddExchangeCurrencies.jsx` calls `getBaseCurrency()` on load.
2. If a base exists → "To Currency (Base)" renders as a **read-only** box showing the
   base; only **From** is selectable.
3. If no base yet → both From and a "Select base currency" dropdown are shown (first
   entry sets the base).
4. A **red note** on the form states: *"TO is your base currency. Once the first
   exchange rate is added, the base currency is locked."*

## 5. Deleting a rate (2026-06-30)
1. `currency_history` has a foreign key to `currencies.id` with no `ON DELETE` rule, so
   deleting a currency that still has history rows was blocked by MySQL.
2. `deleteCurrency` now removes the currency's `currency_history` rows first, then the
   currency itself, inside a **transaction** (all-or-nothing).
3. The company-ownership check is kept — only the caller's own company rows can be
   deleted.
4. **Delete button is disabled by default.** `ExchangeCurrencies.jsx` has a
   developer-only flag `CAN_DELETE_CURRENCY` (currently `false`). While `false`, the
   Delete option renders blurred/faded and unclickable for all users; set it to `true`
   to re-enable the normal delete-with-confirmation flow. (UI only — the delete API is
   unchanged.)

## 6. Related
1. Documents read this same base from `currencies.to_currency` — see
   `INVOICE_DOCUMENT_CURRENCY.md`. This lock keeps that source single and consistent.
2. The PDF/report currency converters still assume **PKR** and were **not** changed by
   this work.
