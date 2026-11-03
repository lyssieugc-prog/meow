# Test Automation

Frameworks, CI and scripted playtests. Belongs to the roblox-testing skill.

## Test Frameworks: Current State

Checked with the GitHub API on 2026-11-01. No framework below was run for this
skill except Lune.

| Tool | State | Use |
| --- | --- | --- |
| `Roblox/jest-roblox` | Not archived, last push 2026-10-02, Wally `roblox/jest@3.20.0` (README) | **Default for new suites.** Runs in Studio or Open Cloud Luau Execution (its docs) |
| `jsdotlua/jest-lua` | Not archived, last release and push 2024-12-23 (v3.10.0) | Community port. Prefer Roblox's own |
| `Roblox/testez` | **Archived**, last release v0.4.2 (2022-01-05) | Don't start new suites. Migrate when a suite needs changes |
| `lune-org/lune` | Active, v0.11.1 (2026-10-21) | Offline pure-logic runs |

```lua
--!strict
-- src/__tests__/LevelCurve.spec.luau (Jest Roblox looks under __tests__/)
-- Point Packages at wherever your Rojo project mounts the Wally folder.
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Packages = ReplicatedStorage:WaitForChild("Packages")
local JestGlobals = require(Packages:WaitForChild("JestGlobals"))
local LevelCurve = require(script.Parent.Parent.LevelCurve)

local describe = JestGlobals.describe
local it = JestGlobals.it
local expect = JestGlobals.expect

describe("levelFromTotalXp", function()
	it("starts at level 1 with 0 xp", function()
		expect(LevelCurve.levelFromTotalXp(0)).toBe(1)
	end)

	it("is consistent with xpForLevel at boundaries", function()
		for level = 2, 50 do
			local xp = LevelCurve.xpForLevel(level)
			expect(LevelCurve.levelFromTotalXp(xp)).toBe(level)
			expect(LevelCurve.levelFromTotalXp(xp - 1)).toBe(level - 1)
		end
	end)
end)
```

Jest's public docs mention an `EnableLoadModule` fast flag only for the internal
`roblox-cli` runner. Whether a Studio run needs anything similar is unverified;
read the error if it fails, and follow `docs/GettingStarted` in the repo for the
`runCLI` entry script.

### Lune: offline pure logic

Tested 2026-11-01 with Lune 0.11.1: the boundary loop above passed under
`lune run` with exit code 0.

- Require by path: `require("./LevelCurve")`. There is no `script`, and `game`
  is `nil`, so specs written with `script.Parent` won't load unmodified.
- Exit with `require("@lune/process").exit(1)` on failure so CI goes red.
- `@lune/roblox` gives you instances and attributes but not the engine:
  `Instance.new("Part")` works, while `part:GetTouchingParts()` errors as not a
  valid member. Use it for file and data work, not behavior.

## Headless Runs in CI: Open Cloud Luau Execution

`POST https://apis.roblox.com/cloud/v2/universes/{universe_id}/places/{place_id}/versions/{version_id}/luau-execution-session-tasks`
with `{"script": "..."}` runs a script on a specific place version, then you
poll the returned task and list its logs. Drop `versions/{version_id}` to use
the place's current version. Scope: `universe.place.luau-execution-session:write`.

- **Doesn't run:** physics, and the place's Server and Local scripts. Your task
  script must `require` what it tests. `IsServer()` is true and `IsRunning()` is
  false there (`RunService` reference). The docs say nothing about players; don't
  assume any exist.
- **Live data risk:** engine calls can read and modify real cloud data such as
  DataStores. Keep tests off them, or use a separate universe.
- DataModel changes are local to the task and aren't persisted.
- **Limits:** script and JSON return value 4 MB each; 5 minutes by default
  (`timeout` is immutable per task); 450 KB logs, oldest discarded; task info
  kept 24 hours; 10 incomplete tasks per place, then HTTP 429.
- **Rates per API key owner:** create 5/minute, get 200/minute, logs 45/minute.
  Poll with backoff; retry 429s.
- `return` a JSON-serializable result and fail the job on a task error, or on a
  failed count in the output. Logs only arrive after completion.

**Locally:** `RobloxStudio --task RunScript --localPlaceFile <place> --runScriptFile <abs path> --outputFile out.log --quitAfterExecution`
runs a script at command-bar permission after the place loads (`studio/cli`). On
Mac the binary is `/Applications/RobloxStudio.app/Contents/MacOS/RobloxStudio`.

## Scripted Playtests from a Plugin

All Studio-only; they do nothing in a published experience (`studio/testing-modes`).

- `StudioTestService:ExecuteMultiplayerTestAsync(numPlayers, args)` launches a
  server and 1–8 clients and yields until `EndTest(value)`, then returns `value`.
  Plugin scripts only, one session at a time, not from inside a test. Also
  `ExecutePlayModeAsync` and `ExecuteRunModeAsync`.
- In the test: `GetTestArgs()` (known issue from client LocalScripts; read it on
  the server), `AddPlayers(n)` and `EndTest(value)` from the server,
  `CanLeaveTest()` then `LeaveTest()` from a client for leave/rejoin flows.
- `UserInputService:CreateVirtualInput()` gives simulated mouse, keyboard and
  pointer input. It can return `nil`, so check; input that would touch CoreGui
  throws. `StudioDeviceSimulatorService` scripts device switching.
- `TestService` is still usable (`Check`, `Done`; `RunAsync` needs plugin security).
  **`FunctionalTest` is deprecated**, as are `TestService:Run` and
  `Is30FpsThrottleEnabled`.

Last verified: 2026-11-01 (Studio 0.745, release notes 745). Sources are listed in `../SKILL.md`.
