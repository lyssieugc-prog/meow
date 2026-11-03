# Streaming and SLIM Reference

Read when writing streaming-aware client code or converting a place. Parent: `../SKILL.md`.

## Model and Join Behavior

Streaming applies only to Workspace descendants; `ReplicatedStorage` is out of scope, so
`Atomic` means nothing there. At join, non-spatial Workspace instances replicate; BaseParts,
Atomic/Persistent/PersistentPerPlayer models and, under `Improved`, models with BaseParts are
deferred. Regions inside `StreamingMinRadius` never stream out. Non-BasePart container models
stay. Client-created instances stay unless parented under a server-created one.

- **`Improved`:** a model streams in with its first eligible part plus non-part descendants;
  it leaves with its last part. **`Legacy`** (the documented default) sends the container at
  join. So under `Improved` the model itself can be absent.
- **Atomic:** all initial descendants arrive together. `WaitForChild` the model, then index
  freely. Instances added **later** aren't atomic.
- **Assemblies:** moving assemblies stream in whole, so huge ones spike network and CPU.
- Persistent inside Atomic makes the whole Atomic behave as Persistent: keep hierarchies flat.
- Split big container models into smaller, spatially close ones.

## Script Patterns

| Pattern | Fix |
| --- | --- |
| `workspace.House.Door` | `WaitForChild` with a timeout when required, `FindFirstChild` plus nil check when optional. |
| `character.Humanoid` in `CharacterAdded` | `WaitForChild`: the model parents before its descendants replicate. |
| Effects on `GetInstanceAddedSignal` or `ChildAdded` | Fires again per stream-in. Guard with an attribute such as `Spawned`, or split logic. |
| Loop over `folder:GetChildren()` to find something | Returns only streamed-in items. Ask the server, or key the lookup differently. |
| Remote carrying an instance | Signal and instance travel independently. `WaitForChild` with a timeout, prefetch, or tolerate nil. |
| Distance check on a streamed-out part | Value is stale. Move to the server, or check it's still under Workspace with `FindFirstAncestorWhichIsA("Workspace")`. |
| Loading screen waits on map geometry | Hangs forever with no character and no focus. Don't depend on a spatial instance, or `AddReplicationFocus` at spawn. |
| Client reparent, clone or destroy of server instances | Client copy loses updates; a destroyed server instance streams back. Local property changes can be lost. |

Tracking a tagged set on the client, the documented way:

```lua
--!strict
local CollectionService = game:GetService("CollectionService")

local TAG = "FlickerLight"
local active: { [Instance]: true } = {}

for _, instance in CollectionService:GetTagged(TAG) do
	active[instance] = true
end
CollectionService:GetInstanceAddedSignal(TAG):Connect(function(instance: Instance)
	active[instance] = true
end)
CollectionService:GetInstanceRemovedSignal(TAG):Connect(function(instance: Instance)
	active[instance] = nil -- also fires on stream-out
end)

return active
```

## Foci and Prediction

- `Player.ReplicationFocus`, `AddReplicationFocus`, `RemoveReplicationFocus`: each focus
  adds server work (nine moving foci is comparable to ten players) and limits low-memory
  adaptation on the client. Client physics only runs in streamed areas, even for Persistent
  instances, so far simulation needs a focus.
- `RequestStreamAroundAsync(position, timeOut?)`: yields; temporary; abandoned on low client
  memory; no guarantee. Call from the server before a known move.
- `PredictiveStreamingMode = Enabled` (opt-in): temporary foci at likely respawn points and
  the place a player just CFramed away from; skipped on constrained devices; no code changes.
- `Player.FrustumStreaming` (server-only, opt-in): `Automatic` on zoom, fast directional
  travel or a far camera; `Enabled` for manual control, such as during a scope equip
  animation. Raises memory and maybe CPU. No occlusion awareness; instances churn on fast
  camera turns; auto-disabled when draw distance falls to or under target radius. Under
  `Opportunistic`, leaving the view streams out after 1.5 s outside other foci.
- 743 (Live) improved frame stability while dense areas stream in. 744 (**Pending**) stops
  the view fully streaming out when a dying character's root falls away.

Pause UI: under `PauseOutsideLoadedArea`, watch `Player.GameplayPaused`;
`GuiService:SetGameplayPausedNotificationEnabled(false)` for a custom one.

Debug: in a live game Shift+F3 (Studio Ctrl/Cmd+Shift+F3), then Shift+1 repeatedly to the
fourth panel, **Streaming**. Viewport colors: red below min radius; yellow min to target; blue
at target; green beyond. Raise `CameraMaxZoomDistance` to see it. Network tag to watch:
`Dispatch StreamJob`.

## SLIM

Cloud-transcoded composite meshes with LODs: distant models stay visible while streamed out,
with fewer draw calls. The guide has no beta marker, but the docs don't state GA explicitly.

**Requires:** `StreamingEnabled` true; place saved to Roblox, not a local file; Team Create
enabled. First transcode can take 1-2 minutes, then rejoin. Where a platform lacks support it
falls back to `Disabled`.

**Models:** `Model.LevelOfDetail = SLIM` (values: `Automatic`, `StreamingMesh` legacy,
`Disabled`, `SLIM`). Static only. Not with runtime part or property changes, animation, a
`Humanoid`, or `Scale` 0. Keep each model around ~64 cubic studs of extent per the docs,
grouping parts that belong together. Known issues: overlap and coplanar faces z-fight (move
0.01 studs); unions can render wrong (isolate them); UVs shift on default materials.

**Avatars:** `Workspace.EnableSLIMAvatars = Enabled` (`RolloutState`; `Default` follows the
engine rollout; Studio only, not scriptable). Docs' sample: ~2,600,000 to ~170,000 triangles
for a far audience. R15 with standard or advanced rig only. Excluded: R6, **NPCs**, custom
proportions or scaling, changes after `CharacterAppearanceLoaded` (tools, accessories,
`Highlight`), `StarterCharacterScripts` changes. Testing needs 2+ clients. Known stat bug:
triangle counts may read wrong with SLIM avatars. A fix for characters spawning with
`HipHeight` 0, more likely under SLIM, is in 744 (**Pending**); watch for sinking avatars.

**Debug tints:**
- Live client: Dev Console **DebugVisualizations > SlimTintMode**, Edit permission required.
  Released. Data model state and LOD only.
- Studio: **Beta** (File > Beta Features > SLIM Debug Tints), Visualization Options menu.
  Edit mode has transcoder status only; Play mode has all.
- YAML names: `Enum.SlimTintMode.LevelOfDetail`, `DataModelState`, `TranscoderStatus`, `None`.
  The guide writes `LOD`, which isn't the enum name.
- Data model state: no tint is full instances, teal is streamed in with SLIM, yellow is
  streamed out with SLIM. Transcoder: orange unknown, teal succeeded, sky blue working, red
  failed (see Output). Selecting a model hides its real tint; deselect first.
- 745 (**Pending**) fixes tint modes persisting across games.
- Memory: `GraphicsSlimModels` tag (699).

## Sources

- https://create.roblox.com/docs/workspace/streaming, /workspace/streaming/techniques, /workspace/streaming/slim, /workspace/streaming/frustum
- DevForum SLIM debug tints: https://devforum.roblox.com/t/4924019

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
