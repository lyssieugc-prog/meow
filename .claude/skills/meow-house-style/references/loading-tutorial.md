# Loading Screen and Tutorial (every game, no exceptions)

> The owner's rule: **every game ALWAYS has (1) a cool, eye-catching, heavily animated, cutscene-style loading screen and (2) a tutorial on how to play that new players MUST complete**, using spotlights to highlight things. Both are part of the first build of any game, never "later".
>
> This overrides generic advice in other skills (for example `roblox-game-ideas` "avoid forced cutscenes / long tutorials"). The rule is to keep them **fast and fun**, not to drop them.

## 1. Loading screen

### Look and motion
- **It's a mini cutscene, not a progress bar on black.** Follow `cutscenes.md`: anticipation → impact → hold.
- **Beat sheet (first join, about 5–8 s, or as long as loading really takes):**
  1. **Black → fade-in** to a themed animated background: parallax layers, drifting particles, sunburst rotating, the game's hero creature / mascot idling.
  2. **Build-up:** the logo pieces fly in one by one with overshoot, glow charges behind them, light camera sway.
  3. **Impact:** a white flash + shake + particle burst as the **title logo slams** in (tilted, thick dark stroke, Fredoka One).
  4. **Hold and loop** while loading finishes: the logo breathes / bobs, the mascot does idle tricks, sparkles orbit, rotating **tips / "did you know"** cards slide in with a stagger.
  5. **Exit:** a themed **transition wipe** (clouds, slime, sparkles, lightning, whatever fits the theme) reveals the game. Never a plain cut.
- **Progress bar:** a chunky themed bar (pill shape, gloss stripe, lip) that fills with a bounce and a shine sweep, plus a percentage or "Loading pets… / Loading world…" stage text. The bar never sits still: if loading stalls, it keeps shimmering.
- **Themed to the game** and **reskinned for events**, same as the UI frame (`SKILL.md` rule 4).
- **Sound:** a whoosh on fly-ins, a boom on the logo slam, soft looping music (respect the mute setting).
- **Returning players:** a shorter version (about 2–3 s) with a **Skip** button that appears once the game is actually loaded. First-time players see the full version.

### Tech
- One client script in `ReplicatedFirst` (`roblox-architecture`): build the GUI in code or from a tiny pre-made ScreenGui, call `ReplicatedFirst:RemoveDefaultLoadingScreen()` straight away.
- `ContentProvider:PreloadAsync` only the loading-screen art, key menu art and the spawn area (`roblox-performance` fix catalog). Keep `ReplicatedFirst` lean or the loading screen itself loads slowly.
- Wait for `game:IsLoaded()` plus the player's data being ready from the server, with a **timeout** (about 20 s), so it can never hang forever. Don't wait on streamed map geometry.
- Motion with TweenService (Back / Elastic easing for pops, Quad / Sine for drifts); keep particle counts mobile-safe.
- Test at phone size in Device Simulator: logo and bar must stay inside the safe area.

## 2. Mandatory tutorial

### Rules
1. **New players must complete it** before playing freely. No skip button the first time. The server stores `TutorialDone` (and the current step) in the player's saved data, so it runs once per account and resumes at the same step if they leave mid-way.
2. **Learn by doing, not reading.** Each step is one action the player performs ("Click the coin!", "Hatch your first egg!", "Equip your pet!"). Max about 8 words of text per step. Total about 1–3 minutes.
3. **Cover the whole core loop once**, end to end: the main action → earning currency → spending / upgrading → the next goal (e.g. unlock the next zone). Include where the shop and the 2x passes are, but **no purchase prompts during the tutorial**.
4. **Spotlight every target:**
   - **UI targets:** dim the whole screen (black at about 60–70 % transparency) with a **cut-out hole** around the target button. Build the dim from 4 frames around the hole, or a 9-slice image with a soft round hole, so the hole can tween to each new target. Add a **pulsing ring** around the hole and a **bouncing pointing hand / arrow**.
   - **World targets** (NPCs, eggs, pads, zones): a `Highlight` (glowing outline, pulsing) on the object, a floating bouncing arrow above it, and a **Beam / glowing trail path** on the ground from the player to it. Optionally a short camera pan to show it first.
   - Only the highlighted thing can be clicked during that step (block other input with the dim layer).
5. **Every step is animated:** the spotlight hole glides between targets, the instruction card pops in with overshoot, and a **mascot / guide character** reacts (talk-bounce, points, cheers).
6. **Celebrate each step:** a check stamp + sparkle burst + "ding". **Finish with a cutscene-style reward moment** (`cutscenes.md`): blackout → flash → big "TUTORIAL COMPLETE!" title slam → a free reward (pet, coins or boost) with a chest / egg reveal.
7. **Replay any time:** a "How to Play" / "Tutorial" button in Settings replays it (optional the second time, with a skip).

### Tech
- The **server owns tutorial state** (`roblox-server-authority`, `roblox-security`): the client asks to advance; the server checks that the step's action really happened (the coin was collected, the egg hatched) before saving the next step and granting the final reward. The completion reward can only be claimed once.
- One `TutorialController` on the client (spotlight, arrows, cards, mascot) driven by a shared, typed list of steps (`id`, target, text, completion check). Adding a step = adding one entry.
- Log each step with `AnalyticsService:LogOnboardingFunnelStepEvent` so drop-off shows in the Creator Dashboard (`roblox-analytics-instrumentation`).
- Mobile-first: big touch targets, spotlight holes sized for thumbs, text inside safe areas, works with gamepad too (`roblox-input`).

## Checklist before showing the owner
- [ ] Loading screen plays as a mini cutscene with a logo slam, looping idle and a themed wipe out.
- [ ] It never hangs (timeout) and is short with a skip for returning players.
- [ ] Tutorial is forced for new players, saved server-side, resumes mid-way.
- [ ] Every step spotlights its target (UI cut-out or world Highlight + arrow + path).
- [ ] Ends with a cutscene reward moment; replayable from Settings.
- [ ] Checked at phone size.
