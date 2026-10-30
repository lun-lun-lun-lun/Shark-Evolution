# Simulating and replicating hundreds of AI fish/NPCs on Roblox cheaply, and the state of Roblox ECS libraries (as of Oct 30, 2026)

*How the version data was gathered: library versions and dates come from the GitHub REST API and the npm registry and Wally index, queried on 2026-10-30. Sources from before 2025 are marked "(older)". If a claim came only from a search-result summary of a thread I did not read in full, it is marked "(snippet)".*

## 1. Architectures for hundreds of NPCs with low bandwidth, and how to split work between server and client

### Takeaway
Roblox developers have converged on one pattern between 2021 and 2026. The server keeps NPCs as plain data with no Instances or Humanoids. Clients own every model, pool them, and interpolate between updates. Updates go out batched, quantized and change-only at about 5–20 Hz. Measured results include 100 RTS units dropping from 35 to 10 KB/s, and about 17.7 KB/s for 1,000 NPCs using 29-bit positions. For comparison, about 110 KB/s for 200 Humanoids moved with `MoveTo`.

For a fish game, the cheapest option is closed-form, time-parameterized motion for calm schools. That means sending seeds, parameters and events instead of positions. Per-fish snapshots are reserved for the few fish that are reacting to a shark. Stepped "deterministic flocking" that must match exactly on every machine is fragile and not recommended.

### Cited Findings
**What engine (Instance/Humanoid) replication costs**
- Bandwidth for Humanoid `:MoveTo()`, measured by the author:
  - 5 NPCs: about 4 KB/s
  - 25 NPCs: about 17 KB/s
  - 75 NPCs: about 40 KB/s
  - 200 NPCs: about 110 KB/s
  - 500 NPCs: about 160 KB/s

  Roblox's replication throttle is 50 KB/s. Above it, Humanoids become "slow, laggy, or teleporting". Tutorial by Downrest, posted Dec 31, 2025 and updated Feb 1, 2026. — [DevForum: Performance tips for custom NPC replication](https://devforum.roblox.com/t/performance-tips-network-bandwidth-client-framerate-etc-for-custom-npc-replication/4202236)
- Moving 2,000 MeshParts on the server cost about 2,500 KB/s received by clients. Sending one signal and letting clients move the parts cost almost nothing. The same author recommends keeping Total Data around 80 KB/s or less (WoloPoints, May 10, 2026). — [DevForum: The definitive tutorial to game optimization](https://devforum.roblox.com/t/the-definitive-tutorial-to-game-optimization-everything-you-need-to-know/4629258)
- Astro Force V1 (older, May 2021) moved server parts with CFrame about 60 times per second, which used "a huge amount of bandwidth". The authors' reference points: 40–50 KB/s is "exceptional", and Phantom Forces or Arsenal typically use 50–80 KB/s. — [DevForum: How we reduced bandwidth usage by 60x in Astro Force](https://devforum.roblox.com/t/how-we-reduced-bandwidth-usage-by-60x-in-astro-force-roblox-rts/1202300)
- A tower defense prototype from Nov 2025 updated 1,000 anchored enemy roots with CFrame at 30 Hz on the server. The fix suggested in the thread was "don't actually move the enemy instances on the server". Instead, send waypoints or discretized positions and let each client interpolate. — [DevForum: How can I heavily reduce the bandwidth of this Enemy System](https://devforum.roblox.com/t/how-can-i-heavily-reduce-the-bandwidth-of-this-enemy-system-foundation-code/4052181)

**Data-only server with client rendering (snapshot replication): real examples and numbers**
- Astro Force V2 (older, 2021):
  - Replication dropped to 10 Hz, with linear interpolation on the client.
  - Y is not sent. The client computes it from a heightmap, which the authors say is about 10x faster than raycasting.
  - Only yaw is sent, and only units flagged `PositionChanged` are included.
  - Each unit uses two `Vector2int16` values. Position is multiplied by 5 (about 0.2-stud precision), and yaw by 100 (about 0.01 rad), giving a range of about ±6,553 studs.
  - Result: 100 moving units went from 35 KB/s to 10 KB/s.

  — [Astro Force 60x thread](https://devforum.roblox.com/t/how-we-reduced-bandwidth-usage-by-60x-in-astro-force-roblox-rts/1202300)
- Astro Force V3 (older, 2021):
  - Each unit update is packed into about 32 bits: an 11-bit id (up to 2,048 units), 7-bit yaw (about 2.8°), and X/Z as 6-bit offsets inside 8×8-stud grid cells (0.125-stud precision).
  - When a unit changes cell, that is sent separately as a `Vector3int16` with an 18-bit cell id (262,144 cells, about 4,096×4,096 studs).
  - The authors report "another 50%" saving and about 60x total versus V1.
  - The listed bit fields add up to 30 bits, not 32. This inconsistency is in the post itself.

  — [Astro Force 60x thread](https://devforum.roblox.com/t/how-we-reduced-bandwidth-usage-by-60x-in-astro-force-roblox-rts/1202300)
- Conquest Napoleonic Wars (WoloPoints, Dec 1, 2024; replies run to Aug 2026):
  - "every unit must be 'fake'": the server holds only positions and health, with no parts.
  - Arrays are used instead of dictionaries.
  - Parallel Luau is used for unit math, and a "swarm module" grid limits target checks to nearby cells.
  - Replication goes out about every 10 frames. Distant units snap to position instead of interpolating.
  - The game runs "800+ units with no problems" without Humanoids. Other developers in the thread report 500+ and 1000+.

  — [DevForum: How we are developing an RTS game with good performance for 200+ units](https://devforum.roblox.com/t/how-we-are-developing-an-rts-game-with-good-performance-for-more-than-200-units/3281168)
- Zombie stress test by vantoanvh (May 22, 2025, updated to Jan 2026):
  - 10,000 NPCs on an iPhone 8 Plus ran at 55–60 FPS, receiving 100–120 KB/s, with no throttling, LOD or Parallel Luau.
  - A later version packs each position into 29 bits (about 3.6 bytes) in a buffer. It sends about every 0.2 s and lerps at 40 Hz on the client, for an estimated 17.7 KB/s per 1,000 NPCs.
  - A later Humanoid-based variant ran at about 20 Hz with 0.2-stud precision, about 15 KB/s for 500 NPCs.
  - In Jan 2026 the author switched to delta coding and rANS entropy coding (replacing ZSTD), and to chunked updates (about 15 Hz effective).
  - Critic Blockunstar says chunked updates cause uneven framerates.

  — [DevForum: 10,000+ Physical NPCs at ~60 fps on low-end device](https://devforum.roblox.com/t/10000-physical-npcs-running-with-60-fps-on-low-end-device/3660293)
- Replies in the Downrest tutorial thread (Jan 2026):
  - 500 NPCs at 20 Hz cost about 20+ KB/s. Delta-of-delta coding brings that to about 12 KB/s.
  - 500 NPCs × 3-bit animation state is about 188 bytes, or about 120 bytes with rANS.
  - Suggested quantization: 0.2 stud and 10°.
  - Downrest's own system peaks at 120 KB/s with 2,000 positions.
  - Another user claims 10,000+ positions under 50 KB/s (unverified).

  — [Downrest thread](https://devforum.roblox.com/t/performance-tips-network-bandwidth-client-framerate-etc-for-custom-npc-replication/4202236)
- Approximate wire sizes:
  - CFrame: about 20 B
  - Vector3: about 12 B
  - Vector3int16: about 6 B
  - Number: about 8 B
  - Each RemoteEvent fire adds about 9 B of overhead.
  - A buffer adds about 9 B of header (8-byte length plus 1 type byte). A reply therefore argues for bit-packing many values into one buffer rather than many small buffers.

  — [Downrest thread](https://devforum.roblox.com/t/performance-tips-network-bandwidth-client-framerate-etc-for-custom-npc-replication/4202236)
- `UnreliableRemoteEvent` limits: "Events with payloads larger than 1000 bytes are dropped." Buffers are compressed before sending, so the final size is hard to predict. Events are not ordered and may be skipped. — [Roblox docs: UnreliableRemoteEvent](https://create.roblox.com/docs/reference/engine/classes/UnreliableRemoteEvent)
- An older thread's title reports 1,500 NPCs at "about 3–5% script activity", but with 300–500+ ping and network "Rate" over 20–30k. — [DevForum: Optimization tips on NPCs (1500 NPCs)](https://devforum.roblox.com/t/optimization-tips-on-npcs-managed-to-get-1500-npcs-300-500-ping-about-3-5-script-activity-but-rate-is-over-20-30k/2511448)
- A search snippet says one author ran 1,000+ unanimated NPCs at about 10–20 KB/s. The snippet did not make clear which of two 1,500-NPC threads this came from, so treat it as weak evidence. — [DevForum 1500 NPCs thread A (older)](https://devforum.roblox.com/t/any-optimalization-tips-managed-about-1500-npcs-mostly-without-lag/284180); [DevForum 1500 NPCs thread B (older)](https://devforum.roblox.com/t/optimization-tips-on-npcs-managed-to-get-1500-npcs-300-500-ping-about-3-5-script-activity-but-rate-is-over-20-30k/2511448)
- Gaffer On Games, snapshot compression (older, Jan 4, 2015):
  - Setup: 901 cubes at 60 snapshots/s, about 17.38 Mbps uncompressed, with a 256 kbps target.
  - Position is quantized to 512 values per meter.
  - Rotation uses "smallest three" encoding at 29 bits, making each cube 80 bits.
  - Snapshots are delta-encoded against a baseline the client has acknowledged, with an index sent only for changed cubes (about 5.5 bits each).
  - A scene at rest costs about 15 kbps.

  — [Gaffer On Games: Snapshot Compression](https://gafferongames.com/post/snapshot_compression/)

**Deterministic and time-parameterized approaches**
- Tower defense pattern: the server sends each enemy's spawn time. The client compares it with `workspace:GetServerTimeNow()` and multiplies the difference by speed to place the enemy on the path. — [DevForum thread 2413006](https://devforum.roblox.com/t/2413006) (snippet). One caveat: if enemies move only on clients, the server does not know where they are, which breaks tower targeting. — [DevForum: How to handle enemy positions](https://devforum.roblox.com/t/2643951) (snippet). The common alternative is for the server to simulate and send positions periodically; one developer reports about 500 enemies that way. — [DevForum: Tower Defense Enemy Movement](https://devforum.roblox.com/t/tower-defense-enemy-movement/1930853) (snippet)
- Deterministic lockstep sends only inputs, so bandwidth scales with inputs rather than object count. However, guaranteeing floating-point determinism across compilers, operating systems and instruction sets is hard (older, 2014–2015). — [Gaffer On Games: Deterministic Lockstep](https://gafferongames.com/post/deterministic_lockstep/)
- Roblox's `Random` docs do not name the generator's algorithm. They also do not promise that a given seed produces the same sequence across platforms or engine versions. The only consistency guarantee is that `Shuffle` makes the same number of `NextInteger` calls across engine versions. — [Roblox docs: Random](https://create.roblox.com/docs/reference/engine/datatypes/Random)
- Even when the client also simulates and extrapolates, about a quarter-second of packet loss can still cause divergence and visible "pops". Gaffer's fix is to hard-snap the simulation and smooth only visually, using error offsets that decay by about 0.85–0.95 per frame. — [Gaffer On Games: State Synchronization](https://gafferongames.com/post/state_synchronization/) (older, Jan 5, 2015)

**Precedents outside Roblox**
- Fortnite Battle Royale: "100 connected players and about 50,000 replicated Actors". Checking every Actor against every client "will bottleneck the server's CPU", which is why Epic built the Replication Graph. — [Unreal docs: Replication Graph](https://dev.epicgames.com/documentation/en-us/unreal-engine/replication-graph-in-unreal-engine)
- ABZÛ (older, 2015–2016 press, snippet): the game went from about 100 fish on screen to about 10,000 once an engineer replaced skeletal animation with math-formula animation. It simulates "tens of thousands of fish" that school and scatter when they see a predator. — [Kill Screen](https://killscreen.com/abzu-2); [PlayStation Blog](https://blog.playstation.com/?p=156708)

### Inferences
- **Recommended split for the shark game: three tiers.**
  - **Tier A, calm schools (most fish).** The server stores one record per school: id, species, seed, spawn time, path parameters, state, and an alive bitmask. Each fish's position is a closed-form function: anchor(t) + offset(seed, fishIndex, t). The server evaluates it only when needed, for bite validation, predator targeting and relevancy checks. The server sends the school record when the school becomes relevant to a player, then only events (scatter, regroup, fish eaten). Server cost scales with the number of schools, not fish. Bandwidth is roughly 20–40 bytes per school once, plus events.
  - **Tier B, reacting fish (fleeing, chased, larger prey).** The server simulates these individually at about 10 Hz. They are replicated as quantized delta snapshots, as in Astro Force and vantoanvh.
  - **Tier C, AI predators (about 5–20).** These get a full state machine and per-entity snapshots at 10–20 Hz with client interpolation, the same way as players.
- **Why closed-form time functions beat stepped "deterministic flocking".** A closed-form position is evaluated fresh from t, so floating-point differences between x86 servers and ARM phones produce only tiny errors that do not accumulate. Stepped boids are chaotic, so any difference grows over time. This follows Gaffer's warnings about determinism and divergence.
- **Hash-based randomness.** Because Roblox does not guarantee `Random` sequences across platforms, derive per-fish randomness from an integer hash of (seed, fishIndex) using `bit32` operations rather than `Random.new` streams.
- **Quantization layout for 3D swimming.** Underwater there is no heightmap to derive Y, so Y must be sent. A suggested layout, following Astro Force's cell-relative trick:
  - Send each school's anchor at full precision.
  - Send each fish as an 8-bit offset per axis (±32 studs at 0.25-stud steps).
  - Add about 6 bits of yaw and 4 bits of pitch.

  That is about 34 bits (about 4.3 B) per fish per update. Batch all fish for one client into a single buffer per tick, kept under the 1,000-byte unreliable limit.
- **Example budget per client** (assumed numbers, not measured): 60 nearby Tier B/C fish at 10 Hz × 6 B = 3.6 KB/s; 40 mid-range at 5 Hz × 6 B = 1.2 KB/s; Tier A events under 1 KB/s. That totals about 5–6 KB/s, roughly a tenth of the 50 KB/s throttle cited above. Server egress for 15 players would be about 90 KB/s.

### Gaps
- I found no public technical write-ups from Fisch, SharkBite or other Roblox ocean games. Searches returned only exploit-script pages.
- I found no official statement on whether Luau `math.sin`, `math.noise` or vector math give bit-identical results on x86 servers and ARM mobile clients.
- Most Roblox numbers above are self-reported DevForum benchmarks with different hardware and methods.

## 2. Interest management, relevancy, and keeping server CPU low

### Takeaway
Use a spatial grid to decide what each player can see, then fill each player's packets by priority, with update rate falling off with distance. Add hysteresis at the edge and send full state when a fish or school enters view. Off-the-shelf ECS replicators only partly handle filter changes, so a fish-specific channel is simpler.

On CPU, the evidence favours preallocated struct-of-arrays data and `--!native` on numeric loops, plus work time-sliced against a per-frame budget. Parallel Luau gives limited gains on servers. Community reports from Nov 2024–Apr 2025 say live servers ran only about 2 parallel worker threads. No official figure has been published since Roblox moved to dynamic CPU allocation in Nov 2025.

### Cited Findings
**Relevancy and priority patterns**
- Unreal's Replication Graph puts each Actor into the grid cells or rooms "from which that Actor can potentially be seen or heard". Each client then reads the Actor list for the cell its camera is in, instead of every Actor being checked against every client. Dormant Actors (placed in the level and rarely changing) and always-relevant Actors are kept in separate lists. — [Unreal docs: Replication Graph](https://dev.epicgames.com/documentation/en-us/unreal-engine/replication-graph-in-unreal-engine)
- Gaffer's priority accumulator (older, 2015):
  - Each object keeps an accumulator. Every frame, its priority is added to it.
  - Objects are sorted by accumulator, and packets are filled in that order up to the bandwidth budget. The demo used 256 kbit/s and at most 64 state updates per packet.
  - Objects that are sent reset to zero. Objects that did not fit keep their total, so they go first next time.
  - Example priorities: player 1,000,000; interacting objects 100; resting objects 1.

  — [Gaffer On Games: State Synchronization](https://gafferongames.com/post/state_synchronization/)
- RoKarts (older, 2021 reply in the Astro Force thread):
  - Replicates at most 4 karts per frame, chosen by distance. Distant karts update about every 6 frames.
  - Current traffic: about 3 KB/s sent, a receive baseline of about 20 KB/s, plus about 8 KB/s per extra player.
  - Early versions used about 400 KB/s with 6 AI karts.

  — [Astro Force 60x thread](https://devforum.roblox.com/t/how-we-reduced-bandwidth-usage-by-60x-in-astro-force-roblox-rts/1202300)
- Use a 250-stud render distance with a 20-stud cushion so NPCs don't flicker at the edge (sydgyx, Jul 2026 reply). — [Downrest thread](https://devforum.roblox.com/t/performance-tips-network-bandwidth-client-framerate-etc-for-custom-npc-replication/4202236)
- Distant units snap to position instead of interpolating. — [WoloPoints RTS thread](https://devforum.roblox.com/t/how-we-are-developing-an-rts-game-with-good-performance-for-more-than-200-units/3281168)
- Halo: Reach networking (older, GDC 2011, secondary summary): packets are filled in priority order and low-priority objects wait. Unreliable delivery is what makes this prioritization possible. Bandwidth fell by more than 80% versus Halo 3, mostly from design changes rather than networking code. — [Edgegap: Halo Reach netcode deep dive](https://edgegap.com/blog/game-backend-deep-dive-halo-reach-netcode-host-migration)
- replecs per-player filtering:
  - v0.3.0 (Feb 17, 2026) added `collect_entity` / `apply_entity` to send one entity's full state. The release notes call this a temporary workaround because "replecs not calculating filter changes": when a player is added to a filter, you must send them the full entity yourself.
  - It also added a `keep` flag that stops tracking without destroying the entity on the client.
  - Unreliable replication auto-chunks at 1 KB.

  — [replecs releases](https://github.com/PepeElToro41/replecs/releases); [replecs README](https://github.com/PepeElToro41/replecs)
- duplecs (new jecs replicator, last pushed Oct 2, 2026):
  - Builds one filtered packet per client.
  - Sends newly joined clients everything currently visible to them in one packet.
  - Lets entities inherit visibility, for example a unit inheriting "its region's proximity filter".
  - `NetworkedOnce` sends a value only when the component is added, for state clients can predict.
  - Provides unreliable chunks for every-frame data.

  — [duplecs README](https://github.com/demi-dog/duplecs)

**Server CPU levers**
- Parallel Luau docs:
  - Scripts under the same Actor run one after another, so parallelism needs several Actors.
  - `require()` is not allowed while desynchronized.
  - A SharedTable is not copied when sent. Its updates are atomic and visible to all Actors immediately.
  - "For the best performance, use more Actors", but don't use too many, because they are hard to maintain. The docs' terrain example uses 32 Actors.

  — [Roblox docs: Parallel Luau](https://create.roblox.com/docs/scripting/multithreading)
- Worker thread counts (Nov 2024 bug thread, before the Nov 2025 CPU change):
  - Clients in-game: 3 workers on desktop (previously 8), 6 reported on mobile. Studio: 8.
  - Servers: users report a cap of 2. One user wrote "It was always capped at 8 for client and 2 for server."
  - Roblox staff, Apr 17, 2025 (secondhand): worker count "won't be a fixed number and will be tied to the number of CPU cores but there's no timeline".
  - The thread is still unresolved.

  — [DevForum: Worker count capped at 3 in-game](https://devforum.roblox.com/t/worker-count-capped-at-3-in-game/3262835)
- The Nov 2025 CPU announcement thread did not answer a question about the Parallel Luau cap. — [DevForum: Launching even more Server CPU!](https://devforum.roblox.com/t/launching-even-more-server-cpu/4044040)
- Older reports (2023–2024): a 6-player game's server got 1 core, and "live game servers do not allocate more cores for parallelized games". — [DevForum: Do Roblox servers have limitations on processing power?](https://devforum.roblox.com/t/do-roblox-servers-have-limitations-on-processing-power/3047273); [Engine bug thread](https://devforum.roblox.com/t/live-game-servers-do-not-allocate-more-cores-for-parallelized-games/2460052)
- Costs of Parallel Luau:
  - The main thread waits each frame for Actors to finish or yield (snippet). — [DevForum: Parallel Lua also slows down the main thread?](https://devforum.roblox.com/t/parallel-lua-also-slows-down-the-main-thread/4526673)
  - "Sending too much data to different threads has a huge impact in performance"; use buffers. — [WoloPoints 2026 tutorial](https://devforum.roblox.com/t/the-definitive-tutorial-to-game-optimization-everything-you-need-to-know/4629258)
  - Each Actor creates a whole new environment, which raises memory use. The tutorial author lowered his Actor recommendation from 32–64 to 16–32. — [Downrest thread](https://devforum.roblox.com/t/performance-tips-network-bandwidth-client-framerate-etc-for-custom-npc-replication/4202236)
- Native code generation:
  - Enabled with `--!native` or `@native`.
  - Helps most with "a lot of mathematical operations on tables and especially `buffer` types" and with functions called every frame.
  - Annotating `Vector3` arguments gets specialized code.
  - There is a total size cap on native code in a game, plus a 1-million-instruction cap per module.
  - The docs describe it only for server scripts.

  — [Roblox docs: Native code generation](https://create.roblox.com/docs/luau/native-code-gen)
- Other CPU tips:
  - Preallocate tables with `table.create` and pool objects (a bullet pool via `table.create(300)` is given as an example). — [WoloPoints 2026 tutorial](https://devforum.roblox.com/t/the-definitive-tutorial-to-game-optimization-everything-you-need-to-know/4629258)
  - Use `vector.create` SIMD vectors with `--!native` (paswa, Aug 2026). — [WoloPoints RTS thread](https://devforum.roblox.com/t/how-we-are-developing-an-rts-game-with-good-performance-for-more-than-200-units/3281168)
- QuickZone (last pushed Oct 6, 2026) is a physics-free spatial-query library using a linear bounding volume hierarchy, at O(log Z) per lookup. It has a "budgeted scheduler" that spreads entity processing across frames and stops when a time budget (e.g. 1 ms) is used up. It reuses contiguous arrays, so it produces "virtually zero garbage". — [QuickZone README](https://github.com/LDGerrits/QuickZone)

### Inferences
- **Relevancy design.**
  - Use a uniform grid of 128–256-stud cells over the ocean.
  - Each player's interest set is the cells within radius R of their shark, with hysteresis (enter at R, leave at R + 20–40 studs).
  - Each tick, collect schools and fish from those cells, score them by distance, approach speed and whether they are targeted, and fill a per-player byte budget using a priority accumulator.
  - Update-rate tiers: under about 60 studs at 10–15 Hz, 60–150 studs at 5 Hz, 150–R at 1–2 Hz or school record only.
  - On entering view, send a full descriptor reliably. On leaving, despawn on the client after a timeout.
  - Keep each player's "known set" as a reusable bitset in a buffer to avoid creating tables.
- **Write a custom fish channel.** For the fish swarm, a hand-written channel is likely simpler and cheaper than per-entity ECS replication. Relevancy changes constantly as sharks swim. replecs 0.3.x needs a manual full-state send whenever a filter changes, and both replecs 0.4 and duplecs are still unproven.
- **Server CPU plan.**
  - Run the fish simulation at 10–15 Hz, time-sliced across Heartbeat frames.
  - Store fish as preallocated struct-of-arrays (`table.create` or f32 buffers) with free-lists. Avoid per-fish tables and closures.
  - Mark the module `--!native`.
  - Start single-threaded. Add 2–4 region-sharded Actors only if profiling shows the budget is exceeded. If the server has about 2 workers, the best case is about 2x, minus the cost of moving data between threads.
- **Budget for p95, not the average.** SMASH's problem was its p95 frame time, and spikes like that usually come from allocation or GC churn or from bursts of work. That is an inference, since there is no SMASH profile to check. Preallocation and budgeted time-slicing, as in QuickZone, address both.

### Gaps
- I found no official current number for server Parallel Luau worker threads after the Nov 2025 dynamic-CPU rollout.
- The native code generation docs say nothing about client or mobile ARM support.

## 3. Cheap schooling and fish AI, and validating "I ate this fish"

### Takeaway
Combine three cheap behaviours. Calm schools follow an anchor or leader, with each fish's offset computed from a formula. Full boids run only on "active" fish near sharks, with a spatial grid and neighbours capped at about 6–7, which matches the topological rule seen in real flocks. Simple integer-enum state machines (wander, school, flee, chase, eat) drive behaviour, and flow fields are only worth it when many agents share one goal.

The server validates bites itself. It recomputes the fish's position at the client's claimed time: in closed form for deterministic fish, or from a short position history for simulated fish. That time is clamped to a small rewind window, and the bite range gets a tolerance.

### Cited Findings
- Starlings each interact with a fixed number of nearest neighbours, about 6–7, rather than with every bird within a set distance (older, 2008). Later analysis estimates 7 ± 1.5. — [Ballerini et al., arXiv:0709.1916 / PNAS 2008](https://arxiv.org/abs/0709.1916); [PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC2234121/)
- ABZÛ (older, snippet): fish are simulated independently, naturally form schools and scatter when they see a predator. Predators eat smaller fish. — [Kill Screen](https://killscreen.com/abzu-2); [PlayStation Blog](https://blog.playstation.com/?p=156708)
- Spatial grids on Roblox:
  - A "swarm module" grid lets each unit check only nearby cells for targets (WoloPoints).
  - For unit collisions using only CFrame data: axis-aligned cylinders, spatial hash grids and multithreading (Sango).

  — [WoloPoints RTS thread](https://devforum.roblox.com/t/how-we-are-developing-an-rts-game-with-good-performance-for-more-than-200-units/3281168)
- Flow fields:
  - FastFlow (bob_factory) computes one flow field shared by many NPCs. — [Downrest thread](https://devforum.roblox.com/t/performance-tips-network-bandwidth-client-framerate-etc-for-custom-npc-replication/4202236)
  - Flow fields suit only shared objectives; use HPA* with path caching otherwise (paswa).
  - Recheck paths every 5 frames instead of every frame.

  — [WoloPoints RTS thread](https://devforum.roblox.com/t/how-we-are-developing-an-rts-game-with-good-performance-for-more-than-200-units/3281168)
  - Limit how often paths are recalculated, and skip pathfinding when the line to the target is clear (sydgyx). — [Downrest thread](https://devforum.roblox.com/t/performance-tips-network-bandwidth-client-framerate-etc-for-custom-npc-replication/4202236)
- "If you have over 100 NPCs following the player and they all use pathfinding, it will lag." — [WoloPoints 2026 tutorial](https://devforum.roblox.com/t/the-definitive-tutorial-to-game-optimization-everything-you-need-to-know/4629258)
- Existing Roblox boids code:
  - LeoStormer's demo runs boids on the client using Parallel Luau. — [GitHub: Boids-Project-Demo](https://github.com/LeoStormer/Boids-Project-Demo)
  - An open-source DevForum boids library (birds, bats, insects); one commenter says it pairs well with their fish simulation. — [DevForum boids library](https://devforum.roblox.com/t/boids-flocking-swarming-simulation-luau-algorithm-library-animated-birdbatsinsects-v13-open-source-free-model/2806062) (snippet)
- Lag compensation on Roblox (mostly older threads, 2019–2020, snippet):
  - Rewind the target by the attacker's latency and compare with a tolerance of about 6 studs.
  - Keep about one second of position history on the server.
  - Check that the claimed hit point lies on the attacker's ray, and raycast for walls in between.

  — [DevForum: Lag Compensation](https://devforum.roblox.com/t/lag-compensation/290669); [DevForum: Projectile Lag Compensation](https://devforum.roblox.com/t/projectile-lag-compensation/672468)

  A secondary guide recommends capping rewind at about 150–200 ms and rate-limiting attacks, because lag compensation opens new exploits. — [creation.dev guide](https://www.creation.dev/learn/how-to-implement-lag-compensation-roblox-shooters)

### Inferences
- **Calm schools.**
  - Use an anchor that follows a time-parameterized path, such as a spline loop or a Lissajous curve inside the biome's bounds.
  - Each fish's offset is a smooth function of a hash of (seed, index) and t, for example a few summed sines.
  - Clients may layer purely cosmetic local boids or noise on top, since bites are validated against the server's version.
- **Scatter as an event.**
  - Once per school per tick, the server does a grid query for sharks inside the trigger radius.
  - On a trigger, it broadcasts (schoolId, t0, origin, seed) reliably.
  - Server and clients evaluate the same closed-form burst: a radial impulse with drag and a per-fish random spread.
  - After a set time T the school reforms around a new anchor. Only fish being chased are promoted to individual (Tier B) simulation.
- **Active-fish boids.**
  - Use a uniform grid with cell size equal to the neighbour radius, rebuilt each tick into flat arrays with a counting sort.
  - Cap neighbours at 6–7.
  - Compute separation every tick, and alignment and cohesion every second tick.
  - Store state as integer enums in a column, so changing state does not trigger structural (archetype) changes.
- **Predators.**
  - Make decisions (pick a target, give up, eat) at 2–5 Hz and steer at 10–20 Hz.
  - Pick targets with the same grid query.
- **Bite validation.**
  1. The client sends bite(fishRef, clientTime).
  2. The server clamps the claimed time to [now − (RTT/2 + interpolation delay + about 50 ms), now], capped at about 200 ms of rewind.
  3. It computes the fish's position at that time: closed form for Tier A fish, or interpolated from a 10 Hz ring buffer of about 1 s for Tier B/C fish.
  4. It checks the distance to the shark's mouth, using the server's record of the shark's position, against the shark's size-scaled bite radius plus a tolerance of about 1–3 studs.
  5. It enforces a bite cooldown. The first valid claim wins, and the server broadcasts "fish eaten" (school id + index) reliably.

### Gaps
- I found no Roblox-specific source on validating eat or pickup claims against deterministic NPC paths. The lag-compensation sources are written for shooters and are mostly from 2019–2020.

## 4. ECS on Roblox in late 2026: jecs, Planck, replecs, alternatives, and whether ECS is worth it

### Takeaway
jecs is the clear community leader, with 477 GitHub stars against at most 117 for any other Luau ECS. Its latest release is v0.11.0 (Mar 10, 2026), and main has unreleased fixes up to Oct 22, 2026. Planck's latest stable release is still v0.2.0 (Jul 10, 2025); 0.3.0 has been stuck at rc.2 since May 10, 2026, and npm shows an anomalous "latest" tag of 1.0.0. replecs v0.3.1 is stable, and v0.4.0 is a rewrite at rc.5 (Oct 25, 2026). Matter has had no repository activity since Dec 31, 2024. ECR is slow-moving, with its last release v0.9.0 in Mar 2025.

For this game: keep jecs and upgrade to 0.11.0 for gameplay entities, but run the uniform fish swarm as a plain struct-of-arrays module. Keep Planck optional at 0.2.0. Do not use replecs or chrono for fish.

### Cited Findings
**jecs (Ukendio/jecs, 477 stars, last pushed Oct 22, 2026)**
- Release dates:
  - v0.9.0: Sep 15, 2025
  - v0.10.0: Feb 18, 2026
  - v0.10.1: Feb 19, 2026
  - v0.10.2: Feb 20, 2026
  - v0.10.3: Mar 1, 2026
  - v0.10.4: Mar 5, 2026
  - v0.11.0: Mar 10, 2026 (latest; also npm `@rbxts/jecs` latest and Wally `ukendio/jecs` 0.11.0)

  — [jecs releases](https://github.com/Ukendio/jecs/releases); [npm @rbxts/jecs](https://www.npmjs.com/package/@rbxts/jecs); [Wally index](https://github.com/UpliftGames/wally-index/blob/main/ukendio/jecs)
- **Breaking change in v0.11.0:** `on_remove` hooks may no longer make structural changes. This now raises a hard error "regardless of debug mode" when the hook is triggered by `world:remove` or by replacing exclusive ids. The suggested workaround is to queue the change and apply it later. The release also fixed `world:entity` returning nil.
- v0.10.4 fixed `world:entity` returning nil for an explicitly requested id.
- v0.10.0 added:
  - `jecs.new`, `jecs.new_w_id` and `jecs.new_low_id`
  - A debug mode via `jecs.world(true)`
  - `query:fini()`, which removes a query from the cache so it can be garbage-collected
  - `query:archetypes(override)`
  - `how_to`, `examples` and `modules` documentation folders

  — [jecs releases](https://github.com/Ukendio/jecs/releases)
- Unreleased on main as of Oct 2026:
  - `world:targets()` (Apr 21, 2026, #311)
  - A fix so `archetype_destroy` releases its references. Before it, "Worlds that repeatedly create and reclaim empty archetypes" grew without bound (Sep 17, 2026, #340).
  - Signals for the relation part of pairs
  - A CHANGELOG "Unreleased" note that `OnRemove` hooks may move an entity's archetype even during deletion, which appears to relax the 0.11 restriction
  - `archetype_move` improvements (Oct 22, 2026)

  — [jecs commits](https://github.com/Ukendio/jecs/commits/main); [jecs CHANGELOG](https://github.com/Ukendio/jecs/blob/main/CHANGELOG.md)
- The CHANGELOG.md file has no entries for 0.9–0.11; its last versioned entry is 0.8.0. The GitHub release notes are the reliable record. — [jecs CHANGELOG](https://github.com/Ukendio/jecs/blob/main/CHANGELOG.md)
- README claims: "Iterate 800,000 entities at 60 frames per second", "Cache friendly archetype/SoA storage", and "Zero-dependency package". Benchmark results are shown only as images (for example "21,000 entities 125 archetypes 4 random components queried"). — [jecs README](https://github.com/Ukendio/jecs)
- Criticism on the DevForum (snippet): the measured gap between libraries is about 40 µs per run, which is 0.25% of a 60 FPS frame. The jecs benchmark is also said to iterate a precomputed table. — [DevForum jecs thread p.4](https://devforum.roblox.com/t/jecs-optimizing-declarative-scene-graphs-with-ecs/3263203?page=4)
- A forum commenter says jecs supports at most 256 components "because of how its archetype id is built". This comes from a snippet, and the exact thread is uncertain. — [DevForum jecs thread](https://devforum.roblox.com/t/jecs-optimizing-declarative-scene-graphs-with-ecs/3263203) The component-id limit has been configurable with `_G.JECS_HI_COMPONENT_ID` since 0.3.1. — [jecs CHANGELOG](https://github.com/Ukendio/jecs/blob/main/CHANGELOG.md)

**Planck (YetAnotherClown/planck, 55 stars)**
- Releases and tags:
  - GitHub releases: v0.2.0 (Jul 10, 2025) and v0.3.0-alpha.1 (Nov 10, 2025).
  - Wally also has 0.3.0-alpha.2, 0.3.0-rc.1 and 0.3.0-rc.2.
  - npm dist-tags: `rc` = 0.3.0-rc.2 (May 10, 2026), `alpha` = 0.3.0-alpha.2 (Jan 4, 2026), and `latest` = **1.0.0 (Jul 25, 2025)**. Version 1.0.0 does not exist on GitHub or Wally.

  — [Planck releases](https://github.com/YetAnotherClown/planck/releases); [Wally index](https://github.com/UpliftGames/wally-index/blob/main/yetanotherclown/planck); [npm @rbxts/planck](https://www.npmjs.com/package/@rbxts/planck)
- **Breaking changes in 0.3:**
  - Systems must return nil, or one or two functions: the per-frame system plus an optional cleanup ("Initializer Systems").
  - Official roblox-ts typings.
  - Startup systems moved to their own Jabby applet.
  - Conditions now use a falsy check (#42, Apr 22, 2026).
  - Migration to the new type solver (Apr 26, 2026).
  - A commit titled "Cache current running system to avoid `debug.info` call" (Nov 21, 2025).
  - Since May 10, 2026 the only commits are README badge updates (Sep 2026).

  — [Planck v0.3.0-alpha.1 release](https://github.com/YetAnotherClown/planck/releases); [Planck commits](https://github.com/YetAnotherClown/planck/commits/main)

**replecs (PepeElToro41/replecs, 62 stars, last pushed Oct 25, 2026)**
- Releases:
  - v0.2.0: Oct 27, 2025
  - v0.3.0: Feb 17, 2026
  - v0.3.1: Feb 23, 2026 (bumped to jecs 0.10.0)
  - v0.4.0-rc.1: Aug 30, 2026
  - v0.4.0-rc.5: Oct 25, 2026
  - npm `latest` = 0.3.1, `next` = 0.4.0-rc.5

  — [replecs releases](https://github.com/PepeElToro41/replecs/releases); [npm @rbxts/replecs](https://www.npmjs.com/package/@rbxts/replecs)
- **v0.4.0 is a rewrite.** Commits: "push rewrite" and PR #39 "rewrite" (Aug 29–30, 2026), "filter groups" (Sep 26), "remove pending players" (Aug 30). The release notes for the RCs are empty. — [replecs commits](https://github.com/PepeElToro41/replecs/commits/main)
- **v0.3.0:**
  - Added specific pair tracking, `collect_entity` / `apply_entity` (the filter-change workaround), the `keep` flag, and `set_serdes`.
  - Saved one byte per relationship target.
  - Breaking: `replecs.pair` was renamed to `replecs.relation`.
- **v0.2.0:**
  - Added an imperative API that marks entities and components as networked without adding the components, which is "more efficient… but it will disable query-ability".
  - Added handshake verification.

  — [replecs releases](https://github.com/PepeElToro41/replecs/releases)
- README features: per-entity replication, "heavily optimized" granular per-player filtering, entity id remapping, serdes, buffer-based packets, tags and relationships, and unreliable replication with an automatic 1 KB buffer limit. It calls itself "bandwidth efficient" but gives no figures. — [replecs README](https://github.com/PepeElToro41/replecs)

**chrono (parihsz/chrono, 146 stars)**
- The repository describes itself as a "Drop-in custom physics replication library". Latest is v2.2.3 (Oct 18, 2026), described as "Reduce jitter for users with high packet loss".
- v2.2.0 (Sep 20, 2026) added interpolation modes. `CFRAME` writes the primary part's CFrame, `ALIGN` drives AlignPosition and AlignOrientation, and `NONE` disables interpolation. It also mirrors Humanoid state, which shows it works on Instance models.

  — [chrono releases](https://github.com/parihsz/chrono/releases)

**Other libraries**
- **Matter:** last GitHub release v0.8.5 (Dec 9, 2024), v0.9.0-beta.0 (Nov 15, 2024); repository last pushed Dec 31, 2024; npm `latest` is 0.8.5. — [Matter releases](https://github.com/matter-ecs/matter/releases); [npm @rbxts/matter](https://www.npmjs.com/package/@rbxts/matter)
- **ECR** (sparse-set ECS, 60 stars):
  - Last release v0.9.0 (Mar 2, 2025).
  - Unreleased commits in 2026: view list iteration (Apr 8), `ecr.entity` storage change (Jun 3), a breaking change to `Registry:add()` (Jul 7), and a type-inference fix (Sep 3).

  — [ECR repo](https://github.com/centau/ecr); [ECR releases](https://github.com/centau/ecr/releases)
- **Newer or minor:**
  - duplecs (jecs replication, 2 stars, last pushed Oct 2, 2026) — [GitHub](https://github.com/demi-dog/duplecs)
  - pepecs (an ECS by the replecs author, 3 stars, last pushed Oct 10, 2026) — [GitHub](https://github.com/PepeElToro41/pepecs)
  - ECSR (built on Matter, 90 stars) — [GitHub](https://github.com/iiitaoge/ECSR)
  - YetAnotherNet, marked "[Deprecated]" — [GitHub](https://github.com/YetAnotherClown/YetAnotherNet)
  - QuickZone (spatial queries, 64 stars) — [GitHub](https://github.com/LDGerrits/QuickZone)
  - "Stew": the only matching repository found is a 0-star add-on. — [GitHub search result](https://github.com/Universeds/ECS-Stew-ReplicatedSystem-Addon)
- **Is ECS worth it? Community opinions:**
  - "Matter is a great ECS but Jecs's design goal is to prioritize performance first"; Matter's real selling point is organization and ergonomics (snippet). — [DevForum: All about ECS](https://devforum.roblox.com/t/all-about-entity-component-system/1664447)
  - "In Roblox it's all built on tables", and ECS "complicates things in a lot of scenarios" (snippet). — [DevForum: ECS or OOP can't decide](https://devforum.roblox.com/t/ecs-or-oop-cant-decide/4359637)

### Inferences
- **jecs: keep it and upgrade 0.10.4 → 0.11.0.** The only breaking change is structural changes inside `on_remove`, which can be fixed by deferring them to a queue. If gameplay creates and destroys archetypes often, consider pinning a main commit from Sep 17, 2026 or later to get the archetype-growth fix.
- **Use jecs only for varied entities.** That means player sharks, AI predators with abilities, pickups and evolutions. Do not make every fish a jecs entity whose state changes by adding or removing tag components. That causes archetype moves and hook calls on the hottest path.
- **Run the fish swarm as a struct-of-arrays module.** Every fish has the same uniform fields: position, velocity, state, school and species. Hold them in plain parallel arrays or buffers indexed by fish slot, which is also what jecs does internally. The module can expose fish ids to jecs systems if needed.
- **Planck: optional; keep 0.2.0.** Keep it if the Jabby per-system profiler and phase ordering are wanted; otherwise a hand-ordered list of system functions is enough for a game with about 10–20 systems. Do not move to 0.3.0-rc.2 yet: it has been unchanged since May 2026 and has breaking return-value rules. Avoid the npm "1.0.0" tag. A pre-0.3 `debug.info` call was cached away in Nov 2025, which suggests some per-run overhead in 0.2.0. It is unlikely to explain SMASH's 41 ms p95, which more plausibly came from 60 Hz per-entity work and allocation spikes. That cannot be confirmed without a profile.
- **replecs: do not use it for the fish swarm.** If low-rate jecs state (sizes, evolutions, buffs) must replicate, use 0.3.1. Re-evaluate once 0.4.0 is final, because the RCs are a rewrite with no release notes. duplecs's visibility inheritance fits the "school → member fish" model well, but it is too new to rely on.
- **chrono: not suited to fish.** It drives Instance models. Whether it suits player sharks belongs to the movement and replication research.
- **Matter: avoid.** It has been dormant for about 22 months.
- **ECR: no reason to switch to it** from jecs.

### Gaps
- I found no independent 2025–2026 benchmark comparing jecs, ECR and plain Luau arrays on Roblox servers.
- I could not determine where the npm `@rbxts/planck@1.0.0` package came from.
- replecs 0.4.0 has no changelog, so I could not confirm whether it now computes filter changes automatically.
- I could not find an actively maintained "Stew" ECS.

## 5. Server capacity: hardware budgets, entity counts, players per server, memory

### Takeaway
Server Heartbeat is capped at 60 FPS, so the frame budget is 16.67 ms. Server memory follows a formula: about 6.25 GiB plus 100 MiB per peak player, and the docs recommend staying under 50%. Since Nov 10, 2025, server CPU scales with the number of players actually connected, but Roblox does not publish the formula. Community reports from before that change say servers ran only about 2 parallel workers.

Shipped Roblox games run 800–1,000+ data-only units on the server and 10,000 client-rendered NPCs on a low-end phone. A 10–20 player fish game with a few hundred to about 1,000 fish is well within reach if the simulation is tiered and runs at 10–15 Hz.

### Cited Findings
- From the docs: "Server heartbeat is capped at 60 FPS for all games, so lower values might indicate a performance issue." Check server frames against 16.67 ms in the MicroProfiler. — [Roblox docs: Identify performance issues](https://create.roblox.com/docs/performance-optimization/identify)
- Memory formula in the docs: "6.25 GiB + (100 MiB * largest_number_of_connected_players)", and "Try to keep server memory usage below 50%". — [Roblox docs: Identify performance issues](https://create.roblox.com/docs/performance-optimization/identify)
- The announcement of Dec 4, 2024 gave "6.4GB + 100MB * (Peak Number of Players)", up from +50 MB per player earlier in 2024. A later bug report flags the mismatch between the docs and the announcement. — [DevForum: Even More Server Memory](https://devforum.roblox.com/t/even-more-server-memory/3293192); [DevForum: Server Memory Formula Inaccurate on Creator Docs](https://devforum.roblox.com/t/server-memory-formula-inaccurate-on-creator-docs/3913740) (snippet)
- CPU announcement of Nov 3, 2025:
  - From Nov 10–12, 2025, the fixed CPU limit based on the Max Players setting was replaced by allocation that tracks the real-time player count.
  - The baseline is higher.
  - Roblox is "intentionally not sharing the precise formula".
  - Staff: "The same processors but optimized to give you more cores/compute", applied per instance.

  — [DevForum: Launching even more Server CPU!](https://devforum.roblox.com/t/launching-even-more-server-cpu/4044040)
- Parallel workers (older reports, Nov 2024–Apr 2025): about 2 on servers and 3 on desktop clients (6 reported on mobile). Staff say a core-based count is planned but has no timeline. — [DevForum: Worker count capped at 3 in-game](https://devforum.roblox.com/t/worker-count-capped-at-3-in-game/3262835)
- Entity-scale data points:
  - jecs claims "Iterate 800,000 entities at 60 frames per second" for trivial queries. — [jecs README](https://github.com/Ukendio/jecs)
  - 800+ data-only RTS units, using Parallel Luau and a grid. — [WoloPoints RTS thread](https://devforum.roblox.com/t/how-we-are-developing-an-rts-game-with-good-performance-for-more-than-200-units/3281168)
  - 1,500 NPCs at about 3–5% script activity (older, snippet). — [DevForum 1500 NPCs](https://devforum.roblox.com/t/optimization-tips-on-npcs-managed-to-get-1500-npcs-300-500-ping-about-3-5-script-activity-but-rate-is-over-20-30k/2511448)
  - 10,000 NPCs rendered on an iPhone 8 Plus. — [vantoanvh thread](https://devforum.roblox.com/t/10000-physical-npcs-running-with-60-fps-on-low-end-device/3660293)
- Mobile:
  - Roblox's docs say lower-end mobile devices have "severe memory limitations" and can crash from running out of memory (snippet). — [Roblox docs: Design for performance](https://create.roblox.com/docs/en-us/performance-optimization/design)
  - Test with the MicroProfiler on a phone, using low-end devices from roughly 2014–2020. — [WoloPoints 2026 tutorial](https://devforum.roblox.com/t/the-definitive-tutorial-to-game-optimization-everything-you-need-to-know/4629258)
  - A DevForum question post observes that top games run "very low player count per server, like 10-20" (older, opinion, snippet). — [DevForum: Optimal player count per server?](https://devforum.roblox.com/t/optimal-player-count-per-server/910475)

### Inferences
- **What the jecs claim implies.** 800,000 entities per 16.67 ms frame works out to about 48,000 entity iterations per ms, or about 21 ns each, presumably on desktop hardware. Real per-fish work (a grid query, 6–7 neighbours, steering, integration) will cost one to two orders of magnitude more.
- **Even a pessimistic cost is affordable.** At an assumed 5 µs per fish update, 600 actively simulated fish at 10 Hz cost 30 ms per second, which averages about 0.5 ms per 60 Hz frame. These are assumed numbers; benchmark in a live server with the MicroProfiler before relying on them.
- **Suggested fish counts.** About 300–1,000 fish in total, with most of them in Tier A schools. Keep fish simulation plus packet building to about 3 ms or less at p95, and the whole server frame to about 10 ms or less at p95. That leaves headroom under the 16.67 ms cap, and it targets p95 directly, which is what failed in SMASH.
- **Memory.** With 20 players, the docs formula gives about 6.25 + 1.95 ≈ 8.2 GiB, so about 4 GiB under the 50% guideline. Fish data is tiny: 1,000 fish × about 100 B is about 100 KB. Memory pressure will come from assets and Instances, not from the simulation.
- **Players per server.** 10–20 players is consistent with community practice. Server bandwidth out grows linearly with players, at about 5–10 KB/s each for fish under the design above. CPU now scales with connected players. Starting with a 12–16 player cap and raising it after live MicroProfiler and bandwidth data is a reasonable mobile-first default; that is a judgment, not an official recommendation.

### Gaps
- Roblox publishes no figures for current per-server CPU cores, clock speed or parallel worker counts after Nov 2025.
- I found no in-engine Luau benchmark of per-entity boid cost on Roblox server hardware. It needs a microbenchmark.
- I found no official Roblox recommendation for maximum players per server in mobile-heavy games.
