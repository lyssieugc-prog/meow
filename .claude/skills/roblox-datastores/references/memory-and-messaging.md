# Memory Stores and MessagingService

Read this when building cross-server state: leaderboards, matchmaking, auctions, caches
or announcements. Both exist for data you can afford to lose. Memory store docs warn of
undocumented limits and call the per-partition values estimates, and messaging limits
may change, so always handle throttling.

## Choosing a memory store structure

Key access: hash map (it spreads across partitions). Order: sorted map. Processing
order: queue. Sorted maps and queues live on one partition, so shard them when hot; the
docs recommend a hash map if you don't need ordering. Keep the smallest TTL that works
and delete explicitly, with expiry as the safety net. `UpdateAsync` on hash and sorted
maps reruns on conflict and aborts on `nil`, so the same no-side-effects rule applies.

```text
Hash map:   SetAsync(key, value, expiration), GetAsync, UpdateAsync, RemoveAsync, ListItemsAsync
Sorted map: SetAsync(key, value, expiration, sortKey?), GetRangeAsync, UpdateAsync, GetSizeAsync
Queue:      AddAsync(value, expiration, priority?), ReadAsync(count, allOrNothing, waitTimeout), RemoveAsync(id)
```

```lua
--!strict
local MemoryStoreService = game:GetService("MemoryStoreService")
local queue = MemoryStoreService:GetQueue("Matchmaking", 30) -- 30 s invisibility

local function processBatch(handle: (any) -> ())
	-- waitTimeout defaults to -1 (wait forever); always pass one
	local ok, items, readId = pcall(function()
		return queue:ReadAsync(10, false, 5)
	end)
	if not ok or not items or #items == 0 then
		return
	end
	for _, item in items do
		handle(item)
	end
	-- Not removed before the invisibility timeout: other servers see the items again
	local removed, err = pcall(function()
		queue:RemoveAsync(readId)
	end)
	if not removed then
		warn(err)
	end
end

return processBatch
```

## Memory stores

| Limit | Value |
| --- | --- |
| Memory quota, per experience | 64 KB + 1.2 KB × users; drops only after an 8-day lookback |
| Request units per minute, per experience | 1000 + 120 × CCU |
| Sorted map or queue | 1,000,000 items; 100 MB |
| Data structure request units | 100,000 per minute |
| Partition throttling begins (estimate) | About 30,000 units per minute |
| Hash map per-key (estimate) | About 5,000 write and 15,000 read units per minute |
| Key / sort key / value | 128 chars / 128 chars / 32 KB |
| Expiration | Maximum 3,888,000 s (45 days) |
| Queue invisibility timeout | Default 30 s |
| Queue `ReadAsync` count | Maximum 100 |

Unit costs: most calls cost 1. Sorted map `GetRangeAsync` costs one per returned item
(1 if empty). Queue `ReadAsync` costs per item, plus 1 every 2 s while waiting.
Hash map `UpdateAsync` costs at least 2. `ListItemsAsync` costs partitions scanned
plus items.

| Status | Action |
| --- | --- |
| `DataStructureRequestsOverLimit`, `PartitionRequestsOverLimit`, `TotalRequestsOverLimit` | Cache, back off, shard |
| `DataStructureItemsOverLimit`, `DataStructureMemoryOverLimit` | Reduce or expire items, shard |
| `TotalMemoryOverLimit` | Delete, wait for expiry, flush (only if the game can lose the data), or Extended Services |
| `DataUpdateConflict` | Exponential backoff, reduce contention |
| `ItemValueSizeTooLarge`, `InvalidRequest` | Fix the request, split the value |
| `InternalError` | Check the status page, report with Universe ID |

Open Cloud Flush Memory Store clears **every** queue and map in the chosen scope
(`LIVE` or `TEST`), including those live features still use.

Dashboard: game > Monitoring > Memory Stores, with email alerts. Studio data is
isolated from production but has the same quota formulas, so CCU-based quotas are
tiny while testing.

## MessagingService

These limits may change, per the reference.

| Limit | Value |
| --- | --- |
| Topic name | 1-80 characters |
| Message size | 1 KB |
| Sent per server | 600 + 240 × players per minute |
| Received per topic | 40 + 80 × servers per minute |
| Received for the game | 400 + 200 × servers per minute |
| Subscriptions per server | 20 + 8 × players |
| Subscribe requests per server | 240 per minute |

Typically delivered within 1-2 s; best effort, not guaranteed.

`MessagingService` replaces the deprecated `GlobalDataStore:OnUpdate`. A lost message
must only cost latency: poll as a fallback for anything that matters, and never move
currency by message alone.

## Maturity

`MemoryStoreDistributedCounter` (733) is `Pending` and has no public docs: not in the
reference or the guides. The Studio 0.745 binary contains `GetDistributedCounter` and
"MemoryStoreDistributedCounter is not enabled." Don't ship code on it. The hash map
pagination fix (745) is also `Pending`, so tolerate changes while paging.

## Sources

- https://create.roblox.com/docs/cloud-services/memory-stores
- https://create.roblox.com/docs/cloud-services/memory-stores/hash-map
- https://create.roblox.com/docs/cloud-services/memory-stores/sorted-map
- https://create.roblox.com/docs/cloud-services/memory-stores/queue
- https://create.roblox.com/docs/cloud-services/memory-stores/best-practices
- https://create.roblox.com/docs/cloud-services/memory-stores/observability
- https://create.roblox.com/docs/cloud-services/cross-server-messaging
- https://create.roblox.com/docs/reference/engine/classes/MessagingService

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
