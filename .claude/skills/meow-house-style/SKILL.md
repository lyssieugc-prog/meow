---
name: meow-house-style
description: The owner's Roblox house style and standing rules. Use whenever designing, building, reviewing or pitching ANY part of the owner's Roblox games (loading screens, tutorials, UI, shops, pets, models, bosses, VFX, combat, cutscenes, maps, monetization, thumbnails, trailers), and whenever the owner sends a reference image or video with a GOOD / BAD verdict.
---

# meow house style

The owner pitches ideas; Claude builds everything. Match the owner's taste exactly. Open only the reference file the current task needs.

## Style mode (decide first)
- **Brainrot / simulator style (default):** everything in this skill.
- **Realistic style:** when the owner asks for a realistic, lifelike, first-person, horror or job-simulator game, load the **`realistic-style`** skill. It replaces rules 3, 5 and 7 and the matching `bad-avoid.md` entries; every other rule here still applies, adapted to a realistic look.

## Non-negotiables
1. **Animate everything.** UI pops with an overshoot, cards stagger in, claims flash + burst + stamp, screens change through transition wipes, idles never stop. Design the motion, camera and timing of every feature.
2. **Cutscenes for big moments.** Anticipation (≥1 s build) → impact (white flash + shake + burst, cut on the hit) → hold. Blackout before reveals, silhouette before the reveal, rarer = longer. End on a hero pose + title slam.
3. **UI look:** thick saturated title bar with an oversized tilted icon in the corner, a red square X, a dark navy panel, saturated cards (gradient + sunburst + stud texture + big 3D render), lime pill Robux buttons with a lip and a gloss stripe, a gift button beside every price, angled tags, rarity colours (rainbow Mythic, "???" silhouettes). Fredoka One with a thick dark stroke.
4. **Theme the frame to the game, and reskin it for events** (clouds for a sky game; pumpkins / webs for Halloween).
5. **Models:** chunky blocky / low-poly, but detailed through many small repeated pieces; one element / theme per model; a dark body with glowing eyes / core / cracks; built-in particles; idle loop. Never "a few boxes in one flat colour" or plain realistic animals.
6. **Monetization:** always a "2x (stat)" that upgrades on every repeat purchase up to **100x** (2→3→5→10→25→50→100, as Developer Products with a saved level), plus engaging pay-to-win passes (Auto-*, +Equip, Luck, VIP exclusives, Starter Pack, timed and server boosts). Show odds on paid random items.
7. **Bright, saturated, readable worlds.** No dark, muddy maps.
8. **Constant upgrades:** the player is never more than 30–60 s from the next upgrade. Use the four stacked loops (core → upgrades → rebirth → return) in `references/game-loop.md`.
9. **Every game ships with a "Season 1" pass:** a battle-pass-style pass named and themed after the game (free + Premium tracks; potions, money, eggs, boosts, exclusive Season 1 pet / tool), with Premium, tier skips and a season countdown.
10. **Every game ALWAYS ships with a cutscene-style loading screen:** eye-catching and heavily animated (logo slam with flash + shake, idle loop, themed wipe out), never a plain bar on black. See `references/loading-tutorial.md`.
11. **Every game ALWAYS has a tutorial new players MUST complete:** learn by doing, spotlight every target (dimmed screen with a cut-out, pulsing ring, pointing hand; world Highlight + arrow + path), saved server-side, ends with a reward cutscene. See `references/loading-tutorial.md`.
12. **Same style, never the same game.** Rules 1–11 fix the *style*; the *ideas* must be new every game. Never reuse a previous game's code, layout or concepts as a template. Invent fresh for each game: the loading-screen concept and logo animation, the transition wipe, the tutorial guide character and how spotlights move, the core action and its feel, the zones, the pets / creatures, the rebirth twist, the Season 1 theme and its exclusive. Before building, list 3 ideas for each of those and pick the most surprising one that still fits the house style. If something feels like "the last game reskinned", redo it.

## Tool routing (ALWAYS, owner's rule)
| Work | Tool |
|---|---|
| 3D models (pets, bosses, props, weapons) | **Blender** via Blender MCP; build and export GLB / FBX for Studio |
| Animations, VFX, 3D model previews / design | **Claude Design** (preview, model handoff doc, animation + VFX design), then build in Blender / Studio |
| UI and UI references | **Figma** via Figma MCP, imported with the `figma-to-roblox-ui` skill, then scripted with `ui-motion.md` |
| Anything else (sound, music, textures, icons, thumbnails, terrain, voice ...) | **Research the best MCP for the job first** (start with `references/tools-mcp.md` → "Other MCPs"), tell the owner which one and why, then use it |
| Code / place | Roblox Studio MCP + Script Sync |

## Process
- Before any creature or prop: write a **model handoff doc** (style rules + part-by-part description), preview it, then build / export.
- After every build: run the **"show your best friend" review**. Simulate a kid showing the game to their best friend, list what they'd be excited to show off and what they'd skip, then fix the weakest parts.
- When the owner sends a reference: describe what you see, **including the animation beat by beat** (camera, timing, easing, VFX), wait for GOOD / BAD, then append it to the matching reference file below. That file is the record.

## Reference files (open only what the task needs)
| Task | File |
|---|---|
| Loading screen, mandatory tutorial, spotlights | `references/loading-tutorial.md` |
| Shops, menus, HUD look | `references/ui-good.md` |
| Exact fonts, colours, component specs | `references/ui-spec.md` |
| UI animation | `references/ui-motion.md` |
| Cutscenes, reveals, trailers, camera | `references/cutscenes.md` |
| Weapons, abilities, combat VFX | `references/combat-vfx.md` |
| Pets, creatures, bosses, props | `references/models-pets.md` |
| What to never do | `references/bad-avoid.md` |
| Maps, thumbnails, promo videos | `references/world-thumbnails-promo.md` |
| Game loop, upgrade pacing, rebirth, Season Pass | `references/game-loop.md` |
| Gamepasses, products, pricing | `references/monetization.md` |
| Build process | `references/workflow.md` |
| MCP / tool setup (Studio, ForgeGUI, Figma, Blender) | `references/tools-mcp.md` |
| Saved inspiration links | `references/saved-tiktoks.md` |

## Tech defaults
Luau `--!strict`, server-authoritative remotes and DataStores (use the `roblox-security`, `roblox-datastores`, `roblox-server-authority` skills), TweenService / Animator KeyframeMarkers for motion, Studio MCP + Script Sync.
