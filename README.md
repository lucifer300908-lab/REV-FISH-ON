# REV-FISH-ON

-- REV-HUB V3 Enterprise (Gabungan dengan Fitur Fishing)
-- Profesional, terintegrasi, dan siap pakai

local debugX = true

--// LOAD FRAMEWORK
local Rayfield = loadstring(game:HttpGet("https://sirius.menu/rayfield"))()

--// CORE OBJECT
local HUB = {
    Version = "3.0",
    Modules = {},
    Permissions = {},
    Events = {},
    Logs = {}
}

--// SAFE LOGGER
function HUB:Log(message)
    table.insert(self.Logs, "[REV] "..message)
    if debugX then
        print("[REV-HUB]", message)
    end
end

--// EVENT BUS
function HUB:On(eventName, callback)
    self.Events[eventName] = self.Events[eventName] or {}
    table.insert(self.Events[eventName], callback)
end

function HUB:Emit(eventName, ...)
    if self.Events[eventName] then
        for _,callback in ipairs(self.Events[eventName]) do
            callback(...)
        end
    end
end

--// PERMISSION SYSTEM
function HUB:SetPermission(userId, level)
    self.Permissions[userId] = level
end

function HUB:CheckPermission(userId, required)
    return (self.Permissions[userId] or 0) >= required
end

--// MODULE REGISTRY
function HUB:RegisterModule(name, permissionLevel, callback)
    self.Modules[name] = {
        Level = permissionLevel,
        Execute = function(...)
            local player = game.Players.LocalPlayer
            if HUB:CheckPermission(player.UserId, permissionLevel) then
                HUB:Log("Executing module: "..name)
                local success,err = pcall(callback,...)
                if not success then
                    warn("Module Error:", err)
                end
            else
                Rayfield:Notify({
                    Title = "Access Denied",
                    Content = "Insufficient permission level.",
                    Duration = 3,
                    Image = "shield"
                })
            end
        end
    }
    HUB:Log("Module Registered: "..name)
end

--// PERFORMANCE MONITOR
local RunService = game:GetService("RunService")
local fps = 0
local last = tick()

RunService.RenderStepped:Connect(function()
    local now = tick()
    fps = math.floor(1 / (now - last))
    last = now
end)

--// WINDOW
local Window = Rayfield:CreateWindow({
   Name = "REV-HUB",
   Icon = "cpu",
   LoadingTitle = "REV-HUB v"..HUB.Version,
   LoadingSubtitle = "Enterprise Core System",
   Theme = "DarkBlue",
   DisableRayfieldPrompts = true,
   DisableBuildWarnings = true,
   ConfigurationSaving = {
      Enabled = true,
      FolderName = "REV-HUB",
      FileName = "REV_Config"
   }
})

--// TABS
local CoreTab = Window:CreateTab("Core", "cpu")
local ModuleTab = Window:CreateTab("Modules", "boxes")
local SystemTab = Window:CreateTab("System", "settings")

CoreTab:CreateSection("System Boot")

--// BOOT SEQUENCE
task.spawn(function()
    local bootSteps = {
        "Initializing Kernel...",
        "Validating Modules...",
        "Binding Event Bus...",
        "Starting Runtime Monitor...",
        "REV-HUB Online."
    }

    for _,msg in ipairs(bootSteps) do
        HUB:Log(msg)
        Rayfield:Notify({
            Title = "REV Boot",
            Content = msg,
            Duration = 1.2,
            Image = "terminal"
        })
        task.wait(0.8)
    end

    HUB:Emit("BootComplete")
end)

--// SAMPLE PERMISSION
HUB:SetPermission(game.Players.LocalPlayer.UserId, 2)

--// SAMPLE MODULE
HUB:RegisterModule("Diagnostics", 1, function()
    Rayfield:Notify({
        Title = "Diagnostics",
        Content = "FPS: "..fps.." | Modules: "..tostring((function()
            local c=0 for _ in pairs(HUB.Modules) do c=c+1 end return c end)()),
        Duration = 4,
        Image = "activity"
    })
end)

ModuleTab:CreateButton({
   Name = "Run Diagnostics",
   Callback = function()
       HUB.Modules["Diagnostics"].Execute()
   end,
})

--// SYSTEM PANEL
SystemTab:CreateToggle({
    Name = "Debug Mode",
    CurrentValue = true,
    Flag = "DebugMode",
    Callback = function(val)
        debugX = val
        HUB:Log("Debug Mode: "..tostring(val))
    end,
})

SystemTab:CreateButton({
    Name = "Show Logs",
    Callback = function()
        for _,log in ipairs(HUB.Logs) do
            print(log)
        end
    end,
})

SystemTab:CreateButton({
    Name = "System Info",
    Callback = function()
        Rayfield:Notify({
            Title = "REV-HUB System",
            Content = "Version "..HUB.Version.." | FPS "..fps,
            Duration = 4,
            Image = "info"
        })
    end,
})

-- ==================== INTEGRASI FITUR FISHING ====================

-- Services dan Remote (dari script kedua)
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local VirtualUser = game:GetService("VirtualUser")
local UserInputService = game:GetService("UserInputService")
local Camera = workspace.CurrentCamera

-- Remote yang digunakan (pastikan sesuai dengan game)
local FishingSync = ReplicatedStorage:FindFirstChild("RemoteEvents") and ReplicatedStorage.RemoteEvents:FindFirstChild("FishingSync")
local ChooseFishRF = ReplicatedStorage:FindFirstChild("RemoteFunctions") and ReplicatedStorage.RemoteFunctions:FindFirstChild("ChooseFish")
local GiveFishRemote = ReplicatedStorage:FindFirstChild("RemoteEvents") and ReplicatedStorage.RemoteEvents:FindFirstChild("GiveFish")
local SellRemote = ReplicatedStorage:FindFirstChild("RemoteEvents") and ReplicatedStorage.RemoteEvents:FindFirstChild("FishSellAll")

-- Variabel global untuk fitur fishing
getgenv().FarmActive = false
getgenv().FishDelay = 0.5
getgenv().LuckActive = false
getgenv().CustomLuck = 25500
getgenv().WeightActive = false
getgenv().CustomWeight = 50
getgenv().SavedPosition = nil
getgenv().WSActive = false
getgenv().WSValue = 16
getgenv().JPActive = false
getgenv().JPValue = 50
getgenv().AntiAFK = false
getgenv().TargetMapPos = nil
getgenv().TargetPlayer = nil
local IsSpectatingTarget = false
local StatusLbl = nil  -- akan diisi nanti

-- Fungsi untuk update list (Map/Player) - sederhana, bisa dikembangkan
local function UpdateList(type, btn)
    if type == "Map" then
        -- Contoh: pilih island pertama (bisa diganti dengan dropdown)
        local maps = {Vector3.new(119.98, 67.28, -926.37), Vector3.new(112.65, 51.34, -846.82)} -- contoh
        getgenv().TargetMapPos = maps[1]
        btn.Text = "Selected"
        task.wait(0.5)
        btn.Text = "Click"
    elseif type == "Player" then
        -- Pilih player pertama selain diri sendiri
        for _,p in ipairs(Players:GetPlayers()) do
            if p ~= LocalPlayer then
                getgenv().TargetPlayer = p
                break
            end
        end
        btn.Text = "Selected"
        task.wait(0.5)
        btn.Text = "Click"
    end
end

-- Fungsi farming
function StartFarming()
    task.spawn(function()
        while getgenv().FarmActive do
            local Char = LocalPlayer.Character
            if not Char then task.wait(1) continue end
            local Rod = LocalPlayer.Backpack:FindFirstChildWhichIsA("Tool") or Char:FindFirstChildWhichIsA("Tool")
            if Rod and Rod.Parent ~= Char then Char.Humanoid:EquipTool(Rod) end

            local Root = Char:FindFirstChild("HumanoidRootPart")
            if not Root then task.wait(1) continue end

            local CastPos = Root.Position + (Root.CFrame.LookVector * 10) + Vector3.new(0, -2, 0)
            if StatusLbl then StatusLbl.Text = "Casting.." end
            if FishingSync then FishingSync:FireServer("StartFishing", CastPos, 3) end
            task.wait(0.3)

            local LuckVal = getgenv().LuckActive and getgenv().CustomLuck or 25500
            local Success, Result = pcall(function()
                return ChooseFishRF and ChooseFishRF:InvokeServer({["baseLuck"] = LuckVal, ["targetPos"] = CastPos})
            end)

            local RealToken = nil
            if Success and Result then
                if type(Result) == "string" then RealToken = Result
                elseif type(Result) == "table" then
                    for _, v in pairs(Result) do
                        if type(v) == "string" and #v > 20 then RealToken = v break end
                    end
                end
            end

            if RealToken then
                if StatusLbl then StatusLbl.Text = "Catching.." end
                local FinalWeight = getgenv().WeightActive and getgenv().CustomWeight or math.random(50, 500)
                if GiveFishRemote then
                    GiveFishRemote:FireServer(RealToken, FinalWeight, FinalWeight, "Ocean")
                end
            end

            task.wait(0.2)
            if FishingSync then FishingSync:FireServer("EndFishing") end
            task.wait(getgenv().FishDelay)
        end
        if StatusLbl then StatusLbl.Text = "Idle" end
    end)
end

-- Buat Tab-Tab dari script kedua menggunakan Rayfield

-- Tab Info
local InfoTab = Window:CreateTab("Info", "❗")
InfoTab:CreateSection("ANNOUNCEMENT:")
InfoTab:CreateLabel("Jangan gunakan Auto Farm saat God Luck aktif! Resiko banned tinggi.")

-- Tab Fishing
local FishTab = Window:CreateTab("Fishing", "🎣")
FishTab:CreateSection("--- MAIN FEATURES ---")
FishTab:CreateToggle({
    Name = "Auto Farm",
    CurrentValue = false,
    Flag = "AutoFarm",
    Callback = function(val)
        getgenv().FarmActive = val
        if val then StartFarming() end
    end
})
FishTab:CreateInput({
    Name = "Delay Speed",
    Placeholder = "0.5",
    Flag = "FishDelay",
    Callback = function(val)
        local n = tonumber(val)
        if n then getgenv().FishDelay = n end
    end
})
FishTab:CreateSection("--- LUCK MODIFIER ---")
FishTab:CreateToggle({
    Name = "God Luck",
    CurrentValue = false,
    Flag = "GodLuck",
    Callback = function(val) getgenv().LuckActive = val end
})
FishTab:CreateInput({
    Name = "Luck Amount",
    Placeholder = "25500",
    Flag = "LuckAmount",
    Callback = function(val)
        local n = tonumber(val)
        if n then getgenv().CustomLuck = n end
    end
})
FishTab:CreateSection("--- WEIGHT MODIFIER ---")
FishTab:CreateToggle({
    Name = "God Weight",
    CurrentValue = false,
    Flag = "GodWeight",
    Callback = function(val) getgenv().WeightActive = val end
})
FishTab:CreateInput({
    Name = "Weight (KG)",
    Placeholder = "50",
    Flag = "WeightKG",
    Callback = function(val)
        local n = tonumber(val)
        if n then getgenv().CustomWeight = n end
    end
})

-- Label status (karena Rayfield tidak punya dynamic label, kita buat dengan elemen sendiri? Bisa pakai CreateLabel tapi tidak update)
-- Alternatif: buat label biasa dan perbarui dengan thread terpisah. Namun untuk sederhana, kita gunakan label statis.
FishTab:CreateLabel("Status: Lihat Console")  -- sementara, karena tidak bisa update otomatis
-- Tapi kita tetap simpan StatusLbl untuk update nanti (tidak akan muncul di Rayfield, tapi kita bisa print di console)
StatusLbl = {Text = "Idle"}  -- dummy

-- Tab Automatically
local AutoTab = Window:CreateTab("Automatically", "⚙️")
AutoTab:CreateSection("--- POSITION MANAGER ---")
AutoTab:CreateButton({
    Name = "Save Position",
    Callback = function()
        if LocalPlayer.Character then
            getgenv().SavedPosition = LocalPlayer.Character.HumanoidRootPart.CFrame
            Rayfield:Notify({Title="Position", Content="Saved!", Duration=1})
        end
    end
})
AutoTab:CreateButton({
    Name = "Load Position",
    Callback = function()
        if getgenv().SavedPosition then
            LocalPlayer.Character.HumanoidRootPart.CFrame = getgenv().SavedPosition
            Rayfield:Notify({Title="Position", Content="Loaded!", Duration=1})
        else
            Rayfield:Notify({Title="Position", Content="No saved position!", Duration=1})
        end
    end
})
AutoTab:CreateButton({
    Name = "Reset Saved",
    Callback = function()
        getgenv().SavedPosition = nil
        Rayfield:Notify({Title="Position", Content="Reset!", Duration=1})
    end
})
AutoTab:CreateSection("--- MERCHANT ---")
AutoTab:CreateButton({
    Name = "Sell All Fish",
    Callback = function()
        if SellRemote then
            SellRemote:FireServer(true, "")
            Rayfield:Notify({Title="Sell", Content="Sold! ✅", Duration=1})
        else
            Rayfield:Notify({Title="Sell", Content="Remote not found!", Duration=1})
        end
    end
})

-- Tab Teleport
local TeleportTab = Window:CreateTab("Teleport", "🏝️")
TeleportTab:CreateSection("--- MAP TELEPORT ---")
TeleportTab:CreateButton({
    Name = "Map List",
    Callback = function()
        UpdateList("Map", {Text=""})  -- dummy btn
        Rayfield:Notify({Title="Map", Content="Selected first map", Duration=1})
    end
})
TeleportTab:CreateButton({
    Name = "Teleport to Map",
    Callback = function()
        if getgenv().TargetMapPos and LocalPlayer.Character then
            LocalPlayer.Character.HumanoidRootPart.CFrame = CFrame.new(getgenv().TargetMapPos + Vector3.new(0, 5, 0))
            Rayfield:Notify({Title="Teleport", Content="Done!", Duration=1})
        else
            Rayfield:Notify({Title="Teleport", Content="Select map first!", Duration=1})
        end
    end
})
TeleportTab:CreateSection("--- PLAYER TELEPORT ---")
TeleportTab:CreateButton({
    Name = "Player List",
    Callback = function()
        UpdateList("Player", {Text=""})
        Rayfield:Notify({Title="Player", Content="Selected first player", Duration=1})
    end
})
TeleportTab:CreateButton({
    Name = "Action Button",
    Callback = function()
        local Target = getgenv().TargetPlayer
        if not Target then
            Rayfield:Notify({Title="Teleport", Content="Select player first!", Duration=1})
            return
        end
        if not IsSpectatingTarget then
            Camera.CameraType = Enum.CameraType.Custom
            local CamSub = Target.Character and (Target.Character:FindFirstChild("Humanoid") or Target.Character:FindFirstChild("Head"))
            if CamSub then
                Camera.CameraSubject = CamSub
                IsSpectatingTarget = true
                Rayfield:Notify({Title="Spectating", Content="Now spectating "..Target.Name, Duration=2})
            else
                Rayfield:Notify({Title="Error", Content="Target far away", Duration=1})
            end
        else
            if LocalPlayer.Character and Target.Character then
                LocalPlayer.Character.HumanoidRootPart.CFrame = Target.Character.HumanoidRootPart.CFrame + Vector3.new(0, 3, 0)
                Rayfield:Notify({Title="Teleport", Content="Success!", Duration=1})
                -- Kembalikan kamera
                Camera.CameraType = Enum.CameraType.Custom
                Camera.CameraSubject = LocalPlayer.Character.Humanoid
                IsSpectatingTarget = false
            end
        end
    end
})

-- Tab Misc
local MiscTab = Window:CreateTab("Misc", "🧩")
MiscTab:CreateToggle({
    Name = "Enable WalkSpeed",
    CurrentValue = false,
    Flag = "WalkSpeedToggle",
    Callback = function(val) getgenv().WSActive = val end
})
MiscTab:CreateInput({
    Name = "Speed Amount",
    Placeholder = "16",
    Flag = "WalkSpeedValue",
    Callback = function(val)
        local n = tonumber(val)
        if n then getgenv().WSValue = n end
    end
})
MiscTab:CreateToggle({
    Name = "Enable JumpPower",
    CurrentValue = false,
    Flag = "JumpPowerToggle",
    Callback = function(val) getgenv().JPActive = val end
})
MiscTab:CreateInput({
    Name = "Jump Amount",
    Placeholder = "50",
    Flag = "JumpPowerValue",
    Callback = function(val)
        local n = tonumber(val)
        if n then getgenv().JPValue = n end
    end
})
MiscTab:CreateToggle({
    Name = "Anti AFK",
    CurrentValue = false,
    Flag = "AntiAFK",
    Callback = function(val) getgenv().AntiAFK = val end
})

-- Tab Settings
local SettingsTab = Window:CreateTab("Settings", "🔧")
SettingsTab:CreateButton({
    Name = "Unload GUI",
    Callback = function()
        getgenv().FarmActive = false
        Rayfield:Destroy()
        -- Hapus ScreenGui jika ada (dari framework lama, tidak perlu karena Rayfield sudah handle)
    end
})
SettingsTab:CreateButton({
    Name = "Anti Lag",
    Callback = function()
        for _,v in ipairs(workspace:GetDescendants()) do
            if v:IsA("BasePart") and not v.Parent:IsA("Model") then
                v.Material = Enum.Material.SmoothPlastic
            end
            if v:IsA("Texture") or v:IsA("Decal") then
                v:Destroy()
            end
        end
        Rayfield:Notify({Title="Anti Lag", Content="Done!", Duration=1})
    end
})

-- System Loops (Anti AFK dan WalkSpeed/JumpPower)
LocalPlayer.Idled:Connect(function()
    if getgenv().AntiAFK then
        VirtualUser:Button2Down(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
        task.wait(1)
        VirtualUser:Button2Up(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
    end
end)

RunService.Stepped:Connect(function()
    local char = LocalPlayer.Character
    if not char then return end
    local hum = char:FindFirstChild("Humanoid")
    if not hum then return end
    if getgenv().WSActive then
        hum.WalkSpeed = getgenv().WSValue
    end
    if getgenv().JPActive then
        hum.JumpPower = getgenv().JPValue
    end
end)

-- Load konfigurasi Rayfield
Rayfield:LoadConfiguration()

HUB:Log("All fishing features integrated successfully!")
