---
description: Server-authoritative security for Roblox games. Validating RemoteEvents/RemoteFunctions (NaN, inf, spoofed tables, Instance args), rate limits, ProximityPrompt/ClickDetector/DragDetector abuse, network ownership and movement checks, secrets store, third-party backdoors and sandboxing, secure teleports, purchase receipts, Device Trust, bans and kicks, group and age gates. Use when writing or reviewing any code that touches remotes, player input, currency, damage, inventory, trading, purchases, admin commands, API keys, or Toolbox/third-party assets.
---

# Roblox Security: Never Trust the Client

A client controls its local state and network traffic. It can decompile every replicated script, take network ownership of its character and nearby unanchored parts, trigger Touched and prompt events at any range, and fire any remote with arbitrary arguments. Only the first `Player` argument of a server handler is trustworthy. Security exists **only** on the server.

Maturity labels below: **GA** generally available, **Beta** Studio/client beta, **Pending** shipped in the binary but not enabled per release notes.

## The Iron Rules

1. **The server owns all state that matters.** Currency, inventory, damage, cooldowns, purchases, progression. The client sends input; the server validates, executes, replicates the result.
2. **Every remote argument is hostile until validated.** Includes `nil`, NaN, ±inf, huge strings, invalid UTF-8, tables posing as Instances.
3. **Anything replicated is public.** LocalScripts, `RunContext = Client` Scripts, and **every ModuleScript that replicates, even if never required or disabled**, can be decompiled. Assume remote names, attributes and unreleased content in a replicated container are seen on the day you publish.
4. **Client-side checks are UX, not security.** Never ban on a client-side detection.
5. **Design cheating into uselessness first.** Detection chases exploiters; design removes the prize (no points for spawn kills, mandatory ordered checkpoints, server-computed damage).

Threat-model each feature before coding it: what if a client sends any value here? What if this fires 1,000+ times a second? Can it hurt other players? What is the worst-case server cost? What is the minimum state I must expose?

## Validating Remote Arguments

Order: **rate limit → type/shape → value → permission/context → plausibility**, return early, drop silently (log if useful).

```lua
--!strict
local function onBuyItem(player: Player, itemId: unknown, quantity: unknown)
	if not allow(player, "BuyItem", 3, 5) then return end

	-- TYPE
	if typeof(itemId) ~= "string" or #itemId > 64 then return end
	if typeof(quantity) ~= "number" then return end

	-- VALUE: isfinite BEFORE any comparison; NaN makes every comparison false
	if not math.isfinite(quantity) or quantity % 1 ~= 0 then return end
	if quantity < 1 or quantity > 100 then return end

	-- PERMISSION / CONTEXT: does it exist, can THIS player do it now?
	local item = ItemCatalog[itemId]
	local profile = PlayerData.get(player)
	if not item or not profile then return end

	-- PLAUSIBILITY: price comes from the SERVER catalog
	local cost = item.price * quantity
	if profile.coins < cost then return end

	-- Mutate with no yield between the check and the write (see "Races")
	profile.coins -= cost
	Inventory.grant(player, itemId, quantity)
end
```

### What actually arrives

| Argument | Facts | Do this |
| --- | --- | --- |
| number | NaN and ±inf are `"number"`. NaN fails every comparison, so `x < 0 or x > max` and `x > balance` all pass it, and it poisons whatever it touches. | `math.isfinite(x)` (GA, 701), after the type check. It replaces `x ~= x` plus `math.huge` checks. |
| Vector3 / CFrame | Components can be NaN or inf; owned parts can replicate them too. Checking only `.X`, as the docs' lightning sample does, misses the other axes. | Check every component (below). |
| string | Length is unbounded by you; DataStores only accept valid UTF-8. | Cap length first, then `utf8.len(s) ~= nil`. |
| table | Copied (new identity). Metatables are lost. Function values become `nil`. Non-string keys such as Instances become strings. The docs say not to rely on mixed keys or `nil` indices. A hostile client controls the rest of the shape. | Prefer scalars. Otherwise iterate everything: valid keys, a key-count cap, each value checked. |
| "Instance" | A table with `Name`, `ClassName`, `Parent`, `Price` fields passes `arg.Name` checks. An Instance only the client has (or only the server has) arrives as `nil`. A real replicated Instance can be any one the client knows. | `typeof(x) == "Instance"` plus `IsA` plus `IsDescendantOf` the expected folder, and read data from the server's copy. |
| anything | Indexing before the type check throws in your handler. | Type check first. |

```lua
--!strict
local function isFiniteVector3(v: unknown): boolean
	return typeof(v) == "Vector3" and math.isfinite(v.X) and math.isfinite(v.Y) and math.isfinite(v.Z)
end

local function isCleanString(s: unknown, maxBytes: number): boolean
	return typeof(s) == "string" and #s <= maxBytes and utf8.len(s) ~= nil
end

local function isShopItem(value: unknown, shopFolder: Folder): boolean
	return typeof(value) == "Instance" and value:IsA("Folder") and value:IsDescendantOf(shopFolder)
end

local function isIdList(value: unknown, maxItems: number): boolean
	if typeof(value) ~= "table" then return false end
	local count = 0
	for key, id in value :: { [unknown]: unknown } do
		count += 1
		if count > maxItems or typeof(key) ~= "number" or key % 1 ~= 0 or key < 1 or key > maxItems then
			return false
		end
		if not isCleanString(id, 64) then return false end
	end
	return true
end
```

For CFrames, run `math.isfinite` over every value of `cf:GetComponents()`. Centralize all this in a guard module; `examples/safe-remote/` in this plugin is a working starting point.

Never let a remote argument choose what the server loads or changes: no `require` from arguments, no dynamic asset loads, no remote that deletes or edits an arbitrary path or Instance, even if it seems trivial. Those get chained with other bugs.

### Races: handlers overlap when they yield

Remote ordering applies to when each handler **starts**. If a handler yields, the next message can start before it resumes. `if coins >= cost` → `task.wait`/DataStore call → `coins -= cost` is a duplication exploit. Check and mutate in one uninterrupted step, or hold a per-player busy flag, or re-validate after the yield. Trades must validate every participant's data **before** committing any of it, and revert everything on failure: the documented dupe is a trade followed by an instant leave with a failed save.

## Remote Limits

| Limit | Value | Implication |
| --- | --- | --- |
| RemoteEvent `FireServer` | About 500 requests/s per client, shared across all RemoteEvents | Spam can throttle your own legitimate traffic. This is not your rate limit. |
| UnreliableRemoteEvent `FireServer` | About 500/s per client, shared by type; excess is dropped rather than delayed | Never use for state that must arrive. |
| UnreliableRemoteEvent payload | Over 1000 bytes: dropped | Any direction. |
| No connected handler | RemoteEvent/RemoteFunction queue is limited by count and memory; Unreliable discards immediately | Connect handlers at startup. |
| RemoteFunction, reliable payload size | The docs publish no number | Don't assume one. Rate limit `OnServerInvoke` yourself. |
| Ordering | RemoteFunction and RemoteEvent have no relative order guarantee | Sequence numbers if strict order matters. |

## Rate Limiting

Everything client-triggerable needs a server-side budget (remotes, Touched, prompts, ClickDetectors), especially anything that hits a throttled backend (`DataStoreService`, `BadgeService`), clones large models, or broadcasts to all clients. A client-side cooldown alone is worthless.

```lua
--!strict
local Players = game:GetService("Players")

type Bucket = { tokens: number, last: number }
local buckets: { [Player]: { [string]: Bucket } } = {}

local function allow(player: Player, key: string, perSecond: number, burst: number): boolean
	local now = os.clock()
	local forPlayer = buckets[player]
	if not forPlayer then
		forPlayer = {}
		buckets[player] = forPlayer
	end
	local b = forPlayer[key]
	if not b then
		b = { tokens = burst, last = now }
		forPlayer[key] = b
	end
	b.tokens = math.min(burst, b.tokens + (now - b.last) * perSecond)
	b.last = now
	if b.tokens < 1 then return false end
	b.tokens -= 1
	return true
end

Players.PlayerRemoving:Connect(function(player: Player)
	buckets[player] = nil
end)
```

Overflow is a signal for the suspicion score (see Detection), not grounds for a ban by itself.

## Instances the Client Can Trigger

These are not remotes, but they are the same attack surface. Exploiters can fire them from anywhere, often ignoring `Enabled` and `MaxActivationDistance`.

| Instance | What the server enforces | Gap |
| --- | --- | --- |
| ProximityPrompt | A distance check on `Triggered` only (the tactics page still warns of activation "at any range", so do your own) | Fires even if `Enabled` is false on the server. `HoldDuration` is client-manipulable, `PromptButtonHoldBegan`/`TriggerEnded` have no distance check. |
| ClickDetector | Nothing | Any event, any distance, even disabled or not parented to something clickable. |
| DragDetector | Drag events: `Enabled` and `PermissionPolicy`; nothing else | Events inherited from ClickDetector are unchecked. |

In every handler check on the server: intended enabled state, player state (character exists, alive, in range), rate limit, minimum elapsed time for timed interactions. If the instance sits on an unanchored assembly, an exploiter can own it and bring it to themselves, defeating distance checks: anchor it or set ownership explicitly (`part:SetNetworkOwner(nil)`).

## Distance Checks and Character Reality

By default the client simulates its character and any part it owns. With ownership a client can teleport, fly, change speed without any property event firing, disable collision, and replicate Inf/NaN CFrames and velocities that also fling others. `Touched` can be forged or suppressed.

```lua
--!strict
local function canReach(player: Player, target: BasePart, maxStuds: number): boolean
	local character = player.Character
	local root = character and character:FindFirstChild("HumanoidRootPart")
	if not root or not root:IsA("BasePart") then return false end
	-- Written as <= so a NaN position fails closed; `> max then reject` would pass NaN
	return (root.Position - target.Position).Magnitude <= maxStuds
end
```

**A distance check is necessary, not sufficient:** the position it reads is client-reported, so a teleporting exploiter passes it. For anything valuable, add server-only progress state: ordered checkpoints, the chest not already open, mob alive, cooldown, item not claimed.

Movement validation has no universal recipe (docs: distance-over-time fails in physics-heavy games):

- Sample on the server and average over time for latency; a leaky-bucket accumulator tolerates bursts and catches sustained violation. Project to XZ for ground movement. Exempt your own teleports.
- Rubber-band to the last valid position first; punish only on accumulated evidence.

**Server authority** removes this class of cheat because clients send only inputs. It has been generally available and opt-in since 2026-07-09 ([Full Release](https://devforum.roblox.com/t/4727993)), though the network-ownership guide still labels it Beta and release notes through 745 still carry Pending fixes. See the `roblox-server-authority` skill; don't assume it's on unless `Workspace.AuthorityMode` is `Server`.

### Combat

Client sends origin and claimed hit, never damage or victim-as-fact. Server checks: origin near the character (with latency tolerance); hit position near the claimed part; no **static** geometry between (dynamic objects cause latency false rejects); fire rate, ammo, reload/sprint state; target alive and an enemy. Damage comes from server weapon stats.

## Client-to-Client Relays

`OnServerEvent` → `FireAllClients(sameArgs)` lets one exploiter spam effects and script errors onto every player. The server is a **gatekeeper, not a relay**: same validation, per-player rate limit, permission check, range check, then broadcast.

## RemoteFunction Hazards

`InvokeClient` (use `FireClient` instead, or two RemoteEvents with a server-side timeout):

- No return from the client: the server thread **yields forever**. Client error: rethrown on the server. Disconnect mid-call: `InvokeClient` throws.

`OnServerInvoke` needs all of the above validation, and its return value is an information channel: return only what that client may see.

## Confidentiality, Places, and Teleports

- **Server code stays out of replicated ModuleScripts.** An `if RunService:IsServer()` branch ships to every client and decompiles cleanly. Server logic lives in `ServerScriptService`/`ServerStorage`.
- **Avoid descriptive, predictable names** for sensitive instances.
- **A client can teleport to any place in the universe once it's in one.** Disabling Direct Access only removes the website Join button. **Kicking on arrival doesn't help**: content replicates before your kick runs.
- Set **Access Control for Places → Secure within universe only** (Creator Dashboard) for gated subplaces; it blocks client-initiated teleports before join. Use `TeleportAsync` from the server; client `Teleport()` is deprecated.
- **Unreleased content goes in a separate private universe.** Nothing else is reliable.
- Fail closed in restricted places: disallow by default, including when the verification call errors.
- **`SetTeleportData` is visible to the client and unencrypted.** Never carry currency or entitlement in it; use DataStores or MemoryStores.

## Secrets

Never put an API key in a script, attribute or data store. Use the per-game **secrets store** (Creator Dashboard → Secrets; owner or group owner only; 500 per game; 1,024 chars each; bind a specific domain rather than `*`).

```lua
--!strict
local HttpService = game:GetService("HttpService")

local apiKey = HttpService:GetSecret("analytics_key") -- server only
print(apiKey) --> Secret(analytics_key), never the value
local url = apiKey:AddPrefix("https://my.example.com/endpoint?apiKey=")
```

- Requires **Allow HTTP Requests**. Works in URLs and headers, **not request bodies**. The value can't be manipulated as a string.
- `Can't find secret with given key` is what clients and local playtests get. For Studio, add **Local Secrets** in Experience Settings → Security. Collaborative testing uses the real store.

## Third-Party Assets and Backdoors

Toolbox models can carry scripts that hand their creator server access. Roblox moderation doesn't catch all of them, and a good rating isn't proof. Before keeping any inserted asset, search it for scripts (Toolbox indicates when a model has them), then:

| Red flag | Why |
| --- | --- |
| `require(<number>)` | Runs a module from an asset ID on the server (`MainModule`). You're running code you didn't review; the docs I read describe no way to pin a version. Treat as a backdoor unless you own it. |
| `loadstring` | Needs `ServerScriptService.LoadStringEnabled`. It defaults to false: leave it off. |
| Obfuscated code, long whitespace runs | Used to hide code off-screen in the editor. |
| Trigger on a specific player joining or command | Typical activation shape. |

Runtime loading:

- `InsertService:LoadAsset`: only assets owned by or shared with the creator, or Roblox-owned, plus `OpenUse` types.
- `AssetService:LoadAssetAsync` is its documented replacement. Its returned Model is **sandboxed with no capabilities by default**, so scripts in it don't run until you grant them. It loads the *latest* version. Third-party loading needs **Allow Loading Third Party Assets** (`AssetService.AllowInsertFreeAssets`, default off). Only enable it knowingly. Wrap in `pcall`.

**Sandboxing is Beta** (the capabilities guide calls it "experimental and available as a client beta"). Set `Workspace.SandboxedInstanceMode` to `Experimental`, then `Sandboxed = true` on the model, and grant the minimum `Capabilities`. Be most careful with Network, DataStore, LoadUnownedAsset, CapabilityControl, LoadString; avoid AccessOutsideWrite. The third-party guide's older names `SandboxedInstance` and `AssetRequire` are stale (`SandboxedInstanceMode` per reference; `LoadUnownedAsset` renamed in 711). Some features error with "lacking Capability Unassigned" during the beta, so test before relying on it.

## Purchases

Grant developer products in `MarketplaceService.ProcessReceipt`, set once in one server Script. Never trust a client signal such as `PromptProductPurchaseFinished`. Failure modes that make idempotency mandatory:

- The callback for one purchase **can run on two servers at once** if the user joins a second server before the first returns.
- Returning `PurchaseGranted` can still fail to record, leaving the purchase unresolved, so you see it again.
- `NotProcessedYet` retries only on another purchase or a rejoin, with no timer.
- No callback at all means receipts are **auto-acknowledged and unrecoverable**.

So: record `PurchaseId` durably in the same atomic write as the grant, and return `PurchaseGranted` only after the save succeeds. Implementation: see the `roblox-datastores` skill.

Client input saved to a DataStore also needs: no Instances, capped strings and key counts, no NaN or nil indices, valid UTF-8, finite numbers.

## Detection and Consequences

Validation comes first; detection only supplements it. Philosophy: server decides, neutralize harm first, stay proportional and reversible.

- **Heuristics are signals:** fastest possible completion time, rate of gain, inhumanly regular action cadence. Keep a suspicion score across **varied** signals; act only past a high threshold. One signal can be a skilled player finding a shortcut.
- **Honeypots:** a remote no legitimate client script ever fires, or a server → client remote fired from the client. High confidence: log and kick.
- **Ladder:** silent logging → quiet mitigation (drop, clamp, resync) → temporary restrictions (trading, leaderboards, matchmaking) → kick, suspension, permanent ban. **Delay** visible consequences so you don't reveal the trigger. Version detection rules and track consequence rates.

`Player:Kick(message)` ends one session only; a server message over 500 **bytes** is discarded for the default. `Players:BanAsync` persists, evicts across servers, and can catch alts. Read [references/platform-gates.md](references/platform-gates.md) before writing a ban or a privilege gate. `PlayerRemoving` passes `Enum.PlayerExitReason` as a second argument (694).

## Platform Gates

Details and samples are in [references/platform-gates.md](references/platform-gates.md). Read it when using any of these.

- **Device Trust** (`IntegrityService.EvaluatePlayerDeviceTrust`, `Player.DeviceTrustLevel`), announced available 2026-10-27. Describes the device, not whether the player cheats. It doesn't replace anything on this page. List allowed levels; never compare them numerically. `Pending` is never trusted, and Studio always stays `Pending`.
- **Groups:** use `GroupService:GetRolesInGroupAsync`. `Player:GetRankInGroupAsync`/`GetRoleInGroupAsync` are deprecated (723). Fail closed.
- **Age check** (`Player.AgeChecked`, `PromptAgeCheck`) and **`Player:IsVerified`**: account signals, never gate core gameplay on them.

## Exploit Surface Checklist

| Pattern | Risk → Fix |
| --- | --- |
| `OnServerEvent:Connect(function(p, dmg) hum:TakeDamage(dmg)` | Client picks damage → compute from server weapon stats. |
| "I killed X, give reward" | Server must witness it (server-side hit validation). |
| Stats read from a Tool or Instance the client passes | Look up server-side by id. |
| Price from a GUI or argument; item prices in client-editable state | Server catalog only. |
| Server later reading a Value/attribute the client "set" | Most client property writes don't replicate, but the client's own view can't be trusted either. Keep authority in server modules. |
| `if x > balance then reject`, range checks with no `isfinite` | NaN passes → `math.isfinite` first. |
| Only `.X` checked on a Vector3 | NaN in Y/Z → check all components. |
| Yield between balance check and deduction | Dupes → atomic step or lock. |
| `require(arg)`, path or Instance from a remote then `:Destroy()` | Arbitrary server action → whitelist by id. |
| Prompt/ClickDetector handler with no server check | Fires from anywhere → full handler validation. |
| Remote that only relays to `FireAllClients` | Griefing → gatekeeper. |
| Teleport/CFrame request with no bounds | Validate destination is legal for that player. |
| `GiveAdmin`, `SetCoins`, test remotes in production | Gate by server-side `UserId` allowlist; don't register QA remotes unless `RunService:IsStudio()`. |
| `InvokeClient` in gameplay | Thread leak → `FireClient`. |
| `PromptProductPurchaseFinished` grants items | Spoofable → `ProcessReceipt`. |
| Reward on `Touched` alone | Forgeable → server distance and state checks. |
| API key in a script, `SetTeleportData` holding entitlements | Readable → secrets store, DataStore. |
| Kick as the only guard on a secret subplace | Content already replicated → secure teleports. |

## When You (Claude) Write Roblox Code

Produce this shape without being asked:

1. `Shared/Types.luau` for payload types.
2. Server module owning all state, with handlers that validate first.
3. Client controller that only *renders* state and *requests* actions.
4. Remotes created by the server in one folder, never by the client.
5. Rate limit plus full validation on every handler and every prompt/detector event.
6. `math.isfinite` on numbers, every component on vectors, length and UTF-8 on strings. Note: selene 0.32's Roblox std flags `math.isfinite` as unknown, a lint false positive; `-- selene: allow(incorrect_standard_library_use)` covers it.

If asked for something inherently insecure ("let the client set its coins"), implement the secure version and explain the change in one sentence. Say plainly when a feature depends on a Beta or Pending API.

## Sources

Base `https://create.roblox.com/docs/`:

- `scripting/security/`: `security-tactics`, `client-server-boundary`, `defensive-design`, `network-ownership`, `access-control`, `third-party-vulnerabilities`, `server-side-detection`, `device-trust`
- `scripting/events/remote`, `scripting/capabilities`, `cloud-services/secrets`, `projects/teleport`, `projects/server-authority`
- `reference/engine/classes/`: `RemoteEvent`, `RemoteFunction`, `UnreliableRemoteEvent`, `AssetService`, `InsertService`, `MarketplaceService`; `reference/engine/libraries/math`

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
