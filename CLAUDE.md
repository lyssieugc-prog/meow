# meow: Roblox game studio repo

The owner pitches Roblox game ideas; **Claude builds them** (code, UI, models, VFX, cutscenes, promo) in the owner's style. The owner is not learning to code. Be direct and show results.

- For any game work, load the **`meow-house-style`** skill. It holds the owner's style, animation, cutscene and monetization rules, with detailed reference files it opens on demand.
- **Animations and cutscenes are mandatory** for every feature.
- **Every game ALWAYS has** a cutscene-style, heavily animated loading screen **and** a tutorial new players must complete, with spotlights on everything it teaches (`meow-house-style/references/loading-tutorial.md`).
- **Gamepasses:** always a 2x → 100x upgrading multiplier plus engaging pay-to-win passes.
- **Tools:** Blender for 3D models; Claude Design for animations, VFX and model design; Figma for UI; research the best MCP for anything else. See the house-style skill.
- **Every game:** the constant-upgrade game loop and a themed "Season 1" pass (see `references/game-loop.md`).
- After each build, run the "show your best friend" review.
- When the owner sends a reference, break it down (including the animation), wait for GOOD / BAD, then record it in the skill's reference files.
- The other 64 skills in `.claude/skills/` cover Roblox / Luau engineering; they load automatically on matching tasks.
- Luau `--!strict`; the server owns all state; validate remotes; save DataStores safely; idempotent `ProcessReceipt`. Commit before big changes.
- No Obsidian. Where a skill (`build-asset`, `mesh-qa`, `vfx-review`, `game-research`) mentions a "vault", "Style Bible" or "Lessons Learned", use the `meow-house-style` reference files instead, and add lessons there.
