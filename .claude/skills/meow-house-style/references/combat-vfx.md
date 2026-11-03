# Combat and VFX

## AI weapon workflow (@lihfolk)
The prompt he uses:
> generate me a roblox ready sword with vfx, 4 attack animations, 1 basic slash, 2 advanced attacks, 1 super (ultimate move) attack, use heavy vfx, very flashy, vibrant and animated movements. make player animations fluid and agile, quick with the same color lightning movement as the weapon. Make the weapon insanely cool, a blade with animation people have not seen before, ensure all VFX settings are good to go through roblox, I want this to import 1:1.

- **Result:** a teal lightning katana with a soulfire edge, a gold guard and a lantern charm; a helix-particle spear.
- **Preview:** a black stage with a neon pink grid floor and a teal ring; the ultimate brings a huge teal light pillar.
- **Full reaction set:** Dash, Block, Dodge, Stagger, Stun, Knock-up, Caught in ult, Death.
- **Delivery:** an installer `.lua` file. Animations must still be uploaded as Roblox assets, so use KeyframeSequences + AnimationIds.

## Ability kit (@ropilot__ai "Glitching Entity")
In order: a magenta glitch beam that RGB-splits the target (tinted offset clones) → pixel dissolve into pink cubes → a chain bind under a floating dark cube → a wireframe cube prison (purple / cyan, glowing corner nodes, floating "error window" panes) → a white neon frame collapse finisher.
- One theme only, a limited palette (magenta / purple / cyan + white impact frames) and moves that escalate.
- **Showcase format:** a gray baseplate and a row of dummies.

## Rules
- One element / theme per kit; the player's animation colour matches the weapon.
- Poke → control → big → ultimate (escalation).
- Every hit: flash + shake + particles + damage number. See `cutscenes.md` (Rules).
- Server-authoritative hitboxes and cooldowns; VFX on clients.
