---
name: realistic-style
description: The owner's REALISTIC game style for Roblox - photoreal lighting and PBR materials, realistic 3D models and characters, and realistic first-person movement and camera. Use whenever the owner asks for a realistic, lifelike, photoreal, first-person, horror, job-simulator or "doesn't look like Roblox" game or asset, or says "realistic mode". Replaces the brainrot look in meow-house-style for that game; the universal house rules still apply.
---

# Realistic style

The owner's default style is the brainrot / simulator look in `meow-house-style`. When a game is **realistic**, this skill replaces that look. The goal: someone scrolling past the thumbnail or a clip thinks "wait, that's Roblox?".

## What changes vs. the brainrot style
| Brainrot style (meow-house-style) | Realistic style (this skill) |
|---|---|
| Chunky low-poly models, glowing cores | Real proportions, PBR textures, high detail inside Roblox limits |
| Bright saturated worlds | Natural light, real colour, contrast, atmosphere, weather |
| Thick cartoon UI, Fredoka One, lime pills | Clean minimal UI: thin fonts, dark glass panels, small icons, diegetic where possible |
| Third-person, Roblox default movement | **First person** with weight: acceleration, head bob, sway, footsteps, visible body |
| White-flash + shake + burst cutscenes | Cinematic cutscenes: real camera moves, depth of field, letterbox, sound design |

## Universal house rules that STILL apply (adapted)
- **Cutscene-style loading screen:** cinematic, e.g. a slow camera flythrough of the real map with fog and light rays, the title fading in with a sound hit, a thin progress line. See `meow-house-style/references/loading-tutorial.md` for the beats and tech; swap the cartoon look for this one.
- **Mandatory tutorial with spotlights:** same rules, realistic look: a soft vignette dim instead of a black overlay, a thin white outline / `Highlight` on the target, short objective text ("Pick up the scanner"), a subtle on-screen key prompt.
- **Animate everything** and **cutscenes for big moments**, but realistic: eased camera moves, no cartoon squash.
- **Game loop, upgrades, 2x → 100x passes and the Season 1 pass** still apply, themed realistically (e.g. "2x Pay", better tools, a "Season 1 Pass" with realistic cosmetics). Ask the owner once per game if a realistic game should skip any of them.
- **Same style, never the same game**, the "show your best friend" review, server-authoritative code, safe DataStores.

## Does NOT apply in realistic mode
`meow-house-style` rules 3 (UI look), 5 (chunky models) and 7 (bright saturated worlds), and its `bad-avoid.md` entries for realistic animals, dark worlds and thin minimal UI.

## Reference files (open only what the task needs)
| Task | File |
|---|---|
| Lighting, materials, atmosphere, post-processing, terrain | `references/visuals-lighting.md` |
| Realistic models, characters, textures, animation, asset sources | `references/models-realistic.md` |
| First-person camera, movement, body, footsteps, viewmodels | `references/first-person-movement.md` |

## Proof before showing the owner
Screenshot in Play mode at eye height in 3 spots (bright, dark, indoor), plus a short first-person walk clip, on PC and at phone size. If any shot looks "Roblox-y" (flat lighting, plastic material, default walk), fix it first.
