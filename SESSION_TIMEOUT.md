# Session Timeout (Auto-Logout)

## Purpose
Automatically log a user out of Astra when they stop working, while capping the total
length of any single login. Follows the standard industry pattern of using **two timers
together**: an idle timeout plus an absolute (hard-cap) timeout.

## Rules
1. **Idle timeout — 30 minutes.** If there is no user activity (mouse move, mouse down,
   key press, scroll, touch, click) for 30 minutes, the user is logged out.
2. **Absolute hard cap — 8 hours.** No login can last longer than 8 hours total, even for a
   continuously active user. Whichever timer runs out first wins.
3. **2-minute warning.** At the 28-minute idle mark a warning toast appears
   ("You will be logged out in 2 minutes due to inactivity.") with a **Stay signed in**
   button. Any activity, or clicking the button, cancels the warning and resets the idle clock.

## How it works
1. **Component:** `psfront/src/components/SessionTimeout.jsx` — an invisible component
   (renders `null`) mounted once in `psfront/src/App.jsx` alongside `<Routes>`.
2. **Activity tracking:** activity events refresh a `lastActivity` timestamp in `localStorage`.
   Because `localStorage` is shared, activity in **any open Astra tab** keeps all tabs signed in
   (no wrongful logout with multiple tabs). Writes are throttled to once per second.
3. **Idle check:** a timer runs every 15 seconds and compares `now - lastActivity` against the
   30-minute threshold.
4. **Absolute cap:** the 8-hour limit is read directly from the login token's own `exp` claim
   (JWT), so no separate "login time" is stored. The check fires proactively (it does not wait
   for the next server call).
5. **Logout action:** on timeout the component dispatches the existing `userLogout` action
   (clears Redux + `localStorage`) and navigates to `/login`.

## Related / existing behavior (unchanged)
1. **Backend token:** issued with `expiresIn: "8h"` in `psback/controllers/auth.controller.js`.
   Auth is stateless JWT; there is no refresh-token mechanism.
2. **redux-persist-expire:** `psfront/src/store/index.js` still expires the persisted `user`
   slice after 8 hours (`expireSeconds: 3600*8`). This remains as a second layer.
3. **401 handling:** `psfront/src/api/instance.js` still logs out + redirects on any `401`
   response. The idle timeout does not change this.

## Tunable values
All in `SessionTimeout.jsx`:
- `IDLE_TIMEOUT_MS` — idle window (default 30 min). Set to `15 * 60 * 1000` for the stricter
  PCI-style 15-minute rule.
- `WARNING_BEFORE_MS` — how long before logout the warning shows (default 2 min).
- `CHECK_INTERVAL_MS` — how often the idle check runs (default 15 s).

## Not included (possible future hardening)
- Short-lived backend token + silent refresh-on-activity. The current idle timeout logs the
  UI out, but the token itself stays valid on the server for its full 8-hour life. For an
  internal system this is the normal, accepted trade-off; refresh tokens would make it airtight.
