---
name: roblox-rojo-workflow
description: Write and change Roblox game code and instances through Rojo, the source of truth for the game. Use whenever writing, reading, editing, searching, or reorganizing Luau scripts, ModuleScripts, remotes, configs, or synced instances; setting up a Rojo project; or deciding where a file belongs in the place hierarchy.
---

# Roblox Rojo Workflow

## Rule: Rojo is the source of truth

The game's Luau code and synced instances live as **files in this Git repository**, mapped into the place by the Rojo project file (`default.project.json`). Studio is a **runtime for testing**, not an editor.

- **Read code from disk.** Use Read / Grep / Glob on the Rojo tree (`src/` or whatever the project file maps). Never read, search, or grep scripts out of Studio — pulling source from Studio is slow, token-heavy, and may be stale versus the files.
- **Write code on disk.** Edit `.luau` files with the normal file tools. `rojo serve` syncs them into Studio live. Never edit scripts in Studio, and never create scripts through Studio tooling — those changes are not in Git and Rojo will overwrite or orphan them.
- **Studio is only for testing.** Use `roblox-studio-testing` (the `rbxstudio` CLI) to playtest, read the console, drive input, and assert runtime state. Never call the `Roblox_Studio` MCP server's tools directly.

## Session start

1. Read `default.project.json` to learn how folders map to services (e.g. `src/server` → `ServerScriptService`, `src/shared` → `ReplicatedStorage.Shared`, `src/client` → `StarterPlayer.StarterPlayerScripts`).
2. Read `ROADMAP.md` and project instructions if present.
3. Start sync in the background if it isn't running: `rojo serve` (default port 34872). The user connects the Rojo plugin in Studio once per Studio session — if `roblox-studio-testing` shows edits aren't arriving, ask them to click **Connect** in the Rojo plugin.

## File conventions (Rojo 7)

| File | Becomes |
|---|---|
| `Name.server.luau` | `Script` |
| `Name.client.luau` | `LocalScript` |
| `Name.luau` | `ModuleScript` |
| `folder/init.luau` (`.server`/`.client`) | the folder becomes that script; siblings are its children |
| `Name.model.json` | instance tree declared as JSON (remotes, configs, values, simple GUIs) |
| `Name.meta.json` / `init.meta.json` | properties/attributes on the script or folder |
| `Name.rbxm` / `.rbxmx` | binary/XML model for complex instances (meshes, built GUIs, maps) |
| `Name.txt` / `.csv` / `.json` | `StringValue` / `LocalizationTable` / `ModuleScript` returning the data |

Declare remotes, tags-driven config folders, and constants as files (`.model.json` / ModuleScripts) — not as instances created by hand in Studio.

## Change loop

1. **Locate** with Grep/Glob in the Rojo tree.
2. **Read** the target file and its callers from disk. Never edit a file you have not read in this task.
3. **Edit** the file (smallest change that does the job).
4. **Static check** when the tools are installed: `selene src`, `stylua --check src`, `luau-lsp analyze` (or the project's configured equivalents from `rokit.toml` / `aftman.toml`).
5. **Verify sync** — `rojo serve` must be running and connected.
6. **Test** with `roblox-studio-testing`: playtest, read `rbxstudio console`, assert state with `rbxstudio exec`.
7. **Commit** the slice with Git once it is tested.

## What does not belong in Studio-only state

- Scripts of any kind.
- Remotes, config folders, attributes that code depends on.
- UI that code drives — build GUIs in code or as `.model.json`/`.rbxmx` under the Rojo tree.

Anything left only in the Studio place is invisible to Git and to every future agent, and may be deleted by Rojo when it syncs a managed container. If a large unmanaged area (terrain, hand-built map geometry) must live in the place file, keep it in containers the project file does not manage (or mark them with `$ignoreUnknownInstances`) and record that in the project instructions.

## Assets

Third-party or generated assets are evaluated in Studio as a test (see `roblox-asset-pipeline`), then **persisted into Rojo**: reference them by asset ID in a config ModuleScript or `.model.json`, or have the user save the vetted model as `.rbxm` into the Rojo tree. Don't leave an asset as a Studio-only instance.

## New project / migrating an existing place

- New: `rojo init`, then shape `default.project.json` to the layout above. Add `selene`, `stylua`, and `luau-lsp` via `rokit` if the user wants them.
- Existing place with code only in Studio: this is a **one-time, human-approved migration** (e.g. `rojo syncback` or exporting the scripts once). After that, files are canonical and Studio is never read for source again.

## Next

Build the change with `roblox-luau-architecture` / `roblox-ui-implementation` as needed, then verify it with `roblox-studio-testing` and `roblox-playtest-qa`.
