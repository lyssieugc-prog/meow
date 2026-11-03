---
description: Scaffold a professional Roblox project with Rojo, Wally, Selene, StyLua, luau-lsp, strict Luau, and GitHub Actions CI, with pinned tool versions. Use when starting a new Roblox project or adding proper tooling to an existing one.
disable-model-invocation: false
---

# /roblox-dev:setup — Scaffold a Roblox Project

Scaffold (or retrofit) a Rojo-based Roblox project with linting, formatting,
type checking, and CI. `$ARGUMENTS` may contain the project name and options.
If the directory already holds a Roblox project (a `*.project.json` or `src/`
with Luau files), retrofit the missing pieces: never overwrite existing
configuration without showing a diff.

## First decide: Rojo or Script Sync

| The project… | Use |
| --- | --- |
| Is new, or wants the whole project as files, Wally packages, or a place built in CI | This scaffold (Rojo) |
| Is an existing place with content made in Studio, wants only code in Git, or uses Team Create | Script Sync: skip Rojo and write only `selene.toml`, `stylua.toml`, `.gitignore`, and a CI job without the Rojo steps |

The evidence behind this table is in `roblox-studio-workflow`. **Retrofitting
Rojo onto an existing place is the dangerous case:** Rojo deletes unknown
instances under any `$path` node, so a place-only model in that subtree is gone
on connect. Audit the tree and export those models to the repository first, and
don't connect until that is done.

## Steps

1. **Project name:** from `$ARGUMENTS` or the directory name. Ask only if the
   directory is non-empty and the name is ambiguous.

2. **Check versions.** The pins in the templates were the latest releases on
   2026-11-01. Compare them with
   `curl -s https://api.github.com/repos/<owner>/<repo>/releases/latest`, and tell
   the user if you changed any:

   | Tool | Repository | Pin |
   | --- | --- | --- |
   | Rokit (the installer, not listed in its own manifest) | `rojo-rbx/rokit` | v1.2.0 |
   | Rojo | `rojo-rbx/rojo` | 7.7.1 |
   | Wally | `UpliftGames/wally` | 0.3.2 |
   | Selene | `Kampfkarren/selene` | 0.32.0 |
   | StyLua | `JohnnyMorganz/StyLua` | 2.6.0 |
   | luau-lsp | `JohnnyMorganz/luau-lsp` | 1.70.1 |
   | CI action | `CompeyDev/setup-rokit` | v0.2.1 |

3. **Create the structure** (see `roblox-architecture`), writing each file from
   [references/templates.md](references/templates.md):

   ```text
   <name>/
   ├── rokit.toml
   ├── default.project.json
   ├── wally.toml            (only if the project will use packages)
   ├── selene.toml
   ├── stylua.toml
   ├── .gitignore
   ├── .github/workflows/ci.yml
   ├── README.md
   └── src/
       ├── shared/  Types.luau, Config.luau
       ├── server/  init.server.luau, Systems/
       └── client/  init.client.luau, Controllers/
   ```

4. **Install and verify. Run these, and report what they printed:**

   ```bash
   rokit install
   stylua --check src
   selene src
   rojo sourcemap default.project.json --include-non-scripts --output sourcemap.json
   curl -fsSL -o globalTypes.d.luau https://raw.githubusercontent.com/JohnnyMorganz/luau-lsp/1.70.1/scripts/globalTypes.d.luau
   luau-lsp analyze --sourcemap=sourcemap.json --definitions=@roblox=globalTypes.d.luau src
   rojo build default.project.json --output build.rbxl
   ```

   `rokit install` asks you to trust new tools when run interactively; the CI action
   passes `--no-trust-check`. If a tool is unavailable, say which check did not run.
   Never report it as passing. `luau-lsp analyze` warns and doesn't download
   definitions when `--definitions` is missing, so always pass it.

5. **Explain the toolchain** in a few lines, then hand over to
   `roblox-studio-workflow` for the edit → sync → play loop.

## What can go wrong

| Symptom | Cause | Fix |
| --- | --- | --- |
| `rojo build`: "could not be turned into a Roblox Instance", `File $path: Packages` | A `$path` target is missing (tested) | Run `wally install` first, or remove the node |
| Wally: "bad CPU type in executable" on an Apple Silicon Mac | The 0.3.2 release ships one macOS build, x86_64 (checked with `file`) | Install Rosetta, or remove Wally if there are no packages. Rosetta is a system change, so let the user decide |
| Client entry point isn't where you expect | `.client.luau` built as a `LocalScript` in ReplicatedStorage (not a documented container) because `emitLegacyScripts` is missing | Keep `"emitLegacyScripts": false` |
| Type check fails with "missing field 'start'" | A system in the bootstrap array lacks `init` or `start` | Export both; empty is fine |

## Syntax support (tested 2026-11-01 with the pinned versions)

- **`const`** passes StyLua, Selene, and luau-lsp, and is safe to use: the guide
  `luau/variables.md` documents it (status details in `luau-strict-typing`).
- **`if local x = f() then`** passes the three tools, but luau-lsp accepts it only
  because `analyze` enables all Luau flags by default: with `--no-flags-enabled`
  it is a `SyntaxError`. It appears nowhere in the Roblox docs or in release notes
  690–745. **Don't write it in Roblox code.**

## Rules

- All starter `.luau` files begin with `--!strict`.
- Add no packages the project doesn't need yet. Package names in `wally.toml` were
  checked against `api.wally.run`, including the ProfileStore scope
  (`lm-loleris`, not `loleris`).
- Commit `rokit.toml` and `wally.lock`. Wally's own README calls the lockfile what
  ensures everyone gets the exact same version of every package.
- The Luau files of the scaffold have not been run in Studio by this skill. After
  the first `rojo serve`, run the evidence loop in `roblox-studio-workflow` and
  look for both `started` lines in the console.

When you (Claude) finish a setup, report each command's real output and name
every check you could not run. "Scaffold created" is not a result.

## Sources

- https://create.roblox.com/docs/projects/external-tools
- https://create.roblox.com/docs/scripting/sync
- https://create.roblox.com/docs/scripting/locations
- https://create.roblox.com/docs/luau/variables
- https://rojo.space/docs/v7/project-format/
- https://github.com/UpliftGames/wally (README, lockfile format)
- https://github.com/CompeyDev/setup-rokit (`action.yml`)
- https://github.com/JohnnyMorganz/luau-lsp (README, `luau-lsp analyze --help`)

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
