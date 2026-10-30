---
title: Architecture
status: planned
code: []
decisions: [D4, D5, D6, D7, D8]
related: [networking, fish-simulation, eating-and-bites, shark-movement, rendering, persistence]
last_verified: n/a (no code yet)
---

# Architecture

## Summary

The server is the authority on outcomes (who ate what, growth, currency, unlocks), not on movement. Each phone simulates its own shark, and fish are compact server data that clients draw locally. This split keeps server CPU and bandwidth low enough for 10–20 players on mobile, which SMASH's full 60 Hz prediction model couldn't manage.

## Spec

### Who owns what [decided]

| State | Authority | How others learn it |
|---|---|---|
| Your shark's position, velocity, orientation | Your client, through network ownership | Engine physics replication; the server checks it ([shark-movement](shark-movement.md)) |
| Other players' sharks | Their own clients | Engine physics replication, about 15–20 Hz (sources disagree) [engine] |
| Tier-A fish positions | Nobody steps them: a closed-form function of the school record and server time | The school record, sent once when the school becomes relevant ([fish-simulation](fish-simulation.md)) |
| Tier-B fish and tier-C predator positions | Server | The state stream ([networking](networking.md)) |
| Whether something was eaten or bitten | Server | Reliable Blink events ([eating-and-bites](eating-and-bites.md)) |
| Growth, XP, currency, unlocks, purchases | Server | Reliable events and profile sync ([persistence](persistence.md)) |
| Chomp effects, hiding a bitten fish | The biting client, straight away | Undone if the server rejects the bite |

### Loops and rates [decided]

| Loop | Runs on | Rate |
|---|---|---|
| Server frame (Heartbeat) | Server | Capped at 60 FPS, 16.67 ms [engine] |
| Tier-B fish step | Server | About 10 Hz, time-sliced across Heartbeat frames under a fixed budget |
| Tier-C predator decisions / steering | Server | 2–5 Hz / 10–20 Hz |
| State stream | Server → client | 10–20 Hz near a player, 2–5 Hz farther out |
| Tier-A fish evaluation | Client | Every rendered frame for nearby fish, at estimated server time |
| Fish rendering updates | Client | Every frame near the camera, ≤ 30 Hz mid-range, 10–15 Hz at the fog edge |
| HUD values | Client | 10–15 Hz |
| Own shark physics | Client | Engine physics step |

### Data flow

```
Your client                               Server                                     Other clients
-----------                               ------                                     -------------
IAS input -> step() -> movers -> physics --engine replication--> movement checks --engine--> draw sharks
mouth overlaps target -> chomp, hide ----BiteClaim (Blink)-----> validate with rewind
                                                                  |- accept -> Eaten (Blink) ----> remove target, effects
                                    <----BiteRejected (Blink)-----'- reject
draw tiers B/C <-- state stream (unreliable, <=950 B) <-- per-player packets <-- cells encoded once per tick
evaluate tier A <-- SchoolEnter / Scatter (Blink, reliable)
```

### Place settings [decided]

- `Workspace.StreamingEnabled = true`, configured as on the [rendering](rendering.md) page.
- `Workspace.SignalBehavior = Deferred`, so Server Authority stays possible later (D4).
- `Workspace.AuthorityMode` stays at its default (client authority) until D4 changes.
- No Terrain water anywhere.
- Lighting as on the [rendering](rendering.md) page.

### Never

- Server Instances for fish, or Humanoids on sharks or fish.
- Re-encoding shared state per player.
- Allocating tables per fish per frame.
- 60 Hz loops on the server: nothing in this game needs them.

## Invariants

- The server never trusts a client for eats, growth, currency, unlocks or purchases.
- Fish never exist as server Instances.
- Anything both sides must compute identically (school paths, hashes, quantization) lives in `src/shared/`.

## Tests

None yet.

## Open questions

- Should other players' sharks stay on engine physics replication, or move into the state stream? Decide after measuring receive bandwidth and smoothness with 20 sharks on a phone.
- Roblox hasn't published how server CPU is allocated per player since November 2025. Measure on live servers.

## History

- 2026-10-30 — Created with the planned design from the tech stack research.
