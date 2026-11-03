# Realistic models, characters and animation

## Models
- **Build in Blender** (house tool rule), export FBX / GLB, import with the 3D Importer. Real-world scale, real proportions.
- **Detail lives in the textures, not the triangles:** a clean mid-poly mesh plus baked normal map + PBR maps (`SurfaceAppearance`) beats a heavy mesh.
- **Roblox limits:** up to 20,000 triangles per MeshPart and 1024×1024 textures. Split big objects into several MeshParts; reuse one texture set (trim sheets / atlases) across many props.
- **LOD:** `RenderFidelity = Automatic` for big meshes; low-poly versions for far-away props.
- **Collision:** simple `CollisionFidelity` (Box / Hull) or invisible collision parts, never "PreciseConvexDecomposition" on detailed meshes unless needed.
- Bevel hard edges slightly so light catches them. Perfectly sharp 90° edges look CG.
- Still run the `mesh-qa` checks (holes, floaters, flipped faces); ignore its chunky-style budget and use the limits above instead.

## Characters
- Realistic humans: a custom rig with realistic proportions (skinned mesh), or R15-compatible realistic bodies. Check that clothes and heads fit the rig.
- Faces: Dynamic Heads (`FaceControls`) for blinking, expressions and talking in cutscenes.
- Animals: real anatomy, real fur / skin textures, realistic idle (breathing, ear and tail twitches, looking around).
- NPCs need life: idle variations, looking at the player (`IKControl` LookAt), reacting to sound.

## Animation
- Realistic animations: motion capture or a mocap library, retargeted in Blender, imported to the Animation Editor. Check every library's licence for game use.
- Blend everything: no snapping between animations (use fade times and weights).
- Foot planting with `IKControl` on slopes and stairs.
- Hands reaching for objects (IK) when picking things up.

## Asset sources (check every licence before use)
- Textures / HDRIs: CC0 PBR libraries (for example Poly Haven, ambientCG).
- Models: Sketchfab and similar sites: only licences that allow commercial use in games; credit if the licence asks.
- Marketplace / Creator Store meshes: run `roblox-creator-store-security-audit` on anything with scripts.
- Never rip assets from other games.
