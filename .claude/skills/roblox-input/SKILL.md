---
description: Roblox cross-platform input — Input Action System (InputContext, InputAction, InputBinding), Input Action Manager, when to use UserInputService vs ContextActionService, keyboard/gamepad/touch/TV remote bindings, modifiers, UIButton binding, priority and sink, key hints in UI, PreferredInput, default PlayerScripts migration and @rbx/PlayerModule, and the server validation boundary. Use when writing or reviewing player controls, keybinds, hotkey hints, mobile buttons, gamepad support, or any UserInputService/ContextActionService code.
---

# Roblox Input

Input code usually fails in one of these ways: it works only on the device it was
written on, one gameplay key fires while a menu is open, a mobile button is bound to
the wrong instance, code polls an event that fires only on change, or the server
trusts what the client pressed. The Input Action System (IAS) addresses most of these
by making input data: contexts, actions and bindings, not per-device scripts.

## Which API

| API | State (YAML + forum) | Use it for |
| --- | --- | --- |
| **Input Action System** | GA since 2026-06-11, publishable | **All new gameplay and UI actions.** Required for the core simulation under Server Authority. |
| `UserInputService` | Not deprecated | Device state and detection: `PreferredInput`, `MouseBehavior`, `MouseIcon*`, `GetStringForKeyCode`, `GetImageForKeyCode`, `OnScreenKeyboard*`, gamepad capabilities, gestures IAS doesn't express. |
| `ContextActionService` | Binding methods "aren't recommended for new work and are planned for deprecation" (class docs). Not yet tagged deprecated, except `BindActionToInputTypes`. | Keep existing code working; migrate when touched. Don't start new work on it. |
| `Mouse` events (`KeyDown`, `KeyUp`) | Deprecated | Never. |
| `InputAction:Fire()` | Deprecated | `InputBinding:Fire()` on a `Scriptable` binding. |

Old posts call IAS "beta". IAS and the Input Action Manager are GA; `InputActionLabel`
isn't (see below).

## The model

```
ReplicatedStorage/Inputs (Folder, recommended)
  PlayContext        InputContext   Enabled, Priority, Sink
    Sprint           InputAction    Type = Bool
      KeyboardBinding  InputBinding KeyCode = LeftShift
      GamepadBinding   InputBinding KeyCode = ButtonY
      TouchBinding     InputBinding UIButton = <GuiButton in PlayerGui>
```

- An action registers with its first `InputContext` ancestor (no ancestor: a default
  context). Nested contexts have no effect.
- **Every action needs a binding per input family:** keyboard/mouse, gamepad, touch.
- Author in **Input Action Manager** (Window → Input → Input Action Manager, GA
  2026-10-29). It flags missing bindings per schema, flags duplicate keycodes within
  a context, and shows a live State column in Play Solo. With the new PlayerScripts
  on, it also shows default actions read-only. Known issues: editing is still
  possible in play mode (unintended), `ResponseCurve` displays poorly for
  `Direction2D`, and it can desync from Properties if both are edited together.

## Action types and values

| `Type` | Binding input | Value | Signals |
| --- | --- | --- | --- |
| `Bool` | Key, button, `UIButton`, analog trigger, `TouchPosition` | boolean | `Pressed`, `Released`, `StateChanged` |
| `Direction1D` | `KeyCode`/`Up` → 0..1, `Down` → 0..-1, `MouseWheel`, `TrackpadPinch`, `TouchPinch` | number | `StateChanged` |
| `Direction2D` | `Thumbstick1/2`, composites `Up/Down/Left/Right`, `MouseDelta`, `TrackpadPan`, `TouchDelta` | Vector2 | `StateChanged` |
| `Direction3D` | Composites plus `Forward` (Z -1) and `Backward` (Z +1) | Vector3 | `StateChanged` |
| `ViewportPosition` | `MousePosition`, `TouchPosition` | Vector2 pixels | `StateChanged` |

`Pressed` and `Released` fire **only** for `Bool`. A mismatched `KeyCode` either
doesn't fire or sends a converted value.

Binding properties:

- Analog to `Bool`: `PressedThreshold` (default 0.5), `ReleasedThreshold` (default 0.2).
- Single-direction sticks: `Thumbstick1Up` etc. (696). Tuning: `Scale`, `Vector2Scale`,
  `Vector3Scale` (both apply). `MouseDelta` and
  `TouchDelta` report pixels, so the guide scales them by `0.01`.
- `ResponseCurve` (1–10, default 1): only `Direction2D` on a thumbstick.
- `ClampMagnitudeToOne` (default true) prevents faster diagonals. Set `false` for
  independent axes or inputs that exceed 1, like wheel velocity.
- `InputBinding.Deadzone` is in the API dump, not the reference: treat as undocumented.

## Poll continuous input

`StateChanged` fires **once per change**, so a thumbstick held at one angle fires
once. For continuous input, read `GetState()` every frame and multiply by delta time.
Use events for discrete actions.

```lua
--!strict
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local inputs = ReplicatedStorage:WaitForChild("Inputs")
local playContext = inputs:WaitForChild("PlayContext") :: InputContext
local menuContext = inputs:WaitForChild("MenuContext") :: InputContext
local sprint = playContext:WaitForChild("Sprint") :: InputAction
local aim = playContext:WaitForChild("Aim") :: InputAction
local actionRequest = ReplicatedStorage:WaitForChild("ActionRequest") :: RemoteEvent

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui") :: PlayerGui
local hud = playerGui:WaitForChild("Hud") :: ScreenGui

-- Touch: bind to the live PlayerGui clone, not the StarterGui original.
local touchBinding = sprint:WaitForChild("TouchBinding") :: InputBinding
touchBinding.UIButton = hud:WaitForChild("SprintButton") :: GuiButton

-- Discrete: events. Input is a request; the server decides.
sprint.Pressed:Connect(function()
	actionRequest:FireServer("SprintStart")
end)
sprint.Released:Connect(function()
	actionRequest:FireServer("SprintStop")
end)

-- Continuous: poll, frame-rate independent.
local AIM_SPEED = 1.5
local yaw = 0
RunService:BindToRenderStep("Aim", Enum.RenderPriority.Camera.Value, function(dt: number)
	local state = aim:GetState() :: Vector2
	yaw -= state.X * AIM_SPEED * dt
end)

-- Modes: swap contexts instead of checking "is menu open" in every handler.
local function setMenuOpen(open: boolean)
	menuContext.Enabled = open
	playContext.Enabled = not open
end
setMenuOpen(false)

-- Adapt UI to the primary device, never to "is mobile".
local function onPreferredInput()
	local preferred = UserInputService.PreferredInput
	local touch = hud:WaitForChild("TouchButtons") :: Frame
	touch.Visible = preferred == Enum.PreferredInput.Touch
end
UserInputService:GetPropertyChangedSignal("PreferredInput"):Connect(onPreferredInput)
onPreferredInput()
```

The sample assumes a `Hud` ScreenGui with `ResetOnSpawn = false`.

## Contexts: enable, priority, sink

- **`Enabled`:** disabled contexts send no signals, except a final "end" signal when
  toggled off while held or non-zero. **Handle `Released` as cleanup**, or sprint
  sticks when a menu opens. `InputAction.Enabled = false` resets that action's state.
- **`Priority`:** higher runs first. **`Sink = true`** blocks the same bound keycodes in
  lower-priority contexts; **equal priority still receives the input**. Sink only
  affects inputs that this context binds, so unbound keys pass through.
- Guide example: `Priority = 2000` + `Sink` to run before the IAS PlayerScripts'
  contexts. Sinking `Space` or the thumbstick there would take them from default jump or
  movement, so bind only what you intend to take.
- Use separate contexts for navigation and gameplay so one button can't fire both.

## Bindings by device

**Gamepad conventions** (`input/gamepad`): `ButtonA` accept/primary, `ButtonB`
cancel/secondary, `Thumbstick1` move, `Thumbstick2` camera, `ButtonL2`/`ButtonR2`
primary actions, `L1`/`R1`/`X`/`Y` secondary. `ButtonStart` opens the Roblox menu.
Test with Controller Emulator.

**Modifiers:** `PrimaryModifier` and `SecondaryModifier` require the keycode held
before the main input; order between the two doesn't matter. `UIModifier` is the same
with a `GuiButton`, for touch (for example a thumbstick responding only to touches
starting inside a region). The reference says `Enum.KeyCode.Unknown` means no
modifier, but that item isn't in the `KeyCode` enum YAML, and the guide compares with
`Enum.KeyCode.None`. Leave the property unset rather than writing either.

**Touch:**

- `UIButton` is for `Bool` actions. Buttons are `GuiButton`s, placed within thumb reach
  and clear of default controls (see roblox-ui). Keep their visibility tied to
  `PreferredInput`.
- **The guide links `UIButton` to the StarterGui button at edit time. A creator report
  (forum 4747338, not staff) found this didn't fire until re-assigned at run time to
  the PlayerGui instance, which is plausible as StarterGui is cloned.** Assign at run
  time as in the sample, and prove it in the Device Simulator on a touch device.
- A `ResetOnSpawn` gui replaces the button: re-bind on respawn, or set it `false`.
  This consequence isn't documented; verify with a respawn test.
- Gestures: `TouchDelta`, `TouchPinch`, `TouchPosition`. Simulator: Alt+drag is
  pinch/rotate, Alt+Shift+drag is two-finger pan.
- The guide's `ViewportPosition` table says `Enum.KeyCode.Touch`, which isn't in the
  enum YAML; `TouchPosition` is.

**TV remote / MicroGamepad:**

- Remote keycodes (`ButtonCenter`, `ButtonBack`, `ButtonUp/Down/Left/Right`) currently
  arrive as **legacy** events: `ButtonA`, `ButtonB`, `DPad*`, and in Classic/Follow
  camera `Thumbstick1/2`. **Bind the legacy codes.**
- Detection conflict: `input/micro-gamepad` says `PreferredInput.MicroGamepad` "is not
  currently set", but release note 739 (Live) says it is. Check both:
  `PreferredInput == MicroGamepad`, or `Gamepad1` connected with neither thumbstick via
  `GamepadSupports`. `ButtonCenter` support identifies a TV remote. Legacy codes alone
  don't, since a full gamepad sends them too.
- UI: visible focus state, every item reachable by D-pad, center = primary, back =
  close. Test with Device Simulator (Android TV) plus Controller Emulator's TV Remote.

## Key hints in UI

Never hard-code "Press E".

| Path | Status | Use |
| --- | --- | --- |
| `InputAction.PreferredBinding` + `DisplayName`/`DisplayImage` | **Live (730)** | Default for shipping today. |
| `GetStringForKeyCode(keyCode, format?)`, `GetImageForKeyCode` | Live; `Abbreviated` format 727 | Fallback when no display override is set. |
| `InputActionLabel` | **Studio Beta at last post (08-06), guide still beta; 732 Pending, non-publishable then** | Not in shipped code until a release post says so; check `File → Beta Features` and the release notes. |
| `InputAction.DisplayName` | Live (740) | Localized action name for controls menus. |

```lua
--!strict
local UserInputService = game:GetService("UserInputService")

local function bindHint(action: InputAction, label: TextLabel)
	local function refresh()
		local binding = action.PreferredBinding
		if binding and binding.DisplayName ~= "" then
			label.Text = binding.DisplayName
		elseif binding and binding.KeyCode ~= Enum.KeyCode.None then
			label.Text = UserInputService:GetStringForKeyCode(binding.KeyCode)
		else
			label.Text = ""
		end
	end
	action:GetPropertyChangedSignal("PreferredBinding"):Connect(refresh)
	refresh()
end

return bindHint
```

`PreferredBinding` can be nil, and a touch or composite binding has no single `KeyCode`,
so give those a `DisplayName` or `DisplayImage`. Pending fixes affect it (732, 745), so
test device switching. `GetImageForKeyCode` is limited to Xbox, PlayStation and Windows.

## Detect input with PreferredInput

Use `UserInputService.PreferredInput` plus its change signal. Not `TouchEnabled` (true
on touch laptops per the guide, and the `TouchScreenEnabled` reference says it's false
on some). Not `GetLastInputType()` as a mode check: it thrashes between mouse and
keyboard and returns values like `Focus`, flickering layouts. Keep that only for logic
needing the exact last input type. Handle all four values, including `MicroGamepad`.

## Default PlayerScripts migration

`Workspace.PlayerScriptsUseInputActionSystem` (`RolloutState`, set in Studio; not
settable by game scripts) moves default controls onto IAS:

| Phase | Default | Effect |
| --- | --- | --- |
| 1, now | `Disabled` | Opt in with `Enabled`. |
| 2, ETA early 2027 | `Enabled` | Games using old scripts or sinking patterns may break. Opt-out is temporary. |
| 3, ETA mid 2027 | Property removed | Old scripts gone. |

Dates are forum estimates ("ETA"), not commitments. Audit now:

- **Forked PlayerModule:** enable the property, then set
  `StarterPlayer.CreateDefaultPlayerModule = false` to keep yours (write access is
  `PluginOrOpenCloud` since 745; the YAML predates this).
- **Calls into internals**, like `PlayerModule:GetControls()`, may no longer exist.
  Roblox recommends not depending on internals at all. When opted in,
  `require("@rbx/PlayerModule")` is available (735, Live). No reference page describes
  its API, so inspect it before use.
- **Custom input alongside defaults:** sinking changes. Several default-script changes
  remain Pending (727, 729, 732, 734). Test movement, camera and jump, with a menu open,
  on each device.

## Server boundary

**Input is a request, never a fact.** See roblox-security.

- **Standard games:** treat handlers as client-side. Fire a remote with intent ("use item
  3"), and the server validates range, cooldown, ownership and state. Never send the
  outcome. The guide's client `WalkSpeed` sprint is a demo, not a security model. The
  Workspace reference says the server "is able to process player inputs" with the
  property on, but the only documented server read is under Server Authority.
- **Server Authority:** requires the property above enabled. Inputs replicate and replay
  during resimulation, so use `InputAction`s for **all inputs affecting the core
  simulation**, not `InputBegan`, and "check for sanity before they're processed".
  `InputContext`s must be **descendants of the `Player`** (clone per player from a server
  script); read them in `RunService:BindToSimulation`. Note 738 Pending: destroying or
  reparenting a context could stop input reaching the server for the session.

## Release-note pitfalls

| Symptom | Note | Status |
| --- | --- | --- |
| Touch-bound action releases when the finger exits the button, not when it lifts | 722 | Pending |
| Mouse clicks on UI buttons or in menus registered as input | 703 | Pending |
| `MouseDelta`/keys sunk by `DragDetector` or `Active` GUI | 743 | Pending |
| Signals firing on rollback | 734 | Pending |
| Wrong `PreferredBinding` from inactive inputs | 745 | Pending |
| Thumbstick overriding bindings in deadzone | 692 | Live |
| Multiple touches on `UIButton`/`UIModifier` | 733 | Live |
| `Direction3D` forward/back sign | 693 made it right-handed | Live: Forward is Z -1 |

Pending: your client may still behave the old way. Add a test, not a permanent workaround.

## Red flags

| Red flag | Fix |
| --- | --- |
| New `BindAction` or `InputBegan` for a gameplay action | IAS action with three bindings. |
| An action with only a keyboard binding | Add gamepad and touch; IAM flags it. |
| `StateChanged` driving continuous aim or movement | Poll `GetState()` × dt. |
| `Pressed` on a non-`Bool` action | Fires never. Use `StateChanged` or change `Type`. |
| `if menuOpen then return end` in input handlers | Contexts with `Enabled`/`Sink`. |
| Sink at high priority over keys you didn't mean to take | Bind only intended keys; test defaults. |
| `UIButton` set only on the StarterGui instance | Assign the PlayerGui clone at run time. |
| Forked PlayerModule, no audit | Handle the 2027 rollout now. |
| `InputAction:Fire()` | `InputBinding:Fire()`, `Type = Scriptable`; throws on `Automatic`, ignored silently if disabled. |

When you (Claude) add a control: define the context and action first, give it all
three bindings, decide discrete (events) vs continuous (poll), define the server's
validation, then add the hint through `PreferredBinding`. Prove it per device in the
Device Simulator and Controller Emulator: state what you ran. Don't call a feature
live from memory: check this table and the release notes.

## Sources

- https://create.roblox.com/docs/input, /input/input-action-system, /input/mobile, /input/gamepad, /input/micro-gamepad, /input/mouse-and-keyboard
- https://create.roblox.com/docs/reference/engine/classes/InputContext, /InputAction, /InputBinding, /InputActionLabel, /UserInputService, /ContextActionService, /Workspace, /StarterPlayer
- https://create.roblox.com/docs/reference/engine/enums/PreferredInput, /InputActionType, /InputBindingType, /KeyCode, /RolloutState
- https://create.roblox.com/docs/projects/server-authority, /studio/device-simulator, /studio/controller-emulator
- Forum: https://devforum.roblox.com/t/4678416 (IAS full release, rollout), /t/4965021 (Input Action Manager full release), /t/4779420 (InputActionLabel), /t/4747338 (community UIButton report)

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
