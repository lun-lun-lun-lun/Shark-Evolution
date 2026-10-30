# Movement Authority, Client Prediction, Lag Compensation, and Input for a 3D Swimming Shark on Roblox (state as of Oct 30, 2026)

Legend: "Official" means Roblox Creator Hub docs or a post by Roblox staff. "Community" means non-staff DevForum posts or GitHub. "(snippet)" means I only saw the search-result snippet, not the full page. API reference pages fetched here show `last_updated: 2026-10-29`. Everything else is dated inline where a date was available.

---

## 1. Roblox native Server Authority: what it does, how it is enabled, constraints, cost, known bugs, and whether it suits a custom mover-driven swimming shark

### Takeaway
Server Authority (SA) has been generally available since July 9, 2026, and stays opt-in for each experience. The server runs the simulation authoritatively. The client predicts automatically and rolls back/resimulates any "Simulation Access" state written inside `RunService:BindToSimulation`, and gameplay inputs may only arrive through the Input Action System. A custom non-Humanoid swimmer driven by movers or ControllerManager is technically supported. As of Aug–Oct 2026, though, SA has several problems that land squarely on a mobile-first game at 150–300 ms: an open report that input delay grows with player count at high ping, misprediction and stutter reports for fast or constraint-driven movement, a client resimulation cost that grows with latency, custom state limited to attributes (64 per instance), no built-in lag-compensated hit detection, and a known issue where mobile clients lag desktop builds and cannot connect.

### Cited Findings

#### Status, timeline, positioning
- Rollout: Early Access in Oct 2025, Studio Beta in Dec 2025, Client Beta in Apr 2026, Full Release in Jul 2026. Each experience opts in separately. — [Community reference, MrChickenRocket/Server-Authority-Tutorial-Cakeman (2026)](https://github.com/MrChickenRocket/Server-Authority-Tutorial-Cakeman/blob/main/docs/reference/server-authority.md)
- Staff (welblander/Bryan) posted the Studio Beta on Dec 9, 2025. It was Studio-only at that point, and published games could not use it. — [Studio Beta post](https://devforum.roblox.com/t/studio-beta-build-fair-responsive-games-with-server-authority/4139157)
- Staff posted the Full Release on July 9, 2026. The newsroom article and the official Roblox X post appeared the same day. — [Full Release post](https://devforum.roblox.com/t/full-release-ship-fair-and-competitive-games-with-server-authority/4727993); [Roblox newsroom, Jul 9 2026](https://about.roblox.com/newsroom/2026/07/creating-responsive-cheat-resistant-games-roblox-server-authority); [Roblox on X](https://x.com/Roblox/status/2075267517111582783)
- A Client Beta post titled "[Client Beta] Publish & Test Your Server Authoritative Experiences" exists. I did not read it in full. — [Client Beta post](https://devforum.roblox.com/t/client-beta-publish-test-your-server-authoritative-experiences/4606949)
- Staff (Khanovich) wrote in the full-release thread that SA will always be opt-in because it does not suit every game, and named World of Warcraft and Destiny 2 as client-authoritative examples. — [Full Release thread](https://devforum.roblox.com/t/full-release-ship-fair-and-competitive-games-with-server-authority/4727993)
- Roblox's own "Network ownership, movement validation, and physics" security page still describes SA as a beta that is coming soon, so that page has not been updated since before July 2026. — [Movement validation doc](https://create.roblox.com/docs/scripting/security/network-ownership)

#### What it does (mechanism)
- The server is the single source of truth and clients report only their inputs. Clients predict their own results immediately. On a misprediction the client rolls back to the server's authoritative state and resimulates. The docs name the combined system latency compensation. — [SA docs](https://create.roblox.com/docs/projects/server-authority)
- The client simulates a few frames ahead of the last known server state, and the number of frames depends on latency. — [SA docs](https://create.roblox.com/docs/projects/server-authority)
- The client times its inputs so they reach the server for the intended frame, based on measured latency. The server simulates the same inputs independently. — [Community reference](https://github.com/MrChickenRocket/Server-Authority-Tutorial-Cakeman/blob/main/docs/reference/server-authority.md)
- Roblox says SA prevents whole classes of cheats, including fly hacks, speed hacks, flings and wall clipping, and recommends it for shooters, racing, sports and combat. — [Security tactics doc](https://create.roblox.com/docs/scripting/security/security-tactics); [Newsroom](https://about.roblox.com/newsroom/2026/07/creating-responsive-cheat-resistant-games-roblox-server-authority)

#### How to enable and the hard requirements
- In Studio's Workspace properties, set `Workspace.AuthorityMode = Server`. This automatically sets `NextGenerationReplication` Enabled, `PlayerScriptsUseInputActionSystem` Enabled, `SignalBehavior` Deferred, `StreamingEnabled` Enabled and `UseFixedSimulation` Enabled. Then save and publish. A reply also mentions an "Automatic" option on the toggle. — [Full Release post](https://devforum.roblox.com/t/full-release-ship-fair-and-competitive-games-with-server-authority/4727993); [SA docs](https://create.roblox.com/docs/projects/server-authority)
- Scripts cannot set `AuthorityMode`. The attempt fails with a capability error. — [Community reference](https://github.com/MrChickenRocket/Server-Authority-Tutorial-Cakeman/blob/main/docs/reference/server-authority.md)
- Roblox's reasons for each requirement:
  - StreamingEnabled is used by the automatic prediction system.
  - Deferred signals keep listeners out of the core simulation loop.
  - NextGenerationReplication is an early preview of a new property-sync system, and SA requires it.
  — [Full Release post](https://devforum.roblox.com/t/full-release-ship-fair-and-competitive-games-with-server-authority/4727993)
- Under the default Automatic mode, StreamingEnabled sets the prediction radius around each character. — [Community reference](https://github.com/MrChickenRocket/Server-Authority-Tutorial-Cakeman/blob/main/docs/reference/server-authority.md)
- Studio Beta testers objected to mandatory streaming. One cited terrain lag spikes; another cited overhead on small maps. — [Studio Beta thread](https://devforum.roblox.com/t/studio-beta-build-fair-responsive-games-with-server-authority/4139157)
- Input rules:
  - Every input that affects simulation must be an InputAction.
  - `UserInputService.InputBegan` must not feed the core simulation.
  - InputContexts must be descendants of the Player, so clone them per player from ReplicatedStorage in a server script.
  - Read InputActions inside bound functions on both client and server.
  — [SA docs](https://create.roblox.com/docs/projects/server-authority)
- More input rules from the community reference:
  - Read action state, not events.
  - Name every action, because the simulation looks actions up by name.
  - The server drops inputs that arrive too early or too late.
  - Range-check input values on the server the same way you would a RemoteEvent.
  — [Community reference](https://github.com/MrChickenRocket/Server-Authority-Tutorial-Cakeman/blob/main/docs/reference/server-authority.md)
- Staff answered a question about custom characters: `SetNetworkOwner` plus a ControllerManager is not enough on its own, and InputActions are required. — [Tech Deep Dive thread](https://devforum.roblox.com/t/server-authority-tech-deep-dive-engineering-insights/4624565)
- Breaking change at full release: the camera InputAction no longer syncs, and `Player.InputContexts.CameraContext.CameraAction` is gone. Use `Player:GetCameraState()` instead. It returns `CFrame`, `FieldOfView` and `ViewportSize`, synced from client to server. — [Full Release post](https://devforum.roblox.com/t/full-release-ship-fair-and-competitive-games-with-server-authority/4727993)
- If aim feeds the simulation, route it through an explicit InputAction. — [Community reference](https://github.com/MrChickenRocket/Server-Authority-Tutorial-Cakeman/blob/main/docs/reference/server-authority.md)
- Set custom InputAction data in RenderStepped or BindToRenderStep, not inside `BindToSimulation`, where it can give wrong results during resimulation. (snippet, from a translated page) — [SA docs (zh-cn)](https://create.roblox.com/docs/zh-cn/projects/server-authority)

#### Fixed simulation and BindToSimulation
- Signature: `RunService:BindToSimulation(function, frequency?: StepFrequency, priority?: int): RBXScriptConnection`. The default frequency is `Hz30` and the default priority is `2000`. Behavior:
  - It is only available when `Workspace.UseFixedSimulation` is enabled.
  - It runs at a fixed rate regardless of frame rate.
  - It is called again whenever the client resimulates.
  - Reading or writing an unsynchronized property inside it raises an error.
  — [RunService API ref (updated 2026-10-29)](https://create.roblox.com/docs/reference/engine/classes/RunService.md)
- `Enum.StepFrequency` values are `Hz60`, `Hz30`, `Hz15`, `Hz10`, `Hz5` and `Hz1`. The physics world runs at 60 Hz, and the reference advises passing `Hz60` explicitly. — [Community reference](https://github.com/MrChickenRocket/Server-Authority-Tutorial-Cakeman/blob/main/docs/reference/server-authority.md)
- During resimulation only `BindToSimulation` callbacks run. Heartbeat and RenderStepped do not. Predicted objects must run identical code on client and server. — [Tech Deep Dive](https://devforum.roblox.com/t/server-authority-tech-deep-dive-engineering-insights/4624565)
- `time()` is synchronized fixed-step time. `tick()`, `os.time()` and `os.clock()` are not synchronized and do not roll back. — [SA docs](https://create.roblox.com/docs/projects/server-authority)
- Only members marked "Simulation Access" are predicted, and only those may be used inside bound functions. Animator `LoadAnimation`, `Play`, `Stop`, `AdjustSpeed` and `TimePosition` have simulation access. — [SA docs](https://create.roblox.com/docs/projects/server-authority)
- Community-verified simulation-access members: `SetAttribute`/`GetAttribute`, `BasePart.AssemblyLinearVelocity`, `VectorForce.Force`, `AlignOrientation.CFrame`, `AlignPosition.Position`, `Constraint.Enabled` and `BasePart.CFrame`. `Name`, `Size` and `Parent` are not. — [Community reference](https://github.com/MrChickenRocket/Server-Authority-Tutorial-Cakeman/blob/main/docs/reference/server-authority.md)
- Every ControllerManager property has `simulationAccess: true`: `ActiveController`, `BaseMoveSpeed`, `BaseTurnSpeed`, `FacingDirection`, `MovingDirection`, `UpDirection`, `RootPart`, `GroundSensor` and `ClimbSensor`. — [ControllerManager API ref (2026-10-29)](https://create.roblox.com/docs/reference/engine/classes/ControllerManager.md)
- SwimController's pitch and roll properties are also marked simulation access. — [SwimController API ref (2026-10-29)](https://create.roblox.com/docs/reference/engine/classes/SwimController.md)
- BuoyancySensor's `FullySubmerged` and `TouchingSurface` are also marked simulation access. — [BuoyancySensor API ref (2026-10-29)](https://create.roblox.com/docs/reference/engine/classes/BuoyancySensor.md)

#### What is and is not rolled back
- Properties with simulation access sync automatically. Attributes are the main way to sync custom data on predicted instances. Rules:
  - An attribute mismatch triggers a full rollback and resimulation.
  - Write attributes only from bound functions.
  - Only the first 64 attributes on an instance replicate.
  - Attribute names and string values are limited to 50 characters each.
  — [SA docs](https://create.roblox.com/docs/projects/server-authority)
- Module-level Luau state does not roll back, and it diverges between client and server. — [Community reference](https://github.com/MrChickenRocket/Server-Authority-Tutorial-Cakeman/blob/main/docs/reference/server-authority.md)
- Official hooks for rolling back custom state:
  - `RunService.Rollback(time)` fires after a misprediction is detected and eligible state has been rolled back, but before resimulation starts. Its documented purpose is rewinding custom state that is not stored in properties or attributes.
  - `RunService:IsResimulating()` reports whether a replay is in progress. Bound callbacks and property-changed signals fire again for each replayed step, which can duplicate sounds or network requests.
  - `RunService.Misprediction(time, instances, stats)` exposes predicted versus authoritative values. It requires Plugin/OpenCloud capability.
  — [RunService API ref](https://create.roblox.com/docs/reference/engine/classes/RunService.md)
- Inside bound functions, `Destroy()` or a `Parent` change on an instance already in the DataModel raises an error. Instead, record the intent in an attribute and remove the instance outside the simulation, for example in a server `PostSimulation` handler. — [SA docs](https://create.roblox.com/docs/projects/server-authority)
- Instances created inside bound functions are "stitched": the client's copy is merged with the server's. — [SA docs](https://create.roblox.com/docs/projects/server-authority); [SA techniques](https://create.roblox.com/docs/projects/server-authority/techniques)
- Sounds and VFX should render from RenderStepped, driven by simulated state, so that mispredicted effects can be undone. — [SA techniques](https://create.roblox.com/docs/projects/server-authority/techniques)
- RemoteEvents are not ordered relative to property and attribute updates. A thread reply puts the skew at about 40–50 ms. — [SA docs](https://create.roblox.com/docs/projects/server-authority); [Full Release thread](https://devforum.roblox.com/t/full-release-ship-fair-and-competitive-games-with-server-authority/4727993)
- Prediction covers whole physics assemblies: welded or jointed parts, the parent Model, and any Humanoid in it. — [SA docs](https://create.roblox.com/docs/projects/server-authority)
- Prediction modes and APIs:
  - `RunService:SetPredictionMode(instance, mode)` is client-only.
  - `GetPredictionStatus` reports whether an instance is predicted, authoritative, or not participating.
  — [RunService API ref](https://create.roblox.com/docs/reference/engine/classes/RunService.md)
- The three modes, per the community reference:
  - `Automatic` (the default) predicts objects near the local character.
  - `On` always predicts and is expensive.
  - `Off` never predicts.
  - Anchored parts are excluded under Automatic, and server snapshots silently overwrite client writes to them.
  — [Community reference](https://github.com/MrChickenRocket/Server-Authority-Tutorial-Cakeman/blob/main/docs/reference/server-authority.md)

#### Movers, constraints, custom movement
- The full release lists custom movement such as dashes and boosts as supported, provided the logic is scripted on both client and server. — [Full Release post](https://devforum.roblox.com/t/full-release-ship-fair-and-competitive-games-with-server-authority/4727993)
- The community reference says constraints, BodyMovers and custom flight/boost are supported if scripted identically in `BindToSimulation`. It reports three problems:
  - Quantization causes drift in some BodyMovers and Constraints.
  - Community reports describe residual jitter with LinearVelocity-driven characters.
  - Windows clients and Linux servers disagree in the low float bits.
  — [Community reference](https://github.com/MrChickenRocket/Server-Authority-Tutorial-Cakeman/blob/main/docs/reference/server-authority.md)
- Staff details on mispredictions (May 2026):
  - A position is flagged as mispredicted at 0.1 studs or more, using quantization buckets.
  - Values near a bucket edge can be flagged falsely. Staff plan to fix this.
  - Floating-point drift causes mispredictions even when jumping on flat ground.
  - Quantized vehicle state leaves unresolved drift in some BodyMovers and Constraints.
  - Player collisions always mispredict.
  — [Tech Deep Dive](https://devforum.roblox.com/t/server-authority-tech-deep-dive-engineering-insights/4624565)
- Community reports from July 2026:
  - Constraints such as ball sockets and VectorForce caused endless mispredictions.
  - Movement faster than about 40 studs/s looks stuttery.
  - One tester said base Humanoid movement feels wrong.
  - A dissenting report says a custom character worked within a few days and stock Humanoids need little work.
  — [Client Beta "nightmare" thread](https://devforum.roblox.com/t/server-authority-client-beta-was-a-damn-nightmare-heres-some-advice/4712758)

#### Costs (client and server)
- Resimulation is the main client CPU cost. At 100 ms of latency, one correction re-simulates 100 ms of physics, which staff put at about 6.25x the load of a client-authoritative game. — [Tech Deep Dive (staff, May 8 2026)](https://devforum.roblox.com/t/server-authority-tech-deep-dive-engineering-insights/4624565)
- In one 3-second single-process sample, 105 steps were resimulated against 91 live steps, so resimulation is the common case. — [Community reference](https://github.com/MrChickenRocket/Server-Authority-Tutorial-Cakeman/blob/main/docs/reference/server-authority.md)
- Mitigations as of May 2026:
  - Dynamic input-latency injection is a prototype. It lets low-end devices trade input lag for less resimulation.
  - Partial prediction is a stated direction.
  - Low-fidelity resimulation is a research topic.
  - Staff call resimulation performance a top internal priority.
  — [Tech Deep Dive](https://devforum.roblox.com/t/server-authority-tech-deep-dive-engineering-insights/4624565)
- The full release claims fewer mispredictions and better framerate on lower-end devices. — [Full Release post](https://devforum.roblox.com/t/full-release-ship-fair-and-competitive-games-with-server-authority/4727993)
- Server cost: utilization rises with SA. Roblox expects the impact to be small unless the server is already near its limit, for example with heavy server physics. It suggests Extended Services for Compute for more CPU per player. — [Full Release post](https://devforum.roblox.com/t/full-release-ship-fair-and-competitive-games-with-server-authority/4727993)
- The Studio Beta post said both server and clients do more work and told developers to expect higher server costs. — [Studio Beta](https://devforum.roblox.com/t/studio-beta-build-fair-responsive-games-with-server-authority/4139157)
- The SA visualizer (Ctrl+Shift+F6) shows:
  - Prediction success rate.
  - Input accept rate.
  - Input drop reasons: too old, out of order, buffer full.
  - Server heartbeat. An RCC heartbeat below 59 FPS means the server cannot keep up.
  — [SA techniques](https://create.roblox.com/docs/projects/server-authority/techniques)
- The release post does not discuss bandwidth. — [Full Release post](https://devforum.roblox.com/t/full-release-ship-fair-and-competitive-games-with-server-authority/4727993)

#### Known issues and bug reports (2026)
- Known issues listed at full release:
  - Mobile and console client builds trail desktop updates by a few days and cannot connect until they update.
  - Each Animator can play at most 8 tracks at once.
  - Instances are limited to 64 attributes.
  - Some Animate script features are unsupported.
  - RemoteEvents are not on the shared timeline.
  - Tool welds can be lost during a misprediction.
  — [Full Release post](https://devforum.roblox.com/t/full-release-ship-fair-and-competitive-games-with-server-authority/4727993)
- July 15, 2026 report: mispredictions rise when the client runs above 60 FPS at 50+ ms ping, and capping at 60 FPS largely fixes it.
  - Jul 23: staff said a fix was coming in version 732.
  - Aug 1–5: users reported mixed results. One said the delay was roughly halved.
  - Aug 6: users pointed to release notes 733, which listed the fix as pending.
  - Aug 20: the thread closed automatically.
  — [FPS/latency bug thread](https://devforum.roblox.com/t/server-authority-mispredictions-scale-with-client-fps-under-high-latency/4739385)
- July 29, 2026 report, tested on a default baseplate with roughly 100+ ms ping (US client to a Brazil server):
  - Input delay grows with every player added, and play becomes unplayable at about 4 or more players.
  - When a new player joins, existing clients see a burst of rollbacks.
  - Staff replied that they were investigating. A second developer reproduced it on Aug 18.
  - The thread reports no fix.
  — [Player-count/high-ping bug thread](https://devforum.roblox.com/t/server-authority-input-delay-scales-with-player-count-at-high-ping/4763481)
- Replies to the Tech Deep Dive describe random mispredictions in near-empty places. Staff tagged an engineer to investigate. — [Tech Deep Dive](https://devforum.roblox.com/t/server-authority-tech-deep-dive-engineering-insights/4624565)
- Reported false rollbacks about once per second, tied to attribute names or `SetAttribute` order. (snippet) — [Attribute rollback bug](https://devforum.roblox.com/t/server-authority-instance-stitching-prediction-inexplicably-causes-rollbacks-based-on-attribute-names-setattribute-order/4754479)
- Reported CFrame jitter in specific regions of the world. Staff said some jitter is expected and is more visible near the camera. (snippet) — [CFrame jitter thread](https://devforum.roblox.com/t/server-authority-issue-basepart-cframe-jitter-in-specific-parts-of-the-world/4728459)
- One fighting-game developer's port (Jul 1, 2026):
  - About 1.5 months to recover the original game feel.
  - Packed flags into bitmasks because of attribute limits.
  - Had to deduplicate VFX and audio that replayed during rollbacks.
  - Found input actions unreliable, so built an input retry buffer with about 2 frames of input delay.
  - Rendered a client-only entity interpolated by RTT/2.
  - Giving each entity its own `BindToSimulation` performed poorly.
  — [Client Beta "nightmare" thread](https://devforum.roblox.com/t/server-authority-client-beta-was-a-damn-nightmare-heres-some-advice/4712758)

#### Lag compensation and other players under SA
- As of May 2026, staff said built-in server-side rewind did not exist yet, though it could be built with existing APIs. Shooter-style latency compensation ranks behind resimulation performance. For melee-style games, forcing prediction on other players may work. — [Tech Deep Dive](https://devforum.roblox.com/t/server-authority-tech-deep-dive-engineering-insights/4624565)
- The Laser Tag template's shooting code is unchanged, and brief shooting mispredictions are expected. A July 2026 reply says the team is looking into synchronizing RemoteEvents and into server-side rewind hit detection. — [Full Release thread](https://devforum.roblox.com/t/full-release-ship-fair-and-competitive-games-with-server-authority/4727993)
- Other players' inputs are not forwarded to clients by default. The pattern used in the Racing template is for the server to write inputs to attributes, which clients then read in `BindToSimulation`. — [SA techniques](https://create.roblox.com/docs/projects/server-authority/techniques)
- Ways to design around latency:
  - Use slower acceleration, which hides position error.
  - Use delayed effects, such as a fuse before an explosion.
  - Smooth rendering by hiding the simulated part and drawing an anchored visual clone that follows it with `TweenService:SmoothDamp` in RenderStepped.
  — [SA techniques](https://create.roblox.com/docs/projects/server-authority/techniques)
- Testing:
  - Studio's Network Simulation was upgraded for latency and packet loss.
  - Racing, Soccer and Laser Tag templates are the reference implementations.
  - The community reference recommends Test → Server & Clients with Network Simulation enabled.
  — [Full Release post](https://devforum.roblox.com/t/full-release-ship-fair-and-competitive-games-with-server-authority/4727993); [Community reference](https://github.com/MrChickenRocket/Server-Authority-Tutorial-Cakeman/blob/main/docs/reference/server-authority.md)

### Inferences
- **Fit for a mover-driven, non-Humanoid shark: technically yes.**
  - Every piece the shark needs is simulation-access: `VectorForce.Force`, `AlignOrientation.CFrame`, `AssemblyLinearVelocity`, the ControllerManager and SwimController properties, and BuoyancySensor outputs.
  - Boost and flight-style movement is explicitly in scope.
  - The whole swim step would live in one shared ModuleScript bound with `BindToSimulation` on both client and server: read the move, boost and aim InputActions, then set the mover targets.
  - Every piece of per-shark state (boost meter, cooldowns, size tier, bite timer) must be an attribute or an engine property, or be rewound by hand in `RunService.Rollback`.
- **The SMASH-plan note is confirmed.** Turning SA on changes the input path (IAS only), the stepping model (`UseFixedSimulation` and `BindToSimulation`), streaming (mandatory), event semantics (Deferred signals apply to the whole codebase, fish AI included) and state sync (attributes). Luau state does not become rollback-safe on its own.
- **Risks specific to this game, highest first:**
  1. The open July 29 report (input delay that scales with player count at 100+ ms) describes almost exactly the target conditions of 10–20 players at 150–300 ms. Treat it as a go/no-go item and verify it in live tests.
  2. Client resimulation cost grows with RTT. Using staff's figure of 6.25x at 100 ms, a 250 ms RTT means roughly 15 or more physics steps re-run per correction. On low-end phones that competes with rendering hundreds of fish. The planned mitigations (input-latency injection, partial prediction) were not confirmed shipped in any source found.
  3. Boosting sharks will likely exceed the roughly 40 studs/s at which stutter was reported.
  4. Every PvP bite is a contact event, and contact always mispredicts.
  5. The 64-attribute and 50-character limits constrain state.
  6. The mobile client-version lag known issue could lock mobile players out for days after some engine updates, which would be very costly for a mobile-first game if it still applies.
  7. Server CPU rises, competing with fish AI on the same server.
  8. Bites still need custom lag compensation under SA.
- **Do not predict AI fish.** Keep them server-only, either anchored or CFrame-driven, or as client-rendered proxies. Anchored parts are excluded from Automatic prediction. Otherwise prediction near the player multiplies resimulation cost.
- **Run the swim simulation at Hz30 (the default) or Hz15.** That should cut Luau simulation and resimulation cost, and slow-accelerating sharks match the docs' advice to hide error with slower acceleration. The physics world still steps at 60 Hz, though, so mover-driven assemblies still cost physics resimulation.
- **Pragmatic hedge.** Write the movement as a pure step function, `step(state, input, dt)`, with state kept in attributes. Ship v1 on the default model (Q2), and keep SA as a later migration once the high-ping and player-count issue is verifiably fixed. Roblox has said SA stays opt-in.

### Gaps
- No official numbers for SA server CPU or bandwidth per player. No official behavior data for 150–300 ms, jitter or packet loss.
- No mobile-specific SA performance or player reports found, beyond the version-lag known issue and the "improved framerate on lower-end devices" claim.
- Whether the July 29 high-ping/player-count bug is fixed as of Oct 30, 2026 is unknown. Whether the v732/733 fix for FPS-scaled mispredictions fully shipped is unknown, with mixed user reports.
- No official support matrix for LinearVelocity, AlignOrientation, VectorForce, ControllerManager/SwimController or Humanoid Terrain-water swimming under SA. Only simulation-access flags and community anecdotes exist.
- Unclear whether the mobile/console version-lag lockout recurs after every engine update or only happened around the launch.
- The prediction-radius values and how they map to streaming radius are not documented in the pages fetched. Remote-player interpolation delay under SA is also undocumented.

---

## 2. Alternatives on high-latency mobile: client network ownership with server checks versus fully custom predicted movement (Chickynoid-style), and what games actually do

### Takeaway
Default client network ownership with server-side sanity checks is still the simplest option, the cheapest on the server, and the best-feeling for the local player at 150–300 ms, because nothing is rolled back or resimulated. Its weakness is that it detects cheating within a tolerance rather than preventing it, and Roblox's guidance gives heuristics without thresholds. Fully custom predicted movement in the Chickynoid style prevents movement cheats and allows custom lag compensation, but it is the most complex and most server-expensive path, and Chickynoid itself is a box-collider platformer controller that looks unmaintained since 2024. No reliable source says what top Roblox games use in 2025–2026.

### Cited Findings

#### (a) Default client network ownership plus server checks
- By default, the server gives clients network ownership of unanchored parts near their characters. An owning client has complete authority over those objects' physics. — [Movement validation doc](https://create.roblox.com/docs/scripting/security/network-ownership)
- Client-owned parts respond faster because there is no server round trip, and they take physics load off the server. Roblox cannot verify the physics of a client-owned part, so exploiters can teleport owned parts, push them through walls or make them fly. — [Physics network-ownership doc](https://create.roblox.com/docs/physics/network-ownership)
- What exploiters can do with ownership:
  - Teleport anywhere, which lets them claim many unanchored parts quickly.
  - Set any WalkSpeed and fly.
  - Bypass collisions.
  - Set extreme velocities, including Inf or NaN, which flings other players.
  - Manipulate when Touched fires.
  — [Movement validation doc](https://create.roblox.com/docs/scripting/security/network-ownership)
- Clients can fire Touched events on parts they own without any real contact, so Touched-derived events must be validated. — [Physics network-ownership doc](https://create.roblox.com/docs/physics/network-ownership)
- Exploiters can trigger Touched and ProximityPrompts at any range or rate, and fire remotes at any rate with any arguments. — [Security tactics doc](https://create.roblox.com/docs/scripting/security/security-tactics)
- Roblox's movement-validation guidance:
  - There is no universal solution.
  - Distance-over-time checks work for simple movement but break down for complex movement systems.
  - Average positions over time because of latency.
  - Use leaky-bucket accumulators to absorb bursts.
  - Project onto a plane (XZ for ground movement) to avoid penalizing vertical motion.
  - Build explicit exemptions for legitimate teleports.
  - Basic heuristics can flag innocent players on unstable connections.
  - Tune thresholds per game. The page gives no code and no numbers.
  — [Movement validation doc](https://create.roblox.com/docs/scripting/security/network-ownership)
- Use manual ownership, or turn off automatic ownership, for gameplay-critical unanchored parts. Use `SetNetworkOwner(nil)` sparingly because it causes jittery physics. — [Movement validation doc](https://create.roblox.com/docs/scripting/security/network-ownership); [Physics network-ownership doc](https://create.roblox.com/docs/physics/network-ownership)
- General rules:
  - Assume all client data is manipulated.
  - Validate on the server that a requested action is possible and permitted. The docs' example is a distance check before a shop purchase.
  - Decide a maximum rate for each remote.
  — [Security tactics doc](https://create.roblox.com/docs/scripting/security/security-tactics)
- A community suggestion is to compare a character's observed velocity against its expected velocity rather than trust WalkSpeed. (snippet; an older thread, around 2019) — [DevForum anti-exploit thread](https://devforum.roblox.com/t/roblox-anti-exploit-question/277167)

#### (b) Fully custom predicted movement (Chickynoid)
- Chickynoid is a server-authoritative character controller by MrChickenRocket and Brooke, meant as a hard replacement for Humanoid characters.
  - Each frame the client builds a command, simulates it locally and sends it.
  - The server replays the commands.
  - When results diverge, the client rolls back and replays its unconfirmed commands.
  - It uses its own collision and math, not Roblox physics.
  — [Chickynoid GitHub](https://github.com/easy-games/chickynoid)
- Features:
  - Ray-scan and projectile weapons with server-side hit detection and lag compensation.
  - Per-client selective replication.
  - Compression that it claims cuts bandwidth to nearly half of stock Roblox (unverified).
  - Server-side bots.
  - Anti-cheat that watches the incoming command stream, with tolerances for unstable connections.
  — [Chickynoid GitHub](https://github.com/easy-games/chickynoid)
- Limitations:
  - The character is a fixed-size box.
  - It cannot push, pull or ride physics parts.
  - Terrain collision is less precise.
  - The README calls it a platforming character with a gun and little else.
  - Higher client FPS adds server work, and clients deliberately lag above 120 FPS by default.
  — [Chickynoid GitHub](https://github.com/easy-games/chickynoid)
- Activity: about 265 stars, 399 commits, 6 open issues and 11 open pull requests. An aggregator lists the last push as May 27, 2024, which suggests the project is dormant. — [Chickynoid GitHub](https://github.com/easy-games/chickynoid); [gittrend](https://gittrend.io/repo/easy-games/chickynoid)
- Chickynoid's author, MrChickenRocket, now publishes a Server Authority tutorial and reference repo. — [Server-Authority-Tutorial-Cakeman](https://github.com/MrChickenRocket/Server-Authority-Tutorial-Cakeman/blob/main/docs/reference/server-authority.md)

#### What games actually do (2025–2026)
- Staff say SA will always be opt-in and point to client-authoritative AAA games as valid designs. — [Full Release thread](https://devforum.roblox.com/t/full-release-ship-fair-and-competitive-games-with-server-authority/4727993)
- A July 24, 2026 article discusses community talk about Rivals enabling SA, but it cites no developer source and confirms no adoption. — [nerdschalk (secondary)](https://nerdschalk.com/rivals-roblox-update-explained-server-authority-skin-case-wins-and-player-counts/)
- One creator testimonial (ThoughtSpinnr, who builds a custom character system on SA) subjectively claims cheating has essentially disappeared. It includes no metrics. — [Roblox newsroom](https://about.roblox.com/newsroom/2026/07/creating-responsive-cheat-resistant-games-roblox-server-authority)

### Inferences
- **How each option feels at 150–300 ms RTT:**
  - **(a) Client ownership.** The local shark responds instantly and is never corrected, because the client owns its own assembly. Remote sharks and fish appear in the past, by roughly half the RTT plus the interpolation buffer. Bites therefore need rewind (Q4).
  - **(c) Native SA.** It feels instant when predictions match. Corrections show up during contact, during jitter, and when inputs are dropped (too old, out of order or buffer full) on lossy mobile links. The resimulation CPU spikes and the open high-ping/player-count bug make it the riskiest of the three on poor networks today.
  - **(b) Custom Chickynoid-style netcode.** The local feel is similar to SA's, but you own every edge case in Luau. That is the architecture whose complexity and server cost overran SMASH's frame budget. You would also need 3D swim collision against terrain and rocks, and Chickynoid's box/platformer collision would not cover it.
- **How exploitable each option is.** (c) and (b) prevent teleport, speed and fly cheats. (a) only detects them within a tolerance.
  - In an open-ocean, mostly-PvE game, the cheat that matters is "eat everything fast": speed or teleport combined with bite spam.
  - Server-validated eats close that hole on their own terms: distance with rewind, rate and cooldown, and a growth budget. Eats need that validation under any model, so it is not extra work.
  - Under (a), a 3D leaky-bucket speed budget (full 3D, since plane projection does not fit swimming) and NaN/Inf velocity sanitizing cover the rest.
  - Turning off shark-to-shark collisions with collision groups removes most fling vectors.
  - Responding by rubber-banding (server snaps position or temporarily takes ownership) rather than kicking avoids punishing laggy mobile players.
- **Ranking for this project** (simplicity and server cost, plus feel on poor mobile networks): (a) first; (c) as a later upgrade path once its high-ping bugs are verified fixed; (b) last.

### Gaps
- No reliable list of which top Roblox games in 2025–2026 use SA, client ownership with checks, or custom netcode. The Rivals adoption is unconfirmed.
- No measured feel or latency comparisons between the models on Roblox at 150–300 ms.
- Roblox publishes no numeric thresholds for movement checks.
- Chickynoid's 2025–2026 maintenance status is unconfirmed, and no swimming adaptation of it was found.
- How much interpolation buffer Roblox's default replication applies to other players' characters was not found.

---

## 3. Swimming implementation: Terrain water and Humanoid swimming versus custom underwater movement; ControllerManager, SwimController and AirController in 2026; buoyancy and gravity; mobile feel

### Takeaway
Roblox's current path for physics characters is a ControllerManager with child controllers and sensors. SwimController (reference updated 2026-10-29) drives the root toward `MovingDirection × BaseMoveSpeed × MoveSpeedFactor`, pitches toward `FacingDirection` and rolls to stay upright, which is close to free 3D swimming. All of these properties are simulation-access, so they are SA-ready. Water is detected by BuoyancySensor, which only recognizes Terrain water. The Luau Character Controller Library adds a Swimming ability, but sources disagree about whether it is released. No official source covers gravity-neutral underwater movement or mobile 3D swim controls, so that part is design work.

### Cited Findings
- ControllerManager manages simulated motion for its `RootPart` and can be used to build a physics-based character controller. Humanoid characters usually assign `Humanoid.RootPart` as the root. Nothing in the reference requires a Humanoid. — [ControllerManager API ref (2026-10-29)](https://create.roblox.com/docs/reference/engine/classes/ControllerManager.md)
- ControllerManager behavior:
  - You set `ActiveController` directly. If activation fails, it retries the next frame. Setting it to `nil` disables the controller.
  - `MovingDirection` has a magnitude from 0 to 1 and is multiplied by `BaseMoveSpeed` to give the target velocity.
  - `FacingDirection` aligns the `RootPart` LookVector.
  - `UpDirection` defaults to (0,1,0). AirController and SwimController align the character to it.
  — [ControllerManager API ref](https://create.roblox.com/docs/reference/engine/classes/ControllerManager.md)
- When SwimController is the active controller, it:
  - Drives the root toward the target velocity, scaled by `MoveSpeedFactor`.
  - Pitches toward `FacingDirection`.
  - Rolls to keep the character oriented.
  - Exposes `PitchMaxTorque`, `PitchSpeedFactor` (multiplied by `BaseTurnSpeed`), `RollMaxTorque` and `RollSpeedFactor`. The torque settings have no effect when `RigidityEnabled` is on.
  - Works only when parented to a ControllerManager.
  - No deprecation notice.
  — [SwimController API ref (2026-10-29)](https://create.roblox.com/docs/reference/engine/classes/SwimController.md)
- BuoyancySensor reports how its part interacts with Terrain water: `FullySubmerged`, and `TouchingSurface` (which can be approximated from the bounding box). It only covers Terrain water. — [BuoyancySensor API ref (2026-10-29)](https://create.roblox.com/docs/reference/engine/classes/BuoyancySensor.md)
- The Character Controller Library (CCL) docs describe a ControllerManager with GroundController, AirController, ClimbController and SwimController children.
  - The Swimming ability has `EnableFastRise` (hold jump to rise faster) and `SpeedMultiplier` (which scales `SwimController.MoveSpeedFactor`).
  - The page says CCL is in beta and is enabled under File > Beta Features > "AvatarAbilities Character Controller Library".
  - Several abilities map onto Humanoid properties.
  — [CCL controllers doc](https://create.roblox.com/docs/characters/character-controller-library/controllers)
- **Conflict on CCL status.** A DevForum weekly recap reportedly says CCL reached full release in the week of Apr 6–10, 2026 (snippet; the recap reportedly includes an auto-generated block). The docs page fetched Oct 2026 still says beta. — [Weekly Recap Apr 6–10, 2026 (snippet)](https://devforum.roblox.com/t/weekly-recap-april-6-10-2026/4568634); [CCL doc](https://create.roblox.com/docs/characters/character-controller-library/controllers)
- An AvatarAbilities Studio Beta post also exists. (snippet) — [AvatarAbilities Studio Beta](https://devforum.roblox.com/t/studio-beta-avatarabilities-library-for-a-luau-based-character-controller/4403904)
- A third-party CCL summary exists but is low-reliability. — [creation.dev](https://www.creation.dev/learn/roblox-character-controller-library-full-release-2026)
- The IAS full-release post calls IAS a foundational pillar of both SA and the CCL. — [IAS Full Release](https://devforum.roblox.com/t/full-release-input-action-system-ias-newly-converted-player-scripts/4678416)
- Under SA:
  - Humanoid and ControllerManager support is not explicitly documented.
  - Physics characters should prefer forces over direct CFrame writes.
  - Kinematic anchored characters should write `BasePart.CFrame` every step.
  — [Community reference](https://github.com/MrChickenRocket/Server-Authority-Tutorial-Cakeman/blob/main/docs/reference/server-authority.md)
- IAS offers a `Direction3D` action type, with an airborne vehicle that levitates and drifts as the example. — [IAS docs](https://create.roblox.com/docs/input/input-action-system)
- `Vector3Scale` tunes each axis of a Direction3D action. `ClampMagnitudeToOne` (default true) stops diagonal input from being faster. — [IAS Full Release](https://devforum.roblox.com/t/full-release-input-action-system-ias-newly-converted-player-scripts/4678416)
- Slower acceleration hides prediction error. — [SA techniques](https://create.roblox.com/docs/projects/server-authority/techniques)

### Inferences
- **Option 1: Terrain water with stock Humanoid swimming.** Least code, but it is built for humanoid avatars. A shark rig with free pitch would fight the Humanoid state machine. Not recommended for a shark. No 2026 source on Humanoid swimming specifics was fetched.
- **Option 2: ControllerManager with SwimController on a custom shark model (no Humanoid).**
  - Set `RootPart` to the shark body and keep `ActiveController = SwimController` permanently, since the whole map is underwater.
  - Feed `MovingDirection` with the camera look vector times the throttle, and `FacingDirection` with the camera look or the velocity.
  - Leave `UpDirection` at world up so the roll alignment keeps the belly down.
  - You get torque-limited pitch and roll and speed factors for free, and it is SA-ready.
  - **Unknowns to prototype first:** whether SwimController activates or behaves correctly outside Terrain water, and how it handles gravity. The references do not say.
- **Option 3: plain movers.**
  - Use LinearVelocity or VectorForce for thrust and AlignOrientation for heading.
  - Neutralize gravity with an upward VectorForce of mass × `workspace.Gravity`, or give LinearVelocity enough MaxForce to dominate gravity.
  - Alternatively, lower `Workspace.Gravity` globally if nothing in the game needs gravity. That affects all unanchored physics.
  - This gives the most control. Under client ownership it has no SA-specific drift or jitter problems. Under SA, note the community reports of LinearVelocity jitter and VectorForce mispredictions.
- **Terrain water or not.**
  - If the ocean is Terrain water, the shark interacts with water, and BuoyancySensor and Humanoid swimming depend on it. Terrain water also brings its own buoyancy and rendering cost (not verified here).
  - A "dry" ocean (no Terrain water, with fog, color grading and caustics as visuals) avoids buoyancy tuning entirely and pairs naturally with Option 3. It also removes any reliance on BuoyancySensor.
  - Weigh the low-end mobile render cost of large Terrain water areas. No source was found on that cost.
- **Mobile control scheme (design inference):**
  - Auto-forward cruise: hold a constant `MovingDirection` magnitude, with the thumbstick modulating speed and steering.
  - Steer with pitch and yaw from the camera (right-side drag) and/or a camera-relative left thumbstick.
  - Put a Boost button in the right-thumb zone, implemented as an IAS Bool action bound to a GuiButton.
  - Optionally add an aim-assist cone for bites.
  - Cap turn rates with `BaseTurnSpeed` and `PitchSpeedFactor`, and give the follow-camera a little lag, to limit motion sickness.
  - Under SA, camera-based steering must be sent as an explicit InputAction (for example a Direction3D "SwimAim" set in RenderStepped), because the camera InputAction no longer syncs.

### Gaps
- No official documentation on how SwimController handles gravity or buoyancy, or whether it can be active without Terrain water.
- AirController behavior beyond aligning to `UpDirection` was not covered in the pages fetched.
- No source fetched on Terrain-water buoyancy forces on custom parts, or on Terrain water performance on low-end phones.
- CCL release status is contradictory between the April 2026 recap and the October 2026 docs.
- No Roblox-specific best-practice source for mobile 3D swim controls such as auto-forward or pitch-from-camera. The recommendations above are design inference.
- Which controllers shipped Roblox shark or fish games use was not found.

---

## 4. Lag compensation for bite/eat interactions: validating client claims, rewind windows, favor-the-attacker versus fairness, cooldowns, and interpolated targets

### Takeaway
The standard technique is server-side rewind. The server keeps a short history of positions and checks a claim against where the target was when the attacker saw it, which is roughly the attacker's RTT plus the client's interpolation delay. The rewind is capped to limit how unfair it can be to the victim. Secondary sources put typical caps at about 200–300 ms. Riot caps rewind per region without publishing numbers. Roblox SA had no built-in rewind as of July 2026, so bite and eat validation must be custom under every model.

### Cited Findings
- General technique:
  - The server keeps timestamped state so it can rebuild the world at any past moment, and processes the shot at the time the shooter fired.
  - The cost is that a victim can be hit shortly after reaching cover. Gambetta calls this somewhat unfair but the most agreeable solution.
  - Clients see other entities in the past because of entity interpolation.
  — [Gabriel Gambetta, Lag Compensation](https://www.gabrielgambetta.com/lag-compensation.html)
- Riot's VALORANT netcode (dated Jul 28, 2020, so older, but the principles still hold):
  - The server rewinds the world to what the shooter saw.
  - Clients send the simulation time they were viewing, which works because client and server timelines are synced to fixed steps.
  - Riot limits how far the server will rewind and tunes the limit per region.
  - Without a limit, a 500 ms player could kill someone half a second after they reached cover.
  - Servers run 128 ticks per second.
  — [Riot, Peeking into VALORANT's Netcode](https://www.riotgames.com/en/news/peeking-valorants-netcode)
- In Riot's playtests, 10 ms of peeker's advantage swung duel win rates from about 90% for the defender to about 90% for the peeker. That shows how sensitive PvP fairness is to latency handling. — [Riot](https://www.riotgames.com/en/news/peeking-valorants-netcode)
- Secondary guides:
  - Rewind caps are typically about 200–300 ms, to limit exploitation.
  - Players beyond the cap lose some accuracy, which the guides call an acceptable trade.
  - A sample server clamps rewind at 0.2 s and estimates the shooter's view as now − RTT/2 − interpolation delay.
  - Log the rewind amount and RTT for every shot.
  — [abratabia (snippet)](https://abratabia.com/multiplayer-web-games/lag-compensation.php); [kindatechnical (snippet)](https://kindatechnical.com/game-development/lag-compensation-prediction-interpolation-and-rollback.html)
- Colyseus clamps the client's render time to the window [now − maxRewind, now]. (snippet) — [Colyseus docs](https://docs.colyseus.io/netcode/lag-compensation)
- A forum post cites a 250 ms cap on client-side hit registration in Apex Legends. This is unverified secondhand commentary. — [EA forum](https://forums.ea.com/discussions/apex-legends-feedback-en/stop-rewarding-laggers-limit-client-side-hit-registration-to-200ms-in-the-past/8435746/replies/8435747)
- Roblox specifics:
  - SA has no built-in server-side rewind yet; staff said in May 2026 it could be built with existing APIs.
  - Forcing prediction on other players may suit melee games.
  — [Tech Deep Dive](https://devforum.roblox.com/t/server-authority-tech-deep-dive-engineering-insights/4624565)
- RemoteEvents run about 40–50 ms off the simulation timeline. The team is looking into server-side rewind hit detection. — [Full Release thread](https://devforum.roblox.com/t/full-release-ship-fair-and-competitive-games-with-server-authority/4727993)
- Under SA, `Touched`/`TouchEnded` can fire inaccurately during rollback. Use `GetTouchingParts()` instead. — [Studio Beta](https://devforum.roblox.com/t/studio-beta-build-fair-responsive-games-with-server-authority/4139157)
- Validate every client claim on the server: check that it is possible, do distance checks, and set a maximum rate per remote. — [Security tactics doc](https://create.roblox.com/docs/scripting/security/security-tactics)
- Touched events on client-owned parts can be spoofed. — [Physics network-ownership doc](https://create.roblox.com/docs/physics/network-ownership)

### Inferences
- **Validation pipeline that works under client ownership and under SA** (design inference):
  1. **Client:** detect the bite locally for instant feedback (play the chomp animation and VFX, and optionally hide the fish optimistically). Send `{targetId, renderServerTime}` through a rate-limited remote. Under SA, a remote claim plus custom rewind is still needed, because a Bool InputAction checked at the server's "now" misses targets the attacker saw in the past.
  2. **Server, cheap checks first:**
     - The attacker is alive and meets the size or tier rule.
     - The cooldown has elapsed, based on server timestamps with a small jitter allowance. A leaky-bucket eats-per-second budget doubles as a growth-rate cap.
     - The target is still alive, and the first valid claim wins.
  3. **Server rewind:**
     - `t = clamp(renderServerTime or (now − RTT − interpDelay), now − maxRewind, now)`.
     - Interpolate the target's position at `t` from a ring buffer.
     - Accept if `distance(attackerMouth, targetPos_t) ≤ biteRange + targetRadius + tolerance`, where tolerance is target speed × sample interval plus a small margin.
     - Also check that the target is inside the mouth cone (dot product).
     - Under client ownership, use the server's current position of the attacker. It already lags the attacker's own view by the uplink latency, which roughly cancels part of the gap.
  4. **Caps (the favor-the-attacker dial):**
     - PvE fish: generous, around 300–400 ms of rewind and larger tolerances. There is no human victim to wrong, and 300 ms mobile players need it.
     - PvP bites: tighter, around 150–250 ms. Beyond the cap the attacker must lead the target. Consider requiring the victim to still be within range plus a slack distance at server time, to limit "bitten after escaping".
- **How much to rewind for interpolated targets.** The attacker rendered server state from about (attacker RTT + interpolation buffer) before the claim arrived. It is better for the client to send the server timestamp of the snapshot it was rendering, as Riot does, than to estimate it. If fish are reconstructed on clients from server seeds or snapshots (another researcher covers this), the rendered time must come from that system's clock.
- **Cost.** There is no need to rewind the whole world. Keep a small ring buffer per fish and per shark (for example 10–20 Hz samples over about 0.5 s), and evaluate only the claimed target. That is O(1) per claim and cheap even with hundreds of fish.
- **Logging.** Record the rewind amount, RTT and rejection reasons for each bite, to tune caps on real mobile networks, as the secondary sources recommend.

### Gaps
- No primary Valve or Riot number for typical rewind caps was retrievable. Valve's developer wiki returned 403, and Gaffer On Games was not consulted. The 200–300 ms figures come from secondary guides.
- Roblox's client interpolation delay for replicated characters and parts is not documented in the pages fetched.
- Whether Roblox's built-in ping APIs are accurate enough to estimate rewind time was not verified.
- No official Roblox sample of server-side rewind, under SA or otherwise, was found.

---

## 5. Input Action System and mobile controls best practice

### Takeaway
The Input Action System (IAS) reached full release on June 11, 2026. It is mandatory for SA simulation inputs, and Roblox's default PlayerScripts are moving onto it: default-on in early 2027 and mandatory in mid 2027. Every action should have keyboard/mouse, gamepad and touch bindings. Touch bindings are GuiButtons attached through `UIButton`. Two gaps reported at release still matter for mobile: there is no clear way to read the default mobile thumbstick direction, and four UI buttons reportedly cannot be mapped onto a Direction2D action. Safe areas are handled with `ScreenGui.ScreenInsets`, which defaults to `CoreUISafeInsets`.

### Cited Findings
- IAS reached full release on June 11, 2026, posted by staff. It consists of InputAction, InputBinding and InputContext.
  - New touch keycodes: `TouchPosition`, `TouchPinch` and `TouchDelta`.
  - `UIModifier` applies on-screen modifiers to touch bindings.
  - `TrackpadPinch` and `MouseWheel` map to Direction1D. `TrackpadPan` and `MouseDelta` map to Direction2D.
  - Gamepad thumbsticks and triggers are supported, with fixes to response curves.
  - `ClampMagnitudeToOne` (default true) and `Vector3Scale` are available.
  — [IAS Full Release](https://devforum.roblox.com/t/full-release-input-action-system-ias-newly-converted-player-scripts/4678416)
- PlayerScripts migration:
  - `Workspace.PlayerScriptsUseInputActionSystem` is Disabled by default for now. The IAS-based scripts are described as functionally identical for default movement and camera.
  - Forked scripts need `StarterPlayer.CreateDefaultPlayerModule` disabled, and legacy internals such as `GetControls()` may be gone.
  - Phase 2 makes it default Enabled in early 2027. Phase 3 removes the property and makes the new scripts mandatory in mid 2027.
  — [IAS Full Release](https://devforum.roblox.com/t/full-release-input-action-system-ias-newly-converted-player-scripts/4678416)
- IAS is described as a foundational pillar of SA and CCL. A commenter explains that, unlike ContextActionService, IAS instances give both server and client access to bindings, which prediction needs. Roblox encourages IAS for custom input needs but does not explicitly deprecate UserInputService or ContextActionService. — [IAS Full Release](https://devforum.roblox.com/t/full-release-input-action-system-ias-newly-converted-player-scripts/4678416)
- Known issues in the release thread:
  - Shift Lock combined with ProximityPrompts blocks camera turning.
  - Camera input is blocked while hovering active UI.
  - Typechecking problems.
  - Four UI buttons cannot be mapped to a Direction2D action; the developer concluded it is currently impossible.
  - A question about how to read the mobile thumbstick direction went unanswered.
  - Gyroscope support was requested.
  — [IAS Full Release](https://devforum.roblox.com/t/full-release-input-action-system-ias-newly-converted-player-scripts/4678416)
- IAS action types: `Bool`, `Direction1D`, `Direction2D`, `Direction3D` and `ViewportPosition`.
  - `Pressed` and `Released` fire for Bool actions only. `StateChanged` fires for every type.
  - `PressedThreshold` and `ReleasedThreshold` apply to analog triggers.
  - Touch binds through `UIButton`, using a GuiButton's press and release.
  - InputContext has `Priority` and `Sink`.
  - Each action should have gamepad, keyboard/mouse and touch bindings.
  - `InputActionLabel` (beta) shows the right glyph for the player's device. `PreferredBinding` is available.
  - The Input Action Manager (beta) flags missing and duplicate bindings.
  — [IAS docs](https://create.roblox.com/docs/input/input-action-system)
- An Input Action Manager Studio Beta followed the full release. (snippet) — [Input Action Manager post](https://devforum.roblox.com/t/studio-beta-input-action-manager/4737890)
- Under SA, read action state with `InputAction:GetState()` rather than listening to `StateChanged`, which can fire inaccurately during rollback. — [Studio Beta](https://devforum.roblox.com/t/studio-beta-build-fair-responsive-games-with-server-authority/4139157); [Community reference](https://github.com/MrChickenRocket/Server-Authority-Tutorial-Cakeman/blob/main/docs/reference/server-authority.md)
- Mobile docs:
  - Use `UserInputService.PreferredInput` to detect the input type. If a gamepad connects, treat it as primary and keep touch as a backup.
  - IAS is recommended for cross-platform input, including virtual buttons.
  - The default orientation is `LandscapeSensor`.
  - Haptics use `HapticEffect`: set `Type` (for example `GameplayCollision` or `UIClick`) and call `Play()`. Supported on most iPhone, Pixel and Galaxy devices.
  - `GyroscopeEnabled` and `AccelerometerEnabled` are exposed.
  — [Mobile input docs](https://create.roblox.com/docs/input/mobile)
- `DevTouchMovementMode` options (snippet):
  - `DynamicThumbstick` shows a thumbstick wherever the player touches the lower part of the screen, with jump on a separate button.
  - `Scriptable` turns off the default controls for fully custom movement.
  - `UserChoice` is the default.
  — [DevTouchMovementMode enum (snippet)](https://create.roblox.com/docs/reference/engine/enums/DevTouchMovementMode.md)
- The older mobile guide points to `ContextActionService:BindAction` for on-screen buttons. (snippet) — [Mobile input guide (snippet)](https://create.roblox.com/docs/scripting/input/mobile-input)
- `ScreenInsets` values (snippet):
  - `CoreUISafeInsets` is the default. It avoids the top bar and device cutouts.
  - `DeviceSafeInsets` avoids only device cutouts.
  - `TopbarSafeInsets` is also available.
  - `None` is discouraged.
  - `ScreenGui.ClipToDeviceSafeArea` defaults to true.
  — [ScreenInsets enum (snippet)](https://create.roblox.com/docs/reference/engine/enums/ScreenInsets); [ScreenGui ref (snippet)](https://create.roblox.com/docs/en-us/reference/engine/classes/ScreenGui.md)

### Inferences
- **IAS is the right default for a late-2026 project under any authority model.** It is generally available, it is the only input path SA accepts, default PlayerScripts must use it by mid 2027, and it shows device-correct prompts for free. Keep UserInputService only for what IAS lacks: custom virtual-joystick finger tracking and gyroscope.
- **Suggested action map:**
  - `Move`: Direction2D, bound to WASD, the left stick and the touch joystick.
  - `Look/Aim`: Direction2D, bound to mouse delta, the right stick and touch drag. Use Direction3D only if aim must feed an SA simulation.
  - `Boost`: Bool, bound to Shift, a gamepad shoulder or trigger, and a touch button.
  - `Bite`: Bool, bound to a mouse button, a gamepad face button and a touch button. Consider auto-bite on proximity for mobile.
  - Optional `Rise/Dive`: Direction1D.
  - Contexts: a high-priority `SwimContext` with Sink enabled, and a separate UI context.
- **Thumbstick on mobile.** A custom UI thumbstick reportedly cannot drive Direction2D through UI buttons. Two options:
  - Keep the default DynamicThumbstick from the IAS PlayerScripts and read their move vector. Verify how that is exposed, since the question went unanswered.
  - Track touches with UserInputService and push the vector into your swim code, or under SA into an InputAction, from RenderStepped.
  - Prototype both early.
- **Layout.** Keep the Boost and Bite buttons large and in the right-thumb zone inside `CoreUISafeInsets`. Add a haptic pulse on a successful eat. Auto-forward plus camera steering reduces the number of simultaneous touches, which suits small phones.

### Gaps
- None of the pages fetched give Roblox's guidance on touch button sizes or thumb zones.
- How to read the default mobile thumbstick under IAS went unanswered in the release thread.
- The API for setting InputAction values programmatically was seen only in a snippet, not on the English docs page.
- The start date of the IAS Client Beta was not verified.
- How `TouchControlsEnabled` interacts with the movement modes was not found.
