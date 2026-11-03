---
name: roblox-mobile-playtest
description: Playtest and QA Roblox experiences for mobile and tablet. Use for device simulator checks, touch controls, safe areas, thumb zones, text readability, orientation, HUD overlap, first-action friction, and mobile performance verification in Studio via the rbxstudio CLI.
---

# Roblox Mobile Playtest

## Goal

Verify that the Roblox experience works and feels good for phone-first players.

## Use Device Simulator

When available, invoke `rbx-device-simulator-lua` before device-specific UI checks. Use it to test phone and tablet form factors, orientation, and layout behavior.

## Workflow

1. Confirm the connection with `rbxstudio list` (see `roblox-studio-testing`).
2. Start play mode with `rbxstudio play start`.
3. Use device simulation when relevant.
4. Exercise controls with `rbxstudio mouse`, `rbxstudio key`, and `rbxstudio nav`.
5. Capture evidence with screenshots.
6. Check `rbxstudio console`.
7. Fix findings in the Rojo files, never in Studio.

## Mobile Checks

- First core action is reachable in 10-30 seconds.
- Tap targets are large and spaced apart.
- Important buttons avoid thumb conflict and system gesture zones.
- HUD does not block movement, combat, collection, or objectives.
- Text is readable on phone and tablet sizes.
- Objective markers and reward feedback are visible without opening menus.
- No keyboard-only requirement exists for core play.
- Menus scroll cleanly and do not hide primary actions.
- Performance remains smooth in the first playable area.

## Report Format

Return:

- Devices or simulated form factors checked.
- First-action time.
- Control and HUD issues.
- Readability issues.
- Console or performance risks.
- Highest-impact fixes before launch.

## Next

Run the full first-session pass with `roblox-playtest-qa`, and send any defect to `roblox-debugging-bugfix`.

