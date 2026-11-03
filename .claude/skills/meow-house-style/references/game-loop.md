# Game loop + constant upgrades (the default for every game)

**Rule:** the player is never more than about **30–60 s from their next upgrade**. Four loops are stacked inside each other.

## 1. Core loop (seconds)
Do the action (click / swing / mine / fish / ride / collect) → currency right away → the number goes up.
Every action gets feedback: "+$120" floats up, the icon pops, the counter ticks up, hit flash + particles + sound.

## 2. Upgrade loop (30 s – 5 min)
Spend → stronger → earn faster → bigger upgrades. Run several tracks at once so there is always something cheap:
| Track | Examples | Cadence |
|---|---|---|
| Tool / weapon | Wood → Stone → Emerald → Magma → Dark Matter (x2 → x1,250) | 1–3 min |
| Stats | Power, Speed, Luck, Backpack, Pets equipped | 30–60 s |
| Pets | Eggs Common → Secret, multipliers | every hatch |
| Zones | Forest → Coral Reef → Cosmic; each pays 5–10x more | 10–20 min |
| Potions / boosts | 2x Luck / 2x Cash, 15 min | any time |

Math:
- Each upgrade costs **1.15–1.5x** the previous one, and the reward grows slightly faster than the cost.
- A new zone multiplies income 5–10x, so old upgrades suddenly feel cheap and the player buys in bursts.
- Use big-number suffixes: K, M, B, T, Qd, Qn.

## 3. Rebirth / prestige (20–60 min)
- Reset cash and upgrades for a permanent multiplier. Always show the payoff first: "REBIRTH NOW! x35 → x36 Power, x2 Coins".
- Rebirth tokens buy their own upgrade tree (permanent luck, auto-collect, pet slots).
- After about 10 rebirths, unlock Ascend / Super Rebirth.

## 4. Return loop (daily / weekly / seasonal)
- Daily rewards on a 7-day streak with an oversized Day 7.
- Playtime gifts on live timers ("Next gift in 2:32").
- Index / collection rewards ("Found 5/52 → CLAIM +3% LUCK!").
- Events with limited eggs, an event shop and countdowns.
- Daily and weekly quests.
- **The Season Pass** (below).

## Season Pass (ALWAYS: every game ships with "Season 1")
A battle-pass-style **"<Game-themed name> Pass", Season 1**, themed to the game (a mining game → "Miner's Pass"; a dragon game → "Dragon Rider Pass"; a sky / jet game → "Pilot Pass"). Frame, icons and rewards all match the theme.

**How it works**
- 30–50 tiers. Players earn **Pass XP** by playing: quests, playtime, hatches, rebirths and the core action itself.
- **Two reward tracks per tier:** the free track (everyone) and the **Premium track** (Robux). Premium shows its locked rewards with a padlock and a glow, so players see what they're missing.
- Rewards are themed to the game, for example:
  - Potions (2x Luck, 2x Cash, 2x XP)
  - Money bags / cash crates
  - Eggs (incl. a pass-exclusive egg)
  - Boost timers
  - Rebirth tokens
  - Pet slots
  - Titles, auras, trails
  - An exclusive Season 1 pet / tool at the final tier. Make it the best-looking item in the game, with its own reveal cutscene.
- **Season length:** 4–6 weeks, with a big live countdown ("Season 1 ends in 21d 04h"). Season 2 brings a new theme and new exclusives. Season 1 exclusives never return; say so on the pass.

**Monetization**
- **Premium Pass** (gamepass per season, or a Developer Product).
- **Premium+:** Premium plus 10–25 tiers skipped instantly.
- **Tier skip** (Developer Product, buy repeatedly).
- A gift button to buy the pass for a friend.

**UI / motion** (follow `ui-good.md`, `ui-motion.md`)
- A horizontal scrolling track with free rewards on the bottom row and Premium on the top row (gold / rainbow frame).
- The current tier glows and bobs. The XP bar fills with particles. On tier-up: flash → "TIER 12!" burst → rewards fly into the inventory icon.
- A claimable reward pulses; a "CLAIM ALL" button.
- A HUD button with a red notification dot when rewards are ready, and the season countdown on the pass header.

**Server rules:** XP and claims are server-side; save tier + claimed flags per season key in the DataStore (`Pass_S1`); `ProcessReceipt` grants Premium / skips idempotently.

## Robux on every loop
- **Core:** the 2x → 100x upgrading multiplier, Auto-click / Auto-collect.
- **Upgrades:** cash packs, Luck boosts, Triple Hatch.
- **Rebirth:** Auto Rebirth, 2x Rebirth tokens.
- **Return:** Season Pass Premium + tier skips, VIP daily bonus, first-session Starter Pack, event bundles.

## Example: first 10 minutes (mining sim)
- **0:00:** wood pickaxe, +$5 a swing.
- **0:30:** stone pickaxe (x2).
- **1–3 min:** backpack and speed upgrades, a free starter egg.
- **4:00:** daily reward. **5:00:** playtime gift. Season Pass tier 1 is earned and the pass is introduced.
- **6:00:** Zone 2 unlock cutscene, income x5.
- **8:00:** Rare hatch with the rarity cutscene.
- **10:00:** the shop opens on "2x Cash".
- **About 25 min:** first rebirth.

Every unlock, zone, rare hatch and tier-up gets a celebration or cutscene, and progress bars to the next goal are always visible.
