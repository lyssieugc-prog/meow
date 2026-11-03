# meow: Roblox game studio repo

The owner pitches Roblox game ideas; **Claude builds them** (code, UI, models, VFX, cutscenes, promo) in the owner's style. The owner is not learning to code. Be direct and show results.

## Always
- Load the **`meow-house-style`** skill for any game work. It holds the owner's GOOD / BAD style rules, animation and cutscene rules, and monetization rules.
- The full reference vault is `vault/` (an Obsidian vault). Start at `vault/00-Start/Home.md`.
- **Animations and cutscenes are mandatory**: design motion, camera and timing for every feature, not just looks.
- **Gamepasses:** always a 2x → 100x upgrading multiplier plus engaging pay-to-win passes.
- After each build, run the "show your best friend" review and fix the weakest parts.

## When the owner sends references
Describe what you see, including the animation beat by beat. Wait for **GOOD** (adopt) or **BAD** (avoid), then record it in the matching `vault/` note.

## Skills available
`.claude/skills/` has 65 Roblox / Luau skills (architecture, security, DataStores, UI, combat, economy, monetization, playtest QA, publishing, live-ops, 3D asset building, mesh QA, VFX review, typed Luau, jecs / ECS, Jest). Use them; they activate on matching tasks.
More kits are linked (not copied) in `third_party/linked/`. Run `git submodule update --init` to fetch them and follow each kit's own README if the owner wants them installed.

## Tools
Roblox Studio MCP (built in) + Script Sync, ForgeGUI MCP, Figma MCP, the figma-to-roblox-ui skill, Blender MCP. Setup: `vault/60-Tools-MCP/Tools - MCP.md`. Remote MCP servers are pre-declared in `.mcp.json`.

## Code defaults
Luau `--!strict`; the server owns all state and rewards; validate every RemoteEvent; save DataStores safely; `ProcessReceipt` must be idempotent. Commit before big changes.
