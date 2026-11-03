---
name: roblox-luau-architecture
description: Design, inspect, and refactor Roblox Luau game architecture. Use for ModuleScript/service boundaries, client-server flow, RemoteEvents/RemoteFunctions, ReplicatedStorage organization, reusable systems, script placement, and Rojo project layout.
---

# Roblox Luau Architecture

## Goal

Keep Roblox game code modular, server-authoritative, and easy to extend. Code lives in the Rojo tree on disk; read the existing files before changing architecture.

## Workflow

1. Read `default.project.json` to see how the Rojo tree maps to services (`roblox-rojo-workflow`).
2. Locate code with Grep/Glob in the Rojo tree.
3. Read relevant files from disk before editing. Never pull scripts from Studio.
4. Edit the files; run `selene` / `luau-lsp analyze` if configured.
5. Playtest with `roblox-studio-testing` and check `rbxstudio console` for regressions.

## Placement Rules

- Put authoritative game logic in `ServerScriptService` or server-owned modules.
- Put shared constants, type definitions, remotes, and pure shared helpers in `ReplicatedStorage`.
- Put player input, camera, local-only UI, and visual prediction in client scripts.
- Keep workspace scripts rare and intentional; prefer service-style scripts that manage tagged instances or configured folders.
- Avoid scattering duplicated constants across client and server.

## Remote Boundaries

- Treat every client remote payload as untrusted.
- Prefer narrow remotes with explicit action names and typed payload checks.
- Rate-limit player-triggered remotes that can affect economy, inventory, combat, or placement.
- Use RemoteFunctions sparingly; avoid long-yielding server calls from the client.
- Never let the client decide currency, inventory, ownership, damage, quest completion, or purchase grants.

## Module Pattern

Use modules for:

- Services with `Init` and `Start` style lifecycle.
- Data access wrappers.
- Economy calculations.
- Reward tables and balancing constants.
- UI controllers on the client.
- Shared utility functions that are deterministic and side-effect-light.

Keep modules small enough that another system can depend on them without importing unrelated behavior.

## Prefer Deep Modules

Aim for **deep** modules: a lot of behavior behind a small, stable interface. A **shallow** module — where the interface is nearly as complex as its implementation — earns its keep only if it hides real complexity.

- **Interface** = everything a caller must know: the function signature *plus* invariants, ordering, error modes, and which side (client/server) may call it. In Roblox, a remote's contract is part of its interface.
- **Deletion test:** imagine deleting a module. If complexity vanishes, it was a pass-through — inline it. If the same complexity reappears across several call sites, it was earning its keep — keep it and deepen it.
- **Locality:** put related behavior and its valuable-state rules in one module so a change, a bug, or the knowledge to reason about it lives in one place — not scattered across a LocalScript, a server Script, and three constants files.
- One adapter satisfying an interface is a hypothetical seam; two is a real one. Don't build indirection for a single implementation.

Good seams get names from the game's own vocabulary (the "coin economy service", the "quest tracker"), not generic ones ("Handler", "Manager").

## Refactor Output

When proposing or making architecture changes, include:

- Current file locations in the Rojo tree and their responsibilities.
- Proposed module/service boundaries.
- Remote contract changes.
- Migration steps.
- Playtest and console checks.

## Next

Slice the architecture work with `roblox-to-issues` so it lands one safe change at a time, and verify each with `roblox-playtest-qa`. For valuable-state modules, review the trust boundary with `roblox-security-economy`; for cleanup of existing debt, use `roblox-code-maintenance-refactor`.

