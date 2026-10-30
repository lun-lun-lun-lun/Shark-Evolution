# Decisions

The choices that other work depends on. Every session follows these until an entry is changed.

- **Change a decision:** edit its entry (status, date, new choice, reason), then add a line to [CHANGELOG.md](CHANGELOG.md). Do this before changing the code that depends on it.
- **Add a decision:** append the next number and add it to the table.
- **Never delete an entry.** Mark it `superseded by Dn` instead.

The evidence for D1–D14 is in the [2026-10-30 tech stack report](../reports/Roblox%20shark%20game%20mobile%20tech%20stack.md); its sources are in `research_notes/`.

| ID | Topic | Status |
|---|---|---|
| [D1](#d1-toolchain) | Toolchain | active |
| [D2](#d2-packages) | Packages | active |
| [D3](#d3-luau-type-checking) | Luau type checking | active |
| [D4](#d4-movement-authority) | Movement authority | active |
| [D5](#d5-networking) | Networking | active |
| [D6](#d6-fish-model-and-ecs-scope) | Fish model and ECS scope | active |
| [D7](#d7-bite-and-eat-validation) | Bite and eat validation | active |
| [D8](#d8-world-rendering) | World rendering | active |
| [D9](#d9-fish-and-shark-rendering) | Fish and shark rendering, animation | active |
| [D10](#d10-player-data-and-purchases) | Player data and purchases | active |
| [D11](#d11-input-and-ui) | Input and UI | active |
| [D12](#d12-testing-and-delivery) | Testing and delivery | active |
| [D13](#d13-performance-budgets-and-first-milestone) | Performance budgets and first milestone | active |
| [D14](#d14-name-and-originality) | Name and originality | active |

## D1. Toolchain

- **Status:** active, 2026-10-30
- **Decision:** Rokit 1.2.0 manages every tool, pinned in `rokit.toml`:

  ```toml
  [tools]
  rojo = "rojo-rbx/rojo@7.7.1"
  wally = "UpliftGames/wally@0.3.2"
  lune = "lune-org/lune@0.11.1"
  rodeo = "rodeo-rbx/rodeo@1.5.1"
  blink = "1Axen/blink@0.18.9"
  luau-lsp = "JohnnyMorganz/luau-lsp@1.70.1"
  stylua = "JohnnyMorganz/StyLua@2.6.0"
  selene = "Kampfkarren/selene@0.32.0"
  asphalt = "jackTabsCode/asphalt@2.0.2"
  ```

  Keep the Rojo Studio plugin on the same version as the CLI. No Aftman and no second tool manifest. Don't use Roblox's Script Sync on anything Rojo manages.
- **Why:** Rokit is the community standard and reads Aftman and Foreman manifests; Aftman is archived. Rojo 7.7.1 fixes Windows `\\?\` path bugs. Lune 0.11 fixes `CFrame.lookAt`, `lookAlong` and `CFrame.new(pos, lookAt)` rotations that aren't purely about the Y axis, which headless tests of a pitching, rolling shark depend on. Blink 0.18.9 stops NaN values passing range checks. Script Sync only syncs scripts, treats Studio as the source of truth and produces no sourcemap.
- **Revisit when:** Wally 0.4 is stable, Blink 1.0 ships (see D5), or a tool ships a fix we need. Not yet checked: that Rokit installs the luau-lsp, rodeo and Asphalt release files.

## D2. Packages

- **Status:** active, 2026-10-30
- **Decision:** these Wally 0.3.2 packages:

  | Package | Wally id | Realm | Use |
  |---|---|---|---|
  | jecs | `ukendio/jecs@0.11.0` | shared | Sharks, predators, pickups. Not the fish swarm (D6). |
  | ProfileStore | `lm-loleris/profilestore@1.0.3` | server | Player data (D10) |
  | vide | `centau/vide@0.4.1` | shared | Menus (D11) |
  | TopbarPlus | `1foreverhd/topbarplus@3.5.0` | shared | Topbar icons |
  | LemonSignal | `data-oriented-house/lemonsignal@2.0.0` | shared | Signals |
  | Trove | `sleitnick/trove@1.8.0` | shared | Cleanup |
  | t | `osyrisrblx/t@3.1.1` | shared | Runtime type checks |
  | Jest Roblox | `roblox/jest@3.20.1` | dev | Engine-context tests (D12) |
  | Planck (optional) | `yetanotherclown/planck@0.2.0` | shared | System scheduling, if needed |
  | Charm and charm-sync (optional) | `littensy/charm@0.11.1`, `littensy/charm-sync@0.4.1` | shared | Syncing profile state to clients |

  Not used: Lyra (no release since July 2025, so staying means maintaining a fork), crunchyroll (no bone support, inactive since December 2025), red-blox utilities (archived March 2025), React-lua and Fusion (no releases since 2024), replecs (doesn't fit constantly shifting relevancy), Matter (inactive), pesde (the key packages aren't on it).
- **Why:** these are the maintained options in late 2026. See the report's library sections.
- **Revisit when:** Planck 0.3 is stable (it has sat at release candidate 2 since May 2026), or jecs releases its main-branch archetype fix. At scaffold time, confirm `lm-loleris/profilestore` is the official upload (ProfileStore issue #21 asks).

## D3. Luau type checking

- **Status:** active, 2026-10-30
- **Decision:** `--!strict` in every file, with zero type errors before committing. Trial the new type solver from day one: set `Workspace.UseNewLuauTypeSolver = Enabled` in Studio, turn it on in luau-lsp and use it in CI. Fall back to the old solver only if generic-heavy ECS types produce too many false errors, and record that here.
- **Why:** strict projects stay on the old solver unless they opt in, and Roblox guarantees the old solver only "through 2026".
- **Revisit when:** Roblox announces the old solver's end date or makes the new solver the default for strict mode.

## D4. Movement authority

- **Status:** active, 2026-10-30
- **Decision:** each client owns its shark's physics through network ownership. The server validates movement (3D speed budget, NaN checks, no shark-to-shark collisions) and pulls violators back; it never simulates players. Roblox Server Authority is not used yet. The swim step is a pure function, so it can be adopted later. Details: [shark-movement](systems/shark-movement.md).
- **Why:** Server Authority (generally available since 2026-07-09) supports a mover-driven swimmer on paper. But an open report (2026-07-29, reproduced 2026-08-18) shows input delay growing with each added player at 100+ ms ping, until play becomes unplayable at about four players. Staff put client resimulation at about 6.25× normal load at 100 ms, and testers report stutter above about 40 studs/s. Fully custom prediction (SMASH, Chickynoid) is the most complex option, and SMASH missed its server budget with it.
- **Revisit when:** the high-ping input-delay bug is verifiably fixed AND a multi-device test at 150–300 ms round trip passes with a realistic player count.

## D5. Networking

- **Status:** active, 2026-10-30
- **Decision:**
  - Typed low-rate events (bites, eats, spawns, growth) go through Blink 0.18.9. A byte-length check and a per-player rate check run before decoding.
  - Continuous fish and predator state is one hand-written stream of 10-byte quantized records over an UnreliableRemoteEvent: at most 950 B per packet, sequence-numbered, 10–20 Hz near a player and 2–5 Hz farther out, filtered by an interest grid.
  - No acknowledged delta snapshots, no custom codec generator, no 60 Hz streams.
  - Per-client ceiling 32 KiB/s; typical 10,000–20,000 B/s.

  Details: [networking](systems/networking.md).
- **Why:** Blink, Zap and ByteNet use about the same bandwidth, and all three have an open oversized-buffer server crash (Blink #45, Zap #219, ByteNet #34), so switching buys no safety. Deltas against acknowledged baselines save little when fish move every tick and baselines are already stale on 300 ms links. The unreliable payload cap has been 1,000 B since 2025-03-12, checked after compression.
- **Revisit when:** Blink 1.0 ships quantized numbers, bit-packing and the oversized-buffer guard, or a time-boxed trial of BlinkBlox (a September 2026 Blink fork with self-reported benchmarks) shows a clear win.

## D6. Fish model and ECS scope

- **Status:** active, 2026-10-30
- **Decision:** three fish tiers: A, calm schools (closed-form, never stepped); B, reacting fish (server arrays, about 10 Hz); C, AI predators (jecs entities). Fish are never server Instances. No stepped flocking that has to match across machines; per-fish randomness comes from `bit32` hashes. The swarm lives in parallel arrays, not the ECS. Details: [fish-simulation](systems/fish-simulation.md).
- **Why:** server cost then scales with schools, not fish. A closed-form position has no error that builds up between x86 servers and ARM phones, and Roblox's `Random` isn't guaranteed identical across platforms. Keeping NPCs as data and rendering them on clients is the pattern every Roblox game that scaled NPCs converged on (Astro Force, a 10,000-NPC iPhone 8 Plus test, an 800+ unit RTS).
- **Revisit when:** measurements show tier-B cost or fish counts outside the D13 budgets.

## D7. Bite and eat validation

- **Status:** active, 2026-10-30
- **Decision:** a bite is a client claim, `{targetId, renderServerTime}`. The server validates it by rewinding the target, clamped per player and per tier:
  - tier A: about half the round trip
  - tiers B and C: round trip + interpolation buffer + margin, with a ceiling near 500 ms
  - PvP: a ceiling of about 150–250 ms that favours the victim

  The first valid claim wins. Every bite is logged (rewind, round trip, result, reason) for tuning. Details: [eating-and-bites](systems/eating-and-bites.md).
- **Why:** Roblox has no built-in server rewind under any authority model. Clamping per player stops low-ping players from claiming stale bites. The PvE ceiling wrongs no human; the PvP ceiling protects victims.
- **Revisit when:** logged mobile traffic shows rejection rates or player complaints that call for different caps.

## D8. World rendering

- **Status:** active, 2026-10-30
- **Decision:**
  - Fog caps visibility at about 120–200 studs.
  - StreamingEnabled is on, with Opportunistic stream-out, Improved model streaming, a target radius of 256–512 and a minimum radius of 64–128.
  - Lighting is Soft, or Realistic with PrioritizeLightingQuality off, chosen by comparing both on a real phone. Bloom is subtle or off; no depth of field or sun rays.
  - No Terrain water: the water is faked.

  Details: [rendering](systems/rendering.md).
- **Why:** low-end devices can already cut draw distance to 200 studs. Most mobile crashes are memory-related, and streaming is Roblox's biggest memory lever. Future lighting (Realistic with PrioritizeLightingQuality on) gives up draw distance on weak devices. Terrain water has open streaming bugs.
- **Revisit when:** Roblox ships the planned 500-stud minimum draw distance or adaptive streaming radius, or re-enables Expanded Terrain's client performance gains (switched off on 2026-10-27).

## D9. Fish and shark rendering

- **Status:** active, 2026-10-30
- **Decision:**
  - Each fish species is one MeshPart asset with a shared appearance, tinted by Color.
  - Fish are anchored, pooled on the client, with physics and shadow flags off, moved by one `BulkMoveTo` per frame and updated less often with distance.
  - Skinned meshes only for player sharks and the 20–30 nearest fish.
  - Player sharks use AnimationController + Animator + one Animation Graph.
  - No Humanoids, no EditableMesh, no crunchyroll.

  Details: [rendering](systems/rendering.md), [animation](systems/animation.md).
- **Why:** instancing needs an identical mesh and appearance. Skinned meshes appear not to instance (one test went from 300 to over 1,000 draw calls). EditableMesh has tight memory budgets and creator restrictions. Animation Graphs (generally available since July 2026) run player graphs on the owning client.
- **Revisit when:** a prototype shows Animation Graphs don't work on bone rigs, or SLIM supports NPCs (planned for early 2027).

## D10. Player data and purchases

- **Status:** active, 2026-10-30
- **Decision:**
  - ProfileStore 1.0.3. Each profile has a schema version, and ordered migrations run before `Reconcile`.
  - Purchases are idempotent through a ring buffer of PurchaseIds in the profile.
  - Leaderboards are written at session end; the live board uses a MemoryStore sorted map.
  - Analytics are aggregated per life.

  Details: [persistence](systems/persistence.md).
- **Why:** Lyra's strengths (transactions, sharding) solve problems this game doesn't have, and it hasn't released since July 2025. ProfileStore's session lock suits flaky mobile rejoins. Ordered writes are the scarcest DataStore budget (30 + 5 × players per server per minute).
- **Revisit when:** the game adds trading; then reconsider transactions.

## D11. Input and UI

- **Status:** active, 2026-10-30
- **Decision:**
  - All gameplay input goes through the Input Action System: Move, Look, Boost, Bite.
  - vide for menus; the HUD is written directly at 10–15 Hz.
  - `ScreenInsets = CoreUISafeInsets`; touch targets of at least 9 mm (about 56 px).
  - TopbarPlus 3.5.0 for topbar icons.

  Details: [input-and-controls](systems/input-and-controls.md), [ui](systems/ui.md).
- **Why:** the Input Action System has been generally available since 2026-06-11, is the only input path Server Authority accepts, and will be required by default PlayerScripts by mid-2027. vide is the only declarative UI library releasing in 2026.
- **Revisit when:** the IAS gaps around the thumbstick and four-button movement are fixed, or they force a custom joystick.

## D12. Testing and delivery

- **Status:** active, 2026-10-30
- **Decision:**
  - Tests: Lune headless specs for logic; Jest Roblox through Open Cloud Luau Execution against a separate test universe; rodeo for local play-mode tests.
  - Network tests: Studio's Network Simulator plus real phones on cellular.
  - CI and publishing: GitHub Actions with `setup-rokit`; publish through the Open Cloud place-publishing API; Asphalt for assets.
  - AI agents edit files on disk and use Studio's MCP server only to run, playtest and inspect.

  Details: [testing-and-performance](systems/testing-and-performance.md).
- **Why:** TestEZ is archived. Luau Execution runs real engine code from CI. Tarmac and Mantle are stale. Studio edits to Rojo-synced scripts get overwritten.
- **Revisit when:** Roblox ships on-device testing from Studio (planned for early 2027), or the publishing API is shown to handle SurfaceAppearance and unions safely.

## D13. Performance budgets and first milestone

- **Status:** active, 2026-10-30
- **Decision:**
  - Budgets:
    - server frame p95 ≤ 10 ms
    - fish simulation plus packet building p95 ≤ 3 ms
    - client memory ≤ 500–600 MB on 2–3 GB phones
    - client Luau 2–4 ms per frame
    - network typically 10,000–20,000 B/s per client, ceiling 32 KiB/s
  - First milestone, before content production: 300 fish and 20 sharks on a 2–3 GB Android phone and an older iPhone over real cellular, measured against these budgets.

  Details: [testing-and-performance](systems/testing-and-performance.md).
- **Why:** SMASH failed on p95 (about 41 ms against a 12 ms target), not on averages. Every fish-movement cost in the research was measured on PC. The riskiest assumptions are cheap to test now and expensive to discover late.
- **Revisit when:** measured numbers come in. Record them on the testing-and-performance page.

## D14. Name and originality

- **Status:** active, 2026-10-30
- **Decision:** "Shark Evolution" is a working title; publish under a distinct name. Sharks, characters, UI, maps, logo and audio are original. Keep a record of where each asset came from (an asset provenance page, once assets exist).
- **Why:** "Hungry Shark" is a Ubisoft trademark, the franchise is still supported, and rights holders can have whole games removed from Roblox. Genre ideas (eating to grow, real species, boosting, PvP biting) aren't protected. This is a reading of Roblox's policy, not legal advice; no trademark registry was checked.
- **Revisit when:** a title is chosen; then check that it isn't confusingly close to an existing game.
