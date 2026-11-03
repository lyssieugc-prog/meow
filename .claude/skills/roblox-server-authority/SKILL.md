---
description: Roblox Server Authority model — Workspace.AuthorityMode = Server, client prediction, rollback and resimulation, RunService:BindToSimulation, InputAction inputs, attributes as synchronized state, instance stitching, mispredictions, limits, adoption decision and debugging tools. Use when building or reviewing competitive or physics-heavy games (FPS, racing, sports, combat), when code uses AuthorityMode, BindToSimulation, SetPredictionMode, GetCameraState or NextGenerationReplication, when debugging mispredictions, or when deciding between this and classic network ownership.
---

# Roblox Server Authority

**Status: generally available since 2026-07-09 and opt-in. No beta flag is needed
to enable or publish it** (DevForum 4727993). Some newer APIs below are `Pending`
in the release notes, and docs and announcements disagree in places. Both are
marked. **The Full Release calls `NextGenerationReplication`, which Server Authority
requires, a Beta and "an early preview", so expect that layer to keep changing.**

**Roblox staff: "Server Authority isn't the right decision for all games."** Decide
first, using the section below. Turning it on changes signals, streaming, physics
stepping and replication for the whole place.

## The Model

The server is the single source of truth and clients are trusted only to report
**inputs**. A client can't report its own position, which removes whole classes of
flyhack and speedhack. To hide latency, the client simulates a few frames ahead of
the last known server state.

```text
Client simulates frame N with local input, and sends that input for frame N
Server runs frame N with the input, and sends authoritative state for frame N
Client receives it a few frames later and compares against its record of frame N
  Match:    nothing happens
  Mismatch: reset to server's frame N, resimulate N+1..now in a single frame
```

Mispredictions are **normal and expected**, since a client can't foresee other
players' inputs. Staff: "Zero mispredicts is not the goal: smooth gameplay is the
goal." Small corrections are fine; design and smoothing hide the rest.

| | Classic (`AuthorityMode.Automatic`) | Server (`AuthorityMode.Server`) |
| --- | --- | --- |
| Character physics | Owning client simulates and replicates it | Server simulates; client predicts |
| Speed/fly/teleport | Detect and bound on the server (see roblox-security, Character Physics Reality) | Server rejects by construction |
| `SetNetworkOwner(nil)` | Authoritative, but input feels a round-trip late | Not needed for this; prediction hides latency |
| Input to server | Your remotes | `InputAction` state, plus remotes for discrete events |
| Custom state | Your tables and remotes | Attributes on predicted instances |
| Client CPU | Lower | Higher: resimulation |

**What it does not replace:** validation of everything else. Purchases, inventory,
trades and remote handlers still follow roblox-security, and input values should be
"checked for sanity before they're processed" (docs). Classic games still need that
skill's character checks.

## Setup

Set `Workspace.AuthorityMode` to `Server` in Properties. That sets the other five:

| Property | Value | What breaks when an existing game flips it |
| --- | --- | --- |
| `NextGenerationReplication` | Enabled | Don't rely on the order of property replication and remote events |
| `PlayerScriptsUseInputActionSystem` | Enabled | Player scripts move under `StarterPlayer`; re-check forked PlayerModule code |
| `SignalBehavior` | `Deferred` | Handlers no longer run at the moment of firing |
| `UseFixedSimulation` | Enabled | Character controllers and joints step at a fixed rate; `time()` becomes fixed-step |
| `StreamingEnabled` | Enabled | Instances can be absent on clients; auto-prediction depends on streaming |

For an existing game, work on a **copy** (staff advice). Games with default avatars
and simple physics need no rewrite; complex custom gameplay must move to
`BindToSimulation` and `InputAction`. Templates: Racing, Soccer, Laser Tag (linked
from the docs). An "Instance Streaming Conversion Skill" exists and is labeled
experimental.

## Simulation Callbacks

Put core logic in a ModuleScript required on **both** client and server, bound with:

`RunService:BindToSimulation(fn: (deltaTime) -> (), frequency: Enum.StepFrequency?, priority: number?) -> RBXScriptConnection`

- **Default frequency is `Hz30`.** The pipeline runs at 60 Hz. Pass `Hz60` for
  movement, and use lower rates (`Hz15`, `Hz10`, `Hz5`, `Hz1`) for periodic checks.
- Default priority 2000; lower runs first; equal priority has no defined order.
- **The callback runs again on resimulation**, so never use it for exactly-once work.
  Staff: nothing else runs then: "No `Heartbeat`, no `RenderStepped`".
- **Simulation Access gate:** touching a member without the reference's **Simulation
  Access** label inside the callback raises an error. That is deliberate.
- Use `time()` for time logic: synchronized, fixed-step and rewound on rollback.
  `tick()`, `os.time()` and `os.clock()` are none of these.

Check a member in the reference. Examples from the YAML, 0.745:

| Has the label | Does not |
| --- | --- |
| `Instance:GetAttribute`, `:SetAttribute`, `:FindFirstChild`, `:Clone` | `Instance:Destroy`, `.Parent` once in the DataModel |
| `BasePart.CFrame`, `.Position`, `.AssemblyLinearVelocity`, `:ApplyImpulse`, `:GetTouchingParts` | `BasePart.Touched` |
| `Humanoid.WalkSpeed`, `.MoveDirection`, `:Move` | `Humanoid.Health`, `:TakeDamage`, `:EquipTool`, `:UnequipTools` |
| `WorldRoot:Raycast`, `InputAction:GetState`, `InputBinding:Fire` | `InputAction.Pressed`, `.StateChanged`, `Sound:Play` |
| `Animator:LoadAnimation`, `:GetTrackByAnimationId` | `RemoteEvent:FireServer`, `:FireAllClients` |
| `Player:GetCameraState`, `RunService:IsResimulating` | `UserInputService.InputBegan` |

**Doc inconsistency:** guide samples read `player.Character`, call
`Players:GetPlayers()` and `RunService:IsServer()` inside callbacks, but the YAML
gives those no label. Whether they error is unverified. Resolve them **outside** the
callback, as in the sample below, which avoids the question.

### Parallel and animation callbacks

| API | Status |
| --- | --- |
| `BindToAnimation` | In YAML; **Pending** (738). Fixed-frequency, before Animators update; same restrictions |
| `BindToSimulationParallel`, `BindToAnimationParallel` | In YAML; **Pending** (743). Script must sit under an `Actor`; serial bindings run first; `priority` orders only within one VM |

`Pending` means shipped but not enabled. Before depending on one, try it in a
playtest of the current build and read the console. If it can't be confirmed, use
`BindToSimulation` and say so.

## What Game Code Must Do

| If you write | What happens | Do this instead |
| --- | --- | --- |
| Game state in a Luau variable | Not rolled back, so it drifts after each correction | Attributes on predicted instances, or rewind in `RunService.Rollback` |
| Attribute write outside the callback, on a predicted instance | Roblox can't track it for resimulation | Write only inside bound functions |
| `UserInputService` in core logic | Not replayed; server never sees it | `InputAction:GetState()` read in the callback |
| Reaction in `InputAction.StateChanged` or `BasePart.Touched` | Can fire on the client for what never happened on the server (Studio Beta post; a fix is Pending, 734) | Poll `GetState()` and `GetTouchingParts()` |
| `:Destroy()` or reparent in the callback | Error | Record intent in an attribute; remove on the server in `PostSimulation` |
| Cached `AnimationTrack` | Rollback may replace it; calls hit a dead track | `Animator:GetTrackByAnimationId()` every step |
| `Sound:Play` or particles in the callback | Error, or replays on resimulation | Read state in `RenderStepped`; undo mispredicted effects |
| Critical logic timed by a remote | Remotes aren't on the shared timeline | Inputs and attributes; remotes for discrete events only |
| Other players' inputs assumed on the client | Not forwarded by default | Server copies them to attributes (below) |
| Different call order on client and server when creating instances | Stitching fails (see below) | Same module, same branch, same count |

**Determinism:** both sides must produce the same result. CCL docs ask for this
explicitly. Anything that differs per machine, like an unseeded random or a value
only one side has, becomes a misprediction (inference from the model; the docs don't
list causes exhaustively). Staff note floating-point can differ across platforms, so
some noise is unavoidable.

### Attributes

Any mismatch on a predicted instance triggers a full rollback, so keep attributes
coarse and stable. To replicate, an attribute must be **among the first 64** on its
instance, with a **name of at most 50 characters** and, if a string, **a value of at
most 50 characters**. Staff are looking at loosening this.

### Prediction control

- Default `Automatic`: near the local character within a dynamic radius, for whole
  assemblies. A welded sword is predicted; a parented but unwelded model is a separate
  assembly and is only predicted inside the radius.
- `RunService:SetPredictionMode(instance, Enum.PredictionMode.On | Off | Automatic)`
  on the client. `Off` means regular network ownership semantics. On the server it is
  documented as client-only; a warning for shared code is removed in a Pending fix (738).
- `RunService:GetPredictionStatus(instance)`: `Authoritative`, `Predicted` or `None`.
- Client simulation, prediction included, only runs in **streamed** areas. Use another
  replication focus for far objects that must keep simulating.

### Creating instances: stitching

`Instance.new`, `:Clone` and `Instance.fromExisting` in a callback appear immediately
on the client and merge with the server's copy. It works by matching GUIDs built from
the class, the source (the script, or the source instance), the simulation frame and
a per-script call counter that resets each frame.

- Parent into the DataModel **in the same frame**. Never hold a speculative instance.
- Set non-simulation properties (`Name`, `Size`, `Parent`) **before** parenting; after
  that, they error.
- `Instance.new` from two scripts with the same text won't match: same script only.

### Input and camera

- `InputContext`s must be descendants of a `Player` (clone a folder per player on the
  server). Server and client read the same data for a frame.
- **Breaking at Full Release:** `Player.InputContexts.CameraContext.CameraAction` is
  gone. Use `Player:GetCameraState()` -> `{CFrame, FieldOfView, ViewportSize}` (Live,
  727). `CFrame` travels with the input stream; the other two replicate normally at
  their own cadence. Client: `LocalPlayer` only. Server: any player.
- `InputBinding:Fire(state)` drives a binding of `Type = Scriptable` from code. Wrong
  type or an `Automatic` binding errors; a disabled action or context silently ignores
  it; a repeated state doesn't re-fire. Default player scripts moving to this is Pending
  (727, 732).

### Other players

By default inputs aren't forwarded, so other characters don't mispredict but render
slightly in the past. For racing, copy inputs to attributes on the server and simulate
from those, as in the Racing template. More forecasting means more mispredictions.

## Minimal Sample

Read [references/minimal-sample.md](references/minimal-sample.md) before writing the
first bound function: a stamina-limited sprint across a shared module, a server script
and a client script. Type-checked, **never played**.

To hide corrections, render a massless, non-colliding clone that follows an invisible
simulated object with `TweenService:SmoothDamp` (docs, Position smoothing). Design
helps as well: slower acceleration, and a fuse before an explosion, make corrections
less visible.

## Limits and Known Incompatibilities

| Limit | Source | Current status |
| --- | --- | --- |
| Mobile and console clients update a few days after desktop and **can't connect** until then | Full Release | Plan releases around it |
| 8 playing tracks per Animator | Full Release | A Pending fix (743) mentions slots 9–16; capacity above 8 unverified |
| Attributes: 64 per instance, 50-char names and strings | Guide | Current |
| Remotes off the shared timeline | Guide, Full Release | Staff looking at options |
| Custom emotes, mood logic and strafing not in the default Animate script | Full Release | Current |
| Changing Tools during a misprediction may delete the Tool's Weld | Full Release | Equip another Tool or unequip to recover |
| **Tools: docs vs announcement conflict.** Guide: "aren't fully supported"; Full Release: "Now fully supported". `EquipTool`/`UnequipTools` have no Simulation Access | Both, YAML | Don't call in callbacks; test your Tools |
| ProximityPrompt, ClickDetector, DragDetector "client-authoritative only" | Client Beta, promised to be fixed before release | **Unverified since:** Full Release and docs are silent. Validate on the server and test |
| Shift-Lock not "100% supporting" | Staff reply, Client Beta | Unverified since |
| Animation graphs were unsupported | Client Beta; graph-editor guide now documents Server mode | `PreviewInStudio` isn't supported there; too many parameters warns of incompatibility (734) |
| Drift with some BodyMovers and Constraints | Staff, May 2026 | `BodyGyro`/`BodyPosition` prediction fix Pending (743); test yours |
| No built-in server-side rewind for hit detection | Staff, May 2026 | Implement it yourself if needed |
| Server use rises; client resimulation costs CPU (staff example: 100 ms of latency = 6.25x simulation load) | Full Release, Deep Dive | Extended Services for Compute exists; try reducing work first |

### Not usable yet

Named in release notes, **absent from the reference YAML and live docs** on 2026-11-01.
Don't write code against them; signatures are unknown.

- `PlayerControlState` (735, Pending): instance for replicating scripted inputs. The
  live reference URL returns 404. A `PlayerControlStateEnabled` flag exists in Studio
  0.745's metadata, an indirect sign only.
- `Workspace.ReplicationFrequency`, `Workspace.ReplicationThroughput` (742, Pending):
  bandwidth controls. Strings are in the 0.745 binary; not documented.

## When to Adopt

**Good fit:** competitive FPS, racing, sports and combat, where movement cheats ruin
the game and fairness matters more than absolute responsiveness.

**Reconsider when:**

- The game is casual, social or building-based, and movement cheats cost little.
  Classic plus roblox-security checks is cheaper.
- It can't run with `StreamingEnabled` or Deferred signals yet. Convert those first.
- It has lots of custom gameplay and little time. All of it must move to callbacks.
- Servers are already near their CPU limit, or most players are on low-end phones
  (test on real ones).
- It depends on anything in the limits table you can't test or work around.
- A launch can't tolerate mobile clients being locked out for days after an update.

**Default avatars and simple physics:** low effort; enable on a copy, play, compare.

## Debugging

| Tool | How | Shows |
| --- | --- | --- |
| Visualizer | Ctrl/Cmd+Shift+F6 in a playtest (docs) | Prediction success (8 s), input accept rate, client-server step delta, server heartbeat FPS (below 59 = can't keep up), predicted count, input drops: too old, out of order, buffer full |
| Timeline Mode | Ctrl/Cmd+Shift+I (staff post; not in docs) | Recent mispredictions. Ctrl/Cmd+U bounding boxes, Ctrl/Cmd+L property labels, Ctrl/Cmd+P pause, Ctrl+Scroll scrub, Ctrl+Y deviation pre/post correction |
| Stitching debugger | Part of the debugger (724) | Instance stitching; how to open it isn't documented |
| Simulation radius | Studio settings: **Are Regions Enabled** | Green cylinder of auto-prediction |
| `RunService.Misprediction` | `(time, instances, stats)`; plugin use | Per-instance `Properties`/`Attributes` with `Predicted` vs `Authoritative`; `ResimulationTime` |
| `RunService.Rollback` | `(time)`, before resimulation | Rewind custom state |
| `RunService:IsResimulating()` | In callbacks and signals | Skip duplicate side effects |
| `Instance.PredictionMode`, `GetPredictionStatus` | Read-only property, client call | Why an instance is or isn't predicted |

Reading the numbers: rare, small corrections are fine. Same property on every frame
means client and server code differ. Inputs "too old" point at the network or a client
that can't keep up. Heartbeat under 59 means the server is overloaded.

### How to test

- **Anti-cheat smoke test (from the Client Beta post):** raise your WalkSpeed from the
  client console in a playtest. The character should snap back.
- **Network Simulator** (see roblox-testing): latency, jitter and packet loss, because
  prediction and rollback are sensitive to them.
- **Two clients** under Server & Clients, one interfering with the other, since other
  players' actions are the main source of mispredictions.
- Real devices: resimulation is a client CPU cost that a desktop hides.

## Review Red Flags

| Red flag | Problem |
| --- | --- |
| `BindToSimulation` without an explicit frequency for movement | Defaults to 30 Hz. |
| Callback bound on only one side | The other side diverges; constant rollbacks. |
| Locals or module tables holding gameplay state | Not rolled back. Use attributes. |
| `tick`, `os.clock`, `os.time` in simulation logic | Unsynchronized. Use `time()`. |
| `UserInputService` deciding gameplay | Server doesn't see it. Use `InputAction`. |
| Effects or `Destroy` inside the callback | Errors or duplicates. Move outside. |
| `AnimationTrack` stored across frames | Rollback can replace it. |
| Long string data in attributes | Over 50 chars doesn't replicate. |
| `CameraContext.CameraAction` | Removed. Use `GetCameraState()`. |
| `SetNetworkOwner` used to "secure" a predicted object | Off the model; check `SetPredictionMode`. |
| `PlayerControlState`, `ReplicationFrequency` in code | Not documented or enabled. |
| "Zero mispredictions" as a goal | Staff say it isn't one. Aim for imperceptible. |
| Purchases trusted because "the game is server authoritative" | It covers simulation, not your remotes. |

When you (Claude) work on a Server Authority game: read `Workspace.AuthorityMode` in the
place first instead of assuming. Check every member used in a callback for the Simulation
Access label. Separate simulation from rendering. State plainly which features are
Pending or conflicting, and run it under Network Simulator before saying it works.
When asked whether to adopt it, give the decision with the limits that apply to this game.

## Sources

- https://create.roblox.com/docs/projects/server-authority
- https://create.roblox.com/docs/projects/server-authority/techniques
- https://create.roblox.com/docs/reference/engine/classes/RunService
- https://create.roblox.com/docs/reference/engine/classes/Workspace
- https://create.roblox.com/docs/reference/engine/classes/Player
- https://create.roblox.com/docs/reference/engine/classes/InputBinding
- https://create.roblox.com/docs/reference/engine/enums/AuthorityMode
- https://create.roblox.com/docs/reference/engine/enums/PredictionMode
- https://create.roblox.com/docs/reference/engine/enums/StepFrequency
- https://create.roblox.com/docs/characters/character-controller-library
- https://create.roblox.com/docs/workspace/streaming
- https://create.roblox.com/docs/animation/graph-editor
- https://create.roblox.com/docs/cloud-services/extended-services
- https://create.roblox.com/docs/scripting/security/network-ownership (its "in beta" note predates the Full Release)
- https://devforum.roblox.com/t/4727993 (Full Release, 2026-07-09)
- https://devforum.roblox.com/t/4606949 (Client Beta, known issues)
- https://devforum.roblox.com/t/4139157 (Studio Beta)
- https://devforum.roblox.com/t/4624565 (Tech Deep Dive, 2026-05-08)

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
