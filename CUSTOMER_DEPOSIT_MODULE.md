# Advanced Deposit from Customer (Customer Deposit) Module

Taking money from a customer in advance (a deposit/credit on account) before any
invoice exists. The deposit is held and later settled against the customer's invoices.

## 1. Where it lives
1. **Page**: `psfront/src/pages/RecieptPage/AdvancedDeposit.jsx` — route
   `/receipt/apply-deposit` (under **Receipt → Advanced Deposit**).
2. **Listing**: the **Deposit** tab on the Documents page (`DepositList.jsx`).
3. **Printed document**: `type=customer_deposit`
   (`psback/views/pages/customerDepositDocument.ejs`, rendered by `getDepositDocument`
   in `deposit.controller.js`); table `customer_deposits`.

## 2. Data model (same as supplier deposit)
1. `amount` = entered amount in the **entered currency**.
2. `exchange_rate` = entered-currency → **base-currency** rate (1 if entered = base).
3. `base_amount` = `current_amount` = `amount × exchange_rate` = the **base-currency**
   equivalent; `current_amount` decreases as the deposit is settled.
4. So **Received/Original = entered currency**, **Available (current_amount) = base
   currency**.

## 3. Currency selection (base-currency aware)
1. The company **base currency** is derived from the loaded currency list — the
   `to_currency` of a configured rate record (e.g. `EUR → USD` ⇒ base **USD**),
   defaulting to `PKR`.
2. The **Currency** dropdown offers only the **base currency** itself plus currencies
   **configured to convert to the base** (`to_currency === base` with `exchange_rate > 0`,
   e.g. EUR). PKR is no longer force-added; for a USD-base company the list is
   **USD + EUR**.
3. The **default** (and post-submit reset) currency is the **base currency**.
4. Selecting the base currency sets `exchange_rate = 1`; the "… Equivalent" helper under
   Amount shows the base-currency equivalent and only appears when the rate ≠ 1.

## 4. Printed document (currency-aware)
1. **Received** shows the entered amount in the **entered currency**.
2. **Amount In Words** uses the entered amount + entered currency via
   `PKRtoWords(amount, currencyCode)` — currency-aware (dollars/cents, euros,
   rupees/paisas, …; unknown currencies fall back to the code).
3. For a foreign-currency deposit, the **Exchange Rate** and **`<base>` Equivalent**
   lines are labelled with the company **base currency** (was hard-coded "PKR"); the base
   currency is derived in the controller from `currencies.to_currency`.

## 5. Related
1. Settling a customer deposit against invoices happens in the Receipt module.
2. Base-currency labelling rules across documents/listings: see
   `INVOICE_DOCUMENT_CURRENCY.md`. Supplier counterpart: `ADVANCE_PAYMENT_MODULE.md`.
