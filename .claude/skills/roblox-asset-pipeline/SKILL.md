---
name: roblox-asset-pipeline
description: Plan and execute Roblox asset workflows. Use for Creator Store search, inventory/group/universe assets, inserting assets, generated meshes, generated materials, procedural models, uploaded images, thumbnails, icons, decals, style consistency, import security audits, and asset performance budgets.
---

# Roblox Asset Pipeline

## Goal

Choose assets that fit the game fantasy, load reliably, and stay within mobile performance budgets.

## Selection Order

1. Reuse assets already referenced in the Rojo tree (asset-ID config modules, `.rbxm` files) before sourcing new ones.
2. Before inserting Creator Store or third-party model/package assets, plan a `roblox-creator-store-security-audit` pass.
3. Insert selected Creator Store assets for evaluation with `rbxstudio store '<query>'` (into a quarantine folder Rojo does not manage).
4. Generate custom meshes with `rbxstudio mesh '<prompt>'` only when existing assets do not fit.
5. Generate custom materials with `rbxstudio material '<prompt>'` for unique surface treatments.
6. For other Studio asset tools (procedural models, image upload), use `rbxstudio raw <tool> '<json>'` — run `rbxstudio tools` to see what's available.
7. Every kept asset ends up in the Rojo tree (ID in config or saved `.rbxm`) — see the workflow below.

## Cross-Owner Caution

When an inventory asset belongs to a different owner than the current place, ask for explicit user consent before inserting it. Name the source owner and destination.

## Security Audit Requirement

Creator Store and third-party model/package imports can include scripts. After inserting any third-party model, package, UI, gameplay asset, or plugin-like asset, use `roblox-creator-store-security-audit` before trusting it or leaving its scripts enabled. Prefer visual-only assets when possible. Treat obfuscated code, suspicious `require` calls, dynamic asset loading, HTTP usage, DataStore mutation, remote creation, hidden scripts, and deceptive names as high risk.

## Art Direction

- Keep a consistent silhouette language, scale, color palette, and material style.
- Make important interactables visually louder than background props.
- Design icons and thumbnails for click clarity at small sizes.
- Avoid dark, noisy, or over-detailed first-area assets that obscure gameplay.

## Performance Budgets

- Prefer low-to-moderate poly meshes for mobile.
- Watch texture size, transparency, particles, dynamic lighting, and physics complexity.
- Anchor decorative props unless they must simulate.
- Convert repeated decorative objects into cheaper reusable patterns where possible.
- Run performance review for large asset drops.

## Workflow

1. Search or generate the asset (`rbxstudio store '<query>'`, `rbxstudio mesh '<prompt>'`, `rbxstudio material '<prompt>'`) — this is an evaluation step in Studio.
2. Insert third-party assets only into a controlled quarantine folder that Rojo does not manage.
3. Inspect with `rbxstudio inspect`; run `roblox-creator-store-security-audit` for any third-party model/package/gameplay/UI asset.
4. Screenshot for visual fit; playtest if it affects collision, navigation, UI, or interaction; check `rbxstudio console`.
5. **Persist into Rojo:** record the asset ID in a config ModuleScript / `.model.json`, or have the user save the vetted model as `.rbxm` into the Rojo tree. Delete the quarantine copy. An asset that only exists in Studio is not done.

## Output

Return:

- Asset source or generation method.
- Placement and ownership notes.
- Security audit result for third-party imports.
- Visual-fit notes.
- Performance risks.
- Verification performed.

## Next

Audit any imported third-party asset with `roblox-creator-store-security-audit` before trusting it, and check its cost with `roblox-performance-optimization`.
