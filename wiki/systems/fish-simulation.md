---
title: Fish simulation
status: planned
code: []
decisions: [D6, D13]
related: [networking, eating-and-bites, rendering, animation]
last_verified: n/a (no code yet)
---

# Fish simulation

## Summary

Fish exist on the server only as data, split into three tiers so that server cost scales with the number of schools, not fish. Most fish (tier A) are never stepped: their positions are a closed-form function of a school record and server time, which the server and clients evaluate identically. Only fish reacting to a shark (tier B) and AI predators (tier C) are simulated and streamed.

## Spec

### Tiers [decided]

| Tier | Which fish | Server state | Network | Client |
|---|---|---|---|---|
| A: calm schools | Most prey | One school record; never stepped | About 20–40 B once, when the school becomes relevant [target], then reliable events (scatter, eaten) | Evaluates the path at estimated server time, with no interpolation buffer |
| B: reacting fish | Fleeing, chased or large prey | A slot in preallocated parallel arrays, stepped at about 10 Hz | 10 B records: 10–20 Hz near a player, 2–5 Hz farther out | Interpolated with a 150–350 ms buffer |
| C: AI predators | A handful to about 20 [target] | A jecs entity that decides at 2–5 Hz and steers at 10–20 Hz | Same stream as tier B, at the near rate | Interpolated and animated |

### Tier A: calm schools

School record [decided]: id, species, seed, spawn time, path parameters, alive bitmask.

Position of fish `i` at server time `t` [decided]:

```
h_i      = hash32(seed, i)                     -- bit32 integer hash
anchor_t = path(pathParams, t - spawnTime)     -- the school's looping path
pos_i(t) = anchor_t + offset(h_i, t)           -- per-fish offset from the hash
```

- `t` is `workspace:GetServerTimeNow()` on both sides. Clients use their estimate of server time and draw tier A with no interpolation buffer [decided].
- Per-fish randomness comes only from `hash32`, built from `bit32` operations [decided]. Never use `Random`: Roblox doesn't promise identical sequences across platforms or engine versions [engine].
- Why closed form, not stepped flocking: stepped boids are chaotic, so tiny floating-point differences between x86 servers and ARM phones compound. A function of `t` is evaluated fresh every time, so its errors never build up.
- The server evaluates a tier-A fish only to validate a bite, aim a predator, or decide relevancy.
- The alive bitmask has one bit per fish. The server clears a fish's bit when it's eaten, and the Eaten event tells clients [decided]. At most 32 fish per school, so the mask fits in one u32 [proposed].
- The path shape, the offset formula, school sizes and respawning aren't designed (see Open questions).

### Scatter [decided]

When a school is disturbed, the server sends a reliable Scatter event `{schoolId, startTime, origin, seed}`. The server and clients then evaluate the same closed-form burst from `startTime`. Only fish that are actually being chased get promoted to tier B. What triggers a scatter isn't designed.

### Promotion to tier B [decided]

- The fish takes a free slot in the tier-B arrays, starting from its closed-form position (and velocity) at the moment of promotion [proposed].
- Clients hide the jump from the closed-form position to the streamed one with an error offset that decays every frame.

### Tier B: reacting fish [decided]

- **Storage:** preallocated parallel arrays (or buffers) with a free-list. Behaviour state is an integer enum, never components that are added and removed.
- **Step:** about 10 Hz, time-sliced across Heartbeat frames under a fixed per-frame budget.
- **Movement:** steering. Full flocking (boids) runs only within this active set, with neighbours found through a spatial hash and capped at 6–7 per fish.
- **History:** a ring buffer of positions at 10–20 Hz, covering about 1 s, for bite validation ([eating-and-bites](eating-and-bites.md)).
- **Network:** streamed as 10 B records ([networking](networking.md)).

### Tier C: AI predators [decided]

jecs 0.11.0 entities that decide at 2–5 Hz and steer at 10–20 Hz. They use the same record format and position history as tier B, streamed at the near rate. Their behaviour isn't designed.

### Counts and server budget [target]

- First milestone: 300 fish. Later, 300–1,000 fish, mostly tier A, if measurements allow.
- Fish simulation plus packet building: p95 ≤ 3 ms per server frame. Whole server frame: p95 ≤ 10 ms.
- An estimate, not a measurement: at a pessimistic 5 µs per tier-B update, 600 active fish at 10 Hz average about 0.5 ms per 60 FPS frame.
- Simulation modules are marked `--!native` [decided]. Parallel Luau is a late lever: live servers reportedly had about 2 worker threads before Roblox switched to player-count-based CPU allocation in November 2025, and the new formula is unpublished [engine].

### Why the swarm isn't in the ECS [decided]

Hundreds of identical records whose state changes constantly belong in parallel arrays, which never trigger archetype moves on the hottest path. jecs is for varied entities: sharks, predators, pickups.

## Invariants

- No fish exists as a server Instance.
- A tier-A position depends only on (school record, fish index, `t`); there is no accumulated state.
- Per-fish randomness comes from `hash32`, never `Random`.
- The simulation allocates no tables per tick.

## Tests

None yet. Planned:

- Lune spec: evaluating the same school at the same `t` through the shared module always gives identical positions.
- Lune spec: golden values for `hash32`, so an accidental change can't silently desync clients.
- A live-server MicroProfiler benchmark of the tier-B step cost per fish.

## Open questions

- Path shapes, the offset formula, school sizes and the species list (the food chain isn't designed).
- What triggers a scatter, how fast the error offset decays, when fish rejoin a school, and respawning.
- Tier-B behaviour states.
- Whether the 5 µs-per-update assumption holds on live servers.

## History

- 2026-10-30 — Created with the planned design from the tech stack research.
