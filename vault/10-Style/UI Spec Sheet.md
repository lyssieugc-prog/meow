---
tags: [style, ui, spec]
---
# UI Spec Sheet

Measured from the owner's GOOD references ([[GOOD - UI]]). The hex values were sampled from phone photos of screens, so treat them as close starting points and adjust by eye.

## Fonts
| Use | Font | Roblox |
|---|---|---|
| Titles, nav buttons, prices, labels, numbers | **Fredoka One** (high confidence) | `Enum.Font.FredokaOne` |
| ALL-CAPS banners (CASH SHARK, PHOENIX GUARANTEED, SAVE 9%, EXCLUSIVE, CLAIM +3% LUCK!) | Heavy geometric sans: Builder Sans Heavy / Gotham Black (medium confidence) | `Font.new("rbxasset://fonts/families/BuilderSans.json", Enum.FontWeight.Heavy)` or `Enum.Font.GothamBlack` |
| Trailer / teaser titles ("ICE GOLEM / COMING SOON") | Heavy condensed caps (Anton / Bebas Neue style) | for videos and thumbnails |

Avoid Luckiest Guy and Cartoon here: they are a different look.

## Text treatment
- White fill. `UIStroke` (ApplyStrokeMode Contextual) in about `#1A1A1A`, 2–3 px for labels and 4–5 px for titles at 1080p.
- Titles also get a dark drop-shadow copy offset 3–4 px down.
- Accents: gold `#FFD93B` for headlines and key numbers; lime `#A8F03C` for good-deal text; yellow for odds %.
- `TextScaled` plus `UITextSizeConstraint`. Never thin or small.

## Palette (approximate)
| Use | Main | Darker edge / lip |
|---|---|---|
| Title bar / close | `#E13A2A` | `#B6261D` |
| Price button | `#7ADE43` | `#57AC35` |
| Blue bar / ODDS | `#2590D1` / `#326BE3` | about 25% darker |
| Yellow | `#E2C608` | orange-gold |
| Purple | `#B678EA` | violet |
| Lime title bar | `#78BE0E` | darker green |
| Pink | `#DD70E1` | magenta |
| Panel body | about `#1E1B33`, slightly transparent | — |

## Components
- **Pill button:** `UICorner` about 0.3 of its height, a dark outline (UIStroke 2–3 px), a darker copy 4–6 px lower as a 3D lip, and a white gloss stripe across the top half at about 70% transparency.
- **Card:** saturated fill, `UIGradient`, a faint sunburst image, tiled stud texture at 85–90% transparency, a lighter inner border and a dark outer stroke.
- **Mythic border:** a `UIGradient` rainbow on a `UIStroke`, rotation animated continuously.
- **Title bar icon:** larger than the bar, with a higher ZIndex, overlapping the top-left and tilted.
- **Tags:** rotated -3° to -10°.

## Layout (from [[Tools - MCP#Figma to Roblox]])
Scale-only sizing and position, AnchorPoint 0.5 / 0.5 by default, UIStroke ScaledSize, UICorner in Scale, PNGs only for detailed art.
