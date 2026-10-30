# Roblox Client Rendering & CPU/Memory Performance on Low/Mid-End Mobile, Applied to an Open-Ocean Game with Hundreds of Fish (state as of Oct 30, 2026)

Tags used below: **[official]** = create.roblox.com docs or Roblox staff announcement; **[community]** = DevForum user post (anecdotal); **[secondary]** = third-party press/blog/wiki; **[older: YYYY]** = pre-2025 source that may be outdated; **[beta]** = not generally released.

## 1. How Roblox renders on mobile in 2026: instancing, draw-call/triangle budgets, mesh LOD, PBR, textures, transparency, particles

### Takeaway
Draw calls are the first-order rendering cost on Roblox. Identical meshes collapse into one instanced draw call only when they share the same MeshContent **and** the same SurfaceAppearance (or same TextureContent, or the same material when untextured). Roblox's own example baseline budget is under 1,000 draw calls and under 1,000,000 triangles. Automatic mesh LOD (RenderFidelity) switches only at 250 and 500 studs, so inside a fog-limited ocean you have to author low-poly fish yourself. Texture streaming (rolling out from Dec 2025, on all platforms by Mar 2026) now caps texture memory dynamically, but texture pixel count still drives memory, and Terrain, Sky and UI textures are not yet streamed.

### Cited Findings
**Instancing / batching rules**
- Meshes with the same MeshContent are batched into one draw call when (a) their SurfaceAppearances are identical, or, if they have none, their TextureContents are identical, and (b) their materials are identical when there is neither a SurfaceAppearance nor a MeshPart.TextureID. **[official, 2026]** — [Roblox: Improve performance](https://create.roblox.com/docs/performance-optimization/improve)
- Importing a whole map at once can give identical meshes different asset IDs, which blocks instancing. Roblox advises importing piece by piece, converting assets to packages, and reusing mesh/texture IDs. Resizing, rotating and overlapping a few meshes "can create varied environments with few draw calls". **[official]** — [Improve](https://create.roblox.com/docs/performance-optimization/improve); [Design for performance](https://create.roblox.com/docs/performance-optimization/design)
- A MeshPart's **Color** does not affect whether it is instanced. Meshes instanced together render in a single draw call. **[community, older: ~2019]** — [DevForum: Does Roblox use mesh instancing](https://devforum.roblox.com/t/does-roblox-use-mesh-instancing/399564)
- Decals, Textures and particles "batch poorly". Property changes on ParticleEmitters can have a "dramatic impact" on performance. **[official]** — [Improve](https://create.roblox.com/docs/performance-optimization/improve)
- You can check the draw-call count under Render Stats > Timing in Studio, or with Shift+F2 in the client. **[official]** — [Improve](https://create.roblox.com/docs/performance-optimization/improve)

**Budgets**
- Roblox's example budget: "stay below 1,000 draw calls and 1,000,000 triangles to run well on the baseline device". The page presents this as an example from one test, not a universal limit. It tells you to choose at least one baseline device and test on it throughout development. **[official]** — [Design for performance](https://create.roblox.com/docs/performance-optimization/design)
- Triangle count affects frame time but ranks below draw-call count in importance. **[official]** — [Improve](https://create.roblox.com/docs/performance-optimization/improve)
- Scale reference from a 200-avatar test scene: 2,639,899 triangles and 59,318 client instances with SLIM and streaming off, versus 173,213 triangles and 4,319 instances with them on (about 15x and 13x lower). That works out to roughly 13k triangles per full platform avatar. **[official, May 12 2026]** — [Introducing Support for Platform Avatars in SLIM](https://devforum.roblox.com/t/introducing-support-for-platform-avatars-in-slim/4632702)
- Moving about 1,000 zombie rigs to single-part **skinned** meshes made draw calls rise "from 300 to over 1000" (PC, Studio). This suggests skinned meshes did not instance in that setup. **[community, Dec 31 2025 – Jan 1 2026]** — [Struggling to efficiently move 1000 assemblies](https://devforum.roblox.com/t/struggling-to-efficiently-move-1000-assemblies/4203225)

**Mesh LOD / RenderFidelity**
- `MeshPart.RenderFidelity` defaults to **Automatic**: under 250 studs is Highest, 250–500 studs is Medium, and 500 studs or more is Lowest. **Precise** always renders the highest detail. **[official]** — [RenderFidelity enum](https://create.roblox.com/docs/reference/engine/enums/RenderFidelity)
- Roblox advises setting RenderFidelity to Automatic or Performance so meshes "can fall back to simpler geometry", and avoiding Precise on many meshes. **[official]** — [Improve](https://create.roblox.com/docs/performance-optimization/improve)
- The 2019 launch post described automatic LODs as the original mesh plus 2 auto-generated levels, each with about half the triangles of the level above. It said auto-LOD "works best for MeshParts without textures". **[official, older: 2019]** — [Levels of Detail for Mesh Parts](https://devforum.roblox.com/t/levels-of-detail-for-mesh-parts/280769)
- A user reported that RenderFidelity does nothing on low-triangle MeshParts. **[community, older: ~2020]** — [More RenderFidelity options](https://devforum.roblox.com/t/more-renderfidelity-options/460741)
- **Low-end draw distance:** the Fall 2026 roadmap plans to "Increase the minimum draw distance on low-end devices from 200 to 500 studs" (Late 2026). Today, low-end devices can therefore cut rendering as close as 200 studs. **[official, Sep 18 2026]** — [Creator Roadmap 2026: Fall Update](https://devforum.roblox.com/t/creator-roadmap-2026-fall-update/4880208)

**SurfaceAppearance / PBR**
- A SurfaceAppearance's look depends on the user's device and graphics quality level. Roblox advises previewing content at several quality levels. **[official]** — [PBR textures guide](https://create.roblox.com/docs/art/modeling/surface-appearance)
- Tinting with `SurfaceAppearance.Color` "does not affect performance", and reusing one color map with different tints saves memory. Most SurfaceAppearance properties can't be changed by scripts because the pre-processing is "too expensive to perform in-game". **[official]** — [SurfaceAppearance reference](https://create.roblox.com/docs/reference/engine/classes/SurfaceAppearance); [Memory docs](https://create.roblox.com/docs/projects/performance-optimization/memory)
- A bug report says SurfaceAppearance loads inconsistently on mobile and often shows at very low quality. The reports started in mid-December 2025, with textures of 256x256 and 128x128. **[community, 2026]** — [Surface Appearance loads inconsistently on mobile](https://devforum.roblox.com/t/surface-appearance-loads-inconsistently-on-mobile-and-often-displays-at-very-low-quality-even-if-the-phone-allows-it/4530853)
- Roblox publishes no numeric cost for normal, roughness or metalness maps on mobile (see Gaps).

**Textures and texture memory**
- Texture memory depends on pixel count, not file size: "a 1024x1024 pixel texture consumes four times the graphics memory of a 512x512 texture." Compressing files or removing alpha channels does not reduce memory. Guidance: usually at most 512x512 per image, under 256x256 for minor images, sprite or trim sheets, and one texture tinted many ways. **[official]** — [Memory docs](https://create.roblox.com/docs/projects/performance-optimization/memory); [Improve](https://create.roblox.com/docs/performance-optimization/improve)
- Built-in materials use far less memory than custom textures. **[official]** — [Design](https://create.roblox.com/docs/performance-optimization/design)
- The engine "starts with lower-quality textures" and raises quality based on device memory, distance and screen coverage. **[official]** — [Memory docs](https://create.roblox.com/docs/projects/performance-optimization/memory)
- **Texture streaming** began rolling out Dec 12, 2025. As of the Mar 23, 2026 update it is supported on Studio, PC, Mac, consoles, and high- and low-end Android and iOS. Low-end Android has "4k textures disabled". No creator action is required. **[official]** — [Introducing Texture Streaming](https://devforum.roblox.com/t/introducing-texture-streaming/4144855)
- Texture streaming coverage as of Mar 23, 2026 **[official]** — [Introducing Texture Streaming](https://devforum.roblox.com/t/introducing-texture-streaming/4144855):
  - Fully supported: MeshPart, SurfaceAppearance, Texture, Decal, MaterialVariant, Beam, ParticleEmitter.
  - Partial: base materials.
  - **Not yet supported:** 2D UI textures, SurfaceGui, Terrain, TerrainDetail, **Sky**, and bundled base materials (Plastic, **Water**, default Sky).
  - No plans: classic clothing, rbxthumb, **EditableImage**.
- Texture streaming numbers (Modern City template, one run): texture memory fell from 280 MB to 155 MB, total download from 236 MB to 110 MB, and time to baseline quality from 20.6 s to 3.0 s. Ideal quality from spawn takes about 50 MB, versus the legacy system "shooting up to 225MB". Best practices: unique textures, atlases only for parts usually seen together, and near-uniform texel density. **[official]** — [Introducing Texture Streaming](https://devforum.roblox.com/t/introducing-texture-streaming/4144855)
- At RDC 2026 (Sep 10–12, 2026), Roblox reportedly said texture streaming cut out-of-memory crashes by 11% and raised playtime by 1.9%, with stronger gains on Android and low-power devices. A later run (Aug 13–30, 2026) reportedly showed −13% crashes and +2.7% playtime. **[secondary live-blog]** — [Deusens RDC 2026 coverage](https://deusens.com/en/blog/roblox-developers-conference-2026-live-coverage)
- After texture streaming launched (Jul–Aug 2026), developers reported textures, particle emitters and beams unloading during long Studio sessions. Staff pointed to a fix in release 732, but some users reported regressions. **[community]** — [Studio Memory Issue Causing Textures to Unload](https://devforum.roblox.com/t/studio-memory-issue-causing-textures-to-unload/4639996?page=4)

**Transparency, shadows, culling, particles, highlights**
- Overlapping partially transparent objects force the engine to render those pixels several times. Avoid transparency values other than 0 and 1, and watch for transparency overdraw. **[official]** — [Improve](https://create.roblox.com/docs/performance-optimization/improve); [Design](https://create.roblox.com/docs/performance-optimization/design)
- Shadows degrade as graphics quality drops and are "disabled entirely below quality level 4". Set `CastShadow = false` on small or distant parts and on moving objects where possible. **[official]** — [Improve](https://create.roblox.com/docs/performance-optimization/improve)
- The engine does frustum and occlusion culling by default. **[official]** — [Improve](https://create.roblox.com/docs/performance-optimization/improve)
- Roblox automatically emits fewer particles at lower graphics settings. `:Emit()` bypasses this, but calling it every frame "lags lower end devices". **[community, older: 2022]** — [How could I bypass this?](https://devforum.roblox.com/t/how-could-i-bypass-this/1994624)
- **Highlights:** the client shows at most **255** simultaneous Highlight instances, and the rest are "silently ignored". Disabled Highlights still use slots, so delete them rather than disable them. The docs give no per-highlight cost. **[official]** — [Highlight reference](https://create.roblox.com/docs/reference/engine/classes/Highlight)

### Inferences
- **Fish meshes:** use one MeshPart asset per species (plus optionally one hand-made low-LOD asset), shared SurfaceAppearance or TextureID, and vary color through `Color` or `SurfaceAppearance.Color` tint. By the instancing rules, 300 fish across about 10 species should cost tens of draw calls, not hundreds. Verify with Shift+F2.
- **Avoid skinned meshes for the crowd.** The only measurement found shows skinned rigs multiplying draw calls (300 → 1000+). Keep skinning for player sharks and a few close "hero" fish.
- **Fog defeats automatic LOD.** If fog caps visibility at about 150–250 studs, almost every visible fish is under the 250-stud Highest-LOD threshold, so RenderFidelity Automatic saves almost nothing. Author fish at low triangle counts (a few hundred triangles is a reasonable target; this is a judgment call). Do manual LOD swaps by pooling simpler meshes for mid-distance fish.
- **Budgets:** treat 1,000 draw calls and 1M triangles as the ceiling for a mid-range baseline phone, and aim well below that for 2–3 GB Android. Confirm on hardware; no official low-end number exists.
- **Textures:** 256² for fish, at most 512² for hero sharks and key props. Tint instead of making color variants, and use trim sheets for reefs. Skybox and UI images are not streamed, so keep them small.
- **Overdraw:** underwater light shafts, bubble sprites, translucent jellyfish and stacked fog planes are the main transparency-overdraw risks on mobile. Use opaque or Neon materials and a few large layers.
- **No per-entity Highlights.** The 255 cap is a hard ceiling, not a budget.

### Gaps
- No official per-device-class draw-call or triangle budgets for low-end phones. No official cost figures for SurfaceAppearance PBR maps on mobile, Beams/Trails, Highlights or BillboardGuis.
- Whether skinned MeshParts can instance is not documented. Only one community measurement exists.
- The exact semantics of `RenderFidelity.Performance` were not captured.
- A claimed particle render cap (16,000 on desktop, 3,600 on mobile/Xbox) appeared in a search snippet, but none of the fetched sources contained it. Treat it as unverified.
- No source found saying whether texture streaming has expanded to Terrain, Sky or UI since the Mar 23, 2026 status update.

## 2. Lighting in 2026: technologies, what low-end mobile renders, cost of Atmosphere/fog/shadows/post-processing

### Takeaway
"Unified Lighting" (announced Jan 21, 2025; fully live Jul 23, 2025) deprecated `Lighting.Technology` in favor of `LightingStyle` (Realistic / Soft) plus `PrioritizeLightingQuality`. The old Voxel, ShadowMap and Future renderers still exist behind those settings. Compatibility lighting was removed in 2024 and migrated to Voxel plus Retro tone mapping. On low-end devices, shadows turn off below quality level 4, and draw distance can currently drop to 200 studs (500 planned for Late 2026). Roblox publishes no mobile cost numbers for Atmosphere, Bloom, DepthOfField, SunRays or ColorCorrection.

### Cited Findings
- Unified Lighting was announced Jan 21, 2025 and fully released Jul 23, 2025. It replaces the Technology property with two new non-scriptable properties: **LightingStyle** and **PrioritizeLightingQuality**. **[official]** — [Let There Be (Unified) Light!](https://devforum.roblox.com/t/let-there-be-unified-light-unified-lighting-is-fully-live/3401512)
- LightingStyle values:
  - **Realistic:** "High-fidelity lighting with detailed shadows and shading."
  - **Soft:** flatter lighting with diffused shadows; "the best option for achieving the classic Roblox look".
  **[official]** — [Unified Lighting announcement](https://devforum.roblox.com/t/let-there-be-unified-light-unified-lighting-is-fully-live/3401512)
- PrioritizeLightingQuality values:
  - **Enabled:** keeps lighting quality high by lowering other things, such as draw distance, first.
  - **Disabled:** lowers lighting quality first to protect draw distance.
  The Lighting docs say true favors advanced shadows and high-quality shaders at closer distances, and false favors view distance. **[official]** — [Unified Lighting](https://devforum.roblox.com/t/let-there-be-unified-light-unified-lighting-is-fully-live/3401512); [Lighting docs](https://create.roblox.com/docs/environment/lighting)
- Mapping from the old Technology values **[official]** — [Unified Lighting](https://devforum.roblox.com/t/let-there-be-unified-light-unified-lighting-is-fully-live/3401512):
  - Future → Realistic + Enabled
  - ShadowMap → Soft + Enabled
  - Voxel → Soft + Disabled
  - Realistic + Disabled "currently" uses Future technology with a shorter shading distance.
- The FAQ says performance "will not degrade" and that Roblox has seen "improved performance for the same perceptible visual quality" in some cases. The system "scales smoothly across all devices". No numbers or mobile details are given. **[official]** — [Unified Lighting](https://devforum.roblox.com/t/let-there-be-unified-light-unified-lighting-is-fully-live/3401512)
- Upcoming items: higher maximum range for local lights and better light attenuation. GPU Light Grid was "de-prioritized". A forum reply claims the light range is still capped at 60 studs; this is unconfirmed. **[official + community]** — [Unified Lighting](https://devforum.roblox.com/t/let-there-be-unified-light-unified-lighting-is-fully-live/3401512)
- Compatibility lighting was removed from the Technology dropdown, and places were auto-migrated to Voxel plus Retro Tone Mapping. **[official, 2024]** — [Compatibility Lighting becomes Retro Tone Mapping](https://devforum.roblox.com/t/compatibility-lighting-becomes-retro-tone-mapping-sunset-migration/3128560)
- A fan wiki dates that removal to the week of Oct 21, 2024, and claims Voxel was also "removed". That claim conflicts with the official Technology enum page, which still lists Voxel and describes Technology as superseded by LightingStyle. **[secondary vs official — conflict]** — [Roblox Wiki: Lighting](https://roblox.fandom.com/wiki/Lighting); [Technology enum](https://create.roblox.com/docs/reference/engine/enums/Technology)
- `ColorGradingEffect.TonemapperPreset` offers **Default** (the post-2019 look) and **Retro** (imitates the pre-2019 look). The post-processing page gives **no performance cost notes** for any effect. **[official]** — [Post-processing effects](https://create.roblox.com/docs/environment/post-processing-effects)
- `ShadowSoftness` only applies with LightingStyle Realistic. EnvironmentDiffuseScale and EnvironmentSpecularScale control ambient and reflected environment light. **[official]** — [Lighting docs](https://create.roblox.com/docs/environment/lighting)
- Shadows are disabled entirely below quality level 4. Other light guidance: set `Light.Shadows = false` where shadows aren't needed, limit light Range and Angle, use fewer lights, and disable lights by range or room. **[official]** — [Improve](https://create.roblox.com/docs/performance-optimization/improve)
- The QualityLevel enum runs from Level01 to Level21 plus Automatic. "On low quality levels some post effects may be completely disabled"; the excerpt does not list which. **[official]** — [QualityLevel enum](https://create.roblox.com/docs/reference/engine/enums/QualityLevel)
- A community wiki says the player slider has levels 1–10, and that level 1 has sun rays, depth of field and neon/bloom off. **[secondary]** — [Roblox Wiki: Graphics settings](https://roblox.fandom.com/wiki/Graphics_settings)
- Minimum draw distance on low-end devices is 200 studs today, with 500 planned for Late 2026. **[official, Sep 18 2026]** — [Creator Roadmap 2026: Fall Update](https://devforum.roblox.com/t/creator-roadmap-2026-fall-update/4880208)
- **Atmosphere:**
  - Density is the "amount of particles in the air"; higher values obscure more objects and terrain, but it does not directly affect the skybox.
  - "Fog properties are hidden when Lighting contains an Atmosphere object."
  - A low Offset causes "ghosting" (skybox showing through objects). A high Offset reveals level-of-detail "popping" on distant terrain and meshes.
  - No performance guidance is given.
  **[official]** — [Atmosphere reference](https://create.roblox.com/docs/reference/engine/classes/Atmosphere)

### Inferences
- **Lighting choice for this game:** A/B test `LightingStyle = Soft` (the old Voxel/ShadowMap path) against `Realistic` + `PrioritizeLightingQuality = Disabled` on the baseline phone.
  - Underwater scenes are dominated by fog and ambient light, so crisp shadows add little.
  - Phones below quality level 4 get no shadows anyway.
  - Roblox gives no mobile cost numbers, so measure it.
- **Moving off SMASH's Future + Bloom:** "Future" now means Realistic + Enabled, which by design trades away draw distance on weak devices to keep lighting quality. In a fog-capped ocean that trade is acceptable for distance, but per-pixel lighting cost is not free.
  - Set `CastShadow = false` on every fish and most props.
  - Use no shadow-casting local lights.
  - Build bioluminescence from Neon material or bright tinted textures rather than many PointLights. At most, use a small pool of short-range lights near the camera.
- **Post-processing:** drop DepthOfField and SunRays underwater. Treat Bloom as optional and subtle, since low quality levels may disable it anyway. Get the "underwater look" from ColorCorrection/ColorGrading tint plus Atmosphere or fog. Change these values only on depth-zone transitions or slowly, not every frame.
- **Use fog as a performance tool:** capping visible distance at about 200 studs or less hides the low-end 200-stud draw-distance cut, so all devices look the same. It also lets you shrink streaming radii and fish-render radii (see §3 and §5).

### Gaps
- No official millisecond or GPU cost for Atmosphere, Bloom, ColorCorrection, DepthOfField or SunRays on mobile. No official list of which effects turn off at which quality level; only a community wiki covers this.
- No documentation of the quality level that `Automatic` picks on 2–4 GB Android phones or older iPhones.
- Unclear whether Lighting.FogStart/FogEnd still apply at runtime when an Atmosphere exists. The docs only say they are "hidden".
- The local-light range cap (60?) is unverified.

## 3. Ocean/underwater visuals: Terrain water vs custom meshes, fog-capped visibility, caustics, skybox, making the ocean feel big

### Takeaway
There is no official benchmark of terrain water against mesh water on mobile. Community evidence says:
- Terrain water is light on the GPU but heavy on memory and physics, and has open bugs with streaming. Its material is read-only, and the water system has gone essentially unchanged for years.
- Skinned or static mesh water surfaces can be very cheap, but EditableMesh wave oceans are reported to be slow on mobile.

For a game played almost entirely underwater, the cheapest controllable approach is "fake water": fog or Atmosphere, color grading by depth, a simple surface sheet seen from below, and meshes or Expanded Terrain for the seabed.

### Cited Findings
- Terrain water's GPU cost is reported as very low, with memory as the main concern. **[community, older: ~2020]** — [How much does Terrain Water cause performance issues?](https://devforum.roblox.com/t/how-much-does-terrain-water-cause-performance-issues/539757)
- Terrain water is described as "not great for performance" over a large area and as causing ships to lag and fling, with a recommendation to use meshes. **[community, older: 2019]** — [Smooth Terrain vs. Meshes for an ocean map](https://devforum.roblox.com/t/smooth-terrain-vs-meshes-for-an-ocean-map-suggestions/324213)
- Terrain water "forces physics", which lagged one developer's game. **[community, older: 2021]** — [Realistic looking water](https://devforum.roblox.com/t/realistic-looking-water/1212799)
- A well-optimized mesh ocean loaded with far less lag than terrain water, though results depend on the code. **[community, older: 2021]** — [Making A Mesh Deformation Ocean](https://devforum.roblox.com/t/making-a-mesh-deformation-ocean/1122587)
- A skinned-mesh ocean costs about 0.05 ms per frame with nine planes. **[community, older: 2022]** — [Performant skinned mesh ocean](https://devforum.roblox.com/t/performant-skinned-mesh-ocean-with-built-in-support-for-floatable-objects-boats-ships-and-more/1820025)
- Adding depth-based coloring to an EditableMesh ocean cost about 1–2 ms. **[community, 2024]** — [Optimized EditableMesh Ocean Project](https://devforum.roblox.com/t/optimized-editablemesh-ocean-project/3236817)
- An EditableMesh Gerstner-wave ocean had "significant performance issues on mobile". **[community, 2025]** — [Simulated Ocean with EditableMesh](https://devforum.roblox.com/t/simulated-ocean-with-editablemesh/3562339?page=2)
- Terrain water renders wrong with StreamingEnabled. Filed in 2024, a reporter says it was still unfixed in early 2026. **[community]** — [Terrain water renders wrong with StreamingEnabled](https://devforum.roblox.com/t/terrain-water-renders-wrong-with-streamingenabled/3068489)
- Terrain water renders with holes and missing triangles; the workaround is filling water from a lower Y level. **[community, May 2026]** — [Terrain Water rendering bug](https://devforum.roblox.com/t/terrain-water-rendering-bug-with-holes-and-missing-triangles/4654068)
- An Aug 2026 feature request says the water system "hasn't changed in more than 5+ years" and asks for underwater effects and caustics. This is not a roadmap item. **[community]** — [More features to the terrain water system](https://devforum.roblox.com/t/more-features-to-the-terrain-water-system/4822004)
- The Shorelines update made water a separate terrain cell. **[official, older: 2023–24]** — [Improved Water with the Shorelines Beta](https://devforum.roblox.com/t/improved-water-with-the-shorelines-beta/2159630)
- **Expanded Terrain** reached full launch on Oct 22, 2026 **[official, Oct 22/27 2026]** — [Full Launch: Custom Terrain Materials and Performance Improvements](https://devforum.roblox.com/t/full-launch-publish-games-with-custom-terrain-materials-and-performance-improvements/4950620):
  - "In one benchmark, these updates reduced CPU memory overhead by 90% and meshing latency by 45%."
  - Terrain is ejected as a separate asset on publish.
  - Water and Air are read-only, so custom water is not possible.
  - Terrain thinner than 16 voxels "can disappear on clients" when streaming is on with a small radius.
  - **As of Oct 27, 2026, the performance improvements are active only in Studio**, because a client crash bug was found. The client re-enable may take "a few weeks".
- At RDC 2026, terrain early access (June 2026) was reported to cut file size by 98%, CPU memory by 90% and meshing latency by 45%. Maximum terrain area reportedly grew from 4×4 km to 16×16 km. **[secondary]** — [Deusens RDC 2026 coverage](https://deusens.com/en/blog/roblox-developers-conference-2026-live-coverage)
- Fall 2026 roadmap terrain and water items:
  - "Improvements to fluid dynamics": delayed to Early 2027.
  - Terrain virtual texturing, SDF terrain and a scattering system: Mid 2027.
  **[official]** — [Creator Roadmap 2026: Fall Update](https://devforum.roblox.com/t/creator-roadmap-2026-fall-update/4880208)
- Texture streaming does not yet cover Terrain, TerrainDetail, Sky, or the bundled Water material (as of Mar 23, 2026). **[official]** — [Introducing Texture Streaming](https://devforum.roblox.com/t/introducing-texture-streaming/4144855)
- Atmosphere density obscures objects and terrain but "does not directly affect the skybox". **[official]** — [Atmosphere](https://create.roblox.com/docs/reference/engine/classes/Atmosphere)
- Decals and Textures batch poorly, and partial transparency causes overdraw. **[official]** — [Improve](https://create.roblox.com/docs/performance-optimization/improve); [Design](https://create.roblox.com/docs/performance-optimization/design)
- Generic (non-Roblox) "poor man's caustics" technique: cycle a looping set of caustic frames, for example 32 images, or project a texture. **[secondary, generic engine forum]** — [Irrlicht forum: Water question](https://irrlicht.sourceforge.io/forum/viewtopic.php?p=18317)

### Inferences
- **Recommended baseline (to verify by prototype):** don't fill the open ocean with a terrain-water volume. Instead:
  - Fake the underwater look: Atmosphere or fog color and density driven by the camera's depth zone, ColorCorrection/ColorGrading tint, and a dark-blue or near-black low-resolution skybox. Atmosphere doesn't fog the sky, and Sky textures aren't streamed.
  - Use one or a few camera-following, opaque or mostly-opaque surface mesh tiles, seen from below, with a slowly scrolling texture.
  - Build the seabed from instanced rock and coral meshes, or from Expanded Terrain at least 16 voxels thick. Don't count on its client performance gains until Roblox re-enables them.
  - Trade-off: terrain water gives built-in swimming physics. Movement is out of scope for this report, but choose with the movement team.
- **Caustics on a budget:** two overlapping Texture objects with caustic patterns on seabed tiles near the camera, scrolled in different directions by animating `OffsetStudsU/V` at about 15–30 Hz. Alternatively, a small flipbook of 256² frames. Restrict this to shallow zones and the area around the camera, and disable it on low quality levels. This is unmeasured on Roblox, and Textures batch poorly and add overdraw, so keep the number of textured tiles small.
- **Making the ocean feel big while rendering little:**
  - Keep the visibility radius at about 120–200 studs, matched to the low-end draw distance and the streaming radii.
  - Parent a "marine snow" ParticleEmitter near the camera for parallax.
  - Show rare, huge, very low-poly silhouettes (whales, wrecks, kelp walls) through the fog.
  - Make each depth zone feel distinct through color, fog and light.
  - Use sound design.
  - The world can be several km wide while each client renders only a 200-stud bubble.

### Gaps
- No official mobile benchmark comparing terrain water and mesh water in GPU time, memory or physics cost.
- Terrain water's internal underwater rendering (its built-in underwater fog and tint) was not documented in any source fetched.
- No Roblox-specific measurements of caustic techniques or of the per-frame cost of animating Texture offsets.

## 4. Animating hundreds of fish cheaply

### Takeaway
The engine's animation LOD (`Animator.PreferLodEnabled`) is documented for **remotely-simulated characters**. For hundreds of client-side fish, drive transforms yourself.

Community measurements give the relative costs:
- Bone/joint `Transform` writes are the cheapest per write: about 0.002 ms per move, versus about 0.023–0.03 ms for a direct CFrame set. This is a single-object test from 2023.
- One batched `workspace:BulkMoveTo` (FireCFrameChanged) beats per-part CFrame writes beyond about 30 parts.
- About 1,000 moving assemblies cost about 6 ms per frame on PC with CFrame or BulkMoveTo, plus 2–3 ms of engine `notifyVisuallyMovingPrimitives`. Switching to Motor6D.Transform cut the movement code to about 1 ms.

Fish should use no Humanoids and should be client-only, pooled, and instanced. Use skinning only for the nearest few. SLIM and EditableMesh are not suitable for the fish crowd.

### Cited Findings
**Engine animation throttling**
- `Animator.PreferLodEnabled` (default **true**) lets "the engine ... reduce animation evaluation frequency for remotely-simulated characters" based on distance, screen coverage and frame budget. Setting it to false gives full fidelity, but "disabling LOD for many animators simultaneously can impact performance".
- `Animator.EvaluationThrottled` reports skipped frames, during which the previous pose is reused.
- `Workspace.ClientAnimatorThrottlingMode` toggles throttling for all animators.
- `LoadAnimation()` creates a new track on every call, "which may impact game performance if overused".
**[official]** — [Animator reference](https://create.roblox.com/docs/reference/engine/classes/Animator)
- Throttling history: introduced as `ClientAnimatorThrottling` in 2021, then updated to give smoother animation to characters closer to the camera and taking more screen space, then the Workspace property was phased out in 2022. **[official, older: 2021–2022]** — [New ClientAnimatorThrottling Property](https://devforum.roblox.com/t/new-clientanimatorthrottling-property/1553697); [Updates to Animation Throttling](https://devforum.roblox.com/t/updates-to-animation-throttling/1672224); [Phasing out Workspace.ClientAnimatorThrottling](https://devforum.roblox.com/t/phasing-out-workspaceclientanimatorthrottling/1968212)
- A 2026 bug report says throttling does not trigger under lag. **[community, 2026]** — [ClientAnimatorThrottling Isn't Working](https://devforum.roblox.com/t/clientanimatorthrottling-isnt-working/4645855)

**Roblox guidance for many NPCs**
- Play NPC animations on the client, and only for NPCs near the player.
- Disable unused HumanoidStateTypes.
- Use an AnimationController instead of a Humanoid for static NPCs.
- An `updateInvalidatedFastClusters` MicroProfiler tag over 4 ms means excessive avatar invalidation.
- Putting a Humanoid in a Model that moves skinned MeshParts avoids cluster rebuilds; use this only for parts that actually move.
**[official]** — [Improve](https://create.roblox.com/docs/performance-optimization/improve)

**BulkMoveTo / CFrame / Bones / Motor6D measurements**
- Bone `Transform` "always took 0.002ms" per move. Direct CFrame took "roughly 0.023-0.03ms" and BulkMoveTo "around 0.012-0.16ms" (the range is as written in the post, possibly a typo). The post concludes "bones are about 7.5x faster to move than BulkMoveTo" because bones don't update physics. This was a one-object test on Heartbeat. **[community, older: Sep 27 2023]** — [How Would I Optimize My Entity System In CPU #25](https://devforum.roblox.com/t/how-would-i-optimize-my-entity-system-in-cpu/2620118/25)
- A 2021 test found BulkMoveTo faster than per-part CFrame once more than about 30 parts are moved. **[community, older: 2021]** — [BulkMoveTo vs Normal CFrame-ing](https://devforum.roblox.com/t/bulkmoveto-vs-normal-cframe-ing/1278502)
- `workspace:BulkMoveTo(..., Enum.BulkMoveMode.FireCFrameChanged)` "increased my FPS by 70% when attempting to render 1000 NPC (HumanoidRootParts) on the client". The poster calls it "Not a serious benchmark". **[community, Dec 26 2025]** — [RTS with 200+ units thread](https://devforum.roblox.com/t/how-we-are-developing-an-rts-game-with-good-performance-for-more-than-200-units/3281168)
- The same RTS thread describes its architecture **[community, Dec 2024 – Aug 2026]** — [RTS thread](https://devforum.roblox.com/t/how-we-are-developing-an-rts-game-with-good-performance-for-more-than-200-units/3281168):
  - The server keeps only data, and clients create the unit models.
  - Unit parts are welded and moved by pivoting the PrimaryPart.
  - Parts are merged and RenderFidelity is set to Performance.
  - The server sends updates about every 10 frames.
  - The author claims 800+ units "without any problem".
- About 1,000 assemblies on PC (Studio), measured Dec 31, 2025 – Jan 1, 2026 **[community]** — [Struggling to efficiently move 1000 assemblies](https://devforum.roblox.com/t/struggling-to-efficiently-move-1000-assemblies/4203225):
  - CFrame or BulkMoveTo: about 6 ms per frame, plus `notifyVisuallyMovingPrimitives` at about 3 ms and rendering at about 6 ms.
  - Single-part skinned rig: BulkMoveTo dropped to about 2.5 ms, but draw calls rose from 300 to over 1,000.
  - Motor6D.Transform (root unanchored, Motor6D from Terrain to the root, CFrame never set): movement code fell from about 6 ms to about 1 ms. What remained was `applyJointTransforms` at about 2 ms and `notifyVisuallyMovingPrimitives` at about 2 ms.
  - Final result: 180 FPS in Studio, starting from about 45 FPS.

**Client-only instances, parallelism, editables, SLIM**
- Instances "created or cloned by client-side scripts are exempted from streaming out" (unless parented under a server-created instance). Client physics runs "only ... in streamed areas, even for locally created instances". **[official]** — [Instance streaming](https://create.roblox.com/docs/workspace/streaming)
- Scripts in a parallel phase "generally can't write to the data model". `RunService:BindToSimulationParallel` and `BindToAnimationParallel` exist for scripts under an Actor. **[official]** — [Parallel Luau](https://create.roblox.com/docs/scripting/multithreading); [RunService](https://create.roblox.com/docs/reference/engine/classes/RunService)
- **EditableMesh** **[official]** — [EditableMesh](https://create.roblox.com/docs/reference/engine/classes/EditableMesh):
  - Limit of 60,000 vertices and 20,000 triangles.
  - "Strict client-side memory budgets"; fixed-size meshes are more memory-efficient.
  - In published experiences it requires a 13+ ID-verified creator.
- **EditableImage** has strict client memory budgets. Multi-referencing one image saves memory, and `Destroy()` frees memory immediately. **[official]** — [EditableImage](https://create.roblox.com/docs/reference/engine/classes/EditableImage)
- A LocalScript hit the client budget at about 8 EditableMeshes, and EditableMeshes don't replicate. **[community, 2026]** — [EditableMesh restrictions](https://devforum.roblox.com/t/editablemesh-restrictions/4566101)
- SLIM excludes "models that play animations", models with a Humanoid, and models changed at runtime. **[official]** — [SLIM docs](https://create.roblox.com/docs/workspace/streaming/slim)
- "SLIM support for NPC characters" (lightweight animated versions of streamed-out NPCs) is planned for Early 2027. **[official]** — [Creator Roadmap 2026: Fall Update](https://devforum.roblox.com/t/creator-roadmap-2026-fall-update/4880208)

### Inferences
- **Fish render architecture:**
  - The AI and server simulation are out of scope here. Clients receive fish state and own the visuals.
  - Fish models are client-created and pooled, so streaming never removes them. The client must therefore cull them itself by distance and frustum, matched to the fog radius.
  - Set every fish part to `Anchored = true`, `CanCollide`/`CanTouch`/`CanQuery = false` and `CastShadow = false`. Client physics doesn't need them, and anchored moves skip assembly simulation.
- **Distance rings** (the numbers are judgment calls to tune on hardware):
  - Near (about 40–60 studs, at most about 20–30 fish): skinned fish, with tail wiggle written to 1–3 Bone `Transform`s each frame. This is the cheapest write per bone, but expect about one draw call per skinned fish.
  - Mid (about 60–150 studs): rigid instanced MeshParts, optionally a 2-part body plus tail both moved in the same BulkMoveTo. Update at 30 Hz or less.
  - Far / in fog (about 150–200 studs): update at 10–15 Hz, or replace a whole school with one merged "school impostor" MeshPart moved as a single unit.
  - Beyond the fog: not rendered at all, returned to the pool.
- **One `workspace:BulkMoveTo(parts, cframes, Enum.BulkMoveMode.FireCFrameChanged)` per frame** for all rigid fish:
  - Keep the parts and cframes arrays pre-allocated.
  - Compute CFrames in a parallel phase (Actors), then write them in the serial phase, since parallel code can't write to the data model.
  - The per-moved-part engine cost (`notifyVisuallyMovingPrimitives`) grows with the number of parts moved, so cut both the count and the update rate.
- **Never give fish Humanoids or Animators.** For player sharks (10–20), use an AnimationController plus Animator, or a Humanoid only if the movement team needs it. Load tracks once and reuse them. Engine LOD throttling is documented only for "remotely-simulated characters", so other players' sharks may get it, but client-created fish won't.
- **Avoid EditableMesh/EditableImage** for fish, oceans or crowds on mobile. The budget is small (about 8 meshes in one report), and it is gated by creator verification.
- **Impostors:** for distant schools, prefer merged-mesh schools (one draw call each) over BillboardGui-based sprites. BillboardGui cost wasn't measured in any source. ParticleEmitter "fish sprites" are possible, but particle counts are auto-reduced on low quality levels.
- **Luau vs GPU:** on low-end phones both matter. Lua-side CFrame math, the engine's moved-primitive bookkeeping, and draw calls each scale with fish count. The levers are fewer moving parts, lower update rates for distant fish, and instanced meshes.

#### What to do differently from SMASH (Future + Bloom, streaming off; accessories, highlights, nameplates and effects dominated with about 30 characters)
- **Characters:** each shark is one skinned MeshPart with a shared SurfaceAppearance per evolution stage, tinted by Color. Evolution swaps the mesh or appearance. There are no per-rig accessories, and cosmetics are merged into the mesh. This avoids the cluster rebuilds Roblox flags with `updateInvalidatedFastClusters` over 4 ms. ([Improve](https://create.roblox.com/docs/performance-optimization/improve))
- **Highlights:** none on fish or players by default. Use them only for one or two transient targets, and delete them rather than disable them, since disabled ones still use one of the 255 slots. ([Highlight](https://create.roblox.com/docs/reference/engine/classes/Highlight))
- **Nameplates:** BillboardGuis only for the nearest few players within a short distance, pooled, and with text updated only when it changes.
- **Effects:** pre-created, pooled emitters toggled with `Enabled` or `Emit`. Create no instances per hit or bite, keep few emitters with low rates, and keep textures at 256² or less.
- **Streaming:** turn StreamingEnabled **on**, with Opportunistic stream-out (§5).
- **Lighting:** Soft, or Realistic with Disabled, instead of Future + Bloom (§2).

### Gaps
- No mobile-device measurements of BulkMoveTo, Bone or Motor6D costs. All the numbers above are from PC or Studio.
- Not documented whether Animator LOD throttling applies to client-created NPC models, or whether skinned MeshParts can share instanced draw calls.
- BulkMoveTo's thread-safety level was not confirmed in the fetched API docs; it is inferred from the parallel-write restriction.
- No measured cost of BillboardGui nameplates or Highlights per instance.

## 5. Instance streaming and SLIM

### Takeaway
Instance streaming is Roblox's main client-memory lever. The defaults are StreamingTargetRadius 1024 and StreamingMinRadius 64. Roblox recommends **Opportunistic** stream-out (a StreamOutBehavior, not an integrity mode) and **PauseOutsideLoadedArea** integrity. For a fog-capped ocean, cut the radii well below the defaults.

SLIM generates composite LOD meshes in the cloud for **static** models (`Model.LevelOfDetail = SLIM`) and requires StreamingEnabled. Its timeline:
- Client Beta on PC/Mac: Oct 30, 2025.
- Client Beta on all platforms, including Android and iOS: Dec 9, 2025.
- Platform-avatar SLIM: May 12, 2026, in phases (updated Oct 20, 2026).
- Debug tints in the Roblox Player: Oct 9, 2026.
- NPC and welded-model support: planned for Early 2027.

A non-beta "live on mobile" status was not confirmed by an official post.

### Cited Findings
**Streaming settings** **[official]** — [Instance streaming](https://create.roblox.com/docs/workspace/streaming)
- StreamingTargetRadius defaults to **1024**; "Use the default of 1024".
- StreamingMinRadius defaults to **64**; "Use the default of 64". Content inside the minimum radius loads first and never streams out.
- The target radius should be larger than the minimum radius, or you get network pauses.
- StreamOutBehavior:
  - **LowMemory** (default): streams out beyond the minimum radius only under memory pressure.
  - **Opportunistic** (recommended): "regions beyond StreamingTargetRadius can be removed on the client even when there is no memory pressure", letting the client "aggressively garbage collect content".
- StreamingIntegrityMode: **PauseOutsideLoadedArea** is recommended. Gameplay pauses when the player moves into an unloaded region; `Player.GameplayPaused` reports this, and `GuiService:SetGameplayPausedNotificationEnabled(false)` hides the default modal.
- ModelStreamingMode:
  - **Nonatomic** (default): behavior follows Workspace.ModelStreamingBehavior (Legacy, or Improved, which is recommended).
  - **Atomic**: all initial descendants stream in together.
  - **Persistent**: never streams out, but is meant for rare cases, and overuse hurts performance.
  - **PersistentPerPlayer**.
- Extra ReplicationFocus points cost server work: "a single player with nine dynamically moving foci could generate server networking and streaming processing comparable to ten players". Too many foci also limit memory adaptation and raise the risk of the OS killing the app.
- Client-created instances are exempt from stream-out.
- To stream more aggressively, reduce StreamingMinRadius and StreamingTargetRadius and use fewer Persistent models. Streaming is the "highest-impact" memory option. **[official]** — [Improve](https://create.roblox.com/docs/performance-optimization/improve); [Memory](https://create.roblox.com/docs/projects/performance-optimization/memory)
- Streaming is recommended for most places because it improves join time, memory footprint and frame rate. **[official]** — [Design](https://create.roblox.com/docs/performance-optimization/design)
- A 2024 bug report says streaming-in can outrun memory cleanup, "most frequently" on Android and iOS. A commenter says only about 800 MB of a 2 GB phone was usable by Roblox. **[community, older: 2024]** — [Streaming Enabled Can Run Over Memory Management](https://devforum.roblox.com/t/streaming-enabled-can-run-over-memory-management-to-crash-client/3305729)
- Fall 2026 streaming roadmap **[official, Sep 18 2026]** — [Creator Roadmap 2026: Fall Update](https://devforum.roblox.com/t/creator-roadmap-2026-fall-update/4880208):
  - "Adaptive radius" ("Adjust each player's streaming radius based on their device's memory and CPU"): Late 2026.
  - "Path pre-fetching": Late 2026.
  - "Frustum streaming": delayed to Late 2026.
  - "Streaming excluded mode": delayed to Mid 2027.

**SLIM: what it is**
- SLIM uses cloud transcoding to build lightweight composite meshes with multiple LODs for Models and platform avatars. It "combines parts into fewer draw calls" and switches detail for a whole model at once. **[official]** — [SLIM docs](https://create.roblox.com/docs/workspace/streaming/slim)
- **Requirements:**
  - StreamingEnabled. Without it, SLIM models fall back to traditional LOD.
  - The place must be saved to Roblox and Team Create enabled.
  - Set `Model.LevelOfDetail = SLIM`. Legacy `StreamingMesh` models can be batch-converted.
  **[official]** — [SLIM docs](https://create.roblox.com/docs/workspace/streaming/slim)
- **Eligibility:**
  - Supported instance types: Part, MeshPart, Decal, Texture, SurfaceAppearance, PartOperation, NegateOperation, SpecialMesh.
  - Excluded: runtime-modified models, animated models, Humanoid models, and Scale-0 models.
  - "On platforms where SLIM is not yet supported, models set to SLIM fall back to Disabled."
  - The first generation can take minutes.
  - Known issues: PartOperation rendering, Z-fighting, UV shifts, and inaccurate triangle counts in render stats.
  - Debug with SlimTintMode.
  **[official]** — [SLIM docs](https://create.roblox.com/docs/workspace/streaming/slim)

**SLIM: rollout status**
- Oct 30, 2025: "now available in Client Beta on PC and Mac". **[official]** — [SLIM announcement thread](https://devforum.roblox.com/t/client-beta-introducing-scalable-lightweight-interactive-models-slim/4034709)
- Dec 9, 2025: "SLIM is now supported on all platforms in Client Beta". Android and iOS are marked yellow in the status list.
- Feb 25, 2026: the transcoders came back online.
- Originally only streamed-out models used SLIM, at a "comparable performance level" to the old ModelLoD. Plan: bring SLIM to live experiences platform by platform.
  **[official]** — [SLIM announcement thread](https://devforum.roblox.com/t/client-beta-introducing-scalable-lightweight-interactive-models-slim/4034709)
- Roblox's Dec 2025 newsroom post says SLIM reduces the draw calls, triangles and data-model instances a client must render, letting high-end PCs and low-end phones share one experience. **[official]** — [Introducing SLIM (newsroom)](https://about.roblox.com/newsroom/2025/12/introducing-roblox-slim-scalable-lightweight-interactive-models)
- Coverage of the Q2 2026 earnings call says SLIM is meant to let complex avatars run on low-end Android. **[secondary]** — [MobileGamer.biz](https://mobilegamer.biz/roblox-says-new-slim-tech-will-let-complex-avatars-run-on-low-end-android-as-it-pushes-mobile-first-game-creation/)
- Avatar SLIM (`Workspace.EnableSLIMAvatars`), posted May 12, 2026 **[official]** — [Introducing Support for Platform Avatars in SLIM](https://devforum.roblox.com/t/introducing-support-for-platform-avatars-in-slim/4632702):
  - Rolls out in 3 phases: opt-in (default Disabled), then opt-out (default Enabled), then the property is removed and SLIM becomes standard.
  - R15 platform avatars only. NPCs with Humanoids, R6, custom proportions, and post-load changes (including highlights) are not processed.
  - Draw calls are "slightly elevated" with SLIM on, under investigation.
  - The Oct 20, 2026 update added support for StarterCharacters whose LevelOfDetail is SLIM.
- Oct 9, 2026: debug tints were "Released" in the Roblox Player (Studio tints are still Studio Beta). The teal tint means "Full instances streamed in on the client; Lightweight SLIM rendering", so SLIM now also renders some fully streamed-in models. **[official]** — [SLIM debug tints](https://devforum.roblox.com/t/slim-debug-tints-in-studio-and-roblox-player/4924019)
- The Improve page now tells developers to set distant models' LevelOfDetail to SLIM and to enable EnableSLIMAvatars. **[official, 2026]** — [Improve](https://create.roblox.com/docs/performance-optimization/improve)
- An RDC 2026 live-blog says "SLIM compositing [has] shipped, benefiting all existing content automatically". This is unverified against an official post. **[secondary]** — [Deusens RDC 2026 coverage](https://deusens.com/en/blog/roblox-developers-conference-2026-live-coverage)

### Inferences
- **Recommended settings** (to tune on hardware):
  - `StreamingEnabled = true`.
  - `StreamOutBehavior = Opportunistic`.
  - `ModelStreamingBehavior = Improved`.
  - `StreamingIntegrityMode = PauseOutsideLoadedArea`, or a softer mode if fast-swimming sharks hit pauses. Test against path pre-fetching once it ships.
  - StreamingTargetRadius about 256–512: at least the fog distance plus a buffer for swim speed, far below the 1024 default.
  - StreamingMinRadius 64–128.
  - Revisit after the "adaptive radius" ships (planned Late 2026).
- **Static world:** group reef, rock and wreck clusters into Atomic models with `LevelOfDetail = SLIM`. If SLIM isn't supported on a device, they fall back to Disabled, so the risk is low. Inside a 200-stud fog bubble the "distant visuals" benefit is smaller, but SLIM's draw-call compositing can still help dense reefs.
- **Persistent:** reserve it for a tiny spawn or hub area at most.
- **Player sharks:** they are custom, non-R15 characters, so avatar SLIM almost certainly won't apply.
- **Fish:** they are animated and client-created, so model SLIM can't apply either, until the "NPC characters" support planned for Early 2027 (designed around Humanoid NPCs).
- **Thick seabed:** with small streaming radii, keep Expanded Terrain at least 16 voxels thick to avoid holes on clients.

### Gaps
- No official post confirms that SLIM has left Client Beta on Android or iOS for live experiences as of Oct 30, 2026.
- No published draw-call, memory or FPS numbers for **model** SLIM (only for avatars).
- Default, Disabled and MinimumRadiusPause integrity modes were not described on the fetched docs page.

## 6. Client Luau cost: native codegen on mobile, Parallel Luau, GC, RenderStepped vs PreRender/Heartbeat, per-frame budget

### Takeaway
Native code generation (NCG) for LocalScripts has **shipped on Android only**, per the Sep 18, 2026 roadmap and Sep–Oct 2026 community reports. iOS, Windows and Mac clients still interpret client code, and the docs still describe NCG as server-side. Parallel Luau (Actors) works, but parallel code can't write to the DataModel. `PreRender` supersedes `RenderStepped` and blocks rendering until it returns. At 60 FPS the whole frame is 16.67 ms; Roblox's example is about 5 ms of chunked work per frame. Luau GC atomic steps can pause for "tens of milliseconds", so per-frame allocation discipline matters.

### Cited Findings
- **Native codegen:**
  - The docs describe NCG for "server-side scripts", enabled per script with `--!native` or per function with `@native`.
  - It suits heavy numeric code "with few calls to Luau libraries or Roblox APIs", especially code that runs every frame. Type annotations (such as `Vector3`) avoid slow checks.
  - Limits: 64K instructions per block, 32K blocks per function, and "1 million instructions for the entire script". Past the overall limit, the rest of the code runs non-natively.
  - Native code uses extra memory.
  **[official]** — [Native code generation](https://create.roblox.com/docs/luau/native-code-gen)
- The original 2023 launch said: "The first full release of this feature will be restricted to Studio and server-side scripts". **[official, older: 2023]** — [Luau Native Code Generation Preview](https://devforum.roblox.com/t/luau-native-code-generation-preview-studio-beta/2572587)
- The Fall 2026 roadmap lists **"Android native code gen"** as **Shipped**: "Near-native execution speeds for local scripts on Android." No other client platforms are listed. **[official, Sep 18 2026]** — [Creator Roadmap 2026: Fall Update](https://devforum.roblox.com/t/creator-roadmap-2026-fall-update/4880208)
- Community confirmation: on Sep 3, 2026, "they actually did enable NCG for Android devices"; on Oct 29, 2026, "Android native codegen was recently added". No iOS or desktop-client mentions. **[community]** — [Enable --!native for clients (p.3)](https://devforum.roblox.com/t/enable-native-for-clients/3170510?page=3)
- An Aug 11, 2026 bug report quotes a client warning, "Native code generation is not supported on this device". LuauHeap showed `[native]` only on the server. **[community]** — [NCG warning inconsistency](https://devforum.roblox.com/t/native-code-generation-warning-message-inconsistency-and-confusion-supported-devices-give-no-warning-if-it-chose-not-to-compile-as-ncg/4804395)
- **Parallel Luau:**
  - Scripts under Actors run in parallel after `task.desynchronize()` or through `ConnectParallel`.
  - `require()` is not allowed while desynchronized, and parallel scripts "generally can't write to the data model".
  - Using more Actors balances load better "even if the device has fewer cores than Actors".
  - SharedTable gives no-copy cross-actor data.
  - The page says nothing about worker-thread counts on mobile.
  **[official]** — [Parallel Luau](https://create.roblox.com/docs/scripting/multithreading)
- **RunService:**
  - `PreRender` supersedes `RenderStepped` ("should be used for new work"), and `PreSimulation` supersedes `Stepped`.
  - For PreRender, "the engine cannot start to render the frame until code running in this event has finished executing". BindToRenderStep likewise blocks rendering.
  - Per-frame order: PreRender → PreAnimation → PreSimulation → PostSimulation → Heartbeat.
  - `BindToSimulationParallel` and `BindToAnimationParallel` exist for scripts under an Actor.
  **[official]** — [RunService](https://create.roblox.com/docs/reference/engine/classes/RunService)
- **Frame budget:** limit code in the per-frame RunService events and spread large tasks across frames. At 60 FPS each frame has a 16.67 ms budget. Roblox's example is "about 5 ms of work per frame" over 20 frames. Other guidance: prefer event-driven code, disconnect unused connections, and keep client-unneeded assets out of ReplicatedStorage. **[official]** — [Improve](https://create.roblox.com/docs/performance-optimization/improve); [Design](https://create.roblox.com/docs/performance-optimization/design)
- "To hit a client frame rate of 60, your CPU time must be below 16 ms." The default client frame cap is 60 FPS. **[official]** — [Performance dashboard docs](https://create.roblox.com/docs/production/analytics/performance); [Identify](https://create.roblox.com/docs/performance-optimization/identify)
- **Luau GC and allocation:**
  - The GC is incremental, and "it's possible, and even desirable, to request garbage collection every frame".
  - Atomic steps can cause "occasional pauses on the order of tens of milliseconds", and a high allocation rate forces "GC assists".
  - Avoid "temporary table and userdata creation" in tight loops.
  - Vectors are a native value type, giving "significantly smaller GC pressure".
  - Closures without upvalues can be cached, but mutable upvalues allocate.
  - Use `table.create` for known sizes. Direct builtin calls (for example `math.max(x,1)`) take the fastcall path.
  **[official Luau]** — [Luau performance](https://luau.org/performance)

### Inferences
- **Plan for the interpreter.** iOS devices and Android devices where NCG fails still run interpreted Luau. Put `--!native` and type annotations on hot client modules (fish transform math, spline or boids interpolation): it is a free win on Android and harmless elsewhere apart from a possible warning. Don't size the fish count around NCG speeds.
- **Per-frame client Luau target** (judgment): keep fish, VFX and UI updates to about 2–4 ms total on the baseline phone. Leave the rest of the 16.67 ms (or 33 ms if you accept 30 FPS on low-end) for engine simulation, `notifyVisuallyMovingPrimitives`, animation and rendering. Measure on device (§7).
- **Event placement:**
  - Use PreRender only for camera work.
  - Put fish interpolation in PreSimulation or Heartbeat, or in `BindToSimulationParallel` inside Actors for the math, followed by one serial BulkMoveTo.
  - Stagger distant-fish updates across frames: update 1/2 to 1/4 of the mid and far fish each frame.
- **Allocation hygiene:**
  - Reuse the BulkMoveTo arrays.
  - Don't create tables per fish per frame.
  - Keep fish state in flat arrays or buffers (coordinate with the networking/ECS researchers).
  - Avoid string building in per-frame UI.
  - CFrame values are userdata in Roblox, so each new CFrame per fish per frame adds GC pressure. Minimize this, for example by skipping unchanged fish.

### Gaps
- No official statement on NCG for iOS or Windows/Mac clients. Apple's JIT restrictions are a plausible blocker but are not documented in any fetched source.
- No documentation of Parallel Luau worker-thread counts or scheduling on mobile SoCs.
- No published per-frame Luau headroom figures for specific low-end phones.

## 7. Memory budgets and crash avoidance on 2–4 GB phones; profiling and testing tools

### Takeaway
Roblox staff say "the overwhelming majority of crashes on mobile are memory-related" (Nov 2025). Roblox publishes no per-device memory cap. Community data points: a 2 GB phone leaves about 800 MB for Roblox, and shipped games use about 300–900 MB on mobile.

The tools are:
- Creator Hub performance dashboards: memory usage percentage by device type, OOM exits, and crash rate by OS and memory.
- The on-device MicroProfiler, viewed in a desktop browser at the phone's IP on port 1338.
- The Developer Console Memory tab (GraphicsTexture, GraphicsMeshParts, LuaHeap, InstanceCount).
- The Studio Device Emulator, which is not accurate for memory. On-device testing from Studio is planned for Early 2027.

### Cited Findings
**Memory and crashes**
- A staff reply says "the overwhelming majority of crashes on mobile are memory-related" and generally within the developer's control. Crashes include OS low-memory-killer terminations; low FPS is not counted as a crash. Android telemetry changed on Nov 4, 2025 and reclassified some sessions. **[official, Nov 7 2025]** — [PSA: Mobile Client Crash Reclassification](https://devforum.roblox.com/t/psa-mobile-client-crash-reclassification-in-creator-hub/4053071)
- In the same thread, a community member says a decent-sized game can use "as low as 300-900MBs on Mobile". Staff attribute part of the mobile/desktop gap to 32-bit pointers on some devices and different graphics APIs. Staff also say the Studio emulator "at least tries to emulate lower mobile memory in some features like streaming". **[community + official]** — [PSA thread](https://devforum.roblox.com/t/psa-mobile-client-crash-reclassification-in-creator-hub/4053071)
- "One of the most common reasons for crashes on low end devices is due to out of memory errors." An insight for **<2 GB Android** crashes appears when at least 5% of concurrent users are on such devices and their crash rate is significantly worse than average. Client crash rate can be broken down by OS and memory. **[official, older: Sep 3 2024]** — [Analytics: Optimize low-end Android crash rates](https://devforum.roblox.com/t/analytics-optimize-low-end-android-crash-rates-server-memory-and-cpu-performance/3145364)
- "Roblox does not have access to all of a device's memory." Client memory matters most because mobile devices "typically have less available memory". The Studio device emulator "isn't accurate for memory usage" because Studio runs the server and client together. **[official]** — [Design](https://create.roblox.com/docs/performance-optimization/design); [Memory docs](https://create.roblox.com/docs/projects/performance-optimization/memory)
- Memory signals in the Developer Console:
  - LuaHeap: "High or growing consumption suggests a memory leak".
  - InstanceCount: steady growth means instances aren't being collected.
  - PlaceScriptMemory: per-script breakdown.
  - PhysicsParts: high values mean collision fidelity should be lowered. Use Box collision on small anchored parts and disable unneeded collisions, since collision geometry still uses memory.
  - Audio "can use surprising amounts of memory".
  **[official]** — [Memory docs](https://create.roblox.com/docs/projects/performance-optimization/memory)
- Read memory in the client, not Studio. Under PlaceMemory, check GraphicsMeshParts, GraphicsTexture and Sounds. **[official]** — [Identify](https://create.roblox.com/docs/performance-optimization/identify)
- Texture streaming cut texture memory from 280 MB to 155 MB in one template, and reaches ideal quality from spawn with about 50 MB. **[official]** — [Introducing Texture Streaming](https://devforum.roblox.com/t/introducing-texture-streaming/4144855)
- A user's 3 GB Android phone crashed in every experience within a minute; others reported games running fine on under-2 GB devices. **[community, 2025]** — [Roblox always crashes on android](https://devforum.roblox.com/t/roblox-always-crashes-on-android/3756150)

**Dashboards and analytics**
- **Client tab metrics:** frame rate, crash rate, unexpected out-of-memory exits, crash count, memory usage, **memory usage percentage**, and client CPU time.
  - Charts break down by platform or OS; memory charts break down by device type.
  - Values can be shown as P90, P50 or P10, and compared by place version.
  - If the Android memory-percentage line "approaches 100%", most Android devices are at risk of out-of-memory crashes. A spike after an update indicates a memory regression.
  - Charts need at least 100 DAU, and Roblox release annotations are shown.
  **[official, 2026]** — [Performance dashboard docs](https://create.roblox.com/docs/production/analytics/performance)
- Investigate if client crash rate rises above 2–3%. Watch for frames longer than 16.67 ms. **[official]** — [Identify](https://create.roblox.com/docs/performance-optimization/identify)
- Mar 17, 2026: server memory snapshots (a treemap under Monitoring > Crashes) for diagnosing **server** OOM crashes. This is not a client tool. **[official]** — [Analytics: Diagnose server crashes with server memory snapshots](https://devforum.roblox.com/t/analytics-diagnose-server-crashes-with-server-memory-snapshots/4525675)
- Fall 2026 roadmap **[official, Sep 18 2026]** — [Creator Roadmap 2026: Fall Update](https://devforum.roblox.com/t/creator-roadmap-2026-fall-update/4880208):
  - Shipped: Analytics alerts and custom analytics dashboards.
  - Late 2026: an "Observability platform" ("Use Microprofiler and Scene Analysis APIs to capture targeted performance data from your players") and "Client sessions & logs".
  - Early 2027: "On-device testing" ("Test games on real devices and debug them directly from Studio").

**Profiling tools**
- **On-device MicroProfiler:**
  - Turn on the MicroProfiler in the mobile client's Settings menu. The phone then shows an IP and port; browse from a computer on the same network to an address like `172.18.56.105:1338`.
  - If several IPs are shown, try each.
  - The web UI shows the last 30 frames by default; append a number for more (for example `/90`) and use Re-capture for a fresh set.
  **[official]** — [MicroProfiler docs](https://create.roblox.com/docs/performance-optimization/microprofiler)
- **Desktop and in-game tools:**
  - MicroProfiler: Ctrl+Alt+F6 (⌘⌥F6). It can dump frames for analysis.
  - Shift+F5: client debug stats.
  - Ctrl+Alt+F7: Performance Stats (memory, CPU, GPU, network, ping).
  - Shift+Ctrl+F1–F5: debug overlays.
  - Shift+F2: render stats with draw calls (from the Improve page).
  **[official]** — [Identify](https://create.roblox.com/docs/performance-optimization/identify); [Improve](https://create.roblox.com/docs/performance-optimization/improve)
- **MicroProfiler labels worth watching** for this game:
  - `notifyVisuallyMovingPrimitives`, `applyJointTransforms` and `BulkMoveTo` **[community]** — [1000 assemblies thread](https://devforum.roblox.com/t/struggling-to-efficiently-move-1000-assemblies/4203225)
  - `updateInvalidatedFastClusters` (over 4 ms is a red flag) **[official]** — [Improve](https://create.roblox.com/docs/performance-optimization/improve)

### Inferences
- **Working memory budget** (judgment from the community numbers above, not an official figure): treat about 800 MB as the hard ceiling on 2 GB Android and keep total client memory around 500–600 MB on 2–3 GB devices. A plausible split:
  - Textures under about 150 MB, which texture streaming now helps enforce.
  - Meshes under about 100 MB.
  - LuaHeap under about 100 MB.
  - Sounds loaded per zone.
  Verify with the dashboard's per-device memory percentage after launch.
- **Biggest memory levers for this game:**
  1. StreamingEnabled with Opportunistic stream-out and small radii (SMASH had streaming off).
  2. Few unique meshes and textures, reused and tinted.
  3. Fish and VFX pools with hard caps.
  4. Box or no collision on all decorative and fish parts.
  5. Per-depth-zone audio loading.
  6. No EditableMesh or EditableImage.
- **Test matrix:**
  - Pick 2–3 baseline devices: a 2–3 GB Android (a target for Android NCG), a 3–4 GB Android, and an older iPhone (no NCG).
  - Profile with the on-device MicroProfiler at 30/90 frames in a 300-fish scene with 20 sharks.
  - Read Dev Console Memory on device, never Studio.
  - Use the Studio emulator only for layout and streaming behavior.
- **After launch:**
  - Watch "unexpected out-of-memory exits" and memory percentage by device type.
  - Investigate any client crash rate above 2–3% and any post-update memory spike, compared by place version.

### Gaps
- No official per-device client memory caps for Android or iOS (for example, iOS jetsam limits on 3–4 GB iPhones).
- No source fetched documented how to open the Developer Console on mobile.
- No official data on how much memory the Roblox app itself uses before experience content on low-end phones. The 300–900 MB and 800 MB figures are community-reported.
