# Roblox networking libraries, serialization, and engine network limits for mobile players on poor connections (state as of 30 Oct 2026)

Note on sourcing: Roblox Creator Hub pages were read from the official `Roblox/creator-docs` GitHub repo at commit `c8f5276` (29 Oct 2026). The links below point to the matching create.roblox.com pages. GitHub release dates, stars and issue states come from the GitHub API and repo clones taken on 30 Oct 2026. DevForum dates, view counts and post counts come from the DevForum JSON endpoints on the same day.

---

## 1. How do Blink, Zap, ByteNet (and successors), Packet, Warp and the 2025–2026 newcomers compare?

### Takeaway
Only two IDL compilers are mature and maintained: **Blink** (stable v0.18.9, 19 Sep 2026, with a from-scratch 1.0 rewrite in pre-release) and **Zap** (v0.6.29, 23 Jun 2026, which ships less often). The original **ByteNet** is effectively dormant; its community successor is **ByteNet Max**. **Packet** (Suphi Kaner) is the most-discussed runtime-schema library. **Warp** 1.1 is still in "pre" releases. Newer entrants from 2025–2026 are Jolt, QuickNet, NetRay Compile and BlinkBlox (a security-focused Blink 0.18 fork). They all produce about the same bandwidth because they all pack into buffers that the engine then compresses. They differ in CPU cost, ergonomics, unreliable-channel handling and above all hostile-input hardening. Blink 0.18.x, Zap 0.6.29 and ByteNet each have an open server-DoS/crash issue involving oversized client buffers.

### Cited Findings

**Blink (1Axen): IDL compiler written in Luau**
- Repo self-description: "An IDL compiler written in Luau for ROBLOX buffer networking". Created 2023-12-24. As of 30 Oct 2026: 194 stars, 35 forks, 20 open issues, last push 2026-09-23 — [GitHub 1Axen/blink](https://github.com/1Axen/blink)
- Latest stable is **v0.18.9, published 2026-09-19**. **v1.0.0-pre.10** came out the same day as a prerelease. Earlier pre-releases: pre.9 on 2026-09-18, pre.8 on 2026-09-16, pre.7 on 2026-08-24, and pre.6 / pre.5 / pre.4 on 11 Apr / 15 Mar / 09 Feb. The page shows no year for those last three, but release ordering and a Feb 2026 issue that mentions "v1.0.0-pre.2" place them in 2026 — [Blink releases](https://github.com/1Axen/blink/releases); [issue #78](https://github.com/1Axen/blink/issues)
- **v0.18.9 fixes a validation bypass:** "NaN scalars and vectors bypassing inexact range validation (`f64(A...B)`, `vector(0..1)`)". So 0.18.8, the SMASH version, accepts NaN in range-checked client fields — [Blink releases](https://github.com/1Axen/blink/releases); [DevForum announcement post #171, 2026-09-19](https://devforum.roblox.com/t/2959671)
- v0.18.8 (11 Apr, 2026 inferred) fixed "DateTime and DateTimeMillis reads advancing the cursor 1 byte instead of 8 (#82)" and several plugin editor issues — [Blink releases](https://github.com/1Axen/blink/releases)
- **1.0 is a rewrite.** It lives on a separate `rewrite` branch, is still Luau, and is built with lute (HEAD dated 2026-09-19). Changes so far: an `@profile` attribute for build profiles; write validations disabled in the `release` profile; an `integer` (int64) type that sits behind an FFlag; removal of the `stable` and `bitpack` attributes and the `*_library` options; `@export` on generics becomes an error. pre.9 fixed "events received before the first `on()` connection not being consumed", "errors when firing to a departed player" and "malformed outgoing buffers after client-side serialization errors" — [Blink releases](https://github.com/1Axen/blink/releases); [rewrite branch](https://github.com/1Axen/blink/tree/rewrite)
- Several features are open "rewrite" enhancements filed 2026-09-15, so they do **not** exist in 0.18.x:
  - #104 "Fixed-point / quantized numerals: encode a bounded float into a small integer"
  - #105 "@bitpack: pack consecutive booleans…"
  - #108 "Compile-time payload size analysis: reject or warn on Unreliable events that can exceed th[e limit]"
  - #102 "Error policy for invalid incoming data"
  - #107 "invoke: timeout, cancellation on player leave"
  - #111 "Use a thread pool for Many* events" (2026-09-18)

  — [Blink open issues](https://github.com/1Axen/blink/issues)
- **Security:** issue **#45 "Server DOS attack via large buffer input"** is open (filed 2025-05-22). The report says that if a client sends e.g. `buffer.create(1000000)`, "the server attempts to parse the entire buffer within a loop", which can crash the server. The maintainer replied: "This is a know issue, but it's out of blink's security scope… I have plans to mitigate this as part of the ongoing rewrite." — [Blink issue #45](https://github.com/1Axen/blink/issues/45)
- README security claim: "Data sent by clients will be **validated** on the receiving side before reaching any critical game code" — [Blink README](https://github.com/1Axen/blink)
- Event model: `From` (Server/Client); `Type` is `Reliable` or `Unreliable` ("maximum size of 1000 bytes"); `Call` is `SingleSync`, `ManySync`, `SingleAsync`, `ManyAsync` or `Polling` (iterate with `Event.Iter()`) — [Blink docs: Events](https://github.com/1Axen/blink/blob/main/docs/pages/language/5-events.mdx)
- Options:
  - `ManualReplication` (default false): "Controls if Blink will replicate events and functions automatically at the end of every frame". When true, Blink exposes `StepReplication` instead.
  - `SyncValidation` (default true) checks whether a sync callback yielded.
  - `WriteValidations` (default false) checks types on send, but "Blink only checks for builtin primitives… structs, maps and enums cannot be validated".
  - `UsePolling` outputs every event with a polling API.

  — [Blink docs: Options](https://github.com/1Axen/blink/blob/main/docs/pages/language/1-options.mdx)
- How the 0.18.9 generator sends: reliable events go into a shared per-frame send buffer that is flushed at frame end. Each **unreliable** fire copies its serialized bytes into a fresh buffer and immediately calls `Unreliable:FireServer` / `FireClient` / `FireAllClients`. Unreliable sends are therefore one remote invocation per fire, not frame-coalesced — [Blink Generator source](https://github.com/1Axen/blink/blob/main/src/Generator/init.luau)
- Blink's own benchmark was **last updated 2025-04-30** with blink v0.17.1, zap v0.6.20 and bytenet v0.4.3, on a Ryzen 9 7900X. Each run fires an event "1000 times per frame with the same data every frame for 10 seconds".
  - "Entities", median FPS: roblox 16, blink 42, zap 39, bytenet 32.
  - "Entities", median Kbps: roblox 559,364, blink 41.81, zap 41.71, bytenet 41.64.
  - "Booleans", median FPS: roblox 21, blink 97, zap 52, bytenet 35.

  — [Blink Benchmarks.md](https://github.com/1Axen/blink/blob/main/benchmark/Benchmarks.md)
- DevForum thread: created 2024-05-06, 171 posts, 30,489 views, 267 likes, last post 2026-09-19. A 2026-07-20 reply reports that the plugin's config editor becomes "unbearably laggy" at about 800 lines, with each keystroke taking about 1 s. GitHub issue #12 ("Plugin lags with larger config files") has been open since 2024-07-02 — [Blink DevForum thread](https://devforum.roblox.com/t/2959671); [Blink issues](https://github.com/1Axen/blink/issues)
- Community view in Sep 2026: Blink was recommended for performance and described as "more active" than Zap and "designed to ditch that packet" when a packet grows too large. One user said Blink "averaged ~700 kb/s" in their system, with no further context — [DevForum: What networker should I use? (Sep 2026)](https://devforum.roblox.com/t/what-networker-should-i-use/4889485)
- Perception of maintenance mode, from the BlinkBlox author (Sep 2026): the 0.18 line "was only getting major fixes" while the project moved "to a from-scratch rewrite" — [BlinkBlox DevForum thread](https://devforum.roblox.com/t/4898948)

**Zap (red-blox): IDL compiler written in Rust, with a wasm playground build**
- "A lightning fast networking solution for roblox". Created 2023-12-14. 190 stars, 30 forks, 9 open issues. **Latest v0.6.29, published 2026-06-23**, which is also the last push. Previous releases: 0.6.28 (2025-12-13), 0.6.27 (2025-10-05), 0.6.26 (2025-09-28), 0.6.25 (2025-07-21) — [GitHub red-blox/zap](https://github.com/red-blox/zap); [Zap releases](https://github.com/red-blox/zap/releases)
- Event types are `Reliable` and `Unreliable` ("maximum size of 1000 bytes"). Calls are `ManyAsync`, `ManySync`, `SingleAsync`, `SingleSync`, plus `Polling` from 0.6.18. Two-way events are not supported. The docs say "events have almost no overhead" — [Zap docs: events](https://github.com/red-blox/zap/blob/main/docs/config/events.md)
- `manual_event_loop`: Zap "automatically sends reliable events and functions each Heartbeat" unless this option is set, in which case you call `SendEvents`. A docs warning dated **January 2024** says Roblox had an issue where "firing remotes at too high of a rate (above 60 hz)" causes very high network response times that "essentially crash" servers, and suggests capping `SendEvents` at 60 Hz. Treat this as **dated** — [Zap docs: options](https://github.com/red-blox/zap/blob/main/docs/config/options.md)
- The compiler enforces the unreliable size limit at compile time. Source sets `MAX_UNRELIABLE_SIZE = 998` and reports "all unreliable events must be under 998 bytes in size" — [Zap convert.rs](https://github.com/red-blox/zap/blob/main/zap/src/parser/convert.rs)
- An **undocumented `OrderedUnreliable` event type** exists in the grammar. It prepends a u16 order id that the receiver uses to discard stale packets — [Zap grammar.lalrpop](https://github.com/red-blox/zap/blob/main/zap/src/parser/grammar.lalrpop); [Zap client codegen](https://github.com/red-blox/zap/blob/main/zap/src/output/luau/client.rs)
- Claims: Zap "packs all data into buffers, which… also compress when passed through RemoteEvents" and "Zap validates all data recieved" — [What is Zap](https://github.com/red-blox/zap/blob/main/docs/intro/what-is-zap.md)
- **Open issues:**
  - **#219 (2026-03-25):** "exploiters can crash entire game servers… with oversized buffers because of constant long loops… This leads to an out of memory crash". The reporter patched `output.rs` locally to fix it.
  - **#216 (2026-01-08):** a `Player` is memory-leaked if `Fire` is called after `PlayerRemoving`.
  - **#225 (2026-09-10):** large games need sharding of the generated module, "forcing a fork".

  — [Zap #219](https://github.com/red-blox/zap/issues/219); [Zap #216](https://github.com/red-blox/zap/issues/216); [Zap issues](https://github.com/red-blox/zap/issues)
- A competitor claims Zap has "no rate limiting at all (its maintainer's position in issue #219 is that throttling is the game's job) and no `pcall` around each event, so one malformed event throws away the rest of that player's batch". This is unverified and comes from a rival library's author — [BlinkBlox DevForum thread](https://devforum.roblox.com/t/4898948)
- No dedicated Zap DevForum resource thread turned up in DevForum title search. Zap is mainly distributed through GitHub, its docs site and the Roblox OSS community — [zap.redblox.dev](https://zap.redblox.dev)

**ByteNet (ffrostfall), now dormant, and ByteNet Max (community successor)**
- ByteNet is a runtime-schema library ("Simple, buffer-based networking"). Created 2023-12-06, 185 stars. **GitHub releases stop at v0.4.3 (2024-03-10, flagged prerelease).** The CHANGELOG goes up to 0.4.6 (Color3 and CFrame fixes). **Last commit 2025-08-01** ("remove the extremely pathetic dev environment") — [GitHub ffrostfall/ByteNet](https://github.com/ffrostfall/ByteNet); [ByteNet CHANGELOG](https://github.com/ffrostfall/ByteNet/blob/main/CHANGELOG.md)
- ByteNet supports unreliable sends (`sendPlayerUnreliable`, `sendAllUnreliable`, `send_unreliable`). On the client these write into a separate unreliable `ChannelState` buffer next to the reliable one. 0.4.2 "Fixed sending unreliable events from client -> server" — [ByteNet source](https://github.com/ffrostfall/ByteNet/tree/main/src); [CHANGELOG](https://github.com/ffrostfall/ByteNet/blob/main/CHANGELOG.md)
- Open issues include **#34 "Ability to crash servers" (2025-04-11)**, #33 "Data typechecking" (2025-03-25) and #26 "CFrames sent to Server doesn't seems to be accurate" — [ByteNet issues](https://github.com/ffrostfall/ByteNet/issues)
- The DevForum thread has 68,669 views and 481 likes. A 2026-06-01 reply says "it just doesnt work… raises constant errors". A 2026-07-07 reply recommends ByteNet Max, which "does fix alot of the issues with bytenet (including crashing vunerabilitys…)" — [ByteNet DevForum thread](https://devforum.roblox.com/t/2733365)
- **ByteNet Max** (by Elitriare) is a fork that adds "queries" (RemoteFunctions). Thread created 2024-11-21, titled v0.2.1, with Wally linking 0.2.7; 188 posts, 27,404 views. On 2026-10-04 a user reported it "has been removed from the creator store" and asked for .rbxm files on GitHub — [ByteNet Max DevForum thread](https://devforum.roblox.com/t/3268469); [GitHub Elitriare/ByteNet-Max](https://github.com/Elitriare/ByteNet-Max)
- A 2026 thread on choosing a networking library praised ByteNet's "great performance" and "great ergonomics" — [DevForum: too many networking libraries (Apr 2026)](https://devforum.roblox.com/t/there-are-too-many-networking-libraries-which-do-i-choose/4571749)

**Packet (Suphi Kaner / 5uphi): runtime schema, Creator Store distribution**
- Features:
  - "Batches all your events into a single event" and "Serializes all data into one buffer".
  - "16 bit floats + 24 bit floats".
  - "DDoS protection".
  - RemoteFunction-style calls via `Response()`.
  - "UnreliableRemoteEvent support", provided as a documented **wrapper**: you create your own `UnreliableRemoteEvent` and call `packet:Serialize(...)` / `Deserialize`.

  The API is `Packet("Name", Packet.NumberS8, Packet.Vector3F32, Packet.Instance)` — [Packet DevForum thread](https://devforum.roblox.com/t/packet-networking-library/3573907)
- Version history: v1.3 "Added rate limit to how many events the client can send per server heartbeat [this only effects exploiters]" and "Client… will fire a maximum of 60 events every second". **Latest is 1.7**, which notes "Fixed vulnerability"; no release date is shown — [Packet thread](https://devforum.roblox.com/t/packet-networking-library/3573907)
- Author's byte sizes, compared with Roblox and ByteNet:
  - CFrame: Packet 12, 15 or 18 B; Roblox 20 B; ByteNet 24 B.
  - Vector3: Packet 6, 9 or 12 B; ByteNet 12 B.
  - Float16: 2 B. Float24: 3 B.
  - Booleans: 1 bit or 8 bits.

  — [Packet thread](https://devforum.roblox.com/t/packet-networking-library/3573907)
- Vulnerabilities reported on launch day (2025-03-26):
  - A client could resume any coroutine waiting on a response.
  - The 8,000-byte limit could be bypassed.
  - Partial or invalid buffer reads were possible.

  The thread does not show how each one was resolved — [Packet thread](https://devforum.roblox.com/t/packet-networking-library/3573907)
- Adoption is the highest of any library thread: created 2025-03-26, last post 2026-10-20, 509 posts, 72,327 views, 778 likes. A 2026-10-20 reply says that "using unreliable remote events in conjuction with the packet module feels rather counterintuitive". The author said they had never used Zap or Blink — [Packet thread](https://devforum.roblox.com/t/packet-networking-library/3573907)

**Warp (imezx)**
- "Dynamic by default. It serializes and deserializes data dynamically without requiring a user-defined schema, although schema support is available as an option". Created 2024-01-05. 35 stars, 0 open issues. **Latest 1.1.0-pre8, 2026-10-09**; pre1 to pre7 were released 2026-02-11 to 2026-04-22 — [GitHub imezx/Warp](https://github.com/imezx/Warp)
- 1.1 adds a Buffer schema system, including `"cframe"` encoded as "f32 position + compressed rotation (f16)". 1.0 has per-event rate limits configured on the server. pre7 and pre8 added and refactored "history xor" encoding for reliable traffic, and pre8 added "float32 precision for Vector3 & Vector2" — [Warp 1.1 buffer docs](https://github.com/imezx/Warp/blob/main/docs/api/1.1/buffer.md); [Warp ratelimit docs](https://github.com/imezx/Warp/blob/main/docs/api/1.0/ratelimit.md); [Warp DevForum posts #460/#462](https://devforum.roblox.com/t/2779813)
- The thread was created 2024-01-05, has 463 posts and 46,119 views, and was last active 2026-10-28. A 2026-10-28 user said 1.1 was hard to figure out from the docs. A search-engine snapshot showed the thread titled "[ARCHIVED] Warp…", but the live thread title has no such tag, so its status is **conflicting** — [Warp DevForum thread](https://devforum.roblox.com/t/2779813)

**Newer options (2025–2026)**
- **Jolt** (ToriumSlurs) is a runtime library at Wally `4.4.1`. It does binary serialization and "Accumulates all `:Fire()` calls during a frame and sends them in a single packet on `RunService.Heartbeat`". Channel names are hashed to 16-bit (FNV-1a). It has `FireUnreliable` / `FireAllUnreliable` and `Invoke` with a 30 s timeout. v4.0.0 (2026-08-29) changed the wire format, and v4 cannot talk to v3. Thread: 2025-11-24 to 2026-10-30, 167 posts, 11,413 views — [Jolt DevForum thread](https://devforum.roblox.com/t/4095212)
- **QuickNet** (breadboardengineer1234) is at v0.4.0. It is a drop-in RemoteEvent-style API with no code generation and claims "reduce the network traffic by as much as 1,000x, while running at up to 10x higher FPS" compared with RemoteEvents. It advertises "Protection against invalid data/'DDoS'", "Custom rate limiting" and unreliable events. Its benchmarks use the same 1000-fires-per-frame, same-data method. Thread: 2026-05-07 to 2026-10-22, 148 posts, 11,960 views — [QuickNet DevForum thread](https://devforum.roblox.com/t/4624342)
- **NetRay Compile** is a Luau IDL compiler in **beta**. v0.2.0 [BETA] (2026-10-28) added optional per-event rate and size limits plus delta compression ("Compression: Repeat and Compression: Fields, with recipient baselines"). It has 3,767 views. The older runtime NetRay thread (2025-04 to 2026-07) has 325 posts — [NetRay Compile thread](https://devforum.roblox.com/t/4348861); [NetRay thread](https://devforum.roblox.com/t/3592849)
- **BlinkBlox** (XopoIII, announced 2026-09-27) is a fork of Blink 0.18 that keeps the `.blink` schema. It adds:
  - "Per-player, per-event token buckets (`Rate`, `Burst`…), a per-player inbound byte budget".
  - "Packet size, event count and Instance count are capped *before* anything is parsed".
  - Quantized floats: "`f32<0.01>(-1..1)` is 1 byte instead of 4".
  - "A per-player outbound budget: events marked `Priority: Low` are the first to be left out when a player's connection is full".
  - "An unreliable event that can't fit in a packet is refused at compile time".

  — [BlinkBlox DevForum thread](https://devforum.roblox.com/t/4898948); [GitHub XopoIII/BlinkBlox](https://github.com/XopoIII/BlinkBlox)
- BlinkBlox's **self-reported** benchmarks (i7-13700K, run on LuneBlox 0.10.9) compare BlinkBlox 0.42.0 against Blink 0.18.9, zap 0.6.29, ByteNet 0.4.3, Packet 1.7.0, QuickNet 0.3.5 and Warp 1.1.0-pre7:
  - Server decode of 100 structs × 1000 per frame: BlinkBlox 3.58 ms, Blink 21.83, zap 21.16, ByteNet 42.49, Packet 52.70, QuickNet 11.37, Warp 26.91.
  - `boolean[1000]`: 128 B for BlinkBlox, QuickNet and Warp; 1003 B for Blink, zap, ByteNet and Packet.
  - `FireAll` of 100 structs to 50 players: Blink "0.711 ms, 50 calls", zap 0.734 ms / 50 calls, ByteNet 0.044 ms / 1 call.
  - Studio FPS at 1000 random structs per frame: RemoteEvents 15, Blink 45, zap 44, ByteNet 34, Packet 27.

  — [BlinkBlox DevForum thread](https://devforum.roblox.com/t/4898948)
- Other 2026 entrants found by DevForum title search: PostService (2026-10-04), Packeteer (2026-07-15) and Network Module (NM) v2.3.4 (2026-03-22). The NM author says Roblox flagged it for "misusing roblox systems" — [PostService](https://devforum.roblox.com/t/4915064); [Packeteer](https://devforum.roblox.com/t/4739399); [Network Module](https://devforum.roblox.com/t/4529765)
- April 2026 community advice was split: ByteNet, Sera (a serialization layer, "The most important thing is compression"), Jolt, Blink, or a custom module (the "Best option") — [DevForum: too many networking libraries](https://devforum.roblox.com/t/there-are-too-many-networking-libraries-which-do-i-choose/4571749)

### Inferences
- There are two architectures:
  - **IDL compilers** (Blink, Zap, NetRay Compile, BlinkBlox) generate specialized read/write code per event. They give the lowest CPU cost and full static types, and the generated reader validates schema types and ranges.
  - **Runtime-schema libraries** (ByteNet/ByteNet Max, Packet, Warp, Jolt, QuickNet) build codecs at runtime. They are easier to adopt but usually cost more CPU (see the decode numbers above).

  For a server that already missed its frame budget once, generated code is the better fit.
- On bandwidth the buffer libraries are indistinguishable. In Blink's own benchmark Blink, Zap and ByteNet all sit at about 42 Kbps on "Entities". The deciding factors are therefore server CPU, hostile-input safety, and how much quantization the schema can express.
- None of the three "classic" libraries (Blink 0.18.9, Zap 0.6.29, ByteNet 0.4.x) bounds the size of a hostile client buffer before decoding, and each has an open crash/DoS report. Any choice except BlinkBlox (and NetRay Compile's opt-in limits) needs a game-side guard that checks `buffer.len` and rate before decoding, or a patched generator.
- Blink 0.18.x sends each unreliable fire as its own remote call (read from the generator). Many tiny unreliable events per frame therefore each carry per-call overhead. Hot unreliable data should be packed into one event per tick, staying under the 1000-byte cap.
- Blink 1.0's planned features are essentially what SMASH's custom generator did by hand: quantized numerals, bitpacking, compile-time unreliable size checks and an invalid-data policy. They are not released yet.

### Gaps
- There is no independent (non-author) 2025–2026 benchmark of these libraries on **mobile hardware**. Every benchmark found runs on desktop Studio/Luau and fires identical data each frame, which inflates the benefit of the engine's compression.
- Packet 1.7's release date, and whether the March 2025 vulnerability reports are all fixed, could not be confirmed. Packet's source is distributed through the Creator Store, not a public Git repo.
- Download counts from Wally, pesde or the Creator Store are not publicly available, so adoption is approximated with GitHub stars and DevForum views/likes.
- The Zap maintainer's own comments on #219 (3 comments) were not read directly, so BlinkBlox's description of Zap's position is unverified.
- No public timeline exists for Blink 1.0 stable.

---

## 2. Engine facts as of 2026: RemoteEvent vs UnreliableRemoteEvent, per-call overhead, batching/compression, `buffer`, bandwidth budgets, and what happens when they are exceeded

### Takeaway
- **RemoteEvent** is reliable and ordered. Client sends over the throttle are delayed, not dropped.
- **UnreliableRemoteEvent** is unordered and unreliable. Its payload cap is **1000 bytes**, raised from 900 on 12 Mar 2025, and anything over the throttle or the size limit is **dropped**.
- Client-to-server traffic is limited to about **500 requests/s per client**, shared by all remotes of the same type.
- Buffers are encoded and compressed by the engine.
- The folkloric "~50 KB/s" figure was the **legacy Physics Sender's throttle**. A Roblox engineer says that under NextGenerationReplication, physics and property updates share a limit "much higher than 50KB/s".
- When a connection can't keep up, data waits in **replication queues** and "data ping" rises above network ping.

### Cited Findings
- **RemoteEvent delivery** (official docs):
  - Lost messages are retransmitted "until it arrives, as long as the recipient stays connected".
  - Order holds across different RemoteEvent instances.
  - "If a client fires events faster than the throttling limit, the server processes the excess events later instead of dropping them, and they keep their order."
  - With no handler connected, messages queue; the queue "is limited in both count and memory", and once it is full further messages are discarded with a `Remote event invocation` error.
  - Ordering applies to when each handler starts, not when it finishes.

  — [Remote events and callbacks](https://create.roblox.com/docs/scripting/events/remote)
- RemoteFunction and RemoteEvent are **not ordered relative to each other**: "send all related messages through the same remote type or include your own sequence number" — [Remote events and callbacks](https://create.roblox.com/docs/scripting/events/remote)
- **UnreliableRemoteEvent delivery:**
  - A message may be dropped if it is lost, if "The payload is larger than 1,000 bytes", or if the client fires faster than the throttle. "Unlike a RemoteEvent, the server drops the excess messages instead of delaying them."
  - If no listener is connected, a message "is discarded immediately, not queued".
  - The docs recommend a timestamp or sequence number and ignoring messages older than the latest one handled.

  — [Remote events and callbacks](https://create.roblox.com/docs/scripting/events/remote)
- **Throttle:** "RemoteEvents and UnreliableRemoteEvents both have a limit of approximately 500 requests per second, per client. This limit is **shared among all remote events of the same type**." This applies to client-to-server `FireServer` — [RemoteEvent reference](https://create.roblox.com/docs/reference/engine/classes/RemoteEvent); [UnreliableRemoteEvent reference](https://create.roblox.com/docs/reference/engine/classes/UnreliableRemoteEvent)
- **Unreliable size and compression:** "Events with payloads larger than 1000 bytes are dropped. When this happens in Studio, a log message in the Output window indicates the number of bytes the event went over." The methods "encode and compress certain object types, such as buffers, which shrinks the payload size and can make it difficult to verify whether you are under the limit prior to firing the event" — [UnreliableRemoteEvent reference](https://create.roblox.com/docs/reference/engine/classes/UnreliableRemoteEvent)
- **History of the limit:**
  - Launched 2023-11-29 with "payloads should be at most 900 bytes" as a known issue.
  - The launch post also says "UnreliableRemoteEvents may be dropped to prioritize bandwidth or CPU usage in addition to any loss that occurs over the network".
  - **Update of March 12, 2025:** "Payload limit has been increased to 1000 bytes on both Client and Server. Fixed a bug where the limit was inconsistent betw[een client and server]".

  So SMASH's 880-byte packets were sized for the old 900-byte limit — [Introducing UnreliableRemoteEvents (DevForum)](https://devforum.roblox.com/t/introducing-unreliableremoteevents/2724155)
- In July 2024 a developer reported that buffers above the stated limit could be sent in practice, which is consistent with the limit applying after compression — [DevForum t/3048788](https://devforum.roblox.com/t/3048788)
- **Per-call overhead (unofficial, 2023 measurement):** Pyseph's RemotePacketSizeCounter uses `BASE_REMOTE_OVERHEAD = 9` bytes per remote event, +5 for client-to-server, +2 for RemoteFunction, and a 1-byte type tag per argument. Its example: "Vector3: 3 float32s, 1 byte overhead: 3*4 + 1 = 13 bytes". **Not official and possibly outdated** — [RemotePacketSizeCounter source](https://github.com/Pyseph/RemotePacketSizeCounter/blob/main/src/init.luau); [DevForum thread (2023)](https://devforum.roblox.com/t/2320709)
- Community size rules of thumb (Dec 2025): CFrame about 20 B, Vector3 about 12 B, Vector3int16 about 6 B, Luau number 8 B (f64). A dictionary key like `"CFrame"` adds about 8 B, so the post recommends arrays over dictionaries — [Downrest: custom NPC replication tips (2025-12-31, updated 2026-02-01)](https://devforum.roblox.com/t/4202236)
- **How the engine batches** (MicroProfiler docs):
  - "Engine network batch data includes traffic due to streaming, instance property updates, and events."
  - `Allocate Bandwidth and Run Senders/Dispatch Data Senders` "Sends property changes, remote event invocations, Humanoid state changes, animation state changes, and replication of new instances".
  - Separate senders exist for physics/touch (`Dispatch Physics Senders and TouchSenders`) and streaming (`Dispatch StreamJob`).
  - Receive side: `Net PacketReceive`, `deserializePacket` and `Replicator/ProcessPackets`.

  — [MicroProfiler network](https://create.roblox.com/docs/performance-optimization/microprofiler/network); [MicroProfiler tag table](https://create.roblox.com/docs/performance-optimization/microprofiler/tag-table)
- **Official optimization guidance:**
  - "Excessive remote traffic… can lead to a large amount of CPU time being spent processing incoming packets each frame".
  - Avoid "Replicating data every frame that does not need to be replicated".
  - "Server-side TweenService… the tweened property is replicated to each client every frame".
  - "creating and destroying large instance hierarchies… at runtime can be very network intensive".
  - Create visual effects on the client.

  — [Improve performance: Networking and replication](https://create.roblox.com/docs/performance-optimization/improve)
- **The "50 KB/s" figure explained by a Roblox engineer** (Khanovich, 2026-07-09): "the old Physics Sender had about a 50 KB/s limit that if you started hitting, it would start throttling and reduce the frequency of updates on parts to fit things into this limit. With `Workspace.NextGenerationReplication`, physics updates and property updates are all put on the same bandwidth limit (much higher than 50KB/s)… In theory you shouldn't need to worry about going over 50KB/s anymore" — [DevForum: Server Authority Issues](https://devforum.roblox.com/t/4728153)
- Origin of the community budget: MrChickenRocket's guest article (2024-08-20) says "Aim for under 50KB/s network receive down on client - (40-60 moving assemblies…)" and calls the numbers "really just my opinion". **Older source** — [Real world building and scripting optimization for Roblox](https://devforum.roblox.com/t/3127146)
- Real 2026 numbers from a 12-player shooter with Server Authority: receive "near 100 KB/s and can fluctuate or spike above 200 KB/s". Disabling Server Authority dropped it "by nearly 100 KB/s". An empty baseplate with a full server ran about 60–70. The engineer said physics replication "jumps from 15Hz to 60Hz, which would lead to about a 4x increase in bandwidth" (hedged with "I think") — [DevForum: Server Authority Issues](https://devforum.roblox.com/t/4728153)
- **What happens when exceeded:** "Data ping… includes time spent in both the client's and server's replication queues, along with TCP-like retransmissions of lost or corrupted packets… When the server generates replication data faster than the network can send it (or faster than the client can process it), data must wait in replication queues… if data ping is significantly higher than network ping, use the MicroProfiler to view network traffic" — [Identify performance issues: Ping](https://create.roblox.com/docs/performance-optimization/identify)
- **Bandwidth APIs:**
  - `NetworkPeer:SetOutgoingKBPSLimit(limit)` (PluginSecurity) sets "maximum outgoing bandwidth… in kilobits per second". The value is clamped to 10–10,000, and 0 or less removes the limit — [NetworkPeer reference](https://create.roblox.com/docs/reference/engine/classes/NetworkPeer)
  - `Stats.DataReceiveKbps` / `DataSendKbps` / `PhysicsReceiveKbps` / `PhysicsSendKbps` describe "roughly how many kilobytes of data… per second" (the property name says Kbps but the description says kilobytes) — [Stats reference](https://create.roblox.com/docs/reference/engine/classes/Stats)
- **`buffer` type:** "Size limit is 1 GiB, or 1,073,741,824 bytes". The library includes bit-offset read/write functions (documented with a `bitOffset` parameter) — [buffer library reference](https://create.roblox.com/docs/reference/engine/libraries/buffer)
- Zap's docs say buffers "compress when passed through RemoteEvents" — [What is Zap](https://github.com/red-blox/zap/blob/main/docs/intro/what-is-zap.md)

### Inferences
- With per-frame batching, a client sends at most about 60 reliable and about 60 unreliable remote calls per second, well under the ~500/s per-type limit. The throttle only bites if code fires many unbatched unreliable events, as Blink 0.18 does for each unreliable fire.
- The engine no longer hard-throttles at 50 KB/s under NextGenerationReplication, but a poor mobile link (low throughput, loss, jitter) becomes the binding constraint. Overshoot shows up as data ping climbing above network ping because of queueing, and reliable traffic suffers head-of-line delays from retransmission. A self-imposed per-client budget is therefore still worth keeping. SMASH's ~32 KiB/s is in line with the community's 50 KB/s target and leaves headroom.
- Engine compression shrinks repetitive buffers, so library benchmarks that repeat identical data each frame overstate savings. Real fish-state traffic (noisy positions) will compress far less.
- Keep a margin under 1000 B before compression (for example ≤ about 950 B, or Zap's 998 B compile-time cap). The compressed size is not observable before firing, and Studio only logs the overshoot after the fact.

### Gaps
- Roblox does not document the compression algorithm, its thresholds, or the per-call wire overhead. The 9-byte figure is a 2023 community measurement.
- No official number exists for the "much higher" shared bandwidth limit under NextGenerationReplication, or for server-to-client throttling of unreliable events. Only the client-to-server ~500/s limit is documented.
- No official 2025–2026 Roblox guidance gives a recommended per-player downstream budget for mobile.

---

## 3. Property replication vs remotes; StreamingEnabled; why experiences render NPCs client-side

### Takeaway
- Under **legacy replication**, property updates were reliable and roughly FIFO (a far-away Color change could delay a nearby weld CFrame), while physics updates were unreliable and relevancy-prioritized at about 15–20 Hz.
- **NextGenerationReplication** (required for Server Authority, and it requires StreamingEnabled) prioritizes everything and can raise physics replication to 60 Hz (about 4× bandwidth). It also **removes the ordering guarantee between property updates and RemoteEvents**, and costs extra server CPU.
- **StreamingEnabled** cuts join time, memory and bandwidth, but every extra replication focus costs server work.
- Humanoid NPCs are expensive to replicate. The common 2025–2026 pattern for large NPC counts is server-side data plus client-side rendering, sending compact quantized state about 10 times a second over (often unreliable) remotes.

### Cited Findings
- **Legacy vs NextGen replication** (Roblox engineer Khanovich, 2025-12-09):
  - "Properties would replicate via a pseudo 'First In First Out' principle, where an update to Color on a tree two miles away would be sent before an update to a Weld CFrame… right next to a player".
  - "Properties would be sent reliably and could block future updates under certain networking conditions".
  - "Physics updates on moving objects WOULD actually be prioritized and sent based on relevancy. Physics updates were unreliable by default and would never block future updates".
  - NextGen: "each player will receive the higher priority updates more frequently".
  - "There are no more ordering guarantees for property updates! This breaks the guarantee that when a RemoteEvent arrives that all properties that were updated BEFORE the event was sent would have replicated."
  - "NextGenerationReplication does a lot of extra CPU work on the Server… Streaming provi[de]s a necessary scalability limit".
  - "Server Performance with 20+ player games needs a lot of work!"

  — [DevForum: Server Authority Studio Beta – Insights from Engineers](https://devforum.roblox.com/t/4139321)
- Official: when `Workspace.NextGenerationReplication` is enabled, "you should not rely on the ordering of property replication and remote events" — [Workspace reference](https://create.roblox.com/docs/reference/engine/classes/Workspace)
- **Physics replication rate:**
  - The engineer (2026-07-09) said: "I think physics replication jumps from 15Hz to 60Hz" with NextGenerationReplication.
  - Older community data (2018) lists the NetworkSettings default `ClientPhysicsSendRate 20`.
  - A 2026 developer describes Roblox physics replication as "a constant 1/20 interval".

  **These conflict (15 Hz vs 20 Hz)**, and the 2018 figure is old — [Server Authority Issues](https://devforum.roblox.com/t/4728153); [DevForum t/168634 (2018)](https://devforum.roblox.com/t/168634); [DevForum t/4563460 (2026)](https://devforum.roblox.com/t/4563460)
- MicroProfiler labels relevant to moving parts:
  - `interpolateNetworkedAssemblies`: "Set the network owner of parts to the current player to reduce this".
  - "Reduce the amount of parts that frequently switch network ownership".
  - `Dispatch Physics Senders and TouchSenders`: "Reduce the amount of moving objects and/or touches".

  — [MicroProfiler tag table](https://create.roblox.com/docs/performance-optimization/microprofiler/tag-table)
- **StreamingEnabled (official):**
  - Benefits are "Faster Join Times", "Memory Efficiency", and that "the server can spend less time and bandwidth synchronizing changes".
  - It is enabled by default for new places.
  - `StreamingMinRadius` defaults to 64 ("increasing… will require more memory and more server bandwidth"); `StreamingTargetRadius` defaults to 1024.
  - `StreamOutBehavior.Opportunistic` is recommended to reduce memory.
  - "Avoid creating moving assemblies with unnecessarily large numbers of instances, as all of the instances streaming in unison may cause network/CPU spikes".
  - Extra foci are expensive: "a single player with nine dynamically moving foci could generate server networking and streaming processing comparable to ten players".
  - "Client-side physics simulation, including prediction and resimulations… only occurs in streamed areas".
  - Persistent models "are sent as a complete atomic unit soon after the player joins".
  - Predictive streaming is additive and is skipped on resource-constrained devices.

  — [Instance streaming](https://create.roblox.com/docs/workspace/streaming)
- Join-time tooling: "Print Join Size Breakdown" (Studio Settings > Network) prints "the top 20 instances by size and a percentage breakdown by instance type". With streaming, only the area near spawn replicates at join — [Identify performance issues: Load times](https://create.roblox.com/docs/performance-optimization/identify)
- An engine-side critique (community, Dec 2025): "Most people dont run amazing internet connections, especially on phones… A constant stream of 500 kb/s back and fourth in even simpler games all because of streaming" — this is opinion — [Insights from Engineers, post #11](https://devforum.roblox.com/t/4139321)
- **Client-side NPC pattern** (community guide, 2025-12-31, updated 2026-02-01):
  - "Pathfind + send position data for NPCs on server (rather than placing actual models and humanoids…)" and "Render position data for NPCs on client".
  - "A common rate is sending data every 1/10th of a second".
  - Send only NPCs that changed position, interpolate on the client, and use arrays rather than dictionaries.
  - Quantize: CFrame (about 20 B) becomes two `Vector3int16` (about 12 B); a yaw-only angle fits in an int16 via `buffer.writei16`.
  - Use `workspace:BulkMoveTo` to apply many CFrames cheaply.

  — [Downrest: custom NPC replication tips](https://devforum.roblox.com/t/4202236)
- Humanoid cost: "humanoids do a lot more than just change positions… such as updating humanoid states and replicating those states to clients" and "Consider not using humanoids if not needed as they are expensive to maintain and replicate (humanoid states, humanoid physics, etc.)" — [DevForum t/4209361](https://devforum.roblox.com/t/4209361); [DevForum t/4214247](https://devforum.roblox.com/t/4214247)
- A 2026 NPC system "sends data from the server to clients through unreliable remote events. You have to chunk the data because of the 1000 byte limit" — [DevForum t/4563460](https://devforum.roblox.com/t/4563460)
- A shipped-style example (2024): "30 pets per player, with advanced techniques to minimize their bandwidth and cpu usage" and "Up to 200 coins per player, with local client physics for the rendering" — [MrChickenRocket article (2024)](https://devforum.roblox.com/t/3127146)

### Inferences
- For hundreds of AI fish, leaving them as server-owned replicated parts or Humanoids routes every movement through physics/property replication at 15–60 Hz. That cost scales with fish count, and the engine controls the prioritization. Server-side data with client-rendered fish over one batched quantized stream gives the game direct control of rate, precision and interest radius.
- If the project adopts Server Authority (covered by another researcher), NextGenerationReplication and StreamingEnabled become mandatory. That raises the bandwidth of the replicated player sharks (about 4× per the engineer) and makes keeping fish **out** of instance replication even more important.
- Because NextGen breaks property/remote ordering, remote payloads should refer to fish by **numeric IDs**, not Instance references, so they never depend on an Instance having replicated first.
- I found no source quantifying the replication cost of Attributes or ValueObjects. By the official sender description they are property changes going through "Data Senders", which are reliable under legacy replication. Treat them as unsuitable for per-tick fish state.

### Gaps
- No official, current number exists for the physics replication frequency (legacy or NextGen), only an engineer's hedged "15Hz to 60Hz" and older community figures of 20 Hz.
- No source measured bytes per Attribute or ValueObject change, or per Instance creation/destruction.
- No official Roblox document specifically recommends client-rendered NPCs; the evidence is community practice and engineer comments about Humanoid and replication cost.

---

## 4. Techniques for poor mobile connections (150–300 ms RTT, jitter, loss, low bandwidth): concrete numbers

### Takeaway
The classic toolbox still applies and has well-documented numbers:
- **Quantize:** about 2 mm positions in 18 bits per axis for a ±256 m range; smallest-three quaternions in 29 bits; 40 B down to 10 B per object.
- **Delta-encode against the last acknowledged snapshot:** about 1 RTT old, with 1 bit for unchanged objects.
- **Size the interpolation buffer** to survive two consecutive losses: 10 Hz → 300–350 ms, 30 Hz → about 150 ms, 60 Hz → about 85 ms. Prefer Hermite over linear at low rates, and avoid extrapolation.
- **Send only what changed, near the player, at about 10 Hz** for NPCs.
- **Sequence-number** unreliable streams.
- **Keep reliable traffic small**, because reliable messages retransmit and block.

On Roblox specifically, unreliable events must be chunked to fit under 1000 B. Zap (`OrderedUnreliable`) and BlinkBlox (outbound budget, `Priority: Low`) provide some of this in the library.

### Cited Findings
- **Quantization and delta compression** (Glenn Fiedler, *Snapshot Compression*, 2015-01-04, **older but foundational**):
  - The test scene is 901 cubes at 60 Hz, starting at about 17.37 Mbps with a 256 kbps target.
  - Positions are bounded to x, y ∈ [-256, 255] m and z ∈ [0, 32] m at "512 values per meter" (about 2 mm), using 18 + 18 + 14 = 50 bits instead of 96.
  - Orientation uses "smallest three" with 9 bits per component, 29 bits total instead of 128. Per-cube state drops to 80 bits (10 bytes).
  - Velocity was dropped because interpolation at 60 Hz sufficed.
  - The delta baseline is "the most recent snapshot the receiver has acknowledged", which in steady state is about one RTT old. Unchanged cubes cost one bit.
  - A stationary scene settles at "about 15 kbps", mostly packet header overhead.

  — [Gaffer On Games: Snapshot Compression](https://gafferongames.com/post/snapshot_compression/)
- **Interpolation buffer sizing** (Fiedler, *Snapshot Interpolation*, 2014-11-30, **older but foundational**):
  - "Buffer enough snapshots to survive two consecutive lost packets". At 2–5% loss, the best delay found was 3× the send interval.
  - 10 pps → 300 ms plus 1–2 frames for jitter (the videos used 350 ms). 30 pps → about 150 ms. 60 pps → about 85 ms.
  - Hermite interpolation (which also sends linear velocity) removes the artifacts linear interpolation shows at 10 pps. Slerp is fine for orientation.
  - Extrapolating 200 ms cut delay to 150 ms but caused visible errors (objects passing through the floor).
  - A lost snapshot should be skipped, not waited on.

  — [Gaffer On Games: Snapshot Interpolation](https://gafferongames.com/post/snapshot_interpolation/)
- **Sequence numbers on unreliable streams** (Roblox docs): unreliable messages "can arrive out of order… include a timestamp or sequence number… ignoring messages older than the latest handled one" — [Remote events and callbacks](https://create.roblox.com/docs/scripting/events/remote)
- Zap has an `OrderedUnreliable` type that adds a u16 order id and drops stale packets. It is undocumented and exists only in the grammar and codegen — [Zap grammar](https://github.com/red-blox/zap/blob/main/zap/src/parser/grammar.lalrpop); [Zap client codegen](https://github.com/red-blox/zap/blob/main/zap/src/output/luau/client.rs)
- **Rate limiting and send only what changed:** "A common rate is sending data every 1/10th of a second" for NPCs, and "only send out position data… of NPCs that have changed positions" — [Downrest NPC tips](https://devforum.roblox.com/t/4202236). For client intent, send "at a fixed rate (e.g., 10 Hz), not every frame" — [DevForum t/3962093](https://devforum.roblox.com/t/3962093)
- **Measured custom result:** one developer reduced "about 10k Vector3s and 10k floats every 100ms to about 45 kb/s, with slight precision loss" using a custom networker — [DevForum: What networker should I use?](https://devforum.roblox.com/t/what-networker-should-i-use/4889485)
- **Quantized types available in libraries:**
  - Packet: Float16 (2 B), Float24 (3 B), Vector3S16, CFrame in 12–18 B.
  - BlinkBlox: `f32<0.01>(-1..1)` in 1 byte.
  - Warp 1.1: CFrame with f16-compressed rotation.
  - Blink 0.18 has none of these; its quantized numerals (#104) are a planned 1.0 enhancement.

  — [Packet thread](https://devforum.roblox.com/t/packet-networking-library/3573907); [BlinkBlox thread](https://devforum.roblox.com/t/4898948); [Warp buffer docs](https://github.com/imezx/Warp/blob/main/docs/api/1.1/buffer.md); [Blink issues](https://github.com/1Axen/blink/issues)
- **Adaptive send rate and priority** (engine precedent): the legacy Physics Sender, when it hit about 50 KB/s, would "reduce the frequency of updates on parts to fit". NextGen sends "higher priority updates more frequently" — [Server Authority Issues](https://devforum.roblox.com/t/4728153); [Insights from Engineers](https://devforum.roblox.com/t/4139321). Library precedent: BlinkBlox's "per-player outbound budget: events marked `Priority: Low` are the first to be left out when a player's connection is full" — [BlinkBlox thread](https://devforum.roblox.com/t/4898948)
- **Keep reliable traffic small:** legacy reliable properties "could block future updates under certain networking conditions" — [Insights from Engineers](https://devforum.roblox.com/t/4139321). Data ping includes "TCP-like retransmissions of lost or corrupted packets" — [Identify performance issues](https://create.roblox.com/docs/performance-optimization/identify)
- **Interest management, engine side:** streaming radius defaults (64 min, 1024 target) bound what instances reach a client, and extra foci are costly. Remote-based state needs its own radius logic — [Instance streaming](https://create.roblox.com/docs/workspace/streaming)
- Network Simulator test prompts from Roblox: "Can unreliable updates be skipped without leaving stale state on screen?" and "Does replicated movement remain understandable when updates arrive unevenly?" — [Network Simulator docs](https://create.roblox.com/docs/studio/network-simulator)

### Inferences
- **Worked budget for the shark game (my arithmetic, not sourced):** a fish update could be:
  - u16 id: 2 B.
  - Position as 3 × u16 quantized over a 2048 m map: about 3 cm resolution, 6 B.
  - Yaw and pitch as 2 × u8: 2 B.
  - Total: about 10 B.

  At 10 Hz for 100 fish inside a player's interest radius, that is about 10 KB/s raw per player before engine compression and per-call overhead. 200 fish at 10 Hz is about 20 KB/s. Both fit under a self-imposed 32–50 KB/s budget. One unreliable packet of ≤ about 950 B carries about 90 such fish.
- **Tiered rates for interest management:** for example 10–15 Hz for fish within about 100 studs of the shark, 2–5 Hz farther out, and nothing beyond the view and streaming radius. Combined with sending only changed or moving fish, this matches the community practice above and approximates Gaffer's "unchanged = 1 bit" without acknowledgements.
- **Interpolation delay:** at 10 Hz, apply Gaffer's 3×-interval rule plus jitter margin. That gives about 300–350 ms of render delay for NPC fish. On the 3G preset (70–90 ms jitter, see Q5), an adaptive buffer that tracks measured inter-arrival jitter is safer than a fixed value. Prey fish are not hit-critical, so delay is acceptable. Hermite is worth it if fish turn sharply at low rates.
- **Acked delta snapshots** (SMASH-style) pay off mainly when most state is static between ticks. Constantly swimming fish change every tick, so deltas save little. Quantization, interest management and lower rates are simpler and capture most of the gain.
- **Keep discrete gameplay events reliable but tiny:** eat/bite results, spawn/despawn, growth/evolution. Use IDs, not Instance references.

### Gaps
- Valve's Source networking reference (default rates, `cl_interp` 100 ms) returned HTTP 403 and was not used.
- No Roblox-specific measurements of loss, jitter or throughput for mobile players (by region or carrier) were found.
- No source quantified the CPU cost on low-end phones of decoding and interpolating hundreds of entities per tick.

---

## 5. How to test bad networks in 2026: Studio tools, Developer Console, MicroProfiler, multi-client, devices

### Takeaway
- **Network Simulator:**
  - Location: the **Network** pill on the new Device Simulator toolbar (Test → Device Simulator).
  - Release: beta 9 Sep 2026, **full release 16 Oct 2026**.
  - Controls: inbound and outbound **latency, jitter and packet loss** set separately, with presets down to "Bad Connection (3G)".
  - Limits: **packet loss is capped at 0.5%**, and it has **no bandwidth-cap control**.
  - Scripting: it is backed by `NetworkSettings.Inbound/Outbound…` properties. The older `IncomingReplicationLag` (seconds) still exists.
- **Measurement tools:** MicroProfiler network capture (desktop/Studio dumps only), data ping (Server Stats tab, Shift+F3), debug stats overlays, Server & Clients tests with up to 8 clients, and real-device cellular tests.

### Cited Findings
- **Network Simulator release:**
  - Announced 2026-09-09 as a Studio beta.
  - **Update of 2026-10-16:** "The new Device Simulator is now out of beta and available for everyone. You no longer need to enable it in Studio's Beta Features window."
  - It added a "Current Device preset" that turns simulation off, and "Pause on close".
  - Simulated latency, jitter and loss "also apply while your game loads"; to avoid slow loads, "start Play Solo first, then turn on Network Simulation".

  — [DevForum: [Full Release] New Device Simulator Toolbar and Network Simulator](https://devforum.roblox.com/t/4861695)
- The docs (repo state of 29 Oct 2026) **still say beta**, enabled through File > Beta Features > "New Device Simulator". They are behind the 16 Oct announcement — [Network Simulator docs](https://create.roblox.com/docs/studio/network-simulator)
- **Opening it:** "From the Test menu, enable Device Simulator… select the Network pill" — [Network Simulator docs](https://create.roblox.com/docs/studio/network-simulator)
- **Presets** (inbound | outbound, as latency / jitter / loss):

  | Preset | Inbound | Outbound |
  |---|---|---|
  | Ideal Fiber (default) | 8 ms / 0 ms / 0.00% | 8 ms / 0 ms / 0.00% |
  | Wired Broadband | 25 / 3 / 0.00% | 25 / 3 / 0.00% |
  | Home Wi-Fi | 30 / 12 / 0.20% | 30 / 15 / 0.30% |
  | Standard Mobile (4G/LTE) | 45 / 20 / 0.40% | 55 / 30 / 0.50% |
  | Bad Connection (3G) | 150 / 70 / 0.50% | 180 / 90 / 0.50% |

  Ideal Fiber "isn't the same as disabling simulation"; to get zero added delay, set everything to 0 — [Network Simulator docs](https://create.roblox.com/docs/studio/network-simulator)
- **Ranges and caps:** latency and jitter 0–1000 ms; packet loss "0.00–0.50%" ("`0.50%` means half of one percent"). The NetworkSettings reference: "To maintain a stable test environment, the value is capped at 0.5%". "Per-packet delay sampled from delay and jitter is rounded to 1 millisecond" — [Network Simulator docs](https://create.roblox.com/docs/studio/network-simulator); [NetworkSettings reference](https://create.roblox.com/docs/reference/engine/classes/NetworkSettings)
- **Scriptable properties** (PluginSecurity): `NetworkSettings.InboundNetworkMinDelayMs`, `OutboundNetworkMinDelayMs`, `InboundNetworkJitterMs`, `OutboundNetworkJitterMs`, `InboundNetworkLossPercent`, `OutboundNetworkLossPercent`. These are "the same underlying emulation settings as the Network tab of Studio Settings" — [Network Simulator docs](https://create.roblox.com/docs/studio/network-simulator)
- `NetworkSettings.IncomingReplicationLag` "Simulates additional network latency in the network receiving path… delaying all incoming messages. Units are seconds". It is still listed and not deprecated — [NetworkSettings reference](https://create.roblox.com/docs/reference/engine/classes/NetworkSettings)
- Roblox's own guidance: a "Bad Connection (3G)" run is a resilience test. Change only inbound or only outbound to isolate a direction. "Large latency changes during a running test can affect congestion control, so a fresh run provides a cleaner comparison". Simulation is "especially useful… for testing UnreliableRemoteEvents, which might be lost or arrive out of order" — [Network Simulator docs](https://create.roblox.com/docs/studio/network-simulator); [Studio testing modes](https://create.roblox.com/docs/studio/testing-modes)
- **MicroProfiler network capture:**
  - Open with Ctrl+Alt+F6 and choose the Network menu level High, Low or Off. "Low… is the default level and the one the MicroProfiler uses when you profile a mobile device".
  - Network data is available "only in frame dumps from the desktop client or Studio, not in the real-time overlay".
  - In the dump, X-Ray / Network shows received (top) and sent (bottom) traffic per frame, colored physics (blue), data (green) and assets (red).
  - The "Network events" window and extra graphs cover Data, Physics, "Large replicator data", Terrain and Asset fetching. Log Scale Graph is available.

  — [MicroProfiler: Network usage](https://create.roblox.com/docs/performance-optimization/microprofiler/network)
- **Ping:**
  - "Network ping" is visible in the Performance Stats bar.
  - "Data ping" is the reliable replication round trip, including queueing and retransmission. It is shown in the Developer Console **Server Stats** tab (server average), and per client with **Shift+F3** ("network debug stats").
  - Shift+F5 shows the debug stats summary.

  — [Identify performance issues](https://create.roblox.com/docs/performance-optimization/identify)
- The Developer Console **Network** tab only covers web calls (HttpService and DataStore requests), not remote traffic — [Developer Console](https://create.roblox.com/docs/studio/developer-console)
- **Multi-client:** "Server & Clients… Usually 1–2 clients is sufficient, although you can simulate up to eight" — [Studio testing modes](https://create.roblox.com/docs/studio/testing-modes)
- **Real devices:** "network behavior over cellular connections can only be validated on real hardware". Use simulation for iteration, "but confirm behavior on a real device over cellular or constrained Wi-Fi". Run 10–15 minute sessions to catch thermal throttling — [Test on hardware](https://create.roblox.com/docs/performance-optimization/test-on-hardware)
- **Bandwidth caps:** Network Simulator exposes no throughput limit. `NetworkPeer:SetOutgoingKBPSLimit` (PluginSecurity, 10–10,000 kbps) is the documented engine API for capping outgoing bandwidth — [NetworkPeer reference](https://create.roblox.com/docs/reference/engine/classes/NetworkPeer)
- **Community tooling:** the "Remote Inspector" plugin (2026-08-02) monitors "traffic going through remotes and signals". Pyseph's Remote Packet Size Counter estimates remote payload bytes (2023) — [Remote Inspector](https://devforum.roblox.com/t/4770963); [Remote Packet Size Counter](https://devforum.roblox.com/t/2320709)
- Roblox says it is "developing a dedicated Network Simulator skill for Studio Assistant". This is a stated plan, not shipped — [Network Simulator announcement](https://devforum.roblox.com/t/4861695)

### Inferences
- **Suggested repeatable matrix for this game:**
  1. Ideal Fiber baseline.
  2. Standard Mobile (4G/LTE).
  3. Bad Connection (3G), which adds about 330 ms RTT plus 70–90 ms jitter, roughly the 150–300 ms target.
  4. A saved custom "worst mobile" preset, for example 150/80/0.5% inbound and 150/80/0.5% outbound.

  Run each with Server & Clients (2–3 clients) plus a server-side `SetOutgoingKBPSLimit`, for example about 256 kbps (32 KB/s), to emulate a thin downlink. The `SetOutgoingKBPSLimit` use is my suggestion and was not verified in a playtest.
- Since loss is capped at 0.5% in Studio, testing realistic mobile loss (1–5%) needs an OS- or router-level network shaper or real cellular devices. Sequence-number and stale-drop logic can also be unit-tested by dropping packets in code.
- Log `Stats.DataReceiveKbps` per client, and compare data ping (Shift+F3) with network ping. A growing gap is the documented sign of replication queueing.

### Gaps
- No official confirmation that `SetOutgoingKBPSLimit` works as a bandwidth-throttling test tool in Studio playtests. Its docs describe only the API.
- No Roblox-published tool for >0.5% loss or for bandwidth caps in Network Simulator. I did not research or source external tools such as OS-level traffic shapers.
- The Studio docs lag the 16 Oct 2026 full release; the exact Studio version that shipped it was not found.

---

## 6. Recommendation for the new shark game: keep Blink or switch? What to keep from SMASH?

### Takeaway
**Keep Blink, but upgrade to 0.18.9 now.** It fixes the NaN range-validation bypass that affects SMASH's 0.18.8. Use Blink only for **typed, validated, low-rate gameplay events**. Do **not** rebuild SMASH's custom generator, acknowledged-delta machinery or 60 Hz snapshot pipeline. Replace them with:
- one hand-written, quantized, interest-filtered fish/shark state stream at about 10–20 Hz, with sequence numbers, sent as a Blink `buffer` payload or a single unreliable event of ≤ about 950 B;
- a small server-side guard (size cap and per-player rate limit before decode), because Blink 0.18 does not protect against oversized hostile buffers (issue #45).

Switching to Zap, ByteNet/ByteNet Max, Packet, Warp, Jolt or QuickNet would buy little or nothing for this workload. BlinkBlox is the one alternative worth a timed evaluation, since it is schema-compatible and adds caps and budgets, but it is about one month old. Re-evaluate Blink 1.0 when it leaves pre-release.

### Cited Findings
- **Why stay on Blink:**
  - It is the actively released IDL compiler (0.18.9 and 1.0-pre.10, both on 2026-09-19).
  - It already validates client data on receive.
  - It is the community's 2026 default for performance-heavy games.
  - Its bandwidth is on par with Zap and ByteNet, with higher FPS in its own benchmark.

  — [Blink releases](https://github.com/1Axen/blink/releases); [Blink README](https://github.com/1Axen/blink); [What networker should I use?](https://devforum.roblox.com/t/what-networker-should-i-use/4889485); [Blink Benchmarks.md](https://github.com/1Axen/blink/blob/main/benchmark/Benchmarks.md)
- **Why upgrade to 0.18.9:** "Fix NaN scalars and vectors bypassing inexact range validation" — [Blink releases](https://github.com/1Axen/blink/releases)
- **Why add a guard:** Blink #45 (oversized client buffer DoS) is still open, and the maintainer considers it "out of blink's security scope" until the rewrite — [Blink issue #45](https://github.com/1Axen/blink/issues/45)
- **Why not Zap:**
  - Same architecture.
  - Releases are slower: 0.6.29 in June 2026 after 0.6.28 in December 2025.
  - It has an open oversized-buffer server-crash issue (#219) and a Player memory leak (#216).

  — [Zap releases](https://github.com/red-blox/zap/releases); [Zap #219](https://github.com/red-blox/zap/issues/219); [Zap #216](https://github.com/red-blox/zap/issues/216)
- **Why not ByteNet:** no commits since 2025-08-01, no GitHub release since v0.4.3, and an open "Ability to crash servers" issue. ByteNet Max has been pulled from the Creator Store as of 2026-10-04 — [ByteNet](https://github.com/ffrostfall/ByteNet); [ByteNet #34](https://github.com/ffrostfall/ByteNet/issues/34); [ByteNet Max thread](https://devforum.roblox.com/t/3268469)
- **Why not Packet:**
  - It has a runtime schema.
  - Unreliable traffic goes through a manual wrapper outside its batching; a 2026 user calls this "counterintuitive".
  - It had launch-time vulnerability reports.
  - It decodes slower in third-party benchmarks.

  — [Packet thread](https://devforum.roblox.com/t/packet-networking-library/3573907); [BlinkBlox thread](https://devforum.roblox.com/t/4898948)
- **Why not Warp, Jolt or QuickNet now:**
  - Warp 1.1 is still "pre" (pre8 on 2026-10-09) and users report its docs are hard to follow.
  - Jolt changed its wire format in v4.0.0 (2026-08-29).
  - QuickNet is v0.4.0.
  - All three are runtime libraries.

  — [Warp thread](https://devforum.roblox.com/t/2779813); [Jolt thread](https://devforum.roblox.com/t/4095212); [QuickNet thread](https://devforum.roblox.com/t/4624342)
- **What BlinkBlox would add:** pre-parse size/count caps, token-bucket rate limits, quantized floats, a per-player outbound budget with `Priority: Low` shedding, and compile-time unreliable size checks, all while keeping `.blink` schemas. Its benchmarks are self-reported, and it was announced 2026-09-27 — [BlinkBlox thread](https://devforum.roblox.com/t/4898948)
- **Unreliable cap is now 1000 B** (since 2025-03-12). SMASH's 880 B packets predate the change but remain a safe size — [Introducing UnreliableRemoteEvents](https://devforum.roblox.com/t/introducing-unreliableremoteevents/2724155)
- **"50 KB/s" is a legacy physics-sender number**, so a self-imposed budget is a design choice, not an engine limit — [Server Authority Issues](https://devforum.roblox.com/t/4728153)

### Inferences
- **Keep from SMASH:**
  - The Blink schema discipline: typed events, ranges on every client-sent number, `Polling` for fixed-step consumption.
  - Per-client byte budgets, used as a monitored target. About 32 KiB/s still fits the mobile 50 KB/s community guideline even though the engine cap is higher.
  - Unreliable packets of about 880–950 B, with sequence numbers and drop-stale logic.
  - Buffer-encoded entity state, but as **one** small hand-written codec for fish and sharks: about 10 B per fish (see Q4), not a generator emitting many specialized codecs.
- **Replace or drop from SMASH:**
  - The custom code generator, which duplicates what Blink 1.0 plans to add (quantized numerals, bitpack, size checks).
  - Acknowledged delta snapshots, which have low payoff for constantly moving fish and high complexity.
  - 60 Hz snapshot rates. Use 10–20 Hz near the shark and 2–5 Hz far away, and let interpolation (about 150–350 ms) hide it.
  - Per-player re-encoding where avoidable. Encode each spatial cell's fish state once per tick and send the same buffer to every player whose interest set includes that cell. Blink 0.18's `FireAll` path issues one call per player and the server frame budget was SMASH's failure point, so encode-once matters.
- **Reliable traffic** should carry only discrete state changes such as eat, bite, spawn/despawn, growth/evolution and score, keyed by numeric IDs. With NextGenerationReplication, remote/property ordering is no longer guaranteed.
- **Guard layer for client input:**
  - Reject buffers above an expected maximum before the Blink decoder runs, by patching or wrapping the generated listener (it sits on the `BLINK_RELIABLE_REMOTE` / `BLINK_UNRELIABLE_REMOTE` remotes) or by adopting BlinkBlox.
  - Rate-limit per player.
  - Treat decode errors as possible exploitation.
- **Re-evaluation trigger:** switch to Blink 1.0 once it is stable and ships #104/#105/#108/#102 plus the #45 mitigation. Switch to BlinkBlox only if a two-week spike shows it is stable with this game's schema and its maintainer is still shipping.

### Gaps
- Mobile-device (low/mid-end phone) CPU measurements for Blink-generated decode versus a hand-written fish codec were not found and should be measured in this project.
- Blink 1.0's syntax changes affect how hard a migration from 0.18 would be; no migration guide was found.
- BlinkBlox's longer-term maintenance and independent security review are unknown.
