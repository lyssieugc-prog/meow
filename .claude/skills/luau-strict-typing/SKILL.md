---
description: Luau language and strict typing for Roblox — which type solver a place really uses (a --!strict place stays on the OLD solver by default), --!strict/--!nonstrict/--!nocheck semantics, narrowing Instance? results, casts, forward declarations, typed remotes, OOP metatable types, generics and f<<T>> instantiation, string require, const, if local (Beta), read-only types, type functions (keyof, index, setmetatable, type function), function attributes, and fixing type-checker errors. Use when writing or reviewing any Luau code, enabling --!strict, choosing new Luau syntax, or resolving Script Analysis or luau-lsp warnings.
---

# Luau Strict Typing

New Luau files start with `--!strict`. The checker catches the bugs that dominate
Roblox crash logs: nil indexing, wrong payload shapes, typo'd members. The solver runs
in Studio only, so it never validates network input.

The failure this skill exists to prevent: writing syntax that one solver or one
tool accepts and the place you ship to rejects. **A clean `luau-lsp analyze` does
not prove a feature works in Studio or a live game** (see below).

## Which solver is this place on? Check before using solver-only types

There are two engines, chosen per place by `Workspace.UseNewLuauTypeSolver`
(`Enum.RolloutState`: `Default`, `Disabled`, `Enabled`; Properties window, Scripting
category; tagged `NotScriptable`, so scripts and MCP Luau can't read or set it).

| Place | With `Default` | Opt in or out |
| --- | --- | --- |
| Only `--!nocheck` / `--!nonstrict` scripts | **New solver** | `Disabled` opts out |
| Contains `--!strict` scripts | **Old solver** | `Enabled` opts in, for everyone editing the place |

Source: [General Release post](https://devforum.roblox.com/t/general-release-luau%E2%80%99s-new-type-solver/4084991)
(20 Nov 2025; Studio Beta removed 7 Jan 2026; old engine kept "through 2026"; no
removal date found). Live flags fetched 2026-11-01 still agree
(`DFFlagStudioLuauNewSolverOptOutPlacesWithStrictScripts=True`). Not tested: how fast
a place flips when its first strict script appears. Set the property explicitly rather
than depending on that.

Consequences, tested with luau-lsp 1.70.1 under both solvers:

| Works on both solvers | New solver only (old reports the error shown) |
| --- | --- |
| `typeof(setmetatable(...))`, `const`, `f<<T>>`, generics, unions, `@native`, `@deprecated` | `keyof`, `rawkeyof`, `index`, `rawget`, `setmetatable<>`, `getmetatable<>` (`Unknown type 'keyof'`) |
| | `read`/`write` modifiers (`read keyword is illegal here`) |
| | `type function` (`This syntax is not supported`) |

So: in a strict place on `Default`, write code valid on both, unless you confirm
`Enabled`. In a Rojo project, set `"Workspace": { "$properties": { "UseNewLuauTypeSolver":
"Enabled" } }` (built to `.rbxlx` and confirmed as token `2`; `rojo serve` sync not tested).

For a new project, prefer `Enabled`, since the old engine is on its way out (this is a
recommendation, not a Roblox rule). For an existing strict codebase, analyze under both
solvers first and tell the owner before flipping it.

## Check your work with the analyzer

```sh
rojo sourcemap default.project.json -o sourcemap.json
luau-lsp analyze --platform roblox --sourcemap sourcemap.json \
  --definitions:@roblox=globalTypes.d.luau --flag:LuauSolverV2=true src/
```

- Definitions: `https://raw.githubusercontent.com/JohnnyMorganz/luau-lsp/main/scripts/globalTypes.d.luau`.
- 1.70.1 defaults to the **old** solver. Pass `--flag:LuauSolverV2=true` to match an
  `Enabled` place, and run both if you don't know.
- `analyze` turns on all Luau FFlags by default, so it accepts `if local` and `export const`,
  which Studio gates. With `--no-flags-enabled` both become syntax errors.
- Run from the project root (the sourcemap resolves relative to it). The rokit shim only
  runs inside a project with `luau-lsp` in `rokit.toml`.

## Modes

Put the directive on the first line. `Workspace.LuauTypeCheckMode` sets the default for
scripts with none; luau.org lists `--!nonstrict` as the language default.

| Directive | Behavior (tested) |
| --- | --- |
| `--!strict` | Tracks inferred and annotated types. Unknown members, nil access and mismatches all report. |
| `--!nonstrict`, new solver | Redesigned: reports definite runtime errors only. Flagged `math.abs("x")`; did **not** flag `local p: Part = 1`, `p.Nope` or `"hello" + 1`. Do not treat silence as safety. |
| `--!nonstrict`, old solver | Unknown types become `any`, annotated values are still checked: flagged `p.Nope`. |
| `--!nocheck` | No type errors at all. For generated code only. |

## Narrowing optionals

`FindFirstChild`, `FindFirstChildOfClass`, `FindFirstChildWhichIsA` and
`FindFirstAncestorOfClass` return `Instance?`. `Player.Character` can be nil before spawn
(luau-lsp types it `Model?`; the YAML lists `Model`).

```lua
--!strict
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local function rootOf(player: Player): BasePart?
	local character = player.Character
	if not character then
		return nil
	end
	local root = character:FindFirstChild("HumanoidRootPart")
	if not root or not root:IsA("BasePart") then
		return nil
	end
	return root -- narrowed to BasePart
end

-- assert also refines: use it when absence is a programmer error.
local remotes = ReplicatedStorage:FindFirstChild("Remotes")
assert(remotes, "Remotes folder must exist")
print(remotes.Name, rootOf(Players.LocalPlayer))
```

- `:IsA("Class")` checks at runtime and narrows. Prefer it to `::`, which checks nothing.
- `WaitForChild(name)` is `Instance`. `WaitForChild(name, timeout)` is `Instance?` in the
  luau-lsp definitions, matching the reference ("will time out … and return `nil`"), so
  guard it. Without a timeout it can yield forever and warns after 5 seconds.
- luau-lsp also specializes `FindFirstChildOfClass("Humanoid")` to `Humanoid?`. The
  reference says `Instance?` and Studio's behavior is unverified, so add `IsA` if you
  need the members and portability.

## Casts

- A cast is allowed only when one side is a subtype of the other, or `any`. Otherwise:
  `Cannot cast 'number' into 'boolean' because the types are unrelated`.
- `x :: any :: T` is a **syntax error**. The escape hatch needs parentheses:
  `(x :: any) :: T`. If you need it, your types are lying somewhere.
- A cast of a multi-value expression keeps only the first value.
- A downcast like `x :: PlayerGui` checks nothing. Use it only with out-of-band certainty.

## Forward declarations

```lua
--!strict
local openMenu: (menuId: string) -> ()
local closeMenu: () -> ()

closeMenu = function()
	openMenu("main")
end

openMenu = function(menuId: string)
	print(menuId)
	closeMenu()
end
```

Assign with `name = function`, not `local function name`, which makes a new local. The
symptom is a `LocalShadow` lint and an `UninitializedLocal` lint on the forward-declared one.

## Remote payloads

Types are erased, so anything from a client is `unknown` until checked. Annotate
parameters `unknown`, then refine with `typeof`. No cast is needed once refined (the
full rules are in the roblox-security skill):

```lua
--!strict
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local remote = ReplicatedStorage:WaitForChild("BuyItem") :: RemoteEvent

local function handleBuy(_player: Player, _itemId: string, _quantity: number) end

remote.OnServerEvent:Connect(function(player: Player, payload: unknown)
	if typeof(payload) ~= "table" then
		return
	end
	local request = payload :: { [string]: unknown }
	local itemId, quantity = request.itemId, request.quantity
	if typeof(itemId) ~= "string" or typeof(quantity) ~= "number" then
		return
	end
	handleBuy(player, itemId, quantity)
end)
```

Keep boundary types in one shared module (`export type`), consumed as `Types.Item`.

## OOP modules

`typeof(setmetatable(...))` is still luau.org's documented form and passes on both
solvers. `setmetatable<Data, typeof(Class)>` is the new-solver-only alternative.

```lua
--!strict
local Cooldown = {}
Cooldown.__index = Cooldown

type CooldownData = { duration: number, lastUsed: number }
export type Cooldown = typeof(setmetatable({} :: CooldownData, Cooldown))

function Cooldown.new(duration: number): Cooldown
	return setmetatable({ duration = duration, lastUsed = 0 }, Cooldown)
end

-- Dot syntax with an annotated self: luau.org calls this annotation required.
function Cooldown.ready(self: Cooldown): boolean
	return os.clock() - self.lastUsed >= self.duration
end

return Cooldown
```

Callers still write `cooldown:ready()`. **Do not define methods with `function C:m()`**:
`self` is inferred per method, not as your class. Tested, `self.n += by` fails on both
solvers (old: `Unknown type used in + operation; consider adding a type annotation to
'self'`; new: read-only and `unknown` errors, then the call site breaks too). Shared self
types are only planned, per luau.org.

## Generics

```lua
--!strict
local function map<T, U>(list: { T }, fn: (T) -> U): { U }
	local out: { U } = table.create(#list)
	for i, value in list do
		out[i] = fn(value)
	end
	return out
end

-- Explicit instantiation (702, 704): for inference that is ambiguous or too narrow.
local function array<T>(): { T }
	return {}
end
local numbers = array<<number>>()
table.insert(numbers, 1)

print(map(numbers, tostring))
```

- Generic functions can't have defaults (`<T = string>` is a syntax error). Generic type
  aliases can.
- Bodies of generic functions are checked: `f(nil)` where `f: (T) -> T` errors (739).
- `keyof`, `index` and friends are type functions written with `<>`, not keywords, and
  they need the new solver: `keyof<T>`, `index<T, "a">`.
  [Read this for type functions and read-only types](references/new-solver-types.md).

## require

Static paths are what give you types: `require(ReplicatedStorage.Shared.Util)` and string
paths both typecheck. A computed one gives `Unknown require: unsupported path`; if it is
truly dynamic, cast with `require(x) :: any` and accept losing the module's types. The
same cast breaks a cyclic dependency.

String require (`Global.LuaGlobals.require`):

| Prefix | Starts at |
| --- | --- |
| `./` | `script.Parent` |
| `../` | `script.Parent.Parent` |
| `@self/` | `script` |
| `@game/` | `game` |

- **It does not wait.** If the module isn't there yet, the call throws. On the client,
  replication order is not guaranteed, so wait first or require from a path that's loaded.
- luau-lsp needs `--sourcemap` to resolve `@game/`, and 1.70.1 reports
  `Unknown require` for `@rbx/PlayerModule`. That is a tool gap, not a bug in your code.

## Language features: what you may ship

| Feature | Status | Notes |
| --- | --- | --- |
| `const` (locals, `const function`) | **GA**: in the [Variables guide](https://create.roblox.com/docs/luau/variables), RFC Implemented; 711's row still reads Pending, but `FFlagLuauConst2=True` for Studio and desktop client on 2026-11-01 | Binding only, not contents: pair with `table.freeze`. Must be initialized. Shadowing allowed. Contextual: `local const = 1` still works. |
| `f<<T, U>>` | GA (702, 704) | Both solvers. |
| `{ read [K]: V }`, `read`/`write` props | GA (721); new solver only | See reference file. |
| `@native`, `@deprecated`, `@[deprecated {use = "...", reason = "..."}]` | GA ([Attributes](https://luau.org/attributes/)) | `@native` is not recursive: inner functions need their own. Annotate `Vector3` params; wrong arg types cost speed. Measure before using. |
| `math.isnan`, `isinf`, `isfinite` | GA (701) | |
| Yielding inside iterator functions | GA (722) | |
| `pcall`/`xpcall` recursion to 20000, from 200 | GA (693) | Each call still counts toward the Luau call stack limit. |
| `require("@rbx/PlayerModule")` | GA (735) if opted in | Needs `Workspace.PlayerScriptsUseInputActionSystem`. |
| `if local` / `if const`, statement and expression | **Studio Beta** ([4918297](https://devforum.roblox.com/t/beta-if-local-statements-and-expressions/4918297), 6 Oct 2026): Beta Features, "If Local/Const Bindings" | Do not ship without saying so. Whether it runs in live games is not stated, and the client flag list has no matching flag. One name and value only: `if local ok, r = pcall(f)` and `;` chains are rejected, `while local` not yet. Binding scoped to its `then` block; taken when truthy (RFC). |
| `export local/const/function` | **Off**: `FFlagLuauExportValueSyntax4=False`; 725 (highlighting) Pending | Don't use. Keep `return Module`. |
| Aliases named like builtins (`type number = ...`) | Pending (708) for new solver | luau-lsp already rejects: `Redefinition of type 'number'`. Avoid. Shadowing engine types like `Vector2` still passes. |
| Literal `const`/immutable locals not counting toward the 200 local/upvalue limits; locals freed after last use | Pending (743) | Don't rely on it yet. |
| Completion for `require(inst:GetAttribute("ModuleRef"))` | Pending (738) | Editor feature; not in the docs. |
| `@checked` | Undocumented | Absent from the Attributes page, which says it lists all. luau-lsp parses it. Don't use. |

To recheck a Pending item, read its row in the current release notes. Flags at
`https://clientsettingscdn.roblox.com/v2/settings/application/PCStudioApp` are a hint, not proof.

## Common errors

Wording is from luau-lsp 1.70.1. **Roblox rewrote these messages in 703 and 734, and Studio may
differ, so match on meaning, not on exact text.**

| Message | Meaning and fix |
| --- | --- |
| `Value of type 'Model?' could be nil` | Guard, `assert` or restructure. Don't sprinkle `::`. |
| `Key 'Humanoid' not found in external type 'Model'` | Children aren't properties in the type system. `FindFirstChild` plus `IsA`. |
| `Type 'Cooldown' does not have key 'bad'` | Typo, or a method missing from the class table. |
| `Cannot add property 'x' to table 'Item'`, `Key 'x' not found in table 'Item'` | Annotated tables are sealed. Add it to the type, as optional if it can be absent. |
| `Table type ... not compatible with type 'Item' because the former is missing fields` | Fill the fields or mark them `?`. Build the whole literal at once. |
| `Expected this to be 'number', but got 'string'` | Real mismatch: fix the value or the annotation. |
| `Cannot call a value of type number` | Name collision or wrong variable. |
| `Variable 'X' is constant and may not be reassigned`, `Missing initializer in const declaration` | `const` rules, reported as syntax errors. Use `local`, or initialize. |
| `Recursive type being used with different parameters.` | `type L<T> = { next: L<{T}>? }`. Pass the same parameters on recursion. Plain `type Node = { next: Node? }` is fine. |
| `Unknown type 'keyof'`, `read keyword is illegal here`, `This syntax is not supported` | **You're on the old solver.** Opt the place in or use a both-solver form. |
| `Property x of table 'P' is read-only` | Writing through a `read` type: intended by that type. |
| Hover shows `never`, with no error | Contradictory refinement: `typeof(x) == "string"` on a `number` reports nothing. Check the logic. |

## Adopting in an untyped codebase

Put `--!strict` on new files first, never big-bang; convert shared modules next, since their
types flow outward; fix the real bugs a conversion reveals before suppressing anything. Per the
post above, strict scripts opt a `Default` place out of the new solver (timing untested), so
set `UseNewLuauTypeSolver` deliberately.

When you (Claude) write a new Luau file, default to `--!strict` with full annotations on
public functions, without asking. Annotate `self` and use dot-defined methods. Stick to the
both-solver column unless you've confirmed the place is `Enabled`. Never ship Beta or Pending
syntax without telling the owner its status. Run the analyzer when it's available, report
which solver you ran, and never quote a clean run as proof that Studio accepts new syntax.

## Sources

- https://create.roblox.com/docs/luau/type-checking , https://create.roblox.com/docs/luau/variables , https://create.roblox.com/docs/luau/native-code-gen
- https://create.roblox.com/docs/reference/engine/classes/Workspace , https://create.roblox.com/docs/reference/engine/classes/Instance , https://create.roblox.com/docs/reference/engine/enums/RolloutState , https://create.roblox.com/docs/reference/engine/globals/LuaGlobals
- https://luau.org/types/ , https://luau.org/types/object-oriented-programs/ , https://luau.org/types/generics/ , https://luau.org/attributes/ , https://luau.org/syntax/
- https://rfcs.luau.org/const-keyword.html , https://rfcs.luau.org/explicit-type-parameter-instantiation.html
- https://devforum.roblox.com/t/general-release-luau%E2%80%99s-new-type-solver/4084991
- https://devforum.roblox.com/t/beta-if-local-statements-and-expressions/4918297

Last verified: 2026-11-01 (Studio 0.745, release notes 745). Samples analyzed with luau-lsp 1.70.1, old and new solver.
