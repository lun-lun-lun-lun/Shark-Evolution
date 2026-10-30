---
title: Networking
status: planned
code: []
decisions: [D5, D6, D7]
related: [architecture, fish-simulation, eating-and-bites, testing-and-performance]
last_verified: n/a (no code yet)
---

# Networking

## Summary

There are two channels. Typed, low-rate events (bites, eats, schools, growth) go through Blink over reliable remotes, behind a size and rate guard. Continuous fish and predator state goes through one hand-written stream of 10-byte quantized records over an UnreliableRemoteEvent, filtered per player by an interest grid. There are no acknowledged delta snapshots and no codec generator, and nothing runs at 60 Hz.

## Spec

### Engine facts [engine]

- An UnreliableRemoteEvent payload can be at most 1,000 B (raised from 900 B on 2025-03-12). The engine compresses before checking, so the real limit is unpredictable: keep packets ≤ 950 B.
- Unreliable messages that are too big or throttled are dropped, not delayed, and they can arrive out of order. Reliable remotes stay in order and get delayed instead of dropped.
- `buffer` reads and writes are little-endian.
- The old "50 KB/s" figure was the legacy physics sender's throttle, not a remote-event limit. On weak links the limit is the link itself: data waits in queues and data ping rises above network ping.
- NextGenerationReplication (forced by Server Authority) no longer guarantees that property changes arrive before a remote fired after them. Refer to entities by numeric id in every remote, never by Instance.
- Blink 0.18 makes one remote call per unreliable fire, so the state stream is one hand-written buffer per packet, not a Blink event.

### Channel 1: Blink events (reliable)

Blink 0.18.9, run with the stock generator; no custom generator [decided].

Planned events. Names and payloads are [proposed]; record the final schema here once it's written.

| Event | Direction | Payload | Notes |
|---|---|---|---|
| BiteClaim | client → server | targetId, renderServerTime | Rate-limited. See [eating-and-bites](eating-and-bites.md). |
| BiteRejected | server → the biting client | targetId, reason | The client shows the target again. |
| Eaten | server → clients that can see it | targetId, eaterId | Removes the target; growth is applied on the server. |
| SchoolEnter | server → one client | School record, about 20–40 B [target] | When a school becomes relevant to that player. |
| SchoolLeave | server → one client | schoolId | When it stops being relevant. |
| Scatter | server → clients that can see it | schoolId, startTime, origin, seed | Clients evaluate the same closed-form burst. |
| Growth | server → one client | Not designed | Defined with the growth design. |

Guard on every client-to-server event, before Blink decodes anything [decided]:

1. Reject any buffer longer than the largest legal encoding of that event.
2. Apply a per-player token bucket per event, and drop over-rate packets.

Why: Blink 0.18 issue #45 lets an oversized buffer crash the server, and the maintainer has deferred the fix to Blink 1.0. Bucket sizes and rates are set at implementation; record them here.

### Channel 2: the state stream (unreliable)

- One UnreliableRemoteEvent; each fire carries one `buffer` [decided].
- Packets are ≤ 950 B [decided].
- Every packet carries a u16 sequence number. Receivers compare with wraparound and drop anything older than the newest packet already applied [decided].
- The rest of the header isn't designed yet (see Open questions).

Record layout, 10 B, for tier-B fish and tier-C predators:

| Bytes | Field | Type | Encoding | Tag |
|---|---|---|---|---|
| 0–1 | id | u16 | Entity id, unique among live streamed entities | [decided] |
| 2–3 | x | u16 | round((x − mapMin.X) × 8), clamped to 0–65,535: 1/8-stud steps | [decided] |
| 4–5 | y | u16 | Same encoding for y | [decided] |
| 6–7 | z | u16 | Same encoding for z | [decided] |
| 8 | yaw | u8 | 256 steps per full turn (1.40625° each) | [proposed] |
| 9 | pitch | u8 | 256 steps from −90° to +90° (about 0.70° each) | [proposed] |

- A u16 at 1/8 stud covers 8,192 studs per axis from `mapMin`. The map size isn't designed; if the map is bigger, use coarser steps or coordinates relative to the cell.
- One packet holds about 90 records: (950 B − header) / 10 B.

Send rates [decided]: 10–20 Hz for entities near the player, 2–5 Hz farther out.

Interpolation on the client [decided]:

- Draw streamed entities about 3 send intervals behind server time, plus a margin for jitter: about 300–350 ms at 10 Hz, about 150 ms at 20–30 Hz. That survives two lost packets in a row.
- Use Hermite interpolation, which removes the wobble that linear interpolation shows at 10 Hz. The record carries no velocity, so estimate it from neighbouring samples [proposed].

### Interest management

- A uniform grid with cells 128–256 studs across [target; pick by measurement].
- Each player's interest radius matches the fog distance, about 200 studs [decided].
- An entity becomes relevant inside the radius, and stops being relevant only beyond the radius plus a 20–40-stud hysteresis band [target].
- Priority accumulator [decided]: priority grows every tick until something is sent; packets fill in priority order up to the byte budget; whatever didn't fit goes first next tick. Because cells are encoded once (below), priority accumulates per (player, cell) [proposed].
- When something becomes relevant, send its full state: SchoolEnter for a school; tier-B and tier-C records in the next packet, at raised priority [proposed].

### Encode once [decided]

Each tick, encode every active cell's records once into a shared buffer. Build each player's packet by copying in the cells they need (`buffer.copy`). Never re-encode per player: per-player encoding is the kind of server work SMASH couldn't afford.

### Budget

- Per-client ceiling: 32 KiB/s [decided]. It's a monitored limit, not a target to fill.
- Typical: 10,000–20,000 B/s per player [target]. Example: 100 nearby records × 10 B × 10 Hz = 10,000 B/s, before engine compression.

## Invariants

- No client-to-server packet reaches Blink's decoder without passing the size and rate guard.
- No unreliable packet exceeds 950 B.
- A state packet older than the newest one applied is never applied.
- Remotes refer to entities by numeric id, never by Instance.
- Shared state is encoded once per tick, not once per player.

## Tests

None yet. Planned:

- Lune spec: the record codec round-trips, clamps at the map edges, and compares sequence numbers correctly across wraparound.
- Lune spec: the guard rejects oversized and over-rate packets before decoding.
- Byte-budget spec: a 20-player, 300-fish scene stays inside the per-client budget.

## Open questions

- Header layout. Besides the u16 sequence, the client needs to place each packet on the server timeline (server time or tick) and know the record count and kind.
- How u16 entity ids are allocated and reused, and how a stale packet referring to a reused id is rejected.
- How `targetId` encodes a tier-A fish (school id + fish index) versus a tier-B or tier-C entity id.
- Where "near" ends for the 10–20 Hz rate versus 2–5 Hz.
- Cell size, and whether cells are 2D columns or 3D boxes.
- Whether other players' sharks stay on engine physics replication or move into this stream ([architecture](architecture.md)).
- Whether estimated velocities are smooth enough, or the record needs a quantized velocity (3–6 more bytes).
- A time-boxed trial of BlinkBlox: a Blink 0.18 fork with size caps checked before parsing, rate limits, quantized floats and per-player send budgets. It's one month old and its benchmarks are self-reported.
- Moving to Blink 1.0 once it ships quantized numbers, bit-packing and size checks.
- Whether to commit Blink's generated modules (SMASH did, so builds don't need the CLI) or generate them in CI.

## History

- 2026-10-30 — Created with the planned design from the tech stack research.
