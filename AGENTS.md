# Shark Evolution: rules for AI agents

These rules apply to every AI agent working in this repo (Claude Code, Codex, Cursor and others). `CLAUDE.md` imports this file, so make changes here.

## The game

An original multiplayer Roblox game in the shark-eats-fish, grow-and-evolve genre: one large open ocean, hundreds of AI fish, player sharks that swim freely in 3D, mostly PvE eating with some PvP biting, about 10–20 players per server, and progression saved across sessions.

**Top priority: smooth on low/mid-end phones (2–3 GB RAM) and playable on bad mobile networks (150–300 ms round trip, jitter, packet loss).** Judge every change against that.

"Shark Evolution" is a working title. It is close to Ubisoft's trademark *Hungry Shark Evolution*, so never copy that game's name, logo, sharks, UI, maps or audio. Genre mechanics (eating to grow, boosting, biting) are fine. See [D14](wiki/decisions.md#d14-name-and-originality).

**Status (2026-10-30): not scaffolded.** There is no game code yet; the wiki describes the planned design. Update this line when that changes.

## The wiki is the source of truth

[`wiki/`](wiki/README.md) holds the exact, current description of every game mechanism: data layouts, formulas, constants, rates, budgets, which side is authoritative, and why.

1. Before changing how something works, read its wiki page and the related entries in [wiki/decisions.md](wiki/decisions.md).
2. Any commit that changes a mechanism also updates its wiki page (Spec and History) and adds a line to [wiki/CHANGELOG.md](wiki/CHANGELOG.md). The change isn't finished until the wiki matches the code.
3. Designing something new (growth curve, shark roster, a hazard)? Create its page from [wiki/_template.md](wiki/_template.md) and list it in [wiki/README.md](wiki/README.md).
4. If the code and the wiki disagree, the code is what runs. Work out which one is wrong, fix it, and note it in the page's History.
5. Don't silently deviate from a decision. If a decision's "Revisit when" condition is met, or new evidence contradicts it, update the decision entry first (status, date, reason), then change the code.

## Stack

Exact versions are pinned in [wiki/decisions.md](wiki/decisions.md) (D1–D3) until `rokit.toml` and `wally.toml` exist; after that, those files are the source of truth for versions.

- **Tools (Rokit):** Rojo, Wally, Lune, Blink, rodeo, luau-lsp, StyLua, Selene, Asphalt. No Aftman. Don't use Roblox's Script Sync on anything Rojo manages.
- **Packages (Wally):** jecs, ProfileStore (server only), vide, TopbarPlus, LemonSignal, Trove, t, and Jest Roblox for tests. Planck, Charm and charm-sync are optional.
- **Not used:** Lyra, crunchyroll, red-blox utilities, React-lua, Fusion, replecs, Matter.

## Architecture rules

Each rule links to the page with the details.

- The server decides who ate what, growth, currency, unlocks and purchases. Never trust a client for those. ([architecture](wiki/systems/architecture.md))
- Each client owns its own shark's physics. The server checks speed and position and pulls offenders back; it never simulates players. Roblox Server Authority is deferred. ([shark-movement](wiki/systems/shark-movement.md), D4)
- Fish are data on the server, never Instances. Most fish belong to "calm schools" whose positions both sides compute from a seed and server time. Clients draw all fish from pooled, anchored meshes. ([fish-simulation](wiki/systems/fish-simulation.md))
- Bites are claims that the server validates by rewinding the target's position within a capped window. ([eating-and-bites](wiki/systems/eating-and-bites.md))
- Continuous state goes over an UnreliableRemoteEvent as quantized buffers of at most 950 bytes with sequence numbers. Typed low-rate events go through Blink, behind a size and rate check that runs before decoding. ([networking](wiki/systems/networking.md))
- Encode shared state once per tick and reuse that buffer for every player who needs it. Never re-encode per player.
- No Humanoids for sharks or fish. No Terrain water. No EditableMesh for crowds. ([rendering](wiki/systems/rendering.md))
- Hot paths allocate nothing per frame: preallocated arrays and reused buffers.

## Code conventions

- `--!strict` in every file; zero type errors before committing.
- `--!native` only on hot numeric modules (fish simulation, codecs). Native code runs on servers and Android clients; iPhones interpret Luau, so size client work for the interpreter.
- A module that implements a wiki-documented mechanism starts with `-- Spec: wiki/systems/<page>.md`.
- Tunable numbers live in config modules, and the wiki page names the constant next to its value.
- Code that must behave identically on server and client lives in `src/shared/`.

## Layout (planned)

| Folder | Mounted at |
|---|---|
| `src/shared` | `ReplicatedStorage.Shared` |
| `src/server` | `ServerScriptService.Server` |
| `src/client` | `StarterPlayer.StarterPlayerScripts.Client` |
| `Packages` (Wally, shared realm) | `ReplicatedStorage.Packages` |
| `ServerPackages` (Wally, server realm, e.g. ProfileStore) | `ServerScriptService.ServerPackages` |

## Commands

Not scaffolded yet. Once it is, list the setup, sync, test and build commands here.

## Verifying changes

- Logic: Lune headless specs. Engine APIs: Jest Roblox through Open Cloud Luau Execution, in a separate test universe. Play mode: rodeo.
- Bad networks: Studio's Network Simulator ("Bad Connection (3G)" preset) and real phones on cellular.
- A performance claim needs a measurement on a real phone, recorded in [testing-and-performance](wiki/systems/testing-and-performance.md) with the date and device. Report p95, not averages.

## Working with Roblox Studio

- Rojo owns the files. Edit on disk, not in Studio; Studio edits to synced scripts get overwritten.
- Use Studio's built-in MCP server only to run code, playtest, read output and capture the screen.

## References

- [Tech stack research report (2026-10-30)](reports/Roblox%20shark%20game%20mobile%20tech%20stack.md): the evidence behind the decisions, with sources. It is a dated snapshot, so don't edit it; record changes in the wiki.
- `research_notes/Roblox shark game mobile tech stack/`: the six sets of research notes behind the report.
- `../SMASH`: the developer's earlier Roblox game, in a sibling folder. Reuse its tooling patterns (Lune spec harness, mobile control tests). Don't copy its 60 Hz prediction netcode, which grew too complex and missed its server frame budget.
