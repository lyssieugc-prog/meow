# Data Store Limits, Prices and Error Codes

Read this when sizing traffic, planning storage, or classifying a specific error.
Source: Roblox docs snapshot and live pages checked 2026-11-01. Memory store and
messaging limits are in [memory-and-messaging.md](memory-and-messaging.md).

## Data stores

| Limit | Value |
| --- | --- |
| Store name, key name, scope | 50 characters each |
| Value | 4,194,304 characters per key (serialized) |
| Metadata key / value / all pairs | 50 / 250 / 300 characters |
| Per-key write throughput | 4 MB per rolling 60 s, all servers |
| Per-key read throughput | 25 MB per rolling 60 s, all servers |
| Throughput rounding | Each request rounds up to the next KB |
| Throttle queues (Set, Ordered set, Get, Ordered get) | 30 requests each, then dropped with `301`-`306` |
| Local `GetAsync` cache | 4 s; cached reads don't count against limits |
| Version backups | First write per UTC hour; expire 30 days after overwrite |
| Deleted store or key | 30-day restore window; a deleted store still counts toward storage meanwhile |

Hot shared keys read by every server: reduce reads first, then shard copies
(`Weather_1..3`, chosen by hashing `JobId`) with a random first delay and jitter. Sharding
doesn't fix write throttling or raise budgets, and writes across copies aren't atomic.

### Experience-level requests per minute

`error-codes-and-limits` gives these. Game servers and Open Cloud share them.

| Type | Standard and ordered |
| --- | --- |
| Read | 300 + CCU × 40 |
| Write | 300 + CCU × 20 |
| List | 300 + CCU × 2 |
| Remove | 300 + CCU × 40 |

`UpdateAsync` decrements read and write. `BatchGetAsync` counts one read per key.

### Storage

`500 MB + 1 MB × lifetime users`, counting the compressed size of each key's latest
version only. Old versions and deleted keys don't count.

**Unresolved doc conflict.** On 2026-11-01 both the repo snapshot and the live
`extended-services` page list lower defaults: Get/Set `250 + CCU × 40/20`, List
`10 + CCU × 2`, Remove `100 + CCU × 40`, ordered GetSorted `100 + CCU × 2`, storage
`100 MB + 1 MB × lifetime players`. The limits page is the more specific, but plan
to the lower numbers until you've read your real limit on the observability
dashboard.

The docs don't state what happens to an experience over its storage limit that
hasn't enrolled in Extended Services, beyond Creator Hub notifications and the
manager's Est. Monthly Costs. Treat that as unknown.

### Per-server limits

`GetRequestBudgetForRequestType` returns the live budget; don't hardcode old
per-server formulas, which the current limits page no longer lists.
`SetRateLimitForRequestType` allowed ranges (`Pending`, 709): `baseLimit` 0-10000
and `perPlayerLimit` 0-200 for `Standard*` and `Ordered*`. Legacy types have far
smaller ranges and shouldn't be used. The limit is `base + perPlayer × players`.

## Data store errors

| Codes or names | Meaning | Action |
| --- | --- | --- |
| `101`, `102` | Key empty or over 50 chars | Fix code |
| `103`, `104` | Value type not allowed or not serializable (invalid UTF-8 too) | Fix code |
| `105` | Value over the size limit | Fix code, split or trim |
| `106`, `107` | Bad `GetSortedAsync` range or page size (1-100) | Fix code |
| `301`-`306` | Throttled and queue full; request dropped | Back off, retry |
| `*ExperienceThrottled`, `*GameServerThrottled` | Over experience or server limit, or queue full | Back off, retry, reduce traffic |
| `KeyThrottled`, `DatastoreThrottled` | Per-key or per-store rate | Back off, retry, reduce or shard |
| `*AsyncThrottle` (versions, listing) | Throttled | Back off, retry |
| `401`, `402` | Server shutting down | Don't retry |
| `403` | Studio API access off | Enable on a test experience only |
| `404`, `501`, `503`-`505` | Possible corruption or parse failure; docs say try later | Bounded retry, log |
| `502` | Rejected by API Services, wraps an inner error | Bounded retry, log the inner text |
| `509` | Blocked on a personal RCC | Don't retry |
| `511`-`513` | Metadata too large, too many user IDs, bad format | Fix code |
| `InternalServerError` | Occasional server error | Retry with backoff |
| `InvalidPlace`, `InvalidUniverse` | No matching ID; docs say try later | Bounded retry |
| `DatastoreDeleted`, `InvalidExclusiveStartKey`, `InvalidTarget`, `InvalidUserIds`, `InvalidTimestamp`, `KeyNotFound` | Request or state can't succeed | Don't retry |
| "No pages to advance to" | Advanced past the last page | Check `IsFinished` |

The docs have no explicit retryable flag. The Action column is derived from each
row's notes and the best-practice rule on invalid requests.

## Extended Services

Pay for usage over the defaults; throttled at your budget. Requires a verified 18+
account in a supported country. Minimum monthly budget per resource is $0.50; billed
at month end or at $10. Emails at 80% and 100%. Enabling it spends the owner's money,
so ask first.

| Resource | Price |
| --- | --- |
| Standard Get / Set / List / Remove | $0.08 / $0.80 / $0.60 / $0.80 per 1M |
| Ordered Get / Set / GetSorted / Remove | $0.08 / $0.25 / $0.32 / $0.25 per 1M |
| Data store storage | $0.12 per GB per month |
| Memory store requests | $0.003 per 1M |
| Memory store storage | $0.10 per GB per hour |

## Sources

- https://create.roblox.com/docs/cloud-services/data-stores/error-codes-and-limits
- https://create.roblox.com/docs/cloud-services/data-stores/versioning-listing-and-caching
- https://create.roblox.com/docs/cloud-services/extended-services
- https://create.roblox.com/docs/reference/engine/classes/DataStoreService

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
