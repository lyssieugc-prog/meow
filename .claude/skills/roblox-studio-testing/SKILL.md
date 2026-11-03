---
name: roblox-studio-testing
description: Test a Roblox place in Studio through the rbxstudio CLI (not the Studio MCP server). Use for connecting to Studio, playtesting, reading console output, driving character/keyboard/mouse input, screenshots, and asserting runtime state with exec. Testing only — code is read and written in the Rojo tree, never pulled from or edited in Studio.
---

# Roblox Studio Testing

## Purpose

Studio is the test runtime. Code lives in the Rojo project on disk (`roblox-rojo-workflow`) and syncs in via `rojo serve`. This skill covers everything you do *in* Studio: start a playtest, drive it, observe it, and assert on it.

Use the **`rbxstudio` CLI** through Bash ([nanlulu/roblox-studio-cli](https://github.com/nanlulu/roblox-studio-cli)). It wraps the Studio MCP server with plain-text output and no per-message tool-schema cost. **Do not call `mcp__Roblox_Studio__*` tools** — if that server is connected to Claude Code, suggest `claude mcp remove Roblox_Studio`.

## Setup (once per machine)

- `npm install -g rbx-studio-cli`
- In Studio: enable **Studio as MCP server** (Assistant → Manage MCP Servers).
- Optional: allow `Bash(rbxstudio:*)` in `.claude/settings.json`; `rbxstudio daemon start` for ~10ms calls.

## Start every test session

1. `rbxstudio list` — confirm Studio is connected.
2. `rbxstudio use <id>` if more than one Studio is open.
3. Confirm Rojo is serving and connected, so Studio is running the code on disk. If unsure, `rbxstudio exec` a check that the module you just changed contains the new behavior (e.g. call it and compare the result), rather than reading its source.

## Allowed commands

| Purpose | Command |
|---|---|
| Playtest | `rbxstudio play start` / `rbxstudio play stop` |
| Console (errors, prints, test output) | `rbxstudio console` |
| Move the character | `rbxstudio nav '<target>'` |
| Keyboard / mouse | `rbxstudio key <key> <press\|release\|tap>` · `rbxstudio mouse <x,y> <click\|move\|scroll>` |
| Assert runtime state, run test harnesses | `rbxstudio exec '<luau>'` · `rbxstudio exec --file tests/<case>.luau` |
| Observe runtime instances | `rbxstudio tree --path ... --type ...` · `rbxstudio inspect <path>` |
| Screenshot | `rbxstudio raw screen_capture '{}'` (check `rbxstudio tools` for the exact tool name) |

`print` output from `exec` lands in `rbxstudio console`, not the exec result — print a tagged line (e.g. `[TEST] pass coins=10`) and read the console.

Keep test harness files on disk (e.g. `tests/*.luau`, outside the synced tree or in a test-only folder) so tests are versioned too.

## Forbidden

- `rbxstudio read`, `search`, `grep` on game scripts — read the Rojo files instead. Pulling Luau out of Studio is the expensive, stale path.
- `rbxstudio edit`, or `exec` that creates/changes scripts, remotes, or persistent instances in Edit mode — write those as files in the Rojo tree.
- Treating anything changed in Studio as done. If an `exec` setup in a test proves useful, turn it into a file-backed change.

The single exception is auditing quarantined **third-party** imports (`roblox-creator-store-security-audit`), where the code under review exists only in Studio.

## Playtest discipline

- Always read `rbxstudio console` after a playtest and after each `exec`.
- Stop the playtest (`rbxstudio play stop`) before editing files that need a fresh run.
- Report evidence: commands run, console lines, screenshots.

## Roadmap discipline

Only mark roadmap items complete after implementation, testing, and explicit human approval. Use the `[~]` awaiting-approval marker when built and tested but not yet approved.

## Next

Use `roblox-playtest-qa` for a full QA pass, `roblox-mobile-playtest` for device checks, or `roblox-debugging-bugfix` if a test fails. Make any fix in the Rojo tree via `roblox-rojo-workflow`.
