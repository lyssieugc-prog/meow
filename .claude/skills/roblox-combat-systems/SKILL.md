---
name: roblox-combat-systems
description: Design, implement, and review Roblox combat systems. Use for melee, ranged, abilities, cooldowns, hit detection, damage validation, status effects, enemy AI hooks, combat UI, balance knobs, VFX/SFX, and exploit-resistant remotes.
---

# Roblox Combat Systems

## Goal

Make combat readable, responsive, and server-authoritative.

## Authority

- Let the client request intent, animation, aim direction, or target candidate.
- Let the server validate range, cooldown, line of sight, team, state, and damage.
- Never let the client decide final hit, damage, reward, kill credit, or loot.
- Rate-limit combat remotes.
- Use deterministic balance tables shared or mirrored carefully.

## Combat Loop

Design:

- Anticipation, action, impact, recovery.
- Clear cooldowns and ability availability.
- Strong feedback for hit, miss, crit, block, dodge, and death.
- Enemy tells and fair counterplay.
- Early encounters that teach through safe repetition.

## Implementation Areas

- Hitboxes and raycasts.
- Cooldown and stamina/energy systems.
- Damage types and modifiers.
- Status effects with duration, stacking, and cleanup.
- Enemy AI target selection and leash rules.
- Reward grants tied to server-confirmed contribution.

## Workflow

1. Locate combat code with Grep/Glob in the Rojo tree.
2. Read attack, damage, enemy, and reward modules from disk; read remote declarations (`.model.json`) too.
3. Edit the files (`roblox-rojo-workflow`).
4. Playtest attacks with `roblox-studio-testing`: `rbxstudio play start`, `rbxstudio key`, `rbxstudio mouse`, screenshots.
5. Assert damage/health state with `rbxstudio exec` and check `rbxstudio console`.

## Output

Return:

- Combat state flow.
- Server validations.
- Balance knobs.
- Feedback hooks.
- Exploit and regression tests.

## Next

Validate every combat remote and damage path with `roblox-security-economy`, add impact with `roblox-animation-audio-feedback`, and verify with `roblox-playtest-qa` and `roblox-mobile-playtest`.

