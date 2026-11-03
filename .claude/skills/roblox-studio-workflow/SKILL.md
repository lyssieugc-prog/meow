---
description: Agent-driven Roblox development in current Studio — choosing Script Sync or Rojo, the official Studio MCP server and its tools, the evidence loop (edit, sync, play, read console, capture, stop), the Rojo deletion pitfall, and Team Create and Team Test limits. Use when connecting an agent to Studio, driving Studio through MCP, playtesting from an agent, reconnecting Rojo, or before saving or publishing a synced place.
---

# Roblox Studio Workflow for Agents

An agent that edits Roblox code without running it is guessing. Files changing is not proof, and neither is a clean type
check. This skill says how code reaches Studio, what the MCP server can do, and what counts as evidence.

## Script Sync or Rojo

Only documented or tested differences:

| | Script Sync (built into Studio) | Rojo (third party, not maintained by Roblox) |
| --- | --- | --- |
| Source of truth | Studio and disk, two-way | File system |
| What syncs | Only `Script`, `LocalScript`, `ModuleScript`, `Folder`; other instances ignored | Whole project: scripts, models (`.rbxm`), JSON, properties through the project file |
| Fits | Existing places; code in Git, everything else in Studio | Whole project in version control |
| Team Create | Works; shows who syncs what | The docs say third-party tools "often do not" work with the collaboration tools |
| Non-script instances | Preserved on delete | **Deleted if unknown under a `$path` node** (below) |
| Attributes and tags on scripts | **Ignored: don't sync such scripts, data loss risk** | Not compared here |
| Conflicts | Resolution dialog lists changes per side (Keep Studio or Keep Disk) | Not compared here |
| Limits | 10,000 scripts per top-level synced instance; 128 top-level instances | None documented that I found |
| Packages | Can sync; the `PackageLink` isn't written to disk | Wally through `$path` |
| Build a place without Studio | No | `rojo build` (tested) |
| Sync status from code | `InstanceFileSyncService` | No equivalent; compare the script text |
| Status | Guide `ai/coding-harness.md` calls it a beta in File → Beta Features; `scripting/sync.md` doesn't. Check the menu | Pin it in `rokit.toml` |

**File suffixes differ, and this bites:**

| On disk | Script Sync | Rojo, default | Rojo, `emitLegacyScripts: false` |
| --- | --- | --- | --- |
| `x.luau` | ModuleScript | ModuleScript | ModuleScript |
| `x.server.luau` | Script, `RunContext = Server` | Script, `RunContext = Legacy` (tested) | Script, `RunContext = Server` (tested) |
| `x.client.luau` | Script, `RunContext = Client` | LocalScript (tested) | Script, `RunContext = Client` (tested) |
| `x.local.luau` | LocalScript | **ModuleScript named `x.local`** (tested, Rojo 7.7.1) | Same |
| `x.legacy.luau` | Script, `RunContext = Legacy` | **ModuleScript named `x.legacy`** (tested) | Same |

Rule: pick one per subtree. Never point both at the same scripts: no document describes that combination.

Script Sync's gaps: per the harness guide it can't reach `StarterGui` or `StarterPack` (use MCP), and no external tool can control the Studio
debugger. Sync folders, not single scripts. Avoid several people editing the same synced script: they can overwrite each other.
To delete a sync root, use **Script Sync → Stop Sync** first; deleting it on disk can throw an error.

## The Rojo pitfall: syncing deletes what only the place has

Rojo's `$ignoreUnknownInstances` **defaults to `false` on any node with `$path`**, which means instances Rojo doesn't know about
under that node are deleted when it syncs. Without `$path` it defaults to `true`. Consequences:

- A model imported in Studio under a `$path` folder vanishes on connect, silently. Your local place is now missing it, and saving
  or publishing makes that permanent in the next version.
- One real project lost 25 models this way and published the broken place. Edits made in Studio to file-backed instances were
  also put back to the disk version on reconnect (field report, not documented behavior).

**General rules, for any sync tool:**

1. Before the first connect, list what's under each `$path`, and export place-only content to the repository, or keep it under a
   node without `$path`.
2. Before connecting, record counts of what must survive: `#folder:GetDescendants()` per key folder via `execute_luau`.
3. **After every connect or reconnect, and before every save or publish: run the play loop below and read the console output.** A game
   that checks its required assets at startup and logs what's missing turns this into one grep. Compare counts again.
4. Recover from [version history](https://create.roblox.com/docs/projects/version-history), then export what was lost.
5. Treat save and publish as outward actions. Only do them when the owner's rules allow.

## The Studio MCP server

Built into Studio, local process, `stdio`. Enable: Assistant → **…** → **Manage MCP Servers** → **Enable Studio as MCP server**.
Quick connect covers Claude Code, Codex CLI, Cursor, Gemini CLI, VS Code, Claude Desktop, Antigravity. On macOS the command is
`/Applications/RobloxStudio.app/Contents/MacOS/StudioMCP`; Windows runs `%LOCALAPPDATA%\Roblox\mcp.bat` through `cmd.exe /c`.
Connecting without opening the Assistant plugin is `Pending` (740), so if a client can't connect, open it. **Connected clients can read
and modify your open places.** `assistant/mcp.md` still links an open source server repo; the built-in one is what `studio/mcp.md` describes.

Every call takes a `studio_id`: get it from `list_roblox_studios`, and don't assume there is only one Studio.

**Documented in `studio/mcp.md`:**

| Group | Tools | Notes |
| --- | --- | --- |
| Scripts | `script_read`, `multi_edit`, `script_search`, `script_grep` | `multi_edit` creates the script if the path is missing, Edit data model only. Search: 10 results. Grep: 50 matches, so an empty-looking tail doesn't mean none |
| Exploration | `search_game_tree`, `inspect_instance`, `subagent` | |
| Luau | `execute_luau` | `datamodel_type` is Edit, Client, or Server |
| Playtest | `get_studio_state`, `start_stop_play`, `get_console_output`, `screen_capture` | State lists available data models |
| Input | `character_navigation`, `user_keyboard_input`, `user_mouse_input` | Can target UI instances or coordinates |
| Assets | `search_asset`, `insert_asset`, `upload_image`, `store_image` | |
| Generation | `generate_mesh`, `generate_material`, `generate_procedural_model`, `wait_job_finished` | |
| Docs | `http_get`, `skill` | `http_get` only reaches allowed Roblox docs URLs |
| Session | `list_roblox_studios` | |

**Present in this client but not in `studio/mcp.md`** (known only from the tool's own schema; expect change, and read the schema first):
`generate_texture` (replaces the source in place), `generate_scene` (runs a generated script in the DataModel), `generate_avatar`,
`auto_setup_avatar`, `segment_mesh`, `list_skills`, `create_skill` (publishes immediately), `edit_skill`.
Assistant Skills themselves are documented (`assistant/skills.md`), tied to the Roblox account and available in every place.

Doc vs. schema disagreement: the guide lists `subagent` types `explore` and `playtest`; the local schema lists `explore`, `screen_capture`,
and `unit_test`. Trust the schema of the client you're running.

From the `execute_luau` schema: code runs on Studio's main thread, and until it yields Studio is frozen and the MCP connection may drop. For bulk
work use chunks with `task.wait()`, and parent small subtrees, since the cost of a `.Parent` write scales with the subtree's size.

Many of these tools change the place or spend quota. Generation and insertion should follow an explicit request.

### Use the right edit path

| Workflow | Edit with | Why |
| --- | --- | --- |
| Rojo | Your file tools | Disk is the source of truth; a Studio edit doesn't outlive the next sync |
| Script Sync | Your file tools, or `multi_edit` | Two-way |
| No sync | `multi_edit` | Only copy is the place |
| `StarterGui`, instances, properties under Script Sync | `execute_luau` in Edit | Not reachable through files |

## The evidence loop

Run it for every behavior change. Each step produces something you can quote.

1. **Locate.** `list_roblox_studios`, then `get_studio_state`. Confirm the place is the right one and is in edit mode.
2. **Edit** files (see above), and run static checks: `stylua --check`, `selene`, `luau-lsp analyze` (see `setup`).
3. **Sync: prove it arrived.** Under Script Sync, in the Edit data model:

   ```lua
   --!strict
   local InstanceFileSyncService = game:GetService("InstanceFileSyncService")
   local ServerScriptService = game:GetService("ServerScriptService")
   print(InstanceFileSyncService:GetStatus(ServerScriptService))
   ```

   Statuses: `SyncedAsRoot`, `SyncedAsDescendant` are good; `NotSynced`, `Errored`, `AncestorErrored` aren't. `PluginSecurity`, not for game code; the harness guide calls it through MCP.
   `GetAllInstances()` and `GetSyncedInstance(absolutePath)` help too. Under Rojo, `script_grep` the line you just changed. Field report: sync reaches only the
   Edit model, so edit before starting play, and restart play after any change.
4. **Play.** `start_stop_play` with `is_start = true`, then `get_studio_state` until `Server` and `Client` are available.
5. **Exercise the real path.** `execute_luau` in `Server` or `Client`; drive with `character_navigation` and `user_*_input`. Fire the actual remote, and also try a rejected input.
   Server and client are separate data models, and modules are separate per environment, so check state on the side that owns it.
6. **Read the console.** `get_console_output`. Look for errors and warnings, not only your expected line. No output is not success: confirm your start line printed.
7. **Capture.** `screen_capture` for anything visual, then actually look at the image. Field reports: the tool's schema says "edit-time screen", while the Assistant guide says the viewport in Edit and Play;
   in one project it did not show CoreGui (backpack bar, chat). So check a phone UI with Device Simulator, and don't claim what a capture can't show.
8. **Stop.** `start_stop_play` with `is_start = false`, confirm with `get_studio_state`. Play-time changes are gone now, so anything you tweaked live has to go into the source.
9. **Report** what ran, the console lines, and what's still untested. Then, and only then, save or publish, and only if allowed.

### Multi-client and other testing

`start_stop_play` isn't documented as selecting a mode. For two players, use the Studio menu **Server & Clients** (up to eight), or `StudioTestService`
(`ExecuteMultiplayerTestAsync`, `AddPlayers`, `EndTest`: plugin-level, Studio-only, and it does nothing in published games). `StudioDeviceSimulatorService` switches devices and `VirtualInput`
simulates hardware input. Whether these all work from `execute_luau` wasn't verified, so try them and report. Network Simulator adds latency and loss, useful for Server Authority.

## Team Create and Team Test

- **Team Test:** one session at a time. Ending it closes it for all testers.
- Script Sync adds a numeric suffix to the name of any new instance under a tree that someone's syncing, to prevent duplicate names, so a name you expected may differ. Check the tree.
- **Drafts mode** is off by default (Experience Settings). When it's on, a script edit isn't shared until committed. Whether sync tools write to drafts isn't documented: confirm before relying on it.
- Team Create has age-based requirements for collaborators. A person who can't join a session still has Edit permissions, per `projects/collaboration.md`.
- MCP and a colleague share the same place. Don't start bulk edits in a shared session without saying so.

## Red Flags

| You're about to | Instead |
| --- | --- |
| Say "tested" after a clean `luau-lsp` run | Run the loop: that was only static |
| Save or publish right after reconnecting Rojo | Play, read the console, compare counts |
| Import a model in Studio under a `$path` node | Export it to the repo first, or expect it deleted |
| Edit a Rojo-managed script with `multi_edit` | Edit the file |
| Sync scripts carrying tags or attributes with Script Sync | Move them, or don't sync them |
| Run an unyielding `execute_luau` over thousands of instances | Chunk it with `task.wait()` |
| Treat empty console output as success | Confirm an expected line appears |
| Leave play running | Stop it and confirm the state |

When you (Claude) work on a Roblox place, identify the sync method first, prove every change with the loop, and quote the console output in your report. If Studio isn't reachable, say
that runtime behavior is untested. Never present a field report here as documented behavior.

## Sources

- https://create.roblox.com/docs/studio/mcp
- https://create.roblox.com/docs/ai/coding-harness
- https://create.roblox.com/docs/scripting/sync
- https://create.roblox.com/docs/projects/external-tools
- https://create.roblox.com/docs/reference/engine/classes/InstanceFileSyncService
- https://create.roblox.com/docs/reference/engine/enums/InstanceFileSyncStatus
- https://create.roblox.com/docs/assistant/guide
- https://create.roblox.com/docs/assistant/skills
- https://create.roblox.com/docs/studio/testing-modes
- https://create.roblox.com/docs/projects/collaboration
- https://create.roblox.com/docs/projects/version-history
- https://rojo.space/docs/v7/project-format/ and https://rojo.space/docs/v7/sync-details/

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
