# Platform Gates: Bans, Device Trust, Groups, Age, Verification

Read this when writing a ban, an admin or staff check, or any feature restricted by device or account status. All checks run on the server.

## Bans and kicks (GA)

`Players:BanAsync` requires `Players.BanningEnabled`, which can only be set in Studio's Players properties, not from Luau. Server only.

```lua
--!strict
local Players = game:GetService("Players")

local function banForOneDay(userId: number, evidence: string): boolean
	local ok, err = pcall(function()
		Players:BanAsync({
			UserIds = { userId }, -- max 50
			Duration = 86400, -- seconds; -1 is permanent; 0 and other negatives are invalid
			DisplayReason = "Broke rule 3. Appeal via the group page.", -- max 400, text filtered
			PrivateReason = evidence, -- max 1000, not filtered, never sent to clients
			ApplyToUniverse = true, -- default true
			ExcludeAltAccounts = false, -- default false: the ban SPREADS to suspected alts
			ApplyDeviceBlock = false, -- default false: true blocks the device for 24 h
		})
	end)
	if not ok then
		warn("Ban failed:", err)
	end
	return ok
end
```

- It yields and makes an HTTP call per ID. It can fail, and errors are aggregated per UserId: always `pcall` and retry or report.
- Banning in the universe's start place effectively excludes the whole universe, even with `ApplyToUniverse = false`.
- Alt propagation is on by default. Weigh it against false positives: **prefer reversible actions when detection confidence is low**, and scale duration with `Players:GetBanHistoryAsync`.
- Before its 24 hours run out, a device block is lifted only by `Players:UnbanAsync`; unbanning any other way leaves it.
- In Studio and team tests it can be called, but bans don't apply to production.
- **Policy:** state your rules somewhere every player can see and give a way to appeal. Display reasons may name platforms but not direct links, handles or personal information.

`Player:Kick(message)`: session only, no persistence. From the server, a message over 500 **bytes** (multibyte chars reach this sooner) is discarded in favor of the default notice. On the client it can only kick the local player. Kicking does not undo replication that already happened.

## Device Trust (announced available 2026-10-27)

The docs and the [announcement](https://devforum.roblox.com/t/improve-your-game-security-with-the-device-trust-api/4961129) don't label it beta. It's absent from the release notes I checked (690-745), so its availability rests on those two sources. I have not tested it live.

Roblox takes no action on the level. It measures how hard cheating is on the device, **not whether the player cheats**, and it doesn't replace validation.

| Level | Meaning |
| --- | --- |
| `Pending` | Still checking. Every player starts here, and it can stay for the whole session. **Always in Studio.** Never trusted enough for sensitive features. |
| `Unknown` | Unsupported platform or Roblox-side error. Not the same as untrusted: you decide. |
| `None` | Failed, or an unverifiable platform often used for cheating. |
| `Standard` | Passed some checks. |
| `Full` | Passed all. |

| Platform | Reports |
| --- | --- |
| Android, Chromebook, Meta Quest | `None`, `Standard` or `Full` (older hardware may not reach the top even when untampered) |
| Web browser | `None` |
| iOS, macOS, **Windows**, Xbox, PlayStation | `Unknown` until supported. The announcement says "in the upcoming year", no dates. |

So today, a Windows PC reports `Unknown`. A policy that blocks `Unknown` blocks all of those platforms. The announcement says Android and Web, and elsewhere "Android-only"; the guide's table above is the fuller list. Check the Creator Hub **Safety → Anti-cheat** page for your player breakdown before enforcing.

Least to most disruptive: route by level, restrict a feature, reject (last resort). Suggested starting points from the guide: competitive FPS ranked requires `Standard`/`Full`; trade hangouts block only `None`; roleplay gives everyone full access; everyone waits on `Pending` where a result is required.

```lua
--!strict
local IntegrityService = game:GetService("IntegrityService")

-- Table of allowed levels. Never compare levels numerically: `Unknown` sorts below `None`.
local TRADE_LEVELS: { [Enum.PlayerDeviceTrustLevel]: boolean } = {
	[Enum.PlayerDeviceTrustLevel.Unknown] = true,
	[Enum.PlayerDeviceTrustLevel.Standard] = true,
	[Enum.PlayerDeviceTrustLevel.Full] = true,
}

local function canTrade(player: Player): boolean
	return TRADE_LEVELS[player.DeviceTrustLevel] == true
end

-- One callback per place; the last assignment wins. Server only.
IntegrityService.EvaluatePlayerDeviceTrust = function(player: Player, _measures: { [string]: any })
	if player.DeviceTrustLevel == Enum.PlayerDeviceTrustLevel.None then
		return { Action = Enum.PlayerDeviceTrustDecision.Reject, Message = "This mode needs a verifiable device." }
	end
	return { Action = Enum.PlayerDeviceTrustDecision.Accept }
end
```

Traps:

- **Rejecting from `PlayerAdded`:** the level usually isn't ready, so trusted players get rejected. Reject only from the callback.
- **The callback runs after the player can already play.** Don't start a ranked match while `Pending`. Players can also reach a place by joining a friend, so assign the callback in the restricted place too.
- It times out after about 10 s and then accepts. Returning nil, an invalid table or erroring also accepts. So it **fails open**: keep it fast, and re-check on the server before each sensitive action.
- It isn't called in Studio, or if the player leaves first.
- **Reading on the client:** other clients always see `Pending`, and the player's own client is untrustworthy. The server decides and tells the client which UI to show.
- **Measures** (`IsPlatformVersionSecure`, `IsPlatformBootSecure`, `IsAppSecure`) are `true` when not available too. Read the level first; use measures only to look for failures.
- **Testing:** keep policy in a function taking the level, skip the check when `RunService:IsStudio()`, and run the policy over `Enum.PlayerDeviceTrustLevel:GetEnumItems()`.

## Group role gates

`Player:GetRankInGroupAsync` and `Player:GetRoleInGroupAsync` are deprecated (723): they see only the highest public role, and players can now hold several. `Player:IsInGroupAsync` is not deprecated, and is still right for plain membership.

**Status of `GroupService:GetRolesInGroupAsync`:** release note 723 lists it as Pending, but the reference documents it and a DevForum post of [2026-07-08](https://devforum.roblox.com/t/feedback-on-playergetrankingroupasync/4656388) says it is re-enabled, with a recommendation to `pcall`. Not tested here. Confirm in a live server before shipping an admin gate on it.

```lua
--!strict
local GroupService = game:GetService("GroupService")

-- Role IDs, not ranks: docs say Rank no longer determines position in the hierarchy.
local function hasAnyRole(userId: number, groupId: number, allowedRoleIds: { [number]: boolean }): boolean
	local ok, result = pcall(function()
		return GroupService:GetRolesInGroupAsync(userId, groupId)
	end)
	if not ok or not result.IsMember then
		return false -- fail closed
	end
	for _, role in result.Roles do
		if allowedRoleIds[role.Id] then
			return true
		end
	end
	return false
end
```

- `Roles` is empty for a member with no non-base roles, so `IsMember == true` doesn't mean a role exists. Only public roles are included.
- Results are cached per user and group on each peer: **a revoked role can stay valid for a while**. For destructive admin commands, keep a server-side allowlist as well.
- Call from the server. A client's answer is not evidence.
- The parameter is typed `User`; `Player.User` and a numeric `UserId` are both accepted according to the release note signature and current Luau definitions. A client-side `Player.User` bug was fixed on [2026-08-28](https://devforum.roblox.com/t/groupservicegetrolesingroupasync-does-not-accept-the-user-property-on-the-client/4815411).

## Age check and verification

`Player.AgeChecked` is a read-only `Enum.AgeCheckStatus` (`Unchecked`, `Checked`). It reports whether the check was completed, not an age. `Player:PromptAgeCheck()` opens the flow from the server (any player) or a LocalScript (local player only). Both are in the reference with no beta tag; the announcement is a staff [DevForum update](https://devforum.roblox.com/t/introducing-the-us-18-devex-rate-earn-42-more-on-spend-from-18-us-players/4607091), dated 2026-10-12. It isn't in the release notes I checked. Not tested here.

- Non-yielding, returns nothing. Per that post: one prompt per player per 15 minutes per session; extra calls drop silently. Prompt on a deliberate player action, not on spawn.
- Console and VR don't support the flow yet, so **don't gate core gameplay** on it.
- A [2026-10-14 bug report](https://devforum.roblox.com/t/calling-playerpromptagecheck-in-studio-playtest-makes-age-check-required-appear-on-an-already-age-checked-account-in-account-switcher/4933802) says calling it in a Studio playtest leaves an "Age check required" notice on the account switcher. Community report; I saw no staff answer.

`Player:IsVerified()` (GA, server only; errors on the client, always `false` in Studio): account-level phone or ID signals. Exercise caution not to block all unverified users. The optional `Enum.VerifiedLevel` argument (`Low` default, `High`) is documented in the reference but listed Pending in release note 733, so confirm before depending on `High`.

## Sources

- https://create.roblox.com/docs/scripting/security/device-trust
- https://create.roblox.com/docs/reference/engine/classes/IntegrityService
- https://create.roblox.com/docs/reference/engine/classes/Players
- https://create.roblox.com/docs/reference/engine/classes/Player
- https://create.roblox.com/docs/reference/engine/classes/GroupService
- https://create.roblox.com/docs/players

Last verified: 2026-11-01 (Studio 0.745, release notes 745)
