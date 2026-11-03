---
name: roblox-code-maintenance-refactor
description: Maintain and refactor Roblox Luau code safely. Use for technical debt, dead code removal, module boundaries, naming cleanup, duplicate logic, dependency review, regression prevention, and long-term codebase health.
---

# Roblox Code Maintenance Refactor

## Goal

Improve code health without changing player-facing behavior accidentally.

## Refactor Rules

- Read before editing.
- Keep changes scoped.
- Preserve remote contracts unless intentionally migrating.
- Avoid mixing behavior changes with cleanup.
- Playtest after meaningful refactors.
- Check console output.

## Targets

- Duplicated constants and reward formulas.
- Large scripts with mixed responsibilities.
- Client/server boundary confusion.
- Dead remotes, stale UI paths, unused modules.
- Repeated event connection patterns.
- Unclear names around valuable state.
- Magic numbers that should be balance tables.

## Finding Deepening Opportunities

Beyond obvious cleanup, hunt for **shallow modules** to deepen — where the interface is nearly as complex as the implementation, so callers carry complexity that should be hidden.

- Where does understanding one concept require bouncing between many tiny scripts?
- Where were pure helpers extracted only for tidiness, but the real bugs live in how they're called (no locality)?
- Where do client and server share a responsibility that leaks across the remote boundary?

Apply the **deletion test** to anything you suspect is shallow: if deleting it just moves complexity around, inline it; if deleting it would scatter the same complexity across many call sites, keep it and give it a small, stable interface named from the game's vocabulary. Present each opportunity as: files involved · the friction today · the proposed change · the benefit (locality + testability). Don't propose new indirection for a single implementation. See `roblox-luau-architecture` for the deep-module model.

## Workflow

1. Grep the Rojo tree for duplication, old names, and call sites.
2. Read all affected files from disk.
3. Make small edits in the files (`roblox-rojo-workflow`); run `selene` / `luau-lsp analyze` if configured.
4. Verify the relevant loop with `roblox-studio-testing` (`rbxstudio play start`, then `rbxstudio console`).
5. Commit each behavior-preserving step separately.

## Output

Return:

- Refactor goal.
- Behavior preserved.
- Scripts changed.
- Verification performed.
- Follow-up debt items.

## Next

Verify no behavior changed with `roblox-playtest-qa`. For larger structural changes, design them in `roblox-luau-architecture` and slice them with `roblox-to-issues` so they land one safe step at a time.

