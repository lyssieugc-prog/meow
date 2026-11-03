---
description: Generate a complete, secure client/server gameplay system for Roblox — shared types, validated server module, client controller, remote registration, and proof it runs — in one pass. Use when adding a feature like a shop, inventory, combat ability, quest, or any system that spans client and server.
---

# /roblox-dev:new-system — Generate a Complete System

Generate a full vertical slice of the gameplay system named in `$ARGUMENTS`
(e.g. `/roblox-dev:new-system DailyRewards`). A "system" is never one file: it is
the four pieces below, generated together so nothing is left insecure or untyped,
and then run.

## Before generating

1. Read the project's existing structure (see `roblox-architecture`: `src/shared`,
   `src/server/Systems`, `src/client/Controllers`, the remotes list, `Types.luau`,
   the bootstrap). **Match existing conventions:** if the project has a `Net`
   wrapper, a signal library, or a data layer (e.g. ProfileStore), use them rather
   than inventing parallel ones.
2. Check how the project moves characters and takes input. Read
   `Workspace.AuthorityMode` and whether the Character Controller Library or the
   Input Action System is on, because they change the code you write:
   - Server Authority on → follow `roblox-server-authority`.
   - The system binds input → follow `roblox-input`.
3. If the rules are ambiguous (costs, cooldowns, limits), ask the one or two
   questions that change the code. Default the rest into tunables:
   `shared/Config.luau` for compile-time values, `ConfigService` (server only) for
   values that must change without a publish.

## The four pieces

### 1. Shared types — added to `Types.luau`

Request/response payload types and any state snapshot type the client renders.

### 2. Server module — `src/server/Systems/<Name>.luau`

- `--!strict`, two-phase shape: `init` wires remotes and state, `start` runs loops.
  Follow the project's bootstrap; the scaffold's requires both functions to exist.
- Owns ALL state for this system in a server-side table (keyed by Player, cleaned
  on `PlayerRemoving`; see `roblox-performance`).
- Every remote handler: rate limit → type validation → range validation →
  permission/plausibility validation → mutate → replicate the result (the ladder in
  `roblox-security`).
- Persistent state goes through the project's data layer, never direct DataStore
  calls inline (see `roblox-datastores`).
- Exposes a clean module API (`<Name>.grant(player, ...)`) so cheat commands and
  other systems reuse the same code paths (see `roblox-testing`).

### 3. Client controller — `src/client/Controllers/<Name>UI.luau`

- Renders state received from the server; requests actions; **computes nothing
  authoritative**.
- Handles the late-join case: render from a full snapshot on join, then apply
  incremental updates. Replication order between a property change and a remote
  isn't guaranteed, so don't assume one arrived before the other.
- Optimistic UI is allowed for feel but must reconcile with the server result
  (revert on rejection).
- Tagged world objects: under streaming, `CollectionService` signals also fire on
  stream in/out, so `attach` and `detach` must both be safe to run repeatedly.

### 4. Registration

- Add the module to the explicit bootstrap array on each side, in the right order.
- Add remote names to the project's remote list/creation point.
- Add cheat-remote entries for QA (grant/reset/trigger), gated per `roblox-testing`.
- If the system has pure logic (curves, drop tables, price math), extract it to
  `shared/Util/<Name>Logic.luau` and add a `.spec.luau` with boundary tests.

## Prove it runs

A system isn't done when the files exist. In this order:

1. Static checks, as in `setup`: `stylua --check src`, `selene src`, `luau-lsp analyze`
   with the sourcemap and definitions.
2. The evidence loop in `roblox-studio-workflow`: sync → play → exercise the real path
   (the remote, not only the module) → read the console → capture → stop.
3. Try one rejected request too (bad type, out of range, too fast) and confirm the
   server refuses it with no state change. That is what backs the security summary.

If Studio isn't reachable, say that the runtime path is untested.

## Output discipline

- Deliver complete files, not fragments, for anything new; targeted edits for
  existing files (remotes list, `Types.luau`, bootstrap).
- After generating, summarize the security posture in 2–3 bullets: what the server
  validates, what an exploiter could try, and why it fails. Say which of those you
  actually exercised.
- Flag any piece intentionally left simple (e.g. "no purchase receipt idempotency
  yet — needed if you sell this for Robux") so scope cuts are visible, not silent.

## Sources

- https://create.roblox.com/docs/projects/client-server
- https://create.roblox.com/docs/scripting/locations
- https://create.roblox.com/docs/scripting/attributes (replication order)
- https://create.roblox.com/docs/production/configs
- https://create.roblox.com/docs/workspace/streaming/techniques

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
