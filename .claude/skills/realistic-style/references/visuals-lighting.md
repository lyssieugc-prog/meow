# Realistic visuals: lighting, materials, atmosphere

Lighting does most of the work. A plain map with great lighting looks more real than a detailed map with default lighting.

## Lighting (Lighting service)
- `Technology = Future`: real-time shadows and light from every light source. Mobile and low-end devices fall back automatically, so check the game still looks fine there.
- `EnvironmentDiffuseScale` and `EnvironmentSpecularScale` near **1**, so PBR materials pick up sky light and reflections.
- `GlobalShadows = true`; keep `ShadowSoftness` low-ish (0.1–0.3) for crisp outdoor shadows.
- Keep `Ambient` / `OutdoorAmbient` **dark**, so shadows have contrast. Bright ambient is what makes Roblox look flat.
- Set the time of day on purpose: golden hour (`ClockTime` around 6–7 or 17–18) looks the most real; noon looks the flattest. Use `GeographicLatitude` to angle the sun.
- `ExposureCompensation` to balance bright outdoor and dark indoor areas.
- Indoors: a few real light sources (`PointLight` / `SpotLight` / `SurfaceLight` with `Shadows = true`), warm vs. cool contrast. Every shadow-casting light costs performance; use as few as look right.

## Atmosphere and sky
- An `Atmosphere` object: some `Density` and `Haze` for depth, `Glare` near the sun, `Color` / `Decay` matching the mood. Distant things must fade into air.
- A realistic `Sky` (photo-based skybox) that matches the sun direction.
- `Terrain.Clouds` for real clouds (Cover, Density, Color).
- Weather when it fits: rain / fog particles, wet surfaces (lower roughness), wind sounds.

## Post-processing (in Lighting, or the camera)
- `ColorCorrectionEffect`: slightly more `Contrast`, slightly *less* `Saturation`, a subtle tint per area. Never oversaturate.
- `BloomEffect`: subtle (low intensity, high threshold), so only real highlights glow.
- `SunRaysEffect`: light, outdoors.
- `DepthOfFieldEffect`: for cutscenes and menus. Keep it off or very subtle in gameplay, since it blurs the whole screen.

## Materials
- **MeshParts with `SurfaceAppearance`**: ColorMap + NormalMap + RoughnessMap + MetalnessMap (PBR). This is the single biggest step away from "plastic Roblox".
- **`MaterialVariant`** in `MaterialService` for custom PBR on Parts and Terrain (set `StudsPerTile` to real-world scale).
- Avoid SmoothPlastic and Neon except for real light sources or screens.
- Add wear: dirt, edge scratches, decals for stains, puddles, cracks. Perfectly clean = fake.
- Textures upload at up to 1024×1024 (bigger ones are downscaled). Reuse texture sets across many meshes.

## Terrain and world
- Real terrain materials (custom `MaterialVariant`s), grass decoration on, rocks and props breaking up flat areas.
- Real scale: doors about 7–8 studs tall, ceilings 10–12, roads and furniture matching the character's size.
- Clutter with purpose: cables, signs, trash, leaves, furniture that tells a story. Empty rooms look fake.

## Performance (realism is expensive)
- `StreamingEnabled` on; `RenderFidelity = Automatic` on big meshes (LOD), and fewer, bigger texture sets.
- Limit shadow-casting lights and transparent overlapping effects.
- Test on a low-end phone with the `roblox-performance` skill before showing the owner.
