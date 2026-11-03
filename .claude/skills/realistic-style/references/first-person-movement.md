# Realistic first-person camera and movement

Default Roblox movement (instant start / stop, floaty jump, no camera motion) is the main thing that makes a game feel like Roblox. Fix all of it.

## Camera
- `StarterPlayer.CameraMode = LockFirstPerson` (or `CameraMaxZoomDistance = 0.5` for the same effect with more control).
- **Field of view:** about 70–80 default; widen by 5–10 while sprinting (tweened); add an FOV slider in Settings.
- **Camera effects:** apply in `RunService:BindToRenderStep` *after* `Enum.RenderPriority.Camera`, by offsetting `Camera.CFrame`. Drive every effect with springs (smooth, slight overshoot), never raw sine snaps.
  - **Head bob:** small vertical + side bob tied to the step cycle and speed; stronger when sprinting, none when still.
  - **Sway / lag:** the view and held item lag slightly behind fast mouse / thumb turns.
  - **Strafe tilt:** roll the camera 1–3° into side movement.
  - **Landing dip:** a quick downward dip on landing, scaled by fall height; a small shake on hard landings.
  - **Breathing:** a tiny idle drift when standing still.
- **Settings toggle** to reduce or turn off head bob and shake (motion sickness).

## Body and hands
- **Visible body:** when looking down, the player sees their own legs and torso. Set `LocalTransparencyModifier = 0` on the body parts every frame on the client (keep the head hidden so it doesn't block the view).
- **Viewmodel arms** for held items (tools, phone, flashlight): a client-only arms model attached to the camera, with its own idle, walk, sprint and use animations, and the sway / bob above.
- Interacting is physical: reach animation, the item moves into the hand, doors swing open with the hand.

## Movement
- **Speeds (tune per game):** walk about 8–12, sprint about 16–22, crouch about 4–6 studs/s. Roblox's default 16 walk feels too fast for realism.
- **Weight / inertia:** speed ramps up and down over about 0.1–0.3 s instead of starting and stopping instantly. Use the physics `ControllerManager` (ground / air controllers with acceleration settings) or smoothly tween the speed yourself.
- **Stamina** for sprinting, with heavier breathing sounds and a slower walk when it's empty.
- **Crouch** (lower camera, slower, quieter), **lean** (Q / E) if it fits the game, and a **realistic jump**: lower `JumpPower` / `JumpHeight`, no air control or only a little.
- **Stairs and slopes** handled smoothly (no camera snapping); slower uphill.

## Footsteps and sound
- Footsteps synced to the step cycle (not a timer), chosen by `Humanoid.FloorMaterial` (wood, concrete, grass, metal, water), with random pitch and volume variation.
- Landing, cloth rustle when moving, heavier breathing when sprinting.
- 3D positional sound and reverb that matches the room (small room vs. hall vs. outdoors).

## Input
- PC: WASD, Shift sprint, C / Ctrl crouch, E interact, Q / E lean.
- Mobile and gamepad must work too (`roblox-input` skill): a sprint button, crouch button, interact button that only appears near usable things, and gyro / sensitivity settings.

## Server rules
Movement still runs on the client, so the server checks speed and position for cheating (`roblox-security`). Never trust the client for stamina-gated or speed-gated rewards.
