# Ragdoll for Eggs: design brief

> Approved by the owner. Build from this file. Follow `CLAUDE.md` and the `meow-house-style` skill (brainrot / simulator style) for everything not spelled out here: the loading screen, the mandatory tutorial, the UI look, animations and cutscenes, the 2x → 100x passes, the Season 1 pass, the group and like-goal rewards, the engagement rules and the "show your best friend" review.

## Player promise
Jump off the edge of a giant mountain, ragdoll down a huge plinko slope, and the further you get, the better the egg. Hatch pets that make you stronger so you get even further next time.

## The map
```
 🏠🏠🏠  THE PEAK (spawn + every player's base)
 ──[ DROP EDGE ]──
   \ • • • •            ZONE 1  GRASSY HILL      (no requirement)
    \ • 🍄 • •          plinko pegs + hazards + egg bins
     ╠══ GATE 1 ══╣     "NEEDS 100 ⚡ FALL POWER"
      \ • • 🍭 •        ZONE 2  CANDY MOUNTAIN
       ╠══ GATE 2 ══╣   "NEEDS 1,000 ⚡"
        \ • 🔥 • •      ZONE 3  LAVA CANYON
         ╠══ GATE 3 ══╣ "NEEDS 25,000 ⚡"
          ...           ZONES 4–6  ICE CLIFFS → SKY ISLANDS → SPACE SLOPE
           ╠══ GATE 6 ══╣ "NEEDS 10M ⚡"
            \  🌀       ZONE 7  SECRET VOID
```
- **The Peak:** spawn, every player's base (incubators, pets, upgrade stations), the Drop Edge, and a board showing the player's odds.
- **One continuous slope.** Every zone is **visible and open to everyone from the start** (like Steal an Egg's maps); the gates decide how far a drop really gets.
- Each zone is a tilted **plinko board**: pegs, mushroom bumpers, spinning bats, trampolines, portals, plus that zone's **hazards**, ending in a **row of egg bins** before its gate.
- **Gates are GLASS WALLS (owner's signature idea, must be built):** between every two zones, a huge glass wall spans the **full width of the slope**, edge to edge, so nothing gets past without breaking it.
  - Thick, slightly tinted glass in the zone's colour (Zone 1→2 green, 2→3 pink candy, 3→4 orange lava, …), a chunky frame, glowing cracks that get bigger near the next zone, and a reflection shine sweeping across it.
  - A **huge sign above each glass wall:** "🔒 NEEDS 1,000 ⚡ FALL POWER", the player's ⚡ with a progress bar, and their **pass chance %**.
  - The **next zone is visible through the glass**, so players see what they're missing.
  - **Smash:** the glass shatters into big flying shards (physics shards + particle burst + glass-break sound + white flash + slow-mo), "BREAKTHROUGH!" slams in, and the ragdoll flies through. The wall **reappears** after a moment (other players see their own gates intact).
  - **Bounce:** the ragdoll hits the glass with a "BONK", a crack spreads, the ragdoll rebounds into that zone's bins, and the sign flashes the gap ("640 / 1,000 ⚡").
- **The map is NOT one long board with bins only at the end.** It's **7 zones stacked down the slope**, each with its own plinko board, its own row of egg bins *just above its glass wall*, its own hazards and its own theme / colours.

| # | Zone | Gate requirement (tune) | Hazards | Eggs |
|---|---|---|---|---|
| 1 | Grassy Hill | none | mud pits, sticky honey | Common – Rare |
| 2 | Candy Mountain | 100 ⚡ | gummy goo, chocolate river | Rare – Epic |
| 3 | Lava Canyon | 1,000 ⚡ | lava pools, geysers shooting you up | Epic – Legendary |
| 4 | Ice Cliffs | 25,000 ⚡ | ice holes, frozen snowman blockers | Legendary |
| 5 | Sky Islands | 500,000 ⚡ | wind gusts off the side, storm clouds | Legendary – Mythic |
| 6 | Space Slope | 2M ⚡ | black holes, meteors | Mythic |
| 7 | Secret Void | 10M ⚡ | glitch holes, void hands | Secret |

## One drop (about 15–25 s)
1. Walk to the Drop Edge and press **JUMP**: slow-motion dive, camera follows from behind.
2. Ragdoll down the board: pegs ("bonk!"), bumpers, bats, trampolines, portals. A **combo meter** counts hits ("x14 BOUNCE COMBO!") and adds bonus eggs at the end.
3. **Glass gates:** with enough ⚡ and a passed roll, you **smash through the glass wall** (shards fly, "BREAKTHROUGH!" slam, slow-mo); otherwise you **bounce off the glass** ("BONK", a crack spreads) and the sign flashes the gap ("640 / 1,000 ⚡").
4. **Wipeouts:** even with enough ⚡ you can get caught by a hazard (cartoon "SPLAT!", stuck in goo, legs kicking): the drop ends in that zone.
5. **Landing:** in the zone where the drop ends, you land in a plinko bin. Middle bins are common, small edge bins are rare, a golden **Jackpot Bin** opens sometimes, and some bins are **BUST** bins (cracked egg + sad trombone).
6. The egg flies to your base. Back at the top in about 2 s. Repeat.

## Results are decided by the server at JUMP (cheat-proof and owner-controlled)
1. The client sends only "jump". The server checks: the player is at the Drop Edge, has no drop in progress, and isn't jumping faster than a drop can finish.
2. The server walks the zones in order. For each zone it rolls the **hazard survival** with the player's pass chance for that zone; if they survive, it checks the **gate** (⚡ ≥ requirement). The first wipeout or the first unbreakable gate ends the drop in that zone.
3. In that zone it rolls the **bin** (egg bins + bust bins) with the zone's odds and the player's luck.
4. The server saves the result (`runId`, zone, outcome, bin, egg) and sends the client the script to play: which gates to smash, where to wipe out or bounce, which bin to land in. The client animates the ragdoll and steers it there (nudged back near the end if physics drifts).
5. After a minimum drop time (about 10 s), or when the player leaves, the server grants the saved egg (or shards for a bust).
- **Bins, gates and hazards have no reward logic.** Flying, teleporting or diving into a bin does nothing; only the saved server result counts.
- **Extra Drops (clones):** each clone gets its own result, rolled the same way.

### Pass chance and wipeouts
- Exactly enough ⚡ for a gate → about **50 %** chance to get through that zone; much more ⚡ → safer, capped at about **95 %** (never 100 %). Tune the curve in settings.
- Upgrades that raise pass chance: **Grip** (goo / honey), **Heat Shield** (lava), **Balance** (ice / wind). **Shield charges** each cancel one wipeout; they're bought **before** a drop and counted in the server roll.
- Shown honestly: the Drop Edge board lists the player's pass chance per zone and bin odds (including bust bins) per zone, updated live for luck, upgrades and events.

### Bust bins and shards
A bust gives **Egg Shards** for that zone; **10 shards = a free egg** from that zone, so every loss is still progress.

### Owner control (one settings module)
- Odds per zone (egg bins, bust bins), the pass-chance curve, the gate requirements, egg rarities and mutation chances.
- Guarantees: the tutorial's first drop lands a decent egg; **pity** (e.g. after 50 drops without a Rare, the next is Rare); the first drop into a newly reached zone is at least Epic.
- Events change the odds for the whole server while active (Jackpot Hour, Server Luck, Low Gravity).
- **Admin command (owner only, logged):** set a player's next drop result, for giveaways, YouTubers and testing.
- **Hard rule:** the odds board and the server read the **same** settings, so the shown odds are always the real odds (Roblox rules + `meow-house-style/references/engagement.md`). Never secretly lower anyone's odds, and never offer a "save" mid-fall (the result is already decided).

## Pets, eggs and progression
- **Hatching** at the base with a reveal cutscene; rarer = longer reveal.
- **Pets** earn cash every second (offline too) and boost ⚡ Fall Power, luck, bounce, magnet range and extra drops.
- **Rarities:** Common, Uncommon, Rare, Epic, Legendary, Mythic, Secret. **Mutations:** Golden, Rainbow, Bouncy, Lava, Galaxy (a Jackpot Bin egg is always mutated).
- **Index** with "???" silhouettes; set bonuses add luck. Secret hatches get a server-wide announcement.
- **⚡ Fall Power** is the main stat (like speed in Steal an Egg). It comes from cash upgrades, pet multipliers, rebirths and passes.
- **Cash upgrades:** Fall Power, Luck, Grip, Heat Shield, Balance, Bounce Power, Air Wiggle (a small left / right nudge, cosmetic within the server-chosen path), Egg Magnet, Extra Drops, pet slots, incubator speed.
- **Rebirth:** reset cash for a permanent ⚡ multiplier and a cooler jump style (cannonball → rocket → superhero dive).

## Social and events
- Everyone drops on the same slope; ragdolls crash into each other (visual only, it doesn't change server results).
- A server board for the furthest zone and the best bin hit; the global furthest-zone leaderboard.
- Events (live admin or bought): Jackpot Hour, Low Gravity, Mega Bumpers, Egg Rain.

## Robux
- **2x → 100x Fall Power** (main seller) and **2x → 100x Cash**: Developer Products that upgrade on every repeat purchase.
- **Server Luck** (better bins and fewer wipeouts for the whole server, buyer announced), **+3 Extra Drops**, **Auto-Drop**, **Shield charges**, **Gate Smash** (break one extra gate on the next drop, still rolled honestly for the zone after), **Jackpot Bin Open** (10 min, whole server), **Instant Hatch**, **+Pet Slots**, **Starter Pack** (first session, countdown).
- **Season 1 "Drop Pass"** with an exclusive Season 1 pet.
- **Join the group for a free Golden Egg** and the **like goal** (`social-rewards.md`).
- Paid random items (eggs bought with Robux, if any) show full odds (`engagement.md`).

## Loading screen (cutscene style)
A ragdoll bounces down a plinko board made of the logo's letters, smashes through a glass gate, lands in a glowing Rainbow bin, and the title slams in with a flash and shake. It loops while loading and exits with a tumbling-pegs wipe.

## Tutorial (mandatory, spotlights)
1. Spotlight the Drop Edge → jump (first drop is guaranteed a decent egg).
2. Spotlight the egg flying home → the hatch button → first pet.
3. Spotlight the Fall Power upgrade → buy it; the gate sign's progress bar moves.
4. Spotlight the odds board ("your pass chance went up!").
5. Spotlight a Shield charge (one free) → the next drop survives a wipeout.
6. Reward cutscene: "TUTORIAL COMPLETE!" + a free egg.

## First 10 minutes
| Time | Moment |
|---|---|
| 0:00 | Loading screen cutscene |
| 0:10–1:30 | Tutorial (first drop, hatch, upgrade, odds board, shield) |
| 3:00 | First gate smashed into Candy Mountain, with a zone reveal cutscene |
| 5:00 | Daily reward, playtime gift, Season Pass tier 1 |
| 7:00 | First Epic, or a dramatic lava wipeout |
| 10:00 | First server event (Jackpot Hour) |
| ~20 min | First rebirth (cannonball jump) |

## Persistence (saved)
Cash, ⚡ and all upgrade levels, pets, eggs, shards per zone, Index, rebirths, 2x → 100x levels, shield charges, pity counters, tutorial progress, claimed rewards (group, like goals, daily, pass tiers), settings. Session-only: the drop in progress (granted on leave).

## Mobile and performance
Big JUMP and hatch buttons, readable gate signs at phone size, ragdoll physics on the client only, simple pet models with LOD, a capped number of visible ragdolls and clones.

## Risks
1. **Ragdoll steering looks fake.** Check: playtest 20 drops; each must look like natural plinko and still land in the server-chosen bin.
2. **Exploits.** Check: hostile tests (spam jump, teleport into bins, fake remotes) give nothing.
3. **Odds board drifts from the real odds.** Check: one shared settings module, plus a test comparing the board to the server's odds.
