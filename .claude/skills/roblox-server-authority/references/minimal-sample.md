# Server Authority: Minimal Sample

Belongs to the roblox-server-authority skill. Requires `Workspace.AuthorityMode = Server`.


A stamina-limited sprint. Uses only names in the 0.745 reference. It has been
type-checked, **never played**: run it before you rely on it.

Assumes `ReplicatedStorage.Inputs` (Folder) > `PlayContext` (InputContext) > `Sprint`
(InputAction, type `Bool`) with a binding.

```lua
--!strict
-- ReplicatedStorage/Simulation (ModuleScript), required by client AND server.
local RunService = game:GetService("RunService")

local WALK_SPEED = 16
local SPRINT_SPEED = 24
local MAX_STAMINA = 5
local REGEN_PER_SECOND = 0.5

local Simulation = {}

local function bindSprint(humanoid: Humanoid, sprint: InputAction): RBXScriptConnection
	return RunService:BindToSimulation(function(deltaTime: number)
		-- Stamina lives in an attribute: Luau locals would not roll back.
		local stored = humanoid:GetAttribute("Stamina")
		local stamina: number = if typeof(stored) == "number" then stored else MAX_STAMINA

		local sprinting = sprint:GetState() == true and stamina > 0
		local change = if sprinting then -deltaTime else deltaTime * REGEN_PER_SECOND

		humanoid:SetAttribute("Stamina", math.clamp(stamina + change, 0, MAX_STAMINA))
		humanoid.WalkSpeed = if sprinting then SPRINT_SPEED else WALK_SPEED
	end, Enum.StepFrequency.Hz60)
end

-- Instances are looked up here, outside the callback.
function Simulation.start(player: Player)
	local context = player:WaitForChild("Inputs"):WaitForChild("PlayContext")
	local sprint = context:WaitForChild("Sprint") :: InputAction
	local connection: RBXScriptConnection? = nil

	local function onCharacter(character: Model)
		if connection then
			connection:Disconnect()
		end
		connection = bindSprint(character:WaitForChild("Humanoid") :: Humanoid, sprint)
	end

	player.CharacterAdded:Connect(onCharacter)
	if player.Character then
		onCharacter(player.Character)
	end
end

return Simulation
```

```lua
--!strict
-- ServerScriptService/SimulationServer (Script)
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local Simulation = require(ReplicatedStorage:WaitForChild("Simulation"))
local inputs = ReplicatedStorage:WaitForChild("Inputs")

local function onPlayerAdded(player: Player)
	inputs:Clone().Parent = player
	Simulation.start(player)
end

Players.PlayerAdded:Connect(onPlayerAdded)
for _, player in Players:GetPlayers() do
	task.spawn(onPlayerAdded, player)
end
```

```lua
--!strict
-- StarterPlayer/StarterPlayerScripts/SimulationClient (LocalScript)
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local Simulation = require(ReplicatedStorage:WaitForChild("Simulation"))

-- Only the local player: other clients' inputs aren't forwarded here.
Simulation.start(Players.LocalPlayer)
```

UI that shows stamina reads the attribute in `RenderStepped`, never inside the callback.

Last verified: 2026-11-01 (Studio 0.745, release notes 745). Sources are listed in `../SKILL.md`.
