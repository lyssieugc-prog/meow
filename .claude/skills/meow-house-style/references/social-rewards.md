# Group + Like rewards (every game, no exceptions)

> The owner's rule: **every game ALWAYS has a "JOIN THE GROUP FOR ___ PRIZE!" reward and a "LIKE THE GAME FOR ___!" reward**, shown big and animated, and players **actually get** the prize.

## 1. Join the group (detected per player, fully automatic)

**What the player sees**
- A HUD button / floating board: **"JOIN THE GROUP FOR A FREE [prize]!"** with the prize's 3D render spinning, a pulsing glow and a bouncing arrow. House UI look (`ui-good.md`, `ui-motion.md`).
- Press it → the official Roblox **join-group prompt** opens right in the game.
- Joined → the button flips to **"CLAIM!"** → claim cutscene: flash, burst, the prize flies into the inventory, "+1 [prize]" stamp.
- A small "Already in the group? Tap CLAIM" so existing members get it too.

**Prize ideas** (pick per game, make it feel exclusive): a Group-only pet / brainrot / tool, a permanent +10 % cash or luck boost, a "Group" tag over the head, a group-only chest that refills daily.

**Tech**
- Client: `GroupService:PromptJoinAsync(groupId)` (in a `pcall`); it returns `Enum.GroupMembershipStatus` (`Joined`, `AlreadyMember`, `JoinRequestPending`, …). If the group needs approval, tell the player "Request sent! Claim once you're accepted."
- **The server decides:** the client only asks "claim", the server checks `player:IsInGroupAsync(groupId)` (in a `pcall`; fail closed), then grants the prize **once**, saved in the player's data (`GroupRewardClaimed = true`). Never trust the client saying it joined.
- Results are cached for a while, so re-check on the claim press, not just on join.
- The group ID lives in one config value. If the owner hasn't made the group yet, ask for its link before publishing.

## 2. Like the game (a server-wide like goal, NOT a per-player like check)

**Roblox cannot tell a game which player liked it.** There's no API for that, and the website endpoints can't be called from game scripts. Rewarding someone *for* liking also risks Roblox's rules against incentivized votes. So games that look like they "detect" likes actually use a **like goal**, and so do we:

**What the player sees**
- A big animated board near spawn: **"👍 LIKE THE GAME! 10,000 LIKES = FREE [prize] FOR EVERYONE!"** with a live progress bar (e.g. 8,412 / 10,000) and a countdown-style fill animation.
- When the goal is reached: a **server-wide celebration cutscene** (confetti, flash, title slam "GOAL REACHED!"), and **every player** (old and new) can claim the prize from the board. Then a new, bigger goal appears (25K, 50K, 100K…), each with a better prize.
- Wording stays friendly ("Like the game to help unlock…"), never "like or you don't get it".

**Prize ideas:** a free pet / brainrot, a 30-minute server luck boost, a free spin / lucky block, a "Like Goal" exclusive badge item.

**Tech**
- The like count: Roblox has no in-game API for it, so use the owner's chosen method and check that it's still allowed when building: either the owner (or Claude, on request) updates the reached goal in a config value / Open Cloud, or a trusted proxy reads the public like count. Never send player data to a third-party proxy.
- The server owns the goals: `LikeGoalsReached` is a global value; each player's claimed goals are saved in their data, so each goal's prize is claimed once per player.
- Optional: also offer **"Favorite the game"** with a prompt if Roblox supports it for experiences at build time; if not, just show a friendly "⭐ Favorite to get notified of updates!" without a reward.

## Checklist before showing the owner
- [ ] Group button → real join prompt → server checks membership → prize granted once and saved.
- [ ] Existing members can claim; pending requests get a friendly message.
- [ ] Like-goal board with live progress, celebration when reached, everyone can claim, next goal appears.
- [ ] No reward is given for a single player's like (rule risk); all prizes granted server-side.
- [ ] Both boards use the house UI look and motion.
