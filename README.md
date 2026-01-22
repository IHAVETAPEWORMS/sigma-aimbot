-- open source
--[[
	WARNING: Heads up! This script has not been verified by ScriptBlox. Use at your own risk!
]]
local repo = "https://raw.githubusercontent.com/deividcomsono/Obsidian/refs/heads/main/"
local Library = loadstring(game:HttpGet(repo .. "Library.lua"))()

-- [[ SINGLETON CHECK ]] --
if getgenv().ObsidianV1Running then
    Library:Notify("Already Running!", 5)
    return 
end
getgenv().ObsidianV1Running = true
-- [[ END CHECK ]] --

local ThemeManager = loadstring(game:HttpGet(repo .. "addons/ThemeManager.lua"))()
local SaveManager = loadstring(game:HttpGet(repo .. "addons/SaveManager.lua"))()
task.wait(0.5)
local Options = Library.Options
local Toggles = Library.Toggles

-- ===============================================
-- === LOGIC VARIABLES ===
-- ===============================================
local PerceivedPos = nil
local PerceivedVel = Vector3.new()
local LastUpdateTimeX = 0
local LastUpdateTimeY = 0
local ReactionDelayX = 0.5
local ReactionDelayY = 0.5
local JustLocked = true
local CurrentLockedTarget = nil
-- Global Target
local Target = nil
-- Script State
local ScriptState = { ClosestHitPart = nil }

-- Silent Aim Smoothness
local SilentAimSmoothedPos = nil
local SilentAimLastTargetPart = nil
local SilentAimInitialized = false

-- Triggerbot
local TriggerNextShot = 0 
local Clicked = false

-- Fly Logic
local FlyKeysPressed = {} 
local FlyBV, FlyBG

-- Services
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local mouse = LocalPlayer:GetMouse() 
local Camera = workspace.CurrentCamera
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Debris = game:GetService("Debris")
local TweenService = game:GetService("TweenService") 
local Lighting = game:GetService("Lighting")

-- ===============================================
-- === VISUALS (SKYBOX) ===
-- ===============================================
local Skyboxes = {}
local Visuals = {}

function Visuals:NewSky(Data)
    Skyboxes[Data.Name] = {
        SkyboxBk = Data.SkyboxBk, SkyboxDn = Data.SkyboxDn, SkyboxFt = Data.SkyboxFt,
        SkyboxLf = Data.SkyboxLf, SkyboxRt = Data.SkyboxRt, SkyboxUp = Data.SkyboxUp,
        MoonTextureId = Data.Moon or "rbxasset://sky/moon.jpg", SunTextureId = Data.Sun or "rbxasset://sky/sun.jpg"
    }
end

function Visuals:SwitchSkybox(Name)
    local OldSky = Lighting:FindFirstChildOfClass("Sky")
    if OldSky then OldSky:Destroy() end
    if Skyboxes[Name] then
        local Sky = Instance.new("Sky", Lighting)
        for Index, Value in pairs(Skyboxes[Name]) do Sky[Index] = Value end
    end
end

if Lighting:FindFirstChildOfClass("Sky") then
    local OldSky = Lighting:FindFirstChildOfClass("Sky")
    Visuals:NewSky({ Name = "Game's Default Sky", SkyboxBk = OldSky.SkyboxBk, SkyboxDn = OldSky.SkyboxDn, SkyboxFt = OldSky.SkyboxFt, SkyboxLf = OldSky.SkyboxLf, SkyboxRt = OldSky.SkyboxRt, SkyboxUp = OldSky.SkyboxUp })
end

Visuals:NewSky({ Name = "Sunset", SkyboxBk = "rbxassetid://600830446", SkyboxDn = "rbxassetid://600831635", SkyboxFt = "rbxassetid://600832720", SkyboxLf = "rbxassetid://600886090", SkyboxRt = "rbxassetid://600833862", SkyboxUp = "rbxassetid://600835177" })
Visuals:NewSky({ Name = "Space", SkyboxBk = "http://www.roblox.com/asset/?id=166509999", SkyboxDn = "http://www.roblox.com/asset/?id=166510057", SkyboxFt = "http://www.roblox.com/asset/?id=166510116", SkyboxLf = "http://www.roblox.com/asset/?id=166510092", SkyboxRt = "http://www.roblox.com/asset/?id=166510131", SkyboxUp = "http://www.roblox.com/asset/?id=166510114" })
Visuals:NewSky({ Name = "Blue Night", SkyboxBk = "http://www.roblox.com/Asset/?ID=12064107", SkyboxDn = "http://www.roblox.com/Asset/?ID=12064152", SkyboxFt = "http://www.roblox.com/Asset/?ID=12064121", SkyboxLf = "http://www.roblox.com/Asset/?ID=12063984", SkyboxRt = "http://www.roblox.com/Asset/?ID=12064115", SkyboxUp = "http://www.roblox.com/Asset/?ID=12064131" })

local SkyboxNames = {}
for Name, _ in pairs(Skyboxes) do table.insert(SkyboxNames, Name) end
table.sort(SkyboxNames)

-- [[ HELPER FUNCTIONS ]] --

local function normalizeSelection(selection)
    if not selection then return {} end
    if type(selection) ~= "table" then local t = {}; t[selection] = true; return t end
    return selection
end

-- Direction Helper
local function getDirection(Origin, Position)
    local unit = (Position - Origin).Unit
    local multi = (Options.MultiplyUnitBy and Options.MultiplyUnitBy.Value) or 1000
    return unit * multi
end

local function CalculateChance(Percentage)
    Percentage = math.floor(Percentage)
    local chance = math.floor(Random.new().NextNumber(Random.new(), 0, 1) * 100) / 100
    return chance <= Percentage / 100
end

local ExpectedArguments = {
    FindPartOnRayWithIgnoreList = { ArgCountRequired = 3, Args = { "Instance", "Ray", "table", "boolean", "boolean" } },
    FindPartOnRayWithWhitelist = { ArgCountRequired = 3, Args = { "Instance", "Ray", "table", "boolean" } },
    FindPartOnRay = { ArgCountRequired = 2, Args = { "Instance", "Ray", "Instance", "boolean", "boolean" } },
    Raycast = { ArgCountRequired = 3, Args = { "Instance", "Vector3", "Vector3", "RaycastParams" } },
    ViewportPointToRay = { ArgCountRequired = 2, Args = { "number", "number" } },
    ScreenPointToRay = { ArgCountRequired = 2, Args = { "number", "number" } }
}

local function ValidateArguments(Args, RayMethod)
    local Matches = 0
    if #Args < RayMethod.ArgCountRequired then return false end
    for Pos, Argument in next, Args do
        if typeof(Argument) == RayMethod.Args[Pos] then
            Matches = Matches + 1
        end
    end
    return Matches >= RayMethod.ArgCountRequired
end

-- Whitelist Checker
local function selectionContains(selection, target)
    if not selection or not target then return false end
    local targetName = (typeof(target) == "Instance" and target.Name) or tostring(target)
    if type(selection) == "table" then
        if selection[targetName] == true then return true end
        for key, value in pairs(selection) do
            if value == true then 
                if type(key) == "string" and key == targetName then return true end
                if typeof(key) == "Instance" and typeof(target) == "Instance" then
                    if key == target then return true end 
                    if key.Name == targetName then return true end 
                end
                if type(key) == "number" and value == targetName then return true end
            end
        end
    end
    return false
end

-- Part Getter
local function getAimPart(character, forcedPartName)
    if not character then return nil end
    local partName = forcedPartName or Options.AimPart.Value
    local part = character:FindFirstChild(partName)
    if not part then
        if partName == "UpperTorso" or partName == "LowerTorso" then part = character:FindFirstChild("Torso") end
        if not part then part = character:FindFirstChild("HumanoidRootPart") end
    end
    return part
end

-- [[ REPLACEMENT WALL CHECK LOGIC ]] --
local function isVisible(part)
    if not part then return false end
    local method = Options.WallCheckMethod.Value
    if method == "None" then return true end
    
    -- DEFAULT ORIGIN: Camera
    local origin = Camera.CFrame.Position
    
    -- FIX: If Peek Check is ON, force origin to Character Head
    if Toggles.CameraPeekCheck and Toggles.CameraPeekCheck.Value and LocalPlayer.Character then
        local checkPart = LocalPlayer.Character:FindFirstChild("Head") or LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        if checkPart then 
            origin = checkPart.Position 
        end
    end

    local character = part.Parent
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Blacklist
    params.FilterDescendantsInstances = {LocalPlayer.Character}
    params.IgnoreWater = true
    
    if method == "Raycast" then
        local direction = (part.Position - origin)
        local result = workspace:Raycast(origin, direction, params)
        if not result then return true end
        return result.Instance:IsDescendantOf(character)
    elseif method == "Corners" then
        local size = part.Size / 2
        local cf = part.CFrame
        local corners = { cf*CFrame.new(size.X, size.Y, size.Z), cf*CFrame.new(-size.X, size.Y, size.Z), cf*CFrame.new(size.X, -size.Y, size.Z), cf*CFrame.new(-size.X, -size.Y, -size.Z), cf*CFrame.new(size.X, size.Y, -size.Z), cf*CFrame.new(-size.X, size.Y, -size.Z), cf*CFrame.new(size.X, -size.Y, -size.Z), cf*CFrame.new(-size.X, -size.Y, -size.Z) }
        local centerResult = workspace:Raycast(origin, (part.Position - origin), params)
        if not centerResult or centerResult.Instance:IsDescendantOf(character) then return true end
        for _, corner in ipairs(corners) do
            local result = workspace:Raycast(origin, (corner.Position - origin), params)
            if not result or result.Instance:IsDescendantOf(character) then return true end
        end
        return false
    elseif method == "Body Parts" then
        local partsToCheck = {"Head", "UpperTorso", "LowerTorso", "HumanoidRootPart", "LeftUpperArm", "RightUpperArm", "LeftUpperLeg", "RightUpperLeg", "Torso", "Left Arm", "Right Arm", "Left Leg", "Right Leg"}
        for _, partName in ipairs(partsToCheck) do
            local p = character:FindFirstChild(partName)
            if p then
                local res = workspace:Raycast(origin, (p.Position - origin), params)
                if not res or res.Instance:IsDescendantOf(character) then return true end
            end
        end
        return false
    elseif method == "Head Only" then
        local head = character:FindFirstChild("Head")
        if not head then return false end
        local res = workspace:Raycast(origin, (head.Position - origin), params)
        if not res then return true end
        return res.Instance:IsDescendantOf(character)
    elseif method == "Bounds Check" then
        local hrp = character:FindFirstChild("HumanoidRootPart")
        if not hrp then return false end
        local checkPoints = {Vector3.new(0,0,0), Vector3.new(0,2,0), Vector3.new(0,-1.5,0), Vector3.new(1,0,0), Vector3.new(-1,0,0), Vector3.new(0,0,1), Vector3.new(0,0,-1)}
        for _, off in ipairs(checkPoints) do
            local tPos = hrp.Position + off
            local res = workspace:Raycast(origin, (tPos - origin), params)
            if not res or res.Instance:IsDescendantOf(character) then return true end
        end
        return false
    end
    -- Fallback
    local direction = (part.Position - origin)
    local result = workspace:Raycast(origin, direction, params)
    if not result then return true end
    return result.Instance:IsDescendantOf(character)
end

-- ===============================================
-- === UI WINDOW & STATS ===
-- ===============================================
local Window = Library:CreateWindow({
    Title = 'Kill<font color="rgb(255, 0, 0)">.all</font>',
    Footer = "Loading Data...",
    NotifySide = "Right",
    ShowCustomCursor = false,
})

if Library.Theme then
    Library.Theme.Accent = Color3.fromRGB(255, 85, 85)
end

task.spawn(function()
    local Stats = game:GetService("Stats")
    local StartTime = tick()
    local LabelInstance = nil
    task.wait(1)
    local Container = Library.MainFrame or game:GetService("CoreGui")
    for _, obj in pairs(Container:GetDescendants()) do
        if obj:IsA("TextLabel") and obj.Text == "Loading Data..." then
            LabelInstance = obj; LabelInstance.RichText = true; break
        end
    end
    while true do
        local Elapsed = tick() - StartTime
        local h = math.floor(Elapsed / 3600); local m = math.floor(Elapsed / 60) % 60; local s = math.floor(Elapsed % 60)
        local pingVal = 0; pcall(function() pingVal = Stats.Network.ServerStatsItem["Data Ping"]:GetValue() end)
        local r, g, b = 90, 255, 90 
        if pingVal > 180 then r, g, b = 255, 65, 65 elseif pingVal > 110 then r, g, b = 255, 140, 30 elseif pingVal > 70 then r, g, b = 255, 230, 80 end
        local pingString = string.format('<font color="rgb(%d,%d,%d)">%.2f</font>ms', r, g, b, pingVal)
        local verString = 'e<font color="rgb(85, 170, 255)">1.38</font>' 
        local dateString = os.date("%B %d, %Y") .. " 📅"; local timeString = os.date("%I:%M %p")
        local uptimeString = string.format("[%02d:%02d:%02d]", h, m, s)
        local finalString = string.format("⚙ Core Version: %s | %s | %s | %s %s", verString, pingString, dateString, timeString, uptimeString)
        if LabelInstance and LabelInstance.Parent then LabelInstance.RichText = true; LabelInstance.Text = finalString
        elseif Window.FooterLabel then Window.FooterLabel.RichText = true; Window.FooterLabel.Text = finalString end
        task.wait(0.5)
    end
end)

local Tabs = {
    Combat = Window:AddTab("Combat", "crosshair"),
    Whitelist = Window:AddTab("Whitelist", "shield"),
    Misc = Window:AddTab("Miscellaneous", "cog"),
    User = Window:AddTab("User", "user"),
    ["UI Settings"] = Window:AddTab("UI Settings", "settings"),
    Hitsound = Window:AddTab("", "sound"),
}

local AimbotGroup = Tabs.Combat:AddLeftGroupbox("Aim")
AimbotGroup:AddToggle("AimbotEnabled", {
    Text = "Aimbot", Default = false,
    Callback = function(value)
        if FOVCircle then FOVCircle.Visible = value and Toggles.FovVisible.Value end
    end,
})

AimbotGroup:AddToggle("TeamCheck", { Text = "Team Check", Default = true })
AimbotGroup:AddToggle("HostileCheck", { Text = "Hostile Check", Default = false, Tooltip = "Aims only if target is holding a tool" })
AimbotGroup:AddToggle("WallCheck", { Text = "Wall Check", Default = true })
AimbotGroup:AddToggle("HealthCheck", { Text = "Health Check", Default = true })
AimbotGroup:AddToggle("ForceFieldCheck", { Text = "ForceField Check", Default = true })
AimbotGroup:AddToggle("AimNPCs", { Text = "NPC Check", Default = false, Tooltip = "Allows targeting NPCs" })

-- [[ PASTE THIS IN AIMBOT GROUP ]] --
AimbotGroup:AddToggle("CameraPeekCheck", { 
    Text = "Camera Peek Check", 
    Default = false, 
    Tooltip = "Prevents shooting walls. Aims using Character visibility instead of Camera." 
})

AimbotGroup:AddSlider("MinHealth", { Text = "Min Health to Aim", Default = 0, Min = 0, Max = 100, Rounding = 0 })
AimbotGroup:AddLabel("Aimbot Key"):AddKeyPicker("AimbotKeybind", { Default = "Q", Mode = "Hold", Text = "Press to aim (hold)", DefaultModifiers = {} })
AimbotGroup:AddLabel("Left click for selecting modes")
AimbotGroup:AddDivider()
AimbotGroup:AddDropdown("WallCheckMethod", { 
    Values = { "Raycast", "Corners", "Body Parts", "Head Only", "Bounds Check", "None" }, 
    Default = 1, 
    Text = "Wall Check Method" 
})
AimbotGroup:AddDropdown("AimMethod", { Values = { "Mouse", "Camera" }, Default = 1, Text = "Aim Method" })
AimbotGroup:AddSlider("Prediction", { Text = "Prediction", Default = 0.18, Min = 0, Max = 0.5, Rounding = 3 })
AimbotGroup:AddSlider("ReactionTimeX", { Text = "X ms (Drift Time)", Default = 500, Min = 0, Max = 1000, Rounding = 0, Callback = function(val) ReactionDelayX = val / 1000; TargetReactionDelay = (ReactionDelayX + ReactionDelayY) / 2 end })
AimbotGroup:AddSlider("ReactionTimeY", { Text = "Y ms (Drift Time)", Default = 500, Min = 0, Max = 1000, Rounding = 0, Callback = function(val) ReactionDelayY = val / 1000; TargetReactionDelay = (ReactionDelayX + ReactionDelayY) / 2 end })
AimbotGroup:AddSlider("SnapBackSpeed", { Text = "Snap Back Speed", Default = 5, Min = 1, Max = 20, Rounding = 1 })
AimbotGroup:AddSlider("XSmoothness", { Text = "X Smoothness", Default = 5, Min = 1, Max = 20 })
AimbotGroup:AddSlider("YSmoothness", { Text = "Y Smoothness", Default = 5, Min = 1, Max = 20 })
AimbotGroup:AddSlider("MaxDistance", { Text = "Aim Distance", Default = 2000, Min = 50, Max = 10000 })
AimbotGroup:AddDivider()
AimbotGroup:AddDropdown("AimPart", { Values = { "Head", "UpperTorso", "HumanoidRootPart" }, Default = 1, Text = "AimPart" })
AimbotGroup:AddLabel("Fov Color"):AddColorPicker("FovColor", { Default = Color3.new(1,1,1) })
AimbotGroup:AddDivider()
AimbotGroup:AddSlider("FOVRadius", { Text = "FOV Size", Default = 200, Min = 10, Max = 500 })
AimbotGroup:AddToggle("FovVisible", { Text = "Show FOV Circle", Default = true, Callback = function(val) if FOVCircle then FOVCircle.Visible = Toggles.AimbotEnabled.Value and val end end })
AimbotGroup:AddToggle("FOVAnim", { Text = "Gradient Fill Spin", Default = false }) 
AimbotGroup:AddToggle("RainbowFov", { Text = "Rainbow FOV", Default = false })
AimbotGroup:AddDivider()
AimbotGroup:AddToggle("PredictionForeshadow", { Text = "Prediction Foreshadow", Default = false })
AimbotGroup:AddLabel("Ghost Color"):AddColorPicker("GhostColor", { Default = Color3.fromRGB(120, 120, 120) })
AimbotGroup:AddSlider("GhostTransparency", { Text = "Ghost Transparency", Default = 0.5, Min = 0, Max = 1, Rounding = 2 })

local WhitelistGroup = Tabs.Whitelist:AddLeftGroupbox("Whitelists")
WhitelistGroup:AddDropdown("PlayerWhitelist", { SpecialType = "Player", ExcludeLocalPlayer = true, Multi = true, Text = "Player Whitelist" })
WhitelistGroup:AddDropdown("TeamWhitelist", { SpecialType = "Team", Multi = true, Text = "Team Whitelist" })
WhitelistGroup:AddDivider()
WhitelistGroup:AddLabel("ESP Whitelists")
WhitelistGroup:AddDropdown("ESPPlayerWhitelist", { SpecialType = "Player", ExcludeLocalPlayer = true, Multi = true, Text = "ESP Player Whitelist" })
WhitelistGroup:AddDropdown("ESPTeamWhitelist", { SpecialType = "Team", Multi = true, Text = "ESP Team Whitelist" })

-- [MISC TAB]
local MiscGroup = Tabs.Misc:AddLeftGroupbox("Miscellaneous")
local SkyboxGroup = Tabs.Misc:AddRightGroupbox("SkyBox")
SkyboxGroup:AddDropdown("SkyboxSelector", {
    AllowNull = false,
    Text = "Select Skybox",
    Default = "Game's Default Sky",
    Values = SkyboxNames
}):OnChanged(function(SelectedSkybox)
    Visuals:SwitchSkybox(SelectedSkybox)
end)
-- [[ AMBIANCE SETTINGS SNIPPET ]] --
local AmbianceGroup = Tabs.Misc:AddRightGroupbox("Ambiance")

AmbianceGroup:AddSlider("AmbBrightness", { Text = "Brightness", Default = Lighting.Brightness, Min = 0, Max = 10, Rounding = 2, Callback = function(v) Lighting.Brightness = v end })
AmbianceGroup:AddSlider("AmbClockTime", { Text = "Time of Day", Default = Lighting.ClockTime, Min = 0, Max = 24, Rounding = 1, Callback = function(v) Lighting.ClockTime = v end })
AmbianceGroup:AddSlider("AmbExposure", { Text = "Exposure", Default = Lighting.ExposureCompensation, Min = -3, Max = 3, Rounding = 2, Callback = function(v) Lighting.ExposureCompensation = v end })
AmbianceGroup:AddLabel("Ambient Color"):AddColorPicker("AmbAmbient", { Default = Lighting.Ambient, Callback = function(v) Lighting.Ambient = v end })
AmbianceGroup:AddLabel("Outdoor Ambient"):AddColorPicker("AmbOutdoor", { Default = Lighting.OutdoorAmbient, Callback = function(v) Lighting.OutdoorAmbient = v end })
AmbianceGroup:AddLabel("Color Shift Top"):AddColorPicker("AmbShiftTop", { Default = Lighting.ColorShift_Top, Callback = function(v) Lighting.ColorShift_Top = v end })
AmbianceGroup:AddLabel("Color Shift Bottom"):AddColorPicker("AmbShiftBot", { Default = Lighting.ColorShift_Bottom, Callback = function(v) Lighting.ColorShift_Bottom = v end })
AmbianceGroup:AddDivider()
AmbianceGroup:AddLabel("Fog Color"):AddColorPicker("AmbFogColor", { Default = Lighting.FogColor, Callback = function(v) Lighting.FogColor = v end })
AmbianceGroup:AddSlider("AmbFogStart", { Text = "Fog Start", Default = Lighting.FogStart, Min = 0, Max = 5000, Rounding = 0, Callback = function(v) Lighting.FogStart = v end })
AmbianceGroup:AddSlider("AmbFogEnd", { Text = "Fog End", Default = Lighting.FogEnd, Min = 0, Max = 100000, Rounding = 0, Callback = function(v) Lighting.FogEnd = v end })
AmbianceGroup:AddDivider()
AmbianceGroup:AddToggle("AmbShadows", { Text = "Global Shadows", Default = Lighting.GlobalShadows, Callback = function(v) Lighting.GlobalShadows = v end })
AmbianceGroup:AddSlider("AmbLatitude", { Text = "Geographic Latitude", Default = Lighting.GeographicLatitude, Min = -90, Max = 90, Rounding = 1, Callback = function(v) Lighting.GeographicLatitude = v end })
AmbianceGroup:AddButton({ Text = "Reset Ambiance", Func = function()
    Lighting.Brightness = 3; Lighting.ClockTime = 14; Lighting.ExposureCompensation = 0
    Lighting.Ambient = Color3.fromRGB(127,127,127); Lighting.OutdoorAmbient = Color3.fromRGB(127,127,127)
    Lighting.ColorShift_Top = Color3.new(0,0,0); Lighting.ColorShift_Bottom = Color3.new(0,0,0)
    Lighting.FogColor = Color3.fromRGB(191,191,191); Lighting.FogStart = 0; Lighting.FogEnd = 100000
    Lighting.GlobalShadows = true; Lighting.GeographicLatitude = 41.733
end })

MiscGroup:AddToggle("TestToggle", { Text = "TestToggle", Default = false })
-- [[ MISCELLANEOUS FEATURES ]] --
MiscGroup:AddToggle("AntiAFK", { Text = "Anti-AFK", Default = false, Callback = function(v)
    if v then
        local VirtualUser = game:GetService("VirtualUser")
        getgenv().AntiAFKConnection = LocalPlayer.Idled:Connect(function()
            VirtualUser:CaptureController()
            VirtualUser:ClickButton2(Vector2.new())
        end)
        Library:Notify("Anti-AFK Enabled", 2)
    else
        if getgenv().AntiAFKConnection then getgenv().AntiAFKConnection:Disconnect() end
    end
end })
MiscGroup:AddToggle("NoFallDamage", { Text = "No Fall Damage", Default = false })
MiscGroup:AddToggle("LowGravity", { Text = "Low Gravity", Default = false, Callback = function(v)
    workspace.Gravity = v and 50 or 196.2
end })
MiscGroup:AddSlider("GravityVal", { Text = "Custom Gravity", Default = 196.2, Min = 0, Max = 500, Rounding = 1, Callback = function(v) 
    if Toggles.LowGravity.Value then workspace.Gravity = v end 
end })
MiscGroup:AddDivider()

-- Rejoin & Server Hop
MiscGroup:AddButton({ Text = "Rejoin Server", Func = function()
    Library:Notify("Rejoining...", 2)
    task.wait(0.5)
    game:GetService("TeleportService"):Teleport(game.PlaceId, LocalPlayer)
end })
MiscGroup:AddButton({ Text = "Server Hop", Func = function()
    Library:Notify("Finding new server...", 2)
    local Http = game:GetService("HttpService")
    local servers = Http:JSONDecode(game:HttpGet("https://games.roblox.com/v1/games/"..game.PlaceId.."/servers/Public?sortOrder=Asc&limit=100"))
    for _, v in pairs(servers.data) do
        if v.playing < v.maxPlayers and v.id ~= game.JobId then
            game:GetService("TeleportService"):TeleportToPlaceInstance(game.PlaceId, v.id, LocalPlayer)
            return
        end
    end
    Library:Notify("No available servers found!", 3)
end })
MiscGroup:AddButton({ Text = "Copy Game Link", Func = function()
    setclipboard("https://www.roblox.com/games/" .. game.PlaceId)
    Library:Notify("Game link copied!", 2)
end })
MiscGroup:AddButton({ Text = "Copy Server ID", Func = function()
    setclipboard(game.JobId)
    Library:Notify("Server ID copied!", 2)
end })

-- [[ EFFECTS GROUP ]] --
local EffectsGroup = Tabs.Misc:AddLeftGroupbox("Effects")
EffectsGroup:AddToggle("RemoveBlur", { Text = "Remove Blur", Default = false, Callback = function(v)
    for _, obj in pairs(Lighting:GetChildren()) do
        if obj:IsA("BlurEffect") then obj.Enabled = not v end
    end
end })
EffectsGroup:AddToggle("RemoveBloom", { Text = "Remove Bloom", Default = false, Callback = function(v)
    for _, obj in pairs(Lighting:GetChildren()) do 
        if obj:IsA("BloomEffect") then obj.Enabled = not v end
    end
end })
EffectsGroup:AddToggle("RemoveDepthOfField", { Text = "Remove DOF", Default = false, Callback = function(v)
    for _, obj in pairs(Lighting:GetChildren()) do
        if obj:IsA("DepthOfFieldEffect") then obj.Enabled = not v end
    end
end })
EffectsGroup:AddToggle("RemoveColorCorrection", { Text = "Remove Color Correction", Default = false, Callback = function(v)
    for _, obj in pairs(Lighting:GetChildren()) do
        if obj:IsA("ColorCorrectionEffect") then obj.Enabled = not v end
    end
end })
EffectsGroup:AddToggle("RemoveSunRays", { Text = "Remove Sun Rays", Default = false, Callback = function(v)
    for _, obj in pairs(Lighting:GetChildren()) do
        if obj:IsA("SunRaysEffect") then obj.Enabled = not v end
    end
end })
EffectsGroup:AddToggle("RemoveAtmosphere", { Text = "Remove Atmosphere", Default = false, Callback = function(v)
    for _, obj in pairs(Lighting:GetChildren()) do
        if obj:IsA("Atmosphere") then obj.Density = v and 0 or 0.395 end
    end
end })
EffectsGroup:AddButton({ Text = "Remove All Effects", Func = function()
    for _, obj in pairs(Lighting:GetChildren()) do
        if obj:IsA("PostEffect") or obj:IsA("Atmosphere") then obj.Enabled = false end
    end
    Library:Notify("All effects removed!", 2)
end })

-- [[ USER INFORMATION GROUP (MOVED TO TOP) ]] --
local UserGroup = Tabs.User:AddLeftGroupbox("User Information")
UserGroup:AddLabel("Player executor: " .. (identifyexecutor() or "Unknown"))
UserGroup:AddLabel("Username: " .. LocalPlayer.Name)
UserGroup:AddLabel("Profile Name: " .. LocalPlayer.DisplayName)
local creationTime = os.time() - (LocalPlayer.AccountAge * 86400)
local creationDate = os.date("%Y-%m-%d", creationTime)
UserGroup:AddLabel("Account created: " .. creationDate)
local hwid = game:GetService("RbxAnalyticsService"):GetClientId()
UserGroup:AddLabel("HWID: " .. hwid)

-- [[ TELEPORT GROUP ]] --
local TeleportGroup = Tabs.User:AddLeftGroupbox("Teleport")
TeleportGroup:AddDropdown("TeleportPlayer", { SpecialType = "Player", ExcludeLocalPlayer = true, Text = "Select Player" })
TeleportGroup:AddButton({ Text = "Teleport to Player", Func = function()
    local plrName = Options.TeleportPlayer.Value
    local targetPlr = Players:FindFirstChild(plrName)
    if targetPlr and targetPlr.Character and targetPlr.Character:FindFirstChild("HumanoidRootPart") then
        if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
            LocalPlayer.Character.HumanoidRootPart.CFrame = targetPlr.Character.HumanoidRootPart.CFrame * CFrame.new(0,0,3)
            Library:Notify("Teleported to " .. plrName, 2)
        end
    else
        Library:Notify("Player not found!", 2)
    end
end })
TeleportGroup:AddButton({ Text = "Teleport to Mouse", Func = function()
    if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") and mouse.Hit then
        LocalPlayer.Character.HumanoidRootPart.CFrame = mouse.Hit + Vector3.new(0,3,0)
    end
end })
TeleportGroup:AddLabel("Teleport to Mouse Key"):AddKeyPicker("TeleportMouseKey", { Default = "T", Mode = "Toggle", Text = "TP Mouse Key" })
TeleportGroup:AddDivider()
TeleportGroup:AddLabel("Saved Position:")
TeleportGroup:AddButton({ Text = "Save Current Position", Func = function()
    if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
        getgenv().SavedPosition = LocalPlayer.Character.HumanoidRootPart.CFrame
        Library:Notify("Position saved!", 2)
    end
end })
TeleportGroup:AddButton({ Text = "Load Saved Position", Func = function()
    if getgenv().SavedPosition and LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
        LocalPlayer.Character.HumanoidRootPart.CFrame = getgenv().SavedPosition
        Library:Notify("Teleported to saved position!", 2)
    else
        Library:Notify("No position saved!", 2)
    end
end })

getgenv().MiscLoopAdded = true

-- Visuals Group
local VisualsGroup = Tabs.Combat:AddRightGroupbox("Visuals")
VisualsGroup:AddToggle("ESPEnabled", { Text = "Enable ESP", Default = false, Callback = function(value) ESP.Enabled = value end })
VisualsGroup:AddToggle("ESPBox", { Text = "Box", Default = true, Callback = function(value) ESP.Drawing.Boxes.Full.Enabled = value; ESP.Drawing.Boxes.Corner.Enabled = value end })
VisualsGroup:AddLabel("Box Color"):AddColorPicker("ESPBoxColor", { Default = Color3.new(1,1,1), Callback = function(value) ESP.Drawing.Boxes.GradientRGB1 = value; ESP.Drawing.Boxes.GradientFillRGB1 = value end })
VisualsGroup:AddToggle("ESPBoxFilled", { Text = "Fill Box", Default = false, Callback = function(value) ESP.Drawing.Boxes.Filled.Enabled = value end })
VisualsGroup:AddLabel("Filled Box Color"):AddColorPicker("ESPBoxFillColor", { Default = Color3.new(1,0,0), Callback = function(value) ESP.Drawing.Boxes.Filled.RGB = value end })
VisualsGroup:AddSlider("ESPBoxFillTransparency", { Text = "Fill Box Transparency", Default = 0.5, Min = 0, Max = 1, Rounding = 2, Callback = function(value) ESP.Drawing.Boxes.Filled.Transparency = value * 100 end })
VisualsGroup:AddToggle("ESPName", { Text = "Name", Default = true, Callback = function(value) ESP.Drawing.Names.Enabled = value end })
VisualsGroup:AddToggle("ESPDistance", { Text = "Distance", Default = true, Callback = function(value) ESP.Drawing.Distances.Enabled = value end })
VisualsGroup:AddToggle("ESPHealth", { Text = "Health Bar", Default = true, Callback = function(value) ESP.Drawing.Healthbar.Enabled = value end })
VisualsGroup:AddToggle("ESPTool", { Text = "Tool", Default = true, Callback = function(value) ESP.Drawing.Weapons.Enabled = value end })
VisualsGroup:AddToggle("ESPHideTeam", { Text = "Hide Teammates", Default = true, Callback = function(value) ESP.TeamCheck = value end })
VisualsGroup:AddSlider("ESPMaxDistance", { Text = "Max Distance", Default = 200, Min = 100, Max = 5000, Rounding = 0, Callback = function(val) ESP.MaxDistance = val end })
VisualsGroup:AddLabel("ESP Color"):AddColorPicker("ESPColor", { Default = Color3.new(1,1,1), Callback = function(value) ESP.Drawing.Weapons.WeaponTextRGB = value; ESP.Drawing.Healthbar.HealthTextRGB = value; ESP.Drawing.Names.RGB = value; ESP.Drawing.Distances.RGB = value end })
VisualsGroup:AddToggle("RainbowESP", { Text = "Rainbow ESP", Default = false })
VisualsGroup:AddToggle("ChamsEnabled", { Text = "Chams", Default = false, Callback = function(value) ESP.Drawing.Chams.Enabled = value end })
VisualsGroup:AddLabel("Chams Color"):AddColorPicker("ChamsColor", { Default = Color3.new(1,1,1), Callback = function(value) ESP.Drawing.Chams.FillRGB = value; ESP.Drawing.Chams.OutlineRGB = value end })
VisualsGroup:AddSlider("ESPFontSize", { Text = "Text Size", Default = 1, Min = 1, Max = 3, Rounding = 0, Callback = function(val) ESP.FontSize = 11 + (val - 1) * 22 end })
VisualsGroup:AddDivider()
VisualsGroup:AddToggle("ESPThroughWall", { Text = "Show Through Walls", Default = true })
VisualsGroup:AddDivider()
VisualsGroup:AddToggle("FOVTracer", { Text = "FOV Tracer", Default = false, Tooltip = "Draws line to valid target in FOV" })
VisualsGroup:AddLabel("FOV Tracer Color"):AddColorPicker("FOVTracerColor", { Default = Color3.fromRGB(255, 50, 50) })

local FOVTracerLine = Drawing.new("Line")
FOVTracerLine.Thickness = 1
FOVTracerLine.Transparency = 1
FOVTracerLine.Visible = false
FOVTracerLine.Color = Color3.new(1, 1, 1)

local HitsoundGroup = Tabs.Hitsound:AddLeftGroupbox("Hitsound.")
HitsoundGroup:AddToggle("HitsoundEnabled", { Text = "Enable Hitsound", Default = false })
HitsoundGroup:AddDropdown("HitsoundType", { Values = { "neverlose.cc", "fatality.win", "bameware.club", "skeet.cc", "rifk7.com", "primordial.dev" }, Default = 1, Text = "Hitsound" })
HitsoundGroup:AddSlider("HitsoundVolume", { Text = "Volume", Default = 3, Min = 0, Max = 10, Rounding = 1, Callback = function(val) if HitSound then HitSound.Volume = val end end })

local HitLogGroup = Tabs.Hitsound:AddRightGroupbox("Hit Logs")
HitLogGroup:AddToggle("HitLogEnabled", { Text = "Enable Hit Notify", Default = false, Tooltip = "Shows notification when damage is dealt." })
HitLogGroup:AddSlider("HitLogDuration", { Text = "Notify Duration", Default = 2, Min = 1, Max = 5, Rounding = 1 })

local HitSound = Instance.new("Sound")
HitSound.Parent = game:GetService("SoundService")
HitSound.SoundId = "rbxassetid://97643101798871"
HitSound.Volume = 3

Options.HitsoundType:OnChanged(function(value)
    if value == "neverlose.cc" then HitSound.SoundId = "rbxassetid://97643101798871"
    elseif value == "fatality.win" then HitSound.SoundId = "rbxassetid://106586644436584"
    elseif value == "bameware.club" then HitSound.SoundId = "rbxassetid://92614567965693"
    elseif value == "skeet.cc" then HitSound.SoundId = "rbxassetid://4817809188"
    elseif value == "rifk7.com" then HitSound.SoundId = "rbxassetid://76064874887167"
    elseif value == "primordial.dev" then HitSound.SoundId = "rbxassetid://85340682645435" end
end)

local function HookHitsound(char)
    local hum = char:WaitForChild("Humanoid", 10)
    if not hum then return end
    local lastHealth = hum.Health
    hum:GetPropertyChangedSignal("Health"):Connect(function()
        local newHealth = hum.Health
        if newHealth < lastHealth then
            local isTarget = (Target and char == Target)
            if not isTarget then
                local mouseT = mouse.Target
                if mouseT and mouseT:IsDescendantOf(char) then isTarget = true end
            end
            
            if not isTarget and ScriptState.ClosestHitPart and ScriptState.ClosestHitPart.Parent == char then
                isTarget = true
            end

            if isTarget then
                if Toggles.HitsoundEnabled.Value then
                    task.spawn(function()
                        local s = HitSound:Clone()
                        s.Parent = game:GetService("SoundService"); s:Play(); Debris:AddItem(s, 2)
                    end)
                end
                
                if Toggles.HitLogEnabled.Value then
                    local partHit = "Unknown"
                    local dmg = math.floor(lastHealth - newHealth + 0.5) 
                    if Toggles.SilentAim.Value and ScriptState.ClosestHitPart then
                        partHit = ScriptState.ClosestHitPart.Name
                    else
                        local aimPart = getAimPart(char)
                        if aimPart then partHit = aimPart.Name end
                    end
                    Library:Notify(string.format("Hit %s in %s for %d HP", char.Name, partHit, dmg), Options.HitLogDuration.Value)
                end
            end
        end
        lastHealth = newHealth
    end)
end
for _, plr in ipairs(Players:GetPlayers()) do
    if plr ~= LocalPlayer then
        if plr.Character then task.spawn(HookHitsound, plr.Character) end
        plr.CharacterAdded:Connect(function(c) task.spawn(HookHitsound, c) end)
    end
end
Players.PlayerAdded:Connect(function(plr)
    if plr ~= LocalPlayer then plr.CharacterAdded:Connect(function(c) task.spawn(HookHitsound, c) end) end
end)

local TabBox = Tabs.Combat:AddRightTabbox("Legit / Blatant / Rage")

-- ============ LEGIT TAB ============
local LegitTab = TabBox:AddTab("Legit")
LegitTab:AddToggle("StickyAim", { Text = "Sticky Aim", Default = false })
LegitTab:AddToggle("StickyWallCheck", { Text = "Sticky Wall Check", Default = true })
LegitTab:AddToggle("SwitchPart", { Text = "Switch Part", Default = false })
LegitTab:AddSlider("SwitchPartDelay", { Text = "Switch Part Delay (s)", Default = 2, Min = 0.1, Max = 10, Rounding = 1 })
LegitTab:AddToggle("TriggerDistanceCheck", { Text = "Trigger Distance Check", Default = false })
LegitTab:AddSlider("TriggerMaxDistance", { Text = "Trigger Max Distance", Default = 500, Min = 50, Max = 2000 })
LegitTab:AddToggle("TriggerbotEnabled", { Text = "Triggerbot (Hold Key)", Default = false })
LegitTab:AddToggle("TriggerbotWallCheck", { Text = "Triggerbot Wall Check", Default = false })
LegitTab:AddSlider("TriggerbotDelay", { Text = "Triggerbot Delay", Default = 0, Min = 0, Max = 1, Rounding = 2 })
LegitTab:AddLabel("Triggerbot Key"):AddKeyPicker("TriggerbotKeybind", { Default = "E", Mode = "Hold", Text = "Press to trigger (hold)" })
LegitTab:AddDivider()
LegitTab:AddSlider("LegitFOV", { Text = "Legit FOV Size", Default = 80, Min = 10, Max = 200, Rounding = 0 })
LegitTab:AddSlider("LegitSmoothness", { Text = "Legit Smoothness", Default = 10, Min = 1, Max = 20, Rounding = 1 })

-- ============ BLATANT TAB ============
local BlatantTab = TabBox:AddTab("Blatant")
BlatantTab:AddToggle("SilentAim", { Text = "Silent Aim", Default = false })
BlatantTab:AddLabel("Auto Shoot Logic")
BlatantTab:AddToggle("TriggerAutoShoot", { Text = "Auto Shoot", Default = false })
BlatantTab:AddLabel("FOR TRIGGERBOT AND ITS FOR SILENT AIM ONLY")
BlatantTab:AddDropdown("SilentMethod", { Values = { "Raycast", "FindPartOnRay", "FindPartOnRayWithIgnoreList", "FindPartOnRayWithWhitelist", "ViewportPointToRay", "ScreenPointToRay", "CounterBlox", "Mouse.Hit/Target" }, Default = 1, Text = "Silent Aim Method" })
BlatantTab:AddDropdown("BlockedMethods", { 
    Text = "Blocked Methods", 
    Default = {}, 
    Values = { "BulkMoveTo", "PivotTo", "TranslateBy", "SetPrimaryPartCFrame" },
    Multi = true,
})
BlatantTab:AddDropdown("SilentTargetPart", { Values = { "Head", "HumanoidRootPart", "Random" }, Default = 1, Text = "Target Part" })
BlatantTab:AddSlider("SilentHitChance", { Text = "Hit Chance", Default = 100, Min = 0, Max = 100, Rounding = 0 })
BlatantTab:AddSlider("MultiplyUnitBy", { Text = "Multiply Unit By", Default = 1000, Min = 1, Max = 10000, Rounding = 0, Tooltip = "Multiplies the direction vector by this value" })
BlatantTab:AddSlider("SilentPrediction", { Text = "Silent Prediction", Default = 0.165, Min = 0, Max = 1, Rounding = 3 })
BlatantTab:AddSlider("SilentSmoothnessX", { Text = "Silent Smoothness X", Default = 1, Min = 1, Max = 20, Rounding = 1, Suffix = "x" })
BlatantTab:AddSlider("SilentSmoothnessY", { Text = "Silent Smoothness Y", Default = 1, Min = 1, Max = 20, Rounding = 1, Suffix = "x" })
BlatantTab:AddDivider()
BlatantTab:AddLabel("Silent Aim Miss Offsets (XYZ)")
BlatantTab:AddSlider("AimOffsetX", { Text = "Miss Offset X", Default = 0, Min = -10, Max = 10, Rounding = 1 })
BlatantTab:AddSlider("AimOffsetY", { Text = "Miss Offset Y", Default = 0, Min = -10, Max = 10, Rounding = 1 })
BlatantTab:AddSlider("AimOffsetZ", { Text = "Miss Offset Z", Default = 0, Min = -10, Max = 10, Rounding = 1 })
BlatantTab:AddDivider()
BlatantTab:AddToggle("SilentAliveCheck", { Text = "Alive Check", Default = true })
BlatantTab:AddToggle("BulletTP", { Text = "Bullet Teleport", Default = false })

-- ============ RAGE TAB ============
local RageTab = TabBox:AddTab("Rage")

RageTab:AddToggle("RageMode", { 
    Text = "Rage Mode", 
    Default = false, 
    Tooltip = "Enable Rage Silent Aim (Auto Shoots). Uses Global Settings (Wallcheck, Hostile Check etc)" 
}):AddKeyPicker("RageKeybind", { Default = "V", Mode = "Toggle", Text = "Rage Toggle" })

RageTab:AddSlider("RageFOV", { Text = "Rage Field of View", Default = 180, Min = 1, Max = 360, Rounding = 0, Tooltip = "360 = All Around, 180 = Front Only" })
RageTab:AddDropdown("RageTargetPart", { Values = { "Head", "HumanoidRootPart", "Random" }, Default = 1, Text = "Rage Hit Part" })

local CheatsGroup = Tabs.User:AddRightGroupbox("Cheats")
CheatsGroup:AddToggle("WalkSpeedEnabled", { 
    Text = "Enable WalkSpeed", 
    Default = false,
    Callback = function(val)
        if not val and LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("Humanoid") then
            LocalPlayer.Character.Humanoid.WalkSpeed = 16 
        end
    end 
})
CheatsGroup:AddSlider("WalkSpeedVal", { Text = "WalkSpeed Value", Default = 16, Min = 0, Max = 300, Rounding = 1 })
CheatsGroup:AddToggle("JumpPowerEnabled", { 
    Text = "Enable JumpPower", 
    Default = false,
    Callback = function(val)
        if not val and LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("Humanoid") then
             LocalPlayer.Character.Humanoid.JumpPower = 50
        end
    end
})
CheatsGroup:AddSlider("JumpPowerVal", { Text = "JumpPower Value", Default = 50, Min = 0, Max = 300, Rounding = 1 })
CheatsGroup:AddToggle("CFrameSpeedEnabled", { Text = "CFrame WalkSpeed", Default = false })
CheatsGroup:AddSlider("CFrameSpeedVal", { Text = "CFrame Speed Factor", Default = 1, Min = 0, Max = 5, Rounding = 1 })
CheatsGroup:AddDivider()
CheatsGroup:AddSlider("FOVVal", { Text = "Field of View", Default = 70, Min = 0, Max = 120, Rounding = 0, Callback = function(val) workspace.CurrentCamera.FieldOfView = val end })
CheatsGroup:AddDivider()
CheatsGroup:AddToggle("InfJump", { Text = "Infinite Jump", Default = false })
CheatsGroup:AddToggle("Noclip", { 
    Text = "Noclip", 
    Default = false, 
    Callback = function(val)
        if not val and LocalPlayer.Character then
            for _, v in pairs(LocalPlayer.Character:GetDescendants()) do
                if v:IsA("BasePart") then v.CanCollide = true end
            end
        end
    end 
})
-- [[ RESTORED FLY LOGIC ]] --
CheatsGroup:AddToggle("Fly", { 
    Text = "Enable Fly", 
    Default = false,
    Callback = function(Value)
        if not Value then
            if LocalPlayer.Character then
                local humanoid = LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
                if humanoid then humanoid.PlatformStand = false end
                if FlyBV then FlyBV:Destroy(); FlyBV = nil end
                if FlyBG then FlyBG:Destroy(); FlyBG = nil end
            end
        else
            local char = LocalPlayer.Character
            if char then
                local hrp = char:FindFirstChild("HumanoidRootPart")
                local humanoid = char:FindFirstChildOfClass("Humanoid")
                if hrp and humanoid then
                    humanoid.PlatformStand = true
                    if hrp:FindFirstChild("BodyVelocity") then hrp.BodyVelocity:Destroy() end
                    if hrp:FindFirstChild("BodyGyro") then hrp.BodyGyro:Destroy() end
                    FlyBV = Instance.new("BodyVelocity"); FlyBV.MaxForce = Vector3.new(1e5,1e5,1e5); FlyBV.Velocity = Vector3.new(0,0,0); FlyBV.Parent = hrp
                    FlyBG = Instance.new("BodyGyro"); FlyBG.MaxTorque = Vector3.new(1e5,1e5,1e5); FlyBG.CFrame = hrp.CFrame; FlyBG.Parent = hrp
                end
            end
        end
    end 
})
CheatsGroup:AddSlider("FlySpeed", { Text = "Fly Speed", Default = 50, Min = 1, Max = 250, Rounding = 1 })
CheatsGroup:AddToggle("FullBright", { Text = "Full Bright", Default = false, Callback = function(Value)
    if Value then
        Lighting.Brightness = 2; Lighting.ClockTime = 14; Lighting.FogEnd = 100000; Lighting.Ambient = Color3.new(1,1,1)
    else
        Lighting.Brightness = 3; Lighting.ClockTime = 12; Lighting.FogEnd = 10000; Lighting.Ambient = Color3.fromRGB(127,127,127)
    end
end})

LocalPlayer.CharacterAdded:Connect(function(char)
end)

UserInputService.MouseBehavior = Enum.MouseBehavior.Default
UserInputService.MouseIconEnabled = true

local function updateMouseLock()
    local character = LocalPlayer.Character
    if not character then return end
    local head = character:FindFirstChild("Head")
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if not head or not humanoid then return end

    local isFirstPerson = (Camera.CFrame.Position - head.Position).Magnitude < 1
    local isRagdolled = humanoid:GetState() == Enum.HumanoidStateType.Physics
    
    local isShiftLock = not isRagdolled and humanoid.AutoRotate == false 
    local rightClicking = UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2)

    if isRagdolled then
        UserInputService.MouseBehavior = Enum.MouseBehavior.Default
        UserInputService.MouseIconEnabled = true
    elseif isShiftLock or isFirstPerson then
        UserInputService.MouseBehavior = Enum.MouseBehavior.LockCenter
        UserInputService.MouseIconEnabled = true
    else
        if rightClicking then
             -- Allow Orbit
        else
             UserInputService.MouseBehavior = Enum.MouseBehavior.Default
             UserInputService.MouseIconEnabled = true
        end
    end
end

local FOVCircle 
pcall(function()
    FOVCircle = Drawing.new("Circle")
    FOVCircle.Thickness = 1
    FOVCircle.Filled = false
    FOVCircle.Transparency = 1
    FOVCircle.Color = Color3.new(1,1,1)
    FOVCircle.Visible = false
end)

local FOVGui = Instance.new("ScreenGui")
FOVGui.Name = "ObsidianFOV_Rotate"
FOVGui.Parent = game:GetService("CoreGui")
FOVGui.IgnoreGuiInset = true 
FOVGui.DisplayOrder = -1 

local FOVFrame = Instance.new("Frame")
FOVFrame.Name = "Circle"
FOVFrame.Parent = FOVGui
FOVFrame.AnchorPoint = Vector2.new(0.5, 0.5)
FOVFrame.BackgroundColor3 = Color3.new(1, 1, 1)
FOVFrame.BackgroundTransparency = 0
FOVFrame.Visible = false
FOVFrame.BorderSizePixel = 0

local FOVCorner = Instance.new("UICorner")
FOVCorner.CornerRadius = UDim.new(1, 0)
FOVCorner.Parent = FOVFrame

local FOVGradient = Instance.new("UIGradient")
FOVGradient.Parent = FOVFrame
FOVGradient.Color = ColorSequence.new{
    ColorSequenceKeypoint.new(0.00, Color3.fromRGB(255, 255, 255)),
    ColorSequenceKeypoint.new(1.00, Color3.fromRGB(150, 150, 150))
}
FOVGradient.Transparency = NumberSequence.new{
    NumberSequenceKeypoint.new(0.00, 0.3),
    NumberSequenceKeypoint.new(0.50, 0.5),
    NumberSequenceKeypoint.new(1.00, 0.8)
}

local RotationAngle = 0
local AnimTick = tick()

local hue = 0
local FOV_RotationAngle = 0
local FOV_Tick = tick()
-- esp logic
local Workspace, RunService, Players, CoreGui, Lighting = cloneref(game:GetService("Workspace")), cloneref(game:GetService("RunService")), cloneref(game:GetService("Players")), game:GetService("CoreGui"), cloneref(game:GetService("Lighting"))

-- [ESP Logic Retained]
local ESP = {
    Enabled = Toggles.ESPEnabled.Value,
    TeamCheck = Toggles.ESPHideTeam.Value,
    MaxDistance = Options.ESPMaxDistance.Value,
    FontSize = 11 + (Options.ESPFontSize.Value - 1) * 22,
    FadeOut = {OnDistance = true, OnDeath = false, OnLeave = false},
    Drawing = {
        Chams = { Enabled = Toggles.ChamsEnabled.Value, Thermal = true, FillRGB = Options.ChamsColor.Value, Fill_Transparency = 50, OutlineRGB = Options.ChamsColor.Value, Outline_Transparency = 0, VisibleCheck = true },
        Names = { Enabled = Toggles.ESPName.Value, RGB = Options.ESPColor.Value },
        Flags = { Enabled = true },
        Distances = { Enabled = Toggles.ESPDistance.Value, Position = "Bottom", RGB = Options.ESPColor.Value },
        Weapons = { Enabled = Toggles.ESPTool.Value, WeaponTextRGB = Options.ESPColor.Value, GradientRGB1 = Color3.fromRGB(255, 255, 255), GradientRGB2 = Color3.fromRGB(119, 120, 255) },
        Healthbar = { Enabled = Toggles.ESPHealth.Value, HealthText = true, Lerp = false, HealthTextRGB = Options.ESPColor.Value, Width = 2.5, Gradient = true, GradientRGB1 = Color3.fromRGB(200, 0, 0), GradientRGB2 = Color3.fromRGB(60, 60, 125), GradientRGB3 = Color3.fromRGB(119, 120, 255) },
        Boxes = { Animate = true, RotationSpeed = 300, Gradient = false, GradientRGB1 = Options.ESPBoxColor.Value, GradientRGB2 = Color3.fromRGB(0, 0, 0), GradientFill = true, GradientFillRGB1 = Options.ESPBoxColor.Value, GradientFillRGB2 = Color3.fromRGB(0, 0, 0), Filled = { Enabled = Toggles.ESPBoxFilled.Value, Transparency = Options.ESPBoxFillTransparency.Value * 100, RGB = Options.ESPBoxFillColor.Value }, Full = { Enabled = Toggles.ESPBox.Value, RGB = Color3.fromRGB(255, 255, 255) }, Corner = { Enabled = Toggles.ESPBox.Value, RGB = Color3.fromRGB(255, 255, 255) } };
    };
    Connections = { RunService = RunService }; Fonts = {};
}
Toggles.ESPEnabled:OnChanged(function(v) ESP.Enabled = v end); Toggles.ESPHideTeam:OnChanged(function(v) ESP.TeamCheck = v end)
Toggles.ChamsEnabled:OnChanged(function(v) ESP.Drawing.Chams.Enabled = v end); Toggles.ESPName:OnChanged(function(v) ESP.Drawing.Names.Enabled = v end)
Toggles.ESPDistance:OnChanged(function(v) ESP.Drawing.Distances.Enabled = v end); Toggles.ESPHealth:OnChanged(function(v) ESP.Drawing.Healthbar.Enabled = v end)
Toggles.ESPTool:OnChanged(function(v) ESP.Drawing.Weapons.Enabled = v end); Toggles.ESPBox:OnChanged(function(v) ESP.Drawing.Boxes.Full.Enabled = v; ESP.Drawing.Boxes.Corner.Enabled = v end)
Toggles.ESPBoxFilled:OnChanged(function(v) ESP.Drawing.Boxes.Filled.Enabled = v end); Options.ESPBoxFillTransparency:OnChanged(function(v) ESP.Drawing.Boxes.Filled.Transparency = v * 100 end)
Options.ESPBoxFillColor:OnChanged(function(v) ESP.Drawing.Boxes.Filled.RGB = v end); Options.ESPBoxColor:OnChanged(function(v) ESP.Drawing.Boxes.GradientRGB1 = v; ESP.Drawing.Boxes.GradientFillRGB1 = v end)
Options.ESPColor:OnChanged(function(v) ESP.Drawing.Weapons.WeaponTextRGB = v; ESP.Drawing.Healthbar.HealthTextRGB = v; ESP.Drawing.Names.RGB = v; ESP.Drawing.Distances.RGB = v end)
Options.ChamsColor:OnChanged(function(v) ESP.Drawing.Chams.FillRGB = v; ESP.Drawing.Chams.OutlineRGB = v end); Options.ESPFontSize:OnChanged(function(v) ESP.FontSize = 11 + (v - 1) * 22 end)
Options.ESPMaxDistance:OnChanged(function(v) ESP.MaxDistance = v end)

local Euphoria = ESP.Connections; local lplayer = Players.LocalPlayer; local Cam = Workspace.CurrentCamera; local RotationAngleESP, Tick = -45, tick()
local Weapon_Icons = { ["Wooden Bow"] = "http://www.roblox.com/asset/?id=17677465400", ["Crossbow"] = "http://www.roblox.com/asset/?id=17677473017", ["Salvaged SMG"] = "http://www.roblox.com/asset/?id=17677463033", ["Salvaged AK47"] = "http://www.roblox.com/asset/?id=17677455113", ["Salvaged AK74u"] = "http://www.roblox.com/asset/?id=17677442346", ["Salvaged M14"] = "http://www.roblox.com/asset/?id=17677444642", ["Salvaged Python"] = "http://www.roblox.com/asset/?id=17677451737", ["Military PKM"] = "http://www.roblox.com/asset/?id=17677449448", ["Military M4A1"] = "http://www.roblox.com/asset/?id=17677479536", ["Bruno's M4A1"] = "http://www.roblox.com/asset/?id=17677471185", ["Military Barrett"] = "http://www.roblox.com/asset/?id=17677482998", ["Salvaged Skorpion"] = "http://www.roblox.com/asset/?id=17677459658", ["Salvaged Pump Action"] = "http://www.roblox.com/asset/?id=17677457186", ["Military AA12"] = "http://www.roblox.com/asset/?id=17677475227", ["Salvaged Break Action"] = "http://www.roblox.com/asset/?id=17677468751", ["Salvaged Pipe Rifle"] = "http://www.roblox.com/asset/?id=17677468751", ["Salvaged P250"] = "http://www.roblox.com/asset/?id=17677447257", ["Nail Gun"] = "http://www.roblox.com/asset/?id=17677484756" }
local Functions = {}
do
    function Functions:Create(Class, Properties)
        local _Instance = typeof(Class) == 'string' and Instance.new(Class) or Class
        for Property, Value in pairs(Properties) do _Instance[Property] = Value end; return _Instance
    end
    function Functions:FadeOutOnDist(element, distance)
        local transparency = math.max(0.1, 1 - (distance / ESP.MaxDistance))
        if element:IsA("TextLabel") then element.TextTransparency = 1 - transparency
        elseif element:IsA("ImageLabel") then element.ImageTransparency = 1 - transparency
        elseif element:IsA("UIStroke") then element.Transparency = 1 - transparency
        elseif element:IsA("Frame") or element:IsA("Highlight") then 
             if element:IsA("Frame") then element.BackgroundTransparency = 1 - transparency
             else element.FillTransparency = 1 - transparency; element.OutlineTransparency = 1 - transparency end
        end
    end
end
do
    local ScreenGui = Functions:Create("ScreenGui", {Parent = CoreGui, Name = "ESPHolder"}); local DupeCheck = function(plr) if ScreenGui:FindFirstChild(plr.Name) then ScreenGui[plr.Name]:Destroy() end end
    local ESPFunc = function(plr)
        coroutine.wrap(DupeCheck)(plr)
        local Folder = Functions:Create("Folder", {Parent = ScreenGui, Name = plr.Name})
        local Name = Functions:Create("TextLabel", {Parent = Folder, Position = UDim2.new(0.5,0,0,-11), Size=UDim2.new(0,100,0,20), AnchorPoint=Vector2.new(0.5,0.5), BackgroundTransparency=1, TextColor3=Color3.fromRGB(255,255,255), Font=Enum.Font.Code, TextSize=ESP.FontSize, TextStrokeTransparency=0, TextStrokeColor3=Color3.fromRGB(0,0,0), RichText=true})
        local Distance = Functions:Create("TextLabel", {Parent=Folder, Position=UDim2.new(0.5,0,0,11), Size=UDim2.new(0,100,0,20), AnchorPoint=Vector2.new(0.5,0.5), BackgroundTransparency=1, TextColor3=Color3.fromRGB(255,255,255), Font=Enum.Font.Code, TextSize=ESP.FontSize, TextStrokeTransparency=0, TextStrokeColor3=Color3.fromRGB(0,0,0), RichText=true})
        local Weapon = Functions:Create("TextLabel", {Parent=Folder, Position=UDim2.new(0.5,0,0,31), Size=UDim2.new(0,100,0,20), AnchorPoint=Vector2.new(0.5,0.5), BackgroundTransparency=1, TextColor3=Color3.fromRGB(255,255,255), Font=Enum.Font.Code, TextSize=ESP.FontSize, TextStrokeTransparency=0, TextStrokeColor3=Color3.fromRGB(0,0,0), RichText=true})
        local Box = Functions:Create("Frame", {Parent=Folder, BackgroundColor3=Color3.fromRGB(0,0,0), BackgroundTransparency=0.75, BorderSizePixel=0})
        local Gradient1 = Functions:Create("UIGradient", {Parent=Box, Enabled=ESP.Drawing.Boxes.GradientFill, Color=ColorSequence.new{ColorSequenceKeypoint.new(0,ESP.Drawing.Boxes.GradientFillRGB1), ColorSequenceKeypoint.new(1,ESP.Drawing.Boxes.GradientFillRGB2)}})
        local Outline = Functions:Create("UIStroke", {Parent=Box, Enabled=ESP.Drawing.Boxes.Gradient, Transparency=0, Color=Color3.fromRGB(255,255,255), LineJoinMode=Enum.LineJoinMode.Miter})
        local Gradient2 = Functions:Create("UIGradient", {Parent=Outline, Enabled=ESP.Drawing.Boxes.Gradient, Color=ColorSequence.new{ColorSequenceKeypoint.new(0,ESP.Drawing.Boxes.GradientRGB1), ColorSequenceKeypoint.new(1,ESP.Drawing.Boxes.GradientRGB2)}})
        local Healthbar = Functions:Create("Frame", {Parent=Folder, BackgroundColor3=Color3.fromRGB(255,255,255), BackgroundTransparency=0})
        local BehindHealthbar = Functions:Create("Frame", {Parent=Folder, ZIndex=-1, BackgroundColor3=Color3.fromRGB(0,0,0), BackgroundTransparency=0}); Functions:Create("UIGradient", {Parent=Healthbar, Enabled=ESP.Drawing.Healthbar.Gradient, Rotation=-90, Color=ColorSequence.new{ColorSequenceKeypoint.new(0,ESP.Drawing.Healthbar.GradientRGB1), ColorSequenceKeypoint.new(0.5,ESP.Drawing.Healthbar.GradientRGB2), ColorSequenceKeypoint.new(1,ESP.Drawing.Healthbar.GradientRGB3)}})
        local HealthText = Functions:Create("TextLabel", {Parent=Folder, Position=UDim2.new(0.5,0,0,31), Size=UDim2.new(0,100,0,20), AnchorPoint=Vector2.new(0.5,0.5), BackgroundTransparency=1, TextColor3=Color3.fromRGB(255,255,255), Font=Enum.Font.Code, TextSize=ESP.FontSize, TextStrokeTransparency=0, TextStrokeColor3=Color3.fromRGB(0,0,0)})
        local Chams = Functions:Create("Highlight", {Parent=Folder, FillTransparency=1, OutlineTransparency=0, OutlineColor=Color3.fromRGB(119,120,255), DepthMode="AlwaysOnTop"})
        local WeaponIcon = Functions:Create("ImageLabel", {Parent=Folder, BackgroundTransparency=1, BorderColor3=Color3.fromRGB(0,0,0), BorderSizePixel=0, Size=UDim2.new(0,40,0,40)}); Functions:Create("UIGradient", {Parent=WeaponIcon, Rotation=-90, Enabled=ESP.Drawing.Weapons.Gradient, Color=ColorSequence.new{ColorSequenceKeypoint.new(0,ESP.Drawing.Weapons.GradientRGB1), ColorSequenceKeypoint.new(1,ESP.Drawing.Weapons.GradientRGB2)}})
        local LeftTop = Functions:Create("Frame", {Parent=Folder, BackgroundColor3=ESP.Drawing.Boxes.Corner.RGB}); local LeftSide = Functions:Create("Frame", {Parent=Folder, BackgroundColor3=ESP.Drawing.Boxes.Corner.RGB}); local RightTop = Functions:Create("Frame", {Parent=Folder, BackgroundColor3=ESP.Drawing.Boxes.Corner.RGB}); local RightSide = Functions:Create("Frame", {Parent=Folder, BackgroundColor3=ESP.Drawing.Boxes.Corner.RGB}); local BottomSide = Functions:Create("Frame", {Parent=Folder, BackgroundColor3=ESP.Drawing.Boxes.Corner.RGB}); local BottomDown = Functions:Create("Frame", {Parent=Folder, BackgroundColor3=ESP.Drawing.Boxes.Corner.RGB}); local BottomRightSide = Functions:Create("Frame", {Parent=Folder, BackgroundColor3=ESP.Drawing.Boxes.Corner.RGB}); local BottomRightDown = Functions:Create("Frame", {Parent=Folder, BackgroundColor3=ESP.Drawing.Boxes.Corner.RGB})
        local Updater = function()
            local Connection; 
            local function HideESP()
                 Box.Visible=false; Name.Visible=false; Distance.Visible=false; Weapon.Visible=false; Healthbar.Visible=false; BehindHealthbar.Visible=false; HealthText.Visible=false; WeaponIcon.Visible=false; Chams.Enabled=false; LeftTop.Visible=false; LeftSide.Visible=false; BottomSide.Visible=false; BottomDown.Visible=false; RightTop.Visible=false; RightSide.Visible=false; BottomRightSide.Visible=false; BottomRightDown.Visible=false
                 if not plr or not plr.Parent then if Connection then Connection:Disconnect() end; Folder:Destroy() end
            end
            Connection = Euphoria.RunService.RenderStepped:Connect(function()
                 if not plr or not plr.Parent then HideESP(); if Connection then Connection:Disconnect() end; return end
                 local espPlayerWL = (Options.ESPPlayerWhitelist and Options.ESPPlayerWhitelist.Value) or {}
                 local espTeamWL = (Options.ESPTeamWhitelist and Options.ESPTeamWhitelist.Value) or {}
                 if not ESP.Enabled then HideESP(); return end
                 if selectionContains(espPlayerWL, plr) or (plr.Team and selectionContains(espTeamWL, plr.Team)) then HideESP(); return end
                 
                 if plr.Character and plr.Character:FindFirstChild("HumanoidRootPart") and plr.Character:FindFirstChild("Humanoid") then
                     local HRP=plr.Character.HumanoidRootPart; local Humanoid=plr.Character.Humanoid; if Humanoid.Health<=0 then HideESP(); return end
                     local Pos, OnScreen = Cam:WorldToScreenPoint(HRP.Position); local Dist=(Cam.CFrame.Position-HRP.Position).Magnitude/3.57
                     
                     if not Toggles.ESPThroughWall.Value then
                         if not isVisible(HRP) then
                             HideESP(); return
                         end
                     end
                     
                     if OnScreen and Dist <= ESP.MaxDistance then
                        local scaleFactor = (HRP.Size.Y*Cam.ViewportSize.Y)/(Pos.Z*2); local w, h = 3*scaleFactor, 4.5*scaleFactor
                        if ESP.FadeOut.OnDistance then Functions:FadeOutOnDist(Box, Dist); Functions:FadeOutOnDist(Name, Dist); Functions:FadeOutOnDist(Distance, Dist); Functions:FadeOutOnDist(Healthbar, Dist); Functions:FadeOutOnDist(BehindHealthbar, Dist); Functions:FadeOutOnDist(HealthText, Dist); Functions:FadeOutOnDist(Weapon, Dist); Functions:FadeOutOnDist(WeaponIcon, Dist); Functions:FadeOutOnDist(Chams, Dist) end

                        if ESP.TeamCheck and plr ~= lplayer and ((lplayer.Team ~= plr.Team and plr.Team) or (not lplayer.Team and not plr.Team)) then
                            Chams.Adornee = plr.Character; Chams.Enabled = ESP.Drawing.Chams.Enabled; Chams.FillColor = Toggles.RainbowESP.Value and Color3.fromHSV(hue,1,1) or Options.ChamsColor.Value; Chams.OutlineColor = Chams.FillColor
                            Chams.DepthMode = Toggles.ESPThroughWall.Value and Enum.HighlightDepthMode.AlwaysOnTop or Enum.HighlightDepthMode.Occluded
                            
                            if ESP.Drawing.Chams.Thermal then local br = math.atan(math.sin(tick()*2))*2/math.pi; Chams.FillTransparency = ESP.Drawing.Chams.Fill_Transparency*br*0.01; Chams.OutlineTransparency = ESP.Drawing.Chams.Outline_Transparency*br*0.01
                            else Chams.FillTransparency=ESP.Drawing.Chams.Fill_Transparency*0.01; Chams.OutlineTransparency=ESP.Drawing.Chams.Outline_Transparency*0.01 end
                            
                            do 
                                LeftTop.Visible = ESP.Drawing.Boxes.Corner.Enabled; LeftTop.Position = UDim2.new(0, Pos.X - w / 2, 0, Pos.Y - h / 2); LeftTop.Size = UDim2.new(0, w / 5, 0, 1)
                                LeftSide.Visible = ESP.Drawing.Boxes.Corner.Enabled; LeftSide.Position = UDim2.new(0, Pos.X - w / 2, 0, Pos.Y - h / 2); LeftSide.Size = UDim2.new(0, 1, 0, h / 5)
                                BottomSide.Visible = ESP.Drawing.Boxes.Corner.Enabled; BottomSide.Position = UDim2.new(0, Pos.X - w / 2, 0, Pos.Y + h / 2); BottomSide.Size = UDim2.new(0, 1, 0, h / 5); BottomSide.AnchorPoint = Vector2.new(0, 5) 
                                BottomDown.Visible = ESP.Drawing.Boxes.Corner.Enabled; BottomDown.Position = UDim2.new(0, Pos.X - w / 2, 0, Pos.Y + h / 2); BottomDown.Size = UDim2.new(0, w / 5, 0, 1); BottomDown.AnchorPoint = Vector2.new(0, 1)
                                RightTop.Visible = ESP.Drawing.Boxes.Corner.Enabled; RightTop.Position = UDim2.new(0, Pos.X + w / 2, 0, Pos.Y - h / 2); RightTop.Size = UDim2.new(0, w / 5, 0, 1); RightTop.AnchorPoint = Vector2.new(1, 0)
                                RightSide.Visible = ESP.Drawing.Boxes.Corner.Enabled; RightSide.Position = UDim2.new(0, Pos.X + w / 2 - 1, 0, Pos.Y - h / 2); RightSide.Size = UDim2.new(0, 1, 0, h / 5); RightSide.AnchorPoint = Vector2.new(0, 0)
                                BottomRightSide.Visible = ESP.Drawing.Boxes.Corner.Enabled; BottomRightSide.Position = UDim2.new(0, Pos.X + w / 2, 0, Pos.Y + h / 2); BottomRightSide.Size = UDim2.new(0, 1, 0, h / 5); BottomRightSide.AnchorPoint = Vector2.new(1, 1)
                                BottomRightDown.Visible = ESP.Drawing.Boxes.Corner.Enabled; BottomRightDown.Position = UDim2.new(0, Pos.X + w / 2, 0, Pos.Y + h / 2); BottomRightDown.Size = UDim2.new(0, w / 5, 0, 1); BottomRightDown.AnchorPoint = Vector2.new(1, 1)
                            end

                            do
                                Box.Position = UDim2.new(0, Pos.X - w/2, 0, Pos.Y - h/2); Box.Size = UDim2.new(0, w, 0, h); Box.Visible = ESP.Drawing.Boxes.Full.Enabled
                                if ESP.Drawing.Boxes.Filled.Enabled then 
                                    Box.BackgroundColor3 = Toggles.RainbowESP.Value and Color3.fromHSV(hue,1,1) or Options.ESPBoxFillColor.Value 
                                    if ESP.Drawing.Boxes.GradientFill then Box.BackgroundTransparency = ESP.Drawing.Boxes.Filled.Transparency/100 else Box.BackgroundTransparency = 1 end
                                    Box.BorderSizePixel = 1
                                else Box.BackgroundTransparency = 1 end
                                RotationAngleESP = RotationAngleESP + (tick()-Tick)*ESP.Drawing.Boxes.RotationSpeed*math.cos(math.pi/4*tick()-math.pi/2) 
                                if ESP.Drawing.Boxes.Animate then Gradient1.Rotation=RotationAngleESP; Gradient2.Rotation=RotationAngleESP else Gradient1.Rotation=-45; Gradient2.Rotation=-45 end
                                Tick = tick()
                            end
                            
                            local hp=Humanoid.Health/Humanoid.MaxHealth; 
                            Healthbar.Visible=ESP.Drawing.Healthbar.Enabled; Healthbar.Position=UDim2.new(0,Pos.X-w/2-6,0,Pos.Y-h/2+h*(1-hp)); Healthbar.Size=UDim2.new(0,ESP.Drawing.Healthbar.Width,0,h*hp); BehindHealthbar.Visible=ESP.Drawing.Healthbar.Enabled; BehindHealthbar.Position=UDim2.new(0,Pos.X-w/2-6,0,Pos.Y-h/2); BehindHealthbar.Size=UDim2.new(0,ESP.Drawing.Healthbar.Width,0,h)
                            
                            if ESP.Drawing.Healthbar.HealthText then 
                                HealthText.Position=UDim2.new(0,Pos.X-w/2-6,0,Pos.Y-h/2+h*(1-hp)+3)
                                HealthText.Text=tostring(math.floor(hp*100))
                                HealthText.Visible=Humanoid.Health<Humanoid.MaxHealth; 
                                HealthText.TextColor3=ESP.Drawing.Healthbar.HealthTextRGB; 
                                HealthText.TextSize=ESP.FontSize 
                            else
                                HealthText.Visible=false
                            end

                            Name.Visible=ESP.Drawing.Names.Enabled; Name.TextColor3=Toggles.RainbowESP.Value and Color3.fromHSV(hue,1,1) or Options.ESPColor.Value; Name.TextSize=ESP.FontSize; Name.Text = lplayer:IsFriendsWith(plr.UserId) and "F" or plr.Name; Name.Position=UDim2.new(0,Pos.X,0,Pos.Y-h/2-9)
                            
                            local bOff = Pos.Y+h/2; 
                            if ESP.Drawing.Distances.Enabled then Distance.Visible=true; Distance.Position=UDim2.new(0,Pos.X,0,bOff+7); Distance.Text=math.floor(Dist).."m"; Distance.TextSize=ESP.FontSize; bOff=bOff+12 else Distance.Visible=false end
                            if ESP.Drawing.Weapons.Enabled then local tool = plr.Character:FindFirstChildOfClass("Tool"); Weapon.Visible=true; Weapon.Position=UDim2.new(0,Pos.X,0,bOff+11); Weapon.Text=tool and tool.Name or "none"; Weapon.TextSize=ESP.FontSize; WeaponIcon.Visible=true; WeaponIcon.Position=UDim2.new(0,Pos.X-21,0,bOff+8); WeaponIcon.Image=tool and Weapon_Icons[tool.Name] or "" else Weapon.Visible=false; WeaponIcon.Visible=false end
                        else HideESP() end
                     else HideESP() end
                 else HideESP() end
            end)
        end
        coroutine.wrap(Updater)()
    end
    for _, v in pairs(Players:GetPlayers()) do if v ~= lplayer then coroutine.wrap(ESPFunc)(v) end end
    Players.PlayerAdded:Connect(function(v) coroutine.wrap(ESPFunc)(v) end)
end

-- ===============================================
-- === GHOST & PREDICTION UTILITY (TXT.46) ===
-- ===============================================
local GhostCharacter = nil
local GhostPartsCache = {}
local function ClearGhost()
    if GhostCharacter then GhostCharacter:Destroy(); GhostCharacter = nil end
    GhostPartsCache = {}
end
local function UpdateGhost(target, predictedPos)
    if not target or not target:FindFirstChild("HumanoidRootPart") then ClearGhost(); return end
    if not GhostCharacter or GhostCharacter.Name ~= target.Name .. "_Ghost" then
        ClearGhost()
        GhostCharacter = Instance.new("Model")
        GhostCharacter.Name = target.Name .. "_Ghost"
        GhostCharacter.Parent = workspace.CurrentCamera 
        for _, v in pairs(target:GetChildren()) do
            if v:IsA("BasePart") and v.Name ~= "HumanoidRootPart" then
                local clone = v:Clone()
                clone:ClearAllChildren()
                clone.Parent = GhostCharacter
                clone.Anchored = true; clone.CanCollide = false; clone.CanTouch = false; clone.CanQuery = false
                clone.Material = Enum.Material.Plastic; clone.Color = Options.GhostColor.Value; clone.Transparency = Options.GhostTransparency.Value
                GhostPartsCache[v.Name] = clone
                if v:IsA("MeshPart") then clone.TextureID = "" end
            end
        end
    end
    local currentHRP = target:FindFirstChild("HumanoidRootPart")
    if currentHRP then
        local offset = predictedPos - currentHRP.Position
        local gColor = Options.GhostColor.Value
        local gTrans = Options.GhostTransparency.Value
        for origName, clonePart in pairs(GhostPartsCache) do
            local originalPart = target:FindFirstChild(origName)
            if originalPart then
                clonePart.CFrame = originalPart.CFrame + offset
                clonePart.Color = gColor; clonePart.Transparency = gTrans
            end
        end
    end
end

-- ===============================================
-- === SYNCED TARGET FINDER (UNIFIED) ===
-- ===============================================
local bodyParts = { "Head", "UpperTorso", "LowerTorso", "HumanoidRootPart", "LeftUpperArm", "RightUpperArm", "LeftLowerArm", "RightLowerArm", "LeftHand", "RightHand", "LeftUpperLeg", "RightUpperLeg", "LeftLowerLeg", "RightLowerLeg", "LeftFoot", "RightFoot", "Torso", "Left Arm", "Right Arm", "Left Leg", "Right Leg" }

-- One logic to rule them all (Ensures FOV accuracy and sync)
local function getClosestTarget()
    local closest, closestDist = nil, math.huge
    local mousePos = UserInputService:GetMouseLocation()
    
    local visibleCheck = (Toggles.WallCheck and Toggles.WallCheck.Value)
    
    -- PLAYER LOOP
    for _, player in pairs(Players:GetPlayers()) do
        if player == LocalPlayer then continue end
        local character = player.Character
        if not character then continue end
        
        -- PROTECTED WHITELIST V3
        local playerWL = (Options.PlayerWhitelist and Options.PlayerWhitelist.Value) or {}
        local teamWL = (Options.TeamWhitelist and Options.TeamWhitelist.Value) or {}
        if selectionContains(playerWL, player) then continue end
        if player.Team and selectionContains(teamWL, player.Team) then continue end

        -- TEAM CHECK
        if Toggles.TeamCheck.Value and LocalPlayer.Team and player.Team == LocalPlayer.Team then continue end

        -- [[ NEW HOSTILE CHECK ]] --
        if Toggles.HostileCheck.Value then
            if not character:FindFirstChildOfClass("Tool") then
                continue -- Skip if they are not holding a tool
            end
        end

        -- CHECKS
        if Toggles.ForceFieldCheck.Value and character:FindFirstChildOfClass("ForceField") then continue end
        local hrp = character:FindFirstChild("HumanoidRootPart")
        if not hrp then continue end

        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if not humanoid or humanoid.Health <= 0 then continue end
        if Toggles.HealthCheck.Value and humanoid.Health < Options.MinHealth.Value then continue end

        local distance = (Camera.CFrame.Position - hrp.Position).Magnitude
        -- Use Slider Max Distance
        if distance > Options.MaxDistance.Value then continue end
        
        -- PART VISIBILITY
        local charBestDist = math.huge
        local part = getAimPart(character) -- Returns Aimbot Part
        
        if part then
            if visibleCheck and not isVisible(part) then 
                part = nil -- Wallcheck enabled, part hidden
            end
        end

        if part then
             local screenPos, onScreen = Camera:WorldToViewportPoint(part.Position)
             -- Use strict "OnScreen" check for reliable FOV logic
             if onScreen then
                 -- Correctly calculated Distance to Crosshair (center screen logic from mouse)
                 charBestDist = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude 
             end
        end
        
        -- ACCURATE FOV CHECK
        if charBestDist <= Options.FOVRadius.Value and charBestDist < closestDist then
            closest = character
            closestDist = charBestDist
        end
    end
    
    -- NPC LOGIC (Generic)
    if Toggles.AimNPCs.Value then
        -- Iterate Workspace to find ANY generic Humanoid character
        for _, npc in pairs(workspace:GetChildren()) do
             local playerFromChar = Players:GetPlayerFromCharacter(npc)
             -- Make sure it's a Model, has Humanoid/Root, and is not a player
             if not playerFromChar and npc:IsA("Model") and npc ~= LocalPlayer.Character then
                  local humanoid = npc:FindFirstChild("Humanoid") or npc:FindFirstChildOfClass("Humanoid")
                  local root = npc:FindFirstChild("HumanoidRootPart")
                  
                  if humanoid and root and humanoid.Health > 0 then
                       
                      -- [[ NPC HOSTILE CHECK ]] --
                      if Toggles.HostileCheck.Value then
                          if not npc:FindFirstChildOfClass("Tool") then
                               continue 
                          end
                      end
                      
                      local charBestDist = math.huge
                      local part = getAimPart(npc)
                      if part and (not visibleCheck or isVisible(part)) then
                            local screenPos, onScreen = Camera:WorldToViewportPoint(part.Position)
                            if onScreen then
                                 local dist = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
                                 if dist <= Options.FOVRadius.Value and dist < closestDist then closest = npc; closestDist = dist end
                            end
                      end
                  end
             end
        end
    end
    
    return closest
end

-- ===============================================
-- === RAGE TARGET SELECTOR (UNIVERSAL NPC FIX) ===
-- ===============================================
local function getRageTarget()
    local closest, closestDist = nil, math.huge
    local maxAngle = Options.RageFOV.Value / 2 
    
    local camCF = workspace.CurrentCamera.CFrame
    local camPos = camCF.Position
    local camLook = camCF.LookVector

    -- [HELPER] CHECKS A SINGLE MODEL VALIDITY
    local function checkEntity(character, player)
        if not character or not character.Parent then return end
        
        -- 1. Check for Humanoid (Living Check)
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if not humanoid or humanoid.Health <= 0 then return end

        -- 2. Team/Player Logic
        if player then
            if Toggles.TeamCheck.Value and LocalPlayer.Team and player.Team == LocalPlayer.Team then return end
        else
            -- NPC Specific Logic
            -- Apply Hostile Check to NPCs (only target if holding tool)
            if Toggles.HostileCheck.Value then
                if not character:FindFirstChildOfClass("Tool") then return end
            end
        end

        -- 3. Universal Health & ForceField
        if Toggles.HealthCheck.Value and humanoid.Health < Options.MinHealth.Value then return end
        if Toggles.ForceFieldCheck.Value and character:FindFirstChildOfClass("ForceField") then return end

        -- 4. [SMART PART FINDER] - Fixes NPCs not being aimed at
        -- Many NPCs don't have 'HumanoidRootPart'. This tries every common option.
        local targetPart = nil
        local requested = Options.RageTargetPart.Value -- "Head" or "HumanoidRootPart" or "Random"

        -- Try User Preference
        if requested == "Head" then targetPart = character:FindFirstChild("Head") end
        if (requested == "HumanoidRootPart" or requested == "Random") then 
            targetPart = character:FindFirstChild("HumanoidRootPart") or character:FindFirstChild("Torso") or character:FindFirstChild("UpperTorso")
        end

        -- FALLBACK 1: Try PrimaryPart
        if not targetPart then targetPart = character.PrimaryPart end

        -- FALLBACK 2: Try Any Body Part
        if not targetPart then
            targetPart = character:FindFirstChild("Head") or character:FindFirstChild("Torso") or character:FindFirstChild("UpperTorso") or character:FindFirstChild("LowerTorso") or character:FindFirstChild("HumanoidRootPart")
        end

        -- FALLBACK 3: Try ANY Part (For weird non-humanoid monsters)
        if not targetPart then
            targetPart = character:FindFirstChildOfClass("BasePart") or character:FindFirstChildOfClass("MeshPart")
        end

        if not targetPart then return end

        -- 5. Calculations
        if Toggles.WallCheck.Value then
            -- Calls our modified isVisible function which handles 3rd Person Toggle
            if not isVisible(targetPart) then return end
        end

        local dist = (camPos - targetPart.Position).Magnitude
        if dist > Options.MaxDistance.Value then return end

        local dir = (targetPart.Position - camPos).Unit
        local angle = math.deg(math.acos(math.clamp(camLook:Dot(dir), -1, 1)))

        if angle <= maxAngle then
            if dist < closestDist then
                closest = character
                closestDist = dist
            end
        end
    end

    -- [[ PLAYER LOOP ]]
    local pWL = (Options.PlayerWhitelist and Options.PlayerWhitelist.Value) or {}
    local tWL = (Options.TeamWhitelist and Options.TeamWhitelist.Value) or {}

    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            if selectionContains(pWL, player) then continue end
            if player.Team and selectionContains(tWL, player.Team) then continue end
            checkEntity(player.Character, player)
        end
    end

    -- [[ NPC LOOP (DEEP SCAN) ]]
    if Toggles.AimNPCs.Value then
        -- Optimization: Scan Workspace children directly (Standard) + 1 level deep (Folders)
        -- We avoid :GetDescendants() for lag, but this catches 99% of NPCs inside folders.
        local children = workspace:GetChildren()
        for i = 1, #children do
            local child = children[i]
            
            -- Case A: NPC is directly in Workspace
            if child:IsA("Model") then
                if child ~= LocalPlayer.Character and not Players:GetPlayerFromCharacter(child) and child:FindFirstChildOfClass("Humanoid") then
                    checkEntity(child, nil)
                end
                
                -- Case B: Check Inside Model/Folder Groups (e.g. Workspace.Zombies.Regular)
                -- (Some games verify using folder contents)
                local sub = child:GetChildren()
                for x = 1, #sub do
                    local grandChild = sub[x]
                    if grandChild:IsA("Model") and grandChild:FindFirstChildOfClass("Humanoid") and not Players:GetPlayerFromCharacter(grandChild) then
                        checkEntity(grandChild, nil)
                    end
                end
            elseif child:IsA("Folder") then
                -- Case C: NPC is inside a Folder (Workspace.Mobs.Enemy)
                local sub = child:GetChildren()
                for x = 1, #sub do
                    local grandChild = sub[x]
                    if grandChild:IsA("Model") and grandChild:FindFirstChildOfClass("Humanoid") and not Players:GetPlayerFromCharacter(grandChild) then
                        checkEntity(grandChild, nil)
                    end
                end
            end
        end
    end

    return closest
end
-- ===============================================
-- === HOOKS (SILENT AIM) ===
-- ===============================================

-- Function to get the smoothed silent aim position
local function GetSmoothedSilentAimPosition()
    if not SilentAimSmoothedPos then return nil end
    return SilentAimSmoothedPos
end

-- Function to update silent aim smoothed position (called in main loop)
local function UpdateSilentAimSmoothing(dt)
    -- IF Rage Mode is enabled, skip smoothing for faster locking
    if Toggles.RageMode and Toggles.RageMode.Value then 
        if ScriptState.ClosestHitPart then
            local offsetVector = Vector3.new(Options.AimOffsetX and Options.AimOffsetX.Value or 0, Options.AimOffsetY and Options.AimOffsetY.Value or 0, Options.AimOffsetZ and Options.AimOffsetZ.Value or 0)
            local prediction = Options.SilentPrediction and Options.SilentPrediction.Value or 0
            SilentAimSmoothedPos = ScriptState.ClosestHitPart.Position + (ScriptState.ClosestHitPart.AssemblyLinearVelocity * prediction) + offsetVector
            return
        end
    end

    if not Toggles.SilentAim.Value or not Toggles.AimbotEnabled.Value then
        SilentAimSmoothedPos = nil
        SilentAimLastTargetPart = nil
        SilentAimInitialized = false
        return
    end
    
    local HitPart = ScriptState.ClosestHitPart
    if not HitPart then
        SilentAimSmoothedPos = nil
        SilentAimLastTargetPart = nil
        SilentAimInitialized = false
        return
    end
    
    -- Calculate target position with prediction and offset
    local offsetVector = Vector3.new(
        Options.AimOffsetX and Options.AimOffsetX.Value or 0, 
        Options.AimOffsetY and Options.AimOffsetY.Value or 0, 
        Options.AimOffsetZ and Options.AimOffsetZ.Value or 0
    )
    local velocity = HitPart.AssemblyLinearVelocity or Vector3.new()
    local prediction = Options.SilentPrediction and Options.SilentPrediction.Value or 0.165
    
    local targetPos
    if Toggles.BulletTP and Toggles.BulletTP.Value then
        targetPos = HitPart.Position
    else
        targetPos = HitPart.Position + (velocity * prediction) + offsetVector
    end
    
    -- Get smoothness values
    local smoothX = Options.SilentSmoothnessX and Options.SilentSmoothnessX.Value or 1
    local smoothY = Options.SilentSmoothnessY and Options.SilentSmoothnessY.Value or 1
    
    -- If target changed or first time, initialize position
    if HitPart ~= SilentAimLastTargetPart or not SilentAimInitialized then
        SilentAimSmoothedPos = targetPos
        SilentAimLastTargetPart = HitPart
        SilentAimInitialized = true
        return
    end
    
    -- Apply smoothing (lerp toward target)
    if SilentAimSmoothedPos then
        -- Calculate smoothing factor (higher smoothness = slower movement)
        local smoothFactorX = 1 / math.max(1, smoothX)
        local smoothFactorY = 1 / math.max(1, smoothY)
        
        -- Apply different different smoothing to X/Z and Y axes
        local currentX = SilentAimSmoothedPos.X
        local currentY = SilentAimSmoothedPos.Y
        local currentZ = SilentAimSmoothedPos.Z
        
        local newX = currentX + (targetPos.X - currentX) * smoothFactorX
        local newY = currentY + (targetPos.Y - currentY) * smoothFactorY
        local newZ = currentZ + (targetPos.Z - currentZ) * smoothFactorX -- Z uses X smoothing
        
        SilentAimSmoothedPos = Vector3.new(newX, newY, newZ)
    else
        SilentAimSmoothedPos = targetPos
    end
end

local oldNamecall
oldNamecall = hookmetamethod(game, "__namecall", newcclosure(function(...)
    local Method = getnamecallmethod()
    local Arguments = {...}
    local self = Arguments[1]
    
    -- [[ BLOCKED METHODS LOGIC (Added) ]] --
    local blockedOptions = Options.BlockedMethods and Options.BlockedMethods.Value
    if blockedOptions and blockedOptions[Method] then
        return
    end
    
    -- Silent Aim Hook Logic (Includes Rage Mode support)
    local enabled = (Toggles.SilentAim.Value and Toggles.AimbotEnabled.Value) or (Toggles.RageMode.Value)
    
    local hitChance = CalculateChance((Options.SilentHitChance and Options.SilentHitChance.Value) or 100)
    
    if not checkcaller() and enabled and self == workspace and hitChance then
        local HitPart = ScriptState.ClosestHitPart 

        if HitPart then
            local currentMethod = Options.SilentMethod.Value
            
            -- Use smoothed position instead of direct calculation
            local targetPos = GetSmoothedSilentAimPosition()
            
            -- Fallback if smoothed position not available
            if not targetPos then
                local offsetVector = Vector3.new(
                    Options.AimOffsetX and Options.AimOffsetX.Value or 0, 
                    Options.AimOffsetY and Options.AimOffsetY.Value or 0, 
                    Options.AimOffsetZ and Options.AimOffsetZ.Value or 0
                )
                local velocity = HitPart.AssemblyLinearVelocity or Vector3.new()
                local prediction = Options.SilentPrediction and Options.SilentPrediction.Value or 0.165
                
                if Toggles.BulletTP and Toggles.BulletTP.Value then 
                    targetPos = HitPart.Position 
                else 
                    targetPos = HitPart.Position + (velocity * prediction) + offsetVector 
                end
            end

            -- No tracers here (Strictly removed)

            if Method == "FindPartOnRayWithIgnoreList" and currentMethod == "FindPartOnRayWithIgnoreList" then
                if ValidateArguments(Arguments, ExpectedArguments.FindPartOnRayWithIgnoreList) then
                    local Origin = Arguments[2].Origin
                    
                    if Toggles.BulletTP.Value and HitPart then
                        Origin = (HitPart.CFrame * CFrame.new(0, 0, 1)).Position
                    end

                    -- Use custom getDirection with Multiplier
                    local Direction = getDirection(Origin, targetPos)
                    Arguments[2] = Ray.new(Origin, Direction)
                    return oldNamecall(unpack(Arguments))
                end
            elseif Method == "FindPartOnRayWithWhitelist" and currentMethod == "FindPartOnRayWithWhitelist" then
                if ValidateArguments(Arguments, ExpectedArguments.FindPartOnRayWithWhitelist) then
                    local Origin = Arguments[2].Origin
                    
                    if Toggles.BulletTP.Value and HitPart then
                        Origin = (HitPart.CFrame * CFrame.new(0, 0, 1)).Position
                    end

                    local Direction = getDirection(Origin, targetPos)
                    Arguments[2] = Ray.new(Origin, Direction)
                    return oldNamecall(unpack(Arguments))
                end
            elseif (Method == "FindPartOnRay" or Method == "findPartOnRay") and (currentMethod == "FindPartOnRay") then
                if ValidateArguments(Arguments, ExpectedArguments.FindPartOnRay) then
                    local Origin = Arguments[2].Origin
                    
                    if Toggles.BulletTP.Value and HitPart then
                        Origin = (HitPart.CFrame * CFrame.new(0, 0, 1)).Position
                    end

                    local Direction = getDirection(Origin, targetPos)
                    Arguments[2] = Ray.new(Origin, Direction)
                    return oldNamecall(unpack(Arguments))
                end
            elseif Method == "Raycast" and currentMethod == "Raycast" then
                if ValidateArguments(Arguments, ExpectedArguments.Raycast) then
                    local Origin = Arguments[2]
                    
                    if Toggles.BulletTP.Value and HitPart then
                        Origin = (HitPart.CFrame * CFrame.new(0, 0, 1)).Position
                    end

                    Arguments[2] = Origin
                    Arguments[3] = getDirection(Origin, targetPos)
                    return oldNamecall(unpack(Arguments))
                end
            elseif Method == "ViewportPointToRay" and currentMethod == "ViewportPointToRay" then
                if ValidateArguments(Arguments, ExpectedArguments.ViewportPointToRay) then
                    local screenPos = Camera:WorldToViewportPoint(targetPos)
                    Arguments[2] = screenPos.X
                    Arguments[3] = screenPos.Y
                    return oldNamecall(unpack(Arguments))
                end
            elseif Method == "ScreenPointToRay" and currentMethod == "ScreenPointToRay" then
                if ValidateArguments(Arguments, ExpectedArguments.ScreenPointToRay) then
                    local screenPos = Camera:WorldToScreenPoint(targetPos)
                    Arguments[2] = screenPos.X
                    Arguments[3] = screenPos.Y
                    return oldNamecall(unpack(Arguments))
                end
            elseif Method == "FindPartOnRayWithIgnoreList" and currentMethod == "CounterBlox" then
                local Origin = Arguments[2].Origin
                
                if Toggles.BulletTP.Value and HitPart then
                    Origin = (HitPart.CFrame * CFrame.new(0, 0, 1)).Position
                end

                local Direction = getDirection(Origin, targetPos)
                Arguments[2] = Ray.new(Origin, Direction)
                return oldNamecall(unpack(Arguments))
            end
        end
    end
    return oldNamecall(...)
end))

local oldIndex
oldIndex = hookmetamethod(game, "__index", newcclosure(function(self, Index)
    -- Include RageMode Check in __index hook
    local enabled = (Toggles.SilentAim.Value and Toggles.AimbotEnabled.Value) or (Toggles.RageMode.Value)
    
    if self == mouse and not checkcaller() and enabled and Options.SilentMethod.Value == "Mouse.Hit/Target" then
        if ScriptState.ClosestHitPart then
            local HitPart = ScriptState.ClosestHitPart
            if not CalculateChance(Options.SilentHitChance and Options.SilentHitChance.Value or 100) then 
                return oldIndex(self, Index) 
            end
            
            if Index == "Target" or Index == "target" then 
                return HitPart
            elseif Index == "Hit" or Index == "hit" then 
                -- Use smoothed position
                local targetPos = GetSmoothedSilentAimPosition()
                
                if targetPos then
                    return CFrame.new(targetPos)
                else
                    -- Fallback
                    local velocity = HitPart.AssemblyLinearVelocity or Vector3.new()
                    local prediction = Options.SilentPrediction and Options.SilentPrediction.Value or 0.165
                    local targetCFrame = HitPart.CFrame + (velocity * prediction)
                    return targetCFrame
                end
            end
        end
    end
    return oldIndex(self, Index)
end))

local function isValidLockedTarget(targetChar)
    if not targetChar or not targetChar.Parent then return false end
    if Toggles.ForceFieldCheck.Value and targetChar:FindFirstChildOfClass("ForceField") then return false end
    local hrp = targetChar:FindFirstChild("HumanoidRootPart")
    if not hrp then return false end
    local humanoid = targetChar:FindFirstChildOfClass("Humanoid")
    if not humanoid or humanoid.Health <= 0 then return false end
    if Toggles.HealthCheck.Value and humanoid.Health < Options.MinHealth.Value then return false end
    
    -- [[ LOCKED TARGET RE-CHECK (HOSTILE) ]] --
    if Toggles.HostileCheck.Value then
        if not targetChar:FindFirstChildOfClass("Tool") then return false end
    end

    local distance = (Camera.CFrame.Position - hrp.Position).Magnitude
    if not Toggles.InfDistance.Value and distance > Options.MaxDistance.Value then return false end
    return true
end

UserInputService.JumpRequest:Connect(function()
    if Toggles.InfJump.Value and LocalPlayer.Character then LocalPlayer.Character:FindFirstChildOfClass("Humanoid"):ChangeState("Jumping") end
end)

UserInputService.InputBegan:Connect(function(input, processed) 
    if processed then return end
    if input.UserInputType == Enum.UserInputType.Keyboard then FlyKeysPressed[input.KeyCode] = true end
end)
UserInputService.InputEnded:Connect(function(input) if input.UserInputType == Enum.UserInputType.Keyboard then FlyKeysPressed[input.KeyCode] = false end end)


-- ===============================================
-- === MAIN LOOP ===
-- ===============================================
RunService.RenderStepped:Connect(function(dt)
    hue = (hue + dt * 0.25) % 1
    updateMouseLock()
    local mousePos = UserInputService:GetMouseLocation()
    
    -- Update Silent Aim Smoothing
    UpdateSilentAimSmoothing(dt)

    -- 1. VISUALS LOGIC UPDATE
    local fovColor = Toggles.RainbowFov.Value and Color3.fromHSV(hue, 1, 1) or Options.FovColor.Value
    local baseRadius = Options.FOVRadius.Value

    -- Standard Drawing Outline
    if FOVCircle then
        FOVCircle.Position = mousePos
        FOVCircle.Radius = baseRadius
        FOVCircle.Color = fovColor
        FOVCircle.Visible = Toggles.AimbotEnabled.Value and Toggles.FovVisible.Value
    end

    -- 2. DYNAMIC FOV FILL (The Spin Animation)
    if Toggles.AimbotEnabled.Value and Toggles.FOVAnim.Value then
        FOVFrame.Visible = true
        FOVFrame.Position = UDim2.fromOffset(mousePos.X, mousePos.Y)
        local d = baseRadius * 2
        FOVFrame.Size = UDim2.fromOffset(d, d)
        local c1 = fovColor
        local c2 = Color3.new(math.max(0, c1.R - 0.4), math.max(0, c1.G - 0.4), math.max(0, c1.B - 0.4))
        FOVGradient.Color = ColorSequence.new{
            ColorSequenceKeypoint.new(0.00, c1),
            ColorSequenceKeypoint.new(1.00, c2)
        }
        local currentTick = tick()
        local Speed = 300 
        FOV_RotationAngle = FOV_RotationAngle + (currentTick - FOV_Tick) * Speed * math.cos(math.pi / 4 * currentTick - math.pi / 2)
        FOVGradient.Rotation = FOV_RotationAngle
        FOV_Tick = currentTick
    else
        FOVFrame.Visible = false
    end
    
    -- [[ FOV TRACER UPDATE ]] --
    local closestTarget = getClosestTarget()
    
    if Toggles.FOVTracer.Value and closestTarget and Toggles.AimbotEnabled.Value then
        local aimPart = getAimPart(closestTarget)
        if aimPart then
            local screenPos, onScreen = Camera:WorldToViewportPoint(aimPart.Position)
            if onScreen then
                FOVTracerLine.From = mousePos
                FOVTracerLine.To = Vector2.new(screenPos.X, screenPos.Y)
                FOVTracerLine.Color = Options.FOVTracerColor.Value
                FOVTracerLine.Visible = true
            else
                FOVTracerLine.Visible = false
            end
        else
            FOVTracerLine.Visible = false
        end
    else
        FOVTracerLine.Visible = false
    end

    -- [[ TP TO MOUSE KEY ]] --
    if Options.TeleportMouseKey and Options.TeleportMouseKey:GetState() and mouse.Hit then
        if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
            LocalPlayer.Character.HumanoidRootPart.CFrame = mouse.Hit + Vector3.new(0,3,0)
        end
    end
    -- end lol
    -- [[ MOVEMENT LOGIC ]] --
    if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("Humanoid") and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
        local humanoid = LocalPlayer.Character.Humanoid
        local hrp = LocalPlayer.Character.HumanoidRootPart
        
        if Toggles.WalkSpeedEnabled.Value then
            humanoid.WalkSpeed = Options.WalkSpeedVal.Value
        end
        if Toggles.JumpPowerEnabled.Value then
            humanoid.UseJumpPower = true
            humanoid.JumpPower = Options.JumpPowerVal.Value
        end
        if Toggles.CFrameSpeedEnabled.Value then
            local moveDir = humanoid.MoveDirection
            if moveDir.Magnitude > 0 then
                hrp.CFrame = hrp.CFrame + (moveDir * Options.CFrameSpeedVal.Value)
            end
        end
        if Toggles.Noclip.Value then
            for _, v in pairs(LocalPlayer.Character:GetDescendants()) do
                if v:IsA("BasePart") and v.CanCollide then
                    v.CanCollide = false
                end
            end
        end
        if Toggles.Fly.Value and FlyBV and FlyBG then
            local cam = workspace.CurrentCamera
            local moveVec = Vector3.new(0,0,0)
            if FlyKeysPressed[Enum.KeyCode.W] then moveVec = moveVec + Vector3.new(0,0,-1) end
            if FlyKeysPressed[Enum.KeyCode.S] then moveVec = moveVec + Vector3.new(0,0,1) end
            if FlyKeysPressed[Enum.KeyCode.A] then moveVec = moveVec + Vector3.new(-1,0,0) end
            if FlyKeysPressed[Enum.KeyCode.D] then moveVec = moveVec + Vector3.new(1,0,0) end
            if FlyKeysPressed[Enum.KeyCode.Space] then moveVec = moveVec + Vector3.new(0,1,0) end
            if FlyKeysPressed[Enum.KeyCode.LeftControl] then moveVec = moveVec + Vector3.new(0,-1,0) end
            local currentFlySpeed = Options.FlySpeed.Value
            if moveVec.Magnitude > 0 then
                local camCF = CFrame.new(Vector3.new(), cam.CFrame.LookVector)
                local relative = (camCF:VectorToWorldSpace(moveVec)).Unit * currentFlySpeed
                FlyBV.Velocity = relative
            else
                FlyBV.Velocity = Vector3.new(0,0,0)
            end
            FlyBG.CFrame = CFrame.new(hrp.Position, hrp.Position + cam.CFrame.LookVector)
        end
    end

    local shootAttempt = false
    -- Manual Triggerbot
    if Toggles.TriggerbotEnabled.Value and Options.TriggerbotKeybind:GetState() then
        local targetPart = mouse.Target
        if targetPart then
             local char = targetPart.Parent; local humanoid = char:FindFirstChildOfClass("Humanoid")
             if not humanoid then char = targetPart.Parent.Parent; humanoid = char:FindFirstChildOfClass("Humanoid") end
             if humanoid and humanoid.Health > 0 and char.Name ~= LocalPlayer.Name then
                 local hrp = char:FindFirstChild("HumanoidRootPart")
                 if hrp then
                     local visible = not Toggles.TriggerbotWallCheck.Value or isVisible(targetPart)
                     if visible then Target = char; shootAttempt = true end
                 end
             end
        end
    end
    
    -- Main Targeting Variables
    local aimbotActive = false
    local currentTarget = nil

    -- RAGE MODE LOGIC
    if Toggles.RageMode.Value then
        -- Find target based on 360/Angle Logic
        local rageTarget = getRageTarget()
        
        if rageTarget then
            local partName = Options.RageTargetPart.Value
            if partName == "Random" then partName = "HumanoidRootPart" end
            
            -- Set the Hit Part for Silent Aim Hooks
            ScriptState.ClosestHitPart = rageTarget:FindFirstChild(partName) or rageTarget:FindFirstChild("Head")
            Target = rageTarget 
            
            -- Force Auto Shoot
            shootAttempt = true 
            
            -- Disable mouse dragging
            CurrentLockedTarget = rageTarget
            aimbotActive = false 
        else
            ScriptState.ClosestHitPart = nil
            Target = nil
        end
    
    -- LEGIT AIMBOT LOGIC
    elseif Toggles.AimbotEnabled.Value then
        
        local newClosest = getClosestTarget()
        
        if newClosest then
            local p = getAimPart(newClosest)
            ScriptState.ClosestHitPart = p
        else
            ScriptState.ClosestHitPart = nil
        end

        -- FOV Auto Shoot
        if Toggles.SilentAim.Value and Toggles.TriggerAutoShoot.Value and ScriptState.ClosestHitPart then
            shootAttempt = true
            Target = newClosest 
        end
        
        if Toggles.SilentAim.Value and Toggles.AimbotEnabled.Value then
            aimbotActive = false
        else
            aimbotActive = Options.AimbotKeybind:GetState()
        end

        if aimbotActive then
            if Toggles.StickyAim.Value then
                if CurrentLockedTarget and not isValidLockedTarget(CurrentLockedTarget) then CurrentLockedTarget = nil end
                if CurrentLockedTarget and Toggles.StickyWallCheck.Value and not isVisible(getAimPart(CurrentLockedTarget)) then CurrentLockedTarget = nil end
                if not CurrentLockedTarget then CurrentLockedTarget = newClosest end
                currentTarget = CurrentLockedTarget
            else
                currentTarget = newClosest
            end
        else
            currentTarget = nil
            JustLocked = true
        end

    else
        -- Disabled
        Target = nil
        ScriptState.ClosestHitPart = nil
        CurrentLockedTarget = nil
        JustLocked = true
    end

    -- Trigger the Click
    if shootAttempt then
        local now = tick(); local delay = Options.TriggerbotDelay.Value
        if (not Clicked) or (now >= TriggerNextShot) then 
            mouse1press()
            task.delay(0, mouse1release) 
            local effectiveDelay = (delay <= 0) and 0.01 or delay; TriggerNextShot = now + effectiveDelay; Clicked = true 
        end
    else
        Clicked = false
    end
    
    if not currentTarget then 
        if not shootAttempt then Target = nil end 
        PerceivedPos = nil; PerceivedVel = Vector3.new(); ClearGhost(); return 
    end
    
    local aimPart = getAimPart(currentTarget)
    if not aimPart then Target = nil; JustLocked = true; ClearGhost(); return end

    -- CALCULATE SMOOTH AIM MOVEMENT
    local currentVelocity = aimPart.AssemblyLinearVelocity or Vector3.new()
    local currentWorldPos = aimPart.Position
    local now = tick()
    
    if JustLocked then PerceivedPos = currentWorldPos; PerceivedVel = currentVelocity; LastUpdateTimeX = now; LastUpdateTimeY = now; JustLocked = false end
    
    if math.abs(currentVelocity.Y) < 1 and PerceivedVel.Y < -10 then
        PerceivedPos = Vector3.new(PerceivedPos.X, currentWorldPos.Y, PerceivedPos.Z); PerceivedVel = Vector3.new(PerceivedVel.X, currentVelocity.Y, PerceivedVel.Z); LastUpdateTimeY = now
    end
    if now - LastUpdateTimeX >= ReactionDelayX then
        PerceivedPos = Vector3.new(currentWorldPos.X, PerceivedPos.Y, currentWorldPos.Z); PerceivedVel = Vector3.new(currentVelocity.X, PerceivedVel.Y, currentVelocity.Z); LastUpdateTimeX = now
    end
    if now - LastUpdateTimeY >= ReactionDelayY then
        PerceivedPos = Vector3.new(PerceivedPos.X, currentWorldPos.Y, PerceivedPos.Z); PerceivedVel = Vector3.new(PerceivedVel.X, currentVelocity.Y, PerceivedVel.Z); LastUpdateTimeY = now
    end
    local perceivedX = PerceivedPos.X + PerceivedVel.X * (now - LastUpdateTimeX)
    local perceivedY = PerceivedPos.Y + PerceivedVel.Y * (now - LastUpdateTimeY)
    local perceivedZ = PerceivedPos.Z + PerceivedVel.Z * (now - LastUpdateTimeX)
    local perceivedWorldPos = Vector3.new(perceivedX, perceivedY, perceivedZ)
    local aimOffset = Vector3.new(Options.AimOffsetX and Options.AimOffsetX.Value or 0, Options.AimOffsetY and Options.AimOffsetY.Value or 0, Options.AimOffsetZ and Options.AimOffsetZ.Value or 0)
    local predictedWorldPos = perceivedWorldPos + PerceivedVel * Options.Prediction.Value + aimOffset
    
    if Toggles.PredictionForeshadow.Value and currentTarget then
        local hrp = currentTarget:FindFirstChild("HumanoidRootPart")
        if hrp then UpdateGhost(currentTarget, hrp.Position + (hrp.AssemblyLinearVelocity * Options.Prediction.Value) + aimOffset) end
    else ClearGhost() end
    
    local screenPos, onScreen = Camera:WorldToViewportPoint(predictedWorldPos)
    if not onScreen then Target = nil; JustLocked = true; ClearGhost(); return end
    
    Target = currentTarget
    
    if aimbotActive and not Toggles.SilentAim.Value then
        local smX = Options.XSmoothness.Value
        local smY = Options.YSmoothness.Value
        
        if Options.AimMethod.Value == "Mouse" then
            local deltaX = (screenPos.X - mousePos.X) / math.max(1, smX)
            local deltaY = (screenPos.Y - mousePos.Y) / math.max(1, smY)
            mousemoverel(deltaX, deltaY)
        elseif Options.AimMethod.Value == "Camera" then
            local camSmooth = (smX + smY) / 2
            Camera.CFrame = Camera.CFrame:Lerp(CFrame.lookAt(Camera.CFrame.Position, predictedWorldPos), 1 / math.max(1, camSmooth))
        end
    end
end)

local MenuGroup = Tabs["UI Settings"]:AddLeftGroupbox("Menu")
MenuGroup:AddToggle("KeybindMenuOpen", { Default = Library.KeybindFrame.Visible, Text = "Keybind Menu", Callback = function(Value) Library.KeybindFrame.Visible = Value end })
MenuGroup:AddLabel("Menu bind"):AddKeyPicker("MenuKeybind", { Default = "RightShift", NoUI = true, Text = "Menu keybind" })
MenuGroup:AddButton({ Text = "Unload", Func = function() 
    getgenv().ObsidianV1Running = false
    ClearGhost()
    -- Removed ClearAllTracers
    if FOVCircle then FOVCircle:Remove() end
    if FOVGui then FOVGui:Destroy() end
    if FOVTracerLine then FOVTracerLine:Remove() end
    Library:Unload() 
end })

Library.ToggleKeybind = Options.MenuKeybind
ThemeManager:SetLibrary(Library)
SaveManager:SetLibrary(Library)
SaveManager:IgnoreThemeSettings()
SaveManager:SetIgnoreIndexes({ "MenuKeybind", "AimbotKeybind", "TriggerbotKeybind" })
ThemeManager:ApplyToTab(Tabs["UI Settings"])
SaveManager:BuildConfigSection(Tabs["UI Settings"])
SaveManager:LoadAutoloadConfig()
Library:Notify("Obsidian GUI Loaded!", 5)
