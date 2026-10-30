# Shark Evolution wiki

The exact, current description of how the game works, written so an AI agent (or a person) can change a mechanism without reverse-engineering it from code. **If something isn't written here, treat it as undecided.**

## Reading order

1. [AGENTS.md](../AGENTS.md): project rules.
2. [decisions.md](decisions.md): what was decided, why, and when to revisit it.
3. [systems/architecture.md](systems/architecture.md): how the pieces fit together.
4. The page for the mechanism you're changing.
5. [glossary.md](glossary.md) for any term you don't recognize.

## Pages

| Page | Covers | Status |
|---|---|---|
| [architecture](systems/architecture.md) | Who owns what, loop rates, data flow, place settings | planned |
| [networking](systems/networking.md) | Blink events, the state stream and its record format, interest management, budgets, guards | planned |
| [fish-simulation](systems/fish-simulation.md) | Fish tiers, closed-form schools, reacting fish, predators, server cost | planned |
| [eating-and-bites](systems/eating-and-bites.md) | Bite claims, server validation, rewind clamps, PvP limits | planned |
| [shark-movement](systems/shark-movement.md) | Swim physics, client ownership, server movement checks | planned |
| [input-and-controls](systems/input-and-controls.md) | Input Action System actions, mobile controls | planned |
| [rendering](systems/rendering.md) | Fog, streaming, lighting, fake water, fish rendering, client budgets | planned |
| [animation](systems/animation.md) | Shark rig and Animation Graph, procedural fish motion | planned |
| [persistence](systems/persistence.md) | Player profiles, migrations, purchases, leaderboards, analytics | planned |
| [ui](systems/ui.md) | vide menus, HUD updates, safe areas, touch sizes | planned |
| [testing-and-performance](systems/testing-and-performance.md) | Test layers, network and device testing, budgets, measurements, first milestone | planned |

Keep the Status column in sync with each page's front matter. Recent changes are listed in [CHANGELOG.md](CHANGELOG.md).

## Not designed yet

Nothing below has been decided, so none of it has a page. When you design one, create its page from [_template.md](_template.md) and add it to the table above.

- Growth and evolution: XP, size tiers, what each size can eat
- Shark roster and unlocks (original species and designs only, see [D14](decisions.md#d14-name-and-originality))
- Fish species and the food chain
- AI predator behaviour and other hazards
- PvP rules beyond bite validation
- Map: size, zones, depth bands, landmarks
- Economy: currencies, sources, sinks, prices
- Monetization: developer products and passes
- Audio
- Onboarding and tutorial
- Exploit handling beyond movement and bite checks
- Asset provenance list ([D14](decisions.md#d14-name-and-originality))

## How to write a page

The goal: another agent reads one page and knows exactly how the mechanism works, without opening the code.

**Structure.** Every page starts with front matter, then uses these sections in order: Summary, Spec, Invariants, Tests, Open questions, History. Extra sections, such as Measurements, go between Spec and Invariants. Copy [_template.md](_template.md).

| Front matter | Meaning |
|---|---|
| `status` | `planned` (designed, not built) · `in progress` (Spec says which parts exist) · `implemented` (Spec matches the code) · `deprecated` (kept for history; links its replacement) |
| `code` | Files that implement the mechanism |
| `decisions` | Related entries in decisions.md |
| `related` | Other wiki pages |
| `last_verified` | Date and commit when someone last checked the page against the code |

**The Spec is exact.** Byte layouts, formulas, the order of steps, edge cases, and which side is authoritative. Anything undecided goes under Open questions, never in the Spec.

**Every number has a unit and a tag.** A tag on a table column or list heading covers every value under it.

| Tag | Meaning |
|---|---|
| `[decided]` | A design choice. Change it by editing the page (and decisions.md, if it's listed there). |
| `[proposed]` | A detail suggested while planning that nobody has confirmed. Confirm or change it when implementing, then retag it. |
| `[target]` | A budget or goal that hasn't been measured. |
| `[measured YYYY-MM-DD, device or setup]` | An observed value. Link the test or capture. |
| `[engine]` | A Roblox limit, default or behaviour. Link the doc or source. |

Once code exists, name the constant that holds each tunable value, for example: `~10 Hz [decided] (FishConfig.TIER_B_HZ)`.

**Units:** studs; s or ms; Hz; bytes (B). Write bandwidth as B/s or KiB/s (1 KiB = 1,024 B).

**History** goes at the bottom of each page, newest first, one line per change: `YYYY-MM-DD — what changed — why`. Add the commit or PR if you know it.

**Also:**

- One page per mechanism. Link to other pages instead of repeating them, so each fact lives in one place.
- Add one line per change to [CHANGELOG.md](CHANGELOG.md).
- Use the terms in [glossary.md](glossary.md), and add new ones there.
- Split a page that grows past about 400 lines.
- Never rename or delete a page; leave a one-line pointer to its replacement.
- Evidence and sources belong in decisions.md or the research report, not in the Spec.
