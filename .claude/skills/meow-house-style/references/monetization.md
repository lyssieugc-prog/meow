# Monetization: standing rules

## Always
1. **A "2x (something)" that upgrades every time the player buys again, all the way to 100x.**
   - Ladder: 2x → 3x → 5x → 10x → 25x → 50x → 100x, applied to the game's main stat (Cash, Power, Speed, Luck, Egg Growth, Mining ...).
   - The shop always shows the **next** step ("Upgrade to 5x Cash!"); earlier steps show as claimed; the price rises each step.
   - A gamepass can only be bought once, so implement the steps as **Developer Products** with the saved level in a DataStore (repeat purchases). Alternative: a chain of gamepasses (2x, 5x, 10x ...) that show on the store page.
2. **Engaging pay-to-win passes** that clearly help payers win:
   - Auto Collect / Auto Hatch / Auto Rebirth / Auto Miner (they keep grinding while the player is away)
   - +Pets Equipped, +Inventory, Triple / Octo Hatch
   - 2x Luck (stacks with server luck), Super Luck for Secret / Mythic
   - VIP: an exclusive pet / aura / weapon stronger than anything from eggs, a VIP area with better drops
   - x2 Walk Speed, Instant Hatch, Skip Cooldowns
   - A Starter Pack in the first session only, with a countdown
   - Timed boosts (15 min / 1 h, "1x > 2x") stacking on the passes
   - Server-wide luck boosts (social: everyone in the server benefits)
   - Lucky blocks / crates with 1 / 3 / 10 bundles (SAVE %) and pity
3. A **gift button** next to every price.
4. **Limited-time offers** with live countdowns, exclusive event bundles, strikethrough old prices.

## Roblox rules to respect
- Paid random items (crates, eggs bought with Robux) must **show their odds**.
- Use `MarketplaceService.ProcessReceipt` correctly: idempotent, save before returning `PurchaseGranted`.
