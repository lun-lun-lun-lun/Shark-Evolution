# Roblox external-tooling toolchain and development workflow (state as of 2026-10-30)

Scope: toolchain managers, sync tools, package managers, runtimes and test runners, static tooling, CI/CD and deployment, AI-agent integration, and project layout for a new Rojo-based strict-Luau project. Networking (Blink), ECS, data, UI and rendering libraries are out of scope. They are mentioned only where they bear on tooling, such as which registry a library publishes to.

Method note: version numbers and dates come from the GitHub REST API (`/repos/{owner}/{repo}/releases`) and the Wally registry API (`api.wally.run`), both queried on 2026-10-30. Links point to the matching human-readable release pages. Many star counts and dates are from those API responses. DevForum dates come from the posts themselves where I fetched them; otherwise the notes flag them as coming from search-result metadata.

---

## 1. Toolchain managers: Rokit vs Aftman vs Foreman

### Takeaway
Rokit (rojo-rbx/rokit, latest **1.2.0, 2025-09-30**) is the de facto community standard. Rojo's own release notes list it as the first install method, and it reads `aftman.toml` and `foreman.toml` natively. **Aftman's repository is archived** (last release v0.3.0, 2024-05-22). Foreman is maintained by Roblox (v1.7.0, 2026-05-01) but leans toward Roblox's internal use. Keep Rokit and drop the project-local Aftman fallback.

### Cited Findings
- **Rokit latest = 1.2.0 (published 2025-09-30)**. It added `.tar.xz` artifact support and fixed tarball extraction. Before that came 1.1.1 (2025-08-31: prebuilt binaries built on Ubuntu 22.04 for better glibc compatibility) and 1.1.0 (2025-08-29: support for `x.y` versioned releases such as the Luau CLI, and the first release with native ARM64 on all major platforms). — [Rokit 1.2.0 release](https://github.com/rojo-rbx/rokit/releases/tag/v1.2.0); [Rokit 1.1.0 release](https://github.com/rojo-rbx/rokit/releases/tag/v1.1.0)
- Rokit 1.1.0 notes say: "It's been almost a year since Rokit 1.0.0 … Rokit 1.0.0 and above remain as stable as ever." 1.0.0 (2024-11-13) promised no breaking changes without a new major version. — [Rokit 1.1.0](https://github.com/rojo-rbx/rokit/releases/tag/v1.1.0); [Rokit 1.0.0](https://github.com/rojo-rbx/rokit/releases/tag/v1.0.0)
- Maintenance signal: the repo was last pushed 2026-09-27, with about 496 stars and 31 open issues. There has been no tagged release for about 13 months, which suggests a stable, low-churn tool. — [rojo-rbx/rokit](https://github.com/rojo-rbx/rokit)
- The Rokit README says it offers "Drop-in compatibility with projects that already use Foreman or Aftman". It is "Usable in any project, even projects using foreman.toml / aftman.toml, while Foreman / Aftman are incompatible with each other by design." It also says Foreman "is maintained by Roblox itself, and has angled itself towards internal uses at Roblox", and that Aftman "is maintained by a third party that is no longer interested in Roblox". — [Rokit README](https://github.com/rojo-rbx/rokit)
- Rokit's manifest discovery (source code): for each directory from the current one up through its ancestors, it checks `rokit.toml`, then `aftman.toml`, then `foreman.toml`. It then checks the home directories (`~/.rokit`, `~/.aftman`, `~/.foreman`). — [rokit lib/discovery/mod.rs](https://github.com/rojo-rbx/rokit/blob/main/lib/discovery/mod.rs)
- Known Rokit gotcha: issue #93 (open) says "If an aftman.toml or foreman.toml fail to parse, Rokit acts as if they do not exist". PR #114 to "Warn when compatibility manifests cannot be parsed" is still open (updated 2026-06-25). — [rokit #93](https://github.com/rojo-rbx/rokit/issues/93); [rokit PR #114](https://github.com/rojo-rbx/rokit/pull/114)
- Rojo's release notes give Rokit as the first install route ("Add a line to the `[tools]` section in `rokit.toml`: `rojo = "rojo-rbx/rojo@7.7.1"`"), ahead of GitHub binaries and `cargo install`. — [Rojo v7.7.1 release](https://github.com/rojo-rbx/rojo/releases/tag/v7.7.1)
- **Aftman: repository archived** (archived=true; last push 2025-07-09). The last release is **v0.3.0 (2024-05-22)**, which switched to rustls and added aarch64 builds. — [LPGhatguy/aftman](https://github.com/LPGhatguy/aftman); [Aftman v0.3.0](https://github.com/LPGhatguy/aftman/releases/tag/v0.3.0)
- **Foreman: v1.7.0 (2026-05-01)** added an `auth-secure` subcommand. v1.6.5 (2026-01-13) added paging of GitHub releases, and v1.6.4 (2025-05-05) added a linux-arm64 build. Commits come from Roblox staff. — [Foreman v1.7.0](https://github.com/Roblox/foreman/releases/tag/v1.7.0); [Foreman v1.6.5](https://github.com/Roblox/foreman/releases/tag/v1.6.5)
- Third-party tools now advertise Rokit installs. For example, jest-roblox-cli documents `rokit add christopher-buss/jest-roblox-cli` alongside `mise use github:christopher-buss/jest-roblox-cli`. — [jest-roblox-cli README](https://github.com/christopher-buss/jest-roblox-cli)
- CI action: `CompeyDev/setup-rokit@v0.2.1` installs Rokit and the project's tools. It accepts a project dir containing "a `rokit.toml`, `aftman.toml`, or `foreman.toml`", has optional binary caching, and uses `github.token` by default to avoid rate limits. — [setup-rokit README](https://github.com/CompeyDev/setup-rokit)

### Inferences
- "Is Rokit the community standard?" Effectively yes. It lives under the rojo-rbx org, Rojo's docs list it first, Aftman is archived, and Foreman is Roblox-internal-leaning. Rokit has about 496 stars vs Aftman's 196 (archived) and Foreman's 263.
- Running Rokit and Aftman side by side:
  - **Redundant:** Rokit already reads `aftman.toml`, and checks `rokit.toml` first in the same directory, so a second manifest adds nothing for Rokit users. Keeping two manifests pinned to the same Rojo version invites version drift.
  - **Shim conflicts (not verified against docs):** each manager installs its own `rojo` shim in its own bin directory on PATH. Whichever directory comes first on PATH handles `rojo`, `wally`, `lune` and so on. This includes calls made by editors, such as luau-lsp's `rojo sourcemap --watch` autogeneration.
  - **Silent failures:** a malformed compatibility manifest is ignored without warning (rokit #93).
- **Verdict: keep Rokit** (run `rokit self-update` to reach 1.2.0). **Remove the Aftman fallback.** If a collaborator cannot use Rokit, `foreman.toml` is the maintained fallback, but none is needed. mise is an emerging general-purpose alternative, though it is not Roblox-specific.

### Gaps
- I found no official documentation of what happens when both the Aftman and Rokit shim directories are on PATH. The precedence behavior above is inferred.
- No Rokit release after 1.2.0 was found. Whether a 1.3 is planned is unknown.

---

## 2. Rojo (latest version, 2025-2026 changes), Studio Script Sync, and Argon

### Takeaway
**Rojo 7.7.1 (2026-10-02)** is current, and there is no evidence of a Rojo 8. Rojo 7.7.0 (2026-07-02) added `rojo syncback`, a one-shot conversion of a place file to files. It also moved the live-sync protocol from HTTP long-polling and JSON to **WebSockets and MessagePack**, and added `.jsonc` support and DNS-rebinding protection. Roblox's native **Studio Script Sync reached full release on 2026-06-17**. It is an editor-interop feature that treats Studio as the source of truth for scripts only. It cannot build places from files in CI or manage non-script instances, so it **does not replace Rojo** for a Wally/CI/headless-test workflow. Argon (2.0.29) remains a smaller alternative.

### Cited Findings
**Rojo versions and status**
- **v7.7.1 — 2026-10-02** is the "first patch of 7.7.0. It's likely that another is coming relatively soon." It fixed "a myriad of problems caused by paths being prefixed with `\\?\` on Windows" and made the plugin handle blocked requests to api.github.com more gracefully. The plugin installs via `rojo plugin install`, an attached `Rojo.rbxm`, or the Creator Store. — [Rojo v7.7.1](https://github.com/rojo-rbx/rojo/releases/tag/v7.7.1)
- **v7.7.0 — 2026-07-02** (release candidate v7.7.0-rc.1 was published 2025-11-27):
  - **Syncback:** `rojo syncback path/to/project --input file.rbxl` pulls instances from a place file onto disk, but only beneath nodes in the project file. It is configured by a new `syncbackRules` project field (`ignoreTrees`, `ignorePaths`, `ignoreProperties`, `syncCurrentCamera`, `syncUnscriptable`). Actors and remote/bindable event and function variants sync back as JSON.
  - **Networking:** `rojo serve` uses WebSockets instead of long polling, and "Rojo's web API now uses MessagePack instead of JSON for all endpoints".
  - **Formats:** `.jsonc` is accepted anywhere `.json` is, `inf` and `nan` property values sync, and `init.plugin.luau` is supported.
  - **Ignore rules:** `globIgnorePaths` supports gitignore-style negation.
  - **Security:** `rojo serve` now validates `Host`/`Origin` headers against DNS rebinding, restricts `/api/open` to local clients, and accepts extra hosts via `--allowed-hosts` or a project `serveAllowedHosts` field.
  - **Robustness:** sibling-order and same-name matching fixes, and clear errors instead of panics.
  — [Rojo v7.7.0](https://github.com/rojo-rbx/rojo/releases/tag/v7.7.0); [Rojo v7.7.0-rc.1](https://github.com/rojo-rbx/rojo/releases/tag/v7.7.0-rc.1)
- Rojo 7.7's WebSocket transport relies on Studio's WebSocket support, and the 7.7.0 notes link to Roblox's "WebSockets support in Studio is now available" announcement. — [Rojo v7.7.0](https://github.com/rojo-rbx/rojo/releases/tag/v7.7.0); [DevForum: WebSockets in Studio](https://devforum.roblox.com/t/websockets-support-in-studio-is-now-available/4021932)
- **v7.6.1 — 2025-11-07** added JSON comments and trailing commas in project, meta and model JSON files, and added `sourcemap.json` to the default `.gitignore`. **v7.6.0 — 2025-10-11** added a delete-and-recreate fallback so MeshParts and Unions can sync; the notes warn this "will break properties that reference them". It also added plugin auto-reconnect and a `--skip-git` flag for `rojo init`. — [Rojo v7.6.1](https://github.com/rojo-rbx/rojo/releases/tag/v7.6.1); [Rojo v7.6.0](https://github.com/rojo-rbx/rojo/releases/tag/v7.6.0)
- Maintenance: the repo was last pushed 2026-10-06, with about 1,835 stars and 221 open issues. — [rojo-rbx/rojo](https://github.com/rojo-rbx/rojo)
- Open issues worth knowing:
  - #1300 (open, updated 2026-09-14): "Rojo serve on 7.7.0 silently stops detecting newly created files (7.6.1 unaffected)". PR #1341, "Keep watching directories recreated at a removed path", was closed 2026-10-16, after 7.7.1 shipped.
  - Live two-way sync is still not stable: #1273 "Made two way Source sync stable" is open, and PR #1325 "resolve two-way sync feedback loops and cursor jumping via echo suppression" is open as of 2026-09-28.
  - #1317 "Expose RunContext in the sourcemap" is open.
  — [rojo #1300](https://github.com/rojo-rbx/rojo/issues/1300); [rojo PR #1341](https://github.com/rojo-rbx/rojo/pull/1341); [rojo #1273](https://github.com/rojo-rbx/rojo/issues/1273); [rojo PR #1325](https://github.com/rojo-rbx/rojo/pull/1325); [rojo #1317](https://github.com/rojo-rbx/rojo/issues/1317)
- No Rojo 8.x tag, pre-release or issue discussion was found. The latest tag is v7.7.1, and an issue search for "rojo 8" returned nothing relevant. — [Rojo releases](https://github.com/rojo-rbx/rojo/releases)

**Studio Script Sync (native)**
- Timeline: an Early Preview thread came first. Public beta was announced in November 2025, listed in the DevForum weekly recap for Nov 10–14, 2025. **"[Full Release] Studio Script Sync" was posted 2026-06-17.** — [Early Preview](https://devforum.roblox.com/t/early-preview-studio-script-sync/3257548); [Beta post](https://devforum.roblox.com/t/studio-script-sync-now-in-beta/4065468); [Weekly recap Nov 10–14 2025](https://devforum.roblox.com/t/weekly-recap-november-10-14-2025/4071175); [Full Release post](https://devforum.roblox.com/t/full-release-studio-script-sync/4688454)
- What it syncs: only Script, LocalScript, ModuleScript and Folder instances, in both directions; other instances are ignored. Packages can be synced, but the PackageLink is not written to disk. The docs warn that "Scripts with attributes or tags aren't handled correctly and can cause data loss". — [Script Sync docs](https://create.roblox.com/docs/scripting/sync)
- File mapping: `name.luau` is a ModuleScript. `name.server.luau` and `name.client.luau` are Scripts with Server and Client RunContext. `.local.luau` is a LocalScript, `.legacy.luau` is legacy RunContext, `.plugin.luau` is plugin RunContext, and `name/init.*.luau` is a script with children. The extension is configurable. — [Script Sync docs](https://create.roblox.com/docs/scripting/sync)
- Limits: 10,000 scripts per top-level synced instance and 128 top-level synced instances. Team Create is supported, with collaborator-sync icons, but concurrent edits of the same script can overwrite each other. The debugger cannot be driven from an external editor. — [Script Sync docs](https://create.roblox.com/docs/scripting/sync); [Full Release post](https://devforum.roblox.com/t/full-release-studio-script-sync/4688454)
- The full release added a conflict-resolution dialog ("Keep Studio" or "Keep Disk") listing added, modified and deleted scripts on each side. Users asked for per-file choices, which are not confirmed as planned. — [Full Release post](https://devforum.roblox.com/t/full-release-studio-script-sync/4688454)
- Positioning: the FAQ calls Rojo a build-and-compose tool that treats the file system as the source of truth, and says Script Sync is about interoperability. The docs direct users who want "to version-control the whole project or use the file system as the source of truth" to third-party tools such as Rojo. Script Sync generates **no sourcemap**. For LSP support the docs recommend luau-lsp plus the "Luau Language Server Companion" Studio plugin. — [Script Sync docs](https://create.roblox.com/docs/scripting/sync); [Full Release post](https://devforum.roblox.com/t/full-release-studio-script-sync/4688454)
- Reported issues: a commenter says a disk resync deletes package instances under scripts. Others report errors for incompatible instances, unsupported properties, duplicate names and names containing "/". — [Full Release post](https://devforum.roblox.com/t/full-release-studio-script-sync/4688454)
- Status conflict: the DevForum calls Script Sync a full release (2026-06-17). However, Roblox's "Build with a coding harness" doc, updated 2026-10-29, still says to enable "Script Sync (beta)" under File → Beta Features. — [Coding harness doc](https://create.roblox.com/docs/ai/coding-harness); contradicted by [Full Release post](https://devforum.roblox.com/t/full-release-studio-script-sync/4688454)

**Argon**
- **Argon 2.0.29 — 2026-05-19** fixed WeakDom crash bugs and sorts JSON properties alphabetically. 2.0.28 (2026-03-04) added Linux and Windows ARM builds and fixed code execution. 2.0.25 (2025-07-17) writes empty scripts on syncback. 2.0.24 (2025-04-28) added Content properties and auto-detection of Wally `ServerPackages`/`DevPackages`. The repo was last pushed 2026-07-01, with about 151 stars. — [Argon 2.0.29](https://github.com/argon-rbx/argon/releases/tag/2.0.29); [Argon 2.0.28](https://github.com/argon-rbx/argon/releases/tag/2.0.28); [Argon 2.0.25](https://github.com/argon-rbx/argon/releases/tag/2.0.25); [Argon 2.0.24](https://github.com/argon-rbx/argon/releases/tag/2.0.24)

### Inferences
- **Verdict: upgrade Rojo 7.7.0 → 7.7.1.** The developer is on Windows, and 7.7.1 specifically fixes Windows `\\?\` path problems. Upgrade the Studio plugin at the same time, since the 7.7 networking protocol changed. If new files stop being picked up during `rojo serve` (#1300), restart serve and watch for 7.7.2.
- Script Sync does not replace Rojo for this project:
  - It cannot produce a place from files in CI (`rojo build`).
  - It ignores non-script instances, so `.model.json`/`.meta.json`, RemoteEvents and attributes are not covered.
  - Studio, not git, is the source of truth.
  - It emits no sourcemap, and luau-lsp's sourcemap-driven strict typing works best with Rojo.
  - It has no notion of Wally `Packages`/`ServerPackages` mounts.
  Script Sync suits teams that build mostly in Studio and want an external or AI editor.
- Do not mix Script Sync and Rojo on the same instance trees. Both write the same scripts, and Script Sync's conflict dialog and Rojo's reconciliation would fight. This is inferred; there is no source on coexistence.
- `rojo syncback` is useful once, to import an existing Studio-built map or prefab place into the repo. Its `ignoreProperties`/`ignoreTrees` rules let you exclude large or binary map content. Live two-way sync remains experimental.
- Argon has no compelling advantage now that Rojo has syncback, and it has a much smaller community. **Do not switch.**

### Gaps
- No official statement from the Rojo maintainers on Rojo's future relative to Script Sync was found.
- Whether #1300 is fixed in 7.7.1 is unverified.
- Whether the 7.7 CLI rejects pre-7.7 plugins with a clear error is not confirmed in the release notes.

---

## 3. Package managers: Wally vs pesde (and library publishing locations)

### Takeaway
**Wally is still the de facto Roblox registry, and it is maintained again.** v0.4.0-alpha.0 (2026-09-26) and alpha.1 (2026-10-12) are the first releases since 0.3.2 (June 2023), and the registry API is live. **pesde (v0.7.4, 2026-09-09) is active and capable**: it handles Roblox, Lune and Luau targets and can consume Wally packages. However, most libraries on this project's likely list publish officially only to Wally (and npm for roblox-ts). On pesde, jecs exists only as a third-party "minimal copy", and Lyra, ProfileStore and Charm were not found. **Keep Wally 0.3.2.**

### Cited Findings
- **Wally v0.4.0-alpha.1 — 2026-10-12 (pre-release)**: "Removed `--locked` flag (#272); Upgrade to Rust 2024 (#273); Fixed exit code (#274)". — [Wally 0.4.0-alpha.1](https://github.com/UpliftGames/wally/releases/tag/v0.4.0-alpha.1)
- **Wally v0.4.0-alpha.0 — 2026-09-26 (pre-release)**:
  - added a `--locked` install flag;
  - added `homepage`/`repository` manifest fields;
  - "Improved lockfile formatting for better text diffs **lockfiles will change**";
  - fixed `cargo install`;
  - added prebuilt binaries for Windows, macOS and Linux on x86_64 and aarch64.
  — [Wally 0.4.0-alpha.0](https://github.com/UpliftGames/wally/releases/tag/v0.4.0-alpha.0)
- **Wally v0.3.2 — 2023-06-05** remains the latest stable release. The repo was last pushed 2026-10-12, with about 506 stars and 88 open issues. — [Wally v0.3.2](https://github.com/UpliftGames/wally/releases/tag/v0.3.2); [UpliftGames/wally](https://github.com/UpliftGames/wally)
- **pesde v0.7.4+registry.0.2.3 — 2026-09-09** updated full-moon "to support new syntax". Earlier releases:
  - 0.7.3 (2026-03-18): "Type functions are now re-exported by dependency linkers".
  - 0.7.2 (2025-12-26): "Use string requires for Roblox packages if not originating from Wally".
  - 0.7.0 (2025-08-05): dev-only installs, `PESDE_HOME`, and engines.
  The repo was last pushed 2026-10-27, with about 130 stars. — [pesde releases](https://github.com/pesde-pkg/pesde/releases); [pesde v0.7.4](https://github.com/pesde-pkg/pesde/releases/tag/v0.7.4%2Bregistry.0.2.3)
- pesde ↔ Wally interop: declare `[wally_indices] default = "https://github.com/UpliftGames/wally-index"` and depend on `foo = { wally = "acme/package", version = "^1.0.0" }`, or run `pesde add wally#acme/package`. "Proper type support requires a `sourcemap_generator` script". Git, workspace and path dependencies are also supported. — [pesde docs: dependencies](https://docs.pesde.dev/guides/dependencies/)
- pesde ships a scripts package for Rojo projects: `pesde/scripts_rojo` 0.2.0 (2025-09-22). The pesde index root lists about 190 scope directories as of 2026-10-30. — [pesde index](https://github.com/pesde-pkg/index); [pesde/scripts_rojo entry](https://github.com/pesde-pkg/index/blob/main/pesde/scripts_rojo)
- Where libraries publish (Wally registry API and repo manifests, checked 2026-10-30):

  | Library | Wally (newest version on api.wally.run) | pesde | Other |
  |---|---|---|---|
  | jecs | `ukendio/jecs` 0.11.0 (73 versions) | `marked/jecs` 0.11.0 (2026-06-26), described as "A minimal copy of jecs published on the official pesde registry", repo `git.devmarked.win/marked/jecs-pesde` (third-party mirror) | npm `@rbxts/jecs` |
  | vide | `centau/vide` 0.4.1 | `centau/vide` 0.4.0 (2026-01-17), official `pesde.toml` in repo | — |
  | Planck | `yetanotherclown/planck` (newest listed 0.3.0-rc.2) | not found at `yetanotherclown/planck` | npm (roblox-ts build) |
  | Lyra | `paradoxum-games/lyra` 0.6.0 | not found | — |
  | ProfileStore | `lm-loleris/profilestore` 1.0.3, realm = "server" | not found | — |
  | Charm | `littensy/charm` 0.11.1 | not found | npm `@rbxts/charm` |
  | Jest Roblox | `roblox/jest` 3.20.1 | not found | Creator Store |

  — [Wally API: ukendio/jecs](https://api.wally.run/v1/package-metadata/ukendio/jecs); [jecs wally.toml](https://github.com/ukendio/jecs/blob/main/wally.toml); [pesde index: marked/jecs](https://github.com/pesde-pkg/index/blob/main/marked/jecs); [vide repo (wally.toml + pesde.toml)](https://github.com/centau/vide); [pesde index: centau/vide](https://github.com/pesde-pkg/index/blob/main/centau/vide); [planck wally.toml](https://github.com/YetAnotherClown/planck/blob/main/wally.toml); [lyra wally.toml](https://github.com/paradoxum-games/lyra/blob/main/wally.toml); [ProfileStore wally.toml](https://github.com/MadStudioRoblox/ProfileStore/blob/main/wally.toml); [charm repo](https://github.com/littensy/charm); [Jest Roblox README](https://github.com/Roblox/jest-roblox)
- Blink's CLI (out of scope, networking) is published to pesde as a Lune-target binary package. The latest entry is 0.18.9 (2026-09-19); SMASH pinned 0.18.8. — [pesde index: 1axen/blink](https://github.com/pesde-pkg/index/blob/main/1axen/blink); [blink pesde.toml](https://github.com/1Axen/blink/blob/main/pesde.toml)
- Lute, Roblox's official standalone runtime, is building its own package publishing. Nightly v1.0.1-nightly.20261029/20261030 merged "Publish PR1–PR4 … implement publish call to registry" and added `loom.config.luau` to batteries. The Lute README says batteries libraries will be published "as independent packages once a good dependency management solution is in place". — [Lute nightly 20261030](https://github.com/luau-lang/lute/releases/tag/v1.0.1-nightly.20261030); [Lute nightly 20261029](https://github.com/luau-lang/lute/releases/tag/v1.0.1-nightly.20261029); [Lute README](https://github.com/luau-lang/lute)

### Inferences
- **Verdict: keep Wally 0.3.2 for the new game.** The libraries in the developer's likely stack, including Roblox's official Jest packages (July 2026), publish to Wally first.
- Moving to pesde would mean:
  - rewriting manifests;
  - relying on pesde's Wally-compat layer for most dependencies;
  - adding a `sourcemap_generator` script for types;
  - adapting the Rojo project to pesde's install folders;
  - reworking the existing Wally-based CI.
  All of that buys little today.
- Do not adopt Wally 0.4.0 alpha mid-project. It changes the lockfile format, and the `--locked` flag was added in alpha.0 and removed in alpha.1, which shows the CLI surface is still moving. Re-evaluate at 0.4.0 stable: prebuilt aarch64 binaries and better lockfile diffs are welcome.
- Watch Lute's emerging registry and package tooling as the likely long-term "official" Luau package story. This is speculative: there is no announced Roblox-project support yet.

### Gaps
- Who is maintaining Wally in 2026, and when 0.4.0 stable is due: no announcement found.
- pesde's Roblox install folder layout and linker structure were not verified in this session.
- Planck's pesde status is unclear. The `yetanotherclown` scope exists in the pesde index, but no `planck` entry was found at the expected path.

---

## 4. Lune, Lute, test frameworks, and running tests in real engine contexts

### Takeaway
**Lune 0.11.1 (2026-10-21)** is current. It is a security-fix release on top of 0.11.0 (2026-10-11), which also fixed `CFrame.lookAt` rotations, a bug that matters for 3D swimming math in headless tests. **Lute 1.0.0** (Roblox's official runtime) went stable on 2026-04-16 and is gaining adoption for scripts and type tests, but Lune's `@lune/roblox` remains the headless option that models Roblox instances and datatypes. For engine-context tests, **Roblox's official Jest Roblox (3.20.1, on Wally since 2026-07-30)** has replaced TestEZ (archived) and jsdotlua/jest-lua (stale since 2024-12). It runs on Roblox servers from CI via the **Open Cloud Luau Execution API**. The developer's **rodeo** (now **1.5.1**) and the new **jest-roblox-cli** (0.4.10) cover local Studio-context runs. run-in-roblox is dead.

### Cited Findings
**Lune**
- **v0.11.1 — 2026-10-21** "addresses a security vulnerability with HTTPS certificates", and upgrading is suggested if you use `net.request` with untrusted or dynamic input (RUSTSEC-2026-0382). The note also says it fixes a regression in `@lune/roblox` datatype constructors with fractional args. That note refers to "Lune 0.12", which looks like a typo in the release text. — [Lune 0.11.1](https://github.com/lune-org/lune/releases/tag/v0.11.1); [RUSTSEC-2026-0382](https://rustsec.org/advisories/RUSTSEC-2026-0382)
- **v0.11.0 — 2026-10-11**:
  - Breaking change: `net.serve` handles use `handle:stop()`.
  - Added `fs.metadata().size` and `Enum.FromName`/`FromValue`.
  - **Fixed `CFrame.lookAt`, `CFrame.lookAlong` and `CFrame.new(pos, lookAt)` producing incorrect rotations when not rotating purely around the Y axis.**
  - Fixed `Color3:ToHex` rounding, and made `Enum`/`EnumItem` identities stable so they work as table keys.
  - Updated to Luau 0.740 and the rbx-dom database 0.741, and moved to mlua 0.12.
  — [Lune 0.11.0](https://github.com/lune-org/lune/releases/tag/v0.11.0)
- **v0.10.5 — 2026-07-02** added `QueryDescendants` and `roblox.registerClass`/`registerService` to the roblox library. **v0.10.4 — 2025-10-14**, the version SMASH pinned, updated to Luau 0.694 and rbx-dom 0.694 and added jsonc in `serde`. The repo was last pushed 2026-10-21, with about 974 stars. — [Lune 0.10.5](https://github.com/lune-org/lune/releases/tag/v0.10.5); [Lune 0.10.4](https://github.com/lune-org/lune/releases/tag/v0.10.4)

**Lute**
- **Lute v1.0.0 — 2026-04-16**, "our first stable release version of Lute", added `luau.requires`, a WebSocket client and server, and `lute transform` docs. Nightlies continue (v1.0.1-nightly.20261030, which updated to Luau 0.744 on 2026-10-28). The README describes a general-purpose runtime (`lute` C++ libs, a `std` Luau standard library, and `batteries`). Roblox is "working within Roblox to make `std` a shared interface across our Luau runtimes". — [Lute v1.0.0](https://github.com/luau-lang/lute/releases/tag/v1.0.0); [Lute nightly 20261028](https://github.com/luau-lang/lute/releases/tag/v1.0.1-nightly.20261028); [Lute README](https://github.com/luau-lang/lute)
- Adoption signals: Charm's repo runs `"test": "lute run test"`, `"build": "lute run build"`, and `"check": "lute run check"`. jest-roblox-cli runs Luau type tests (`.spec-d.luau`) "when `lute` is on your PATH". rodeo supports `--adapter lune|lute`. — [charm package.json](https://github.com/littensy/charm); [jest-roblox-cli README](https://github.com/christopher-buss/jest-roblox-cli); [rodeo v1.5.0-rc.15](https://github.com/rodeo-rbx/rodeo/releases/tag/v1.5.0-rc.15)

**Test frameworks**
- **TestEZ: archived** (archived=true; last release v0.4.2 on 2022-01-05; last push 2024-03-05). — [Roblox/testez](https://github.com/Roblox/testez)
- **jsdotlua/jest-lua: v3.10.0 (2024-12-23)**, no activity since. It is still on Wally as `jsdotlua/jest` 3.10.0 and used as a dev-dependency by Planck and Lyra. — [jest-lua v3.10.0](https://github.com/jsdotlua/jest-lua/releases/tag/v3.10.0); [Wally API: jsdotlua/jest](https://api.wally.run/v1/package-metadata/jsdotlua/jest)
- **Roblox/jest-roblox (official, open source)** is "a Roblox port of Jest v27.4.7". It "can run within Roblox itself, including via Roblox's OCALE (Open Cloud API for Luau Execution) for testing on CI systems", and Roblox uses it for apps, core scripts and Studio plugins.
  - Install: `Jest = "roblox/jest@=3.20.0"` and `JestGlobals = "roblox/jest-globals@=3.20.0"` as Wally dev-dependencies, or from the Creator Store.
  - Tests are `.spec.luau` files under `__tests__/`.
  — [Jest Roblox README](https://github.com/Roblox/jest-roblox)
- Jest Roblox changelog:
  - **3.20.1 (2026-08-30)**: bumped luau-regexp.
  - **3.20.0 (2026-07-30)**: "now published to Wally as 34 packages under the `roblox/*` scope".
  - 3.19.0 (2026-06-18): `jest.retryTimes`.
  - 3.18.0 (2026-06-16): removed the LuauPolyfill dependency and emittery, giving a speedup.
  - Unreleased: string requires resolve via `ScriptService:ResolveModulePath` "when the engine supports it … (including `.config` aliases)".
  Wally's API lists `roblox/jest` 3.20.1 as newest. — [Jest Roblox CHANGELOG](https://github.com/Roblox/jest-roblox/blob/master/CHANGELOG.md); [Wally API: roblox/jest](https://api.wally.run/v1/package-metadata/roblox/jest)
- **jest-roblox-cli** (christopher-buss): **v0.4.10 — 2026-10-25**; the repo was created 2026-03-17 and has about 31 stars.
  - Runs "roblox-ts and Luau tests inside Roblox", with "Three backends: Open Cloud (remote), Studio (attached, local), and Studio CLI (self-launched headless Studio, local)".
  - Offers source-mapped errors, in-process Luau coverage, and formatters for human, agent, JSON and GitHub Actions output.
  - Installs via npm `@isentinel/jest-roblox` or as a standalone binary via Rokit.
  - Frequent breaking changes (for example, in 0.4.9 and 0.4.10).
  — [jest-roblox-cli v0.4.10](https://github.com/christopher-buss/jest-roblox-cli/releases/tag/v0.4.10); [jest-roblox-cli README](https://github.com/christopher-buss/jest-roblox-cli)

**Engine-context execution**
- **Open Cloud Luau Execution API** ("OCALE"):
  - Endpoints: `POST /cloud/v2/universes/{u}/places/{p}/luau-execution-session-tasks`, or `…/versions/{v}/luau-execution-session-tasks` to target a version. Poll the Get Task endpoint and list logs separately.
  - Scopes: `universe.place.luau-execution-session:write` (and `:read`).
  - Limits:
    - **5-minute default timeout** (configurable `timeout`);
    - **at most 10 incomplete tasks per place** (HTTP 429 beyond that);
    - 4 MB script and 4 MB return value;
    - 450 KB of retained logs;
    - task info kept for 24 h;
    - binary input up to 100 MiB and binary output up to 256 MiB;
    - rate limits of **5/min** for create task, 200/min for get task, and 45/min for list logs.
  - Execution: runs **headless against a place version**. "Physics simulation does not run, and server and local scripts do not run automatically". DataModel changes are not persisted unless you call `SavePlaceAsync`. Scripts **can touch real cloud data such as DataStores**.
  - Docs list automated testing as a primary use case.
  — [Luau Execution reference](https://create.roblox.com/docs/cloud/reference/features/luau-execution)
- Roblox's referenced CI demo, `Roblox/place-ci-cd-demo`, is **archived** (created 2024-09-03, last push 2024-09-17). The binary-payloads example is also archived. — [place-ci-cd-demo](https://github.com/Roblox/place-ci-cd-demo); [open-cloud-execution-binary-payloads-example](https://github.com/Roblox/open-cloud-execution-binary-payloads-example)
- Community OCALE wrappers:
  - `rbxcloud` v0.17.0 (2025-03-23) added the Luau execution API and has no release since.
  - `Dekkonot/opencloud-luau-execute-lune` is "A thin wrapper … written for Lune" (last push 2025-09-24).
  - `thegamerbay/roblox-luau-execution-action` is a composite GitHub Action with about 2 stars, so adoption is low.
  — [rbxcloud v0.17.0](https://github.com/Sleitnick/rbxcloud/releases/tag/v0.17.0); [opencloud-luau-execute-lune](https://github.com/Dekkonot/opencloud-luau-execute-lune); [roblox-luau-execution-action](https://github.com/thegamerbay/roblox-luau-execution-action)
- **rodeo** (rodeo-rbx/rodeo, created 2025-11-03): **v1.5.1 — 2026-10-27** fixed output ordering. **v1.5.0 — 2026-10-26** is "The first stable release since 1.3.0".
  - Upgrade note: "**The CLI, the serve and the Studio plugin must be the same build.**"
  - Breaking change (rc.15): `@lune/*` and `@lute/*` requires need `--adapter lune|lute`.
  - Capabilities: runs code in any Studio environment (`--mode edit|run|test|play`, `--dom edit|server|client`, `--context plugin|server|client|mcp|cmd`), opens, runs and closes places, pipes stdio, exports and imports models, and "bakes" runtime data to `.luau`.
  - "macOS and Windows are fully supported. Linux currently is not."
  — [rodeo v1.5.1](https://github.com/rodeo-rbx/rodeo/releases/tag/v1.5.1); [rodeo v1.5.0](https://github.com/rodeo-rbx/rodeo/releases/tag/v1.5.0); [rodeo v1.5.0-rc.15](https://github.com/rodeo-rbx/rodeo/releases/tag/v1.5.0-rc.15); [rodeo README](https://github.com/rodeo-rbx/rodeo)
- **run-in-roblox**: last release v0.3.0 (2020-07-20), last push 2024-03-06. — [rojo-rbx/run-in-roblox](https://github.com/rojo-rbx/run-in-roblox)

### Inferences
- **Lune: upgrade 0.10.4 → 0.11.1.** It fixes a security advisory, and the 0.11.0 `CFrame.lookAt`/`lookAlong` fix means any SMASH-era headless tests of non-yaw rotations were validating wrong math under 0.10.4. That matters for free 3D shark swimming. Expect small breakages: `handle:stop()`, `Color3:ToHex` lowercase output, and Enum identity changes.
- **Keep the Lune harness (lunetest.luau) as the fast, offline layer** for deterministic logic: growth and evolution formulas, bite and damage resolution, AI steering math, and save-data migrations. Do not migrate it to Lute now. Lute has no documented Roblox-instance library equivalent to `@lune/roblox`.
- **Add an engine-context layer with Jest Roblox 3.20.1** (Wally `roblox/jest` as a dev-dependency):
  - Run it on CI via OCALE against a dedicated test place in a separate test universe, because OCALE scripts can hit real DataStores.
  - OCALE has no physics and no auto-running scripts, so keep it to module-level tests using real engine types and services.
  - Mind the 5 creates/min and 10 concurrent tasks/place limits. Batch all specs into one task.
- **For play-mode, client/server and physics tests**, use rodeo 1.5.1 locally (`--mode test --context server/client`), or jest-roblox-cli's Studio backends. Both need Windows or macOS with Studio installed, so they cannot run on stock Linux GitHub runners.
- **rodeo: upgrade 1.0.1 → 1.5.1**, and reinstall the plugin at the same build. The README badges reference `revvy02/rodeo`, so the repo moved to the `rodeo-rbx` org; update `rokit.toml` to `rodeo-rbx/rodeo@1.5.1`.
- TestEZ: replace with Jest Roblox if any TestEZ remains. Avoid jsdotlua/jest-lua for new work.
- jest-roblox-cli is promising: OCALE plus Studio backends, coverage, and an agent output formatter. But it is about 7 months old and breaks often, so pin it exactly if adopted.

### Gaps
- Whether OCALE supports HttpService, MessagingService or MemoryStore is not documented beyond "DataStores".
- I found no adopted, officially maintained GitHub Action for OCALE; Roblox's demo is archived.
- Lute's support for Roblox place and instance manipulation was not verified beyond its README.

---

## 5. Static tooling: luau-lsp, StyLua, Selene, the new type solver, darklua

### Takeaway
All three core static tools shipped in Sept–Oct 2026: **luau-lsp 1.70.1** (2026-09-27), **StyLua 2.6.0** (2026-10-10) and **Selene 0.32.0** (2026-10-01). They track Luau's weekly releases (**Luau 0.744**, 2026-10-23), including new syntax (`const`, `if local`, `f<<T>>()`). The **new type solver is the default only for nocheck and nonstrict scripts** (general release 2025-11-20; beta toggle removed 2026-01-07). **Strict-mode projects stay on the old solver unless `Workspace.UseNewLuauTypeSolver = Enabled`.** The old solver is promised "through 2026", and luau-lsp still ships the new solver as an opt-in "preview" flag. **darklua 0.19.0** (2026-06-10) is optional for a game.

### Cited Findings
**luau-lsp**
- **1.70.1 — 2026-09-27**: synced to Luau 0.740, shows `const` in hover, fixed Roblox string-require auto-imports with `.luaurc` aliases, and more. **1.70.0 — 2026-09-20** added `luau-lsp.analyzeLuaFiles` and a `nearestAbsolute` require style. The `[Unreleased]` changelog syncs to Luau 0.742. The VS Code extension `package.json` version is 1.70.1. — [luau-lsp 1.70.1](https://github.com/JohnnyMorganz/luau-lsp/releases/tag/1.70.1); [luau-lsp CHANGELOG](https://github.com/JohnnyMorganz/luau-lsp/blob/main/CHANGELOG.md); [VS Code extension package.json](https://github.com/JohnnyMorganz/luau-lsp/blob/main/editors/code/package.json)
- Earlier 2026 releases:
  - 1.69.0 (2026-07-18): Moonwave tags in hover.
  - 1.68.1 (2026-06-14) and 1.68.0 (2026-05-16): improved function-snippet completions.
  - 1.67.0 (2026-05-10): a `completion.imports.useConst` setting; "Sourcemap-based auto-imports … are now server/client boundary aware".
  — [luau-lsp releases](https://github.com/JohnnyMorganz/luau-lsp/releases)
- New-solver support: `luau-lsp.fflags.enableNewSolver` is **default `false`, tagged "preview"** ("Enables the flags required for Luau's new type solver"). `luau-lsp analyze` respects it when it is set in the `--settings` file. The changelog lists many new-solver crash and autocomplete fixes. — [VS Code extension package.json](https://github.com/JohnnyMorganz/luau-lsp/blob/main/editors/code/package.json); [luau-lsp CHANGELOG](https://github.com/JohnnyMorganz/luau-lsp/blob/main/CHANGELOG.md)
- Other settings:
  - Sourcemaps are generated by running `rojo sourcemap --watch default.project.json --output sourcemap.json` (requires Rojo ≥ 7.3.0). `luau-lsp.sourcemap.autogenerate` is on by default, and `sourcemap.includeNonScripts` "may be disabled for expensive DataModels".
  - A companion Studio plugin (`luau-lsp.studioPlugin.enabled`, port 3667) supplies DataModel information not in the Rojo tree.
  - `luau-lsp.platform.type` defaults to `roblox`.
  - It can run standalone for CI type and lint checks.
  — [luau-lsp README](https://github.com/JohnnyMorganz/luau-lsp); [VS Code extension package.json](https://github.com/JohnnyMorganz/luau-lsp/blob/main/editors/code/package.json)

**StyLua and Selene**
- **StyLua v2.6.0 — 2026-10-10**: Luau `if local`/`if const` bindings and an `Input` mode for `collapse_simple_statement`; it also fixed sort-requires with `const` and `{ read Foo }` array types. Earlier releases:
  - v2.5.0 (2026-05-16): Luau `const` declarations, and the npm package now ships platform binaries.
  - v2.4.0 (2026-03-07): explicit type instantiation `f<<T>>()` and a `--no-ignore-vcs` flag.
  The repo has about 2,327 stars. — [StyLua v2.6.0](https://github.com/JohnnyMorganz/StyLua/releases/tag/v2.6.0); [StyLua v2.5.0](https://github.com/JohnnyMorganz/StyLua/releases/tag/v2.5.0); [StyLua v2.4.0](https://github.com/JohnnyMorganz/StyLua/releases/tag/v2.4.0)
- StyLua configuration: it looks for `stylua.toml` or `.stylua.toml` from the file's directory up to the current directory. Set `syntax = "Luau"` to avoid union-syntax collisions; the README notes Lua 5.2 `::label::` vs Luau `x :: number`. Keep the config in the project root. — [StyLua README](https://github.com/JohnnyMorganz/StyLua)
- **Selene 0.32.0 — 2026-10-01**: parser support for `if local`/`if const`. Earlier releases:
  - 0.31.0 (2026-05-21): `const`, `EnumItem.EnumType`/`:IsA()`, `Enum:FromName()`/`FromValue()`.
  - 0.30.1 (2026-03-03): "`require` now accepts strings in Roblox".
  - 0.30.0 (2026-01-23): a parser update for recent Luau features.
  The repo has about 835 stars and 165 open issues. — [Selene 0.32.0](https://github.com/Kampfkarren/selene/releases/tag/0.32.0); [Selene 0.31.0](https://github.com/Kampfkarren/selene/releases/tag/0.31.0); [Selene 0.30.1](https://github.com/Kampfkarren/selene/releases/tag/0.30.1); [Selene 0.30.0](https://github.com/Kampfkarren/selene/releases/tag/0.30.0)
- Selene's Roblox setup is `std = "roblox"`. The generated Roblox std auto-updates every 6 hours. You can pin it with `roblox-std-source = "pinned"`, which writes `roblox.yml` into the project, useful for reproducible CI. The TestEZ std (`roblox+testez`) is legacy. — [Selene Roblox guide](https://kampfkarren.github.io/selene/roblox.html)

**Luau and the new type solver**
- Luau ships weekly releases: **0.744 (2026-10-23)**, 0.743 (2026-10-16), 0.742 (2026-10-10), 0.741 (2026-10-02). 0.740 (2026-09-25) introduced an **experimental** "exact tables by default" feature (RFC #11). 0.739 (2026-09-18) added **experimental** `if local` expressions behind a flag ("may likely change in the future"). — [Luau 0.744](https://github.com/luau-lang/luau/releases/tag/0.744); [Luau 0.740](https://github.com/luau-lang/luau/releases/tag/0.740); [Luau 0.739](https://github.com/luau-lang/luau/releases/tag/0.739)
- **"[General Release] Luau's New Type Solver" (2025-11-20)**:
  - The new solver rolled out automatically for **nocheck and nonstrict**, and a redesigned nonstrict mode is on by default.
  - **"Strict mode users stay on the old solver by default and can opt in"**; projects with strict-mode scripts are opted out by default.
  - The Studio Beta toggle was removed **2026-01-07**. Strict users must set `UseNewLuauTypeSolver` to Enabled.
  - `LuauTypeCheckMode` sets the default mode, and `--!strict` still overrides it.
  - The old engine stays available **through 2026**, and the property remains for at least one year and until the old engine is retired.
  - Known issues: larger or less clear errors, slower performance and excessive memory in Studio, lingering "blocked type"/"inference failed" errors, and "strict mode is sometimes too strict".
  — [DevForum: General Release, New Type Solver](https://devforum.roblox.com/t/general-release-luau%E2%80%99s-new-type-solver/4084991)
- Later replies in the thread (Feb–Apr 2026) report high error counts versus the old solver, broken generics in parameter checks, and many type errors in PlayerModule under the new solver. These come from community replies in the search summary and were not individually verified. — [DevForum thread, later pages](https://devforum.roblox.com/t/general-release-luau%E2%80%99s-new-type-solver/4084991?page=6)
- The API reference for `Workspace.UseNewLuauTypeSolver` (type `RolloutState`) says it "controls whether the new Luau type solver is used" and that the "Default follows the current engine-wide rollout state". `LuauTypeCheckMode` controls the mode. — [Workspace API reference](https://create.roblox.com/docs/reference/engine/classes/Workspace)

**darklua**
- **darklua v0.19.0 — 2026-06-10**:
  - **content loaders**, glob → loader mappings (`copy` for Rojo `*.model.json`, `json`/`toml`/`yaml` data loaders, and base64/zstd/gzip embedding);
  - `const` support with a `make_assignment_local` rule that converts `const` to `local`;
  - type-instantiation support;
  - `rename_variables` that detects globals.
  v0.18.0 (2026-02-24) added type function statements, property modifiers and function attributes. The repo was last pushed 2026-07-13. — [darklua v0.19.0](https://github.com/seaofvoices/darklua/releases/tag/v0.19.0); [darklua v0.18.0](https://github.com/seaofvoices/darklua/releases/tag/v0.18.0)
- darklua's `convert_require` rule rewrites string requires into Roblox instance paths. — [darklua convert_require docs](https://darklua.com/docs/rules/convert_require)

### Inferences
- **Keep luau-lsp, StyLua and Selene, and pin them in `rokit.toml`** at 1.70.1, 2.6.0 and 0.32.0. Upgrade them together: any new Luau syntax adopted in source (`const`, `if local`) breaks older parsers in StyLua, Selene, darklua and pesde.
- Do not use `if local` or exact tables yet; both are experimental flags upstream. Confirm that `const` is live in the Roblox engine before using it.
- Strict Luau and the new solver:
  - Keep `--!strict` everywhere, or set a `.luaurc` `languageMode` of strict. This is a standard `.luaurc` key, not re-verified here.
  - For a project starting in late 2026, the old solver's guaranteed availability ends with 2026. It is reasonable to **trial the new solver from day one**: set `UseNewLuauTypeSolver = Enabled` in the place, `luau-lsp.fflags.enableNewSolver: true` in the editor, and pass the same settings to `luau-lsp analyze` in CI so editor, CI and Studio agree.
  - Budget for verbose errors and occasional false positives, especially in generic-heavy code. Fall back to the old solver if editor performance or memory is unacceptable.
  - The gate is a short spike on representative modules (generic signal or ECS query types, deep table types).
- **darklua: not needed for the game.** Roblox runs its own bytecode compilation, so minifying shipped game code buys nothing. Rojo maps source directly, and Roblox supports relative string requires natively. Use darklua only if a module must run in both Lune and Roblox under different require schemes, or to down-level `const`.

### Gaps
- No primary 2026 source says whether strict-mode scripts were ever switched to the new solver by default after the Nov 2025 announcement, or gives a date for retiring the old solver. The rollout appears unchanged, but this is unconfirmed.
- I could not confirm whether the Roblox engine resolves `.luaurc`/`.config.luau` custom aliases natively in late 2026. The evidence is Jest Roblox's "when the engine supports it … `.config` aliases" note and the Luau RFC for `.config.luau` ([RFC](https://rfcs.luau.org/config-luauconfig.html)). The original require-by-string launch had no aliases ([DevForum](https://devforum.roblox.com/t/introducing-require-by-string/3405078)).
- Whether `const` syntax is enabled in live Roblox servers and clients is unverified.

---

## 6. CI/CD and deployment: GitHub Actions + Rokit, Open Cloud place publishing, asset tools

### Takeaway
The standard 2026 pipeline is: `setup-rokit` → `wally install` → `rojo sourcemap` → `luau-lsp analyze` / `selene` / `stylua --check` → `lune run` (fast tests) → `rojo build` → publish to a **test** place via the Open Cloud place-publishing API (`versionType=Saved`) → run Jest Roblox via **Luau Execution** → publish production (`versionType=Published`). **Asphalt 2.0.2** (2026-07-07) is the maintained asset-sync tool. **Tarmac is stale** (last release 2024-01). **Mantle** and **rbxcloud** have had no releases since March 2025.

### Cited Findings
- **Place publishing API**:
  - Request: `POST` to the universes endpoint for a Universe ID and Place ID, with query `versionType=Published` (goes live) or `Saved` (latest version without going live), an `x-api-key` header, and a `.rbxl` (binary) or `.rbxlx` (XML) body.
  - The API key needs universe-places write permission.
  - The guide notes that experiences containing **EditableImage, EditableMesh, PartOperation, SurfaceAppearance or BaseWrap** instances should be published from Studio after modifying them.
  — [Place publishing usage guide](https://create.roblox.com/docs/cloud/guides/usage-place-publishing)
- A July 2025 DevForum report of Published uploads "not incrementing version" turned out to be a duplicated CI request, not a Roblox bug. — [DevForum thread](https://devforum.roblox.com/t/open-cloud-publishing-api-no-longer-seems-to-increment-version-number-on-place/3820071)
- Luau Execution can target a specific place version, which suits CI: publish Saved, then execute against that version. — [Luau Execution reference](https://create.roblox.com/docs/cloud/reference/features/luau-execution)
- **setup-rokit**: `CompeyDev/setup-rokit@v0.2.1`, with inputs `version`, `path`, `cache` and `token`. (`ok-nick/setup-aftman@v0.4.2` is the Aftman equivalent.) — [setup-rokit](https://github.com/CompeyDev/setup-rokit); [setup-aftman](https://github.com/ok-nick/setup-aftman)
- **Asphalt v2.0.2 — 2026-07-07** updated rbx-binary and rbx-xml to fix animation syncing. **v2.0.0 — 2026-04-30**:
  - concurrent syncs and respect for Roblox rate-limit headers;
  - breaking change: `asphalt sync --target X` became `asphalt sync [cloud|studio|debug]`, and `--dry-run` moved under `sync cloud`;
  - a project-directory argument for monorepos;
  - `ROBLOX_STUDIO_PATH` replaced `ROBLOX_CONTENT_PATH`.
  The repo was last pushed 2026-07-28, with about 165 stars. — [Asphalt v2.0.2](https://github.com/jackTabsCode/asphalt/releases/tag/v2.0.2); [Asphalt v2.0.0](https://github.com/jackTabsCode/asphalt/releases/tag/v2.0.0)
- **Tarmac v0.7.5 — 2024-01-08**; the repo was last pushed 2024-03-05 and has about 35 stars. — [rojo-rbx/tarmac](https://github.com/rojo-rbx/tarmac)
- **Mantle v0.11.18 — 2025-03-14**: moved image uploads to the Open Cloud API. Earlier, v0.11.15 (2024-06-26) required `MANTLE_OPEN_CLOUD_API_KEY` after Roblox deprecated the legacy place-upload API. The repo was pushed 2026-05-28 and has 54 open issues. — [Mantle v0.11.18](https://github.com/blake-mealey/mantle/releases/tag/v0.11.18); [Mantle v0.11.15](https://github.com/blake-mealey/mantle/releases/tag/v0.11.15)
- **rbxcloud v0.17.0 — 2025-03-23**: JSON outputs, the Luau Execution API, Inventory and User Restriction APIs. v0.14.0 (2024-07-01) added the Place API. There have been no pushes since 2025-03-23. — [rbxcloud v0.17.0](https://github.com/Sleitnick/rbxcloud/releases/tag/v0.17.0); [rbxcloud v0.14.0](https://github.com/Sleitnick/rbxcloud/releases/tag/v0.14.0)

### Inferences
- Recommended GitHub Actions layout:
  1. **PR job** (ubuntu): setup-rokit (cache on) → `wally install` → `rojo sourcemap default.project.json -o sourcemap.json` → `luau-lsp analyze` with the Roblox definitions and new-solver settings → `selene src` → `stylua --check src` → `lune run tests`.
  2. **Main-branch job**: `rojo build -o build.rbxl` → publish to a *staging/test place* with `versionType=Saved` → create a Luau Execution task running the Jest Roblox entry → poll and fail on non-zero failures → on green, publish to production with `versionType=Published`, or keep that step manual.
  Store the API key as a GitHub secret, scoped only to place publish and luau-execution on the specific universe.
- Publishing has a risk for this game. The place-publishing guide warns about **SurfaceAppearance and PartOperation** (unions), which are likely in an ocean map (PBR shark skins, union reefs). Keep heavy art in a Studio-authored base place, or publish map changes from Studio, and let CI publish code builds. Alternatively, verify in a test universe that Rojo-built places containing those instances survive API publishing.
- **Asphalt over Tarmac** for images, audio and animations. Prefer direct Open Cloud calls (curl, or small Lune scripts) over Mantle or rbxcloud for publishing and execution; both look under-maintained in 2026.

### Gaps
- No official, maintained GitHub Action from Roblox for publishing or OCALE was found; the demo is archived.
- The exact current place-publishing endpoint path and size limits were taken from the search snippet of the guide, not from a full read of the page.

---

## 7. AI-assisted tooling: Roblox Studio MCP server, Assistant, and external coding agents

### Takeaway
In 2026 Roblox moved from a standalone Rust MCP server (**archived, "Product discontinued" 2026-04-03**) to an **MCP server built into Studio**, enabled from Assistant. It offers quick-connect for Claude Code, Codex CLI, Cursor, Gemini CLI, VS Code, Antigravity and Claude Desktop, plus about 25 tools, including `execute_luau` in Edit, Client or Server datamodels, playtest control, console output, screen capture and input simulation. Roblox's official "coding harness" guide pairs **Script Sync + Studio MCP** and does not mention Rojo. In a Rojo project, agents should edit files on disk and use MCP for running, playtesting and inspecting.

### Cited Findings
- `Roblox/studio-rust-mcp-server` is **archived**. Its final "END — Product discontinued" release (2026-04-03) says: "We've shifted ongoing engineering investment to the built-in MCP Server included with Roblox Studio". The last versioned release was v0.2.365 (2026-02-24). — [studio-rust-mcp-server END release](https://github.com/Roblox/studio-rust-mcp-server/releases/tag/END)
- Built-in MCP server (official docs):
  - **Enabling:** Assistant → … → Manage MCP Servers → "Enable Studio as MCP server".
  - **Clients:** quick connect for Antigravity, Codex CLI, Claude Code, Claude Desktop, Cursor, Gemini CLI and VS Code. Any stdio-capable client also works.
  - **Launch:** server name `Roblox_Studio`. On Windows, `cmd.exe /c %LOCALAPPDATA%\Roblox\mcp.bat`; on macOS, `/Applications/RobloxStudio.app/Contents/MacOS/StudioMCP`.
  - **Tools:**
    - scripts: `script_read`, `multi_edit`, `script_search` (≤10 results), `script_grep` (≤50 matches);
    - assets: `generate_mesh`, `generate_material`, `generate_procedural_model`, `search_asset`, `insert_asset`, `upload_image`, `store_image`, `wait_job_finished`;
    - data model: `search_game_tree`, `inspect_instance`;
    - execution: `execute_luau` (Edit, Client or Server);
    - playtesting: `get_studio_state`, `start_stop_play`, `get_console_output`, `screen_capture`;
    - input: `character_navigation`, `user_keyboard_input`, `user_mouse_input`;
    - other: `http_get`, `skill`, `subagent`, `list_roblox_studios`.
  - Every call takes a `studio_id`.
  - The docs warn that clients can read and modify open places, so connect only trusted clients.
  — [Studio MCP docs](https://create.roblox.com/docs/studio/mcp)
- 2026 DevForum timeline (dates from search-result metadata; I did not open each post):
  - "Studio MCP Server Updates and External LLM Support for Assistant" (~2026-02-21): BYOK external LLMs for Assistant, and MCP to be integrated into Studio.
  - "Assistant Updates: Studio Built-in MCP Server and Playtest Automation" (~2026-03-05).
  - "Assistant Updates: Mesh Generation, New MCP Server Tools, Screenshot Tool" (~2026-03-19).
  - "[Studio Beta] Studio Assistant & MCP Playtest Agent" (~2026-04-09).
  - "Assistant Updates: Introducing Creator-Authored Skills" (~2026-07-09): skills usable from third-party clients.
  - "Studio MCP: Multi-Agent Improvements and Connected AI Clients" (~2026-08-19).
  — [Feb post](https://devforum.roblox.com/t/studio-mcp-server-updates-and-external-llm-support-for-assistant/4415631); [Mar 5 post](https://devforum.roblox.com/t/assistant-updates-studio-built-in-mcp-server-and-playtest-automation/4474643); [Mar 19 post](https://devforum.roblox.com/t/assistant-updates-mesh-generation-new-mcp-server-tools-screenshot-tool-and-more/4527258); [Apr beta post](https://devforum.roblox.com/t/studio-beta-studio-assistant-mcp-playtest-agent/4566767); [Jul skills post](https://devforum.roblox.com/t/assistant-updates-introducing-creator-authored-skills/4728447); [Aug multi-agent post](https://devforum.roblox.com/t/studio-mcp-multi-agent-improvements-and-connected-ai-clients/4820583)
- Roblox CEO post: "Studio MCP Server now supports full agentic loops, and Assistant can connect to external LLMs via Bring Your Own Key (BYOK)." — [David Baszucki on X](https://x.com/DavidBaszucki/status/2025246197209071959)
- Roblox doc "Build with a coding harness" (updated 2026-10-29):
  - Recommends VS Code (with an extension such as Claude Code) or Cursor, plus Git.
  - Uses Script Sync to link ServerScriptService, ReplicatedStorage and StarterPlayerScripts to disk folders.
  - Uses the Studio MCP server for things Script Sync can't reach, such as StarterGui and StarterPack, and to run scripts and check output.
  - Suggests a context `.md` file for the agent, approving agent commands or allowlisting the MCP tool, and verifying with `InstanceFileSyncService:GetStatus(instance)`.
  - Does not mention Rojo.
  — [Coding harness doc](https://create.roblox.com/docs/ai/coding-harness)
- rodeo can run code at the StudioMCP identity (`--context mcp`, which needs Studio's MCP server) and drive play and test sessions from the terminal. — [rodeo README](https://github.com/rodeo-rbx/rodeo)
- jest-roblox-cli includes an "agent" output formatter. — [jest-roblox-cli README](https://github.com/christopher-buss/jest-roblox-cli)
- **Caution about GitHub:** searching "studio mcp roblox" on 2026-10-30 returned many repos with SEO-stuffed "2026" descriptions, about 70 stars each and pushes within hours. Examples: "Best Roblox Studio MCP Server for AI Runtime Debugging 2026" and "Best Open-Source Roblox Studio AI Workflow for Claude Code 2026". The same pattern appears for "Jest Roblox Testing Framework 2026". — [GitHub search API results](https://api.github.com/search/repositories?q=studio+mcp+roblox&sort=updated)

### Inferences
- Recommended agent workflow for this Rojo project:
  - Agents edit files in the git repo (`src/**`), and Rojo syncs them into Studio.
  - The built-in MCP server is used **read/run-only**: `execute_luau` to run Jest in Server or Client datamodels, `start_stop_play` + `get_console_output` + `screen_capture` for playtest checks, and `search_game_tree` / `inspect_instance` to inspect.
  - Avoid MCP script-writing tools (`multi_edit`) on Rojo-managed trees, because Studio-side edits diverge from disk and will be overwritten. This is inferred.
  - Keep an `AGENTS.md`/`CLAUDE.md` in the repo describing layout, strict typing and test commands.
- Rojo 7.7's Host/Origin validation and the `/api/open` gate reduce DNS-rebinding risk to the local Rojo server while browsers and agents run. Keep `rojo serve` bound to localhost.
- Treat the "2026"-branded community MCP and test repos as untrusted (likely spam or malware vectors). Prefer Roblox's built-in server and established tools.

### Gaps
- I did not find whether the MCP server can emulate mobile devices or poor network conditions, which matters given this game's mobile priority.
- The exact GA date of the built-in server, and whether the MCP playtest subagent left beta, were not verified from the posts themselves.

---

## 8. Recommended starter layout, default.project.json, and config defaults (2026)

### Takeaway
The `rojo init` place template still maps `src/shared` → `ReplicatedStorage.Shared`, `src/server` → `ServerScriptService.Server`, and `src/client` → `StarterPlayer.StarterPlayerScripts.Client`. Keep the SMASH layout and Packages mount. Add a `ServerPackages` mount, because ProfileStore is a server-realm Wally package. Keep `DevPackages` (Jest) out of the production project. Configs: `stylua.toml` with `syntax = "Luau"`, `selene.toml` with `std = "roblox"` (optionally pinned), strict Luau, and `sourcemap.json` git-ignored.

### Cited Findings
- Rojo's place template `assets/project-templates/place/default.project.json` maps `ReplicatedStorage.Shared` ← `src/shared`, `ServerScriptService.Server` ← `src/server`, and `StarterPlayer.StarterPlayerScripts.Client` ← `src/client`. It also sets a Workspace baseplate, Lighting properties (`Technology: "Voxel"`) and `SoundService.RespectFilteringEnabled`. — [Rojo place template](https://github.com/rojo-rbx/rojo/blob/master/assets/project-templates/place/default.project.json)
- Rojo accepts comments and trailing commas in project, meta and model JSON (7.6.1) and `.jsonc` everywhere `.json` is allowed (7.7.0). It adds `sourcemap.json` to the default `.gitignore` (7.6.1), and `syncbackRules` lives in the project file (7.7.0). — [Rojo v7.6.1](https://github.com/rojo-rbx/rojo/releases/tag/v7.6.1); [Rojo v7.7.0](https://github.com/rojo-rbx/rojo/releases/tag/v7.7.0)
- Wally realm conventions: ProfileStore's `wally.toml` declares `realm = "server"`. Argon's notes refer to Wally `ServerPackages` and `DevPackages` folders. Jest Roblox is installed as a dev-dependency. — [ProfileStore wally.toml](https://github.com/MadStudioRoblox/ProfileStore/blob/main/wally.toml); [Argon 2.0.24](https://github.com/argon-rbx/argon/releases/tag/2.0.24); [Jest Roblox README](https://github.com/Roblox/jest-roblox)
- luau-lsp expects `sourcemap.json` at the workspace root and runs `rojo sourcemap --watch default.project.json --output sourcemap.json` by default. — [luau-lsp README](https://github.com/JohnnyMorganz/luau-lsp)
- Roblox supports require-by-string with `./` and `../` relative paths. Aliases were not part of the launch. — [DevForum: Introducing Require-by-String](https://devforum.roblox.com/t/introducing-require-by-string/3405078)
- Several tools gained Roblox string-require awareness in 2026: Selene 0.30.1, pesde 0.7.2, and luau-lsp 1.70.x (`@self` and `.luaurc` alias fixes). — [Selene 0.30.1](https://github.com/Kampfkarren/selene/releases/tag/0.30.1); [pesde releases](https://github.com/pesde-pkg/pesde/releases); [luau-lsp CHANGELOG](https://github.com/JohnnyMorganz/luau-lsp/blob/main/CHANGELOG.md)
- Luau's `.config.luau` (Luau-syntax config) is implemented in Luau. It is sandboxed (cannot require other scripts). Having both `.luaurc` and `.config.luau` in the same folder triggers an ambiguity error. — [Luau RFC: config-luauconfig](https://rfcs.luau.org/config-luauconfig.html)
- StyLua: put `.stylua.toml` in the repo root and set `syntax`. Selene: `std = "roblox"`, with `roblox-std-source = "pinned"` for reproducibility. — [StyLua README](https://github.com/JohnnyMorganz/StyLua); [Selene Roblox guide](https://kampfkarren.github.io/selene/roblox.html)

### Inferences
Suggested starter layout (SMASH-compatible):
```
rokit.toml              # rojo 7.7.1, wally 0.3.2, lune 0.11.1, rodeo 1.5.1, luau-lsp 1.70.1, stylua 2.6.0, selene 0.32.0, asphalt 2.0.2 (+ blink per networking researcher)
wally.toml / wally.lock
default.project.json    # production tree
test.project.json       # = default tree + DevPackages + test specs (for Jest via OCALE/rodeo)
.luaurc                 # {"languageMode": "strict"}; no custom aliases until Roblox engine alias support is confirmed
.stylua.toml            # syntax = "Luau" (plus team prefs: column_width, indent_type, quote_style, call_parentheses)
selene.toml             # std = "roblox"; roblox-std-source = "pinned" for CI determinism
.gitignore              # Packages/, ServerPackages/, DevPackages/, sourcemap.json, *.rbxl(x) build outputs, roblox.yml if not pinned-in-repo
src/shared/  src/server/  src/client/
tests/lune/   (lunetest.luau specs)     tests/roblox/__tests__/ (Jest Roblox *.spec.luau)
assets/ + asphalt.toml
```
`default.project.json` core (extends the rojo init template):
```json
{
  "name": "shark-game",
  "tree": {
    "$className": "DataModel",
    "ReplicatedStorage": {
      "Shared":   { "$path": "src/shared" },
      "Packages": { "$path": "Packages" }
    },
    "ServerScriptService": {
      "Server":         { "$path": "src/server" },
      "ServerPackages": { "$path": "ServerPackages" }
    },
    "StarterPlayer": {
      "StarterPlayerScripts": { "Client": { "$path": "src/client" } }
    }
  }
}
```
- **Keep** Packages at `ReplicatedStorage.Packages` and the `src/shared|server|client` split; both match the Rojo template and Wally conventions.
- **Add** `ServerPackages` mounted under ServerScriptService (or ServerStorage). It keeps server-only code such as ProfileStore and Lyra out of client replication, which also trims what mobile clients download.
- Map `DevPackages` only in `test.project.json`.
- Exclude `Packages/`, `ServerPackages/` and `DevPackages/` from StyLua and Selene runs.
- Use instance-path or relative string requires (`./`, `../`), both natively supported and understood by luau-lsp. Defer `@alias` requires until Roblox-side alias resolution is confirmed. Never keep `.luaurc` and `.config.luau` in the same folder.
- If the new solver is adopted, set `UseNewLuauTypeSolver = Enabled` in the place file through the project's Workspace `$properties` or in Studio. Mirror it with `luau-lsp.fflags.enableNewSolver` in `.vscode/settings.json` and the CI settings file. Whether Rojo can set this property via `$properties` is unverified.

### Gaps
- No single authoritative "2026 community template" was found. The layout above extends the official `rojo init` template with conventions evidenced in tool docs.
- `wally-package-types` (re-exporting types through Wally link thunks) was not checked in this session.

---

## 9. Verdict summary vs the SMASH toolchain (keep / upgrade / replace)

### Takeaway
The SMASH stack is still the right architecture in late 2026. Upgrade **Rojo (7.7.1), Lune (0.11.1) and rodeo (1.5.1)**. **Remove the Aftman fallback**, since Aftman is archived. **Keep Wally 0.3.2** and do not adopt the 0.4 alpha or pesde. Add **Jest Roblox + Open Cloud Luau Execution** as the engine-context test layer. Keep strict Luau, but decide deliberately on the new type solver.

### Cited Findings
- Versions and dates are as cited in sections 1–8. Key ones:
  - Rojo v7.7.1 (2026-10-02): [release](https://github.com/rojo-rbx/rojo/releases/tag/v7.7.1)
  - Rokit 1.2.0 (2025-09-30): [release](https://github.com/rojo-rbx/rokit/releases/tag/v1.2.0)
  - Aftman archived: [repo](https://github.com/LPGhatguy/aftman)
  - Wally 0.4.0-alpha.1 (2026-10-12): [release](https://github.com/UpliftGames/wally/releases/tag/v0.4.0-alpha.1)
  - Lune 0.11.1 (2026-10-21): [release](https://github.com/lune-org/lune/releases/tag/v0.11.1)
  - rodeo 1.5.1 (2026-10-27): [release](https://github.com/rodeo-rbx/rodeo/releases/tag/v1.5.1)
  - Jest Roblox 3.20.1 (2026-08-30): [changelog](https://github.com/Roblox/jest-roblox/blob/master/CHANGELOG.md)
  - New type solver rollout: [DevForum](https://devforum.roblox.com/t/general-release-luau%E2%80%99s-new-type-solver/4084991)
  - Blink 0.18.9 on pesde (2026-09-19): [pesde index](https://github.com/pesde-pkg/index/blob/main/1axen/blink)

### Inferences

| SMASH choice | Latest (date) | Verdict | Why |
|---|---|---|---|
| Rokit (toolchain manager) | 1.2.0 (2025-09-30) | **Keep** (`rokit self-update`) | Community standard; Rojo docs list it first; reads aftman.toml/foreman.toml |
| Project-local Aftman fallback pinning Rojo | Aftman 0.3.0 (2024-05-22), **repo archived** | **Replace: delete it** | Redundant (Rokit checks rokit.toml first and also reads aftman.toml); drift risk; possible PATH-shim confusion; unmaintained |
| rojo 7.7.0 | 7.7.1 (2026-10-02) | **Upgrade** (CLI and plugin together) | Windows path fixes; patch on a protocol-changing release; watch #1300 / 7.7.2 |
| Studio Script Sync (new option) | Full release 2026-06-17 | **Don't adopt for this project** | Scripts-only, Studio is source of truth, no CI build or sourcemap; fine for Studio-centric teams |
| Argon (alternative) | 2.0.29 (2026-05-19) | **Don't switch** | Rojo 7.7 now has syncback; smaller community |
| wally 0.3.2 | 0.3.2 stable; 0.4.0-alpha.1 (2026-10-12) | **Keep 0.3.2** | Ecosystem (jecs, Lyra, ProfileStore, Charm, Planck, roblox/jest) is on Wally; alpha changes the lockfile and flags |
| pesde (alternative) | 0.7.4 (2026-09-09) | **Don't adopt now** | Only mirrors or partial coverage for key libs; migration cost; revisit later (Lute's registry may also reshape this) |
| blink 0.18.8 | 0.18.9 (2026-09-19, pesde) | Out of scope (networking) | Bump available; defer to the networking research |
| lune 0.10.4 | 0.11.1 (2026-10-21) | **Upgrade** | Security fix; CFrame.lookAt/lookAlong rotation fix (critical for 3D swim-math tests); newer rbx-dom database |
| Lune harness (lunetest.luau) | — | **Keep, and add an engine layer** | Fast offline logic tests; add Jest Roblox 3.20.1 via OCALE (CI) and rodeo or jest-roblox-cli (local Studio) |
| TestEZ / jsdotlua jest-lua (if used) | archived / stale since 2024-12 | **Replace** with `roblox/jest` 3.20.1 | Official, maintained, OCALE-ready |
| rodeo 1.0.1 | 1.5.1 (2026-10-27) | **Upgrade** | First stable since 1.3; CLI/serve/plugin must match; `--adapter` breaking change; repo moved to rodeo-rbx |
| strict Luau everywhere | New solver default only for nonstrict/nocheck; old solver "through 2026" | **Keep strict; trial the new solver from day one** | Avoid a 2027 forced migration; mirror the setting in Studio, luau-lsp and CI; fall back if too noisy |
| luau-lsp / StyLua / Selene | 1.70.1 / 2.6.0 / 0.32.0 | **Pin latest in rokit.toml** | Track Luau syntax; upgrade together |
| darklua | 0.19.0 (2026-06-10) | **Optional / skip** | No runtime benefit for game code; only for cross-runtime require conversion |
| Packages at ReplicatedStorage.Packages; src/shared, src/server, src/client | — | **Keep, and add ServerPackages** | Matches the rojo init template; keeps server-realm packages off clients |
| Asset pipeline (new) | Asphalt 2.0.2 (2026-07-07) | **Adopt Asphalt** (not Tarmac) | Tarmac stale since 2024 |
| Deployment (new) | Open Cloud place publishing + Luau Execution | **Adopt** via a setup-rokit-based GitHub Actions workflow | Official APIs; Mantle and rbxcloud have had no releases since 2025-03 |
| AI agents (new) | Built-in Studio MCP server | **Adopt for run/playtest/inspect**; agents edit files on disk | Official; the standalone Rust MCP server is discontinued |

Proposed `rokit.toml`:
```toml
[tools]
rojo = "rojo-rbx/rojo@7.7.1"
wally = "UpliftGames/wally@0.3.2"
lune = "lune-org/lune@0.11.1"
rodeo = "rodeo-rbx/rodeo@1.5.1"
luau-lsp = "JohnnyMorganz/luau-lsp@1.70.1"
stylua = "JohnnyMorganz/StyLua@2.6.0"
selene = "Kampfkarren/selene@0.32.0"
asphalt = "jackTabsCode/asphalt@2.0.2"
```
Each `owner/repo@version` above corresponds to a GitHub release verified on 2026-10-30. Rokit artifact compatibility for luau-lsp, rodeo and asphalt was not individually tested.

### Gaps
- Rokit's installability of every proposed tool, which depends on release asset naming, was not tested here. SMASH already installed rodeo and blink via Rokit.
- Mobile-specific automated testing on devices or emulated poor networks was not covered by any tool found in this research.
