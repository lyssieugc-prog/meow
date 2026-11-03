---
name: roblox-creator-store-security-audit
description: Audit Roblox Creator Store, Toolbox, inventory, group, universe, package, model, UI, gameplay, plugin-like, and other third-party imported assets for malicious or risky scripts. Use whenever Claude imports, inserts, reviews, or trusts Creator Store assets, especially assets with Scripts, LocalScripts, ModuleScripts, hidden descendants, require calls, HTTP, DataStore, remote creation, obfuscated code, or suspicious names.
---

# Roblox Creator Store Security Audit

## Goal

Treat every third-party Creator Store or Toolbox import as untrusted until inspected. Roblox documentation identifies third-party assets as a security risk because they can contain malicious scripts/backdoors, and Creator Store assets can include scripts. Audit before enabling, shipping, or building on imported behavior.

## Source Baseline

When current policy or platform behavior matters, verify official Roblox docs first:

- `https://create.roblox.com/docs/scripting/security/third-party-vulnerabilities`
- `https://create.roblox.com/docs/scripting/capabilities`
- `https://create.roblox.com/docs/production/creator-store`

Use the audit workflow even if Roblox adds platform protections. Platform safeguards reduce risk; they do not replace reading and understanding imported scripts.

## Import Rule

For third-party model/package/gameplay/UI assets:

1. Insert into a quarantine folder or isolated location.
2. Do not enable HTTP requests or other security-sensitive settings because an imported asset prompts for it.
3. Disable or remove imported scripts unless the game explicitly needs them and they pass review.
4. Prefer visual-only extraction: keep meshes, parts, textures, sounds, particles, and UI art; remove unknown behavior.
5. Record asset name, asset ID, creator/source, inserted path, and audit result.

## Audit Workflow

This is the one place where reading scripts out of Studio is allowed: the code under review is third-party and exists only in the quarantined import. Use the `rbxstudio` CLI (`roblox-studio-testing`), never the Studio MCP tools directly.

1. Confirm the connection with `rbxstudio list` (and `rbxstudio use <id>` if needed).
2. Locate the imported asset with `rbxstudio tree --path <quarantine folder>`.
3. Inspect the imported root with `rbxstudio inspect`.
4. Search the quarantined code with `rbxstudio grep` for suspicious patterns:
   - `require`
   - `getfenv`
   - `setfenv`
   - `loadstring`
   - `HttpService`
   - `GetObjects`
   - `InsertService`
   - `MarketplaceService`
   - `DataStore`
   - `MessagingService`
   - `TeleportService`
   - `RemoteEvent`
   - `RemoteFunction`
   - `rbxassetid`
   - `asset`
   - `string.reverse`
   - `string.char`
   - `byte`
   - `spawn`
   - `coroutine`
   - `while true`
5. Use `rbxstudio read` on every imported `Script`, `LocalScript`, and `ModuleScript` in the quarantine folder.
6. Inspect descendants for scripts hidden under unexpected objects, disabled scripts, deceptive names, or deeply nested containers.
7. Strip scripts that are not needed (targeted `rbxstudio exec` on the quarantine copy only).
8. Playtest in isolation with `rbxstudio play start`, then check `rbxstudio console`.
9. Persist only the vetted result into the Rojo tree (asset ID in config, or a saved `.rbxm`); any script kept from the asset becomes a reviewed `.luau` file under Rojo (`roblox-rojo-workflow`).

## Red Flags

Treat as high risk:

- Obfuscated or unusually complex code.
- Code that fetches or executes external modules by asset ID.
- Prompts to enable `HttpService`, paste command-bar code, or install another asset/plugin.
- Unknown remotes, admin panels, backdoor-looking commands, or hidden GUIs.
- Scripts named like `AntiLag`, `MainModule`, `Loader`, `Handler`, `Update`, `Version`, `HDAdmin`, `Free`, `Fix`, or names unrelated to the asset purpose.
- DataStore writes, purchase handling, teleporting, moderation/admin commands, or player messaging inside a decorative asset.
- Code that reparents itself, clones itself, destroys audit-visible objects, or spreads through services.

## Verdicts

Return one of:

- `Safe to keep visual content only`: scripts removed or unnecessary.
- `Safe with reviewed scripts`: scripts are simple, necessary, and explained.
- `Quarantine`: suspicious but not proven malicious; do not ship.
- `Reject/remove`: malicious, obfuscated, backdoor-like, or unjustified sensitive behavior.

## Output

Report:

- Asset name, ID/source, creator when known, and inserted path.
- Script inventory and suspicious-pattern findings.
- Actions taken: disabled, removed, retained, or unchanged.
- Security-sensitive settings checked.
- Final verdict and follow-up tests.

## Next

Once an asset is cleared, bring it in through `roblox-asset-pipeline` and check its runtime cost with `roblox-performance-optimization`.

