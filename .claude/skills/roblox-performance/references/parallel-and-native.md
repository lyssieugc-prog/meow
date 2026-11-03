# Parallel Luau and Native Code

Read when moving computation to Actors or adding `--!native`. Parent: `../SKILL.md`.

Use when the Dashboard shows **low cores used with high server CPU**, or profiles show
Luau computation that doesn't need to write the DataModel.

```lua
--!strict
-- Must live under an Actor. Same-Actor scripts run serially with each other.
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")

local params = RaycastParams.new()
local origin = Vector3.new(0, 50, 0)
local direction = Vector3.new(0, -100, 0)

RunService.Heartbeat:ConnectParallel(function()
	local result = Workspace:Raycast(origin, direction, params) -- thread safe
	if result then
		task.synchronize() -- writes must be serial
		result.Instance:SetAttribute("LastHitAt", os.clock())
	end
end)
```

- A member with no listed thread safety is **Unsafe**. Check the reference.
- No `require()` in a desynchronized phase; require in serial first.
- More Actors than cores balances better (docs' example: 64+ for raycast validation on
  4 cores), split by logic unit. Don't nest Actors without a reason.
- A long computation still blocks in parallel. Slice it as well.
- Cross-Actor: `Actor:SendMessage` + `BindToMessageParallel`, or `SharedTable` for large
  shared state.

**Native:** `--!native` for a script or `@native` per function. The docs scope this to
**server-side** scripts with heavy numeric work on tables and `buffer`; it's not a
default. Costs: compile time at server start, memory, and a total size limit past which
code silently runs non-natively. Check with `debug.dumpcodesize()` in the Command Bar
under Server view. Annotate: `v: Vector3` gets specialized code; since 701 typed
`Vector2`, `CFrame`, `RaycastResult`, `UDim`, `Color3`, `NumberRange` locals get up to
20x faster property access. `getfenv`/`setfenv`, wrongly typed arguments and breakpoints
defeat it. Confirm with `<native>` in the Script Profiler, and measure both ways.

## Recent Language Speedups

Language speedups are free; don't restructure code for them: dynamic key table access
10-25% faster (739); frozen metatables much cheaper metamethod lookup via `table.freeze`
(739); pre-allocated `table.create(N)` no longer shrinks on a non-numeric insert (739).
Pending: metamethod dispatch 5-25% (739), `table.find` 2-3x (743), `string.split` (742).
`Instance` shrank by 8 bytes (699); nothing to act on.

## Sources

- https://create.roblox.com/docs/scripting/multithreading, /luau/native-code-gen, /production/analytics/performance

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
