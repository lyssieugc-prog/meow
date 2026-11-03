---
description: Roblox UI and visual effects patterns — safe areas and ScreenInsets, responsive layout (list/flex/grid, AutomaticSize, constraints), text sizing, StyleSheet tokens and themes, UIGradient/UIShadow/UIStroke, CanvasGroup, ViewportFrame, TweenService animation, UIDragDetector, chat UI migration, game feel, and Device Simulator proof at phone size. Use when building or reviewing ScreenGuis, HUDs, shop/inventory interfaces, UI animation, or VFX in Roblox.
---

# Roblox UI & VFX

"The majority of Roblox sessions are played on mobile devices" (official,
`input/mobile`). A UI signed off on a desktop viewport is an untested UI. Most
UI failures have four usual causes: a layout authored for one aspect ratio, text
that shrinks instead of the box growing, controls under the notch or thumbstick,
and a feature only ever enabled in Studio Beta.

## The proof rule (non-negotiable)

**Every UI claim is proven by a capture in the Device Simulator at phone size,
and by measurement, before you say it works.**

1. Test → Device Simulator, Phone category, landscape **and** any orientation
   the game allows; add one tablet, and one console if you ship there. You can
   switch device during a playtest.
2. Use **Actual resolution** to measure pixels, **Physical size** to judge
   whether a button is big enough for a thumb (calibrate it once).
3. Measure from the running client, don't eyeball: `AbsoluteSize`,
   `AbsolutePosition`, `TextFits`, `TextBounds`, `ScrollingFrame.AbsoluteCanvasSize`.
4. Report: device, orientation, screenshot, the numbers. "Looks fine" or a
   capture from the edit viewport is not proof.

Limits (official FAQ): it models screen size, scaling and input, **not hardware
performance**, and doesn't replace a real device. Older pages say "Device Emulator".
Network Simulator is in the same toolbar: use it for loading and retry UI.

## Check maturity before you depend on it

| Feature | Status 2026-11-01 | Note |
| --- | --- | --- |
| Device Simulator, Network Simulator | GA (forum update 10-16) | Docs page lags. |
| `UIGradient.Type` Radial/Conical/Elliptical, `Scale`, `TileMode` | GA (forum update 10-29; 726) | Guide still shows a beta box; forum: flag removed, publishable. Known banding on dark gradients. |
| `UIShadow` base (`BlurRadius`, `Color`, `Offset`, `Spread`, `Transparency`, `ZIndex`, `Enabled`) | Live (711, 724) | |
| `UIShadow.Mode`, `Inset`, `ShowBehindParent`, child `UIGradient` | **Studio Beta, not publishable** ("Upgraded UIShadows"; 733, 735, 738 Pending) | Don't ship depending on these. |
| `InputActionLabel` | **Studio Beta, not publishable** at its 08-06 post; guide still beta | See roblox-input for the live `PreferredBinding` path. |
| `GuiObject.InputSink` | Live (734) | Successor to `Active`, which is not yet tagged deprecated. |
| `ViewportCamera` | Live (734); 736, 739 fixes Pending | **No reference page** in the docs; exists in the API dump. |
| `UIDragDetector.Selectable` | Pending (743) | In API dump, not in reference YAML. |
| `TextBox` copy on mobile with `TextEditable=false` | Pending (734) | |
| `ChatWindowConfiguration.TextChannelDisplayMode` | Pending (739) | Not in reference. |
| TextDocument import of .md/.txt | Pending (742) | |
| Blurry `TextScaled` fix | Pending (717) | |

`Pending` = shipped in the binary, not enabled. Before relying on one, re-read the
release note and check a published client.

## Safe areas: ScreenInsets, not IgnoreGuiInset

| `ScreenGui.ScreenInsets` | Use for |
| --- | --- |
| `CoreUISafeInsets` (default) | Anything interactive. Clear of top bar and cutouts. |
| `DeviceSafeInsets` | Clear of cutouts only; can sit under top bar controls. |
| `TopbarSafeInsets` | Content in the band beside the top bar; flexes with it. |
| `None` | Non-interactive full-bleed art only. |

- `IgnoreGuiInset` is not deprecated but is only a legacy switch: `true` turns
  `CoreUISafeInsets` into `DeviceSafeInsets`. That is **not** full-bleed. Set
  `ScreenInsets` directly.
- `ClipToDeviceSafeArea` defaults to `true`: art in a `None` gui is still clipped
  until you set it `false`.
- `SafeAreaCompatibility` defaults to `FullscreenExtension`, a compatibility shim
  that stretches fullscreen objects. Official advice: avoid it for new work, set
  it to `None` and choose insets per ScreenGui.
- Layer ScreenGuis (a `None` background, a `CoreUISafeInsets` HUD) with `DisplayOrder`;
  toggle with `Enabled`. A disabled one doesn't render, take input or update.
- `ResetOnSpawn` defaults to `true` and also applies when the gui is an indirect
  descendant of StarterGui: UI state and references die on respawn. Set `false`
  for persistent HUDs on a direct child of StarterGui.
- **Mobile:** default controls occupy the bottom-left and bottom-right corners.
  Don't put information there; place custom buttons relative to the jump button,
  within the thumb's reach (differs between phone and tablet).
- Don't detect mobile with `TouchEnabled`: docs say it can be true on touch
  laptops, and `TouchScreenEnabled`'s reference says it can be false on some.
  Use `UserInputService.PreferredInput` for input and
  `GuiService.ViewportDisplaySize` (Small/Medium/Large) for layout.

## Layout

- **Scale for what adapts, Offset for hairlines.** `UIAspectRatioConstraint`
  keeps square things square; `UISizeConstraint` bounds extremes.
- **Order of reach:** `UIListLayout` for rows and columns, flex for distribution
  (`HorizontalFlex`, `VerticalFlex`, `ItemLineAlignment`, per-item `UIFlexItem`
  with `FlexMode` `Fill`/`Grow`/`Shrink`/`Custom`), `UIGridLayout` for uniform
  tiles. Flex has a slight cost when resizing or adding items, so use it with
  purpose. `Padding` on the layout is only between items; use `UIPadding` around.
- Layouts override `Position`/`Size`, **so tweening them on a laid-out child does
  nothing useful**. Tween a child `UIScale`, or an inner frame. Size and aspect
  constraints override the layout's size.
- `AutomaticSize` makes `Size` the **minimum**. `ScrollingFrame` uses
  `AutomaticCanvasSize` instead. Respects `AnchorPoint`.
- Show controls by context (e.g. `ProximityPrompt`) rather than permanently. Studio's
  "Group as Frame" wraps selected GUI objects (727).
- `SurfaceGui`/`BillboardGui`: set `BillboardGui.MaxDistance`.
  `DistanceLowerLimit`/`DistanceUpperLimit` are deprecated (693).

## Text

Official guidance has moved away from what most Roblox UI code does:

- **Avoid `TextScaled` on screen UI; use `AutomaticSize`.** Reference: "avoid
  usage of `TextScaled`". It can scale to unreadable, and **it ignores
  `GuiService.PreferredTextSize`**, breaking the player's Text Size setting. It
  remains useful in `BillboardGui`. Never combine it with `AutomaticSize` on one
  label.
- If you must use it, add `UITextSizeConstraint` with `MinTextSize` ≥ 9 (docs).
  Note a constraint also caps the player's preferred size.
- `TextWrapped` plus `AutomaticSize = Y` for anything translated; test the longest
  locale. `TextFits` tells you if it was clipped.
- Non-integer `TextSize` from a rule used to blur (fixed 709): use integers.
- Don't tween `UIStroke.Thickness` on text: it renders many glyph sizes, causing
  cost or flicker (docs).
- Player-authored text must go through filtering. See `ui/text-filtering`.

## Styling: tokens, themes, queries

Prefer one `StyleSheet` over per-instance colors. A `StyleLink` connects it to a
ScreenGui; **only one StyleSheet applies per tree**, so compose with `StyleDerive`.
Selectors: `TextButton`, `.Tag`, `#Name`, `A > B`, `A >> B`, `A, B`, `:Hover`,
`::UICorner`, nested `::UIStroke::UIGradient` (692), `@Query`.
Built-in queries: `@ViewportDisplaySizeSmall|Medium|Large`,
`@PreferredInputTouch|Gamepad|KeyboardAndMouse`, `@ReducedMotionEnabledTrue|False`.
`StyleRule:SetPropertyTransitions` gives hover and press animation without code.
StyleSheets also style plugins (699).

Read [references/styling.md](references/styling.md) when building a token or theme
system; it has a full sample.

## Appearance

- **`UIGradient`:** two-color or evenly-spaced gradients are fastest; animating
  type, scale, tile, offset or rotation is cheaper than color or transparency;
  stay under 1000 at once (forum). Works on `UIStroke` child and `Path2D` (726).
- **`UIStroke`:** `ApplyStrokeMode` picks text vs border; two strokes can do both;
  layer with `ZIndex` (equal `ZIndex` order is undefined). `BorderStrokePosition`,
  `BorderOffset`, `LineJoinMode`.
- **`UICorner`:** per-corner radii `TopLeftRadius` etc.; `Scale ≥ 0.5` gives a pill.
- **`UIShadow`:** follows `UICorner` and `Rotation`, not `Path2D` or textures.
  Under 100 on screen (forum). Replace 9-slice fake shadows.
- **Press and disabled states:** `GuiObject.Interactable = false` sets
  `GuiState.NonInteractable`; `GuiButton.SecondaryActivated` covers right-click and
  long-press. `ImageButton` has `HoverImage`/`PressedImage`. Touch has no cursor, so
  never hide information behind hover.
- **Input through UI:** `InputSink` (`None`/`Activate`/`All`) decides what passes to
  elements and the 3D world behind. `GuiButton` defaults to `Activate`.

## CanvasGroup: clipping and fades, at a price

`ClipsDescendants` clips to the plain rectangle: clipping by rounded corners or by
a rotated container **is not supported** (docs). `StarterGui.ClipsDescendantsSupportsRotation`
only covers rotated *descendants*. `CanvasGroup` + `UICorner` solves both, and
`GroupTransparency` fades a whole panel at once.

Documented costs:

- Consumes extra texture memory; quality and total memory are limited by the
  client's `QualityLevel`. **Over the cap it renders blank.**
- Recommended with **static sizes**: resizing allocates a new texture, so tween a
  `CanvasGroup`'s transparency, not its `Size`.
- Flattens only when the `LayerCollector.ZIndexBehavior` is `Sibling`.
- Always clips descendants to its bounds.

Softness: quality limited by graphics level is documented, and creators report
blur on low settings (forum 2312514). Keep body text outside when crispness matters
and check a capture at low graphics quality. Forum bug 4933832 (status not checked)
reports gradients on a CanvasGroup failing on Adreno 7-series GPUs: test real Android.

## ViewportFrame (3D in UI)

Caveats: no shadows or post-processing, `Neon`/`Glass` at lowest quality, no nested
GuiObjects, and environment lighting off unless a `Sky` child is present. Light with
`Ambient`, `LightColor`, `LightDirection`.

```lua
--!strict
local function frameModel(viewport: ViewportFrame, model: Model, fovDegrees: number)
	local camera = Instance.new("Camera")
	camera.FieldOfView = fovDegrees
	camera.Parent = viewport
	viewport.CurrentCamera = camera
	model.Parent = viewport

	-- Derive distance from the bounds: fixed offsets break on small or odd models.
	local center, size = model:GetBoundingBox()
	local radius = size.Magnitude / 2
	local distance = radius / math.sin(math.rad(fovDegrees) / 2)
	local eye = center.Position + center.LookVector * distance
	camera.CFrame = CFrame.lookAt(eye, center.Position)
end
```

- To move the view, **update the camera, not the objects** (docs).
- **Animation needs a `WorldModel`:** parts inside can be animated and raycast but
  aren't simulated, and Humanoid joints are set up correctly. Create it only while
  shown and delete it when unused (docs).
- Authoring at edit time: a plain `Camera` didn't replicate. `ViewportCamera` (734)
  inherits from `Camera` and replicates; Studio upgrades pasted cameras (739,
  Pending). Runtime client code can keep `Instance.new("Camera")`.

## Animation

```lua
--!strict
local GuiService = game:GetService("GuiService")
local TweenService = game:GetService("TweenService")

local OPEN = TweenInfo.new(0.25, Enum.EasingStyle.Back, Enum.EasingDirection.Out)
local CLOSE = TweenInfo.new(0.15, Enum.EasingStyle.Quad, Enum.EasingDirection.In)

-- Honor the player's Reduce Motion setting: time 0 snaps to the target.
local function motion(info: TweenInfo): TweenInfo
	if GuiService.ReducedMotionEnabled then
		return TweenInfo.new(0)
	end
	return info
end

local function setOpen(panel: CanvasGroup, scale: UIScale, open: boolean)
	if open then
		panel.Visible = true
	end
	local info = motion(if open then OPEN else CLOSE)
	TweenService:Create(scale, info, { Scale = if open then 1 else 0.85 }):Play()
	local fade = TweenService:Create(panel, info, { GroupTransparency = if open then 0 else 1 })
	fade.Completed:Connect(function(state: Enum.PlaybackState)
		-- Cancelled means a newer tween took over: don't hide a panel reopening.
		if not open and state == Enum.PlaybackState.Completed then
			panel.Visible = false
		end
	end)
	fade:Play()
end
```

- Hand-rolled easing loops leak connections and are hard to cancel. Use TweenService,
  or Style transitions for state changes. For frame logic use `RunService.PreRender`;
  `RenderStepped` is superseded.
- `GuiObject:TweenSize`/`TweenPosition`/`TweenSizeAndPosition` are deprecated, as
  are `GuiObject.Draggable`, `DragBegin` and `DragStopped` (use `UIDragDetector`).
- **Two tweens on one property don't fight:** the first is cancelled and overwritten
  (TweenService reference). The real bug is the `Completed` handler that assumes the
  tween finished: check the `PlaybackState`, as above.

## Drag and world-space

- **`UIDragDetector`** under a GuiObject gives input-agnostic dragging with no code:
  `DragStyle` (`TranslateLine`, `TranslatePlane`, `Rotate`, `Scriptable`),
  `ResponseStyle`, `BoundingUI`, events `DragStart`, `DragContinue`, `DragEnd`. It
  doesn't respond in Studio while Select/Move/Scale/Rotate is active. It currently
  forces the object `Selectable` (743 Pending makes that configurable), so check
  gamepad navigation. For 3D objects use `DragDetector` (`ui/3D-drag-detectors`).
- **World→UI:** `Camera:WorldToViewportPoint` ignores the GUI inset;
  `WorldToScreenPoint` accounts for it. Pick by your ScreenGui's insets and **measure
  it in the simulator**: a wrong choice shows up as a top-bar-sized offset.
- **Particles:** `emitter:Emit(n)` for bursts, with `Enabled = false`. Cap and pool
  emitters on mobs (see roblox-performance). Anchor effects to `GetBoundingBox` or
  a named attachment, not fixed Y-offsets.

## Chat UI (migration deadline)

Platform-integrated chat becomes **required in May 2027** (forum 4948433,
`chat/platform-integrated-chat`). Test with `TextChatService.PlatformIntegratedChat =
Enabled` in Studio. Then message text isn't readable from code (APIs return `""`), and
custom chat UIs can't render messages, so a bespoke chat frame is a dead end. Use
`TextChannelWindow` (also works in `SurfaceGui`): `Target`, `FontFace` applied when
`UseDefaultFont = false` (reference; release note 736 used other names). It only
accepts `UICorner`, `UIAspectRatioConstraint`, `UISizeConstraint` and scripts as
children, and `ChatWindowConfiguration` doesn't apply to it.

## Game feel cheatsheet

Ordered by cost-effectiveness. Durations are starting points, not rules:

1. **Scale pop** on press or reward: `UIScale` to 1.1 and back, ~0.1 s out, ~0.15 s back.
2. **Color flash:** a brief bright overlay.
3. **Sound** on every meaningful action, with slight `PlaybackSpeed` variation.
4. **Count-up numbers:** tween a `NumberValue` rather than snapping.
5. **Particle burst** at the reward source.
6. **Shake:** tiny, rare, big wins only. Off under `ReducedMotionEnabled`.
7. **Anticipation:** a ~0.3 s wind-up before a reveal.
8. **Haptics:** `HapticEffect` on supported phones and gamepads.

## Red flags

| Red flag | Fix |
| --- | --- |
| "Works" with no simulator capture | Capture at phone size, measure, then claim. |
| `IgnoreGuiInset = true` for full-bleed | `ScreenInsets = None`, `ClipToDeviceSafeArea = false`. |
| Buttons in a `None` or `DeviceSafeInsets` gui | `CoreUISafeInsets`. |
| Controls in the bottom corners on touch | Move clear of default controls, relative to jump. |
| `TextScaled` on HUD labels | `AutomaticSize` + wrap; honors Text Size setting. |
| `TouchEnabled` as "is mobile" | `PreferredInput`, `ViewportDisplaySize`. |
| Tweening `Size` on a laid-out child or a CanvasGroup | Child `UIScale`. |
| `Completed` handler ignoring `PlaybackState` | Check for `Completed` vs `Cancelled`. |
| Hard-coded colors across instances | Tokens in a StyleSheet. |
| Shipping an Upgraded UIShadow feature or `InputActionLabel` | Beta: not publishable. Use the fallback or wait. |
| Custom chat frame reading message text | `TextChannelWindow` before May 2027. |
| ScreenGui logic reading workspace state directly | Controllers receive state (see roblox-architecture); UI renders. |
| Hard-coded "Press E" hints | `PreferredBinding` (see roblox-input). |

When you (Claude) build UI: lay out the hierarchy first (containers, layouts,
constraints, insets), confirm it reads at phone size in the Device Simulator, then
wire tokens, animation and state: visuals before logic. Report device, orientation,
capture and measurements. Never state a feature's status from memory.

## Sources

- https://create.roblox.com/docs/ui, /ui/on-screen-containers, /ui/position-and-size, /ui/size-modifiers, /ui/appearance-modifiers, /ui/list-flex-layouts, /ui/grid-table-layouts, /ui/frames, /ui/scrolling-frames, /ui/viewport-frames, /ui/animation, /ui/ui-drag-detectors, /ui/3D-drag-detectors, /ui/2D-paths, /ui/styling, /ui/styling/css-comparisons, /ui/text-filtering
- https://create.roblox.com/docs/reference/engine/classes/ScreenGui, /CanvasGroup, /TextLabel, /UIShadow, /UIGradient, /WorldModel, /ViewportFrame, /TweenService, /GuiObject, /TextChannelWindow
- https://create.roblox.com/docs/studio/device-simulator, /studio/network-simulator, /production/publishing/adaptive-design, /production/publishing/accessibility, /projects/cross-platform, /input/mobile, /chat/platform-integrated-chat
- Forum: https://devforum.roblox.com/t/4861695 (Device/Network Simulator), /t/4846594 (gradients), /t/4934553 (UIShadow), /t/4779420 (InputActionLabel), /t/4948433 (chat)

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
