---
title: Persistence
status: planned
code: []
decisions: [D10]
related: [eating-and-bites, ui]
last_verified: n/a (no code yet)
---

# Persistence

## Summary

Each player's progress is a small ProfileStore profile, session-locked so flaky mobile rejoins can't lose or duplicate data. The same profile makes purchases idempotent. Leaderboards and analytics are budgeted carefully, because their request limits are the scarce resource.

## Spec

### Library [decided]

ProfileStore 1.0.3 (Wally `lm-loleris/profilestore`, server realm, mounted under ServerPackages). Its session lock asks the server holding a profile to save one last time before handing it over.

### Profile

- Contents: currency, unlocked sharks, upgrades, settings. A few KB [target]. The exact fields aren't designed.
- `schemaVersion` is an integer [decided]. On load, run the ordered migration steps from the stored version to the current one, then `Reconcile`, which only adds missing keys [decided].

### Purchases [decided]

- Roblox keeps no per-user purchase history. It retries `ProcessReceipt` on rejoin when the handler returns `NotProcessedYet`, and it warns against granting items from `PromptProductPurchaseFinished` [engine].
- The profile keeps a bounded ring buffer of recent PurchaseIds. In one synchronous update: check the id, grant the item, record the id. Return `PurchaseGranted` only after the profile has saved successfully. Follow ProfileStore's documented developer-product pattern when implementing.
- Show prices from `GetProductInfoAsync`: managed regional pricing and Roblox Plus discounts change them per user [engine].
- A newer `BindReceiptHandler` API exists, but the docs don't say whether it replaces `ProcessReceipt`. Check when implementing.
- Roblox Plus (launched 2026-04-30) pays creators 250 Robux a month for each new subscriber's first three months when the sign-up comes through `PromptRobloxSubscriptionPurchase` [engine].

### DataStore limits [engine]

Per server, per minute:

| Request | Budget |
|---|---|
| Standard read, write, remove | 60 + 40 × players |
| Standard list | 5 + 2 × players |
| Ordered read | 60 + 40 × players |
| Ordered write, remove | 30 + 5 × players |
| Ordered list | 5 + 2 × players |

Per experience, per minute, shared with Open Cloud: standard read and remove 300 + 40 × CCU; standard write 300 + 20 × CCU; ordered read and remove 300 + 40 × CCU; ordered write 300 + 20 × CCU; lists 300 + 2 × CCU.

- `UpdateAsync` uses both read and write budget.
- Per key: values up to 4,194,304 B; at most 4 MB/min written and 25 MB/min read.
- Storage: 500 MB + 1 MB per lifetime player (since 2026-07-29).

### Leaderboards [decided]

- All-time stats go to an OrderedDataStore at session end. Cache top-N reads for about a minute. Which stats are tracked isn't designed.
- The live "biggest sharks" board uses a MemoryStore sorted map.

### Analytics [decided]

- Limits: 120 + 20 × CCU AnalyticsService requests per minute, and 100 custom event names [engine].
- Aggregate per life or per session. Never send an event per fish eaten.

## Invariants

- A purchase is granted at most once, even across rejoins and server crashes.
- Migrations run before `Reconcile`, in version order.
- Nothing writes to a DataStore or sends analytics per fish.

## Tests

None yet. ProfileStore ships a mock store that specs can use.

## Open questions

- Profile fields (these depend on the growth, roster and economy designs).
- The ring buffer length for PurchaseIds.
- `BindReceiptHandler` versus `ProcessReceipt`.
- Which stats get leaderboards.

## History

- 2026-10-30 — Created with the planned design from the tech stack research.
