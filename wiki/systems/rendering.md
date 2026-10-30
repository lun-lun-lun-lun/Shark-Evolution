---
title: Rendering
status: planned
code: []
decisions: [D8, D9]
related: [fish-simulation, animation, ui, testing-and-performance]
last_verified: n/a (no code yet)
---

# Rendering

## Summary

Low-end phones fail on memory first and frame time second, so the world is built inside a fog bubble of about 200 studs. That one choice shrinks every other budget: streaming radius, fish render radius, network interest radius and texture memory. Fish are instanced, anchored, pooled meshes moved in one batch per frame; the water is faked; lighting takes the cheap path.

## Spec

### Visibility [decided]

- Fog caps underwater visibility at about 120–200 studs; the exact distance is chosen on a real phone. Low-end devices can already cut draw distance to 200 studs, and Roblox plans a 500-stud minimum for late 2026 [engine].
- The ocean feels big through colour shifts by depth zone, "marine snow" particles near the camera, and rare huge low-poly silhouettes looming through the murk, not through draw distance.

### Streaming [decided]

| Setting | Value | Engine default |
|---|---|---|
| `StreamingEnabled` | true | — |
| Stream-out behaviour | Opportunistic | — |
| `ModelStreamingBehavior` | Improved | — |
| `StreamingTargetRadius` | 256–512 studs [target] | 1,024 [engine] |
| `StreamingMinRadius` | 64–128 studs [target] | 64 [engine] |
| `StreamingIntegrityMode` | PauseOutsideLoadedArea, if it holds up with boosting sharks under the 3G network preset [proposed] | — |

- Instances created by client scripts never stream out [engine], so the fish pool culls itself by distance and camera view.
- SLIM can merge dense reef and wreck clusters into fewer draw calls and falls back harmlessly where it isn't supported. It needs streaming and excludes animated and runtime-modified models, so it won't help with fish or sharks. No official post confirms it has left Client Beta on phones; NPC support is planned for early 2027 [engine].

### Lighting [decided]

- Compare `LightingStyle = Soft` against Realistic with `PrioritizeLightingQuality` off on the baseline phone, and pick one.
- Not Future lighting. Future is Realistic with PrioritizeLightingQuality on, which lowers draw distance on weak devices to protect lighting quality [engine].
- Bloom subtle or off. No DepthOfField or SunRays.
- Shadows disappear below graphics quality level 4 anyway [engine].
- Bioluminescence uses Neon materials, not lots of PointLights.

### Water [decided]

No Terrain water volume: it's read-only and has an open rendering bug with streaming. Fake the ocean with:

1. Atmosphere and fog driven by the camera's depth zone.
2. A ColorCorrection tint.
3. A small, dark skybox (Atmosphere doesn't fog the sky [engine]).
4. A camera-following surface mesh seen from below.

The seabed is meshes. Expanded Terrain (full launch 2026-10-22) is usable only for a thick seabed: its client performance gains were switched off on 2026-10-27 pending a crash fix, and terrain thinner than 16 voxels can vanish under small streaming radii [engine].

### Textures [decided]

- Fish textures 256²; hero sharks 512² or smaller. Tint with Color instead of making colour variants.
- Texture memory scales with pixel count: 1024² costs four times as much as 512² [engine].
- Texture streaming has covered all platforms since March 2026, but skips Terrain, Sky, UI and the Water material [engine], so keep skybox and UI images small.

### Fish rendering [decided]

- **Instancing:** identical meshes share one draw call only if they also share the same SurfaceAppearance, texture or material [engine]. Each species is one MeshPart asset with a shared appearance, tinted by `Color`. Check draw calls with Shift+F2.
- **Fish parts:** `Anchored = true`; `CanCollide`, `CanTouch`, `CanQuery` and `CastShadow` all false.
- **Movement:** one preallocated `workspace:BulkMoveTo` call per frame for all rigid fish.
- **Update rate by distance:** every frame near the camera; ≤ 30 Hz mid-range; 10–15 Hz at the fog edge, or merged school impostors there.
- **Skinning:** skinned meshes appear not to instance (one test went from about 300 to over 1,000 draw calls). Only player sharks and the 20–30 nearest fish are skinned, writing 1–3 tail-bone transforms per frame ([animation](animation.md)).
- **LOD:** automatic mesh LOD switches only at 250 and 500 studs, beyond the fog [engine]. Author low-poly fish, and swap in pooled lower-detail meshes ourselves.
- **Never:** Humanoids or Animators on fish; EditableMesh (limited to ID-verified 13+ creators, with strict client memory budgets [engine]).

### Overlays [decided]

- At most 255 Highlights render per client, and disabled ones still take slots [engine]. Use them for one or two temporary targets only.
- Nameplates are pooled and shown only for the nearest few players.

### Client CPU [decided]

- Native code for LocalScripts has shipped on Android only [engine]. Assume iPhones interpret client Luau, and size fish counts for that.
- `PreRender` blocks rendering until it returns [engine]: use it only for the camera.
- Spread distant-fish updates across frames, and never allocate per fish per frame.
- Budget: 2–4 ms of client Luau per frame for fish, effects and UI [target].

### Memory [target]

- About 500–600 MB in total on 2–3 GB phones. A 2 GB phone reportedly leaves Roblox about 800 MB, and Roblox staff say most mobile crashes are memory-related.
- Roblox's own example budget for a baseline device is under 1,000 draw calls and 1 million triangles [engine].

## Invariants

- No Terrain water.
- Every fish part is anchored, with collision, touch, query and shadows off.
- No per-fish Highlights.

## Tests

On-device checks only; see [testing-and-performance](testing-and-performance.md).

## Open questions

- Soft or Realistic lighting, and the exact fog distance.
- The phone cost of `BulkMoveTo` versus writing `Motor6D.Transform`. On PC with 1,000 assemblies, CFrame or BulkMoveTo cost about 6 ms of script plus about 3 ms of engine work, while Motor6D.Transform on parts jointed to Terrain cost about 1 ms.
- Whether skinned meshes can instance at all (undocumented).
- Whether SLIM is out of beta on phones.
- A reported particle cap (16,000 on desktop, 3,600 on mobile) is unverified.

## History

- 2026-10-30 — Created with the planned design from the tech stack research.
