---
name: roblox-ui-implementation
description: Implement Roblox UI as files in the Rojo tree, verified in Studio. Use for ScreenGui, HUDs, menus, mobile-safe layouts, UI state controllers, touch controls, buttons, progress bars, shop panels, reward surfaces, and verifying UI with the rbxstudio CLI.
---

# Roblox UI Implementation

## Goal

Build Roblox UI that is readable, responsive, touch-friendly, and connected to real game state without trusting the client for valuable decisions.

## Design Gate (read before building)

Do not improvise UI. Before building a non-trivial surface, read its approved design brief:

- `design/DESIGN.md` — global palette, type scale, spacing, and patterns.
- `design/features/<feature-slug>/design.md` — this surface's layout, states, and mobile/tablet/desktop behavior. It must say `Status: approved`.

If the brief is missing or still `draft`, stop and run `roblox-game-ux` to produce and approve it first (or get an explicit user override). Build every state the brief specifies — default, empty, loading, error, and permission/locked — not just the happy path, and match the mobile/tablet/desktop behavior it describes.

## Architecture

- Keep UI construction and state updates in client-side controllers.
- Keep reward, purchase, inventory, and currency decisions on the server.
- Put shared UI constants or formatting helpers in `ReplicatedStorage`.
- Use remotes to request actions, not to grant value directly.
- Separate HUD state, menu state, notification state, and input handling when the UI grows.

## Layout Rules

- Design phone-first.
- Use `UIScale`, constraints, and anchors deliberately.
- Keep important buttons away from corners and system gesture zones.
- Use large tap targets and clear pressed/disabled/loading states.
- Avoid text that can overflow in small buttons or compact panels.
- Keep core HUD visible without blocking the playfield.

## Common Surfaces

- HUD: objective, currency, health/energy, progress, next reward.
- Upgrade panel: current value, next value, cost, affordability.
- Shop: optional offers, clear benefit, no first-session pressure.
- Rewards: short celebration, visible delta, next goal.
- Settings: audio, graphics/performance, input preferences.

## Build Workflow

1. Locate UI code and GUI definitions with Grep/Glob in the Rojo tree.
2. Read controllers and GUI files (`.luau`, `.model.json`, `.rbxmx`) from disk.
3. Build or change UI in files (`roblox-rojo-workflow`) — in code or declarative GUI files, never by hand in Studio.
4. Verify with `roblox-studio-testing`: `rbxstudio play start`, `rbxstudio mouse`, `rbxstudio key`, screenshots.
5. Check `rbxstudio console`.

## Verification

Confirm:

- Text fits on mobile-sized screens.
- Buttons have obvious affordances and state.
- UI does not block first action.
- Valuable actions are server-validated.
- The built surface matches the approved design brief across mobile, tablet, and desktop, including empty/loading/error states.
- No new console errors.

## Next

Verify the surface with `roblox-mobile-playtest` and `roblox-playtest-qa`. If the UI drove new server actions, review the trust boundary with `roblox-security-economy`. Return to `roblox-to-issues` / `roblox-game-development-lifecycle` for the next slice.

