# Advance Payment to Supplier (Supplier Deposit) Module

Pre-paying a supplier before any invoice/cost exists. The advance is held as a
**supplier deposit** and later settled against the supplier's cost documents.

## 1. Where it lives
1. **Page**: `psfront/src/pages/PaymentsPage/AdvancePayment.jsx` — route
   `/payments/advance-payment` (under the **Payments** menu).
2. **Listing**: the **Advance** tab on the Documents page
   (`psfront/src/pages/Document/AdvancePaymentList.jsx`).
3. **Printed document**: rendered via `type=supplier_deposit`
   (`/documents/<payment_number>?type=supplier_deposit`).
4. **Backend**: `psback/controllers/supplier_deposit.controller.js`; table
   `supplier_deposits`.

## 2. Page flow
1. **Supplier Information**: select a supplier and a branch.
2. **Supplier Deposits (n)**: existing advances for that supplier (see §5).
3. **Form of Payment**: date, payment method, currency, amount, account/bank,
   cheque number (if cheque), remarks.
4. On submit, `createSupplierDeposit` runs; the user is redirected to the printed
   supplier-deposit document.

## 3. Data model (important — what each amount means)
On create (`createSupplierDeposit`):
1. `amount` = the **value the user typed**, in the **entered currency**.
2. `exchange_rate` = entered-currency → **base-currency** rate (1 if entered currency
   is the base).
3. `base_amount` = `current_amount` = `round(amount × exchange_rate)` = the **base-currency
   equivalent**.
4. `current_amount` then **decreases** as the advance is settled against costs; it is the
   remaining **base-currency** balance.
5. So **Original Amount = `amount` (entered currency)** and, by storage, **Available
   Amount = `current_amount` (base currency)**. For a deposit entered directly in the
   base currency (rate 1) the two are the same currency.
6. **Open item (display)**: the listings currently label Available Amount with the
   *entered* currency for consistency with Original Amount. That reads correctly for the
   present test data (PKR advances with `exchange_rate = 1.0`, so base == entered) but is
   technically wrong for a real foreign advance (where `current_amount` is the base
   value). The dummy PKR record has a placeholder rate of `1.0` instead of a real
   PKR→base rate, which is why "USD 10,000" looked wrong. Revisit labelling Available
   Amount with the **base** currency once real foreign advances exist.

## 3a. Bank account selection (Cheque / Bank Transfer / Card)
1. The bank dropdowns (pay types 2, 6, 12 and the Card block 3/7) list **bank
   accounts**, not banks: each option shows `bank name (account_number)`.
2. One bank can own **many accounts** (e.g. five Meezan accounts all under bank id
   42). Each option's value is therefore the **bank account id** (`account.id`,
   unique), not the shared `bank.id`.
3. On select, the chosen account is looked up by `account.id` and the form stores its
   `bank_id`, its real `account_number`, and its `chart_of_account_id`; the dropdown's
   selected value is tracked via `bank_account_id`.
4. **History / why**: options previously used `account.bank.id` as the value and looked
   the account up by `bank.id`. With multiple accounts per bank that lookup always
   matched the **first** account, so picking the 2nd Meezan account printed the 1st
   account's number on the document. Fixed by keying selection on the unique account id.

## 4. Currency selection (base-currency aware)
1. The company **base currency** is derived from the loaded currency list — the
   `to_currency` of a configured rate record (e.g. `EUR → USD` ⇒ base **USD**),
   defaulting to `PKR`.
2. The **Currency** dropdown offers only:
   - the **base currency** itself (always), and
   - currencies **configured to convert to the base** (`to_currency === base` with a
     real `exchange_rate > 0`, e.g. EUR).
   PKR is no longer force-added; for a USD-base company the list is **USD + EUR**.
3. The **default** (and post-submit reset) currency is the **base currency**.
4. Selecting the base currency sets `exchange_rate = 1`; selecting a foreign currency
   uses that currency's saved rate. The "… Equivalent" helper under Amount shows the
   base-currency equivalent (`amount × exchange_rate`) and only appears when the rate ≠ 1.

## 5. Supplier Deposits list (on the page)
1. Shows the selected supplier's advances: Date, Payment No., Supplier No./Name,
   **Original Amount** (entered currency), **Available Amount**, Status.
2. **Voided deposits are excluded** from this list — it only shows live advances
   (`status !== 'Void'`). A voided advance is finished and not reusable, so it is hidden
   here.

## 6. Printed document (currency-aware)
1. Template `psback/views/pages/supplierDepositDocument.ejs`, rendered by
   `getDepositDocument` (`type=supplier_deposit`).
2. **Paid** shows the entered amount in the **entered currency** (e.g. `USD 6,000.00`).
3. **Amount In Words** uses the **entered amount + entered currency** via
   `PKRtoWords(amount, currencyCode)` — now currency-aware (dollars/cents, euros,
   rupees/paisas, …; unknown currencies fall back to the code). So a USD advance reads
   "Six Thousand Dollars (USD)", not "… Rupees (PKR)".
4. For a foreign-currency advance (entered ≠ base) the doc also shows **Exchange Rate**
   and **`<base>` Equivalent** lines, labelled with the company **base currency** (was
   hard-coded "PKR"). The base currency is derived in the controller from
   `currencies.to_currency`.
5. The **customer** deposit equivalent (`customerDepositDocument.ejs` /
   `deposit.controller.js`) has the same fix applied — see `CUSTOMER_DEPOSIT_MODULE.md`.

## 7. Void behaviour
1. Voiding a supplier deposit checks it is not in use by a payment settlement, then sets
   `status = 'Void'` and `current_amount = 0`.
2. Voided advances disappear from the on-page Supplier Deposits list (§5) but remain
   visible (greyed/"Void" status) on the Documents → Advance listing for audit.

## 8. Related
1. Settling an advance against supplier costs happens in the Payment module.
2. Base-currency labelling rules across documents/listings: see
   `INVOICE_DOCUMENT_CURRENCY.md`.
