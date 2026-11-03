---
description: Roblox project architecture — script locations and RunContext, ModuleScript patterns over _G, bootstrapping, require rules, remotes, CollectionService and QueryDescendants, Configs and experiments, packages and overrides, Actors, collision groups, Character Controller Library, PlayerModule, and logging. Use when starting a Roblox project, adding a system, restructuring spaghetti scripts, or deciding where code or a tunable should live.
---

# Roblox Architecture

A Roblox game that survives growth has few entry points, modules everywhere, one
obvious home for every piece of code, and remotes treated as a typed API boundary.

**Fact or convention?** The Engine enforces very little of this. Below, *Docs* means
the official guide recommends it, and *Convention* means it is this plugin's opinion.
Don't report a convention as a platform rule, and follow a project's existing
convention if it has one.

## Canonical Layout

```text
src/
├── shared/          → ReplicatedStorage/Shared
│   ├── Types.luau        (shared type definitions; see luau-strict-typing)
│   ├── Config.luau       (compile-time defaults)
│   └── Util/             (pure functions usable on both sides)
├── server/          → ServerScriptService/Server
│   ├── init.server.luau  (the server entry point)
│   └── Systems/          (one ModuleScript per system)
└── client/          → ReplicatedStorage/Client
    ├── init.client.luau  (the client entry point: Script, RunContext Client)
    └── Controllers/      (one ModuleScript per concern)
```

- **A single `Script` per side, everything else a ModuleScript.** *Docs recommend it*
  ("a single entry point"), and name exceptions: Creator Store models and packages may
  need scripts inside them, each with an explicit `RunContext`. A `ReplicatedFirst`
  loading script is another. It is a recommendation, not an Engine rule, and a game
  with more scripts still runs.
- **The client entry is a `Script` with `RunContext = Client` in ReplicatedStorage.**
  *Docs recommend it, and it replaces this skill's old "one LocalScript" rule*: use
  `LocalScript`s sparingly, because scripts in `Starter*` containers are cloned, so the
  Output window links to an ephemeral copy rather than the stable script. Under Rojo this
  needs `"emitLegacyScripts": false` (tested: without it `.client.luau` builds a
  `LocalScript`).
- Server-only code lives in `ServerScriptService`/`ServerStorage`, never in
  ReplicatedStorage "for convenience": the client can read all of it (see
  `roblox-security`).

### Script types and locations

| Script | Runs where |
| --- | --- |
| `Script`, `RunContext = Legacy` (default) | Server, only in a server container (`ServerScriptService`, `Workspace`…) |
| `Script`, `RunContext = Server` | Server; can also run in ReplicatedStorage (docs: don't, it replicates to clients) |
| `Script`, `RunContext = Client` | Client; can run in ReplicatedStorage. **Not in `Starter*` containers**: the original and the copy both run |
| `LocalScript` | Client; in `StarterPlayerScripts`, `StarterCharacterScripts`, `StarterGui`, `StarterPack`, or `ReplicatedFirst` |
| `ModuleScript` | Wherever it is required from; no run context |

Client startup order (*Docs*): `ReplicatedFirst` scripts run first and can't safely reach
other services. After `game.Loaded`, client scripts in ReplicatedStorage and `PlayerScripts`
run and can read ReplicatedStorage without `WaitForChild`. Instances the server creates at
runtime, and anything in Workspace, still need `WaitForChild`.

## Bootstrapping

```lua
-- src/server/init.server.luau
--!strict
type System = {
	init: () -> (),
	start: () -> (),
}

-- The array order is the startup order, and every entry is type checked.
local systems: { System } = {
	require(script.Systems.PlayerData),
	require(script.Systems.Shop),
}

for _, system in systems do
	system.init()
end
for _, system in systems do
	task.spawn(system.start)
end
```

Two phases (*Convention*; the docs' sample has just `start()`): `init` is synchronous and
wires dependencies, `start` runs long loops. Every `init` completes before any `start`,
which removes "A used B before it was ready" bugs without a framework.

**Don't loop over `GetChildren()` into a dictionary.** That's what this skill used to show,
and it called the result deterministic. The startup order was whatever the table iteration
gave, and a dynamic `require` loses the module's type.

## Modules and `require`

- `_G`/`shared` create invisible dependencies and defeat the type checker. **Require
  explicitly.**
- **One module instance per Luau environment, not per game.** `ModuleScript` runs once per
  environment and returns the same value after that. So the client and server have separate
  copies, the Command Bar has another, and **each `Actor` runs its own VM with its own copy, so
  module-level state isn't shared across Actors or with the main thread.**
- `execute_luau` over MCP: whether its `require` returns the running game's instance isn't
  documented. Probe it first; for QA changes use gated cheat remotes (see `roblox-testing`).
- A module that yields during `require` makes its requirer yield too.
- **Cycles error at runtime:** `Requested module was required recursively`. Extract the shared
  piece to a third module, or pass callbacks at `init`.
- Require forms: instance paths (`require(script.Systems.Shop)`) are what the docs' samples use.
  Require-by-string exists too, including the `@game` alias (700) and `@rbx/*` libraries; if you
  adopt it, confirm that your sync tool and `luau-lsp` resolve it first. `require()` can't be
  called in a desynchronized parallel phase.
- Cross-system events: expose a signal on the module (`Shop.ItemPurchased:Connect(...)`) rather
  than reaching into another system's internals.

## Remotes as a Typed API Boundary

*Convention:* create remotes on the server at startup, in one folder, from one list.

```lua
-- src/shared/Remotes.luau
--!strict
export type RemoteName = "BuyItem" | "EquipItem" | "StateSnapshot"
local NAMES: { RemoteName } = { "BuyItem", "EquipItem", "StateSnapshot" }
return table.freeze(NAMES)
```

The client gets them with `WaitForChild`, since they exist only after the server creates them.
Wrap access in a small module (`Net.fire(name, ...)`, `Net.on(name, handler)`) so adding a remote
is one line, every handler passes through validation and rate-limit middleware (see
`roblox-security`), and payload types sit next to the names (see `luau-strict-typing`). Never let
the client create remotes.

## World Objects: Tags, Attributes, Queries

Docs: avoid scripts inside Workspace instances; tag them and run one manager.

```lua
--!strict
local CollectionService = game:GetService("CollectionService")

local TAG = "Chest"
local cleanups: { [Instance]: () -> () } = {}

local function detach(chest: Instance)
	local cleanup = cleanups[chest]
	if cleanup then
		cleanups[chest] = nil
		cleanup()
	end
end

local function attach(chest: Instance)
	detach(chest)
	local connection = chest:GetAttributeChangedSignal("Tier"):Connect(function()
		print(chest:GetAttribute("Tier"))
	end)
	cleanups[chest] = function()
		connection:Disconnect()
	end
end

for _, chest in CollectionService:GetTagged(TAG) do
	attach(chest)
end
CollectionService:GetInstanceAddedSignal(TAG):Connect(attach)
CollectionService:GetInstanceRemovedSignal(TAG):Connect(detach)
```

- Handle both "already tagged" and "added later". Under streaming, these signals also fire on
  stream in and out, indistinguishable from real spawns and removals, so never treat "removed" as
  "destroyed".
- Per-instance *data* goes in Attributes, per-instance *behavior* in the manager.
- Tag in code or in the Studio Tags section: the docs show both. *Convention:* for what server logic
  depends on, tagging from a known folder at startup is easier to audit. Script Sync ignores tags and
  attributes on scripts, so don't put them there.

**`Instance:QueryDescendants(selector)`** (GA, 697; `:not`, `:has`, `[$Attr]` in 700) returns a
one-time array by selector. It is not a live set, so it doesn't replace the signals above.

```lua
--!strict
local Workspace = game:GetService("Workspace")

local tierThree = Workspace:QueryDescendants("Model.Chest[$Tier = 3]")
local unlocked = Workspace:QueryDescendants(".Door:not([$Locked])")
print(#tierThree, #unlocked)
```

Selectors: `Class`, `.Tag`, `#Name`, `[Prop = v]`, `[$Attr]`, `[$Attr = v]`; combinators `>`, `>>`, `,`.
To test an attribute's absence use `:not([$Attr])`, never `= nil`. **`Pending`, don't depend on them:**
inferred return types (737) and the spatial in-radius filter (740).

## Tunables: Config file, Configs, Experiments

| Need | Home | Why |
| --- | --- | --- |
| Constant shared by both sides, changes with a publish | `shared/Config.luau` | Readable by clients, nothing secret |
| Change without a publish or restart, kill switch, timed event | **Configs** (`ConfigService`), GA | Server only: calling it from a client errors. Up to 1,000 active |
| Different values per player segment | `ConfigService:GetConfigForPlayerAsync(player)` | Conditional configs |
| Measure the effect of a change | **Experiments** | In-game experiments are built on configs; 14–60 days, up to two variants plus control |
| Value only the server may know | ServerStorage or a server module | Never replicated |

Configs traps: a `ConfigSnapshot` is a point in time and **doesn't update by itself**. Connect
`UpdateAvailable` and call `Refresh()` when your game can accept new values. `GetConfigAsync` can throw
if it never loaded, and `GetValue` returns `nil` for unknown keys, so keep fallbacks. New values are
*staged*, visible in Studio play sessions only, until published. A client gets values only through your
own replication, and the server still validates against its own copy.

## Packages

Packages share one asset across places. Script edits in an unmodified package are locked until you unlock
them, and doing so flags the package as modified. **Package Overrides is a Studio Beta** (File → Beta Features,
Studio 743+, not yet in the guide): local edits are kept and reapplied on update, and yours wins on the same
property. Known issues per its announcement: everyone in a Team Create session must enable the beta or their
edits aren't tracked, and overrides inside nested packages can be mistaken for local edits. Don't build a
pipeline on it without saying so.

## Parallelism: Actors

For a measured hot loop that splits into independent chunks (see `roblox-performance`). Scripts under one Actor run serially
relative to each other, so speedup needs several Actors, and being under one isn't enough: call `task.desynchronize()` or
`ConnectParallel`, then `task.synchronize()` before unsafe writes. Messages (`SendMessage`, `BindToMessage[Parallel]`) pass
arguments **by copy**; shared state goes in `SharedTable`. A singleton module is a different instance inside each Actor.

## Collision Groups (732)

`PhysicsService` **is deprecated**; existing calls forward to `Workspace`. Use `WorldRoot`:
`workspace:RegisterCollisionGroup(name)`, `workspace:CollisionGroupSetCollidable(a, b, false)`,
`part.CollisionGroup = name`. A `WorldModel` now has its own groups, unless
`UseWorkspaceCollisionGroups = true`. Registering has a cost that scales with part count, so docs
recommend registering at edit time.

## Character and Player Scripts

| Feature | Status | What to know |
| --- | --- | --- |
| Character Controller Library (`require("@rbx/AvatarAbilities")`) | **Studio Beta**: File → Beta Features, then Avatar Settings | Abilities replace the `Humanoid` state machine |
| Default Sprint and Crouch | **Beta** (with CCL) | Enable per experience in Avatar Settings. Sprint is 1.5× speed, Crouch 0.5× speed; both default to Toggle |
| Custom Abilities API | **Beta**, had breaking changes on 2026-10-08 | `RunsWhile` takes world conditions only, input goes in `Input.Mode`, duplicate `Name` errors. Targets Server Authority |
| `Workspace.PlayerScriptsUseInputActionSystem` | Opt-in, set in Studio | Default player scripts move under `StarterPlayer`, run on the Input Action System. **Required by Server Authority** |
| `StarterPlayer.CreateDefaultPlayerModule` | Live; plugin or Open Cloud only (745) | Visible only with the opt-in above. `false` means no default camera and control scripts |
| `require("@rbx/PlayerModule")` | Live (735), with the opt-in only | Only the release note mentions it; its API isn't in the docs, so inspect it before use |

CCL compatibility: per its announcement (at that post's date), not recommended with Tools, Inventory/Backpack, or
Health. Confirm in the current thread before adopting. Details: `roblox-input`, `roblox-server-authority`.

## Logging and Errors

- **Structured logging is `Pending`** (716, 727): `LogService:Output/Info/Warn/Error/Log(message, context)`, with
  `{key}` placeholders. `LogService:Error` *always throws* a table. The members are in the reference, but check with
  a probe in the target Studio before shipping code that depends on them; `print`/`warn` remain the safe default.
- **Server error logs on Creator Hub link to script and line** in the exact place version (announced 2026-10-23; not yet
  in the docs snapshot). Frames from scripts cloned at runtime have no stable reference (`isDynamic`), one more reason not
  to clone scripts. Server Management Service exposes the data as `structuredStackTrace`.

## Where Does This Code Go?

| Code | Home |
| --- | --- |
| Validates, mutates, or stores game state | `server/Systems/` |
| Renders state, plays effects, reads input | `client/Controllers/` |
| Pure logic both sides run (damage formula, level curve) | `shared/Util/`; server result is authoritative, client copy is for prediction/UI only |
| Types crossing the boundary | `shared/Types.luau` |
| Tunable numbers | See *Tunables* above |
| Per-instance parameters | Attributes on the instance |
| Assets the client needs | ReplicatedStorage |
| Assets spawned by the server only | ServerStorage |
| Loading screen | `ReplicatedFirst`, one client script, minimal modules |
| How code reaches Studio | `roblox-studio-workflow` |

## Growth Rules

- New feature = new module + entry in the bootstrap array + remotes in the list + types in `Types.luau`. If it needs edits
  in five unrelated files, fix the boundaries first.
- Establish **one fully worked exemplar system** and make later systems copy its shape.
- Delete dead paths when replacing a system; parallel old and new paths are where duplication bugs live.

## Red Flags

| You see | Do this |
| --- | --- |
| `PhysicsService:*CollisionGroup*` | Move to `workspace:` (deprecated service) |
| `for _, m in folder:GetChildren() do require(m)` as startup order | Explicit ordered array |
| Module state "shared" with code under an Actor | Separate VMs: use messages or `SharedTable` |
| `Script` with `RunContext = Client` in `StarterPlayerScripts` | It runs twice: move it to ReplicatedStorage |
| `LocalScript` in ReplicatedStorage | Not a documented container: use a `Script` with `RunContext = Client` |
| Client calling `ConfigService` | Errors: read on the server and replicate |
| Code depending on a `Pending` or Beta API with no note | State it, and probe it |

When you (Claude) add any new system to a Roblox project, follow this layout and bootstrap, and produce the server module,
client controller, shared types, and remote registration together, not only the piece asked for if the rest doesn't exist
yet. Label each structural recommendation you make as docs or convention, then prove it with the loop in
`roblox-studio-workflow`.

## Sources

- https://create.roblox.com/docs/scripting/locations
- https://create.roblox.com/docs/scripting/module
- https://create.roblox.com/docs/scripting/attributes
- https://create.roblox.com/docs/reference/engine/classes/ModuleScript
- https://create.roblox.com/docs/reference/engine/classes/Actor
- https://create.roblox.com/docs/scripting/multithreading
- https://create.roblox.com/docs/reference/engine/classes/Instance#QueryDescendants
- https://create.roblox.com/docs/workspace/streaming/techniques
- https://create.roblox.com/docs/production/configs
- https://create.roblox.com/docs/production/experiments
- https://create.roblox.com/docs/projects/assets/packages
- https://devforum.roblox.com/t/4936628 (Package Overrides beta)
- https://create.roblox.com/docs/workspace/collisions
- https://create.roblox.com/docs/characters/character-controller-library
- https://devforum.roblox.com/t/4863739 (CCL abilities beta)
- https://create.roblox.com/docs/reference/engine/classes/StarterPlayer#CreateDefaultPlayerModule
- https://create.roblox.com/docs/projects/server-authority
- https://create.roblox.com/docs/reference/engine/classes/LogService
- https://devforum.roblox.com/t/4953062 (server log deep links)

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
