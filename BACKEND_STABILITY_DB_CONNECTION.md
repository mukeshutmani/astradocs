# Backend Stability — Remote DB Connection Handling

## Background (July 2026)

The backend crashed repeatedly with:

```
SequelizeDatabaseError / read ECONNRESET
sql: 'START TRANSACTION;'
```

### Root cause

1. The MySQL database is **remote** (AWS host, forwarded port `5307`), reached over the internet. The port-forward/NAT silently drops idle TCP connections; the MySQL server had logged 4,300+ aborted clients.
2. The Sequelize pool kept `min: 2` connections open forever with no TCP keep-alive, so after any idle period the pool held dead sockets.
3. The first request after an idle period picked up a dead socket. If that request started a transaction in a code path with no error catcher, the rejection reached process level — and on Node 15+ an unhandled rejection **kills the whole process**. Nodemon then sat at "waiting for file changes".
4. Most frequent trigger: `login` (`psback/controllers/auth.controller.js`) started a fire-and-forget transaction (no `await`, no `.catch`) on every login.

## Fixes applied

| File | Change |
|------|--------|
| `psback/index.js` | Added `process.on("unhandledRejection")` handler (log instead of crash). A DB drop now fails one request, not the server. |
| `psback/models/index.js` | Pool: `min` 2 → 0 (idle connections are evicted instead of going stale). Added `dialectOptions.enableKeepAlive` + `keepAliveInitialDelay: 10000` (TCP keep-alive defeats the NAT idle drop). Added `retry: { max: 3, match: [/ECONNRESET/, /PROTOCOL_CONNECTION_LOST/] }` so standalone queries landing on a dead socket are retried on a fresh connection. |
| `psback/controllers/auth.controller.js` | `login`: appended `.catch()` to the transaction promise. Failures of `START TRANSACTION` / `COMMIT` now log and return 500 (guarded by `res.headersSent`) instead of crashing. |
| 10 controllers — cn/dn/gl/invoice/xo import, `lccImport`, `service` (4 handlers), `customer_required_data` (7), `data` (3), `jvPeriod` (2) | Second pass (28 handlers): moved `await sequelize.transaction()` from one line *before* the `try` to the first line *inside* it, and null-guarded the catch rollback (`if (t && !t.finished) await t.rollback()`). A failed transaction start now returns a proper 500 to the client instead of a silently hanging request. |
| `psback/controllers/report.controller.js` | `getBankActivityReport`: appended `.catch()` to the whole-handler transaction (same pattern as login) — transaction/report/commit failures now reply 500 (guarded by `res.headersSent`) instead of hanging. |
| `psback/controllers/journal_entry.controller.js` | `generateJournalEntries`: wrapped the `generateJournal(...)` call in try/catch — System JE generation failures now reply 500 instead of hanging. |

### Notes on the retry setting

1. Sequelize's `retry` applies per query. Queries **inside** a transaction are not duplicated by it — if the transaction's connection dies, the transaction fails as a whole and nothing commits.
2. For standalone (non-transactional) writes there is a theoretical duplicate-write window if the connection dies after the server applied the query but before the response arrived. Match list is kept narrow (`ECONNRESET`, `PROTOCOL_CONNECTION_LOST`) to limit this.

## Known remaining risks (not yet fixed)
1. ~18 heavy report handlers wrap the entire run (Excel/PDF/upload, 1–5 min) in a transaction they barely use — the pinned connection idles past the NAT drop window and the final COMMIT can fail.
2. `services/documentNumbering.js` holds its `GET_LOCK` on a pinned connection; if that connection dies mid-import, MySQL releases the named lock early and a concurrent import could allocate a **duplicate document number**.
3. The global error handler at `index.js` (`app.all("*", errorHandler)`) is registered after the 404 catch-all and as a route (not 4-arg error middleware), so it never runs.
