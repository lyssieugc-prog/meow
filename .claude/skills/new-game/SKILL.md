---
name: new-game
description: Start a brand-new Roblox game for the owner from just a name and a description. Use whenever the owner's message is basically a game name plus a pitch or description (e.g. "Pet Sky Sim - hatch pets on floating islands..."), or they say "new game", "make a game", "start a game", "build this game". The owner should never have to type commands or set anything up.
---

# New game (name + description → build)

The owner types a game name and a description, nothing else. Do everything below yourself, running any commands needed. Never ask the owner to run commands.

## 0. Get the latest vault
Quietly run `git pull` in this repo so the newest skills and rules are used. If it fails (offline, local changes), carry on with what's here.

## 1. Load the rules
Pick the style from the description: brainrot / simulator (default) or **realistic** (realistic, lifelike, first-person, horror, job simulator). For realistic, also load `realistic-style`. If it's unclear, make this one of your questions.
Load `meow-house-style` and follow **every** non-negotiable and the tool routing. In particular, the first build always includes:
- the cutscene-style loading screen and the mandatory spotlight tutorial (`references/loading-tutorial.md`),
- the constant-upgrade game loop and the Season 1 pass (`references/game-loop.md`),
- the 2x → 100x upgrading passes (`references/monetization.md`),
- animations and cutscenes on every feature.

## 2. Make the game's folder
Create `games/<game-name-in-kebab-case>/` in this repo (it inherits this repo's `CLAUDE.md` and skills). All of the game's code, UI, docs and assets go there. Commit after creating it.

## 3. Fill gaps, don't interrogate
From the name and description, work out: the core action, the theme, the upgrade tracks, the zones, the rebirth, the Season 1 pass theme, the tutorial steps and the loading-screen beats. Make confident, house-style choices. Ask **at most 3 short questions**, only for things that would change the whole game (e.g. "Is it a simulator or a tycoon?"). If the description already answers them, ask nothing.

## 4. Design brief → slices → stop
1. Write a short design brief with `roblox-to-prd` into `games/<name>/docs/brief.md`: pitch, core loop, the four loops, first 10 minutes, monetization, Season 1 pass, loading screen beats, tutorial steps.
2. Split it into small playable slices with `roblox-to-issues` into `games/<name>/docs/slices.md`.
3. Show the owner a short, friendly summary of the brief and the slice list (no code). **Stop and wait for their OK** before building.

## 5. Build slice by slice (after the OK)
- **Slice 1 is always:** the loading screen + the core action + the tutorial for it.
- Code through Roblox Studio MCP + Script Sync, Luau `--!strict`, server-authoritative.
- If Studio isn't connected, tell the owner in one plain sentence how to turn it on (Studio → Assistant → Manage MCP Servers → Enable Studio as MCP server), and keep writing files meanwhile.
- After each slice: playtest, fix errors, run the "show your best friend" review, commit, and show the owner what's new in plain language.
