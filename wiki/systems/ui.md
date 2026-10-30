---
title: UI
status: planned
code: []
decisions: [D11]
related: [input-and-controls, persistence, rendering]
last_verified: n/a (no code yet)
---

# UI

## Summary

Menus are built with vide; values that change every frame are written straight to the HUD. Everything stays inside the device's safe area and is sized for thumbs.

## Spec

- **Menus** (shop, shark selection, settings): vide 0.4.1, using the `create(class)(props)` form [decided].
- **HUD values** (growth bar, boost meter and similar): written directly to Instances at 10–15 Hz, not through reactive state [decided].
- **Safe area:** `ScreenGui.ScreenInsets = CoreUISafeInsets` (the default). Don't use the legacy FullscreenExtension [decided].
- **Touch targets:** at least 9 × 9 mm, which is about 56 px at Roblox's normalized ~160 dpi. `TextSize` 20 is about 1/8 inch tall [engine].
- **CanvasGroup:** not on large panels; it renders blank once a device's texture memory cap is exceeded [engine].
- **Topbar:** TopbarPlus 3.5.0 [decided].
- **State sync (optional):** Charm 0.11.1 with charm-sync 0.4.1 can send profile state to clients as small patches [decided].
- **Images:** texture streaming doesn't cover UI textures [engine], so keep UI images small.

## Invariants

- Nothing interactive sits outside the safe area.
- No touch target is smaller than 9 mm.

## Tests

None yet. SMASH's `tools/test-mobile-ui.luau` is a model for a headless layout check.

## Open questions

- HUD layout, menu flows and art style.

## History

- 2026-10-30 — Created with the planned design from the tech stack research.
