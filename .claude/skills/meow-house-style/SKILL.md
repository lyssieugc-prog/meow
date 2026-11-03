---
name: meow-house-style
description: The owner's Roblox house style and standing rules. Use whenever designing, building, reviewing or pitching ANY part of the owner's Roblox games (UI, shops, pets, models, bosses, VFX, combat, cutscenes, maps, monetization, thumbnails, trailers), and whenever the owner sends a reference image or video with a GOOD / BAD verdict.
---

# meow house style

The owner pitches ideas; Claude builds everything. Match the owner's taste exactly. The full notes are in `vault/` (Obsidian); read the relevant note before building.

## Non-negotiables
1. **Animate everything.** UI pops with an overshoot, cards stagger in, claims flash + burst + stamp, screens change through transition wipes, idles never stop. Every feature has motion, camera and timing designed: `vault/20-Animation/`.
2. **Cutscenes for big moments.** Anticipation (≥1 s build) → impact (white flash + shake + burst, cut on the hit) → hold. Blackout before reveals, silhouette before the reveal, rarer = longer. End on a hero pose + title slam. `vault/20-Animation/Cutscenes.md`.
3. **UI look:** thick saturated title bar with an oversized tilted icon in the corner, a red square X, a dark navy panel, saturated cards (gradient + sunburst + stud texture + big 3D render), lime pill Robux buttons with a lip and a gloss stripe, a gift button beside every price, angled tags, rarity colours (rainbow Mythic, "???" silhouettes). Fredoka One with a thick dark stroke. Spec: `vault/10-Style/UI Spec Sheet.md`.
4. **Theme the frame to the game, and reskin it for events** (clouds for a sky game; pumpkins / webs for Halloween).
5. **Models:** chunky blocky / low-poly, but detailed through many small repeated pieces; one element / theme per model; a dark body with glowing eyes / core / cracks; built-in particles; idle loop. Never "a few boxes in one flat colour" or plain realistic animals. `vault/10-Style/GOOD - Models and Pets.md`, `vault/10-Style/BAD - Avoid.md`.
6. **Monetization:** always a "2x (stat)" that upgrades on every repeat purchase up to **100x** (2→3→5→10→25→50→100, as Developer Products with a saved level), plus engaging pay-to-win passes (Auto-*, +Equip, Luck, VIP exclusives, Starter Pack, timed and server boosts). Show odds on paid random items. `vault/30-Monetization/`.
7. **Bright, saturated, readable worlds.** No dark, muddy maps.

## Process
- Before any creature or prop: write a **model handoff doc** (style rules + part-by-part description), preview it, then build / export.
- After every build: run the **"show your best friend" review**. Simulate a kid showing the game to their best friend, list what they'd be excited to show off and what they'd skip, then fix the weakest parts.
- When the owner sends a reference: describe what you see, **including the animation beat by beat** (camera, timing, easing, VFX), wait for GOOD / BAD, then add it to the right vault note.

## Tech defaults
Luau `--!strict`, server-authoritative remotes and DataStores (use the `roblox-security`, `roblox-datastores`, `roblox-server-authority` skills), TweenService / Animator KeyframeMarkers for motion, Studio MCP + Script Sync. Tool setup: `vault/60-Tools-MCP/Tools - MCP.md`.
