---
title: Shark movement
status: planned
code: []
decisions: [D4]
related: [input-and-controls, eating-and-bites, animation, architecture]
last_verified: n/a (no code yet)
---

# Shark movement

## Summary

Each player's shark is a physics assembly owned by that player's client, so swimming responds instantly at any ping and the server never simulates players. The server only checks the results (speed, position, invalid values) and pulls offenders back. The swim step is a pure function, so Roblox Server Authority can be adopted later without a rewrite.

## Spec

### Body [decided]

- No Humanoid: a shark that pitches freely would fight the Humanoid state machine.
- Dry ocean: there's no Terrain water, so swimming is our own force model.
- Movers: LinearVelocity or VectorForce for thrust, AlignOrientation for heading, and a constant upward force of `mass × workspace.Gravity` to cancel gravity.
- Alternative to prototype for a day: ControllerManager with a SwimController. It drives the root toward `MovingDirection × BaseMoveSpeed`, pitches toward `FacingDirection` and rolls upright, with no Humanoid needed. But its BuoyancySensor only detects Terrain water, and nothing documents how it behaves outside water or how it handles gravity [engine].

### Swim step [decided]

- `step(state, input, dt) -> state` lives in `src/shared/`. It's pure: no Instance reads or writes, no globals. A thin adapter writes its result into the movers.
- Input comes from the Input Action System ([input-and-controls](input-and-controls.md)).
- Controls: the shark cruises forward automatically and steers relative to the camera; Boost and Bite are buttons.
- Speeds, acceleration, turn rates and boost costs aren't designed.

### Ownership [decided]

- The server gives each player network ownership of their own shark's assembly.
- Collision groups turn off shark-to-shark collisions, which removes the main way to fling other players.

### Server checks [decided]

- **Speed budget, in full 3D.** Roblox's XZ-plane suggestion doesn't fit swimming. Each server tick adds `maxSpeed × dt` (boost-aware) to an allowance capped at a burst size, then subtracts the distance actually moved since the last check; a negative allowance is a violation [proposed formula].
- **Invalid values.** Reject NaN or infinite positions and velocities.
- **Response.** Pull the shark back to its last valid position. Never kick on these heuristics: Roblox warns that they flag innocent players on unstable connections.
- The burst size and tolerances are set from measurements on real phones.

### Keeping Server Authority possible [decided]

Write the pure step function, read input through the Input Action System, build with Deferred signals and streaming on, and refer to entities by numeric id in remotes. Adopt Server Authority only when the condition in [D4](../decisions.md#d4-movement-authority) is met.

What adopting it would change [engine]:

- `Workspace.AuthorityMode = Server` can only be set in Studio, not from scripts.
- It forces NextGenerationReplication, Deferred signals, StreamingEnabled, fixed-step simulation, and player scripts built on the Input Action System.
- Gameplay logic runs in `RunService:BindToSimulation` (30 Hz by default).
- Only state stored in attributes rolls back: at most 64 attributes per instance, with strings of at most 50 characters. `RunService.Rollback` lets you rewind other state yourself.
- Staff put client resimulation at about 6.25× normal load at 100 ms. Player collisions always mispredict. There's no built-in server rewind for hits.

## Invariants

- The server never accepts a position without the speed and NaN checks.
- `step()` has no side effects.

## Tests

None yet. Planned Lune specs:

- `step()` respects maximum speed and boost limits, and gives the same output for the same input. This needs Lune 0.11 or later: earlier versions compute `CFrame.lookAt` wrongly off the Y axis.
- The speed bucket flags a teleport and accepts a legal boost.

## Open questions

- Movers or SwimController: decide by prototype.
- Speed, acceleration, turn rate and boost numbers.
- Whether growth changes mass and forces (growth isn't designed).
- Collisions with the seabed and obstacles.

## History

- 2026-10-30 — Created with the planned design from the tech stack research.
