---
name: studio-visible-build
description: The owner's rule that everything in a game must be real and visible in Roblox Studio's Edit mode - the map, every model, pet, egg and prop, and every UI screen - not created by scripts only when Play is pressed. Use at the start of every game and whenever building or changing a map, model, UI, VFX or any visual part of the owner's games.
---

# Everything visible in Studio (not only in Play)

The owner opens Studio and must **see the whole game without pressing Play**: the map, the models, the UI, the effects. Scripts make things *work*; they never *create* what the owner should be able to see and edit.

## The rule
- **Build things as real instances saved in the place**, never with `Instance.new` / code-generated parts or GUIs at runtime.
- **Scripts only bring things to life:** they move, animate, show / hide, clone templates and fill in numbers. They don't build the look.
- **Save the place after building** (File → Save to Roblox, or remind the owner) so the work isn't lost.

## Where everything goes
| What | Where it lives | Visible in Edit mode? |
|---|---|---|
| The map, zones, spawn, bases, signs, decorations | `Workspace`, organised in folders (`Map/Zone1`, `Map/Peak`, …) | Yes, as built |
| Every pet, egg, creature, tool, prop | A template in `ReplicatedStorage/Assets` (or `ServerStorage`) **plus a copy in `Workspace/Showcase`** | Yes, via the Showcase |
| HUD and every menu / popup | Real `ScreenGui`s with real Frames, buttons and images in `StarterGui` | Yes (see "UI" below) |
| Particles, beams, glows | Real `ParticleEmitter` / `Beam` / `Light` objects on the models and map | Yes (set `Enabled` on for showcase copies) |
| Lighting, sky, atmosphere | Real objects in `Lighting` | Yes |
| Sounds | Real `Sound` objects | n/a |

### The Showcase
- `Workspace/Showcase`: one of **every** pet, egg, creature, tool and prop, lined up on labelled podiums (name + rarity sign) near the spawn but outside the play area, with their effects on and idle animations previewable.
- A server script **deletes the Showcase when the game starts**, so players never see it. In Edit mode, it stays.
- When a new model is added, it goes into the template folder **and** the Showcase.

### UI
- Build every screen in Studio as real objects: HUD, shop, inventory, Index, Season Pass, settings, loading screen, tutorial overlay, popups, reward screens.
- `StarterGui.ShowDevelopmentGui = true`, so the GUIs show in the Edit viewport.
- The **HUD** is enabled. Every **other** screen is a separate `ScreenGui` with `Enabled = false`, so they don't pile up; the owner ticks `Enabled` to preview one. Scripts toggle them at runtime.
- Fill screens with **realistic sample content** (sample item cards, sample prices, sample pass tiers), so the owner sees the finished look, not empty frames. Scripts replace the sample content at runtime, using the sample cards as templates.
- Animations (pop-ins, staggers, wipes) are scripted, but every screen's **resting look** is built and visible.

### Things that must be created during play
Player-specific or random things (a player's base assignment, spawned eggs, clones, pets following players, damage numbers) are **cloned from visible templates**: the template is in the Showcase or the template folder and looks exactly like the in-game version. Never build their look in code.

## How Claude builds
- Use the Roblox Studio MCP to create the instances directly in the open place (or, if the game uses Rojo, as `.rbxm` / `.model.json` files that sync in). Scripts go through Script Sync / Rojo as usual.
- After building anything visual: check it **in Edit mode, not Play**, from the owner's point of view: the map is there, the Showcase has every model, every UI screen exists, sample content fills the screens. Take Edit-mode screenshots if the tools allow.
- Then playtest to check the scripts bring it to life correctly and the Showcase disappears for players.

## Checklist
- [ ] The map is fully built in `Workspace` and looks finished in Edit mode.
- [ ] Every model has a template and a Showcase copy with a label.
- [ ] Every UI screen exists as a real `ScreenGui` with sample content; HUD on, the rest previewable.
- [ ] No script builds a part, model or GUI from code (except cloning visible templates).
- [ ] The Showcase is removed for players at runtime.
- [ ] The place is saved.
