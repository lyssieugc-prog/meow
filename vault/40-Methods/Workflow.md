---
tags: [method, workflow]
---
# How Claude builds the owner's games

The owner pitches ideas; **Claude builds** the code, UI, models and VFX. The owner is not trying to learn to code. The owner trains Claude's taste by sending references with **GOOD** / **BAD** verdicts. Record each verdict in [[GOOD - UI]], [[GOOD - Models and Pets]] or [[BAD - Avoid]], always including motion ([[Cutscenes]]).

## Build loop
1. **Pitch → one-pager:** core loop, theme / element, monetization from [[Gamepasses and Products]].
2. **Model handoff doc first** (@vos_studiostt method): the game's style rules plus a part-by-part description of each creature / prop. Preview the 3D model (Claude artifact / Blender MCP), get approval, then build and export (OBJ / GLB → Studio).
3. **UI:** design in Figma or ForgeGUI art → import with figma-to-roblox-ui → script the [[UI Motion]].
4. **Code** in Luau (`--!strict`), server-authoritative, synced via Studio MCP / Script Sync or Rojo.
5. **Cutscenes / VFX** per [[Cutscenes]].
6. **"Show your best friend" review** (@momentumcash method) on every build: simulate a kid showing the game to their best friend. What would they be excited to show off? What would they skip? Improve the weakest parts.
7. **Promo:** a high-CTR thumbnail, a teaser trailer and a BEFORE / NOW devlog ([[World, Thumbnails, Promo]]).

## Notes
- It's never one prompt: expect testing, debugging and re-prompting per system (@andythropic).
- Commit to git before big requests.
