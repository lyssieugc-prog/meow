# Data Store Operations, Recovery and RTBF

Read this when rolling back data, using the Creator Hub tools, or setting up RTBF.

## Versions and recovery

- The first write to a key in each UTC hour becomes a versioned backup; later
  writes that hour permanently overwrite. Backups expire 30 days after being
  overwritten. Rollback granularity is about an hour.
- Tools: `ListVersionsAsync`, `GetVersionAsync`, `GetVersionAtTimeAsync`.
  **`RemoveVersionAsync` is deprecated (712).**
- Restoring a version: pass its metadata, as the official sample does, or it's
  lost. With session locks, that metadata can bring back an old `LockId`, so clear
  it and restore only while the player isn't in a session (my inference from the
  lock design, not in the docs).
- `RemoveAsync` writes a tombstone; versions stay readable until expiry and removed
  objects are deleted after 30 days. On an **ordered** store it deletes permanently,
  and writes overwrite with no history.

## Dashboards and tools

- **Data Stores Manager** (Creator Hub > game > Configurations > Data stores
  manager): browse keys by prefix (must start with the scope name), view value,
  metadata and versions, compare and revert versions, mark stores or keys for
  deletion (restorable during a cooldown; 30 days for stores), RTBF templates. Shows size against limit and
  **Est. Monthly Costs** once over quota. Above 100 stores, size and counts
  disappear from the list; use Open Cloud or the Data Stores Batch Processor.
- **Observability** (game > Monitoring > Data Stores): storage, request count by
  API and status, quota usage per type, 30 days; the latest 3 minutes may be
  incomplete. Requires ownership or analytics group permission. Add custom alerts.
- **Open Cloud shares the experience budget with game servers.** Rate-limit bulk
  scripts so they can't throttle live saves.
- **Delete expired data.** Reduce storage and requests before evaluating Extended
  Services.

## Right to be forgotten

- Preferred: **deletion templates** with a case-sensitive `{UserId}` token, up to
  100, in the manager's RTBF Deletion tab or the Configs API
  (`user_data_templates`). Roblox matches them when it processes a request.
- Templates match the **name or scope of the store or key**, by static pattern.
  Types: standard store, standard key, ordered key. **Whole-store deletion is
  standard only.** Default-scope key templates use `scope_pattern: "global"`.
- **The `userIds` argument isn't what templates match.** It's recommended for GDPR
  tracking and removal, so still pass it, but a key like `Save_8` with `{123}` in
  userIds isn't found by a template. Put the ID in the key.
- Otherwise use the right-to-erasure webhook: for data outside data stores,
  unsupported schemas, or custom logic.
- Verify on a test experience with a dummy account, and confirm live deletions
  happen within 30 days.

## Sources

- https://create.roblox.com/docs/cloud-services/data-stores/versioning-listing-and-caching
- https://create.roblox.com/docs/cloud-services/data-stores/data-stores-manager
- https://create.roblox.com/docs/cloud-services/data-stores/observability
- https://create.roblox.com/docs/cloud-services/data-stores/best-practices
- https://create.roblox.com/docs/cloud-services/data-stores/right-to-be-forgotten
- https://create.roblox.com/docs/cloud/webhooks/automate-right-to-erasure
- https://create.roblox.com/docs/reference/engine/classes/GlobalDataStore

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
