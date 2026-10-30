---
title: Input and controls
status: planned
code: []
decisions: [D11]
related: [shark-movement, ui]
last_verified: n/a (no code yet)
---

# Input and controls

## Summary

All gameplay input goes through Roblox's Input Action System (IAS): it's the only input path Server Authority accepts, and Roblox's default PlayerScripts will require it by mid-2027. Four actions cover the game, and the controls are built for one thumb on each side of a phone.

## Spec

### Actions [decided]

| Action | Type | Used for |
|---|---|---|
| Move | Direction2D | Steering |
| Look | Direction2D | Camera |
| Boost | Bool | Speed burst |
| Bite | Bool | Bite or lunge |

- Keyboard and gamepad bindings aren't designed. Touch buttons bind through `UIButton` [engine].
- The shark cruises forward automatically and steers relative to the camera, which keeps simultaneous touches to a minimum.
- Boost and Bite are large buttons in the right-thumb zone, sized as on the [ui](ui.md) page.
- Every successful eat plays a `HapticEffect` pulse.

### IAS status [engine]

Generally available since 2026-06-11. Roblox's default PlayerScripts start using it in early 2027 and require it by mid-2027.

### Known gaps to prototype early

- Four on-screen buttons reportedly can't drive one Direction2D action.
- It's unclear how to read the default mobile thumbstick through IAS, so a custom UserInputService touch joystick may be needed.
- There's no gyroscope support yet.

## Invariants

- Gameplay code reads actions, never devices directly (except for a custom joystick, if the thumbstick gap forces one).

## Tests

None yet. SMASH's `tools/test-mobile-controls.luau` is a model for a headless touch-input spec.

## Open questions

- Keyboard and gamepad bindings.
- The default thumbstick or a custom joystick.
- Camera control on touch screens.

## History

- 2026-10-30 — Created with the planned design from the tech stack research.
