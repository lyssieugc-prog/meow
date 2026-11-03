# Player Data Reference Implementation

Read this when writing or reviewing a player data module. It combines session lock,
ordered saves, autosave, shutdown and receipts, following Roblox's documented design
(`player-data-purchasing`).

**Status: type-checked under `--!strict` with luau-lsp (both solvers). Never run
against a real data store.** Test it on a separate experience, including the failure
paths, before relying on it. `retryAsync` is the function in `SKILL.md`, as a module.

```lua
--!strict
local DataStoreService = game:GetService("DataStoreService")
local HttpService = game:GetService("HttpService")
local MarketplaceService = game:GetService("MarketplaceService")
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")

local retryAsync = require(script.Parent.retryAsync)

type Data = { version: number, coins: number, purchases: { [string]: boolean } }
type Status = "loading" | "ready" | "nonSaving" | "released"
type Session = {
	player: Player,
	key: string,
	lockId: string,
	status: Status,
	data: Data,
	saving: boolean,
}

local STORE = DataStoreService:GetDataStore("PlayerData")
local AUTOSAVE_SECONDS = 180
local LOCK_EXPIRY_MS = (AUTOSAVE_SECONDS * 2 + 60) * 1000
local LOAD_ATTEMPTS, LOAD_RETRY_SECONDS = 12, 10 -- about 2 minutes behind a lock
local SHUTDOWN_BUDGET_SECONDS = 25
local CURRENT_VERSION = 1

local PRODUCT_HANDLERS: { [number]: (Data) -> () } = {
	[0] = function(data) -- replace 0 with the developer product ID
		data.coins += 100
	end,
}

local sessions: { [Player]: Session } = {}

local function newData(): Data
	return { version = CURRENT_VERSION, coins = 0, purchases = {} }
end

local function migrate(data: any): any
	-- Apply version steps here. Must not yield, the transform may rerun.
	return data
end

local function loadAsync(session: Session)
	local locked, tooNew = false, false
	for _ = 1, LOAD_ATTEMPTS do
		locked, tooNew = false, false
		local result = retryAsync(function()
			return STORE:UpdateAsync(session.key, function(current: any, keyInfo: DataStoreKeyInfo?)
				locked, tooNew = false, false -- this function can run more than once
				local metadata: any = if keyInfo then keyInfo:GetMetadata() else {}
				local age = if keyInfo
					then DateTime.now().UnixTimestampMillis - keyInfo.UpdatedTime
					else 0
				if
					metadata.LockId
					and metadata.LockId ~= session.lockId
					and age < LOCK_EXPIRY_MS
				then
					locked = true
					return nil
				end
				if current and current.version > CURRENT_VERSION then
					tooNew = true -- saved by a newer server build; don't downgrade it
					return nil
				end
				metadata.LockId = session.lockId
				return migrate(current or newData()), { session.player.UserId }, metadata
			end)
		end)
		if not result.ok then
			warn(`Load failed for {session.key}: {result.err}`)
			break
		end
		if not locked and not tooNew then
			session.data = result.value
			session.status = "ready"
			return
		end
		if tooNew or session.player.Parent ~= Players then
			break
		end
		task.wait(LOAD_RETRY_SECONDS)
	end
	session.status = "nonSaving" -- defaults only: never written, nothing sold
end

local function saveAsync(session: Session, release: boolean, deadline: number?): boolean
	while session.saving do -- one in-flight request per key keeps retries in order
		task.wait(0.1)
	end
	if session.status ~= "ready" then
		return false
	end
	session.saving = true
	local lockLost = false
	local result = retryAsync(function()
		return STORE:UpdateAsync(session.key, function(_, keyInfo: DataStoreKeyInfo?)
			local metadata: any = if keyInfo then keyInfo:GetMetadata() else {}
			lockLost = metadata.LockId ~= session.lockId
			if lockLost then
				return nil
			end
			metadata.LockId = if release then nil else session.lockId
			return session.data, { session.player.UserId }, metadata
		end)
	end, deadline)
	session.saving = false
	if lockLost then
		session.status = "nonSaving"
		session.player:Kick("Your data is open on another server. Please rejoin.")
		return false
	end
	if result.ok and release then
		session.status = "released"
	end
	return result.ok
end

local function endSession(player: Player, session: Session, deadline: number?)
	if session.status == "loading" then
		return -- the join thread sees the player gone and finishes this
	end
	if session.status == "ready" and not saveAsync(session, true, deadline) then
		warn(`Final save failed for {session.key}; the lock expires on its own`)
	end
	if sessions[player] == session then
		sessions[player] = nil
	end
end

local function hasWriteBudget(): boolean
	-- UpdateAsync spends one read and one write
	return DataStoreService:GetRequestBudgetForRequestType(Enum.DataStoreRequestType.StandardRead)
			> 0
		and DataStoreService:GetRequestBudgetForRequestType(
				Enum.DataStoreRequestType.StandardWrite
			)
			> 0
end

local function onPlayerAdded(player: Player)
	local session: Session = {
		player = player,
		key = `User_{player.UserId}`,
		lockId = HttpService:GenerateGUID(false),
		status = "loading",
		data = newData(),
		saving = false,
	}
	sessions[player] = session
	loadAsync(session)
	if player.Parent ~= Players then
		endSession(player, session)
		return
	end
	task.wait(math.random() * AUTOSAVE_SECONDS)
	while session.status == "ready" and sessions[player] == session do
		if hasWriteBudget() then
			saveAsync(session, false)
		end
		task.wait(AUTOSAVE_SECONDS)
	end
end

local function processReceipt(receipt: any): Enum.ProductPurchaseDecision
	local notYet = Enum.ProductPurchaseDecision.NotProcessedYet
	local player = Players:GetPlayerByUserId(receipt.PlayerId)
	local session = if player then sessions[player] else nil
	while session and session.status == "loading" and session.player.Parent == Players do
		task.wait(0.25)
	end
	if not session or session.status ~= "ready" then
		return notYet
	end
	local purchaseId = tostring(receipt.PurchaseId)
	if not session.data.purchases[purchaseId] then
		local handler = PRODUCT_HANDLERS[receipt.ProductId]
		if not handler or not pcall(handler, session.data) then
			return notYet
		end
		session.data.purchases[purchaseId] = true
	end
	-- Granted only after the ID is durable. A retried receipt skips the award above.
	if saveAsync(session, false) then
		return Enum.ProductPurchaseDecision.PurchaseGranted
	end
	return notYet
end

MarketplaceService.ProcessReceipt = processReceipt -- set once, before any yield

Players.PlayerAdded:Connect(onPlayerAdded)
for _, player in Players:GetPlayers() do
	task.spawn(onPlayerAdded, player)
end

Players.PlayerRemoving:Connect(function(player)
	local session = sessions[player]
	if session then
		endSession(player, session)
	end
end)

game:BindToClose(function()
	if RunService:IsStudio() then
		return
	end
	local deadline = os.clock() + SHUTDOWN_BUDGET_SECONDS
	local pending = 0
	for player, session in sessions do
		pending += 1
		task.spawn(function()
			endSession(player, session, deadline)
			pending -= 1
		end)
	end
	while pending > 0 and os.clock() < deadline do
		task.wait(0.1)
	end
end)
```

## Migration steps

Plug into `migrate` above. Must not yield, since the transform may rerun.

```lua
--!strict
local CURRENT_VERSION = 3

local MIGRATIONS: { [number]: (any) -> any } = {
	[1] = function(data)
		data.settings = { music = true }
		return data
	end,
	[2] = function(data)
		data.inventory = data.items or {}
		data.items = nil
		return data
	end,
}

local function migrate(data: any): any
	while data.version < CURRENT_VERSION do
		local step = MIGRATIONS[data.version]
		if not step then
			error(`missing migration from v{data.version}`)
		end
		data = step(data)
		data.version += 1
	end
	return data
end

return migrate
```

## Decisions to review for your game

- **Lock timing:** `LOCK_EXPIRY_MS` is 2 autosaves + 60 s, so one skipped cycle
  doesn't expire it. A crashed server blocks that player until expiry. The docs give
  the rule (save interval below expiry) but no specific expiry.
- **Lock-wait outcome:** after about 2 minutes behind a lock, this sample falls to a
  non-saving session. Your UI must tell the player, and block purchases and trades;
  kicking with "rejoin in a moment" is a simpler alternative.
- **Replication** is omitted. Send the client what it needs from the in-memory table,
  plus load and save error states so it can disable purchase prompts.
- **A release that landed but reported failure** retries, finds no lock, and reports
  a lost lock. That's harmless because the player is already leaving.
- **Purchase history** is unbounded here. See the note in `SKILL.md`.
- **Product handlers** must be synchronous and change only `data`, or a failure
  midway leaves a partial award.

## ProfileStore API

Read from `ProfileStore.luau` and its API page on 2026-11-01. Not run here.

- `ProfileStore.New(name, template)`, `store:StartSessionAsync(key, { Cancel = fn })`
  returning a profile or `nil`, `store:MessageAsync`, `:GetAsync`, `:VersionQuery`,
  `:RemoveAsync`, `store.Mock`.
- `profile.Data`, `.LastSavedData`, `:Reconcile()`, `:AddUserId(id)`, `:Save()`,
  `:EndSession()`, `:IsActive()`, `.OnSessionEnd`, `.OnLastSave`.
- Defaults: autosave 300 s, session steal after 40 s, assume dead at 630 s. `Steal = true`
  is debug only. The author says it isn't for leaderboards or global state.
- Its docs point to `LastSavedData` and `Save()` for purchases but don't document a
  `ProcessReceipt` flow, so the receipt rules in `SKILL.md` still apply.
- Use `User_{UserId}` as the key so RTBF templates match.

## Failure drills

| Drill | Expected |
| --- | --- |
| Load errors | Non-saving session; key unchanged; receipts return `NotProcessedYet` |
| Second server opens the key during a session | Second server waits and doesn't load |
| Lock expires, another server takes it, first server saves | First stops saving and sends the player to rejoin |
| Player leaves during load | Lock released after load finishes |
| Save fails after awarding a product | `NotProcessedYet`; next receipt run doesn't award twice |
| Server closes with many players | Every release attempted in parallel inside 30 s |
| Older build sees a newer schema version | Load aborted; data unchanged |

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
