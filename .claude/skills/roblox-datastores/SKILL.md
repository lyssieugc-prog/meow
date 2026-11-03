---
description: Safe persistence and cross-server data in Roblox. DataStore retries by error code, UpdateAsync, session locking, autosave, BindToClose, ProcessReceipt idempotency by PurchaseId, schema migration, versions and rollback, RTBF templates, limits and quotas, memory stores, MessagingService, ProfileStore. Use when writing or reviewing code that touches DataStoreService, MemoryStoreService, MessagingService, player profiles, developer product receipts, saving and loading, or data loss and item duplication bugs.
---

# Roblox DataStores: Data Loss Is a Design Failure

Players quit forever over lost progress or a paid item that vanished. Data store
calls fail routinely (throttling, service errors, shutdown mid-write), and a
failed write doesn't even tell you whether it landed. Treat persistence as a
system, not a `SetAsync` call.

## Non-Negotiables

1. **Wrap every call in `pcall`. Retry only transient failures**, with capped
   exponential backoff and jitter. Retrying `104` or `105` can't succeed.
2. **Keep retries in order per key.** An older retry can overwrite a newer
   write. Allow one in-flight request per key; don't `task.spawn` overlapping
   saves.
3. **A failed write has an unknown outcome.** The backend may have completed it.
   Design writes to be idempotent, or verify with
   `GetAsync(key, options)` where `DataStoreGetOptions.UseCache = false`. A plain
   `GetAsync` can return a local copy up to 4 s old.
4. **Read-modify-write uses `UpdateAsync`.** Its transform receives the latest
   value and reruns if another server wrote in between.
5. **One server owns a player's data at a time (session locking).** Without it,
   a rejoining player can load stale data on server B while server A's final
   save is still retrying. Roblox's docs call this a common source of item
   duplication exploits.
6. **Save periodically, on leave, in `game:BindToClose`, and at purchase
   checkpoints.** Gameplay mutates an in-memory table, never the data store.
7. **Fallback data must never overwrite real data.** A failed load produces a
   session flagged non-saving. Loading defaults and then saving them is the
   classic total wipe.
8. **Grant nothing as final until `PurchaseId` is persisted.** See
   [Purchases](#purchases-processreceipt).

## Build or Adopt

Official guidance: modules are an option, but review ownership, maintenance status
and features, and know how to reach your data without them. **DataStore2 is legacy;
don't use it in new games.** Roblox's
[Purchase Handling sample](https://create.roblox.com/store/asset/14376223840/Purchase-Handling)
is explicitly reference-only, not proven in a popular game.

**ProfileStore** (loleris, Mad Studio; successor to ProfileService) implements session
locking, autosave and shutdown handling. State on 2026-11-01: repo
`MadStudioRoblox/ProfileStore` not archived, 6 open issues/PRs, last commit 2025-07-31,
so re-check before adopting. Wally name is `lm-loleris/profilestore`
(`loleris/profilestore` doesn't resolve). Real API names are in the
[reference file](references/player-data-reference.md); don't guess them.

Whatever you choose, the patterns below are the review checklist.

## Retry by Error Code

```lua
--!strict
export type Result<T> = { ok: true, value: T } | { ok: false, err: string }

-- From the official error table: invalid requests and states retrying can't fix.
-- 101-107 bad key, value or range; 401-402 shutting down; 403 Studio API access off;
-- 509 writes blocked on this RCC; 511-513 bad metadata or userIds.
local NON_RETRYABLE_CODES = " 101 102 103 104 105 106 107 401 402 403 509 511 512 513 "
local NON_RETRYABLE_NAMES = {
	"InvalidTimestamp",
	"InvalidExclusiveStartKey",
	"InvalidTarget",
	"InvalidUserIds",
	"DatastoreDeleted",
	"KeyNotFound",
	"No pages to advance to",
}

local function isRetryable(message: string): boolean
	local code = string.match(message, "^(%d%d%d):")
	if code and string.find(NON_RETRYABLE_CODES, ` {code} `, 1, true) then
		return false
	end
	for _, name in NON_RETRYABLE_NAMES do
		if string.find(message, name, 1, true) then
			return false
		end
	end
	return true -- throttles, 5xx, unknown: bounded retry
end

local function retryAsync<T>(fn: () -> T, deadline: number?): Result<T>
	local MAX_ATTEMPTS, BASE_DELAY, MAX_DELAY = 5, 2, 32
	local lastError = "not attempted"
	for attempt = 1, MAX_ATTEMPTS do
		local ok, result = pcall(fn)
		if ok then
			return { ok = true, value = result }
		end
		lastError = tostring(result)
		if not isRetryable(lastError) or attempt == MAX_ATTEMPTS then
			break
		end
		local backoff = math.min(MAX_DELAY, BASE_DELAY * 2 ^ (attempt - 1))
		local delaySeconds = math.min(MAX_DELAY, backoff + math.random() * backoff)
		if deadline and os.clock() + delaySeconds >= deadline then
			break
		end
		task.wait(delaySeconds)
	end
	return { ok = false, err = lastError }
end

return retryAsync
```

The official table has no "retryable" column, so this denylist is derived from its
notes plus the rule "don't retry invalid requests." Throttles and 5xx are retried;
unknown errors get a bounded retry because a lost save costs more. The docs don't define
the message format: `NNN: text` and `Name: text` come from strings in the Studio 0.745
binary, which is evidence, not a contract. `502` wraps an inner error.

Defer non-critical work with `DataStoreService:GetRequestBudgetForRequestType()` using
`Enum.DataStoreRequestType.Standard*` or `Ordered*` (`Read`, `Write`, `List`, `Remove`).
The enum's reference says the older values (`GetAsync`, `UpdateAsync`, ...) should not be
used. `UpdateAsync` spends read and write. Never defer a final or receipt save.

## UpdateAsync Contract

`store:UpdateAsync(key, function(current, keyInfo) -> (newValue, userIds?, metadata?))`

- **The transform can't yield and can run several times.** No `task.wait`, no
  data store calls, no granting items or firing remotes inside it. Set outcome
  flags, and reset them at the top of each run.
- **Return `nil` to abort the write.** The server stops attempting. Use your own
  outcome variable rather than reading meaning into the return value.
- **Return userIds and metadata every time.** Omitting them clears the stored
  ones. Same on `SetAsync(key, value, userIds, options)` and
  `IncrementAsync(key, delta, userIds, options)`, with metadata set through
  `DataStoreSetOptions:SetMetadata()` and `DataStoreIncrementOptions`.
- `keyInfo` offers `Version`, `CreatedTime`, `UpdatedTime` (ms since epoch),
  `GetUserIds()`, `GetMetadata()`. It is always `nil` on ordered data stores.
  Treat it as optional on a key's first write too; the docs don't say what it is
  there.
- `SetAsync` is fine for a new key, or a value that doesn't depend on the old one.

## Session Locking

Roblox's reference design keeps the lock in **key metadata**, written atomically
in the same `UpdateAsync` that loads the data:

- **Claim:** generate a GUID for the session. If metadata has a different
  `LockId` updated less than the expiry ago, abort and retry after a delay. An
  expired lock may be taken over. A key's `UpdatedTime` works as the age because
  every autosave rewrites it.
- **Every write:** abort unless metadata's `LockId` is still *yours*. A `nil`
  lock isn't enough: another server may have taken your expired lock and released
  it. If you lost it, stop saving and send the player to rejoin.
- **Release:** the final write removes `LockId`.
- **Expiry > autosave interval**, with room for a skipped cycle and retries. The
  autosave is what keeps the lock alive. ProfileStore's ratio is 300 s and 630 s.
- Metadata is small: 300 characters in total. A GUID fits; profile data doesn't.

Complete strict sample: [references/player-data-reference.md](references/player-data-reference.md).
Read it when writing or reviewing a player data module.

## The Profile Lifecycle

```text
PlayerAdded ──▶ claim lock + migrate + load ──▶ in-memory profile (source of truth)
                                                  │ gameplay mutates ONLY this table
                                                  ├─ autosave (random first offset)
                                                  ├─ save at purchases and trades
PlayerRemoving ─▶ final save + release (or finish after an in-progress load)
BindToClose ───▶ final save + release for ALL, in parallel, wait for all
```

- **Autosave cadence:** the official sample uses 180 s. Pick one that fits your
  limits and stays below the lock expiry. Give each player a random first offset
  so servers don't flush together.
- **`BindToClose`:** the server waits for all bound functions, in parallel, for
  30 s, then shuts down regardless. Save in parallel, stop retries against a
  deadline near 25 s, and include saves `PlayerRemoving` already started. Check
  `RunService:IsStudio()` so a Studio stop doesn't hang.
- **Failed load:** the official recommendation is defaults, clear messaging, a
  flag that blocks all saving, and blocked purchases. Kicking is the stated
  alternative. Either is safe. Saving is what isn't.
- **Throttling blocks you:** a throttled call yields in a queue (limit 30 per
  queue, then `301`-`306`). Don't yield a critical path on it without a deadline.
- **Studio access:** "Enable Studio Access to API Services" reaches the *same*
  data as the live game. Enable it only on a separate test experience. Memory
  stores are isolated between Studio and production; data stores are not.

## Purchases (ProcessReceipt)

Facts from the reference, each of which shapes the code:

| Fact | So you must |
| --- | --- |
| No callback set means receipts are **auto-acknowledged**, and you can't get them back | Assign once, in one server script, at boot |
| No timeout on yields; result accepted when it returns | Wait for the profile to load, but stop if the player leaves |
| No timed retry: `NotProcessedYet` reruns only on a new purchase or rejoin | Don't loop in the callback; persist progress |
| Player must be on the server to invoke it | Handle the player being absent or leaving |
| Same purchase **can run on two servers at once** | Rely on the session lock and `PurchaseId` |
| `PurchaseGranted` may fail to record, so the receipt comes back | Grants must be idempotent by `PurchaseId` |
| Several pending receipts run in non-deterministic order | Don't depend on order |

The order, from the official sample:

1. Verify the profile is loaded, is current, and didn't fall back to defaults.
2. If `PurchaseId` is recorded and persisted, return `PurchaseGranted`.
3. Otherwise award it in memory and record `PurchaseId` **in the same profile**,
   so both persist atomically.
4. Save. Return `PurchaseGranted` only if the save succeeded; otherwise
   `NotProcessedYet`. A later retry finds the ID and doesn't award it twice.

The trade-off is documented: if that save fails, the player keeps the item for
the session without it being final. Never use `PromptProductPurchaseFinished` to
grant; firing doesn't mean a purchase happened. Roblox also checks `ProcessReceipt`
on **Listed** products on an ongoing basis, and a flagged product stops appearing
outside your game until fixed (Creator Hub > Monetization > Shop shows a banner).

`PurchaseId` history grows unless trimmed. The docs don't cover trimming; a pruned ID can
be granted again if an old receipt resurfaces, so keep it long.

## Schema Versioning and Migration

Data outlives your code. Store `version` from day one and run ordered, non-yielding
migrations inside the `UpdateAsync` that claims the lock (sample in the reference
file). Abort if the stored version is newer than this build: an old server must not
overwrite data shaped by a new one. Deep-copy defaults; a shared table corrupts across
players. Before publishing storage changes, take a snapshot (Open Cloud, once a day).

## Values and Keys

- Key `User_{UserId}` in one `PlayerData` store: a small, fixed set of stores,
  static key patterns, never display names, never one store per player. Split a
  record only by access pattern or size, keeping atomic data together.
- Name, key and scope: 50 characters each. Value: 4,194,304 characters per key,
  measured as serialized JSON (`HttpService:JSONEncode`).
- Supported: nil, booleans, numbers, strings, tables of those, and `buffer`.
  Anything else either errors **or silently becomes `nil`**, so preview with
  `JSONEncode`. Convert `CFrame`, `Vector3`, `Color3` and enums at the boundary.
- Strings must be valid UTF-8; store binary as `buffer`. Don't store `inf`, `-inf`
  or `nan`. Avoid arrays with holes and mixed-key tables; the official docs don't
  define their behavior.
- Don't pre-compress; the service compresses. Scopes are legacy for new games; use key
  prefixes with `ListKeysAsync`.
- `Player.UserId` remains right for keys. The reference now calls `Player.User`
  the standard identity in new code and describes domain-scoped IDs, but RTBF's
  token is `{UserId}` and no data store guide covers re-keying. Don't re-key.

Exact limits, budgets, prices and the error table:
[references/limits-and-errors.md](references/limits-and-errors.md). Read it when
sizing traffic, planning storage, or classifying a specific error.

## Throughput, the Real Per-Key Limit

There is no documented "one write per key every N seconds". Per key, across all
servers, over a rolling 60 s: **4 MB written and 25 MB read**, each request
rounded up to the next KB. A 1 MB profile saved every 15 s reaches the write
limit; a 4 MB value allows about one write a minute. Throttles surface as
`KeyThrottled` or `DatastoreThrottled`.

## Operations, Recovery, RTBF

Read [references/operations.md](references/operations.md) when rolling back data, using
Data Stores Manager or the dashboards, or setting up RTBF. What must change your code:

- First write per UTC hour becomes a 30-day backup, so rollback granularity is about an
  hour. **`RemoveVersionAsync` is deprecated (712).** Ordered stores have no history.
- Open Cloud shares the experience budget with game servers; rate-limit bulk scripts.
- **RTBF templates match the user ID in the store or key name or scope**, through a
  `{UserId}` token. The `userIds` argument isn't what they match: still pass it (the
  reference recommends it for GDPR tracking), but put the ID in the key.

## Which Service

| Need | Use | Key property |
| --- | --- | --- |
| Progress, inventory, purchases | Data store | Permanent, versioned |
| Persistent ranked integers | Ordered data store | Integers only, no versions or metadata |
| Live leaderboard, auction, cache | Memory store sorted or hash map | Up to 45 days, not durable |
| Matchmaking and work queues | Memory store queue | Invisibility timeout |
| Announcements, nudges | MessagingService | Best effort, 1 KB, usually 1-2 s |

Never keep the only copy of owned items in a memory store or a message.
`MessagingService` replaces the deprecated `GlobalDataStore:OnUpdate`. Read
[references/memory-and-messaging.md](references/memory-and-messaging.md) when using
either.

## Maturity of Recent APIs

Release-note `Pending` means shipped in the binary but not yet enabled.

| API | Docs | Status | Before depending on it |
| --- | --- | --- | --- |
| `OrderedDataStore:BatchGetAsync(keys)` (730): up to 100 keys, one read per key, missing keys omitted; standard stores throw | Guide and YAML | `Pending`, gate `DataStoreEnableOrderedBatchGetAsync` | `pcall` it and keep a `GetAsync` fallback |
| `DataStoreService:SetRateLimitForRequestType(type, base, perPlayer)` (709): once per type at server init; `UpdateAsync` and `OnUpdate` types error | Guide and YAML | `Pending` | `pcall` at init; treat failure as "default limits" |
| `MemoryStoreDistributedCounter` (733) | **None**, not in YAML or guides | `Pending` | **Don't ship.** Binary has `GetDistributedCounter` and "is not enabled", with no public contract |
| Hash map pagination reliability fix (745) | Release note | `Pending` | Tolerate changes while paging |

Docs alone can't confirm the live flag state of any of these.

## Red Flags

| Red flag | Why |
| --- | --- |
| Retry loop with no error classification or cap | Wastes budget on `104`/`105`; hides real bugs |
| `task.spawn` retry alongside newer saves | Older retry overwrites newer data |
| `GetAsync` + `SetAsync` for read-modify-write | Lost updates across servers |
| Side effects or yields in an `UpdateAsync` transform | It reruns; yielding isn't allowed |
| `pcall` failure on load treated as a new player | Default data overwrites real data |
| Saving on every coin pickup | Burns per-key throughput and budgets; buffer in memory |
| `BindToClose` missing, or sequential saves | Last minutes lost; 30 s isn't enough in series |
| `PurchaseGranted` before the save succeeds, or `PurchaseId` in a separate key | Lost grant or double grant |
| `ProcessReceipt` unset, assigned twice, or granting from `PromptProductPurchaseFinished` | Receipts auto-acknowledged or fake grants |
| Studio API access enabled on the live experience | Tests overwrite production data |
| One key per item, or one store per player | Requests, rollback and limits all get worse |
| User data in keys with no `{UserId}` pattern | RTBF automation can't match it |
| `RemoveVersionAsync`, `OnUpdate`, legacy `DataStoreRequestType` values | Deprecated or not to be used |
| Queue `ReadAsync` with no `waitTimeout`, or items never removed | Waits forever; items reappear |
| Trusting client-sent data into the profile | See the roblox-security skill |

## When You (Claude) Write or Review This Code

- Classify every call site: retried by code, ordered per key, shutdown deadline.
- Trace the failure paths first: load fails, lock held, lock lost, save fails after
  the write landed, player leaves mid-load, server closes mid-save.
- Say plainly that data code was only type-checked unless it ran against a real
  test experience. Never point Studio at live data to test.
- Re-check a number against `error-codes-and-limits` when a decision depends on it.

## Sources

- https://create.roblox.com/docs/cloud-services/data-stores
- https://create.roblox.com/docs/cloud-services/data-stores/best-practices
- https://create.roblox.com/docs/cloud-services/data-stores/error-codes-and-limits
- https://create.roblox.com/docs/cloud-services/data-stores/versioning-listing-and-caching
- https://create.roblox.com/docs/cloud-services/data-stores/player-data-purchasing
- https://create.roblox.com/docs/cloud-services/data-stores-vs-memory-stores
- https://create.roblox.com/docs/cloud-services/memory-stores
- https://create.roblox.com/docs/cloud-services/extended-services
- https://create.roblox.com/docs/production/monetization/developer-products
- https://create.roblox.com/docs/reference/engine/classes/GlobalDataStore
- https://create.roblox.com/docs/reference/engine/classes/DataStoreService
- https://create.roblox.com/docs/reference/engine/enums/DataStoreRequestType
- https://create.roblox.com/docs/reference/engine/classes/MarketplaceService
- https://create.roblox.com/docs/reference/engine/classes/DataModel
- ProfileStore: https://github.com/MadStudioRoblox/ProfileStore, https://madstudioroblox.github.io/ProfileStore/api/, https://devforum.roblox.com/t/profilestore-save-your-player-data-easy-datastore-module/3190543

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
