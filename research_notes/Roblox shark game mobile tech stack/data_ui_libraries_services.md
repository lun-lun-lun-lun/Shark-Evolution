# Data, UI, Animation, State/Utility Libraries, Monetization/Analytics and IP Risk for a Mobile-First Roblox Shark Game (state as of 30 Oct 2026)

Scope: persistence, UI, animation, state, utilities, monetization/analytics, and an IP check for an original "shark eats fish, grows, evolves" Roblox game. It targets 10-20 players per server, mobile first, and poor networks. Toolchain, networking, ECS, rendering and input are out of scope.

How the data was collected: version and date facts come from GitHub release pages, GitHub Atom feeds (`/releases.atom`, `/commits/<branch>.atom`), repo `wally.toml` files, and the Wally registry index (`raw.githubusercontent.com/UpliftGames/wally-index/main/<scope>/<name>`), all read on 30 Oct 2026. The GitHub REST API was rate-limited, so the Atom feeds supplied the exact years. Some GitHub release pages show only day and month. Where that happened, the year comes from the Atom feed.

---

## 1. Player data persistence: Lyra vs ProfileStore vs others, plus the 2026 DataStore, MemoryStore and OrderedDataStore limits

### Takeaway
**ProfileStore** (Wally `lm-loleris/profilestore@1.0.3`) is the lower-risk default for this game. Lyra has had no tagged release since v0.6.0 (8 Jul 2025). Its own README still calls it "early development". Main carries unreleased fixes, and nothing upstream addresses lock refresh. The player data here is small and has no trades, so Lyra's main advantages (transactions, auto-sharding) aren't needed.

The DataStore platform changed twice in 2025-2026:
- New access and storage limits were scheduled to start in April 2026, with a paid "Extended Services" overflow.
- On 29 Jul 2026 storage rose to **500 MB + 1 MB × lifetime players**. In-game and Open Cloud requests now share one per-experience budget with a base of 300.

### Cited Findings

**Library status snapshot (30 Oct 2026)**

| Library | Latest published version (date) | Last repo activity | Source |
|---|---|---|---|
| Lyra (paradoxum-games) | 0.6.0 on Wally and as a GitHub release (8 Jul 2025). Wally index lists only 0.3.3, 0.4.0, 0.4.1, 0.5.0-rc.0, 0.6.0 | Main commits up to 25 Mar 2026; open PR activity up to 20 Aug 2026 | [Lyra releases](https://github.com/paradoxum-games/lyra/releases), [releases Atom](https://github.com/paradoxum-games/lyra/releases.atom), [Wally index](https://raw.githubusercontent.com/UpliftGames/wally-index/main/paradoxum-games/lyra), [commits](https://github.com/paradoxum-games/lyra/commits/main) |
| ProfileStore (loleris / MadStudioRoblox) | Wally `lm-loleris/profilestore` 1.0.0 to 1.0.3. The repo's own wally.toml declares `lm-loleris/profilestore` 1.0.3, realm "server". No GitHub releases exist | Last commit 31 Jul 2025 | [wally.toml](https://raw.githubusercontent.com/MadStudioRoblox/ProfileStore/main/wally.toml), [Wally index](https://raw.githubusercontent.com/UpliftGames/wally-index/main/lm-loleris/profilestore), [releases (none)](https://github.com/MadStudioRoblox/ProfileStore/releases), [commits](https://github.com/MadStudioRoblox/ProfileStore/commits/main) |
| Suphi's DataStore Module | Version not verified. A third-party GitHub mirror shows a "V0.7" tag | DevForum thread active (date not verified) | [DevForum thread](https://devforum.roblox.com/t/suphis-datastore-module/2425597), [mirror](https://github.com/NameTakenBonk/SuphisDataStoreModule/releases/tag/V0.7) |
| DataStore2 (Kampfkarren) | No `kampfkarren/datastore2` entry in the Wally index (own check) | Last commit 17 Mar 2023 ("Add support for DataStore2.0 GDPR Tags") | [Kampfkarren/Roblox commits](https://github.com/Kampfkarren/Roblox/commits/master) |

**Lyra details**
- README features:
  - Transactions: atomic multi-player operations such as `txAsync`; the README says these make item duplication impossible.
  - Session locking.
  - Validation with `t` schemas (`t.strictInterface`).
  - Auto-sharding of large data across multiple keys.
  - Migrations ("change your data format without breaking existing saves").
  - Drop-in import of existing data.
  — [Lyra README](https://github.com/paradoxum-games/lyra)
- README status: "has been tested and is used in production" but "still in early development". It advises against using Lyra in production games where data loss would be catastrophic until it has been tested more. Repo stats: 155 stars, 552 commits, 3 open PRs, 2 open issues. — [Lyra README](https://github.com/paradoxum-games/lyra)
- Release history:
  - **v0.6.0 (8 Jul 2025):** immutable update and transaction APIs (`updateImmutable`, `updateImmutableAsync`, `txImmutable`, `txImmutableAsync`). Breaking: removed `disableReferenceProtection` in favor of frozen tables. `changedCallbacks` now reconcile into a copy-on-write table.
  - **v0.5.0-rc.0 (3 May 2025):** "fixed a race condition with locks and added tests".
  - **v0.4.0 (30 Jan 2025):** `PlayerStore:peek(userId)`; sharding switched to JSON-encoded buffers.
  — [Lyra releases](https://github.com/paradoxum-games/lyra/releases)
- Main already bumps `wally.toml` to **0.6.1** (authors nomer888 and outofbears; dependencies `evaera/promise@4.0.0` and `osyrisrblx/t@3.1.1`). 0.6.1 has **not** been published to Wally. — [Lyra wally.toml](https://raw.githubusercontent.com/paradoxum-games/lyra/main/wally.toml), [Wally index](https://raw.githubusercontent.com/UpliftGames/wally-index/main/paradoxum-games/lyra)
- Unreleased changes on main since 0.6.0. — [commits](https://github.com/paradoxum-games/lyra/commits/main), [PRs](https://github.com/paradoxum-games/lyra/pulls?q=is%3Apr)
  - `useMock` for PlayerStore (9 Aug 2025).
  - Type fixes (1 Sep 2025).
  - "Prevent duplicate entries in orphaned files list" (3 Nov 2025).
  - Optional `useDSSLocking` (DataStoreService-based locking) and "ignoring existing locks" when testing in Studio (8-10 Nov 2025; PR #17 merged 21 Nov 2025).
  - PR #16 "Prevent exponential orphan file growth in Session:orphanFile()" (merged 22 Nov 2025).
  - "fix mergeDeep and copyDeep clearing buffers" (25 Mar 2026).
- Open PRs:
  - #18 performance improvements for immutable updates (13 Feb 2026).
  - #20 "Add `lockShardCount` to shard the MemoryStore lock HashMap" (1 May 2026).
  - #22, another "fix mergeDeep and copyDeep clearing buffers" (20 Aug 2026).
  - #21 "Fix UpdateAsync version hour comparison" was closed without merging (17 May 2026).
  — [Lyra PRs](https://github.com/paradoxum-games/lyra/pulls?q=is%3Apr)
- No Lyra issue, PR or commit title mentions lock **refresh, renewal or heartbeat**. — [issues](https://github.com/paradoxum-games/lyra/issues?q=is%3Aissue), [PRs](https://github.com/paradoxum-games/lyra/pulls?q=is%3Apr), [commits](https://github.com/paradoxum-games/lyra/commits/main)
- Open issue #6 (22 Feb 2025): "get + getAsync do not appear to do any yielding operation, yet the API says they are async". — [Lyra issues](https://github.com/paradoxum-games/lyra/issues?q=is%3Aissue)

**ProfileStore details**
- Described as a "Periodic DataStore saving solution with session locking". It is a single ModuleScript and "is not designed (and never will be) for in-game leaderboards or any kind of global state". Repo stats: 350 stars, 66 forks, 5 open PRs, 1 open issue. — [ProfileStore repo](https://github.com/MadStudioRoblox/ProfileStore)
- The only open issue is #21 (28 Jun 2026), which asks whether an official Wally upload exists. No open data-loss or locking bug reports. — [ProfileStore issues](https://github.com/MadStudioRoblox/ProfileStore/issues)
- Recent commits: "Blocking prevention for MessagingService" (25 Mar 2025); type fix (18 Jun 2025); PR merge (31 Jul 2025). — [commits](https://github.com/MadStudioRoblox/ProfileStore/commits/main)
- API behavior. — [ProfileStore API docs](https://madstudioroblox.github.io/ProfileStore/api/)
  - When a server starts a session on a key that is already in use, the owning server is told "to make a final save" before the new session is granted.
  - The `Steal` option skips that final save. The docs warn: "DO NOT USE THIS ARGUMENT FOR LOADING PLAYER DATA NORMALLY".
  - A `Cancel` callback lets you abort a pending start.
  - `StartSessionAsync()` can return nil in a rare race, and the docs suggest kicking the player.
  - `Profile:Reconcile()` fills missing fields from a template.
  - `MessageAsync` costs one `UpdateAsync` to send plus one on the receiving server. It is meant for critical data such as gifting paid items.
  - `OnSessionEnd` fires even when a session is stolen. `OnLastSave` reports "Manual", "External" or "Shutdown".
  - A Mock store is available.
  - `SetConstant` accepts `AUTO_SAVE_PERIOD`, `START_SESSION_TIMEOUT` and `SESSION_STEAL`.
  - Use `Profile:Save()` for critical moments, such as making sure developer-product purchase data is saved before a server crash. `Profile.LastSavedData` is described as useful "for securely handling developer product purchases".
  - Forgetting `EndSession()` makes the game "perform more and more DataStore requests".
- A community answer describes ProfileStore as the successor to ProfileService with no lost features. — [DevForum](https://devforum.roblox.com/t/should-i-use-profilestore-or-profileservice/3871636)

**Suphi's DataStore Module (alternative)**
- Stores the session lock in MemoryStore with built-in expiration, rather than relying on `os.time()` as ProfileService does. — [GitHub mirror, third party](https://github.com/NameTakenBonk/SuphisDataStoreModule/releases/tag/V0.7), [DevForum reply](https://devforum.roblox.com/t/data-session-locking/2521423/2)
- Locking cannot be disabled.
- One user reported a case where two servers opened the same key, with no visible resolution.
- One user reported 200+ CCU with no data issues; another hit MemoryStore limits at scale.
- These are anecdotal forum posts. — [DevForum p7](https://devforum.roblox.com/t/suphis-datastore-module/2425597?page=7), [DevForum p3](https://devforum.roblox.com/t/suphis-datastore-module/2425597?page=3)

**DataStore limits (current Creator Hub page)** — [Data store error codes and limits](https://create.roblox.com/docs/cloud-services/data-stores/error-codes-and-limits)
- Per-server budgets, per minute:
  - StandardRead, StandardWrite, StandardRemove: 60 + numPlayers × 40.
  - StandardList: 5 + numPlayers × 2.
  - OrderedRead: 60 + numPlayers × 40.
  - OrderedWrite and OrderedRemove: 30 + numPlayers × 5.
  - OrderedList: 5 + numPlayers × 2.
  - Servers get a one-time startup burst.
  - Budgets can be changed with `SetRateLimitForRequestType()`.
- Per-experience budgets, per minute, shared by game servers and Open Cloud:
  - Standard Read and Standard Remove: 300 + CCU × 40.
  - Standard Write: 300 + CCU × 20.
  - Standard List: 300 + CCU × 2.
  - Ordered Read and Ordered Remove: 300 + CCU × 40.
  - Ordered Write: 300 + CCU × 20.
  - Ordered List: 300 + CCU × 2.
- `UpdateAsync` consumes both read and write budget.
- Per-key throughput: 25 MB/min read and 4 MB/min write (rolling 60 s), each request rounded up to the next KB.
- Size limits: value 4,194,304 per key (serialized; check with `JSONEncode`). Data store name, key name and scope are 50 characters each. Metadata totals 300 characters.
- Storage: **500 MB + 1 MB × lifetime user count**. Measured as the compressed size of the latest version of each key, so don't pre-compress.
- Request queues hold 30 per type. Overflow produces errors 301-306. Throttle errors include `StandardWriteExperienceThrottled`.

**2025-2026 DataStore platform changes**
- April 2025: Roblox announced updated DataStores access and storage limits and a paid "Extended Services" option for very large experiences. This comes from search-result snippets; the post itself was not fetched. — [Announcing Roblox Extended Services](https://devforum.roblox.com/t/announcing-roblox-extended-services/3621439)
- 14 Nov 2025 "Extended Services Update". — [Extended Services Update](https://devforum.roblox.com/t/extended-services-update/4070634)
  - New Data Stores Access & Storage limits were to start in April 2026, with Extended Services available beforehand.
  - Storage alerts fire automatically at 60%, 80%, 90% and 100%.
  - Above 100%, Data Stores Manager shows estimated costs.
  - Roblox said fewer than 30 of 55 million experiences would hit the limit.
  - No prices are listed. A commenter's "$0.12/GB/month" figure is unofficial.
- Announced 15 Jul 2026, live **29 Jul 2026**. — [Unifying Data Stores Open Cloud and Game APIs, and Increasing Storage Limits](https://devforum.roblox.com/t/unifying-data-stores-open-cloud-and-game-apis-and-increasing-storage-limits/4739240), [Weekly Recap 13-17 Jul 2026](https://devforum.roblox.com/t/weekly-recap-july-13-%E2%80%93-17-animation-graphs-go-live-5%C3%97-more-data-store-storage/4743286)
  - Base storage went from 100 MB to 500 MB, giving 500 MB + 1 MB × lifetime players.
  - In-game and Open Cloud DataStore requests now draw from the same per-experience counters, with the base raised to 300.
  - Open Cloud usage is now visible in the dashboard.
  - Extended Services now covers Open Cloud DataStores.
  - Per-key limits, including 4 MB/min write, did not change.

**MemoryStore (cross-server features)** — [Memory stores docs](https://create.roblox.com/docs/cloud-services/memory-stores)
- Memory quota: 64 KB + 1.2 KB × users. This is dynamic and takes 8 days to drop after users leave.
- Request units: 1000 + 120 × CCU per minute. Exceptions to the one-unit cost:
  - `GetRangeAsync`: one unit per item returned.
  - `ReadAsync`: one unit per item, plus an extra unit every two seconds while reading.
  - `HashMap:UpdateAsync`: at least 2 units.
  - `ListItemsAsync`: partitions scanned plus items returned.
- Per sorted map or queue: 1,000,000 items and 100 MB.
- Partition throttling starts around 30,000 RU/min, and sorted maps and queues each sit on a single partition.
- Hash map per-key limits: about 5,000 write RU/min and 15,000 read RU/min.
- Item value maximum is 32 KB. Maximum expiration is 3,888,000 s (45 days).
- Data structures: sorted map, queue, hash map.

### Inferences
- **Choose ProfileStore 1.0.3 for this game.** The profile (currency, unlocked sharks, upgrades, settings, a version number and recent receipt IDs) will be a few KB. That is far below the 4 MB per-key cap, and sharding is never needed. With no trading, Lyra's `txAsync` brings no benefit.
  - ProfileStore is feature-complete and has a single open question-type issue. The developer already knows it from an experimental project.
  - Lyra needs a vendored lock-refresh patch that upstream has not addressed. Its newest fixes (orphan-file growth, buffer clearing, DataStoreService-lock option) exist only on unreleased main.
  - Staying on Lyra therefore means owning a fork anyway.
- **If the developer stays on Lyra anyway**, pin to a reviewed main commit rather than Wally 0.6.0, because that brings in the orphan-file and buffer fixes. Keep the lock-refresh patch, and try to upstream it.
  - The `useDSSLocking` flag and the open "shard the MemoryStore lock HashMap" PR suggest Lyra's default lock lives in a MemoryStore HashMap. If so, loads depend on MemoryStore health and quota (1000 + 120 × CCU RU/min), which is shared with any other MemoryStore features. This is inferred from PR and commit titles only.
- **Migrations with ProfileStore:**
  - Store `Data.SchemaVersion`.
  - After `StartSessionAsync`, run ordered migration functions `vN to vN+1`, then `Reconcile()`.
  - ProfileStore's Reconcile only adds missing keys. Renames and type changes need explicit migration steps.
- **Poor mobile networks mean frequent disconnects and rejoins**, so session hand-off speed matters. ProfileStore's "ask the owner to final-save" mechanism suits this.
  - Use the `Cancel` callback to stop waiting if the player has already left.
  - Kick on a nil session.
  - Never use `Steal` for normal loading.
  - Always call `EndSession` on `PlayerRemoving`.
- **Budget headroom (20-player server):**
  - Standard write budget is 60 + 20 × 40 = 860/min per server.
  - Ordered write is only 30 + 20 × 5 = 130/min, and ordered list only 5 + 20 × 2 = 45/min.
  - Autosaving 20 profiles every 1-5 minutes uses a small fraction of the standard budget.
  - Leaderboard writes (OrderedDataStore) are the scarce resource.
- **Global leaderboards:**
  - All-time stats such as "largest size reached" or "total fish eaten": write to OrderedDataStore at session end or every few minutes. Do not write per kill.
  - Each server should fetch top N with `GetSortedAsync` at most about once per 60 s and cache the result.
  - Daily or weekly boards can use one OrderedDataStore per period key, for example `Weekly_2026W44`.
  - A live cross-server "biggest sharks right now" board fits a MemoryStore sorted map with a short TTL (for example 120 s), refreshed by each server.
- **Storage is not a concern** at 500 MB + 1 MB per lifetime player with KB-sized profiles. Don't log per-event history (fish eaten, deaths) into DataStores; send it to AnalyticsService instead (see Section 5).

### Gaps
- ProfileStore's default `AUTO_SAVE_PERIOD`, `START_SESSION_TIMEOUT` and lock-expiry values were not shown on the fetched docs page.
- Lyra's lock TTL and refresh defaults are not documented in its README. The specific lock-refresh bug the developer patched could not be found upstream, so it can't be confirmed whether main fixes it.
- Suphi's module: current version and maintenance date not verified.
- DataStore2: session-locking behavior not re-verified in this session. Its repo has been inactive since Mar 2023.
- Enforcement when an experience exceeds storage without buying Extended Services is not described (only "estimated costs" are shown). Official Extended Services pricing was not found.
- No 2025-2026 independent data-loss incident reports were found for either ProfileStore or Lyra.

---

## 2. UI: vide vs React-lua vs Fusion vs plain Instances, and 2026 mobile UI practice

### Takeaway
**Vide** is the only one of the three declarative libraries with releases in 2026: 0.4.0 on 17 Jan 2026 and 0.4.1 on 11 Jul 2026, with more fixes merged in late Oct 2026. React-lua (jsdotlua) has been dormant since Dec 2024. Fusion's last release was 0.3 in Aug 2024, though main still gets occasional fixes. Keep vide and upgrade to 0.4.1.

For mobile:
- Use `ScreenGui.ScreenInsets` (default `CoreUISafeInsets`) and avoid the legacy `FullscreenExtension`.
- Size touch targets to about 9 mm.
- Keep text at TextSize 20 or more where possible, and never let MinTextSize go below 9.
- Use flex layouts only where needed.
- Treat CanvasGroup as a memory-capped luxury.
- Upgrade TopbarPlus to 3.5.0, released 19-20 Oct 2026.

### Cited Findings

**Library maintenance**
- **vide (centau):**
  - 0.4.1 (11 Jul 2026): allows `branch()` inside reactive scopes, deprecates the `create()` overloads (only `create(class)(props)` remains), and fixes `create()` types in the new Luau solver.
  - 0.4.0 (17 Jan 2026): the largest release. Adds `create("ClassName", {props})` syntax, a `spring()` setter, and delayed scope destruction. Breaking: removed aggregate construction, active scopes can't be destroyed, and implicit child effects unparent children.
  - Earlier releases: 0.3.1 (9 Oct 2024), 0.3.0 (6 Oct 2024), 0.2.0 (22 Nov 2023).
  - All releases are marked pre-release.
  — [vide releases](https://github.com/centau/vide/releases), [releases Atom](https://github.com/centau/vide/releases.atom)
- Vide commits after 0.4.1 (unreleased):
  - "Fix scope stack staying broken after a yield in strict mode (#105)" (22 Oct 2026).
  - "Fix actions not running in priority order (#107)" (22 Oct 2026).
  - "feat(create): Add styling types (#94)" (26 Oct 2026).
  — [vide commits](https://github.com/centau/vide/commits/main)
- Vide's README describes it as "a reactive Luau UI library inspired by Solid", fully Luau type-checkable, declarative and reactive. It makes no performance claims. Repo stats: 352 stars, 59 forks, 9 open issues. The repo root ships `wally.toml`, `pesde.toml` and `rokit.toml`. — [vide README](https://github.com/centau/vide)
- **React-lua (jsdotlua):**
  - Latest release v17.2.1 (4 Dec 2024). v17.2.0 came out 27 Sep 2024.
  - Last commit 4 Dec 2024 ("Fix dev-dependencies and test files requires"). The prior fix was "Fix bindings leak (#40)" on 5 Nov 2024.
  - 17 open issues.
  - Wally `jsdotlua/react` and `jsdotlua/react-roblox` top out at 17.2.1.
  — [react-lua releases](https://github.com/jsdotlua/react-lua/releases), [commits](https://github.com/jsdotlua/react-lua/commits/main), [Wally index](https://raw.githubusercontent.com/UpliftGames/wally-index/main/jsdotlua/react)
- **Fusion (dphfox):**
  - Latest release "Fusion 0.3" (v0.3-beta, 30 Aug 2024). No 0.4 has been released.
  - 0.3 introduced use-functions in place of `:get()`, scopes in place of Maids, contextuals, safe expressions, and hybrid push/pull execution.
  - Main still receives fixes: Oklab blending (26 Oct 2025), `ForPairs` error context (31 Dec 2025), and "scope should invoke :disconnect() and :Disconnect methods" (2 Feb 2026).
  - Wally `elttob/fusion` latest is 0.3.0.
  — [Fusion releases](https://github.com/dphfox/Fusion/releases), [Fusion commits](https://github.com/dphfox/Fusion/commits/main), [Wally index](https://raw.githubusercontent.com/UpliftGames/wally-index/main/elttob/fusion)
- No benchmark comparing vide, Fusion and React-lua on mobile turned up in searches. The Fusion README claims it "can run smoothly across mobile, console, PC, and VR", without benchmarks. — [search result: Fusion repo fork](https://github.com/raphtalia/fusion) (a fork; treat with caution)

**Safe areas and insets**
- `ScreenGui.ScreenInsets` values. — [ScreenGui reference (md)](https://create.roblox.com/docs/reference/engine/classes/ScreenGui.md), [ScreenInsets enum](https://create.roblox.com/docs/reference/engine/enums/ScreenInsets)
  - The enum is None (0), DeviceSafeInsets (1), CoreUISafeInsets (2), TopbarSafeInsets (3).
  - The default, **CoreUISafeInsets**, keeps all descendants inside the core-UI safe area, clear of both the topbar and device cutouts. Roblox recommends it for content that must stay unobscured.
  - **DeviceSafeInsets** avoids notches and cutouts but adds no inset for Roblox's topbar buttons.
  - **TopbarSafeInsets** is a dynamic area "limited to the space available within the top bar area itself".
- `ScreenGui.SafeAreaCompatibility` has two options, None and FullscreenExtension, with **FullscreenExtension** as the default. FullscreenExtension extends only the background of a qualifying fullscreen object. Roblox recommends avoiding it "for new work" and using ScreenInsets instead. — [ScreenGui reference (md)](https://create.roblox.com/docs/reference/engine/classes/ScreenGui.md); history in [Notched Screen Support - Full Release](https://devforum.roblox.com/t/notched-screen-support-full-release/2074324) (older post, about 2022)

**Touch targets and text**
- Roblox's UI curriculum cites the WCAG touch-target recommendation of at least **9 × 9 mm** on mobile. — [Choose an art style](https://create.roblox.com/docs/tutorials/curriculums/user-interface-design/choose-an-art-style)
- Text-size guidance:
  - Don't set `MinTextSize` below 9. — [Size modifiers and constraints](https://create.roblox.com/docs/ui/size-modifiers)
  - Choose fonts readable on phones and test at the smallest expected size on real devices. — [Adaptive design guidelines](https://create.roblox.com/docs/production/publishing/adaptive-design)
  - Use a text size constraint so text doesn't get too small or too large. — [Cross-platform development](https://create.roblox.com/docs/projects/cross-platform)
- A Roblox staff post says TextSize 20 is meant to be about 1/8 inch tall on a roughly 160 dpi normalization, and smaller sizes are unlikely to give a good experience. — [DevForum: UI scaling best practices](https://devforum.roblox.com/t/specific-details-about-ui-scaling-best-practices/3888004)

**Flex layouts and CanvasGroup**
- "UIFlexLayout" is not a separate class. Flex lives in `UIListLayout` and `UIFlexItem`. — [List and flex layouts](https://create.roblox.com/docs/ui/list-flex-layouts)
  - `UIListLayout` gained `HorizontalFlex`/`VerticalFlex`, `ItemLineAlignment` and `Wraps`.
  - `UIFlexItem.FlexMode` can be Fill, Grow, Shrink or Custom.
  - Flex "adds a slight performance cost… especially when the layout resizes or flex items are added or removed dynamically". Use it only when it serves a purpose.
- **CanvasGroup**:
  - Flattens its descendants into one texture, and only when the ancestor's `ZIndexBehavior` is Sibling.
  - Uses extra texture memory, with quality and total memory capped by the client's QualityLevel. "When exceeding the memory cap, CanvasGroup will render as a blank texture."
  - Keep sizes static, because resizing creates a new texture.
  — [CanvasGroup reference (md)](https://create.roblox.com/docs/reference/engine/classes/CanvasGroup.md)

**Upcoming platform change**
- Rotated GuiObjects and `Path2D` will be clipped by their parents. It is opt-in via `StarterGui.ClipsDescendantsSupportsRotation`, became the default in **August 2026**, and the property is **removed in November 2026**. — [Weekly Recap 13-17 Jul 2026](https://devforum.roblox.com/t/weekly-recap-july-13-%E2%80%93-17-animation-graphs-go-live-5%C3%97-more-data-store-storage/4743286)

**TopbarPlus (1ForeverHD)**
- Latest is **v3.5.0**, published 19 Oct 2026 (release page) / 20 Oct 2026 (Atom timestamp). It brings:
  - appearance matching Roblox's Core UI;
  - dropdowns that close on outside clicks;
  - four new icon methods;
  - Reduce Motion support;
  - right-aligned icons that move aside for the health bar;
  - icon spacing that matches Roblox "on every device".
- Earlier releases: v3.4.0 (17-18 Sep 2025); v3.3.0 (22 Jul 2025) with dropdown tweens and better PreferredInput handling; v3.2.5 (Jun 2025), which handled Roblox changes that broke console support.
- The Wally index lists 3.5.0.
— [TopbarPlus releases](https://github.com/1ForeverHD/TopbarPlus/releases), [releases Atom](https://github.com/1ForeverHD/TopbarPlus/releases.atom), [Wally index](https://raw.githubusercontent.com/UpliftGames/wally-index/main/1foreverhd/topbarplus)

### Inferences
- **Keep vide.** It is actively maintained in 2026 and the developer knows it. Its Solid-style fine-grained reactivity updates only the properties whose sources changed, which should mean fewer allocations and less per-update work than React-lua's reconciliation on low-end phones. This is an inference; no benchmark was found.
  - Upgrade to 0.4.1 and migrate to `create(class)(props)`.
  - Pick up the next release for the Oct 2026 strict-mode and priority-order fixes, or pin a main commit.
- **Don't start a new project on React-lua (jsdotlua).** It has had no release or commit for about 23 months.
- **Fusion 0.3 is viable but stagnant on releases**, and switching offers nothing over vide.
- **Plain Instances (or direct property writes) for hot paths:**
  - Per-frame HUD values such as the growth/XP bar, boost meter, minimap blips and damage numbers should be written directly or throttled to about 10-15 Hz rather than pushed through reactive graphs every frame.
  - Use vide for menus, shop, shark-select and settings.
- **Mobile UI checklist:**
  - HUD ScreenGuis use `CoreUISafeInsets`.
  - Full-bleed backgrounds or vignettes use a separate ScreenGui with `DeviceSafeInsets` or `None`.
  - Set `SafeAreaCompatibility = None` on new UI.
  - Minimum touch target: 9 mm ≈ 0.354 in, which at Roblox's ~160 dpi normalization is about **56-57 px**. This combines the two sources above.
  - Primary buttons such as bite and boost should be larger, and kept clear of the thumbstick area.
  - Body text around TextSize 18-20+, with `UITextSizeConstraint` MinTextSize at 9 or above (better 12+).
  - Use `UIScale` and `UIAspectRatioConstraint` for scaling.
- **CanvasGroup:** avoid it on large, frequently resized or always-visible panels, because it renders blank when the low-end memory cap is exceeded. Use it only for short fades of small, fixed-size panels, or tween individual transparencies instead.
- **Rotated elements:** audit rotated elements inside clipping frames, such as compass needles or rotated fin icons, before the November 2026 property removal.
- **TopbarPlus:** upgrade 3.4.0 to 3.5.0 for the current Core-UI look and health-bar spacing. For one or two simple topbar buttons, a ScreenGui with `ScreenInsets = TopbarSafeInsets` is a dependency-free alternative, but you lose TopbarPlus's overflow, dropdown and console handling.

### Gaps
- No measured mobile benchmark (frame time or memory) of vide vs Fusion vs React-lua was found.
- The meaning and default of `ScreenGui.ClipToDeviceSafeArea`, and the current status of `IgnoreGuiInset`, were not documented on the pages fetched.
- No official Roblox pixel minimum for touch targets exists; only the 9 mm WCAG reference was found.
- Whether CanvasGroup memory caps differ per device tier in numbers is not documented.

---

## 3. Animation: crunchyroll vs Roblox's Animator/AnimationController (and Animation Graphs) for non-humanoid creatures

### Takeaway
crunchyroll is a niche Luau keyframe solver that computes Motor6D-style rig CFrames without an Animator. Its examples are R6, and it says nothing about Bones or skinned meshes.

For skinned-mesh sharks, use Roblox's own Animator under an AnimationController. **Animation Graphs**, fully released in July 2026, handle blended swim, boost and bite states with automatic parameter replication. The hundreds of AI fish should get cheap client-side procedural or LOD'd animation, not one Animator each.

### Cited Findings
- **What crunchyroll is:**
  - "A dedicated library for calculating coordinate frames of a rig from an array of animation tracks". `create_rig` represents Roblox Motor6Ds, `load_keyframe_sequence` loads Roblox animations, and `solve_animation` writes per-limb CFrames into `rig.result_coordinate_frames`.
  - The demo is captioned "no Animator!" and uses an R6 rig.
  - Stated benefits: no instances are created, which helps ping compensation and "secure headshot detection" and suits viewport frames. With your own player you get numerical priorities, instant stops and animation skipping.
  - Bones and skinned meshes are not mentioned.
  - 98 stars, MIT license.
  — [crunchyroll README](https://github.com/ffrostfall/crunchyroll)
- **crunchyroll versions:**
  - GitHub releases: 0.0.0 and 0.1.0 (22 May 2025), and 0.1.1 (4 Jul 2025), which switched to quaternion normalized interpolation and binary keyframe search.
  - Wally lists 0.0.0-a, 0.0.0-b, 0.1.0 and **0.2.0**. 0.2.0 is not a GitHub release; it was published around 17 Nov 2025 ("update changelog, update wally" commit).
  - 0.2.0 changes: constant easing, keyframe holes, Weight = 0 nodes no longer animated, `math.lerp`/`vector.lerp`, exposed `limb_transforms`, and a binary-search fix.
  - Unreleased on main (13 Dec 2025): `skip_result_coordinate_frames`, "flip quat if it's the shortest path (#15)" and "fix edge case for keyframe holes (#14)". No commits since then.
  — [CHANGELOG](https://raw.githubusercontent.com/ffrostfall/crunchyroll/main/CHANGELOG.md), [commits](https://github.com/ffrostfall/crunchyroll/commits/main), [releases Atom](https://github.com/ffrostfall/crunchyroll/releases.atom), [Wally index](https://raw.githubusercontent.com/UpliftGames/wally-index/main/ffrostfall/crunchyroll)
- **Animation Graphs, full release July 2026:**
  - A visual editor (Avatar tab) for building and blending motion. Graphs "run live in active games and replicate automatically", with 280+ bug fixes since beta.
  - The July 2026 recap also announced a new **Client CPU Time** graph (Monitoring > Performance > Client) that breaks client CPU into Scripts, Networking, Physics, **Animation** and Miscellaneous.
  — [Weekly Recap 13-17 Jul 2026](https://devforum.roblox.com/t/weekly-recap-july-13-%E2%80%93-17-animation-graphs-go-live-5%C3%97-more-data-store-storage/4743286)
- **Animation Graph scripting** — [Animation Graph Editor docs](https://create.roblox.com/docs/animation/graph-editor)
  - Selecting a rig creates an `AnimationGraphDefinition` asset.
  - To play a graph, publish it and set `Animation.AnimationId` to the graph ID. Then call `Animator:LoadAnimation()` and `AnimationTrack:Play()`.
  - Drive it with `AnimationTrack:SetParameter(name, value)` and listen with `GetMarkerReachedSignal`.
  - **Automatic** replication mode: player graphs run on the owning client and NPC graphs run on the server. Parameters replicate, multiple SetParameter calls in one frame are coalesced, and other clients see updates with a small delay.
  - **Server** mode: the server simulates, and clients predict and reconcile.
  - Only Linear and CubicV2 transition easing is supported.
  - The docs page says nothing about AnimationController, non-humanoid rigs or Bones.
- State-machine transitions default to a 0.15 s blend. The reference graph uses a `Blend1DNode` on a Speed parameter for idle/run. — [Graph Editor state machines](https://create.roblox.com/docs/animation/graph-editor-state-machine)
- A third-party blog claims a Roblox staffer said AnimationControllers can play graphs. This is unverified, low-quality sourcing. — [gamertagmythras blog](https://gamertagmythras.com/blog/roblox/roblox-animation-graph-editor-guide)
- **AnimationController** is "an object which allows animations to be loaded and applied to a character or model in place of a Humanoid". You create an Animator under it and load via the Animator. `AnimationController:LoadAnimation` and related members are marked obsolete in favor of Animator. — [AnimationController reference](https://create.roblox.com/docs/reference/engine/classes/AnimationController)
- **Skinned mesh and bone costs (forum anecdotes, 2020-2023; older, may be outdated):**
  - Skinning added roughly 10-15% GPU overhead depending on platform. CPU cost depends on how much of the mesh is skinned to bones. — [Skinned MeshParts are live, p2](https://devforum.roblox.com/t/skinned-meshparts-are-live/831011?page=2)
  - In one mobile case, about 30 skinned NPCs lagged on a Pixel 7 Pro. RenderFidelity changes did nothing. The real culprits were per-NPC network-owner scripts and default animation scripts; without them, hundreds of NPCs caused no noticeable lag. — [DevForum 2649641](https://devforum.roblox.com/t/game-with-skinned-meshes-has-poor-performance-on-mobile-renderfidelity-doing-nothing/2649641)
  - Updating many bones individually is expensive. — [reply 628](https://devforum.roblox.com/t/skinned-meshparts-are-live/831011/628), [DevForum 2528802](https://devforum.roblox.com/t/2528802)

### Inferences
- **Player sharks (10-20 per server):**
  - Use a skinned MeshPart rig with a handful of spine, tail, jaw and fin bones, under an AnimationController with an Animator.
  - Build one Animation Graph: Blend1D on swim speed (idle, cruise, dash), a state machine or priority layer for bite, boost and hurt, and marker events for bite timing.
  - Automatic mode runs player graphs on the owning client, so animation stays responsive on bad networks while parameters replicate to others. Verify AnimationController and Bone support for graphs in a Studio prototype first, because the official docs don't confirm it.
- **Hundreds of AI fish:**
  - Don't create one Animator or AnimationTrack per fish.
  - Animate fish procedurally on the client: a shared-clock sine tail sway on one or two bones, or a whole-model CFrame wiggle, applied only within a few tens of studs of the camera.
  - Fish beyond that distance get no animation, or a cheaper non-skinned LOD mesh.
  - Avoid per-fish server scripts and network-ownership churn, which the forum evidence identifies as the larger mobile cost.
  - Validate with the new Client CPU Time "Animation" category.
- **AI predators (few):** an Animator is acceptable. Note that in Automatic mode NPC graphs run on the server and replicate, so with many predators consider client-side visual models (networking is out of scope).
- **crunchyroll is not a good fit here.** It targets Motor6D-style rigs, has no stated bone support, has been inactive since Dec 2025, and the Wally 0.2.0 build lacks two later correctness fixes (quaternion shortest-path, keyframe holes).
  - Its unique value is server-side pose solving for lag-compensated hit detection. A shark's bite hitbox can be a sphere or capsule on a jaw Attachment, so that value isn't needed.

### Gaps
- No official 2026 confirmation was found that Animation Graphs drive AnimationController-based, Bone-skinned rigs, or of their per-instance runtime cost on mobile.
- No official mobile budget for bones or skinned MeshParts was found. The forum figures are 2020-2023 anecdotes.
- Animator distance-based throttling and LOD settings were not researched.
- No other new third-party Luau animation libraries from 2025-2026 were found beyond crunchyroll. littensy/ripple (springs) is UI and tween-oriented; see Section 4.

---

## 4. State and utilities: Charm/charm-sync, signals, cleanup, Promise/Future, runtime type checks, rate limiting

### Takeaway
Actively maintained in 2026:
- **Charm 0.11.1 + charm-sync 0.4.1** (1 Oct 2026)
- **LemonSignal 2.0.0** (13 Jul 2026)
- **Sleitnick RbxUtil** (Trove 1.8.0, Signal 2.0.3; repo active Jul 2026)
- littensy **ripple 0.10.2** (18 Jul 2026)

Stable but dormant: evaera Promise 4.0.0 (last commit Oct 2023), `t` 3.1.1 (last commit Mar 2025), Janitor 1.18.3 (last commit Oct 2025).

**red-blox/Util (Signal, Promise, Ratelimit, Guard) was archived on 4 Mar 2025.**

### Cited Findings
- **Charm (littensy):**
  - **v0.11.1 (1 Oct 2026):** Luau type fixes. charm-sync changes: `addSignalsToClient` runs outside the tracking scope, global patch list order is preserved when patches are nil, and a warning fires when a client syncs an unregistered signal.
  - **v0.11.0 (21 Jun 2026):** reactivity switched to **alien-signals**. New `signal(initial)` returns a getter and a setter. Added `effectScope`, `onCleanup`, `listen`, `untracked` and `trigger`. Breaking: `peek()` became `untracked()`, and nested effects and effects inside `observe()` now auto-dispose.
  - **rc.6 (6 May 2026):** effect callbacks run in `batched()` (breaking), plus `mapped()` cleanup fixes and computed/observe performance work.
  - Release candidates rc.1 to rc.3 came out Nov to Dec 2025. v0.10.0 came out 9 Mar 2025.
  - Earlier sync features: `serializeArrays`/`autoSerialize` and `flatten()` (v0.9) and the refined `autoSerialize` patch type (v0.10).
  — [Charm releases](https://github.com/littensy/charm/releases), [releases Atom](https://github.com/littensy/charm/releases.atom)
- Wally: `littensy/charm` latest 0.11.1; `littensy/charm-sync` latest **0.4.1** (preceded by 0.4.0 and rc.2 to rc.7). — [Wally index charm](https://raw.githubusercontent.com/UpliftGames/wally-index/main/littensy/charm), [Wally index charm-sync](https://raw.githubusercontent.com/UpliftGames/wally-index/main/littensy/charm-sync)
- **LemonSignal:**
  - **v2.0.0 (13 Jul 2026):** "Variadics at connect time have been removed in favor of proper types". This is breaking for code like `sig:Connect(fn, extraArg)`.
  - v1.11.0 (6 Aug 2025) fixed variadic args being dropped when the first argument is false or nil.
  - Last commit 13 Jul 2026. Wally latest 2.0.0.
  — [LemonSignal releases](https://github.com/data-oriented-house/LemonSignal/releases), [releases Atom](https://github.com/data-oriented-house/LemonSignal/releases.atom), [Wally index](https://raw.githubusercontent.com/UpliftGames/wally-index/main/data-oriented-house/lemonsignal)
- **Sleitnick RbxUtil:**
  - Wally `sleitnick/trove` latest **1.8.0** and `sleitnick/signal` latest **2.0.3**.
  - Repo commits: "Stream library (#259)" (11 Nov 2025), "Fix vector writes (#260)" (12 Nov 2025), "Better types for TypedRemote" (27 Jul 2026).
  — [Wally index trove](https://raw.githubusercontent.com/UpliftGames/wally-index/main/sleitnick/trove), [Wally index signal](https://raw.githubusercontent.com/UpliftGames/wally-index/main/sleitnick/signal), [RbxUtil commits](https://github.com/Sleitnick/RbxUtil/commits/main)
- **Janitor (howmanysmall):** Wally latest 1.18.3. Last GitHub release 12 Aug 2024; last commit 19 Oct 2025. — [Wally index](https://raw.githubusercontent.com/UpliftGames/wally-index/main/howmanysmall/janitor), [Janitor repo](https://github.com/howmanysmall/Janitor)
- **evaera Promise:** Wally `evaera/promise` latest 4.0.0. Last GitHub release 3 Mar 2022; last commit 16 Oct 2023. Lyra itself depends on `evaera/promise@4.0.0` and `osyrisrblx/t@3.1.1`. — [Wally index](https://raw.githubusercontent.com/UpliftGames/wally-index/main/evaera/promise), [repo](https://github.com/evaera/roblox-lua-promise), [Lyra wally.toml](https://raw.githubusercontent.com/paradoxum-games/lyra/main/wally.toml)
- **t (osyrisrblx), runtime type checks:** Wally latest 3.1.1. Last GitHub release Dec 2021; last commit 11 Mar 2025. — [Wally index](https://raw.githubusercontent.com/UpliftGames/wally-index/main/osyrisrblx/t), [t repo](https://github.com/osyrisrblx/t)
- **red-blox/Util:**
  - "This repository was archived by the owner on Mar 4, 2025. It is now read-only." Last commit 25 Jul 2024.
  - README: "a collection of smaller utility modules… all single file modules".
  - Final Wally versions: `red-blox/signal` 2.0.2, `red-blox/promise` 1.1.1, `red-blox/ratelimit` 1.0.1, `red-blox/guard` 1.0.1.
  — [red-blox/Util](https://github.com/red-blox/Util), [Wally index signal](https://raw.githubusercontent.com/UpliftGames/wally-index/main/red-blox/signal), [ratelimit](https://raw.githubusercontent.com/UpliftGames/wally-index/main/red-blox/ratelimit), [guard](https://raw.githubusercontent.com/UpliftGames/wally-index/main/red-blox/guard), [promise](https://raw.githubusercontent.com/UpliftGames/wally-index/main/red-blox/promise)
- **ripple (littensy, springs/motion):** releases 0.10.0 (12 Dec 2025), 0.10.1 (26 Feb 2026) and v0.10.2 (18 Jul 2026). Wally latest 0.10.2. — [ripple releases Atom](https://github.com/littensy/ripple/releases.atom), [Wally index](https://raw.githubusercontent.com/UpliftGames/wally-index/main/littensy/ripple)

### Inferences
- **State:**
  - Client state (currency, size/XP, unlocked sharks, settings, shop state) lives in Charm 0.11.1 atoms.
  - Server-authoritative player state replicates through charm-sync 0.4.1, which sends patches rather than full snapshots. That is good for poor mobile bandwidth, but tune the sync interval.
  - Bridge Charm into vide UI via subscriptions or effects that set vide sources, or skip Charm for UI-only state that vide sources already handle.
  - Mind the 0.11 breaking changes (`peek` to `untracked`, auto-disposing nested effects, batched effects).
- **Signals:** LemonSignal 2.0.0 (typed, actively maintained) or `sleitnick/signal` 2.0.3. Either replaces the archived red-blox Signal. Code relying on connect-time variadics must be rewritten for LemonSignal 2.x.
- **Cleanup:** Trove 1.8.0, from the actively maintained RbxUtil. Inside UI and state code, prefer vide's `cleanup()` and Charm's `effectScope`/`onCleanup` so lifetimes follow scopes.
- **Promise/Future:** evaera Promise 4.0.0 is stable, ubiquitous and already a Lyra dependency, but dormant since 2023. For most game code, `task.spawn`/`task.defer` and coroutines plus ProfileStore's yielding API are simpler and allocate less on mobile.
- **Runtime checks:** `t` 3.1.1 for validating RemoteEvent arguments on the server (shark ID, upgrade ID, purchase-intent payloads).
- **Rate limiting:** red-blox Ratelimit is frozen. A roughly 20-line per-player token bucket keyed by UserId and action works, or use the networking library's limiter (other researcher's scope).
- **Archived ≠ broken.** The red-blox modules are tiny single files, so vendoring them is low-risk. A new project should still prefer maintained equivalents to avoid Luau type-solver drift.

### Gaps
- GoodSignal (stravant) was not checked for 2026 status.
- red-blox "Future" was not checked; the archived repo's package list wasn't visible.
- Release dates for Trove 1.8.0, Signal 2.0.3 and Janitor 1.18.3 on Wally could not be read (the index has no dates).
- No benchmarks of the signal libraries were found, and charm-sync's wire format and bandwidth were not measured.

---

## 5. Monetization and analytics: passes, products, receipts, subscriptions, Roblox Plus, ads, managed pricing, AnalyticsService

### Takeaway
Use these monetization tools:
- game passes for permanent perks and cosmetics;
- developer products for currency, revives and boosts, processed idempotently against the session-locked profile;
- **Managed pricing** (regional pricing plus price optimization), with prices always shown via `GetProductInfoAsync`;
- optionally a subscription, Roblox Plus sign-up prompts and rewarded video ads.

In 2026 these were added or changed: Roblox Plus (launched 30 Apr 2026, with creator sign-up bonuses and Roblox-funded discounts), Premium sales ended, cross-game developer-product sales were disabled from 30 May 2026, and a new `BindReceiptHandler` API appeared.

Instrument economy, funnel and custom events within the analytics limits: 120 + 20 × CCU requests/min, 3 custom fields, 100 custom event names.

### Cited Findings

**Receipt processing**
- **`ProcessReceipt` best practice**, from the developer products docs:
  - Validate each receipt and grant only when its status is Open.
  - Return `PurchaseGranted` only after granting. Return `NotProcessedYet` otherwise, and the system "tries again the next time the user joins".
  - Only one server script may set the callback.
  - Don't grant from `PromptProductPurchaseFinished`.
  - Roblox keeps no per-user purchase history, so store it yourself.
  - `receiptInfo` contains the exact Robux paid, even during price tests.
  — [Developer products](https://create.roblox.com/docs/production/monetization/developer-products)
- The `MarketplaceService.ProcessReceipt` sample:
  - records granted `PurchaseId`s with `UpdateAsync` to prevent double grants;
  - returns NotProcessedYet on DataStore errors;
  - skips players who aren't in the server, since the callback reruns on rejoin;
  - warns grant handlers not to yield inside `UpdateAsync`;
  - "does not completely mitigate cross-server data failure scenarios".
  — [MarketplaceService reference](https://create.roblox.com/docs/reference/engine/classes/MarketplaceService#ProcessReceipt)
- Other MarketplaceService APIs on the same page. — [MarketplaceService reference](https://create.roblox.com/docs/reference/engine/classes/MarketplaceService#ProcessReceipt)
  - `BindReceiptHandler(...)` binds handlers by `Enum.ReceiptType` (DeveloperProduct, RobuxTransferSender, RobuxTransferReceiver) and returns `Enum.ReceiptDecision`.
  - `GetProductInfoAsync` returns `PriceInRobux`, `UserBasePriceInRobux`, `PriceDiscountDetails` and `TimedOptions`.
  - `RankProductsAsync` and `RecommendTopProductsAsync` handle personalized store ordering.
  - `GetUsersPriceLevelsAsync` and `PromptBulkPurchase` are also available.
  - `PromptRobuxTransferAsync` starts Robux transfers.
  - Subscription APIs: `GetUserSubscriptionStatusAsync`, `GetRobloxSubscriptionDetailsAsync`, `GetUserSubscriptionPaymentHistoryAsync`, `PromptSubscriptionPurchase`.
- Developer-product notes. — [Developer products](https://create.roblox.com/docs/production/monetization/developer-products)
  - **Cross-game developer product sales disabled starting 30 May 2026**; integrate Robux transfers instead.
  - For external sales, a product must be Listed and have a correct ProcessReceipt, which Roblox checks continuously.
  - Price range is 1 to 1 billion Robux. Icons are at most 512×512.
  - `RankProductsAsync` is strictly rate-limited, so call it once at join.
  - `RecommendTopProductsAsync` returns up to 50 items and needs at least one sale in the last 28 days.
  - The analytics tab charts up to 8 top products.

**ProfileStore support for receipts**
- `Profile:Save()` for critical moments such as developer-product purchases, and `Profile.LastSavedData` for "securely handling developer product purchases". — [ProfileStore API](https://madstudioroblox.github.io/ProfileStore/api/)

**Pricing**
- **Managed pricing** "brings regional pricing and price optimization together", is opt-in once, and keeps prices optimized across products and regions. Price optimization finds price points for passes and developer products. — [Monetization overview](https://create.roblox.com/docs/production/monetization)
- Regional pricing:
  - Launched for in-experience items in April 2025. — [Roblox newsroom, Apr 2025](https://corp.roblox.com/newsroom/2025/04/roblox-launches-regional-pricing-for-in-experience-items)
  - Extended to developer products. Games with gifting or trading must use the User Price Level API before opting in. — [Introducing Regional Pricing for Developer Products](https://devforum.roblox.com/t/introducing-regional-pricing-for-developer-products/3971235)
  - A third-party claim that passes default to regional pricing from 30 Mar 2026 is **unverified**. — [creation.dev guide](https://www.creation.dev/learn/roblox-regional-pricing-game-passes-guide)

**Subscriptions (current docs)** — [Subscriptions](https://create.roblox.com/docs/production/monetization/subscriptions)
- **Robux-priced:** at least 49 Robux, with regional pricing on by default. Pays 70% monthly with about a 5-day hold.
- **Local-currency:** tiers of $2.99, $4.99, $7.99, $9.99 and $14.99. Pays 70% in month 1, then **100%** from month 2, held 30 days.
  - Example: a $5 subscription pays 350 Robux in month 1, then 500 Robux a month.
  - Requires ID or phone verification, and is unavailable in 12 countries.
- Up to 50 subscriptions per game. Types: Durable, Consumable, Currency.
- The Robux price can change once every 60 days, with 30 days' notice for increases.
- APIs: `GetUserSubscriptionStatusAsync` (server), `PromptSubscriptionPurchase` (client), `Players.UserSubscriptionStatusChanged`.
- Cloud webhooks fire for purchased, renewed, cancelled and refunded.

**Roblox Plus**
- Announced 10 Apr 2026, launched 30 Apr 2026 at $4.99/month. — [Introducing Roblox Plus (DevForum)](https://devforum.roblox.com/t/introducing-roblox-plus/4567894/1), [press release](https://secure.businesswire.com/news/home/20260410099173/en/Roblox-Announces-Roblox-Plus-a-New-Subscription-Plan-Offering-Exclusive-Platform-Benefits)
  - Roblox funds a 10% discount on in-game items (20% from the third month). Creators keep their revenue share, and Roblox recommends displaying prices via `GetProductInfoAsync`, not hard-coding them.
  - Creators earn **250 Robux/month for a new subscriber's first 3 consecutive months (up to 750)** when sign-ups come through `PromptRobloxSubscriptionPurchase`.
  - Paid private servers: a server qualifies when a subscriber spends 60+ minutes in it over 30 days. Each subscriber's top 5 qualifying servers count, at up to 100 Robux per subscriber per server.
  - Premium is no longer sold to new subscribers. On 30 May 2026 Premium's 10% Robux bonus and logo ended.
  - Earnings appear in Creator Hub > Monetization > Roblox Plus.
- From **24 Sep 2026**, Plus members get one free private server per game instead of unlimited. — [Weekly Recap 13-17 Jul 2026](https://devforum.roblox.com/t/weekly-recap-july-13-%E2%80%93-17-animation-graphs-go-live-5%C3%97-more-data-store-storage/4743286)

**Rewarded video ads**
- How they work (search summary of the official docs): check availability with `AdService:GetAdAvailabilityNowAsync`, build the reward with `CreateAdRewardFromDevProductId`, and show it with `ShowRewardedVideoAdAsync`. The server grants the reward after the full video, and optional Placement IDs give per-placement metrics. — [Rewarded video ads docs](https://create.roblox.com/docs/production/promotion/rewarded-video-ads)
- Eligibility was widened in Aug 2025. — [More Creators Can Now Use Rewarded Video Ads](https://devforum.roblox.com/t/more-creators-can-now-use-rewarded-video-ads/3838678)
- A Studio plug-in with templates, a pre-gameplay surface and a multi-reward option were teased at RDC for Q4 2026. — [MediaPost](https://www.mediapost.com/publications/article/417959/roblox-eyes-expanded-placements-for-rewarded-vid.html)
- The monetization overview also lists immersive ads, paid access (Robux or local currency) and private servers. — [Monetization overview](https://create.roblox.com/docs/production/monetization)

**AnalyticsService**
- **Custom events** — [Custom events](https://create.roblox.com/docs/production/analytics/custom-events)
  - `LogCustomEvent(player, name, value?)` is server-only and works only in published games.
  - Up to **100** custom event names. Put variation into custom fields, because event names have "a much tighter cardinality limit".
  - Data is aggregated daily, so charts take up to 24 h.
  - Results show on the **Explore** page with count, unique users, sum, average, min, max and average per user.
  - Batch values (one event with value 10 instead of ten events with value 1).
- **Limits** (from create.roblox.com excerpts via search; pages not fully fetched) — [AnalyticsService](https://create.roblox.com/docs/reference/engine/classes/AnalyticsService), [Custom fields](https://create.roblox.com/docs/production/analytics/custom-fields), [Event types](https://create.roblox.com/docs/production/analytics/event-types)
  - Global rate limit of **120 + 20 × CCU** AnalyticsService requests per minute.
  - **3 custom fields** per economy, funnel or custom event. Values must be strings.
  - **8,000** unique combinations across the custom fields; beyond that, values become "Other".
  - Events roll off **90 days** after the last data.
  - An economy limit of 10 resource types. This conflicts with the economy page's "up to five currencies" in the dashboard; see below.
- **Economy events** — [Economy events](https://create.roblox.com/docs/production/analytics/economy-events)
  - Signature: `LogEconomyEvent(player, flowType Source/Sink, currency, amount (positive), endingBalance, transactionType, itemSku?)`.
  - Default transaction types: IAP, TimedReward and Onboarding are sources; Shop and Gameplay can be either; ContextualPurchase is a sink. Custom types are allowed.
  - The dashboard supports up to **five currencies** and filtering by up to 3 custom fields.
  - Metrics: sources vs sinks, average wallet balance (all users, payers, non-payers), top sources and sinks.
- **Funnel events** — [Funnel events](https://create.roblox.com/docs/production/analytics/funnel-events)
  - `LogOnboardingFunnelStepEvent(player, step, stepName?)` and `LogFunnelStepEvent(player, funnelName, funnelSessionId, step, stepName)`.
  - Up to **10** funnels. Steps expire after 180 days.
  - The 10 most recent session IDs per user per funnel are tracked.
  - Repeated steps are ignored by the funnel but still count against the rate limit. Logging a later step auto-completes earlier ones.
  - Recommended funnels: onboarding (start here), shop (GUID session ID), progression, and item upgrades (session ID `<playerId>-<itemId>`).

### Inferences
**Recommended receipt pattern with session-locked data** (synthesis of the Roblox docs and ProfileStore API above):
1. Set `ProcessReceipt` in exactly one server module.
2. If the player isn't in the server, or their profile isn't loaded or active yet, return `NotProcessedYet`.
3. If `receipt.PurchaseId` is already in `profile.Data.RecentPurchaseIds`, return `PurchaseGranted`. Keep this as a bounded ring buffer of about the last 50-100 IDs.
4. Otherwise apply the grant and append the PurchaseId in the same synchronous mutation, with no yield in between.
5. Call `Profile:Save()` and confirm the PurchaseId appears in `Profile.LastSavedData` (or the save completes) before returning `PurchaseGranted`. If the save fails, return `NotProcessedYet`.
   - A retry in the same session is idempotent because the ID is already in memory.
   - A crash before the save loses both the grant and the ID, so the retry grants exactly once.
- Use `BindReceiptHandler` only after confirming whether it supersedes `ProcessReceipt` (see Gaps).

**Product plan for a grow-and-evolve shark game:**
- **Passes:** cosmetic shark skins and trails, an extra loadout slot, a VIP chat tag. Avoid strong PvP-advantage passes (e.g. permanent +damage), which hurt retention in mixed PvE/PvP lobbies.
- **Developer products:** currency packs (one "Gems" premium currency plus earned "Coins"), "revive at current size" (also a rewarded-ad reward), and timed growth boosts.
- **Subscription (optional):** a monthly "Reef Pass" of the Currency or Durable type for a daily gem stipend and a cosmetic. The local-currency version pays 100% from month 2.
- **Roblox Plus:** add a `PromptRobloxSubscriptionPurchase` entry point (e.g. in the shop) for 250 Robux/month bonuses per new subscriber.
- **Paid private servers:** these now earn Plus payouts.
- **Managed pricing:** opt passes and products in, and always render prices from `GetProductInfoAsync`, which is needed for regional prices and the Plus discount.

**Analytics plan:**
- Economy events for Coins and Gems with itemSku = product or upgrade ID.
- Onboarding funnel: spawned → first fish eaten → first size-up → first evolution/tier-up → first shark purchased → opened shop.
- Recurring funnels:
  - shop (GUID per visit);
  - per-shark progression (`<userId>-<sharkId>`);
  - death and revive (died → saw revive offer → revived).
- Custom events with fields such as `DeathCause` (predator type, player, starvation), `Biome` and `SharkTier`.
- **Never log per fish eaten.** With hundreds of prey and the 120 + 20 × CCU/min limit, aggregate per player per minute or per life (e.g. a `FishEatenPerLife` value).
- Watch the Developer Products analytics tab, the Roblox Plus earnings tab and the new Client CPU Time performance graph.

### Gaps
- Whether `BindReceiptHandler` replaces or deprecates `ProcessReceipt` could not be confirmed. The exact retry cadence of ProcessReceipt within a session is also undocumented on the pages fetched.
- The formal `customFields` parameter shape for the Log* methods was not captured.
- The conflict between "10 resource types" and "five currencies" for economy events is unresolved.
- Exact rewarded-video eligibility thresholds and current eCPMs: third-party figures ($5-15 or $6-12 CPM) are unverified.
- Creator Rewards (engagement payouts) after Premium's sunset were not covered by the pages fetched.
- Whether regional pricing is now on by default for passes is unverified.

---

## 6. IP risk: Roblox IP policy, how takedowns work, and what an original shark-genre game must avoid

### Takeaway
The genre and its mechanics (a shark that eats, grows and unlocks bigger sharks) are ideas, and Roblox's own guidance says copyright doesn't protect ideas. The franchise's specific expression is protected: its art, UI, characters, levels, audio and text. So is the "Hungry Shark" trademark.

Ubisoft actively owns and supports the franchise:
- Evolution and World remain supported as of 2025.
- An animated series is in development.

Ubisoft could file copyright or trademark notices through Roblox's IP channels, and those can remove whole experiences. Repeat or egregious infringers can lose their accounts.

Use an original, distinct name. The working title "Shark Evolution" closely echoes "Hungry Shark Evolution". Also use original branding, characters, UI and maps.

### Cited Findings
**Roblox IP guidelines** — [IP guidelines](https://create.roblox.com/docs/production/publishing/ip-guidelines)
- Roblox prohibits unauthorized IP use and removes violating content.
- Accounts that "repeatedly or egregiously infringe" may be suspended or terminated.
- "Copyright does not protect facts or ideas, but it may protect the specific expression of a fact or idea."
- Infringement can occur even without monetization, with credit given, after buying a copy, or when modified, "including by AI tools".
- A trademark is "a name, logo, or design that identifies the source of goods or services".

**How notices and takedowns work** — [IP guidelines](https://create.roblox.com/docs/production/publishing/ip-guidelines)
- **Copyright notices** go to copyright_agent@roblox.com, by mail or phone, or through the IP Platform (18+). They require a description of the work, the infringing URL, and sworn good-faith statements. Valid notices lead to quick removal plus notification of the user.
- **Counter-notices:** content is generally restored "within 14 business days" unless the reporter shows a lawsuit has been filed.
- **Trademark notices** require the registration country and number. They go to trademark_agent@roblox.com or the IP Platform. Appeals go by email.
- False notices can create DMCA §512(f) liability. Abuse can bring processing delays, tooling restrictions and a lower reporter standing.
- EU DSA IP reporting is handled separately.

**Rights Manager and IP licensing**
- Registered rights holders or their representatives can file removal requests for **assets, games and avatar items** under Intellectual Property > Removal Requests. Requests show as Pending, Approved or Rejected. — [Rights Manager docs](https://create.roblox.com/docs/production/publishing/rights-manager), [Introducing Rights Manager (2024)](https://devforum.roblox.com/t/introducing-rights-manager/2880717)
- Forum reports say reviews sometimes take longer than 2 weeks (anecdotal, 2025). — [DevForum](https://devforum.roblox.com/t/right-manager-dmcas-taking-longer-than-2-weeks/3744199)
- Roblox also runs an IP-licensing program for rights holders, a possible official route to licensed IP. — [IP licensing: get started](https://create.roblox.com/docs/ip-licensing/get-started)

**The Hungry Shark franchise**
- History and gameplay. — [Wikipedia: Hungry Shark](https://en.wikipedia.org/wiki/Hungry_Shark) (last edited 2 Jun 2026)
  - Created by Future Games of London, acquired by Ubisoft. Ubisoft has published since *Hungry Shark Evolution* (2012; Ubisoft from 2013).
  - Titles include *Hungry Shark World* (2016) and *Hungry Shark Arena* (2020, online multiplayer).
  - The franchise's gameplay: real shark species such as mako, great white, hammerhead and megalodon; eat prey to earn points, grow and unlock stronger sharks; health that drains over time; hazards such as helicopters dropping explosive barrels and fishermen; abilities such as freezing breath or boost; a gold and gems freemium economy.
  - Ubisoft London closed in 2023. On 27 Jan 2025 Ubisoft ended *Hungry Shark Primal* to focus on supporting *Evolution* and *World*.
  - The animated series *Hungry Shark Squad* is in development at Ubisoft Motion Pictures.
- Trademark notices:
  - The 2013 acquisition notice states "Hungry Shark is a trademark of Future Games of London" (a Ubisoft subsidiary). — [AMF filing (2013)](https://echanges.dila.gouv.fr/OPENDATA/AMF/CNS/2013/10/FCCNS120680_20131001.pdf), [MCV coverage](https://www.mcvuk.com/future-games-of-london-acquired-by-ubisoft/)
  - Current store listings use the ® mark ("Hungry Shark® World"). — [Nintendo eShop listing](https://ec.nintendo.com/apps/0100E1A00AF40000/US)
- Ubisoft London's closure followed the loss of Hungry Shark development duties. — [PocketGamer.biz](https://www.pocketgamer.biz/ubisoft-london-studio-to-close-following-loss-of-hungry-shark-development-duties/), [80.lv](https://80.lv/articles/ubisoft-is-closing-its-london-studio-responsible-for-hungry-shark-games/)

### Inferences
These are not legal advice. They apply Roblox's stated copyright/trademark split to this concept.

**Low risk (genre ideas and facts):**
- Eat-to-grow, tiered sharks, real species names (great white, hammerhead, megalodon), an open ocean, prey and predators, boosts, health drain, PvP biting, and dual soft/premium currencies.

**High risk; avoid:**
- **Name and metadata:**
  - Any title using "Hungry Shark" or close variants such as "Hungry Sharks".
  - Titles that read as "[X] Shark Evolution", or that ride the full title. The project's working name "Shark Evolution" is close enough to *Hungry Shark Evolution* that a distinct, ownable title is the safer choice.
  - The franchise name in descriptions, tags, thumbnails or icons, including "like Hungry Shark".
- **Logo and key art:** don't imitate the franchise logo's lettering, color scheme or composition, or the composition and style of its app icon and store art.
- **Distinctive characters:** any franchise-specific sharks that aren't real species (named, fantasy or themed sharks) and their skins, plus signature props or hazards rendered in the franchise's style. The specific character roster wasn't researched; check the franchise's current store listings before naming or designing special sharks.
- **UI:** don't replicate the HUD layout, iconography, menu flow or screens such as shark-select or mission cards.
- **Levels:** no recreation of specific named maps or landmarks.
- **Audio and text:** no ripped music, SFX, mission text or achievement names.
- **AI-generated lookalikes:** these still count, because Roblox treats modified or AI-altered copies as potentially infringing.

**Practical safeguards:**
- Keep an asset-provenance log (who made each mesh, texture and sound, and the license).
- Use an original art direction (silhouettes, palette, UI language).
- Give species-based or original shark names.
- If a notice arrives, respond via the counter-notice or appeal process rather than re-uploading.

### Gaps
- No USPTO, EUIPO or UKIPO registry record was checked. The exact registered marks (whether "Hungry Shark Evolution" is registered as a phrase) and their current owner entity are unconfirmed.
- No evidence was found, in either direction, of Ubisoft filing takedowns against Roblox shark games.
- Roblox's Terms of Use and Community Standards IP sections were not quoted directly. The IP guidelines page links only to the general Terms.

---

## 7. Keep / upgrade / replace verdicts for the developer's current packages

### Takeaway
- Lyra 0.6.0 (with its patch): replace with ProfileStore 1.0.3.
- vide 0.4.0: upgrade to 0.4.1.
- crunchyroll 0.2.0: remove, in favor of Animator, Animation Graphs and procedural fish animation.
- TopbarPlus 3.4.0: upgrade to 3.5.0.
- red-blox Signal, Promise, Ratelimit and Guard (repo archived 4 Mar 2025): replace with LemonSignal 2.0.0 or sleitnick/signal 2.0.3, evaera Promise 4.0.0 (or plain `task`), an in-house token bucket, and `t` 3.1.1.
- Add Charm 0.11.1 + charm-sync 0.4.1 and Trove 1.8.0.

### Cited Findings

| Current choice | Verdict | Target | Key evidence |
|---|---|---|---|
| `paradoxum-games/lyra@0.6.0` + vendored lock-refresh patch | **Replace** (or, second best, pin a reviewed main commit and keep owning the patch) | `lm-loleris/profilestore@1.0.3` | Lyra: no release since 8 Jul 2025; README says "early development"; unreleased fixes on main (orphan-file growth, buffer clearing, DSS locking); nothing upstream on lock refresh — [releases](https://github.com/paradoxum-games/lyra/releases), [README](https://github.com/paradoxum-games/lyra), [PRs](https://github.com/paradoxum-games/lyra/pulls?q=is%3Apr), [commits](https://github.com/paradoxum-games/lyra/commits/main). ProfileStore: single module, session locking with final-save hand-off, Reconcile, Mock, receipt helpers — [API](https://madstudioroblox.github.io/ProfileStore/api/), [repo](https://github.com/MadStudioRoblox/ProfileStore) |
| ProfileStore (experimental project) | **Keep and adopt as primary** | 1.0.3 (Wally `lm-loleris/profilestore`) | [wally.toml](https://raw.githubusercontent.com/MadStudioRoblox/ProfileStore/main/wally.toml); 1 open issue, a question rather than a bug — [issues](https://github.com/MadStudioRoblox/ProfileStore/issues) |
| `centau/vide@0.4.0` | **Upgrade** | 0.4.1 (11 Jul 2026); watch for next release with Oct 2026 strict-mode fixes | [vide releases](https://github.com/centau/vide/releases), [commits](https://github.com/centau/vide/commits/main) |
| `ffrostfall/crunchyroll@0.2.0` | **Replace (drop)** | Animator under AnimationController with Animation Graphs for player sharks; client procedural animation with LOD for AI fish | Motor6D/R6-oriented, no bone support stated, inactive since 13 Dec 2025 with fixes unreleased — [README](https://github.com/ffrostfall/crunchyroll), [CHANGELOG](https://raw.githubusercontent.com/ffrostfall/crunchyroll/main/CHANGELOG.md); Animation Graphs GA Jul 2026 — [recap](https://devforum.roblox.com/t/weekly-recap-july-13-%E2%80%93-17-animation-graphs-go-live-5%C3%97-more-data-store-storage/4743286) |
| `1foreverhd/topbarplus@3.4.0` | **Upgrade** | 3.5.0 (19-20 Oct 2026) | Core-UI-matching appearance, health-bar spacing, Reduce Motion — [releases](https://github.com/1ForeverHD/TopbarPlus/releases) |
| `red-blox/signal` 2.0.2 | **Replace** | `data-oriented-house/lemonsignal@2.0.0` or `sleitnick/signal@2.0.3` | red-blox/Util archived 4 Mar 2025 — [repo](https://github.com/red-blox/Util); LemonSignal 2.0.0 — [releases](https://github.com/data-oriented-house/LemonSignal/releases) |
| `red-blox/promise` 1.1.1 | **Replace** (or remove) | `evaera/promise@4.0.0`, or plain `task`/coroutines | [Wally index](https://raw.githubusercontent.com/UpliftGames/wally-index/main/evaera/promise), [repo](https://github.com/evaera/roblox-lua-promise) |
| `red-blox/ratelimit` 1.0.1 | **Replace** | Small in-house per-player token bucket, or the chosen networking library's limiter | [red-blox/Util archived](https://github.com/red-blox/Util) |
| `red-blox/guard` 1.0.1 | **Replace** (vendoring acceptable) | `osyrisrblx/t@3.1.1` | [Wally index t](https://raw.githubusercontent.com/UpliftGames/wally-index/main/osyrisrblx/t), [t repo](https://github.com/osyrisrblx/t) |
| (new) state | **Add** | `littensy/charm@0.11.1` and `littensy/charm-sync@0.4.1` | [Charm releases](https://github.com/littensy/charm/releases), [Wally index](https://raw.githubusercontent.com/UpliftGames/wally-index/main/littensy/charm-sync) |
| (new) cleanup | **Add** | `sleitnick/trove@1.8.0` | [Wally index](https://raw.githubusercontent.com/UpliftGames/wally-index/main/sleitnick/trove), [RbxUtil commits](https://github.com/Sleitnick/RbxUtil/commits/main) |

### Inferences
- **Lyra to ProfileStore migration:** ProfileStore can't read Lyra's internal format directly. On first load with ProfileStore, read the old Lyra key (or use Lyra's `peek`) and convert once. Or, since this is a new game with a fresh universe, just start clean, which avoids migration entirely.
- If the developer insists on Lyra for its built-in migrations and validation, the minimum safe setup is:
  - pin a main commit at or after 25 Mar 2026;
  - re-apply and test the lock-refresh patch with forced server shutdowns and rejoin storms (simulating flaky mobile connections);
  - watch the MemoryStore request budget, because the default locks appear MemoryStore-based.
- **vide 0.4.0 to 0.4.1 is low-risk.** The main source change is replacing deprecated `create()` overload forms with `create(class)(props)`.
- **When upgrading LemonSignal to 2.x**, audit any `Connect(fn, extra...)` usage.

### Gaps
- The contents of the developer's vendored Lyra lock-refresh patch weren't available, so it's unknown whether upstream main has changed the code it touches.
- The ProfileStore Wally package `lm-loleris/profilestore` matches the repo's wally.toml. However, open issue #21 (Jun 2026) asks whether an "official" Wally upload exists, and the answer wasn't visible in the fetched content.
