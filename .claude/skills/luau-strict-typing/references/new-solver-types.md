# New-solver-only types

Read this when you need `keyof`, `index`, `read`/`write` or `type function`. **All of it
needs `Workspace.UseNewLuauTypeSolver = Enabled` in a place with strict scripts**, and fails
on the old solver (see SKILL.md). Samples passed luau-lsp 1.70.1 with
`--flag:LuauSolverV2=true`; none were run in Studio.

## Built-in type functions

They use type-argument syntax, and they are not keywords.

```lua
--!strict
type Stats = { health: number, name: string }

local key: keyof<Stats> = "health" -- "health" | "name"
local rawKey: rawkeyof<Stats> = "name" -- ignores the metatable's __index
local health: index<Stats, "health"> = 100 -- number
local anyValue: index<Stats, keyof<Stats>> = "x" -- number | string
local rawHealth: rawget<Stats, "health"> = 100

local Meta = { __index = { level = 1 } }
type WithMeta = setmetatable<Stats, typeof(Meta)>
local meta: getmetatable<WithMeta> = Meta

print(key, rawKey, health, anyValue, rawHealth, meta)
```

## Read-only and write-only

```lua
--!strict
type Config = { read maxPlayers: number, write lastError: string }

local function sum(values: { read [number]: number }): number
	local total = 0
	for _, value in values do
		total += value
	end
	return total
end

local config: Config = { maxPlayers = 8, lastError = "" }
print(config.maxPlayers, sum({ 1, 2 }))
```

- `{ read [K]: V }` landed in 721, the `{ read V }` array form parses too, and the type
  function runtime gained support in 727. A write through it reports
  `Property ... is read-only`.
- Compile-time only: for runtime immutability use `table.freeze`.
- Fixes for read-only indexer unification against invariant indexers are Pending (738).

## User-defined type functions

They run during analysis, on types. Environment: the `types` library, `assert`, `error`,
`print`, `next`, `ipairs`, `pairs`, `select`, `unpack`, `getmetatable`, `setmetatable`,
`rawget`, `rawset`, `rawlen`, `raweq`, `tonumber`, `tostring`, `type`, `typeof`, and the
`math`, `table`, `string`, `bit32`, `utf8`, `buffer` libraries. `pcall`/`xpcall` are also
allowed since 734, though luau.org's list hasn't caught up.

```lua
--!strict
type function Partial(source)
	assert(source:is("table"), "Partial expects a table type")
	local result = types.newtable()
	for key, property in source:properties() do
		if property.read then
			result:setproperty(key, types.optional(property.read))
		end
	end
	return result
end

type function IsString(candidate)
	return types.singleton(candidate:issubtypeof(types.string)) -- 727
end

type function TagOf(candidate)
	return types.singleton(candidate.tag)
end

type Profile = { name: string, score: number }
local patch: Partial<Profile> = { score = 1 }
local yes: IsString<"hi"> = true
local tag: TagOf<Part> = "extern"
print(patch, yes, tag)
```

- **`extern`, not `class`:** engine types carry the tag `extern` since 706. Code testing
  `"class"` no longer matches them.
- `==` on types is syntactic: `true | false` is not equal to `boolean`. Use `issubtypeof`.
- `types.unionof`/`intersectionof` flatten nested unions and intersections (703).
- You can't mutate a top-level alias from a type function: it raises an error (705).
- `export type function` works across modules (tested: `TF.Tag<Part>`). 732 fixed exports
  degrading to `*error-type*`.
- Full API: https://luau.org/types-library/

Sources: https://luau.org/types/type-functions/ , https://luau.org/types-library/ ,
https://rfcs.luau.org/keyof-type-operator.html , https://rfcs.luau.org/index-type-operator.html ,
https://rfcs.luau.org/rawget-type-operator.html , https://rfcs.luau.org/metatable-type-functions.html ,
https://rfcs.luau.org/user-defined-type-functions.html , https://rfcs.luau.org/udtf-is-extern.html

Last verified: 2026-11-01 (Studio 0.745, release notes 745).
