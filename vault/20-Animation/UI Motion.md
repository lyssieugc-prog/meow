---
tags: [animation, ui]
---
# UI Motion

Owner's note: "notice how the UI is very animated." **Nothing static.** Every open, close, claim and hover gets motion and juice.

## Reference: @hallodi.roblox ("all UI done with Claude")
- **Windows pop open:** they scale up from the centre with an overshoot bounce over a slowly rotating sunburst.
- **Cards stagger in** one by one. The claimable card glows and bobs.
- **CLAIM:** a white flash on the card → "DAY 1!" burst text → confetti and coins → a shockwave ring → a checkmark stamp → the streak bar fills → the window shrinks and flies away.
- **Live countdown timers** on every locked reward ("Next gift in 2:32", "12:05:07").
- **Screen transitions:** a white flash → a blue fade → a hyperspace star-warp → the new themed scene (cosmic Card Fusion with spinning neon portal rings, sparks, a pulsing FUSE button, a light beam, parallax planets). The reverse warp exits.
- **Errors slam in** as big glowing text ("No duplicates yet - open more crates").
- **Customizers** (club crest: shape / pattern / overlay / emblem / colours) update a live preview.

## Other references
- **@.6ix13:** a cloud-poof particle animation when the shop opens or scrolls.
- **@ekxu2:** section headers **decode in from scrambled text** ("Hallow88L N2S$VV6G6" → "Halloween Exclusive"); falling particles behind the panel.

## Defaults
| Interaction | Motion |
|---|---|
| Open window | Scale 0 → 1.08 → 1, `Enum.EasingStyle.Back` Out, about 0.35 s; fade in the backdrop |
| Close | Scale 1 → 0.9 + fade, about 0.15 s |
| Card entrance | Stagger about 0.05 s apart; slide up 10% + fade |
| Hover | Scale 1.05 + rotate 2–3°, 0.1 s |
| Press | Squash to 0.95, then spring back |
| Claim | Flash → burst text → particles → stamp → counter tick-up → fly-away |
| Currency change | The number counts up; the icon pops; "+$X" floats up |
| Idle | Sunbursts rotate, icons bob, Mythic borders cycle the rainbow, tags wiggle |
| Screen change | A transition wipe (white flash, star warp, portal) |
| Text | Types or decodes in |
