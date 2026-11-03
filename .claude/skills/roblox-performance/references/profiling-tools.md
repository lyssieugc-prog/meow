# Profiling Tools Reference

Read when capturing or interpreting performance data. Parent: `../SKILL.md`.

## Shortcuts

| Tool | Where | Shortcut |
| --- | --- | --- |
| Developer Console | Client, Studio, live | F9 (the server-dump steps say Ctrl/Cmd+F9) |
| MicroProfiler | Client, Studio, live | **Docs disagree:** `identify`, the walkthrough, `network` and `test-on-hardware` say Ctrl+Alt+F6 (Cmd+Opt+F6); the MicroProfiler index says Ctrl+F6 (Cmd+F6). Try the first, then the second. Not tested here. |
| Pause capture (opens detailed mode) | MicroProfiler | Ctrl/Cmd+P |
| Search labels | MicroProfiler | Ctrl/Cmd+F, F3 next, Shift+F3 previous |
| Performance Stats bar | Client, Studio | Ctrl+Alt+F7 (Cmd+Opt+F7), or settings menu |
| Debug stats | Live client | Shift+F1..F5: Shift+F2 render, Shift+F3 network, Shift+F5 summary |
| Debug stats | Studio | Ctrl+Shift+F1..F5 (Cmd+Shift); or **Window > Performance > Stats** |
| Scene Analysis | Studio, during play | **Window > Performance Summary > Scene Analysis** |

## MicroProfiler Workflow

Find a tall bar, pause, then read the timeline; wider means longer. The web UI (phone and
saved dumps) is preferred and adds X-Ray memory coloring, CPU and memory flame graphs,
and diff flame graphs from a second dump (Combine & Compare; don't combine different places).

- **Save:** Dump menu on desktop; file lands in `%LOCALAPPDATA%\Roblox\logs` or `~/Library/Logs/Roblox`.
- **Server:** on the desktop client, in a game you can edit: Dev Console > MicroProfiler >
  Server tab, up to 60 frames with up to 4 s delay. Only this shows server cost directly.
- **Label your code:** `debug.profilebegin("MobAI")` / `debug.profileend()`. A long-standing
  bug mislabeling these regions has a fix in 732 (**Pending**), so double-check surprising names.
- **Counters mode:** `Luau/heap` and `Luau/heap_core` split plugin and core-script memory
  apart from yours; per-VM execution time is shown (693).
- **Assistant and Studio MCP:** docs say a paused capture can be analyzed through a
  MicroProfiler API exposed by the Studio MCP server. Check whether your tool list has it
  before relying on it.

## Tag to Action

| Label | Means | Do |
| --- | --- | --- |
| `RunService.*`, `fireBindToRenderSteppedCallbacks` | Your frame callbacks | Less work, spread across frames. Labels keep names like `RunService.Stepped`. |
| `GC` | Luau garbage collection | Fewer temporaries, pool tables. |
| `WaitingHybridScriptJob` | Resumes `WaitForChild` and global `wait`, budgeted, usually 30 times per second | Fewer waiting scripts, shorter runs before yielding. |
| `stepHumanoid` | Humanoid state and movement | Fewer Humanoids, disable states, fewer `StateChanged` callbacks. |
| `stepAnimation` | Animators | Fewer Animators and joints, fewer animation event callbacks. |
| `physicsStepped`, `worldStep/stepContacts` | Physics | Fewer bodies colliding, simpler collision. |
| `assemble` | Assembly tree | Fewer joints created or destroyed. |
| `filterStep`, `Distributed Physics Ownership` | Ownership | Don't switch ownership often; keep groups apart. |
| `handleFallenParts` | Falling cleanup | Lower the destroy height or fall count. |
| `rasterizeTile`, `computePath` | Pathfinding | Stop non-navigable parts moving; reuse paths; shrink `ComputeAsync` extents. |
| `ProcessPackets`, `deserializePacket` | Incoming network | Fewer or smaller replicated things. |
| `Dispatch Data Senders` | Property changes, remotes, new instances | Fewer replicated changes. |
| `Dispatch Physics Senders` | Motion and touches | Fewer moving objects and touches. |
| `Dispatch StreamJob` | Streaming sends | Lower min and target radii. |
| `updateInvalidParts` | Changed parts | Spread bulk property updates across frames. |
| `updateInvalidatedFastClusters` | Humanoid or skinned geometry; over 4 ms is a warning | Fewer visual or scale changes; stop re-instantiating. |
| `updateInstancedClusters` | Bounding box changes | Fewer `CFrame`, `Size`, `Motor6D.Transform` updates. |
| `updateDynamicParts` | Beams, particles, Humanoids | Fewer visible. |
| `queryFrustumOrdered` | Frustum culling; high means many elements | Fewer pieces; consider a larger detailed mesh over many small ones. |
| `computeLightingPerform`, `LightGridCPU`, `ShadowMapSystem` | Lighting and shadows | Fewer lights, disable shadows, anchored geometry. |
| `Id_Transparent` | Partial transparency | Reduce. |
| `updateParticles` | Particles | Fewer emitters, lower rate and lifetime, less emitter motion. |
| `UpdateUILayouts`, `Rebuild Z-order list`, `fillGuiVertices` | UI | Fewer elements, stop reparenting, fewer `UIGradient` and `UICorner` on text. |
| `Present/waitUntilCompleted` | Waiting on the GPU | Reduce scene complexity; see the `Scene` tags. |
| `Sound/stepInstances` | Active sounds | Fewer playing. |

Full tag list: libmp `ALL-SCOPES.md` on GitHub, linked from the tag reference.

## Developer Console Tools

Tools documented today: **Log**, **Memory**, **LuauHeap**, **Network** (web calls, not
replication), **MicroProfiler**, **Script Profiler**; `identify` also uses **Server Jobs**
and **Server Stats**. The old "Script Performance" panel appears nowhere in the current
docs: use **Script Profiler** (1 or 10 kHz, Client or Server, records CPU-consuming calls only).

**Memory** groups: `CoreMemory`, `PlaceMemory`, `UntrackedMemory`, `PlaceScriptMemory`,
`CoreScriptMemory`. `PlaceMemory` and `PlaceScriptMemory` are the ones you control.

`PlaceMemory` categories: `HttpCache`, `Instances`, `Signals`, `LuaHeap`, `Script`,
`PhysicsCollision`, `PhysicsParts`, `GraphicsSolidModels`, `GraphicsMeshParts`,
`GraphicsParticles`, `GraphicsParts`, `GraphicsSpatialHash`, `GraphicsTerrain`,
`GraphicsTexture`, `GraphicsTextureCharacter`, `Sounds`, `StreamingSounds`, `TerrainVoxels`,
`TerrainPhysics`, `Gui`, `Animation`, `Navigation`. (The improve page says "InstanceCount"
for what this list calls `Instances`.) Assets are usually `GraphicsMeshParts`,
`GraphicsTexture`, `Sounds`.

**LuauHeap** snapshots: Graph (Size vs Self, from `registry`), Object Tags, Memory Categories,
Object Classes, Unique References, Unparented Instances. Compare two snapshots over time. It
covers the Luau VM, not engine memory.

Use client numbers, not Studio's, for accurate readings.

## Scene Analysis

GA: DevForum "[Full Release] Scene Analysis Tool" (topic 4620678) states it left beta on
2026-07-10, though release notes 718/723/729 still list items as `Pending`. Studio, during
play or test, client and server. Views: Script memory, Unparented instances, Instance
composition, Audio memory, Animation memory, Triangle composition. Triangle counts may not
match players' devices because of scaling.

`SceneAnalysisService` (Yields; also exposed through the Studio MCP server) returns
`{Name, Size or Sizes, Children}` trees: `GetInstanceCompositionAsync`, `GetScriptMemoryAsync`,
`GetUnparentedInstancesAsync`, `GetTriangleCompositionAsync`, `GetAnimationMemoryAsync`,
`GetAudioMemoryAsync`. Unparented entries with a parent of `Unknown` are pending GC, not a leak.

Collision Geometry Memory API (745, **Pending, undocumented**): absent from the reference
YAML and the live page. The 0.745 Mac binary contains the strings
`GetCollisionGeometryMemoryAsync` and `SceneAnalysisCollisionGeometryMemoryEnabled`, an
indirect sign of the name and of a flag gate, not a contract. Don't call it from shipped code.

```lua
--!strict
-- Run during a Studio playtest. Not executed while writing this skill.
local SceneAnalysisService = game:GetService("SceneAnalysisService") :: any

local function collectLeaves(node: any, out: { any })
	local children = node.Children
	if children == nil then
		table.insert(out, node)
		return
	end
	for _, child in children do
		collectLeaves(child, out)
	end
end

local leaves: { any } = {}
collectLeaves(SceneAnalysisService:GetScriptMemoryAsync(), leaves)
table.sort(leaves, function(a, b)
	return a.Size > b.Size
end)
for index = 1, math.min(10, #leaves) do
	print(leaves[index].Name, math.round(leaves[index].Size / 1024), "KB")
end
```

## Stats Service (runtime telemetry)

Readable from scripts: `FrameTime` (client only), `HeartbeatTime`, `PhysicsStepTime`,
`RenderCPUFrameTime`, `RenderGPUFrameTime`, all **in seconds**; `InstanceCount`,
`PrimitivesCount`, `MovingPrimitivesCount`, `ContactsCount`, `SceneDrawcallCount`,
`SceneTriangleCount`, `ShadowsDrawcallCount`, `UI2DDrawcallCount`, data and physics Kbps.

- **Deprecated:** `HeartbeatTimeMs`, `PhysicsStepTimeMs`.
- `GetTotalMemoryUsageMb()` comes from the OS, so it differs from the sum of tags.
- `GetMemoryUsageMbForTag(Enum.DeveloperMemoryTag.X)` returns 0 and warns if
  `MemoryTrackingEnabled` is false. Tags include `LuaHeap`, `Instances`, `GraphicsTexture`,
  `GraphicsSlimModels` (SLIM, 699), `GeometryCSG`. `BaseParts` is the tag name.
- `RenderBreakdown` category in the `Stats` tree (714, Live): triangle and draw call
  breakdowns. No path or key is documented, and `StatsItem` values are described as plugin
  readable. For a documented route use `GetTriangleCompositionAsync` or `SceneDrawcallCount`.
- Structured logging `LogService:Warn(template, context)`, `Info`, `Error`, `Output`, `Log`,
  `GetLogger`: in the reference; **Pending** per 716 and 727. Use `warn` unless you've
  confirmed it runs in your place.

```lua
--!strict
-- Server: log when a documented budget is exceeded.
local Players = game:GetService("Players")
local Stats = game:GetService("Stats")

local FRAME_BUDGET_SECONDS = 1 / 60
local peakPlayers = 0

local function memoryBudgetMb(): number
	-- 50% of total server memory: 6.25 GiB + 100 MiB per peak player
	return (6.25 * 1024 + 100 * peakPlayers) * 0.5
end

task.spawn(function()
	while true do
		task.wait(30)
		peakPlayers = math.max(peakPlayers, #Players:GetPlayers())
		local heartbeat = Stats.HeartbeatTime
		local memory = Stats:GetTotalMemoryUsageMb()
		if heartbeat > FRAME_BUDGET_SECONDS or memory > memoryBudgetMb() then
			warn(string.format("perf: heartbeat %.1f ms, memory %.0f MB", heartbeat * 1000, memory))
		end
	end
end)
```

## Performance Dashboard

Creator Hub > game > Monitoring > Performance; charts need at least 100 DAU. Filter by
average or percentile, break down by **Place Version**. Client: frame rate, crash rate, crash
count, unexpected OOM exits, memory, memory percentage, CPU time. Server: CPU time, frame rate,
memory, memory by age, cores used, compute efficiency. CPU categories map to MicroProfiler
groups; memory tags map to `Enum.DeveloperMemoryTag`. **Roblox update** annotations help
separate your regression from a platform one. A dashed crash line means noisy data; check the count.

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
