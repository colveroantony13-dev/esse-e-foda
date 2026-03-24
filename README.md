--[[
    ═══════════════════════════════════════════════════════════════════════════
         TORNADO TRACKER PRO - SISTEMA COMPLETO DE RASTREAMENTO
    ═══════════════════════════════════════════════════════════════════════════
    
    FUNCIONALIDADES:
    ✓ UI moderna com fundo animado de chuva e trovões
    ✓ ESP em tempo real para tornados
    ✓ Previsão de trajetória e rota dos tornados
    ✓ Auto-farm inteligente com teletransporte seguro
    ✓ Indicadores de distância e direção
    ✓ Sistema de logs em tempo real
    ✓ Animações suaves e efeitos visuais
    
    CONTROLES:
    • Botão ESP → Ativa/desativa visualização dos tornados
    • Botão ROTA → Ativa/desativa exibição da trajetória
    • Botão OVERLAY → Ativa/desativa ESP na tela
    • Botão AUTO-FARM → Ativa/desativa teletransporte automático
    • Tecla F → Atalho ESP | Tecla R → Rota | Tecla T → Auto-farm
--]]

-- Carregar serviços
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local CoreGui = game:GetService("CoreGui")
local Workspace = game:GetService("Workspace")
local Lighting = game:GetService("Lighting")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TeleportService = game:GetService("TeleportService")

-- Configurações do jogador
local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()
local Camera = Workspace.CurrentCamera

-- Variáveis de controle
local ESPActive = false
local RouteActive = false
local OverlayActive = true
local AutoFarmActive = false
local Tornados = {}
local ESPObjects = {}
local PredictionLines = {}
local SafePoints = {}

-- Variáveis do Auto-farm
local lastTeleportTime = 0
local teleportCooldown = 5
local safeDistance = 150
local isTeleporting = false

-- Variáveis do clima
local rainSound = nil
local thunderSound = nil
local rainParticles = {}

-- Cores do sistema
local Colors = {
    Primary = Color3.fromRGB(70, 130, 200),
    Secondary = Color3.fromRGB(100, 80, 160),
    Tornado = Color3.fromRGB(80, 180, 255),
    Danger = Color3.fromRGB(255, 50, 50),
    Safe = Color3.fromRGB(50, 255, 100),
    Route = Color3.fromRGB(255, 200, 100),
    Text = Color3.fromRGB(255, 255, 255),
    Background = Color3.fromRGB(15, 15, 35),
    Border = Color3.fromRGB(50, 50, 80)
}

-- Criar ScreenGui principal
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "TornadoTracker"
ScreenGui.Parent = CoreGui
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.IgnoreGuiInset = true

-- ========== FUNDO ANIMADO DE CHUVA ==========
local RainBackground = Instance.new("Frame")
RainBackground.Size = UDim2.new(1, 0, 1, 0)
RainBackground.BackgroundTransparency = 0.85
RainBackground.BackgroundColor3 = Color3.fromRGB(10, 10, 25)
RainBackground.Parent = ScreenGui

-- Efeito de gradiente dinâmico
local SkyGradient = Instance.new("UIGradient")
SkyGradient.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(20, 20, 45)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(35, 25, 55)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(15, 15, 35))
})
SkyGradient.Rotation = 90
SkyGradient.Parent = RainBackground

-- Criar gotas de chuva animadas
local function CreateRainDrops()
    for i = 1, 150 do
        local drop = Instance.new("Frame")
        drop.Size = UDim2.new(0, math.random(1, 3), 0, math.random(8, 15))
        drop.Position = UDim2.new(math.random(), 0, math.random(), 0)
        drop.BackgroundColor3 = Color3.fromRGB(180, 200, 255)
        drop.BackgroundTransparency = math.random(40, 80) / 100
        drop.BorderSizePixel = 0
        drop.Parent = RainBackground
        
        local dropCorner = Instance.new("UICorner")
        dropCorner.CornerRadius = UDim.new(1, 0)
        dropCorner.Parent = drop
        
        -- Animação da chuva
        local speed = math.random(30, 80)
        spawn(function()
            while drop and drop.Parent do
                local yPos = drop.Position.Y.Scale + 0.01
                if yPos > 1 then
                    yPos = 0
                    drop.Position = UDim2.new(math.random(), 0, 0, 0)
                end
                drop.Position = UDim2.new(drop.Position.X.Scale, 0, yPos, 0)
                wait(0.05)
            end
        end)
        
        table.insert(rainParticles, drop)
    end
end

-- Efeito de raio/trovão
local function CreateThunderEffect()
    local thunderFrame = Instance.new("Frame")
    thunderFrame.Size = UDim2.new(1, 0, 1, 0)
    thunderFrame.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    thunderFrame.BackgroundTransparency = 1
    thunderFrame.Parent = ScreenGui
    
    spawn(function()
        while thunderFrame and thunderFrame.Parent do
            wait(math.random(5, 15))
            if not AutoFarmActive and ESPActive then
                thunderFrame.BackgroundTransparency = 0.7
                TweenService:Create(thunderFrame, TweenInfo.new(0.1), {BackgroundTransparency = 0.7}):Play()
                wait(0.05)
                TweenService:Create(thunderFrame, TweenInfo.new(0.2), {BackgroundTransparency = 1}):Play()
                
                -- Efeito de vibração na tela
                local originalPos = ScreenGui.Position
                ScreenGui.Position = UDim2.new(0, math.random(-3, 3), 0, math.random(-3, 3))
                wait(0.05)
                ScreenGui.Position = originalPos
            end
        end
    end)
end

-- ========== UI PRINCIPAL ==========
local MainContainer = Instance.new("Frame")
MainContainer.Size = UDim2.new(0, 400, 0, 650)
MainContainer.Position = UDim2.new(0, 20, 0.5, -325)
MainContainer.BackgroundTransparency = 0.1
MainContainer.BackgroundColor3 = Colors.Background
MainContainer.BorderSizePixel = 0
MainContainer.ClipsDescendants = true
MainContainer.Parent = ScreenGui

-- Arredondamento
local ContainerCorner = Instance.new("UICorner")
ContainerCorner.CornerRadius = UDim.new(0, 20)
ContainerCorner.Parent = MainContainer

-- Sombra
local Shadow = Instance.new("ImageLabel")
Shadow.Size = UDim2.new(1, 40, 1, 40)
Shadow.Position = UDim2.new(0, -20, 0, -20)
Shadow.BackgroundTransparency = 1
Shadow.Image = "rbxassetid://1316045217"
Shadow.ImageColor3 = Color3.fromRGB(0, 0, 0)
Shadow.ImageTransparency = 0.7
Shadow.ScaleType = Enum.ScaleType.Slice
Shadow.SliceCenter = Rect.new(10, 10, 10, 10)
Shadow.Parent = MainContainer

-- Borda gradiente animada
local BorderGradient = Instance.new("Frame")
BorderGradient.Size = UDim2.new(1, 0, 1, 0)
BorderGradient.BackgroundTransparency = 1
BorderGradient.BorderSizePixel = 2
BorderGradient.BorderColor3 = Colors.Primary
BorderGradient.Parent = MainContainer

-- ========== CABEÇALHO ==========
local Header = Instance.new("Frame")
Header.Size = UDim2.new(1, 0, 0, 100)
Header.BackgroundColor3 = Color3.fromRGB(20, 20, 45)
Header.BackgroundTransparency = 0.5
Header.BorderSizePixel = 0
Header.Parent = MainContainer

local HeaderCorner = Instance.new("UICorner")
HeaderCorner.CornerRadius = UDim.new(0, 20)
HeaderCorner.Parent = Header

local WeatherIcon = Instance.new("TextLabel")
WeatherIcon.Size = UDim2.new(0, 50, 0, 50)
WeatherIcon.Position = UDim2.new(0, 20, 0.5, -25)
WeatherIcon.BackgroundTransparency = 1
WeatherIcon.Text = "🌪️"
WeatherIcon.TextColor3 = Colors.Tornado
WeatherIcon.TextSize = 45
WeatherIcon.Font = Enum.Font.GothamBlack
WeatherIcon.Parent = Header

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, -80, 0, 35)
Title.Position = UDim2.new(0, 80, 0, 20)
Title.BackgroundTransparency = 1
Title.Text = "TORNADO TRACKER PRO"
Title.TextColor3 = Colors.Text
Title.TextSize = 22
Title.Font = Enum.Font.GothamBlack
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = Header

local SubTitle = Instance.new("TextLabel")
SubTitle.Size = UDim2.new(1, -80, 0, 25)
SubTitle.Position = UDim2.new(0, 80, 0, 55)
SubTitle.BackgroundTransparency = 1
SubTitle.Text = "Sistema de Rastreamento e Previsão"
SubTitle.TextColor3 = Color3.fromRGB(150, 150, 200)
SubTitle.TextSize = 11
SubTitle.Font = Enum.Font.Gotham
SubTitle.TextXAlignment = Enum.TextXAlignment.Left
SubTitle.Parent = Header

-- ========== BOTÕES DE CONTROLE ==========
local ButtonContainer = Instance.new("Frame")
ButtonContainer.Size = UDim2.new(1, -40, 0, 200)
ButtonContainer.Position = UDim2.new(0, 20, 0, 115)
ButtonContainer.BackgroundTransparency = 1
ButtonContainer.Parent = MainContainer

-- Criar botões modernos
local function CreateModernButton(text, icon, yPos, callback)
    local button = Instance.new("TextButton")
    button.Size = UDim2.new(1, 0, 0, 50)
    button.Position = UDim2.new(0, 0, 0, yPos)
    button.Text = icon .. " " .. text .. " DESATIVADO"
    button.TextColor3 = Colors.Text
    button.BackgroundColor3 = Color3.fromRGB(35, 35, 55)
    button.Font = Enum.Font.GothamBold
    button.TextSize = 14
    button.BorderSizePixel = 0
    button.Parent = ButtonContainer
    
    local buttonCorner = Instance.new("UICorner")
    buttonCorner.CornerRadius = UDim.new(0, 12)
    buttonCorner.Parent = button
    
    -- Efeitos hover
    button.MouseEnter:Connect(function()
        TweenService:Create(button, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(50, 50, 75)}):Play()
    end)
    button.MouseLeave:Connect(function()
        TweenService:Create(button, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(35, 35, 55)}):Play()
    end)
    
    -- Efeito ripple
    button.MouseButton1Click:Connect(function()
        local ripple = Instance.new("Frame")
        ripple.Size = UDim2.new(0, 0, 0, 0)
        ripple.Position = UDim2.new(0.5, 0, 0.5, 0)
        ripple.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        ripple.BackgroundTransparency = 0.5
        ripple.BorderSizePixel = 0
        ripple.Parent = button
        
        local rippleCorner = Instance.new("UICorner")
        rippleCorner.CornerRadius = UDim.new(1, 0)
        rippleCorner.Parent = ripple
        
        TweenService:Create(ripple, TweenInfo.new(0.5), {Size = UDim2.new(2, 0, 2, 0), BackgroundTransparency = 1}):Play()
        game:GetService("Debris"):AddItem(ripple, 0.5)
        
        callback()
    end)
    
    return button
end

-- Variáveis dos botões
local EspButton = nil
local RouteButton = nil
local OverlayButton = nil
local AutoFarmButton = nil

-- Estado dos botões
local buttonStates = {
    esp = false,
    route = false,
    overlay = true,
    autofarm = false
}

-- Função para atualizar texto dos botões
local function UpdateButtonText(button, name, isActive)
    local icon = ""
    if name == "ESP" then icon = "👁️"
    elseif name == "ROTA" then icon = "📍"
    elseif name == "OVERLAY" then icon = "🎯"
    elseif name == "AUTO-FARM" then icon = "⚡"
    end
    
    button.Text = icon .. " " .. name .. (isActive and " ATIVADO" or " DESATIVADO")
    button.BackgroundColor3 = isActive and Color3.fromRGB(76, 175, 80) or Color3.fromRGB(35, 35, 55)
end

-- Criar botões
EspButton = CreateModernButton("ESP", "👁️", 0, function()
    buttonStates.esp = not buttonStates.esp
    ESPActive = buttonStates.esp
    UpdateButtonText(EspButton, "ESP", ESPActive)
    AddLog(ESPActive and "ESP ativado" or "ESP desativado")
end)

RouteButton = CreateModernButton("ROTA", "📍", 60, function()
    buttonStates.route = not buttonStates.route
    RouteActive = buttonStates.route
    UpdateButtonText(RouteButton, "ROTA", RouteActive)
    AddLog(RouteActive and "Previsão de rota ativada" or "Previsão de rota desativada")
end)

OverlayButton = CreateModernButton("OVERLAY", "🎯", 120, function()
    buttonStates.overlay = not buttonStates.overlay
    OverlayActive = buttonStates.overlay
    UpdateButtonText(OverlayButton, "OVERLAY", OverlayActive)
    AddLog(OverlayActive and "Overlay ativado" or "Overlay desativado")
end)

AutoFarmButton = CreateModernButton("AUTO-FARM", "⚡", 180, function()
    buttonStates.autofarm = not buttonStates.autofarm
    AutoFarmActive = buttonStates.autofarm
    UpdateButtonText(AutoFarmButton, "AUTO-FARM", AutoFarmActive)
    AddLog(AutoFarmActive and "Auto-farm ativado - Buscando pontos seguros" or "Auto-farm desativado")
end)

-- ========== PAINEL DE INFORMAÇÕES ==========
local InfoPanel = Instance.new("Frame")
InfoPanel.Size = UDim2.new(1, -40, 0, 140)
InfoPanel.Position = UDim2.new(0, 20, 0, 330)
InfoPanel.BackgroundColor3 = Color3.fromRGB(25, 25, 45)
InfoPanel.BackgroundTransparency = 0.4
InfoPanel.BorderSizePixel = 0
InfoPanel.Parent = MainContainer

local InfoCorner = Instance.new("UICorner")
InfoCorner.CornerRadius = UDim.new(0, 12)
InfoCorner.Parent = InfoPanel

-- Status ESP
local ESPStatus = Instance.new("TextLabel")
ESPStatus.Size = UDim2.new(1, -20, 0, 30)
ESPStatus.Position = UDim2.new(0, 10, 0, 10)
ESPStatus.BackgroundTransparency = 1
ESPStatus.Text = "🌪️ ESP: 🔴 DESATIVADO"
ESPStatus.TextColor3 = Color3.fromRGB(255, 100, 100)
ESPStatus.TextSize = 13
ESPStatus.Font = Enum.Font.GothamBold
ESPStatus.TextXAlignment = Enum.TextXAlignment.Left
ESPStatus.Parent = InfoPanel

-- Status Rota
local RouteStatus = Instance.new("TextLabel")
RouteStatus.Size = UDim2.new(1, -20, 0, 30)
RouteStatus.Position = UDim2.new(0, 10, 0, 45)
RouteStatus.BackgroundTransparency = 1
RouteStatus.Text = "📍 ROTA: 🔴 DESATIVADA"
RouteStatus.TextColor3 = Color3.fromRGB(255, 100, 100)
RouteStatus.TextSize = 13
RouteStatus.Font = Enum.Font.GothamBold
RouteStatus.TextXAlignment = Enum.TextXAlignment.Left
RouteStatus.Parent = InfoPanel

-- Status Auto-farm
local AutoFarmStatus = Instance.new("TextLabel")
AutoFarmStatus.Size = UDim2.new(1, -20, 0, 30)
AutoFarmStatus.Position = UDim2.new(0, 10, 0, 80)
AutoFarmStatus.BackgroundTransparency = 1
AutoFarmStatus.Text = "⚡ AUTO-FARM: 🔴 DESATIVADO"
AutoFarmStatus.TextColor3 = Color3.fromRGB(255, 100, 100)
AutoFarmStatus.TextSize = 13
AutoFarmStatus.Font = Enum.Font.GothamBold
AutoFarmStatus.TextXAlignment = Enum.TextXAlignment.Left
AutoFarmStatus.Parent = InfoPanel

-- Tornado mais próximo
local NearestTornado = Instance.new("TextLabel")
NearestTornado.Size = UDim2.new(1, -20, 0, 30)
NearestTornado.Position = UDim2.new(0, 10, 0, 115)
NearestTornado.BackgroundTransparency = 1
NearestTornado.Text = "📊 TORNADO MAIS PRÓXIMO: --m"
NearestTornado.TextColor3 = Color3.fromRGB(200, 200, 200)
NearestTornado.TextSize = 12
NearestTornado.Font = Enum.Font.Gotham
NearestTornado.TextXAlignment = Enum.TextXAlignment.Left
NearestTornado.Parent = InfoPanel

-- ========== PAINEL DE LOGS ==========
local LogPanel = Instance.new("Frame")
LogPanel.Size = UDim2.new(1, -40, 0, 120)
LogPanel.Position = UDim2.new(0, 20, 0, 485)
LogPanel.BackgroundColor3 = Color3.fromRGB(25, 25, 45)
LogPanel.BackgroundTransparency = 0.4
LogPanel.BorderSizePixel = 0
LogPanel.Parent = MainContainer

local LogCorner = Instance.new("UICorner")
LogCorner.CornerRadius = UDim.new(0, 12)
LogCorner.Parent = LogPanel

local LogTextBox = Instance.new("TextBox")
LogTextBox.Size = UDim2.new(1, -20, 1, -20)
LogTextBox.Position = UDim2.new(0, 10, 0, 10)
LogTextBox.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
LogTextBox.BackgroundTransparency = 0.3
LogTextBox.TextColor3 = Color3.fromRGB(180, 180, 200)
LogTextBox.Text = "> Sistema iniciado\n> Clique nos botões para ativar os recursos\n> Tecla F = ESP | R = Rota | T = Auto-farm"
LogTextBox.TextWrapped = true
LogTextBox.TextXAlignment = Enum.TextXAlignment.Left
LogTextBox.TextYAlignment = Enum.TextYAlignment.Top
LogTextBox.ClearTextOnFocus = false
LogTextBox.Font = Enum.Font.Code
LogTextBox.TextSize = 10
LogTextBox.Parent = LogPanel

local LogCornerInner = Instance.new("UICorner")
LogCornerInner.CornerRadius = UDim.new(0, 8)
LogCornerInner.Parent = LogTextBox

-- ========== FUNÇÕES AUXILIARES ==========
local function AddLog(message)
    local timestamp = os.date("%H:%M:%S")
    local lines = {}
    for line in LogTextBox.Text:gmatch("[^\r\n]+") do
        table.insert(lines, line)
    end
    table.insert(lines, "> [" .. timestamp .. "] " .. message)
    if #lines > 8 then table.remove(lines, 1) end
    LogTextBox.Text = table.concat(lines, "\n")
end

-- Atualizar status na UI
local function UpdateUIStatus()
    ESPStatus.Text = ESPActive and "🌪️ ESP: 🟢 ATIVADO" or "🌪️ ESP: 🔴 DESATIVADO"
    ESPStatus.TextColor3 = ESPActive and Color3.fromRGB(100, 255, 100) or Color3.fromRGB(255, 100, 100)
    
    RouteStatus.Text = RouteActive and "📍 ROTA: 🟢 ATIVADA" or "📍 ROTA: 🔴 DESATIVADA"
    RouteStatus.TextColor3 = RouteActive and Color3.fromRGB(100, 255, 100) or Color3.fromRGB(255, 100, 100)
    
    AutoFarmStatus.Text = AutoFarmActive and "⚡ AUTO-FARM: 🟢 ATIVADO" or "⚡ AUTO-FARM: 🔴 DESATIVADO"
    AutoFarmStatus.TextColor3 = AutoFarmActive and Color3.fromRGB(100, 255, 100) or Color3.fromRGB(255, 100, 100)
end

-- ========== DETECTAR TORNADOS ==========
local function FindTornados()
    local found = {}
    for _, obj in pairs(Workspace:GetDescendants()) do
        if obj:IsA("Model") or obj:IsA("Part") then
            local name = obj.Name:lower()
            if name:find("tornado") or name:find("cyclone") or name:find("twister") or 
               name:find("vortex") or name:find("storm") or name:find("tornado") or
               (obj:IsA("Part") and obj.Size.Y > 15 and obj.Size.X > 8) then
                local root = obj:FindFirstChild("HumanoidRootPart") or obj:FindFirstChild("Head") or obj
                if root and root:IsA("BasePart") then
                    table.insert(found, root)
                end
            end
        end
    end
    return found
end

-- ========== SISTEMA ESP ==========
local function CreateTornadoESP(tornado)
    if ESPObjects[tornado] then return end
    
    -- Contorno do tornado
    local outline = Instance.new("Frame")
    outline.Size = UDim2.new(0, 80, 0, 80)
    outline.BackgroundTransparency = 0.8
    outline.BackgroundColor3 = Colors.Tornado
    outline.BorderSizePixel = 3
    outline.BorderColor3 = Colors.Tornado
    outline.Visible = false
    outline.Parent = ScreenGui
    
    local outlineCorner = Instance.new("UICorner")
    outlineCorner.CornerRadius = UDim.new(1, 0)
    outlineCorner.Parent = outline
    
    -- Ícone do tornado
    local icon = Instance.new("TextLabel")
    icon.Size = UDim2.new(0, 40, 0, 40)
    icon.Position = UDim2.new(0.5, -20, 0.5, -20)
    icon.BackgroundTransparency = 1
    icon.Text = "🌪️"
    icon.TextColor3 = Colors.Tornado
    icon.TextSize = 35
    icon.Font = Enum.Font.GothamBlack
    icon.Parent = outline
    
    -- Texto de distância
    local distLabel = Instance.new("TextLabel")
    distLabel.Size = UDim2.new(0, 80, 0, 20)
    distLabel.Position = UDim2.new(0.5, -40, 1, 5)
    distLabel.BackgroundTransparency = 1
    distLabel.TextColor3 = Colors.Text
    distLabel.TextSize = 11
    distLabel.Font = Enum.Font.GothamBold
    distLabel.Parent = outline
    
    -- Barra de intensidade/perigo
    local dangerBar = Instance.new("Frame")
    dangerBar.Size = UDim2.new(1, 0, 0, 3)
    dangerBar.Position = UDim2.new(0, 0, 1, 8)
    dangerBar.BackgroundColor3 = Colors.Tornado
    dangerBar.BorderSizePixel = 0
    dangerBar.Parent = outline
    
    ESPObjects[tornado] = {
        Outline = outline,
        Icon = icon,
        Dist = distLabel,
        DangerBar = dangerBar
    }
end

local function UpdateESP()
    if not ESPActive then
        for _, data in pairs(ESPObjects) do
            if data.Outline then data.Outline.Visible = false end
        end
        return
    end
    
    local tornados = FindTornados()
    local closestDistance = math.huge
    
    for _, tornado in pairs(tornados) do
        if tornado and tornado.Parent then
            CreateTornadoESP(tornado)
            local data = ESPObjects[tornado]
            
            if data and OverlayActive then
                local screenPos, onScreen = Camera:WorldToScreenPoint(tornado.Position)
                local distance = (LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") and 
                    (tornado.Position - LocalPlayer.Character.HumanoidRootPart.Position).Magnitude) or 0
                
                if distance < closestDistance then
                    closestDistance = distance
                end
                
                if onScreen then
                    local size = math.clamp(200 / (distance / 10), 50, 150)
                    data.Outline.Size = UDim2.new(0, size, 0, size)
                    data.Outline.Position = UDim2.new(0, screenPos.X - size/2, 0, screenPos.Y - size/2)
                    data.Outline.Visible = true
                    
                    -- Efeito pulsante
                    local pulse = math.sin(tick() * 4) * 0.3 + 0.5
                    data.Outline.BackgroundTransparency = 0.6 + pulse * 0.3
                    data.Outline.BorderSizePixel = 2 + pulse * 2
                    
                    -- Distância
                    data.Dist.Text = string.format("%.0fm", distance)
                    data.Dist.TextColor3 = distance < 100 and Colors.Danger or Colors.Text
                    
                    -- Barra de perigo
                    local dangerLevel = math.clamp(1 - (distance / 200), 0, 1)
                    data.DangerBar.Size = UDim2.new(dangerLevel, 0, 1, 0)
                    data.DangerBar.BackgroundColor3 = distance < 100 and Colors.Danger or Colors.Tornado
                    
                    -- Mudar cor se perigo iminente
                    if distance < 75 then
                        data.Outline.BorderColor3 = Colors.Danger
                        data.Icon.TextColor3 = Colors.Danger
                    else
                        data.Outline.BorderColor3 = Colors.Tornado
                        data.Icon.TextColor3 = Colors.Tornado
                    end
                else
                    data.Outline.Visible = false
                end
            end
        end
    end
    
    -- Atualizar texto do tornado mais próximo
    if closestDistance ~= math.huge then
        NearestTornado.Text = string.format("📊 TORNADO MAIS PRÓXIMO: %.0fm", closestDistance)
        NearestTornado.TextColor3 = closestDistance < 100 and Colors.Danger or Color3.fromRGB(200, 200, 200)
    else
        NearestTornado.Text = "📊 TORNADO MAIS PRÓXIMO: --m"
    end
    
    -- Limpar ESP de tornados que não existem mais
    for tornado, data in pairs(ESPObjects) do
        if not tornado or not tornado.Parent then
            if data.Outline then data.Outline:Destroy() end
            ESPObjects[tornado] = nil
        end
    end
end

-- ========== PREVISÃO DE TRAJETÓRIA ==========
local function CalculateTrajectory(tornado, steps, stepSize)
    local points = {}
    if not tornado or not tornado.Parent then return points end
    
    local currentPos = tornado.Position
    local currentVelocity = tornado.AssemblyLinearVelocity or Vector3.new(0, 0, 0)
    
    if currentVelocity.Magnitude < 0.1 then
        local time = tick()
        currentVelocity = Vector3.new(
            math.sin(time) * 12,
            math.random(1, 5),
            math.cos(time) * 12
        )
    end
    
    for i = 1, steps do
        local predictedPos = currentPos + (currentVelocity * stepSize * i)
        predictedPos = predictedPos + Vector3.new(
            math.sin(tick() + i) * 2,
            math.cos(tick() * 0.5 + i) * 1.5,
            math.cos(tick() + i) * 2
        )
        table.insert(points, predictedPos)
    end
    
    return points
end

local function CreateTrajectoryLine(tornado)
    if PredictionLines[tornado] then return end
    
    PredictionLines[tornado] = {
        Lines = {},
        Markers = {}
    }
end

local function UpdateTrajectory()
    if not RouteActive then
        for _, data in pairs(PredictionLines) do
            for _, line in pairs(data.Lines) do
                if line then line:Destroy() end
            end
            for _, marker in pairs(data.Markers) do
                if marker then marker:Destroy() end
            end
            data.Lines = {}
            data.Markers = {}
        end
        return
    end
    
    local tornados = FindTornados()
    
    for _, tornado in pairs(tornados) do
        if tornado and tornado.Parent then
            CreateTrajectoryLine(tornado)
            local data = PredictionLines[tornado]
            
            -- Limpar linhas antigas
            for _, line in pairs(data.Lines) do
                if line then line:Destroy() end
            end
            for _, marker in pairs(data.Markers) do
                if marker then marker:Destroy() end
            end
            data.Lines = {}
            data.Markers = {}
            
            -- Calcular pontos da trajetória
            local points = CalculateTrajectory(tornado, 20, 0.6)
            local screenPoints = {}
            
            for _, point in pairs(points) do
                local screenPos, onScreen = Camera:WorldToScreenPoint(point)
                if onScreen then
                    table.insert(screenPoints, screenPos)
                end
            end
            
            -- Desenhar linhas entre os pontos
            if #screenPoints > 1 then
                for i = 1, #screenPoints - 1 do
                    local p1 = screenPoints[i]
                    local p2 = screenPoints[i + 1]
                    
                    local length = math.sqrt((p2.X - p1.X)^2 + (p2.Y - p1.Y)^2)
                    local angle = math.atan2(p2.Y - p1.Y, p2.X - p1.X)
                    
                    local lineFrame = Instance.new("Frame")
                    lineFrame.Size = UDim2.new(0, length, 0, 4)
                    lineFrame.Position = UDim2.new(0, p1.X, 0, p1.Y)
                    lineFrame.Rotation = math.deg(angle)
                    lineFrame.BackgroundColor3 = Colors.Route
                    lineFrame.BackgroundTransparency = 0.3
                    lineFrame.BorderSizePixel = 0
                    lineFrame.Parent = ScreenGui
                    
                    table.insert(data.Lines, lineFrame)
                    
                    -- Efeito de brilho
                    local glow = Instance.new("Frame")
                    glow.Size = UDim2.new(1, 6, 1, 2)
                    glow.Position = UDim2.new(0, -3, 0, -1)
                    glow.BackgroundColor3 = Colors.Route
                    glow.BackgroundTransparency = 0.7
                    glow.BorderSizePixel = 0
                    glow.Parent = lineFrame
                end
            end
            
            -- Adicionar marcadores de posição futura
            for i, point in pairs(points) do
                if i % 4 == 0 then
                    local screenPos, onScreen = Camera:WorldToScreenPoint(point)
                    if onScreen then
                        local marker = Instance.new("Frame")
                        marker.Size = UDim2.new(0, 8, 0, 8)
                        marker.Position = UDim2.new(0, screenPos.X - 4, 0, screenPos.Y - 4)
                        marker.BackgroundColor3 = Colors.Route
                        marker.BackgroundTransparency = 0.4 + (i / 40)
                        marker.BorderSizePixel = 0
                        marker.Parent = ScreenGui
                        
                        local markerCorner = Instance.new("UICorner")
                        markerCorner.CornerRadius = UDim.new(1, 0)
                        markerCorner.Parent = marker
                        
                        table.insert(data.Markers, marker)
                    end
                end
            end
        end
    end
end

-- ========== SISTEMA AUTO-FARM ==========
local function FindSafePoint(tornado)
    if not tornado or not tornado.Parent then return nil end
    
    -- Encontrar pontos seguros nas ruas
    local safeSpots = {}
    local tornadoPos = tornado.Position
    local tornadoDirection = tornado.AssemblyLinearVelocity or Vector3.new(1, 0, 1)
    local directionNorm = tornadoDirection.Unit
    
    -- Buscar por estradas/ruas no mapa
    for _, obj in pairs(Workspace:GetDescendants()) do
        if obj:IsA("Part") and (obj.Name:lower():find("road") or obj.Name:lower():find("street") or 
           obj.Name:lower():find("pavement") or obj.Name:lower():find("asphalt")) then
            local point = obj.Position
            local distanceToTornado = (point - tornadoPos).Magnitude
            
            -- Verificar se está à frente do tornado
            local relativePos = point - tornadoPos
            local dot = relativePos:Dot(directionNorm)
            
            if dot > 0 and distanceToTornado > safeDistance and distanceToTornado < safeDistance + 100 then
