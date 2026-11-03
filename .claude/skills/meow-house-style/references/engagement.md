# Engagement: why players WANT to keep playing

Apply on every game, together with `game-loop.md`. Goal: "just one more" without tricking anyone.

## The 8 hooks that create "one more"
1. **Variable rewards (the strongest hook).** Fixed rewards get boring; unpredictable ones don't. Every egg, chest, catch, crit and drop rolls rarity. Most results are small, but a big one *could* come any time. Make rare hits loud: rarity ladders, cutscenes (`cutscenes.md`), server-wide announcements ("PLAYER hatched a SECRET Phoenix!").
2. **Goal gradient: always almost there.** Players speed up as a goal gets close. Show 3+ progress bars at once (next upgrade, next zone, next pass tier, index %, rebirth). Make sure one of them is ≥80% full at almost every moment.
3. **Unfinished business.** Leave something open on every exit: an egg mid-hatch, a playtime gift at 1:30, a pass tier 90% done, a quest at 4/5. Show it on the exit / pause screen.
4. **Head start (endowed progress).** Start bars partly filled (the first pass tier is free on join, the daily streak starts at "Day 1 ✓", the index shows 1/52 straight away). A started bar is more motivating than an empty one.
5. **Escalation and big numbers.** Each zone multiplies income 5–10x; numbers climb K → M → B → T → Qd → Qn. Power fantasy: a rebirth shows "x35 → x36 forever".
6. **Collection (gotta catch them all).** Index with "???" silhouettes, Found 5/52, set bonuses, shiny / golden / rainbow / huge variants of every pet (each variant counts again). Empty slots pull harder than full ones.
7. **Social proof and showing off.** This is the kid-to-best-friend test. Big pets follow you around, auras, titles, leaderboards in the lobby, trade plaza, server luck boosts that make one purchase help the whole server (cheers + "thanks!" moments), and gifting.
8. **Reasons to come back.** Daily streak (Day 7 big prize), playtime gifts, offline earnings ("While you were away: +$2.4M", claim x2 with a pass), timed events with countdowns, Season Pass with Season 1 exclusives, weekly quests, a new update every 1–2 weeks with a "NEW!" zone or egg.

## Session shape
| Moment | Must happen |
|---|---|
| First 10 s | Doing the core action already; first "+$" pop |
| First 60 s | First upgrade bought; first big visual payoff |
| First 3 min | First pet hatched (starter egg is free and guaranteed decent) |
| First 5 min | Daily reward + playtime gift + Season Pass tier 1 introduced |
| First 10 min | New zone unlocked with a cutscene; a rare-ish drop |
| Every 30–60 s | Something new to buy or claim |
| Every 5–10 min | A "wow" moment (rare hatch, zone, boss, event) |
| On leave | Something unfinished shown + "come back for Day 2 reward" |

## Rhythm: tension and release
Grind (short) → anticipation (egg shaking, rarity ladder climbing) → payoff (flash, cutscene, number jump) → new goal revealed straight away. Never end a payoff without showing the next goal.

## Retention targets
- D1 retention ≥ 30%, D7 ≥ 10% are healthy for simulators. Instrument with the `roblox-analytics-instrumentation` skill: first action time, first upgrade time, first hatch, session length, quit points.
- Find quit points, then put a reward or a "wow" 30 s before each.

## Hard lines (Roblox rules: breaking these gets games moderated, and the audience is mostly kids)
- **Paid random items** (eggs / crates / spins bought with Robux or Robux-bought currency): show **every outcome with % odds totaling 100%** before purchase, via a labeled "Info / Details" button (not just an (i) icon); explain luck boosts and pity numerically and update the shown odds while they're active; check `PolicyService:GetPolicyInfoForPlayerAsync()`, and if `ArePaidRandomItemsRestricted` is true, use an allowed alternative (free path, direct sale, or hide / block); block trading of paid items when `IsPaidItemTradingAllowed` is false.
- **Timers and scarcity must be real.** If it says "ends in 2 days", it ends. Season 1 exclusives really never return.
- **No fake or rigged odds, no fake "near misses" on paid rolls, no tricking players into tapping Buy** (no Buy button where Close usually is; no instant re-prompt after they decline).
- **Keep free play fun.** Pay-to-win should speed you up and show off, but a free player should still progress and feel good. That also keeps the server full for payers to show off to.

Sources: [Compulsion loop (Wikipedia)](https://en.wikipedia.org/wiki/Compulsion_loop) · [Variable reward loop](https://screenwiseapp.com/guides/gaming-reward-systems-and-the-variable-reward-loop) · [Why some Roblox games keep players for years](https://urgametips.com/why-some-roblox-games-keep-players-for-years/) · [Roblox paid random items policy](https://create.roblox.com/docs/production/monetization/paid-random-items)
