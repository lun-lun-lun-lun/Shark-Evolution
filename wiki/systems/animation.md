---
title: Animation
status: planned
code: []
decisions: [D9]
related: [rendering, shark-movement]
last_verified: n/a (no code yet)
---

# Animation

## Summary

Player sharks are skinned meshes animated by Roblox's Animator through one Animation Graph. Fish have no Animator; they move procedurally. crunchyroll, which SMASH used, isn't used here: it targets Motor6D/R6 rigs and has no bone support.

## Spec

### Player shark rig [decided]

- One skinned MeshPart with a few bones: spine, tail, jaw and fins.
- AnimationController + Animator; no Humanoid.
- One Animation Graph that blends idle, cruise and dash on a speed parameter, with bite and boost layered on top.
- The graph runs in Automatic mode, where player graphs run on the owning client and replicate their parameters [engine]. That keeps animation responsive on bad networks.
- Risk: the Animation Graph docs don't confirm support for AnimationController or bone-driven rigs. Prototype this first, and watch the Animation category of Creator Hub's Client CPU Time chart while testing.

### Fish [decided]

- No Animator and no Humanoid.
- Skinned fish (the 20–30 nearest): 1–3 tail-bone transforms written per frame, as a procedural sway.
- Rigid fish: a small sway added to the CFrame passed to `BulkMoveTo` [proposed].
- Precedent: ABZÛ reportedly went from about 100 to about 10,000 on-screen fish by replacing skeletal animation with math.

## Invariants

- No fish has an Animator.

## Tests

None yet.

## Open questions

- Graph parameter names and states; bone count.
- The sway formula (amplitude and frequency against swim speed).
- How the rig scales as a shark grows (growth isn't designed).

## History

- 2026-10-30 — Created with the planned design from the tech stack research.
