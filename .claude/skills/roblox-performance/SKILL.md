---
description: Roblox performance and memory - decision procedure and documented targets (frame time, server heartbeat, memory, join time), MicroProfiler/Developer Console/Scene Analysis, testing on low-end hardware, connection and table leaks, RunService event choice, Parallel Luau, native codegen, networking and remote limits, physics, Humanoids, rendering, assets, StreamingEnabled and SLIM. Use when diagnosing lag, stutter, crashes, memory growth or slow joins, when writing RunService loops, spawners, VFX or streaming-aware client code, or when reviewing Roblox code for performance.
---

# Roblox Performance and Memory

Optimizing without a measurement usually moves work around and fixes nothing.
Find out which of frame rate, memory, load time or network is failing, on which
side (client or server), on which device, then fix that. Say "fixed" only with a
before and after number from the same place on the same device.

Status labels: **GA** generally available; **Beta** Studio beta; **Pending**
shipped in the binary but not enabled per release notes. Unlabeled text is a
documented, generally available behavior. **(judgment)** marks engineering advice
the docs do not state.

## The Procedure

1. **Reproduce on a real low-end device, not in Studio.** Studio runs client and
   server together on a fast machine, which skews stats and memory. A capped
   gaming PC hides a 4 ms to 16 ms regression. Pick at least one baseline device
   and keep testing on it. Device Simulator does **not** model memory or rendering
   performance (see Testing below).
2. **Classify** with the table below.
3. **Drill down** with the tool for that row; find the worst *child* label, since a
   parent can't be shorter than the sum of its children.
4. **Fix one cause** using the section for that category. Multiple simultaneous
   changes make the result unreadable. (judgment)
5. **Measure again the same way.** Dump-to-dump comparison: drop a second dump into
   the MicroProfiler web UI for a diff flame graph. Live: Performance Dashboard
   **Breakdown by > Place Version**.
6. **Soak.** 10-15 minutes of active play on mobile to expose thermal throttling (a
   steady FPS decline); a long session while watching memory to expose leaks. A leak
   can look fine in a short Studio test and still crash devices in long sessions.

| Symptom | Where to look | Documented target or signal |
| --- | --- | --- |
| Client stutter, low FPS | Shift+F5 summary on the client; MicroProfiler | 16.67 ms per frame at the default 60 FPS cap (Windows users can raise it to 240). Consistency matters: a 410 ms frame among 10 ms frames is a visible hitch at "60 FPS". |
| Lag for everyone, rising ping | Dev Console **Server Jobs > Heartbeat > Steps Per Sec**; server MicroProfiler dump | Heartbeat capped at 60; server CPU under 16 ms per frame. |
| Server crashes or memory growth | Dashboard **Server memory**, **Server memory by age** | Under 50% of `6.25 GiB + 100 MiB * peak_players` (never shrinks when players leave); dashboard guideline: under 3 GB. |
| Client crashes | Dashboard **crash rate**, **Unexpected out-of-memory exits**, **Client memory usage percentage** | Investigate above 2-3% crash rate, especially a jump after an update; a line near 100% memory means imminent OOM on that platform. |
| Slow join | Stopwatch; Studio Settings > Network > **Print Join Size Breakdown** | No built-in tool or numeric target. Check the three stages: ReplicatedFirst, rest of the DataModel, avatar. |
| Input feels late | Compare network ping (Performance Stats) with data ping (Dev Console **Server Stats**, Shift+F3) | Data ping well above network ping means replication queues; open the MicroProfiler network view. |
| Too many draw calls or triangles | Shift+F2; Scene Analysis **Triangle composition** | No global limit. Derive yours from the baseline device. The docs' "1,000 draw calls, 1,000,000 triangles" is an example for one hypothetical device, not a target. |

Reading MicroProfiler frame bars: **orange** = worker jobs (scripts, physics,
animation) exceeded render; **blue** = render thread exceeded jobs (object density,
movement, lighting); **red** = blue plus GPU wait over 2.5 ms (object complexity,
texture size, visual effects). Colors only guide you when you are missing the time
target; no color is "good". Set graphics quality to maximum in the client settings
when measuring so the frame rate manager doesn't hide the cost.

Tools, shortcuts, labels and memory categories in full:
read [references/profiling-tools.md](references/profiling-tools.md) when you are
actually capturing or interpreting data.

## Testing on Hardware

- Most players are on lower-spec mobile: Android is roughly 65%, about 60% of those on
  2-4 GB RAM. Docs' example Android set: Infinix Smart 9, Moto G05, Oppo A18, Fire HD 10
  (2023), Galaxy S22 Ultra.
- MicroProfiler on phone: Settings > MicroProfiler **On**, then open the shown `IP:port` from a
  browser on the same network (append `/90` for 90 frames).
- **Device Simulator** (Test menu): screen, scaling and input only. Roblox states it does not
  simulate memory or rendering.
- **Network Simulator** (Network button on that toolbar): latency, jitter, packet loss per
  direction, Studio playtests only; confirm on a real device over cellular. Status conflict:
  DevForum (2026-10-16) says the new Device Simulator left beta; the 2026-10-29 docs still say
  enable **New Device Simulator** in Beta Features. If the toolbar is missing, enable it there.

## Connection and Table Leaks

The engine never collects a connected callback or anything it captures while the
connection is alive. A connection dies by `Disconnect()`, by destroying the instance
the event belongs to, or by destroying the script it traces back to.

```lua
--!strict
local RunService = game:GetService("RunService")

local function watchHealth(mob: Model, updateHealthbar: (Model) -> ())
	-- LEAK: RunService is never destroyed, so this runs and pins `mob` forever
	RunService.PostSimulation:Connect(function()
		updateHealthbar(mob)
	end)
end
```

Every object that makes connections owns a cleanup list, run exactly once:

```lua
--!strict
type Cleanup = () -> ()

local function makeMaid()
	local tasks: { Cleanup } = {}
	return {
		add = function(item: RBXScriptConnection | Cleanup | Instance)
			table.insert(tasks, function()
				if typeof(item) == "RBXScriptConnection" then
					item:Disconnect()
				elseif typeof(item) == "Instance" then
					item:Destroy()
				else
					item()
				end
			end)
		end,
		clean = function()
			for _, cleanup in tasks do
				cleanup()
			end
			table.clear(tasks)
		end,
	}
end
```

(Trove, Janitor and Maid do the same. The discipline matters, not the library.)

Checklist for every system:

- **Players are the classic trap.** Unless told to, the engine does not destroy the
  `Player` or the character when someone leaves, so connections on them survive and
  leak per join on long-lived servers. Either set
  `Workspace.PlayerCharacterDestroyBehavior = Enabled` (Studio property, not
  scriptable; `Default` follows the engine default) or destroy them yourself: `task.defer(player.Destroy, player)` in
  `PlayerRemoving`, `task.defer(character.Destroy, character)` in `CharacterRemoving`.
  Pending: 731 also destroys the character on the client, 736 delays `Player`
  destruction by some seconds, so don't depend on exact timing.
- `PlayerRemoving` clears every table keyed by that Player.
- Long-lived tables keyed by Instances: key by the instance and **guarantee removal**
  on the path that ends its life (`Destroying`, `PlayerRemoving`, a tag's removed
  signal). Don't reach for `Instance:GetDebugId()`: it is `PluginSecurity`, so game
  scripts can't call it. Don't use a weak table to track whether an instance still
  exists: Luau can collect the reference while the instance is alive.
- `Instance.Destroying` or your maid tears down per-instance loops. **On the client,
  stream-out is not destruction**: the instance is parented to `nil` and may return,
  so `Destroying` does not mark its departure. Use tag added/removed signals.
- A `task.delay` callback never runs if its script is disabled or destroyed first. For
  guaranteed cleanup of an instance use `Debris:AddItem(item, seconds)`.
- Animations are a common production leak: scripts keep them alive after the model is
  gone. Check Scene Analysis **Animation memory** and **Unparented instances**.

Finding a leak: Dev Console **Memory**, watch `PlaceMemory > LuaHeap` and `Instances`
for sustained growth, `PlaceScriptMemory` for the script; then **LuauHeap** snapshots
over time (Graph, Unique References, Unparented Instances show the pinning path). Name
regions with `debug.setmemorycategory("MobAI")`.

## Choosing a RunService Event

`RenderStepped` and `Stepped` are **superseded**, not removed: the reference carries
a migration note, but they have no `Deprecated` tag and aren't in the deprecated list.
Use the replacements in new code and don't churn old code just for this. `Heartbeat`
is **not** superseded.

| Use | For | Notes |
| --- | --- | --- |
| `PreRender` (was `RenderStepped`) | Client only. Work that must follow input and precede the frame, like camera | Blocks rendering until it finishes. Use `BindToRenderStep` when order matters. |
| `PreAnimation` | Adjusting animation objects | Before Animators step. |
| `PreSimulation` (was `Stepped`) | Writing physics state, `Motor6D.Transform` | Takes only `deltaTimeSim`, while `Stepped` took `(time, deltaTime)`. |
| `PostSimulation` | Reading physics results | `deltaTimeSim` excludes throttling; may differ from real frame time at low FPS. |
| `Heartbeat` | Periodic game logic | End of frame; real elapsed `deltaTime`. Waiting `task` threads resume here. |
| `BindToSimulation`, `BindToAnimation` | Fixed-rate simulation and prediction | Needs `Workspace.UseFixedSimulation`; errors on unsynchronized members. `Hz60`-`Hz1`, default `Hz30`. |
| `BindToSimulationParallel`, `BindToAnimationParallel` | The same, under an Actor | **Pending (743).** In the reference, but don't ship on it without checking it's enabled and saying so. |

## Frame Budget Discipline

- **Prefer events to per-frame code.** Use a frame event only when high-frequency
  invocation is essential. The MicroProfiler walkthrough's own bug is 1,500 raycasts
  per `Stepped`.
- **Lower cadence is fine and documented.** `task.wait(0.1)` in a loop is legitimate:
  it resumes on the first Heartbeat when due, doesn't throttle, and returns the actual
  elapsed time. Bare `task.wait()` in a loop *is* per-frame work. The global `wait()`
  is deprecated.
- **A low-frequency spike is still a spike.** 100 ms once per second means 59 smooth
  frames and one visible hitch. Slice it:

```lua
--!strict
-- Docs' example is 5 ms of work per frame; pick the budget from your own profile.
local SLICE_SECONDS = 0.005

local function processInSlices<T>(items: { T }, handle: (T) -> ())
	local sliceStart = os.clock()
	for _, item in items do
		handle(item)
		if os.clock() - sliceStart >= SLICE_SECONDS then
			task.wait()
			sliceStart = os.clock()
		end
	end
end

return processInSlices
```

- **Reduce temporary allocation in hot paths.** A long `GC` label means pool tables or
  create fewer temporaries. Hoist closures out of loops, reuse buffers. (judgment on
  specifics; the GC advice is from the tag reference)
- Call a method once and store the value; don't re-run `FindFirstChild` chains per
  frame. `GetChildren`/`GetDescendants` allocate an array per call, despite being
  faster since 699/701. `QueryDescendants` selectors are GA (697).
- Tween on the client with `TweenService` instead of per-frame easing in Luau. Tweens
  cost too: they appear as `TweenService` under `PreRender` (client) or `Heartbeat`
  (server), so limit the count and keep completion callbacks small.
- Many entities: a single manager over `CollectionService` tags rather than one script
  per entity (judgment; see roblox-architecture). Measure with Scene Analysis **Script memory**.

## Parallel Luau and Native Code

Use Actors when the Dashboard shows **low cores used with high server CPU**, or profiles
show Luau computation that doesn't need to write the DataModel: `ConnectParallel` or
`task.desynchronize`, `task.synchronize` before writes. Same-Actor scripts run serially, no
`require()` while desynchronized, unlisted thread safety means **Unsafe**, and long work still
blocks. `--!native`/`@native` is documented for **server-side** numeric work, has startup,
memory and size-limit costs, and must be measured, never blanket-applied. Details and a sample:
read [references/parallel-and-native.md](references/parallel-and-native.md) before using either;
it also lists recent Luau speedups (739, 743), which need no code changes.

## Network Traffic

- Never fire remotes per frame or per input event without a throttle. Send on change or
  on a fixed low cadence such as 10 Hz (judgment on the number). Send what changed, not
  the whole inventory.
- **Client-to-server rate limit:** about 500 requests per second per client, shared
  across all remotes of the same type. Excess `RemoteEvent`s are processed late, in
  order; excess `UnreliableRemoteEvent`s are **dropped**.
- **`UnreliableRemoteEvent`:** payload over **1,000 bytes is dropped** (Studio logs by
  how much). Compression makes the size hard to pre-check, so stay well below it or use
  `RemoteEvent`. No order guarantee: include a sequence number and drop stale messages.
  No listener connected means immediate discard, not a queue. Use for ephemeral or
  continuously changing data only.
- **Don't tween world objects on the server.** The property replicates to each client
  every frame, adding traffic and latency jitter. Send one event; tween on clients.
- Visual effects, first-person view models: create on the client. The server only needs
  the location to decide the outcome.
- Creating or destroying large trees replicates: chunk it across frames. Rigs often
  carry Animation Editor metadata; remove it before cloning them regularly.
- Network view: **only in dumps from desktop client or Studio**. Save a dump, open it,
  **X-Ray > Network**, right-click a bar for events; physics blue, data green, assets red.

## Physics, Humanoids, Rendering, Assets, Load Time

Quick rules; the catalog has the specifics, limits and statuses. Read
[references/fix-catalog.md](references/fix-catalog.md) once the profile names one of these.

- **Physics:** anchor what doesn't simulate; `Box`/`Hull` fidelity, since collision geometry
  stays in memory even with collisions off. `SetNetworkOwner(nil)` is server-side, for
  gameplay-critical parts, set once and used conservatively because it can add jitter.
- **Humanoids:** disable unused states, `AnimationController` on static NPCs, pool respawns,
  animate NPCs on the client, update `Motor6D.Transform` rather than `C0`/`C1`.
- **Temporary objects:** `Debris:AddItem` is current, not deprecated; hard cap of 1,000 items,
  oldest destroyed early, so `lifetime` is a maximum.
- **Rendering:** upload each mesh once and duplicate in Studio so instancing collapses draw
  calls (the importer doesn't deduplicate). Limit shadow casters; keep transparency at 0 or 1.
- **Assets:** texture memory is pixels, and compression doesn't help. 512 max in general, under
  256 for minor images.
- **Load:** lean `ReplicatedFirst` and `ReplicatedStorage`, targeted `PreloadAsync`, streaming on.
- **Last resort:** Extended Services for Compute is paid and most games see no benefit; that's the owner's decision, after optimizing.

## StreamingEnabled

On by default for **new** places, can't be set by script, so check your place rather than
assuming. Recommended Workspace values (Studio only): `ModelStreamingBehavior = Improved`,
`StreamingIntegrityMode = PauseOutsideLoadedArea`, `StreamingMinRadius = 64`,
`StreamingTargetRadius = 1024`, `StreamOutBehavior = Opportunistic`.

- Client code can't assume a Workspace descendant exists: `.` errors, `FindFirstChild`
  returns `nil`, `WaitForChild` can hang forever (pass a timeout), and `GetChildren`
  returns only the streamed-in subset. Server code sees everything.
- Stream-in and stream-out fire `ChildAdded`, tag signals and the like, indistinguishable
  from real spawns. Reads on a streamed-out instance return stale values.
- `ModelStreamingMode`: `Default`, `Nonatomic`, `Atomic`, `Persistent`, `PersistentPerPlayer`.
  `Atomic` when a script needs the whole model. `Persistent` only for rare must-always-exist
  cases, because it occupies memory permanently; wait for `Workspace.PersistentLoaded`.
- Prefetch **from the server**, always with a timeout, since the default is effectively
  infinite. Results are temporary and not guaranteed:

```lua
--!strict
local function moveWithPrefetch(player: Player, target: CFrame)
	player:RequestStreamAroundAsync(target.Position, 5)
	local character = player.Character
	if character and character.Parent then
		character:PivotTo(target)
	end
end

return moveWithPrefetch
```

Script patterns, predictive and frustum streaming, replication foci, SLIM (models,
avatars, debug tints) and streaming debug panels: read
[references/streaming-and-slim.md](references/streaming-and-slim.md) when writing
streaming-aware code or converting a place.

## Review Red Flags

| Pattern | Problem |
| --- | --- |
| `:Connect` in `PlayerAdded`/`CharacterAdded` with no cleanup, `Player` never destroyed | Leak per respawn or join; serious on long-lived servers. |
| Tables keyed by `Player`/`Instance` with no removal path | Pins the instance and everything under it; iteration slows as well. |
| Weak table used to detect "instance still exists" | Luau can collect the reference while the instance is alive. |
| `RenderStepped`/`Stepped` in new code | Superseded by `PreRender`/`PreSimulation`. Note `Stepped`'s different arguments. |
| Heavy work in a frame event, or a bare `task.wait()` polling loop | Use an event or a slower cadence; slice long work. |
| `wait()`, `spawn()`, `delay()` | `wait` is deprecated; use `task.*`. |
| `FindFirstChild` chains or `GetChildren()` every frame | Cache outside the loop; `GetChildren` allocates each call. |
| Server-side `TweenService` on world objects | Replicates every frame; tween on clients. |
| Remote fired from `PreRender` or mouse move | Throttle; ~500/s per client shared by remote type. |
| `UnreliableRemoteEvent` with large tables or critical data | Over 1,000 bytes is silently dropped; unordered; never queued. |
| Per-frame `Instance.new` for effects | Churn plus GC; pool. |
| Same-named meshes with different asset IDs | Duplicate uploads: lost instancing, double memory. |
| `PreciseConvexDecomposition` on small props | Costly in physics time and memory. |
| `workspace.Map.Chest` in client code | Errors when not streamed in. |
| `Persistent` models as a streaming workaround | Permanent memory; defeats streaming. |
| `--!native` on every script | Startup, memory, size-limit costs; can crowd out hot code. |
| Performance claim based on a Studio number | Studio skews stats; verify on the client on hardware. |

When you (Claude) write frame loops, mob spawners, VFX, or anything that connects to
events, include the cleanup path in the same edit: cleanup added "later" is a leak shipped
now. When asked to make something faster, ask for or take a measurement first, name the
tool and the number, and report before and after. Never claim an improvement you did not
measure on the target; say what remains unverified, especially on real devices, which you
usually can't reach.

## Sources

- https://create.roblox.com/docs/performance-optimization (design, identify, improve, monitor, test-on-hardware, scene-analysis)
- https://create.roblox.com/docs/performance-optimization/microprofiler (use-microprofiler, modes, tag-table, task-scheduler, network)
- https://create.roblox.com/docs/studio/developer-console, /studio/optimization/memory-usage, /studio/optimization/scriptprofiler
- https://create.roblox.com/docs/studio/device-simulator, /studio/network-simulator
- https://create.roblox.com/docs/workspace/streaming (techniques, slim, frustum)
- https://create.roblox.com/docs/scripting/multithreading, /luau/native-code-gen, /luau/metatables
- https://create.roblox.com/docs/scripting/events/remote, /physics/network-ownership, /cloud-services/extended-services
- https://create.roblox.com/docs/production/analytics/performance
- https://create.roblox.com/docs/reference/engine/classes/RunService (also Stats, SceneAnalysisService, Debris, UnreliableRemoteEvent, BasePart, Player, Workspace)
- DevForum: Device/Network Simulator https://devforum.roblox.com/t/4861695; Collision Summaries https://devforum.roblox.com/t/4918568

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
