# Tools and MCP servers

## Roblox Studio MCP (built in)
In Studio: open your Place → Assistant → … → Manage MCP Servers → **Enable Studio as MCP server** → enable Claude Code under Quick connect. Pair it with **Script Sync** (Studio beta), which mirrors scripts to `.luau` files on disk. The browser version of Claude cannot reach Studio; use Claude Code / Claude desktop on the same computer.
Guide: https://vi3w.in/guides/build-roblox-games-with-claude

## ForgeGUI MCP (AI art for thumbnails, icons, GUI / HUD)
- `claude mcp add --transport http forge-mcp https://forgegui.net/mcp`, then `/mcp` → approve in the browser.
- It produces images, not scripted UI. It uses ForgeGUI credits; generation needs the Starter plan or higher. It can publish to Roblox with a personal API key.
- Plugin alternative: `claude plugin marketplace add elwinhe/forgegui-plugins` → `claude plugin install mcp@forgegui` (it was on a staging endpoint).
- https://forgegui.net/forgegui-mcp · tutorial https://www.youtube.com/watch?v=rMKNtIL2fhk

## Figma MCP
- Remote: `claude mcp add --transport http figma https://mcp.figma.com/mcp`
- Desktop: Figma app → Dev Mode → Inspect → Enable desktop MCP server → `http://127.0.0.1:3845/mcp`
- Needs a paid Dev / Full seat (the free Starter plan allows about 6 calls a month).

### Figma to Roblox
- **figma-to-roblox-ui** skill: `git clone https://github.com/xtrair/figma-to-roblox-ui ~/.claude/skills/figma-to-roblox-ui`. Scale-only GUI import with a Luau validator. It leaves out animations and button logic; script those per `ui-motion.md`.
- Alternative bridge: https://github.com/RaRdq/FigmaRobloxForge

## Blender MCP
https://github.com/ahujasid/blender-mcp. Claude drives Blender to build low-poly models; export GLB / FBX → Studio import. Demos: @pixelrouter, @rileybrown.ai, @robonuggets, @vos_studiostt.

## Ropilot
An AI Studio plugin used in the @ropilot__ai showcases (Glitching Entity, fishing reveal).

## Obsidian
Optional (not used by this repo; skills replace it):
- @lihfolk's free 40GB+ Roblox dev vault (link in his Discord / bio): https://www.tiktok.com/@lihfolk/video/7701541453754862861 and "connect Obi vault + Graphify" https://www.tiktok.com/@lihfolk/video/7695815632251538702
- Claude Code + Obsidian memory: https://github.com/AgriciDaniel/claude-obsidian · https://github.com/eugeniughelbur/obsidian-second-brain
- Roblox Dev Mastery Vault notes: https://github.com/romangigliotti123-arch/vault-notes-01 (kit linked in `third_party/linked/`)
