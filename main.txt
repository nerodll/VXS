local _ENV = (getgenv or getrenv or getfenv)()

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")

local Player = Players.LocalPlayer;

local DialogueEvent = ReplicatedStorage.BetweenSides.Remotes.Events.DialogueEvent;
local CombatEvent = ReplicatedStorage.BetweenSides.Remotes.Events.CombatEvent;
local ToolEvent = ReplicatedStorage.BetweenSides.Remotes.Events.ToolsEvent;
local QuestsNpcs = workspace.IgnoreList.Int.NPCs.Quests;
local Enemys = workspace.Playability.Enemys;

local QuestsDecriptions = require(ReplicatedStorage.MainModules.Essentials.QuestDescriptions)

local EnemiesFolders = {}
local QuestsData = {}
local CFrameAngle = CFrame.Angles(math.rad(-90), 0, 0)

local GetCurrentQuest do
	QuestsData.QuestsList = {}
	QuestsData.QuestsNPCs = {}
	QuestsData.EnemyList = {}
	
	table.clear(QuestsData.QuestsList)
	
	local CurrentQuest = nil;
	local CurrentLevel = -1;
	
	for _, QuestData in QuestsDecriptions do
		if QuestData.Goal <= 1 then continue end
		
		table.insert(QuestsData.QuestsList, {
			Level = QuestData.MinLevel;
			Target = QuestData.Target;
			NpcName = QuestData.Npc;
			Id = QuestData.Id;
		})
	end
	
	table.sort(QuestsData.QuestsList, function(a, b)
		return a.Level > b.Level;
	end)
	
	GetCurrentQuest = function()
		local Level = tonumber(Player.PlayerGui.MainUI.MainFrame.StastisticsFrame.LevelBackground.Level.Text);
		
		if Level == CurrentLevel then
			return CurrentQuest;
		end
		
		for _, QuestData in QuestsData.QuestsList do
			if QuestData.Level <= Level then
				CurrentLevel, CurrentQuest = Level, QuestData
				return QuestData
			end
		end
	end
end

local Settings = {
	ClickV2 = false;
	TweenSpeed = 125;
	SelectedTool = "CombatType";
}

local EquippedTool = nil;

local Connections = _ENV.rz_connections or {} do
	_ENV.rz_connections = Connections
	
	for i = 1, #Connections do
		Connections[i]:Disconnect()
	end
	
	table.clear(Connections)
end

local function IsAlive(Character)
	if Character then
		local Humanoid = Character:FindFirstChildOfClass("Humanoid");
		return Humanoid and Humanoid.Health > 0;
	end
end

local BodyVelocity do
	BodyVelocity = Instance.new("BodyVelocity")
	BodyVelocity.Velocity = Vector3.zero
	BodyVelocity.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
	BodyVelocity.P = 1000
	
	if _ENV.tween_bodyvelocity then
		_ENV.tween_bodyvelocity:Destroy()
	end
	
	_ENV.tween_bodyvelocity = BodyVelocity
	
	local CanCollideObjects = {}
	
	local function AddObjectToBaseParts(Object)
		if Object:IsA("BasePart") and Object.CanCollide then
			table.insert(CanCollideObjects, Object)
		end
	end
	
	local function RemoveObjectsFromBaseParts(BasePart)
		local index = table.find(CanCollideObjects, BasePart)
		
		if index then
			table.remove(CanCollideObjects, index)
		end
	end
	
	local function NewCharacter(Character)
		table.clear(CanCollideObjects)
		
		for _, Object in Character:GetDescendants() do AddObjectToBaseParts(Object) end
		Character.DescendantAdded:Connect(AddObjectToBaseParts)
		Character.DescendantRemoving:Connect(RemoveObjectsFromBaseParts)
	end
	
	table.insert(Connections, Player.CharacterAdded:Connect(NewCharacter))
	task.spawn(NewCharacter, Player.Character)
	
	local function NoClipOnStepped(Character)
		if _ENV.OnFarm then
			for i = 1, #CanCollideObjects do
				CanCollideObjects[i].CanCollide = false
			end
		elseif Character.PrimaryPart and not Character.PrimaryPart.CanCollide then
			for i = 1, #CanCollideObjects do
				CanCollideObjects[i].CanCollide = true
			end
		end
	end
	
	local function UpdateVelocityOnStepped(Character)
		local BasePart = Character:FindFirstChild("UpperTorso")
		local Humanoid = Character:FindFirstChild("Humanoid")
		local BodyVelocity = _ENV.tween_bodyvelocity
		
		if _ENV.OnFarm and BasePart and Humanoid and Humanoid.Health > 0 then
			if BodyVelocity.Parent ~= BasePart then
				BodyVelocity.Parent = BasePart
			end
		elseif BodyVelocity.Parent then
			BodyVelocity.Parent = nil
		end
		
		if BodyVelocity.Velocity ~= Vector3.zero and (not Humanoid or not Humanoid.SeatPart or not _ENV.OnFarm) then
			BodyVelocity.Velocity = Vector3.zero
		end
	end
	
	table.insert(Connections, RunService.Stepped:Connect(function()
		local Character = Player.Character;
		
		if IsAlive(Character) then
			UpdateVelocityOnStepped(Character)
			NoClipOnStepped(Character)
		end
	end))
end

local PlayerTP do
	local TweenCreator = {} do
		TweenCreator.__index = TweenCreator
		
		local tweens = {}
		local EasingStyle = Enum.EasingStyle.Linear
		
		function TweenCreator.new(obj, time, prop, value)
			local self = setmetatable({}, TweenCreator)
			
			self.tween = TweenService:Create(obj, TweenInfo.new(time, EasingStyle), { [prop] = value })
			self.tween:Play()
			self.value = value
			self.object = obj
			
			if tweens[obj] then
				tweens[obj]:destroy()
			end
			
			tweens[obj] = self
			return self
		end
		
		function TweenCreator:destroy()
			self.tween:Pause()
			self.tween:Destroy()
			
			tweens[self.object] = nil
			setmetatable(self, nil)
		end
		
		function TweenCreator:stopTween(obj)
			if obj and tweens[obj] then
				tweens[obj]:destroy()
			end
		end
	end
	
	local function TweenStopped()
		if not BodyVelocity.Parent and IsAlive(Player.Character) then
			TweenCreator:stopTween(Player.Character:FindFirstChild("HumanoidRootPart"))
		end
	end
	
	local lastCFrame = nil;
	local lastTeleport = 0;
	
	PlayerTP = function(TargetCFrame)
		if not IsAlive(Player.Character) or not Player.Character.PrimaryPart then
			return false
		elseif (tick() - lastTeleport) <= 1 and lastCFrame == TargetCFrame then
			return false
		end
		
		local Character = Player.Character
		local Humanoid = Character.Humanoid
		local PrimaryPart = Character.PrimaryPart
		
		if Humanoid.Sit then Humanoid.Sit = false return end
		
		lastTeleport = tick()
		lastCFrame = TargetCFrame
		_ENV.OnFarm = true
		
		local teleportPosition = TargetCFrame.Position;
		local Distance = (PrimaryPart.Position - teleportPosition).Magnitude;
		
		if Distance < Settings.TweenSpeed then
			PrimaryPart.CFrame = TargetCFrame
			return TweenCreator:stopTween(PrimaryPart)
		end
		
		TweenCreator.new(PrimaryPart, Distance / Settings.TweenSpeed, "CFrame", TargetCFrame)
	end
	
	table.insert(Connections, BodyVelocity:GetPropertyChangedSignal("Parent"):Connect(TweenStopped))
end

local CurrentTime = workspace:GetServerTimeNow()

local function DealDamage(Enemies)
	CurrentTime = workspace:GetServerTimeNow()
	
	CombatEvent:FireServer("DealDamage", {
		CallTime = CurrentTime;
		DelayTime = workspace:GetServerTimeNow() - CurrentTime;
		Combo = 1;
		Results = Enemies;
	})
end

local function GetMobFromFolder(Folder, EnemyName)
	for _, Enemy in Folder:GetChildren() do
		if Enemy:GetAttribute("Respawned") and Enemy:GetAttribute("Ready") then
			if Enemy:GetAttribute("OriginalName") == EnemyName then
				return Enemy;
			end
		end
	end
end

local function GetClosestEnemy(EnemyName)
	local EnemyFolder = EnemiesFolders[EnemyName]
	
	if EnemyFolder then
		return GetMobFromFolder(EnemyFolder, EnemyName)
	end
	
	local Islands = Enemys:GetChildren()
	
	for i = 1, #Islands do
		local Enemies = Islands[i]:GetChildren()
		
		for x = 1, #Enemies do
			if Enemies[i]:GetAttribute("OriginalName") == EnemyName then
				EnemiesFolders[EnemyName] = Islands[i]
				return GetMobFromFolder(Islands[i], EnemyName)
			end
		end
	end
end

local function BringEnemies(Enemies, Target)
	for _, Enemy in Enemies do
		local RootPart = Enemy:FindFirstChild("HumanoidRootPart")
		
		if RootPart then
			RootPart.Size = Vector3.one * 30
			RootPart.CFrame = Target
		end
	end
	
	pcall(sethiddenproperty, Player, "SimulationRadius", math.huge)
end

local function IsSelectedTool(Tool)
	return Tool:GetAttribute(Settings.SelectedTool)
end

local function EquipCombat(Activate)
	if not IsAlive(Player.Character) then return end
	
	if EquippedTool and IsSelectedTool(EquippedTool) then
		if Activate then
			EquippedTool:Activate()
		end
		
		if EquippedTool.Parent == Player.Backpack then
			Player.Character.Humanoid:EquipTool(EquippedTool)
		elseif EquippedTool.Parent ~= Player.Character then
			EquippedTool = nil;
		end
		return nil
	end
	
	local Equipped = Player.Character:FindFirstChildOfClass("Tool")
	
	if Equipped and IsSelectedTool(Equipped) then
		EquippedTool = Equipped
		return nil;
	end
	
	for _, Tool in Player.Backpack:GetChildren() do
		if Tool:IsA("Tool") and IsSelectedTool(Tool) then
			EquippedTool = Tool
			return nil;
		end
	end
end

local function HasQuest(EnemyName)
	local QuestFrame = Player.PlayerGui.MainUI.MainFrame.CurrentQuest;
	return QuestFrame.Visible and QuestFrame.Goal.Text:find(EnemyName);
end

local function TakeQuest(QuestName, QuestId)
	local Npc = QuestsNpcs:FindFirstChild(QuestName, true)
	local RootPart = Npc and Npc.PrimaryPart
	
	if RootPart then
		DialogueEvent:FireServer("Quests", { ["NpcName"] = QuestName; ["QuestName"] = QuestId })
		PlayerTP(RootPart.CFrame)
	end
end

local Libary = loadstring(game:HttpGet("https://raw.githubusercontent.com/tlredz/Library/refs/heads/main/V5/Source.lua"))()
local Window = Libary:MakeWindow({ "Anonymous", "by NightShadow", "rz-VoxSeas.json"})

local MainTab = Window:MakeTab({ "Farm", "Home" })
local ConfigTab = Window:MakeTab({ "Config", "Settings" })

do
	MainTab:AddSection("Farming")
	MainTab:AddToggle({"Auto Farm Level", false, function(Value)
		_ENV.OnFarm = Value
		
		while task.wait() and _ENV.OnFarm do
			local CurrentQuest = GetCurrentQuest()
			if not CurrentQuest then continue end
			
			if not HasQuest(CurrentQuest.Target) then
				TakeQuest(CurrentQuest.NpcName, CurrentQuest.Id)
				continue
			end
			
			local Enemy = GetClosestEnemy(CurrentQuest.Target)
			if not Enemy then continue end
			
			local HumanoidRootPart = Enemy:FindFirstChild("HumanoidRootPart")
			
			if HumanoidRootPart then
				if not HumanoidRootPart:FindFirstChild("BodyVelocity") then
					local BV = Instance.new("BodyVelocity", HumanoidRootPart)
					BV.Velocity = Vector3.zero
					BV.MaxForce = Vector3.one * math.huge
				end
				
				HumanoidRootPart.Size = Vector3.one * 35
				HumanoidRootPart.CanCollide = false
				
				EquipCombat(true)
				DealDamage({ Enemy })
				PlayerTP((HumanoidRootPart.CFrame + Vector3.yAxis * 10) * CFrameAngle)
			end
		end
	end})
end

do
	ConfigTab:AddToggle({"Click V2", false, {Settings, "ClickV2"} })
	ConfigTab:AddToggle({"Tween Speed", 50, 200, 10, 125, {Settings, "TweenSpeed"} })
	ConfigTab:AddDropdown({"Select Tool", {"CombatType"}, "CombatType", {Settings, "SelectedTool"} })
end
