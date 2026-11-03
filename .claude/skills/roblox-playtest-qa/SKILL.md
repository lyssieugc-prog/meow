---
name: roblox-playtest-qa
description: Playtest Roblox experiences in Studio via the rbxstudio CLI with attention to first-session fun, mobile UX, console errors, progression, social hooks, performance, and exploit-sensitive flows.
---

# Roblox Playtest QA

## Purpose

Use this skill to verify that a Roblox change works in Studio and supports player-facing quality. A playtest should prove both behavior and feel.

## Setup

Follow `roblox-studio-testing`: the code under test is the Rojo tree on disk, synced by `rojo serve`.

1. Run `rbxstudio list`.
2. Use `rbxstudio use <id>` if needed.
3. Confirm Rojo is connected so Studio runs the current files.
4. Start play mode with `rbxstudio play start` when verification requires live behavior.

## Test Checklist

- First action is reachable within 10-30 seconds.
- Controls work without keyboard-only assumptions.
- HUD text is readable and not blocking gameplay.
- Rewards, currency, XP, or progress update correctly.
- The next objective is visible after each loop.
- Social features are visible early when relevant.
- Monetization prompts are optional and do not interrupt first-session flow.
- Server-side systems own valuable state.
- No obvious frame drops or runaway instance/script growth.
- `rbxstudio console` has no new errors.

## Commands To Use

- `rbxstudio nav` for movement and route checks.
- `rbxstudio key` and `rbxstudio mouse` for controls and UI.
- Screenshots (`rbxstudio raw screen_capture`) for visual verification.
- `rbxstudio console` for errors and warnings.
- `rbxstudio inspect` or `rbxstudio exec` for state checks after interaction.

Don't read or edit scripts in Studio during QA; failures go back to the Rojo files.

## Report Format

Return:

- What was tested.
- Evidence gathered, including screenshots or console results when available.
- Pass/fail by area.
- Bugs or risks ranked by player impact.
- Recommended next verification step.

Do not mark roadmap items complete without human approval, even after a passing playtest.

## Next

Send any bug to `roblox-debugging-bugfix` and any performance issue to `roblox-performance-optimization`. If the slice passes, mark it in `roblox-triage` / `ROADMAP.md` and prepare the build with `roblox-release-operations`.

