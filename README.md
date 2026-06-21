-- =============================================================================
-- BASE & SERVIÇOS DO ROBLOX
-- =============================================================================
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TeleportService = game:GetService("TeleportService")
local HttpService = game:GetService("HttpService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer
local punchEvent = ReplicatedStorage:WaitForChild("Events"):WaitForChild("Punch")
local Module = require(ReplicatedStorage:WaitForChild("Modules"):WaitForChild("SharedLocal"))

-- Opções de Configuração (Toggles)
local Config = {
    OrbFarm = false,
    HeroFarm = false,
    VillianFarm = false,
    FarmAll = false,
    HideTitle = false,
    Invisible = false,
    KillPlayer = false,
    SelectedPlayerTarget = nil
}

-- Aguarda o personagem carregar completamente
if not player.Character then
    repeat task.wait()
        pcall(function()
            if player:WaitForChild("PlayerScripts"):FindFirstChild("IntroScript") then
                getsenv(player.PlayerScripts.IntroScript).Play()
            end
        end)
    until player.Character and Module.IsValidActor(player.Character)
end

-- Loop de Energia Infinita
task.spawn(function()
    while task.wait(0.5) do
        pcall(function()
            if player:FindFirstChild("PlayerScripts") and player.PlayerScripts:FindFirstChild("GameClient") then
                getsenv(player.PlayerScripts.GameClient)._G.energy = math.huge
            end
        end)
    end
end)

-- =============================================================================
-- FUNÇÕES PRINCIPAIS
-- =============================================================================
local function heavyPunch()
    task.spawn(function() 
        punchEvent:FireServer(0.4, 0.1, 1) 
    end)
end

local function goInvisible()
    if not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") then return end
    Config.Invisible = true
    Config.HideTitle = true
    
    local hrp = player.Character.HumanoidRootPart
    local ogCFrame = hrp.CFrame
    
    hrp.CFrame = CFrame.new(-2463.92, 256.45, -2009.25)
    task.wait(0.5)
    
    if player.Character:FindFirstChild("LowerTorso") and player.Character.LowerTorso:FindFirstChild("Root") then
        local clone = player.Character.LowerTorso.Root:Clone()
        player.Character.LowerTorso.Root:Destroy()
        clone.Parent = player.Character.LowerTorso
    end
    
    task.wait(0.5)
    hrp.CFrame = ogCFrame
end

player.CharacterAdded:Connect(function(char)
    Config.Invisible = false
end)

local function ServerHop()
    local api = "https://games.roblox.com/v1/games/" .. game.PlaceId .. "/servers/Public?sortOrder=Asc&limit=100"
    local servers = {}
    local function fetch(url)
        local success, result = pcall(function() return HttpService:JSONDecode(game:HttpGet(url)) end)
        if success and result and result.data then
            for _, server in pairs(result.data) do
                if server.playing and tonumber(server.playing) < tonumber(server.maxPlayers) and server.id ~= game.JobId then
                    table.insert(servers, server.id)
                end
            end
            return result.nextPageCursor
        end
        return nil
    end
    local cursor = fetch(api)
    if cursor then fetch(api .. "&cursor=" .. cursor) end
    if #servers > 0 then
        TeleportService:TeleportToPlaceInstance(game.PlaceId, servers[math.random(1, #servers)], player)
    else
        TeleportService:Teleport(game.PlaceId, player)
    end
end

-- =============================================================================
-- LOGICA DOS CHEATS
-- =============================================================================
task.spawn(function()
    while task.wait(0.5) do
        if (Config.HideTitle or Config.Invisible) and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
            pcall(function()
                if player.Character.HumanoidRootPart:FindFirstChild("titleGui") then
                    player.Character.HumanoidRootPart.titleGui.Frame:Destroy()
                end
            end)
        end
    end
end)

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
                if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
                    player.Character.HumanoidRootPart.CFrame = target.HumanoidRootPart.CFrame * CFrame.new(0, 0, 0.7)
                    heavyPunch()
                end
            end)
        until not target or not target:FindFirstChild("Humanoid") or target.Humanoid.Health <= 0 or not Module.IsValidActor(target)
        
        if not isPlayer and target and target.Parent == workspace and target:FindFirstChild("Humanoid") and target.Humanoid.Health <= 0 then 
            pcall(function() target:Destroy() end) 
        end
        if isPlayer and (not target or not target:FindFirstChild("Humanoid") or target.Humanoid.Health <= 0) then
            Config.KillPlayer = false
            Config.SelectedPlayerTarget = nil
        end
    end
end

task.spawn(function()
    while task.wait(0.1) do
        if Config.KillPlayer and Config.SelectedPlayerTarget then
            local tgtPlr = Players:FindFirstChild(Config.SelectedPlayerTarget)
            if tgtPlr and tgtPlr.Character then attackTarget(tgtPlr.Character, true) else Config.KillPlayer = false end
        elseif Config.HeroFarm then
            attackTarget(workspace:FindFirstChild("Thug"), false)
        elseif Config.VillianFarm then
            attackTarget(workspace:FindFirstChild("Civilian") or workspace:FindFirstChild("Police"), false)
        elseif Config.FarmAll then
            local targetsPool = {}
            for _, name in ipairs({"Thug", "Civilian", "Police"}) do
                local found = workspace:FindFirstChild(name)
                if found then table.insert(targetsPool, found) end
            end
            if #targetsPool > 0 then
                attackTarget(targetsPool[math.random(1, #targetsPool)], false)
            end
        end
    end
end)

-- =============================================================================
-- INTERFACE GRÁFICA (Premium Dark Minimal)
-- =============================================================================
local UI = Instance.new("ScreenGui")
UI.Name = "shadowls_PremiumMenu"
UI.Parent = player:WaitForChild("PlayerGui")
UI.ResetOnSpawn = false

local function AddCorner(parent, radius)
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, radius)
    corner.Parent = parent
end

local MenuToggle = Instance.new("TextButton")
MenuToggle.Name = "MenuToggle"
MenuToggle.Parent = UI
MenuToggle.Size = UDim2.new(0, 40, 0, 40)
MenuToggle.Position = UDim2.new(0, 20, 0, 20)
MenuToggle.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
MenuToggle.Text = "🦉"
MenuToggle.TextSize = 18
MenuToggle.TextColor3 = Color3.fromRGB(255, 255, 255)
AddCorner(MenuToggle, 8)
local ToggleStroke = Instance.new("UIStroke")
ToggleStroke.Color = Color3.fromRGB(40, 40, 50)
ToggleStroke.Parent = MenuToggle

local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Parent = UI
MainFrame.Size = UDim2.new(0, 550, 0, 320)
MainFrame.Position = UDim2.new(0.5, -275, 0.5, -160)
MainFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
MainFrame.BorderSizePixel = 0
AddCorner(MainFrame, 10)

-- MENU LATERAL DE JOGADORES (LADO DIREITO DO SCRIPT)
local PListPanel = Instance.new("Frame")
PListPanel.Name = "PlayerListPanel"
PListPanel.Parent = MainFrame
PListPanel.Size = UDim2.new(0, 180, 1, 0)
PListPanel.Position = UDim2.new(1, 5, 0, 0)
PListPanel.BackgroundColor3 = Color3.fromRGB(18, 18, 18)
PListPanel.Visible = false
AddCorner(PListPanel, 10)

local PListTitle = Instance.new("TextLabel")
PListTitle.Parent = PListPanel
PListTitle.Size = UDim2.new(1, 0, 0, 35)
PListTitle.BackgroundTransparency = 1
PListTitle.Text = "Selecione o Alvo"
PListTitle.Font = Enum.Font.GothamBold
PListTitle.TextSize = 13
PListTitle.TextColor3 = Color3.fromRGB(200, 200, 200)

local PlayerListFrame = Instance.new("ScrollingFrame")
PlayerListFrame.Size = UDim2.new(1, -10, 1, -45)
PlayerListFrame.Position = UDim2.new(0, 5, 0, 35)
PlayerListFrame.BackgroundTransparency = 1
PlayerListFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
PlayerListFrame.ScrollBarThickness = 3
PlayerListFrame.ScrollBarImageColor3 = Color3.fromRGB(80, 80, 80)
PlayerListFrame.Parent = PListPanel

local PlayerListLayout = Instance.new("UIListLayout")
PlayerListLayout.Parent = PlayerListFrame
PlayerListLayout.Padding = UDim.new(0, 4)

local CloseBtn = Instance.new("TextButton")
CloseBtn.Name = "CloseBtn"
CloseBtn.Parent = MainFrame
CloseBtn.Size = UDim2.new(0, 20, 0, 20)
CloseBtn.Position = UDim2.new(1, -25, 0, 8)
CloseBtn.BackgroundTransparency = 1
CloseBtn.Text = "x"
CloseBtn.Font = Enum.Font.Gotham
CloseBtn.TextSize = 16
CloseBtn.TextColor3 = Color3.fromRGB(150, 150, 150)

local Sidebar = Instance.new("Frame")
Sidebar.Name = "Sidebar"
Sidebar.Parent = MainFrame
Sidebar.Size = UDim2.new(0, 160, 1, 0)
Sidebar.BackgroundColor3 = Color3.fromRGB(18, 18, 18)
AddCorner(Sidebar, 10)

local FixCorner = Instance.new("Frame")
FixCorner.Size = UDim2.new(0, 10, 1, 0)
FixCorner.Position = UDim2.new(1, -10, 0, 0)
FixCorner.BackgroundColor3 = Color3.fromRGB(18, 18, 18)
FixCorner.BorderSizePixel = 0
FixCorner.Parent = Sidebar

local InfoLabel = Instance.new("TextLabel")
InfoLabel.Parent = Sidebar
InfoLabel.Size = UDim2.new(1, -20, 0, 50)
InfoLabel.Position = UDim2.new(0, 15, 0, 10)
InfoLabel.BackgroundTransparency = 1
InfoLabel.Text = "shadowls\nAge of Heroes"
InfoLabel.Font = Enum.Font.GothamBold
InfoLabel.TextSize = 14
InfoLabel.TextColor3 = Color3.fromRGB(230, 230, 230)
InfoLabel.TextXAlignment = Enum.TextXAlignment.Left

local TabContainer = Instance.new("Frame")
TabContainer.Parent = Sidebar
TabContainer.Size = UDim2.new(1, -20, 1, -70)
TabContainer.Position = UDim2.new(0, 10, 0, 65)
TabContainer.BackgroundTransparency = 1

local TabList = Instance.new("UIListLayout")
TabList.Parent = TabContainer
TabList.Padding = UDim.new(0, 5)

local ContentContainer = Instance.new("Frame")
ContentContainer.Parent = MainFrame
ContentContainer.Size = UDim2.new(1, -180, 1, -40)
ContentContainer.Position = UDim2.new(0, 170, 0, 30)
ContentContainer.BackgroundTransparency = 1

local Pages = {}
local TabButtons = {}

local function CreatePage(pageName)
    local Page = Instance.new("ScrollingFrame")
    Page.Size = UDim2.new(1, 0, 1, 0)
    Page.BackgroundTransparency = 1
    Page.Visible = false
    Page.CanvasSize = UDim2.new(0, 0, 0, 0)
    Page.ScrollBarThickness = 0
    Page.Parent = ContentContainer
    
    local List = Instance.new("UIListLayout")
    List.Parent = Page
    List.Padding = UDim.new(0, 8)
    
    List:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        Page.CanvasSize = UDim2.new(0, 0, 0, List.AbsoluteContentSize.Y + 10)
    end)
    
    Pages[pageName] = Page
    
    local TabBtn = Instance.new("TextButton")
    TabBtn.Size = UDim2.new(1, 0, 0, 32)
    TabBtn.BackgroundColor3 = Color3.fromRGB(24, 24, 24)
    TabBtn.Text = "  " .. pageName
    TabBtn.Font = Enum.Font.GothamSemibold
    TabBtn.TextSize = 12
    TabBtn.TextColor3 = Color3.fromRGB(140, 140, 140)
    TabBtn.TextXAlignment = Enum.TextXAlignment.Left
    AddCorner(TabBtn, 6)
    TabBtn.Parent = TabContainer
    
    TabButtons[pageName] = TabBtn
    
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

CreatePage("Farming")
CreatePage("Player")
CreatePage("Visual")
CreatePage("Utility")
CreatePage("Server")

Pages["Farming"].Visible = true
TabButtons["Farming"].BackgroundColor3 = Color3.fromRGB(28, 28, 28)
TabButtons["Farming"].TextColor3 = Color3.fromRGB(255, 255, 255)

local function NewToggle(parentPage, labelText, configKey)
    local Row = Instance.new("Frame")
    Row.Size = UDim2.new(1, 0, 0, 40)
    Row.BackgroundColor3 = Color3.fromRGB(22, 22, 22)
    Row.BorderSizePixel = 0
    AddCorner(Row, 6)
    Row.Parent = Pages[parentPage]
    
    local Label = Instance.new("TextLabel")
    Label.Size = UDim2.new(0.7, 0, 1, 0)
    Label.Position = UDim2.new(0, 12, 0, 0)
    Label.BackgroundTransparency = 1
    Label.Text = labelText
    Label.Font = Enum.Font.Gotham
    Label.TextSize = 13
    Label.TextColor3 = Color3.fromRGB(180, 180, 180)
    Label.TextXAlignment = Enum.TextXAlignment.Left
    Label.Parent = Row
    
    local SwitchOuter = Instance.new("TextButton")
    SwitchOuter.Size = UDim2.new(0, 36, 0, 20)
    SwitchOuter.Position = UDim2.new(1, -48, 0.5, -10)
    SwitchOuter.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
    SwitchOuter.Text = ""
    AddCorner(SwitchOuter, 10)
    SwitchOuter.Parent = Row
    
    local SwitchBall = Instance.new("Frame")
    SwitchBall.Size = UDim2.new(0, 14, 0, 14)
    SwitchBall.Position = UDim2.new(0, 3, 0.5, -7)
    SwitchBall.BackgroundColor3 = Color3.fromRGB(100, 100, 100)
    AddCorner(SwitchBall, 10)
    SwitchBall.Parent = SwitchOuter
    
    SwitchOuter.MouseButton1Click:Connect(function()
        Config[configKey] = not Config[configKey]
        if Config[configKey] then
            TweenService:Create(SwitchOuter, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(0, 140, 255)}):Play()
            TweenService:Create(SwitchBall, TweenInfo.new(0.2), {Position = UDim2.new(1, -17, 0.5, -7), BackgroundColor3 = Color3.fromRGB(255, 255, 255)}):Play()
        else
            TweenService:Create(SwitchOuter, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(45, 45, 45)}):Play()
            TweenService:Create(SwitchBall, TweenInfo.new(0.2), {Position = UDim2.new(0, 3, 0.5, -7), BackgroundColor3 = Color3.fromRGB(100, 100, 100)}):Play()
        end
    end)
end

local function NewButton(parentPage, labelText, callback)
    local Btn = Instance.new("TextButton")
    Btn.Size = UDim2.new(1, 0, 0, 38)
    Btn.BackgroundColor3 = Color3.fromRGB(24, 24, 24)
    Btn.Text = labelText
    Btn.Font = Enum.Font.GothamSemibold
    Btn.TextSize = 13
    Btn.TextColor3 = Color3.fromRGB(200, 200, 200)
    AddCorner(Btn, 6)
    Btn.Parent = Pages[parentPage]
    
    Btn.MouseButton1Click:Connect(callback)
end

-- Populando as Abas
NewToggle("Farming", "Auto Collect Orbs", "OrbFarm")
NewToggle("Farming", "Hero Farm (Thugs)", "HeroFarm")
NewToggle("Farming", "Villian Farm (Civils/Police)", "VillianFarm")
NewToggle("Farming", "Farm All (Everything)", "FarmAll")

NewToggle("Player", "Kill Selected Player", "KillPlayer")

local SelectedLabel = Instance.new("TextLabel")
SelectedLabel.Size = UDim2.new(1, 0, 0, 25)
SelectedLabel.BackgroundTransparency = 1
SelectedLabel.Text = "Alvo Selecionado: Nenhum"
SelectedLabel.Font = Enum.Font.GothamSemibold
SelectedLabel.TextSize = 12
SelectedLabel.TextColor3 = Color3.fromRGB(0, 140, 255)
SelectedLabel.TextXAlignment = Enum.TextXAlignment.Left
SelectedLabel.Parent = Pages["Player"]

PlayerListLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    PlayerListFrame.CanvasSize = UDim2.new(0, 0, 0, PlayerListLayout.AbsoluteContentSize.Y + 5)
end)

local function UpdatePlayerList()
    for _, child in ipairs(PlayerListFrame:GetChildren()) do
        if child:IsA("TextButton") then child:Destroy() end
    end
    
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= player then
            local PBtn = Instance.new("TextButton")
            PBtn.Size = UDim2.new(1, -10, 0, 30)
            PBtn.BackgroundColor3 = (Config.SelectedPlayerTarget == p.Name) and Color3.fromRGB(0, 140, 255) or Color3.fromRGB(24, 24, 24)
            PBtn.Text = "  " .. p.DisplayName
            PBtn.Font = Enum.Font.Gotham
            PBtn.TextSize = 12
            PBtn.TextColor3 = (Config.SelectedPlayerTarget == p.Name) and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(180, 180, 180)
            PBtn.TextXAlignment = Enum.TextXAlignment.Left
            AddCorner(PBtn, 4)
            PBtn.Parent = PlayerListFrame
            
            PBtn.MouseButton1Click:Connect(function()
                Config.SelectedPlayerTarget = p.Name
                SelectedLabel.Text = "Alvo Selecionado: " .. p.DisplayName
                UpdatePlayerList()
            end)
        end
    end
end

Players.PlayerAdded:Connect(UpdatePlayerList)
Players.PlayerRemoving:Connect(function(p)
    if Config.SelectedPlayerTarget == p.Name then
        Config.SelectedPlayerTarget = nil
        Config.KillPlayer = false
        SelectedLabel.Text = "Alvo Selecionado: Nenhum"
    end
    UpdatePlayerList()
end)

UpdatePlayerList()

-- Novo Botão na Aba Player para abrir a lista lateral
NewButton("Player", "Selecionar Jogador", function()
    PListPanel.Visible = not PListPanel.Visible
end)

-- Aba Visual
NewToggle("Visual", "Hide Username / Title", "HideTitle")
NewButton("Visual", "Go Invisible", function() goInvisible() end) -- Movido para cá

-- Aba Server
NewButton("Server", "Rejoin Server", function() TeleportService:Teleport(game.PlaceId, player) end)
NewButton("Server", "Serverhop", function() ServerHop() end)

-- Interatividade
MainFrame.Visible = true

MenuToggle.MouseButton1Click:Connect(function()
    MainFrame.Visible = not MainFrame.Visible
    if not MainFrame.Visible then PListPanel.Visible = false end
end)

CloseBtn.MouseButton1Click:Connect(function()
    MainFrame.Visible = false
    PListPanel.Visible = false
end)

-- Sistema de Arrastar (Drag) Corrigido
local dragging, dragInput, dragStart, startPos
local function update(input)
    local delta = input.Position - dragStart
    MenuToggle.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
end

MenuToggle.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = MenuToggle.Position
        
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        update(input)
    end
end)
