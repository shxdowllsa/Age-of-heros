-- =============================================================================
-- SCRIPT AGE OF HEROES: SHADOWLS PREMIUM V2 (UI EVOLUÍDA)
-- =============================================================================

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TeleportService = game:GetService("TeleportService")
local HttpService = game:GetService("HttpService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local VirtualUser = game:GetService("VirtualUser")

local player = Players.LocalPlayer
local punchEvent = ReplicatedStorage:WaitForChild("Events"):WaitForChild("Punch")
local Module = require(ReplicatedStorage:WaitForChild("Modules"):WaitForChild("SharedLocal"))

-- Configurações de Estado
local Config = {
    OrbFarm = false,
    HeroFarm = false,
    VillianFarm = false,
    FarmAll = false,
    HideTitle = false,
    Invisible = false,
    KillPlayer = false,
    SelectedPlayerTarget = nil,
    AntiAFK = true,
    ESP = false
}

-- =============================================================================
-- LOGICA DE BACKEND (CHEATS)
-- =============================================================================

-- Anti-AFK
player.Idled:Connect(function()
    if Config.AntiAFK then
        VirtualUser:CaptureController()
        VirtualUser:ClickButton2(Vector2.new(0, 0))
    end
end)

-- Energia Infinita
task.spawn(function()
    while task.wait(0.5) do
        pcall(function()
            if player:FindFirstChild("PlayerScripts") and player.PlayerScripts:FindFirstChild("GameClient") then
                getsenv(player.PlayerScripts.GameClient)._G.energy = math.huge
            end
        end)
    end
end)

-- Sistema de ESP
local function ApplyESP(p)
    local function createGui()
        if p == player then return end
        local char = p.Character or p.CharacterAdded:Wait()
        local head = char:WaitForChild("Head", 10)
        if not head then return end
        if head:FindFirstChild("ESP_Tag") then head.ESP_Tag:Destroy() end
        
        local bGui = Instance.new("BillboardGui", head)
        bGui.Name = "ESP_Tag"
        bGui.Size = UDim2.new(0, 100, 0, 50)
        bGui.StudsOffset = Vector3.new(0, 3, 0)
        bGui.AlwaysOnTop = true

        local label = Instance.new("TextLabel", bGui)
        label.BackgroundTransparency = 1
        label.Size = UDim2.new(1, 0, 1, 0)
        label.Text = p.DisplayName or p.Name
        label.Font = Enum.Font.GothamBold
        label.TextSize = 13
        label.TextStrokeTransparency = 0.5

        local color = Color3.fromRGB(255, 255, 255)
        if p.Team and (p.Team.Name:lower():find("hero") or p.Team.Name:lower():find("police")) then
            color = Color3.fromRGB(0, 170, 255)
        elseif p.Team and (p.Team.Name:lower():find("villain") or p.Team.Name:lower():find("criminal")) then
            color = Color3.fromRGB(255, 0, 0)
        end
        label.TextColor3 = color
        
        task.spawn(function()
            while bGui and bGui.Parent do
                bGui.Enabled = Config.ESP
                task.wait(0.5)
            end
        end)
    end
    p.CharacterAdded:Connect(createGui)
    if p.Character then createGui() end
end
for _, p in pairs(Players:GetPlayers()) do ApplyESP(p) end
Players.PlayerAdded:Connect(ApplyESP)

-- Funções de Combate
local function heavyPunch()
    punchEvent:FireServer(0.4, 0.1, 1)
end

local function goInvisible()
    if not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") then return end
    Config.Invisible = true
    local hrp = player.Character.HumanoidRootPart
    local ogCFrame = hrp.CFrame
    hrp.CFrame = CFrame.new(-2463.92, 256.45, -2009.25)
    task.wait(0.5)
    if player.Character:FindFirstChild("LowerTorso") and player.Character.LowerTorso:FindFirstChild("Root") then
        player.Character.LowerTorso.Root:Destroy()
    end
    task.wait(0.5)
    hrp.CFrame = ogCFrame
end

-- Farm Loops
task.spawn(function()
    while task.wait(0.1) do
        if Config.OrbFarm and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
            local orbs = workspace:FindFirstChild("ExperienceOrbs")
            if orbs then
                for _, orb in pairs(orbs:GetChildren()) do
                    if firetouchinterest then
                        firetouchinterest(player.Character.HumanoidRootPart, orb, 0)
                        firetouchinterest(player.Character.HumanoidRootPart, orb, 1)
                    end
                end
            end
        end
    end
end)

local function attackTarget(target, isPlayer)
    if target and target:FindFirstChild("HumanoidRootPart") and target:FindFirstChild("Humanoid") then
        repeat task.wait(0.05)
            if isPlayer and not Config.KillPlayer then break end
            if not isPlayer and (not Config.HeroFarm and not Config.VillianFarm and not Config.FarmAll) then break end
            pcall(function()
                player.Character.HumanoidRootPart.CFrame = target.HumanoidRootPart.CFrame * CFrame.new(0, 0, 0.7)
                heavyPunch()
            end)
        until not target or not target:FindFirstChild("Humanoid") or target.Humanoid.Health <= 0
        if isPlayer then Config.KillPlayer = false end
    end
end

task.spawn(function()
    while task.wait(0.1) do
        if Config.KillPlayer and Config.SelectedPlayerTarget then
            local t = Players:FindFirstChild(Config.SelectedPlayerTarget)
            if t and t.Character then attackTarget(t.Character, true) end
        elseif Config.HeroFarm then
            attackTarget(workspace:FindFirstChild("Thug"), false)
        elseif Config.VillianFarm then
            attackTarget(workspace:FindFirstChild("Civilian") or workspace:FindFirstChild("Police"), false)
        elseif Config.FarmAll then
            local pool = {workspace:FindFirstChild("Thug"), workspace:FindFirstChild("Civilian"), workspace:FindFirstChild("Police")}
            for _, v in pairs(pool) do if v then attackTarget(v, false) break end end
        end
    end
end)

-- =============================================================================
-- INTERFACE GRÁFICA (ESTILO PREMIUM DARK DO SCRIPT 2)
-- =============================================================================

local UI = Instance.new("ScreenGui")
UI.Name = "shadowls_FinalPremium"
UI.Parent = player:WaitForChild("PlayerGui")
UI.ResetOnSpawn = false

local function AddCorner(parent, radius)
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, radius)
    corner.Parent = parent
end

-- Botão Flutuante (🦉)
local MenuToggle = Instance.new("TextButton", UI)
MenuToggle.Size = UDim2.new(0, 40, 0, 40)
MenuToggle.Position = UDim2.new(0, 20, 0, 20)
MenuToggle.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
MenuToggle.Text = "🦉"
MenuToggle.TextSize = 18
MenuToggle.TextColor3 = Color3.fromRGB(255, 255, 255)
AddCorner(MenuToggle, 8)
local ToggleStroke = Instance.new("UIStroke", MenuToggle)
ToggleStroke.Color = Color3.fromRGB(40, 40, 50)

-- Frame Principal
local MainFrame = Instance.new("Frame", UI)
MainFrame.Size = UDim2.new(0, 550, 0, 320)
MainFrame.Position = UDim2.new(0.5, -275, 0.5, -160)
MainFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
MainFrame.Visible = true
AddCorner(MainFrame, 10)

-- Menu Lateral de Jogadores
local PListPanel = Instance.new("Frame", MainFrame)
PListPanel.Size = UDim2.new(0, 180, 1, 0)
PListPanel.Position = UDim2.new(1, 5, 0, 0)
PListPanel.BackgroundColor3 = Color3.fromRGB(18, 18, 18)
PListPanel.Visible = false
AddCorner(PListPanel, 10)

local PListTitle = Instance.new("TextLabel", PListPanel)
PListTitle.Size = UDim2.new(1, 0, 0, 35)
PListTitle.BackgroundTransparency = 1
PListTitle.Text = "Selecione o Alvo"
PListTitle.Font = Enum.Font.GothamBold
PListTitle.TextSize = 13
PListTitle.TextColor3 = Color3.fromRGB(200, 200, 200)

local PlayerListFrame = Instance.new("ScrollingFrame", PListPanel)
PlayerListFrame.Size = UDim2.new(1, -10, 1, -45)
PlayerListFrame.Position = UDim2.new(0, 5, 0, 35)
PlayerListFrame.BackgroundTransparency = 1
PlayerListFrame.ScrollBarThickness = 2
local PlayerListLayout = Instance.new("UIListLayout", PlayerListFrame)
PlayerListLayout.Padding = UDim.new(0, 4)

-- Sidebar
local Sidebar = Instance.new("Frame", MainFrame)
Sidebar.Size = UDim2.new(0, 160, 1, 0)
Sidebar.BackgroundColor3 = Color3.fromRGB(18, 18, 18)
AddCorner(Sidebar, 10)

local InfoLabel = Instance.new("TextLabel", Sidebar)
InfoLabel.Size = UDim2.new(1, -20, 0, 50)
InfoLabel.Position = UDim2.new(0, 15, 0, 10)
InfoLabel.BackgroundTransparency = 1
InfoLabel.Text = "shadowls\nAge of Heroes"
InfoLabel.Font = Enum.Font.GothamBold
InfoLabel.TextSize = 14
InfoLabel.TextColor3 = Color3.fromRGB(230, 230, 230)
InfoLabel.TextXAlignment = Enum.TextXAlignment.Left

local TabContainer = Instance.new("Frame", Sidebar)
TabContainer.Size = UDim2.new(1, -20, 1, -80)
TabContainer.Position = UDim2.new(0, 10, 0, 70)
TabContainer.BackgroundTransparency = 1
local TabListLayout = Instance.new("UIListLayout", TabContainer)
TabListLayout.Padding = UDim.new(0, 5)

local ContentContainer = Instance.new("Frame", MainFrame)
ContentContainer.Size = UDim2.new(1, -180, 1, -40)
ContentContainer.Position = UDim2.new(0, 170, 0, 30)
ContentContainer.BackgroundTransparency = 1

local Pages = {}
local TabButtons = {}

local function CreatePage(name)
    local Page = Instance.new("ScrollingFrame", ContentContainer)
    Page.Size = UDim2.new(1, 0, 1, 0)
    Page.BackgroundTransparency = 1
    Page.Visible = false
    Page.ScrollBarThickness = 0
    local PageLayout = Instance.new("UIListLayout", Page)
    PageLayout.Padding = UDim.new(0, 8)
    Pages[name] = Page
    
    local TabBtn = Instance.new("TextButton", TabContainer)
    TabBtn.Size = UDim2.new(1, 0, 0, 32)
    TabBtn.BackgroundColor3 = Color3.fromRGB(24, 24, 24)
    TabBtn.Text = "  " .. name
    TabBtn.Font = Enum.Font.GothamSemibold
    TabBtn.TextSize = 12
    TabBtn.TextColor3 = Color3.fromRGB(140, 140, 140)
    TabBtn.TextXAlignment = Enum.TextXAlignment.Left
    AddCorner(TabBtn, 6)
    TabButtons[name] = TabBtn
    
    TabBtn.MouseButton1Click:Connect(function()
        for _, p in pairs(Pages) do p.Visible = false end
        for _, b in pairs(TabButtons) do 
            b.BackgroundColor3 = Color3.fromRGB(24, 24, 24)
            b.TextColor3 = Color3.fromRGB(140, 140, 140)
        end
        Page.Visible = true
        TabBtn.BackgroundColor3 = Color3.fromRGB(28, 28, 28)
        TabBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    end)
end

-- Toggles Animados (Estilo Script 2)
local function NewToggle(parentPage, labelText, configKey)
    local Row = Instance.new("Frame", Pages[parentPage])
    Row.Size = UDim2.new(1, 0, 0, 40)
    Row.BackgroundColor3 = Color3.fromRGB(22, 22, 22)
    AddCorner(Row, 6)
    
    local Label = Instance.new("TextLabel", Row)
    Label.Size = UDim2.new(0.7, 0, 1, 0)
    Label.Position = UDim2.new(0, 12, 0, 0)
    Label.BackgroundTransparency = 1
    Label.Text = labelText
    Label.Font = Enum.Font.Gotham
    Label.TextSize = 13
    Label.TextColor3 = Color3.fromRGB(180, 180, 180)
    Label.TextXAlignment = Enum.TextXAlignment.Left
    
    local SwitchOuter = Instance.new("TextButton", Row)
    SwitchOuter.Size = UDim2.new(0, 36, 0, 20)
    SwitchOuter.Position = UDim2.new(1, -48, 0.5, -10)
    SwitchOuter.BackgroundColor3 = Config[configKey] and Color3.fromRGB(0, 140, 255) or Color3.fromRGB(45, 45, 45)
    SwitchOuter.Text = ""
    AddCorner(SwitchOuter, 10)
    
    local SwitchBall = Instance.new("Frame", SwitchOuter)
    SwitchBall.Size = UDim2.new(0, 14, 0, 14)
    SwitchBall.Position = Config[configKey] and UDim2.new(1, -17, 0.5, -7) or UDim2.new(0, 3, 0.5, -7)
    SwitchBall.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    AddCorner(SwitchBall, 10)
    
    SwitchOuter.MouseButton1Click:Connect(function()
        Config[configKey] = not Config[configKey]
        if Config[configKey] then
            TweenService:Create(SwitchOuter, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(0, 140, 255)}):Play()
            TweenService:Create(SwitchBall, TweenInfo.new(0.2), {Position = UDim2.new(1, -17, 0.5, -7)}):Play()
        else
            TweenService:Create(SwitchOuter, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(45, 45, 45)}):Play()
            TweenService:Create(SwitchBall, TweenInfo.new(0.2), {Position = UDim2.new(0, 3, 0.5, -7)}):Play()
        end
    end)
end

local function NewButton(parentPage, labelText, callback)
    local Btn = Instance.new("TextButton", Pages[parentPage])
    Btn.Size = UDim2.new(1, 0, 0, 38)
    Btn.BackgroundColor3 = Color3.fromRGB(24, 24, 24)
    Btn.Text = labelText
    Btn.Font = Enum.Font.GothamSemibold
    Btn.TextSize = 13
    Btn.TextColor3 = Color3.fromRGB(200, 200, 200)
    AddCorner(Btn, 6)
    Btn.MouseButton1Click:Connect(callback)
end

-- Configuração das Páginas e Itens
CreatePage("Farming")
CreatePage("Player")
CreatePage("Visual")
CreatePage("Utility")

NewToggle("Farming", "Auto Collect Orbs", "OrbFarm")
NewToggle("Farming", "Hero Farm (Thugs)", "HeroFarm")
NewToggle("Farming", "Villian Farm (Police)", "VillianFarm")
NewToggle("Farming", "Farm Everything", "FarmAll")

local SelectedLabel = Instance.new("TextLabel", Pages["Player"])
SelectedLabel.Size = UDim2.new(1, 0, 0, 25)
SelectedLabel.BackgroundTransparency = 1
SelectedLabel.Text = "Alvo: Nenhum"
SelectedLabel.TextColor3 = Color3.fromRGB(0, 140, 255)
SelectedLabel.Font = Enum.Font.GothamBold
SelectedLabel.TextSize = 12

NewButton("Player", "Selecionar Jogador", function() PListPanel.Visible = not PListPanel.Visible end)
NewToggle("Player", "Kill Selected Player", "KillPlayer")
NewButton("Player", "Go Invisible", goInvisible)

NewToggle("Visual", "ESP Players", "ESP")
NewToggle("Visual", "Hide Title/Name", "HideTitle")

NewToggle("Utility", "Anti-AFK System", "AntiAFK")
NewButton("Utility", "Server Hop", function() 
    local servers = HttpService:JSONDecode(game:HttpGet("https://games.roblox.com/v1/games/"..game.PlaceId.."/servers/Public?sortOrder=Asc&limit=100")).data
    for _, s in pairs(servers) do if s.playing < s.maxPlayers and s.id ~= game.JobId then TeleportService:TeleportToPlaceInstance(game.PlaceId, s.id) end end
end)
NewButton("Utility", "Rejoin Server", function() TeleportService:Teleport(game.PlaceId, player) end)

-- Sistema de Lista de Alvos
local function UpdatePlayerList()
    for _, v in pairs(PlayerListFrame:GetChildren()) do if v:IsA("TextButton") then v:Destroy() end end
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= player then
            local b = Instance.new("TextButton", PlayerListFrame)
            b.Size = UDim2.new(1, -5, 0, 30)
            b.Text = "  " .. p.DisplayName
            b.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
            b.TextColor3 = Color3.fromRGB(200, 200, 200)
            b.TextXAlignment = Enum.TextXAlignment.Left
            b.Font = Enum.Font.Gotham
            b.TextSize = 12
            AddCorner(b, 4)
            b.MouseButton1Click:Connect(function()
                Config.SelectedPlayerTarget = p.Name
                SelectedLabel.Text = "Alvo: " .. p.DisplayName
                PListPanel.Visible = false
            end)
        end
    end
end
Players.PlayerAdded:Connect(UpdatePlayerList)
Players.PlayerRemoving:Connect(UpdatePlayerList)
UpdatePlayerList()

-- Draggable (Arrastar)
local dragging, dragInput, dragStart, startPos
MainFrame.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then dragging = true dragStart = input.Position startPos = MainFrame.Position end
end)
UserInputService.InputChanged:Connect(function(input)
    if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
        local delta = input.Position - dragStart
        MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)
UserInputService.InputEnded:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseButton1 then dragging = false end end)

-- Toggle Menu
MenuToggle.MouseButton1Click:Connect(function()
    MainFrame.Visible = not MainFrame.Visible
    if not MainFrame.Visible then PListPanel.Visible = false end
end)

Pages["Farming"].Visible = true
TabButtons["Farming"].BackgroundColor3 = Color3.fromRGB(28, 28, 28)
TabButtons["Farming"].TextColor3 = Color3.fromRGB(255, 255, 255)

print("ShadowLS Premium V2: Carregado com Sucesso!")
