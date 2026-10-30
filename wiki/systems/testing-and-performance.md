---
title: Testing and performance
status: planned
code: []
decisions: [D12, D13]
related: [networking, rendering, fish-simulation]
last_verified: n/a (no code yet)
---

# Testing and performance

## Summary

Three test layers (Lune headless specs, Jest Roblox in the engine, rodeo in play mode) plus checks on real phones over real cellular. A performance number counts only if it was measured on a device and recorded in the Measurements table below.

## Spec

### Test layers [decided]

| Layer | Tool | Where it runs | Use it for |
|---|---|---|---|
| Headless logic | Lune 0.11.1 specs | Locally and in CI | Codecs, closed-form schools, hashes, `step()`, validation, byte budgets |
| Engine context | Jest Roblox 3.20.1 (`roblox/jest`, dev dependency) through Open Cloud Luau Execution | CI, against a test place in a separate test universe | Code that needs real engine APIs |
| Play mode | rodeo 1.5.1 | Locally, on Windows or macOS (not Linux) | Client-server flows in Studio |

- Open Cloud Luau Execution [engine]: no physics, no scripts start automatically, at most 5 new tasks per minute and 10 unfinished tasks per place, a 5-minute default timeout, and it can touch real DataStores. So batch every spec into one task and use a separate test universe.
- rodeo's CLI, server and plugin must be the same build [engine].
- Studio's Server & Clients mode runs at most 8 clients [engine]. Testing with 20 players needs live test servers.

### Network testing [decided]

- Studio's Network Simulator (generally available since 2026-10-16), using the "Bad Connection (3G)" preset: 150 ms latency with 70 ms jitter inbound, 180 ms with 90 ms jitter outbound [engine].
- The simulator caps packet loss at 0.5% and can't limit bandwidth [engine], so also play on real phones over cellular, in 10–15-minute sessions.
- Watch data ping against network ping in the Developer Console, or with Shift+F3 on a client.

### Device profiling

- MicroProfiler on the phone, viewed from a desktop browser at the phone's IP address on port 1338 [engine].
- Creator Hub performance dashboard: memory use by device type, out-of-memory exits, and the Client CPU Time chart.
- Studio's device emulator is inaccurate for memory [engine], so validate memory only on hardware.

### Budgets [target]

| Metric | Budget |
|---|---|
| Server frame, p95 | ≤ 10 ms |
| Fish simulation + packet building, p95 | ≤ 3 ms |
| Client memory on 2–3 GB phones | ≤ 500–600 MB |
| Client Luau per frame (fish, effects, UI) | 2–4 ms |
| Draw calls / triangles | < 1,000 / < 1 million (Roblox's baseline example) |
| Network per client | Typically 10,000–20,000 B/s; ceiling 32 KiB/s |

Always report p95 (and p99), never only averages: SMASH failed on p95.

### First milestone [decided]

Before any content production: a vertical slice with 300 fish and 20 sharks, running on a 2–3 GB Android phone and an older iPhone over a real cellular connection, judged against the budgets above.

### CI pipeline [decided]

- **Every pull request:** `setup-rokit` → `wally install` → `rojo sourcemap` → `luau-lsp analyze`, `selene`, `stylua --check` → Lune specs.
- **Every merge:** `rojo build` → publish to the test place with `versionType=Saved` through the Open Cloud place-publishing API → run Jest Roblox through Luau Execution → promote to `Published` behind a gate.
- Roblox's publishing guide says places containing SurfaceAppearance, PartOperation (unions) or EditableMesh should be published from Studio after changes [engine]. Keep art in a Studio-authored base place, or prove in the test universe that the automated publish preserves it.

## Measurements

None yet. Add a row for every measurement, newest first.

| Date | Device / setup | Build or commit | Scenario | Result | Link |
|---|---|---|---|---|---|

## Invariants

- No performance claim without a row in Measurements.

## Tests

This page defines the test layers; each system page lists its own specs.

## Open questions

- Which exact phones are the baseline devices.
- How to simulate bandwidth limits, since the Network Simulator has none.

## History

- 2026-10-30 — Created with the planned design from the tech stack research.
