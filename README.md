# Script-FTAP-BETA-

loadstring([[
-- FlokzyHub — Integrated functional loadstring (Orion UI)
-- All buttons and toggles are functional client-side. Server actions use RemoteEvent "SafeToyControlEvent" (server must validate).
-- Place as a LocalScript or run via executor. DOES NOT call SetNetworkOwner client-side.

-- Try load Orion UI
local ok, OrionLib = pcall(function()
    return loadstring(game:HttpGet("https://raw.githubusercontent.com/VerbalHubz/Verbal-Hub/main/Orion%20Hub%20Ui", true))()
end)
if not ok or type(OrionLib) ~= "table" then
    warn("OrionLib couldn't be loaded. Ensure HttpGet allowed and URL accessible.")
    return
end

-- Services
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Debris = game:GetService("Debris")
local StarterGui = game:GetService("StarterGui")

local localPlayer = Players.LocalPlayer
if not localPlayer then return end

-- Config (client-side admin list only for UI visibility; server must still validate)
local REMOTE_EVENT_NAME = "SafeToyControlEvent"
local CLIENT_ADMINS = {
    -- put your UserId here to enable Admin tab controls client-side
    [12345678] = true,
}

-- State
local state = {
    character = localPlayer.Character or localPlayer.CharacterAdded:Wait(),
    invConn = nil,
    antiGrabConn = nil,
    antiFlingConn = nil,
    flyConn = nil,
    flyBV = nil,
    autoHealConn = nil,
    serverToyName = "ServerToy",
    consoleLogs = {},
}

local function consoleLog(...)
    local s = table.concat({ ... }, " ")
    table.insert(state.consoleLogs, os.date("%X") .. " - " .. s)
    if #state.consoleLogs > 300 then table.remove(state.consoleLogs, 1) end
end

local function notify(title, text, dur)
    pcall(function()
        StarterGui:SetCore("SendNotification", { Title = title or "FlokzyHub", Text = text or "", Duration = dur or 3 })
    end)
end

local function refreshCharacter()
    state.character = localPlayer.Character or state.character
end
localPlayer.CharacterAdded:Connect(function(c) state.character = c end)

local function tryGetHumanoid()
    local c = state.character
    if not c then return nil end
    return c:FindFirstChildOfClass("Humanoid")
end

local function tryGetRoot()
    local c = state.character
    if not c then return nil end
    return c:FindFirstChild("HumanoidRootPart")
end

local function fireServer(action, params)
    local evt = ReplicatedStorage:FindFirstChild(REMOTE_EVENT_NAME)
    if not evt or not evt:IsA("RemoteEvent") then
        notify("Server", "RemoteEvent '"..REMOTE_EVENT_NAME.."' not found", 3)
        consoleLog("RemoteEvent missing:", REMOTE_EVENT_NAME)
        return
    end
    pcall(function() evt:FireServer(action, params) end)
    consoleLog("Fired server action:", action)
end

-- Create Window
local Window = OrionLib:MakeWindow({
    Name = "Fling Things and People Script V1",
    HidePremium = false,
    SaveConfig = true,
    IntroEnabled = true,
    IntroText = "FlokzyHub",
    IntroIcon = "rbxassetid://121559441445555",
    ConfigFolder = "FlokzyHub_Orion"
})

-- UTILITIES TAB
local TabUtilities = Window:MakeTab({
    Name = "Utilities",
    Icon = "rbxassetid://121559441445555",
    PremiumOnly = false
})
TabUtilities:AddLabel("Local-only utilities (safe)")

TabUtilities:AddButton({
    Name = "Pular Alto (local) - 6s",
    Callback = function()
        refreshCharacter()
        local h = tryGetHumanoid()
        if h then
            local prev = h.JumpPower
            h.JumpPower = 150
            h:ChangeState(Enum.HumanoidStateType.Jumping)
            notify("Utilities", "JumpBoost enabled (6s)", 2)
            consoleLog("JumpBoost applied")
            delay(6, function() if h and h.Parent then h.JumpPower = prev end end)
        else notify("Utilities", "Humanoid not ready", 2) end
    end
})

TabUtilities:AddButton({
    Name = "Velocidade x2 (local) - 8s",
    Callback = function()
        refreshCharacter()
        local h = tryGetHumanoid()
        if h then
            local prev = h.WalkSpeed
            h.WalkSpeed = math.max(16, prev * 2)
            notify("Utilities", "Speed x2 enabled (8s)", 2)
            consoleLog("Speed x2 applied")
            delay(8, function() if h and h.Parent then h.WalkSpeed = prev end end)
        else notify("Utilities", "Humanoid not ready", 2) end
    end
})

TabUtilities:AddToggle({
    Name = "Auto Heal (Local)",
    Default = false,
    Callback = function(on)
        if on then
            if state.autoHealConn then state.autoHealConn:Disconnect() end
            state.autoHealConn = RunService.Heartbeat:Connect(function()
                local h = tryGetHumanoid()
                if h then
                    local maxH = (h.MaxHealth and h.MaxHealth > 0) and h.MaxHealth or 100
                    pcall(function() h.Health = maxH end)
                end
            end)
            notify("Utilities", "Auto Heal ON", 2)
            consoleLog("AutoHeal ON")
        else
            if state.autoHealConn then state.autoHealConn:Disconnect() state.autoHealConn = nil end
            notify("Utilities", "Auto Heal OFF", 2)
            consoleLog("AutoHeal OFF")
        end
    end
})

-- FTAP TAB
local TabFTAP = Window:MakeTab({
    Name = "FTAP",
    Icon = "rbxassetid://121559441445555",
    PremiumOnly = false
})
TabFTAP:AddLabel("Proteções cliente e controles de toys")

-- Invincibility
TabFTAP:AddToggle({
    Name = "Invincibility (Local)",
    Default = false,
    Callback = function(on)
        if on then
            if state.invConn then state.invConn:Disconnect() end
            state.invConn = RunService.Heartbeat:Connect(function()
                local h = tryGetHumanoid()
                if h then
                    local maxH = (h.MaxHealth and h.MaxHealth > 0) and h.MaxHealth or 100
                    pcall(function() h.Health = maxH end)
                end
            end)
            notify("FTAP", "Invincibility ON", 2)
            consoleLog("Invincibility ON")
        else
            if state.invConn then state.invConn:Disconnect() state.invConn = nil end
            notify("FTAP", "Invincibility OFF", 2)
            consoleLog("Invincibility OFF")
        end
    end
})

-- Anti-Grab
local antiGrabForbidden = {
    BodyVelocity=true, LinearVelocity=true, VectorForce=true, BodyForce=true,
    BodyGyro=true, AlignPosition=true, AlignOrientation=true, WeldConstraint=true,
    RopeConstraint=true, RodConstraint=true, Attach=true
}
local function neutralizeInstance(inst)
    pcall(function()
        if not inst or not inst.Parent then return end
        if inst:IsA("BodyVelocity") or inst:IsA("LinearVelocity") then
            inst.Velocity = Vector3.new(0,0,0)
            if inst.MaxForce then inst.MaxForce = Vector3.new(0,0,0) end
        elseif inst:IsA("BodyForce") then
            inst.Force = Vector3.new(0,0,0)
        elseif inst:IsA("VectorForce") and inst:FindFirstChild("Force") then
            inst.Force = Vector3.new(0,0,0)
        elseif inst:IsA("BodyGyro") then
            if inst.MaxTorque then inst.MaxTorque = Vector3.new(0,0,0) end
        elseif inst:IsA("AlignPosition") or inst:IsA("AlignOrientation") then
            if inst.Enabled ~= nil then inst.Enabled = false end
            if inst.Responsiveness ~= nil then inst.Responsiveness = 0 end
        elseif inst:IsA("WeldConstraint") or inst:IsA("RodConstraint") or inst:IsA("RopeConstraint") then
            if inst.Enabled ~= nil then inst.Enabled = false end
        end
    end)
end

TabFTAP:AddToggle({
    Name = "Anti-Grab (Local)",
    Default = false,
    Callback = function(on)
        if on then
            if state.antiGrabConn then state.antiGrabConn:Disconnect() end
            state.antiGrabConn = RunService.Heartbeat:Connect(function()
                local char = state.character
                if not char then return end
                for _, inst in ipairs(char:GetDescendants()) do
                    if antiGrabForbidden[inst.ClassName] and not inst:IsA("Motor6D") then
                        neutralizeInstance(inst)
                    end
                end
            end)
            notify("FTAP", "Anti-Grab ON", 2)
            consoleLog("AntiGrab ON")
        else
            if state.antiGrabConn then state.antiGrabConn:Disconnect() state.antiGrabConn = nil end
            notify("FTAP", "Anti-Grab OFF", 2)
            consoleLog("AntiGrab OFF")
        end
    end
})

-- Anti-Fling
local function neutralizeFling(root)
    pcall(function()
        if root:FindFirstChild("Flokzy_AntiFlingBV") then root.Flokzy_AntiFlingBV:Destroy() end
        local bv = Instance.new("BodyVelocity")
        bv.Name = "Flokzy_AntiFlingBV"
        bv.MaxForce = Vector3.new(1e5,1e5,1e5)
        local vel = root.AssemblyLinearVelocity or root.Velocity or Vector3.new(0,0,0)
        local counter = -vel * 1.05
        local maxClamp = 250
        if counter.Magnitude > maxClamp then counter = counter.Unit * maxClamp end
        bv.Velocity = counter
        bv.Parent = root
        Debris:AddItem(bv, 0.16)
        local gyro = Instance.new("BodyGyro")
        gyro.Name = "Flokzy_AntiFlingGyro"
        gyro.MaxTorque = Vector3.new(1e5,1e5,1e5)
        gyro.P = 3000
        gyro.CFrame = root.CFrame
        gyro.Parent = root
        Debris:AddItem(gyro, 0.16)
    end)
end

TabFTAP:AddToggle({
    Name = "Anti-Fling (Local)",
    Default = false,
    Callback = function(on)
        if on then
            if state.antiFlingConn then state.antiFlingConn:Disconnect() end
            state.antiFlingConn = RunService.Heartbeat:Connect(function()
                local root = tryGetRoot()
                if not root then return end
                local vel = root.AssemblyLinearVelocity or root.Velocity or Vector3.new(0,0,0)
                if vel.Magnitude >= 120 then
                    neutralizeFling(root)
                    consoleLog("Anti-Fling engaged (speed="..math.floor(vel.Magnitude)..")")
                end
            end)
            notify("FTAP", "Anti-Fling ON", 2)
            consoleLog("AntiFling ON")
        else
            if state.antiFlingConn then state.antiFlingConn:Disconnect() state.antiFlingConn = nil end
            notify("FTAP", "Anti-Fling OFF", 2)
            consoleLog("AntiFling OFF")
        end
    end
})

-- Fly
TabFTAP:AddToggle({
    Name = "Fly (Local)",
    Default = false,
    Callback = function(on)
        if on then
            local root = tryGetRoot()
            if not root then notify("FTAP", "HumanoidRootPart not found", 2); return end
            if state.flyBV and state.flyBV.Parent then state.flyBV:Destroy() end
            local bv = Instance.new("BodyVelocity")
            bv.Name = "Flokzy_FlyBV"
            bv.MaxForce = Vector3.new(1e5,1e5,1e5)
            bv.Velocity = Vector3.new(0,0,0)
            bv.Parent = root
            state.flyBV = bv
            if state.flyConn then state.flyConn:Disconnect() end
            state.flyConn = RunService.RenderStepped:Connect(function()
                if not state.flyBV or not state.flyBV.Parent then return end
                local cam = workspace.CurrentCamera
                local mv = Vector3.new()
                if UserInputService:IsKeyDown(Enum.KeyCode.W) then mv = mv + cam.CFrame.LookVector end
                if UserInputService:IsKeyDown(Enum.KeyCode.S) then mv = mv - cam.CFrame.LookVector end
                if UserInputService:IsKeyDown(Enum.KeyCode.A) then mv = mv - cam.CFrame.RightVector end
                if UserInputService:IsKeyDown(Enum.KeyCode.D) then mv = mv + cam.CFrame.RightVector end
                if UserInputService:IsKeyDown(Enum.KeyCode.Space) then mv = mv + Vector3.new(0,1,0) end
                if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then mv = mv - Vector3.new(0,1,0) end
                if mv.Magnitude > 0 then state.flyBV.Velocity = mv.Unit * 90 else state.flyBV.Velocity = Vector3.new(0,0,0) end
            end)
            notify("FTAP", "Fly enabled", 2)
            consoleLog("Fly started")
        else
            if state.flyConn then state.flyConn:Disconnect() state.flyConn = nil end
            if state.flyBV then state.flyBV:Destroy() state.flyBV = nil end
            notify("FTAP", "Fly disabled", 2)
            consoleLog("Fly stopped")
        end
    end
})

TabFTAP:AddButton({
    Name = "Spawn Bola (Local)",
    Callback = function()
        local root = tryGetRoot()
        if not root then notify("FTAP", "HumanoidRootPart not found", 2); return end
        local part = Instance.new("Part")
        part.Name = "LocalToy_" .. tostring(math.random(1000,9999))
        part.Shape = Enum.PartType.Ball
        part.Size = Vector3.new(2,2,2)
        part.Position = root.Position + root.CFrame.LookVector * 4 + Vector3.new(0,3,0)
        part.Anchored = false
        part.CanCollide = true
        part.Parent = workspace
        local bv = Instance.new("BodyVelocity")
        bv.Velocity = root.CFrame.LookVector * 80 + Vector3.new(0,30,0)
        bv.MaxForce = Vector3.new(1e5,1e5,1e5)
        bv.P = 1250
        bv.Parent = part
        Debris:AddItem(bv, 0.5)
        Debris:AddItem(part, 10)
        notify("FTAP", "Local ball spawned", 2)
        consoleLog("Local toy spawned: " .. part.Name)
    end
})

-- MODULES TAB
local TabModules = Window:MakeTab({ Name = "Modules", Icon = "rbxassetid://121559441445555", PremiumOnly = false })
TabModules:AddLabel("Modules (extensible) - use _G.FlokzyHub.addModule")

TabModules:AddButton({
    Name = "Example Module: Local Logger",
    Callback = function()
        local t = Window:MakeTab({ Name = "Logger", Icon = "rbxassetid://121559441445555", PremiumOnly = false })
        t:AddParagraph("Local Logger", "Example module - writes a log entry.")
        t:AddButton({ Name = "Write log", Callback = function() consoleLog("Example log at "..os.time()); notify("Module","Logged",1) end })
        consoleLog("Example module opened")
    end
})

-- expose module API
_G = _G or {}
_G.FlokzyHub = _G.FlokzyHub or {}
_G.FlokzyHub.addModule = function(name, setupFn)
    pcall(function()
        local t = Window:MakeTab({ Name = name, Icon = "rbxassetid://121559441445555", PremiumOnly = false })
        setupFn(t)
        consoleLog("Module added: " .. tostring(name))
    end)
end

-- SETTINGS TAB
local TabSettings = Window:MakeTab({ Name = "Settings", Icon = "rbxassetid://121559441445555", PremiumOnly = false })
TabSettings:AddButton({ Name = "Re-check RemoteEvent", Callback = function() local found = ReplicatedStorage:FindFirstChild(REMOTE_EVENT_NAME) ~= nil; notify("Settings", "RemoteEvent: "..(found and "found" or "not found"), 3); consoleLog("RemoteEvent rechecked:", found) end })
TabSettings:AddDropdown({ Name = "UI Toggle Key (Orion default RightShift)", Default = "RightShift", Options = {"RightShift","LeftShift","F1","F2"}, Callback = function(opt) notify("Settings", "UI key suggestion: "..tostring(opt), 2); consoleLog("UI key selected:", opt) end })
TabSettings:AddButton({ Name = "Restaurar Valores Padrão (local)", Callback = function()
    local h = tryGetHumanoid()
    if h then h.JumpPower = 50; h.WalkSpeed = 16 end
    -- disconnect local conns
    pcall(function() if state.invConn then state.invConn:Disconnect(); state.invConn=nil end end)
    pcall(function() if state.antiGrabConn then state.antiGrabConn:Disconnect(); state.antiGrabConn=nil end end)
    pcall(function() if state.antiFlingConn then state.antiFlingConn:Disconnect(); state.antiFlingConn=nil end end)
    pcall(function() if state.flyConn then state.flyConn:Disconnect(); state.flyConn=nil end end)
    pcall(function() if state.flyBV then state.flyBV:Destroy(); state.flyBV=nil end end)
    if state.autoHealConn then state.autoHealConn:Disconnect(); state.autoHealConn=nil end
    notify("Settings", "Defaults restored (local)", 2)
    consoleLog("Defaults restored")
end })

-- CONSOLE TAB
local TabConsole = Window:MakeTab({ Name = "Console", Icon = "rbxassetid://121559441445555", PremiumOnly = false })
TabConsole:AddLabel("Local console - recent logs")
TabConsole:AddButton({ Name = "Refresh Console", Callback = function() TabConsole:AddParagraph("Logs", table.concat(state.consoleLogs, "\n") ~= "" and table.concat(state.consoleLogs, "\n") or "No logs yet") end })
TabConsole:AddButton({ Name = "Clear Console", Callback = function() state.consoleLogs = {}; notify("Console", "Cleared", 1); consoleLog("Console cleared") end })

-- EXTRAS TAB
local TabExtras = Window:MakeTab({ Name = "Extras", Icon = "rbxassetid://121559441445555", PremiumOnly = false })
TabExtras:AddButton({ Name = "Toggle UI (RightShift/X)", Callback = function() Window:Toggle() end })
TabExtras:AddButton({ Name = "Show Character Info", Callback = function() local c = state.character; if not c then notify("Extras","Character not ready",2); return end; local pos = c:FindFirstChild("HumanoidRootPart") and tostring(c.HumanoidRootPart.Position) or "No root"; notify("Extras", "Char: "..tostring(c.Name).." Pos: "..pos, 4) end })

-- CREDITS TAB
local TabCredits = Window:MakeTab({ Name = "Credits", Icon = "rbxassetid://121559441445555", PremiumOnly = false })
TabCredits:AddParagraph("Credits", "FlokzyHub - safe client hub\nAuthor: You\nUse only in games you own/administer.")

-- ADMIN TAB (client-side visibility only)
local TabAdmin = Window:MakeTab({ Name = "Admin", Icon = "rbxassetid://121559441445555", PremiumOnly = false })
TabAdmin:AddLabel("Admin (client-side UI only; server must validate)")
TabAdmin:AddTextbox({ Name = "Toy Name (server)", Default = state.serverToyName, TextDisappear = false, Callback = function(txt) state.serverToyName = txt end })
TabAdmin:AddButton({ Name = "Spawn Toy (server)", Callback = function()
    if not CLIENT_ADMINS[localPlayer.UserId] then notify("Admin","Not authorized (client-side)",2); return end
    fireServer("SpawnToy", { name = state.serverToyName or "ServerToy", forwardOffset = 6, power = 60 })
end })
TabAdmin:AddButton({ Name = "Set NetOwner to Me (server)", Callback = function()
    if not CLIENT_ADMINS[localPlayer.UserId] then notify("Admin","Not authorized (client-side)",2); return end
    fireServer("SetNetworkOwner", { targetName = state.serverToyName or "ServerToy", assignToPlayer = true })
end })
TabAdmin:AddButton({ Name = "Clear NetOwner (server)", Callback = function()
    if not CLIENT_ADMINS[localPlayer.UserId] then notify("Admin","Not authorized (client-side)",2); return end
    fireServer("SetNetworkOwner", { targetName = state.serverToyName or "ServerToy", assignToPlayer = false })
end })
TabAdmin:AddButton({ Name = "Destroy Toy (server)", Callback = function()
    if not CLIENT_ADMINS[localPlayer.UserId] then notify("Admin","Not authorized (client-side)",2); return end
    fireServer("DestroyToy", { targetName = state.serverToyName or "ServerToy" })
end })

-- Extra keybind: X toggles UI (also Orion's RightShift remains)
UserInputService.InputBegan:Connect(function(input, typing)
    if typing then return end
    if input.KeyCode == Enum.KeyCode.X then
        Window:Toggle()
    end
end)

-- Cleanup on respawn
localPlayer.CharacterAdded:Connect(function(char)
    pcall(function()
        if state.invConn then state.invConn:Disconnect(); state.invConn = nil end
        if state.antiGrabConn then state.antiGrabConn:Disconnect(); state.antiGrabConn = nil end
        if state.antiFlingConn then state.antiFlingConn:Disconnect(); state.antiFlingConn = nil end
        if state.flyConn then state.flyConn:Disconnect(); state.flyConn = nil end
        if state.flyBV then state.flyBV:Destroy(); state.flyBV = nil end
        if state.autoHealConn then state.autoHealConn:Disconnect(); state.autoHealConn = nil end
    end)
    state.character = char
    consoleLog("Character respawned; cleaned local connections.")
end)

consoleLog("FlokzyHub integrated functional loadstring loaded.")
notify("FlokzyHub", "Loaded — toggle with RightShift or X. Use responsibly.", 4)
]])()
