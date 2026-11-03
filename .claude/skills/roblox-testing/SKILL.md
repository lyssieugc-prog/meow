---
description: Testing and QA for Roblox games — agent evidence loop, gated cheat remotes, negative tests with hostile remote payloads, pure-logic modules, Jest Roblox / TestEZ status, Lune offline runs, Open Cloud Luau Execution in CI, StudioTestService, Server & Clients, Device and Network Simulator, Team Test server inspection, real-hardware checks, early testers and experiments. Use when setting up tests or CI, verifying a gameplay system, debugging state-dependent bugs, reporting a failure, or reviewing QA tooling.
---

# Roblox Testing & QA

Roblox games are stateful, networked and hard to test from the outside. Make
logic pure where possible, build **gated cheat tooling** to reach every other
state in seconds, and attack your own remotes. A claim that something works
needs evidence from a run of the current build.

## Which Layer Catches What

| Layer | Runs where | Catches | Blind to |
| --- | --- | --- | --- |
| Plain Luau under Lune | Offline, no Roblox | Formula, table and migration logic | Everything in the engine |
| Jest Roblox in Studio | Inside Roblox | Module logic using engine types | Real clients, latency |
| Open Cloud Luau Execution | Headless on a place version, in CI | Module and DataModel logic on that exact version | Physics, gameplay scripts (neither runs) |
| Solo **Test** (F5) | Studio client + server | Client/server split, UI flow | Other players |
| **Server & Clients** (F7) | Studio, up to 8 clients | Replication, join order, late joiners | Real networks, device limits |
| Network / Device Simulator | Studio playtests | Latency, loss, screen size, touch | Device performance |
| Team Test | Real Roblox server | Real server, geography, real collaborators | Devices you don't use |
| Real hardware | Phone, console | Memory, frame rate, thermal, touch feel | Nothing here substitutes for it |

## Agent Evidence Loop

When you can drive Studio, follow the **roblox-studio-workflow** skill for the
tools. Every gameplay check has the same shape:

1. Start a playtest of the current code; confirm it is the build you changed.
2. Reach the state through cheat remotes, not by grinding.
3. Perform the action.
4. Read the console for **both** sides. Output labels client messages blue and
   server messages green; a clean client log says nothing about the server.
5. Capture the screen when the result is visual.
6. Stop the playtest and report: action, expected, observed, the log line or
   capture.

**Re-run before reporting a failure.** An old result is not evidence. Before
saying "broken", "no access" or "blocked", repeat the check now against the
current build. Only report what that run shows, and include how to repeat it.
The same rule applies to success: never write "tested" for a check you did not
run, and name what was left unverified. Compiling, analyzing and reading code
are not a playtest.

## The Command-Bar Trap

**Field experience, not documented behavior.** `require()` from the Studio
command bar has been seen to return a module instance separate from the one the
running game uses, so changing its tables leaves live state alone: "my test
worked but the game didn't change". The docs only state that a module returns
one reference per client/server side (`scripting/module`) and that the command
bar runs at a higher permission level than scripts. They don't describe this
case, so if the result matters, prove it with a print from the game's own code.

Instances are different: the docs themselves change live state from the Server
command bar during a playtest (`workspace.Player1.Humanoid.Health = 0`). Check
which side you are on. For state held in module tables, use cheat remotes.

## Gated Cheat Remotes

```lua
--!strict
-- ServerScriptService/CheatCommands: QA remotes. Only game systems behind them.
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")

local DEV_USER_IDS: { [number]: true } = {
	[12345678] = true, -- your UserId
}
local STAGING_PLACE_IDS: { [number]: true } = {} -- places where live QA is allowed

local CheatCommands = {}

local enabled = RunService:IsStudio() or STAGING_PLACE_IDS[game.PlaceId] == true
local folder: Folder? = nil

local function isAuthorized(player: Player): boolean
	return RunService:IsStudio() or DEV_USER_IDS[player.UserId] == true
end

function CheatCommands.register(name: string, handler: (player: Player, ...any) -> ())
	if not enabled then
		return -- Production: the remote never exists, so it can't be attacked.
	end
	local parent = folder
	if parent == nil then
		parent = Instance.new("Folder")
		assert(parent)
		parent.Name = "QA"
		parent.Parent = ReplicatedStorage
		folder = parent
	end
	local remote = Instance.new("RemoteEvent")
	remote.Name = name
	remote.OnServerEvent:Connect(function(player: Player, ...: any)
		if isAuthorized(player) then
			handler(player, ...)
		end
	end)
	remote.Parent = parent
end

return CheatCommands
```

- **The gate lives on the server** in the shared wrapper. Hiding the panel does
  nothing, because any client can fire any remote it can see (see
  roblox-security).
- `IsStudio()` is `true` in Team Test too (`RunService` reference), and `false`
  on live servers. Anyone in Team Test is a collaborator, so that is fine, but
  don't read it as "local machine only".
- Handlers call the **same system APIs** as gameplay
  (`Economy.addCoins(player, n)`), never raw data, or cheats test a path players
  never run. Cheat arguments still get type checks.
- Cover: grant/remove currency and items, set progression, trigger rare events
  (pity thresholds, level-ups), spawn/clear mobs, god mode, and **reset profile
  to default**.
- Live QA only on places you listed, and only for allowlisted UserIds. Never
  point cheats at production data stores.

## Negative Tests: Attack Your Own Remotes

A happy-path test proves nothing about security. For each client-to-server
remote, fire hostile payloads from a client during a Studio playtest and assert
that server state did not change and the server console has no handler error.

```lua
--!strict
-- Client-side QA harness, Studio only. Snapshot comes from a gated QA remote.
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")

assert(RunService:IsStudio(), "QA harness must not ship")

local function nested(depth: number): { any }
	local root: { any } = {}
	local cursor = root
	for _ = 1, depth do
		local inner: { any } = {}
		cursor[1] = inner
		cursor = inner
	end
	return root
end

local HOSTILE: { { name: string, args: { any }, count: number } } = {
	{ name = "no args", args = {}, count = 0 },
	{ name = "nil", args = { nil }, count = 1 },
	{ name = "NaN", args = { 0 / 0 }, count = 1 },
	{ name = "inf", args = { math.huge }, count = 1 },
	{ name = "-inf", args = { -math.huge }, count = 1 },
	{ name = "negative", args = { -1 }, count = 1 },
	{ name = "fractional", args = { 0.5 }, count = 1 },
	{ name = "huge", args = { 2 ^ 53 }, count = 1 },
	{ name = "wrong type", args = { "1" }, count = 1 },
	{ name = "long string", args = { string.rep("x", 100000) }, count = 1 },
	{ name = "deep table", args = { nested(200) }, count = 1 },
	{ name = "mixed keys", args = { { 1, a = 2, [3.5] = 4 } }, count = 1 },
	{ name = "instance", args = { workspace }, count = 1 },
}

local qa = ReplicatedStorage:WaitForChild("QA")
local snapshotRemote = qa:WaitForChild("Snapshot") :: RemoteFunction

local function attack(remote: RemoteEvent): { string }
	local failures: { string } = {}
	for _, case in HOSTILE do
		local before = snapshotRemote:InvokeServer()
		remote:FireServer(table.unpack(case.args, 1, case.count))
		-- Remote events and functions have no guaranteed relative order.
		task.wait(0.5)
		if snapshotRemote:InvokeServer() ~= before then
			table.insert(failures, `{remote.Name}: state changed on "{case.name}"`)
		end
	end
	return failures
end

return attack
```

- The snapshot is a server-built string of what the remote could affect.
- Add cases per remote: a target instance owned by another player, an ID the
  player doesn't own, an action out of range, on cooldown or after death.
- Fire bursts as well. Client remotes have a shared limit of about 500 requests
  per second per client, and excess `RemoteEvent` messages are processed later
  rather than dropped (`RemoteEvent` reference, `scripting/events/remote`).
- Non-string table indices are converted to strings in transit, so a handler
  that expects a numeric key may see another shape than the client sent.

## Designing for Testability: Pure Core, Thin Shell

Logic that touches no Roblox APIs can be tested anywhere, including CI:

```lua
--!strict
-- shared/Util/LevelCurve.luau  (pure: numbers in, numbers out)
local LevelCurve = {}

function LevelCurve.xpForLevel(level: number): number
	return math.floor(100 * level ^ 1.5)
end

function LevelCurve.levelFromTotalXp(xp: number): number
	local level = 1
	while xp >= LevelCurve.xpForLevel(level + 1) do
		level += 1
	end
	return level
end

return LevelCurve
```

Put damage formulas, drop tables, economy math, level curves and validation
predicates in pure modules. Test first where silent bugs cost most: money math,
data migrations against fixture profiles (see roblox-datastores), drop/pity
tables, validation predicates. Property-style loops over boundaries catch far
more than single values.

**Determinism:** pure modules take randomness as an injected function
(`rng: () -> number`). Tests pass a generator from `Random.new(seed)`.

## Automation

| Need | Use | Status |
| --- | --- | --- |
| New spec suite | `Roblox/jest-roblox`, specs in `__tests__/*.spec.luau` | Maintained by Roblox |
| Existing TestEZ | `Roblox/testez` | **Archived.** Don't start new suites |
| Offline pure logic | Lune | Tested here |
| CI on a place version | Open Cloud Luau Execution | Stable. No physics, no game scripts |
| Multi-client from code | `StudioTestService` (plugin) | In reference |

Read [references/automation.md](references/automation.md) when writing specs, a CI
job or a scripted playtest.

## Playtesting Multiplayer Behavior

**Server & Clients** in the test mode dropdown of the mezzanine (F7), up to 8
clients, usually 1–2 are enough. It is the cheapest way to catch state you forgot
to replicate, remotes assuming one player, and UI showing player A's data to player
B.

- Test **join order** both ways. Every system needs a "send full snapshot on
  join" path, not only incremental updates.
- Leave and rejoin mid-interaction (trade, dungeon, shop open): cleanup and
  session locks (see roblox-datastores) only break in these windows.
- With StreamingEnabled, walk a client far away and back. Does the system survive
  its instances streaming out?
- Solo Test also has Pause/Resume and Step Forward (1/60 s). They pause physics,
  animation and some `RunService` signals, not running scripts.
- Party features: **Party Simulator**. Localization and policy: **Player Emulator**.

### Other environments

Read [references/playtest-environments.md](references/playtest-environments.md) when
testing under bad networks, on devices, in Team Test (Server Inspection is a Studio
Beta) or with early testers and experiments. Essentials: Network Simulator values add
to the real connection; Device Simulator does **not** simulate device performance; a
real low-end device is required before a performance sign-off.

## Bug Reproduction Discipline

1. Reach the exact state with cheat remotes.
2. Add a temporary log at the suspect boundary on the correct side, reproduce,
   read it, remove it. For timing, wrap the suspect in
   `debug.profilebegin`/`debug.profileend` and read the MicroProfiler.
3. Fix, re-run the same steps, then add the cheapest permanent guard: an assertion,
   a validation check, a spec or a new hostile payload.

## Review Red Flags

| Red flag | Problem |
| --- | --- |
| "Fails" based on a result from earlier in the session | Stale evidence. Re-run on the current build before reporting. |
| "Tested" with only analyzer output or code reading | Not a test. Say what was and wasn't run. |
| Checked only the client console | Server errors are invisible there. Read both. |
| QA steps that change module state from the command bar | May touch a separate module instance (field experience). Use cheat remotes. |
| Cheats gated only by hiding UI | Any client can fire them. Gate on the server, don't create them in production. |
| Cheat handlers editing raw data | Skips the real path. Route through system APIs. |
| Remote tested only with valid arguments | No evidence it rejects hostile input. Add negative tests. |
| New suite on TestEZ or `FunctionalTest` | Archived repo; deprecated class. Use Jest Roblox and `TestService` where needed. |
| CI test expects physics or game scripts under Luau Execution | Neither runs there. Require modules, or use a playtest. |
| CI touching DataStores in a production universe | The API reaches real data. Use a test universe. |
| Performance sign-off from Device Simulator | It doesn't simulate device performance. Use real hardware. |
| Multiplayer feature never run under Server & Clients | Late-joiner and replication bugs go unseen. |
| `math.random` inside formulas under test | Inject the rng. |

When you (Claude) implement a gameplay system, also produce: (a) its cheat entries,
(b) specs for its pure logic, and (c) hostile payload cases for each new remote.
Then run what you can and report only what that run showed. A system without a fast
way to reach its states won't get tested.

## Sources

- https://create.roblox.com/docs/studio/testing-modes
- https://create.roblox.com/docs/studio/network-simulator
- https://create.roblox.com/docs/studio/device-simulator
- https://create.roblox.com/docs/studio/cli
- https://create.roblox.com/docs/studio/overview
- https://create.roblox.com/docs/studio/debugging
- https://create.roblox.com/docs/cloud/reference/features/luau-execution
- https://create.roblox.com/docs/performance-optimization/test-on-hardware
- https://create.roblox.com/docs/production/experiments
- https://create.roblox.com/docs/scripting/events/remote
- https://create.roblox.com/docs/scripting/module
- https://create.roblox.com/docs/reference/engine/classes/StudioTestService
- https://create.roblox.com/docs/reference/engine/classes/TestService
- https://create.roblox.com/docs/reference/engine/classes/FunctionalTest
- https://create.roblox.com/docs/reference/engine/classes/VirtualInput
- https://create.roblox.com/docs/reference/engine/classes/RunService
- https://devforum.roblox.com/t/4861695 (Device and Network Simulator full release)
- https://devforum.roblox.com/t/4966700 (Server Inspection beta)
- https://devforum.roblox.com/t/4636610 (Remote Breakpoints beta)
- https://devforum.roblox.com/t/4930524 (Playtest audience, early testers)
- https://github.com/Roblox/jest-roblox, https://github.com/Roblox/testez, https://github.com/jsdotlua/jest-lua, https://github.com/lune-org/lune

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
