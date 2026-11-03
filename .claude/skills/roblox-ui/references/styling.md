# UI styling: tokens, themes, queries

Read this when building a token or theme system. Model: tokens are attributes of a
`StyleSheet`, a theme derives from tokens and maps semantic names to them, a design
sheet derives from one theme and holds the rules, and a `StyleLink` applies it to a
ScreenGui. Only one StyleSheet applies per tree. Themes that swap must define the
same tokens.

```lua
--!strict
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local function sheet(name: string): StyleSheet
	local s = Instance.new("StyleSheet")
	s.Name = name
	s.Parent = ReplicatedStorage
	return s
end

local function derive(child: StyleSheet, parent: StyleSheet)
	local d = Instance.new("StyleDerive")
	d.StyleSheet = parent
	d.Parent = child
end

local function rule(parent: Instance, selector: string, props: { [string]: any }): StyleRule
	local r = Instance.new("StyleRule")
	r.Selector = selector
	r:SetProperties(props)
	r.Parent = parent
	return r
end

-- Tokens: raw values.
local tokens = sheet("Tokens")
tokens:SetAttribute("Violet", Color3.fromHex("7A3CFF"))
tokens:SetAttribute("Radius", UDim.new(0, 12))

-- Theme: semantic names. Swap by retargeting the design sheet's StyleDerive.
local theme = sheet("ThemeDark")
derive(theme, tokens)
theme:SetAttribute("Accent", "$Violet")

local design = sheet("Design")
derive(design, theme)

local button = rule(design, "TextButton.Primary", {
	BackgroundColor3 = "$Accent",
	BorderSizePixel = 0,
})
button:SetPropertyTransitions({
	BackgroundTransparency = TweenInfo.new(0.12, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
})
rule(design, "TextButton.Primary:Hover", { BackgroundTransparency = 0.15 })
rule(design, "TextButton.Primary::UICorner", { CornerRadius = "$Radius" })

-- Built-in query: larger text when touch is the primary input. Nested under
-- the rule it refines, as in the official samples.
rule(button, "@PreferredInputTouch TextButton.Primary", { TextSize = 24 })

local function apply(screenGui: ScreenGui)
	local link = Instance.new("StyleLink")
	link.StyleSheet = design
	link.Parent = screenGui
end

return apply
```

Notes:

- Tags come from `CollectionService`; `.Primary` matches the tag `Primary`.
- `rule.SelectorError` (read-only) reports a bad selector. Check it when a rule
  doesn't apply.
- Properties influenced by a style show a warning marker in Properties; an instance
  override shows bold (`ui/styling`).
- Custom container query: a `::StyleQuery #Name` rule with
  `SetProperty("MinSize", Vector2.new(400, 0))` activates `@Name` selectors.
- `ui/styling/compatibility` lists the stylable classes and properties. Check it
  before assuming a property can be styled.
- `SetDefaultPropertyTransition(info)` equals `SetPropertyTransition("*", info)`.

Verified 2026-11-01 against `ui/styling/index`, `ui/styling/css-comparisons`, and
`reference/engine/classes/StyleRule`, `StyleQuery`. Compiled with luau-lsp; not run
in Studio.
