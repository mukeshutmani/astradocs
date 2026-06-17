# Documents — Base Currency Display

How the printed/preview **invoice** (`psback/views/pages/invoiceDocument.ejs`) and
**cost/XO document** (`psback/views/pages/costDocument.ejs`) decide which currency
label to show (both rendered by `document.controller.js`), plus the credit/debit note
documents and the frontend Documents listing page (see §5–§6).

## 1. What changed (2026-06-16)
1. The document no longer hard-codes the word **"PKR"** for the reference/base amounts.
2. It now shows the **company's base currency** (USD, AED, GBP, …) wherever it used to
   print "PKR". Pakistani companies fall back to **PKR**, so nothing changes for them.

## 2. Where the base currency comes from
1. The controller reads it from the company's **currency setup** — the `to_currency`
   of the company's row(s) in the `currencies` table (matched by `company_code`).
2. Example: company `1010` has `from_currency = EUR`, `to_currency = USD`, so the base
   currency is **USD**.
3. It is **not** stored in `system_table` (that table only holds table names).
4. If a company has no currency setup, the base currency defaults to `PKR`.
5. The controller passes this as `baseCurrency` to the template and sets
   `exchangeRateInfo.toCurrency = baseCurrency` (was previously hard-coded `"PKR"`).

## 3. How amounts work
1. The entered/invoice currency (e.g. EUR) is the **primary** amount shown.
2. The **saved** `exchange_rate` on the invoice (e.g. 1.15) converts the entered
   currency to the base currency — so the grey reference number is already correct;
   only the label was wrong before.
3. The grey reference line shows only when the invoice currency differs from the base
   currency (`showItemPKR` / `needsConversion`).

## 4. Labels now driven by base currency
1. Currency Conversion Notice ("… amounts are shown for reference at … = 1.15 USD").
2. Per-line grey reference amounts (Unit Fare, Tax, Amount, Transaction Fee, Supplementary Fee).
3. Total SST, Grand Total (incl. amount-in-words), Received, Received by JE, Balance.
4. The multi-currency branches (when one document mixes currencies).

## 5. Done so far / still pending
1. **Done**: `invoiceDocument.ejs` and `costDocument.ejs` (XO) — both compute
   `baseCurrency` in their `document.controller.js` branch and print it instead of "PKR".
2. **Done**: `creditNote.ejs` and `debitNote.ejs` — the template already structured
   primary vs. base amounts via a local `baseCurrency`; it now reads `baseCurrencyCode`
   passed from the controller (falls back to `PKR` if absent). Render paths updated:
   `invoice.controller.js` (2 credit-note + 2 debit-note `res.render`) and the
   `debit_note.controller.js` PDF-fallback `templateData`.
3. **Done**: `supplierDepositDocument.ejs` — currency-aware Amount In Words + base-currency
   Equivalent lines (`supplier_deposit.controller.js`).
4. **Done**: `paymentSettlement.ejs` (the Payment / settlement voucher, e.g. `MKPY…`) —
   all printed "PKR" labels now show the company `baseCurrency`, derived in
   `payment.controller.js` (from `currencies.to_currency`). Settlement amounts have no
   currency column → they are stored in the base currency, so only the labels were wrong.
   Logic checks (`!== 'PKR'`) were intentionally left as-is.
5. **Done**: `supplierDepositDocument.ejs` and `customerDepositDocument.ejs` —
   currency-aware Amount-In-Words + base-currency Equivalent lines.
6. All document templates are now base-currency-aware. Still pending: reports/summaries
   labelled "(PKR)"; and parts of the Supplier-Payments page (`Payments.jsx`) settlement
   summary still print "PKR".

## 6. Document listing page (frontend tabs)
The Documents list page (`psfront/src/pages/Document/*List.jsx`) showed a hard-coded
"PKR" prefix on its amount columns. All tabs now show the **company base currency**:

Two different cases — pick the label by what the stored value actually is:

**A. Amounts converted to base** → label with the company **base currency**.
1. **Invoice** (`InvoiceList`) and **XO** (`CostList`) convert each amount to base via
   `convertToPKR` (× the entered→base rate). They derive `baseCurrency` from the loaded
   currency list (`response.find(c => c.to_currency && c.currency_code !== c.to_currency
   && parseFloat(c.exchange_rate) > 0)`, default `PKR`) and label with it. Their
   "already base?" guard also matches `code === baseCurrency`.
2. **Receipt** (`ReceiptList`) and **Payment** (`PaymentList`): the `receipts`/
   `payment_settlements` tables have **no currency column** — the amount is stored in the
   company's base/home currency. So these label with the derived `baseCurrency` (added a
   small `getCurrencies` effect to each).
3. **Credit Note** / **Debit Note** lists use `DualCurrencyDisplay`; its `baseCurrencyCode`
   prop falls back to the derived `baseCurrency` instead of `'PKR'` (the record's own
   `base_currency` relation is still preferred when present).

**B. Amounts stored in the ENTERED currency** → label with the entered currency, NOT base.
1. **Advance** (`AdvancePaymentList`, supplier_deposits) and **Deposit** (`DepositList`,
   customer_deposits) store `amount`/`current_amount` in the currency the deposit was
   actually paid in (`currency_id` + `exchange_rate` + separate `base_amount`).
2. Both the "Original/Amount" and "Available" columns label with the record's
   `currency_code` (e.g. a PKR advance shows `PKR …` in both). Do **not** show the base
   currency here — these values are not converted. (`baseCurrency` was briefly added to
   these two files and then removed.)
3. The `currencyCode` (entered-currency) columns elsewhere keep their `|| 'PKR'` fallback
   — that is the original currency, not the base.

## 7. Note / open item
1. Internal variable names in the templates (e.g. `grandTotalPKR`, `totalSSTPKR`) and
   the `convertToPKR` helper keep the `PKR` in their names — they hold base-currency
   values; only the printed/displayed labels changed.
