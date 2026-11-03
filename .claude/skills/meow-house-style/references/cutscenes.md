# Cutscenes and Animation

> The owner's correction: **always account for ANIMATIONS AND CUTSCENES**, not just how things look. Every build and every reference breakdown covers motion, camera and timing.

## Rules
1. **Anticipation before every impact:** at least 1 s of build-up (glow, cracks, charging).
2. **Impact:** a 1–3 frame white flash, camera shake and a particle burst. **Cut on the impact**, then hold 0.5–1 s.
3. **Vary the shots:** wide establishing → close-up on the face → 3/4 action → low-angle hero → final hero pose with the title.
4. **Slow push-ins** during the build-up.
5. **Darken or black out the lighting right before the big reveal.**
6. **Silhouette before the reveal.**
7. **Rarer = longer, bigger cutscene.** Common is instant; Mythic / Secret gets the full sequence.
8. End on a **held hero pose with a title slam** (about 2.5 s).
9. Creatures **always idle**: breathing, bobbing, orbiting props, flickering glow.
10. Give players a **skip button** after the first viewing.

## Ice Golem trailer (@goldrushpets, about 20 s, about 10 shots)
| Time | Camera | Beat |
|---|---|---|
| 0–3 s | Wide, slow push-in | Still crystal; snow and aurora drift (anticipation) |
| 3–4 s | Close-up | Glowing cracks spread; the crystal blows out to white |
| 4.5 s | Shake | SHATTER: shards fly at the camera; golem revealed from a low angle |
| 5 s | Cut to the face | Eyes flare on |
| 6–7.5 s | Push-in | Roar; a white shockwave ring comes from the mouth |
| 8–10.5 s | 3/4 side | Frost-breath cone sweeps the floor |
| 11–13 s | Front, medium | Power-up pose; debris orbits the body; core pulses |
| 13.5–14.5 s | Low angle | Hero swing toward the camera |
| 15–16.5 s | Shake | Ground pound; ice spikes erupt; fog burst |
| 17–19.5 s | Locked | Hero pose; light pillar behind; "ICE GOLEM / COMING SOON" title slam |

## Fishing rarity reveal (@ropilot__ai, about 10 s)
1. The rarity **ladders up live**: purple Unusual → pink Rare → gold Legendary, about 1 s each. The line and the water ripple rings recolour at each step.
2. A white light pillar and a flash.
3. A hard cut to darkness, then a shockwave ring.
4. A lightning-rune crystal forms with crackling arcs and radial rays; slow push-in.
5. A neon square portal spins at the camera (transition wipe), into hyperspace streaks.
6. Dark space; a black silhouette grows toward the camera.
7. Flash → reveal on a rotating sunburst inside spinning tick rings; the item turntable-spins; the name plate types in ("ASTRAL LEVIATHAN" + rarity + stars).

Reuse this structure for egg hatches, crates, gacha rolls and boss spawns.

## Other motion references
- **Steal a Dragon portal shop:** a white silhouette inside a pink / purple portal with a light beam before the dragon is shown.
- **Sea-serpent dragon (@vos_studiostt):** rears up, wing-flap loop, takeoff, beam attack with smoke puffs, sparks and an impact ring.
- **Pets (@jubydandev / @boltzstudios0):** orbiting planets / crystals, sparkle trails, flickering flames, a bobbing idle.

## Roblox build approach
- Camera: `CameraType.Scriptable`, then `TweenService` on `CFrame` and `FieldOfView` (Quad / Sine Out for push-ins), plus a camera-shake module (decaying noise offset).
- Lighting: tween `Lighting.Brightness`, a `ColorCorrectionEffect` (flash: Brightness up for 1–3 frames; blackout: Brightness down, Saturation down) and a `BloomEffect`.
- VFX: `ParticleEmitter:Emit(n)` bursts, `Beam`s, neon parts, expanding ring meshes tweened in Size + Transparency.
- Animation: `Animator` tracks with **KeyframeMarkers** (`GetMarkerReachedSignal`) to fire VFX exactly on the frame.
- Client-side cutscene runner driven by a server RemoteEvent; the server still decides the reward.

See `ui-motion.md` and `combat-vfx.md`.
