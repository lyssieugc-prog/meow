---
name: roblox-animation-audio-feedback
description: Improve Roblox feel through animation, audio, VFX, haptics-like feedback, reward juice, movement feel, combat feedback, UI transitions, emotes, and satisfying loop feedback without hurting mobile performance.
---

# Roblox Animation Audio Feedback

## Goal

Make every core action feel satisfying and understandable.

## Feedback Stack

Use layered feedback:

- Animation: anticipation, impact, recovery.
- Sound: action, hit, reward, failure, purchase, unlock.
- VFX: particles, highlights, trails, popups, camera-safe effects.
- UI: counters tick, bars fill, buttons press, reward deltas appear.
- World response: object changes, enemy reaction, collectible burst.

## Rules

- Feedback must clarify state, not obscure it.
- Keep first-session effects readable and quick.
- Budget particles, transparency, lights, and looping sounds for mobile.
- Use consistent audio language for success, danger, rarity, and progression.
- Make rare rewards feel distinct without requiring long animations.

## Workflow

1. Grep the Rojo tree for animation, sound, reward, and effect hooks; read them from disk.
2. Check `rbxstudio tree` / `rbxstudio inspect` only to confirm runtime instances exist during a test.
3. Pick audio/visual assets via `roblox-asset-pipeline`; record their IDs in a config module.
4. Patch trigger code and effect config in the Rojo files (`roblox-rojo-workflow`).
5. Playtest with `roblox-studio-testing` and capture before/after screenshots.
6. Check `rbxstudio console`.

## Output

Return:

- Feedback moments.
- Asset or effect choices.
- Trigger ownership.
- Mobile performance risks.
- Verification notes.

## Next

Confirm the added juice does not hurt mobile frame rate with `roblox-performance-optimization` and `roblox-mobile-playtest`.

