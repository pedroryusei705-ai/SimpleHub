--//====================================================--
--//         SIMPLE HUB - SPEED + INFINITE JUMP        --//
--//====================================================--
--//  Exibe o nome da experiência +                     --//
--//  Speed Boost (ajustável) + Infinite Jump          --//
--//  Atalhos: Keypad 1 = Speed, Keypad 2 = Jump       --//
--//====================================================--

local Players = game:GetService("Players")
local CoreGui = game:GetService("CoreGui")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local LocalPlayer = Players.LocalPlayer

-- Obtém o nome da experiência
local gameName = game.Name
local success, info = pcall(function()
    return game:GetService("MarketplaceService"):GetProductInfo(game.PlaceId)
end)
if success and info and info.Name then
    gameName = info.Name
end

-- Estado das features
local state = {
    speedEnabled = false,
    speedValue = 160,
    infiniteJumpEnabled = false,
}

-- Cache do personagem
local charCache = { root = nil, hum = nil, lastUpdate = 0 }
local function getCharComponents()
    local now = tick()
    if now - charCache.lastUpdate > 0.3 then
        local char = LocalPlayer.Character
        charCache.root = char and char:FindFirstChild("HumanoidRootPart")
        charCache.hum = char and char:FindFirstChildOfClass("Humanoid")
        charCache.lastUpdate = now
    end
    return charCache.root, charCache.hum
end

-- Conexões ativas
local connections = {}

local function disconnect(name)
    if connections[name] then
        connections[name]:Disconnect()
        connections[name] = nil
    end
end

-- Speed loop (igual ao FFO)
local function speedLoop()
    if not state.speedEnabled then return end
    local root, hum = getCharComponents()
    if root and hum then
        local moveDir = hum.MoveDirection
        if moveDir.Magnitude > 0 then
            local curY = root.AssemblyLinearVelocity.Y
            root.AssemblyLinearVelocity = (moveDir.Unit * state.speedValue) + Vector3.new(0, curY, 0)
        end
    end
end

-- Ativa/desativa Speed
local function setSpeed(enabled)
    state.speedEnabled = enabled
    if enabled then
        disconnect("speed")
        connections.speed = RunService.Heartbeat:Connect(speedLoop)
    else
        disconnect("speed")
    end
end

-- Ativa/desativa Infinite Jump (evento JumpRequest)
local function setInfiniteJump(enabled)
    state.infiniteJumpEnabled = enabled
end

-- Cria GUI
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "SimpleHub"
screenGui.ResetOnSpawn = false
screenGui.IgnoreGuiInset = true
screenGui.Parent = CoreGui

-- Frame principal (arrastável)
local mainFrame = Instance.new("Frame")
mainFrame.Size = UDim2.new(0, 340, 0, 320)
mainFrame.Position = UDim2.new(0.5, -170, 0.5, -160)
mainFrame.BackgroundColor3 = Color3.fromRGB(13, 13, 18)
mainFrame.BorderSizePixel = 0
mainFrame.Active = true
mainFrame.Draggable = true
mainFrame.Parent = screenGui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 10)
corner.Parent = mainFrame

local stroke = Instance.new("UIStroke")
stroke.Color = Color3.fromRGB(100, 100, 120)
stroke.Thickness = 1
stroke.Transparency = 0.6
stroke.Parent = mainFrame

-- Header
local header = Instance.new("Frame")
header.Size = UDim2.new(1, 0, 0, 40)
header.BackgroundColor3 = Color3.fromRGB(18, 18, 25)
header.BorderSizePixel = 0
header.Parent = mainFrame

local headerCorner = Instance.new("UICorner")
headerCorner.CornerRadius = UDim.new(0, 10)
headerCorner.Parent = header

local titleLabel = Instance.new("TextLabel")
titleLabel.Size = UDim2.new(1, -50, 1, 0)
titleLabel.Position = UDim2.new(0, 15, 0, 0)
titleLabel.BackgroundTransparency = 1
titleLabel.Text = "✦  SIMPLE HUB  ✦"
titleLabel.Font = Enum.Font.GothamBold
titleLabel.TextSize = 13
titleLabel.TextColor3 = Color3.fromRGB(100, 180, 255)
titleLabel.TextXAlignment = Enum.TextXAlignment.Left
titleLabel.Parent = header

local closeBtn = Instance.new("TextButton")
closeBtn.Size = UDim2.new(0, 28, 0, 28)
closeBtn.Position = UDim2.new(1, -38, 0, 6)
closeBtn.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
closeBtn.Text = "✕"
closeBtn.Font = Enum.Font.GothamBold
closeBtn.TextSize = 18
closeBtn.TextColor3 = Color3.fromRGB(200, 200, 210)
closeBtn.BorderSizePixel = 0
closeBtn.AutoButtonColor = false
closeBtn.Parent = header

local closeCorner = Instance.new("UICorner")
closeCorner.CornerRadius = UDim.new(0, 6)
closeCorner.Parent = closeBtn

closeBtn.MouseButton1Click:Connect(function()
    screenGui:Destroy()
    for _, conn in pairs(connections) do
        if conn then pcall(conn.Disconnect, conn) end
    end
end)

-- Área de conteúdo (rolável)
local contentFrame = Instance.new("ScrollingFrame")
contentFrame.Size = UDim2.new(1, -20, 1, -55)
contentFrame.Position = UDim2.new(0, 10, 0, 50)
contentFrame.BackgroundTransparency = 1
contentFrame.BorderSizePixel = 0
contentFrame.ScrollBarThickness = 3
contentFrame.ScrollBarImageColor3 = Color3.fromRGB(80, 80, 100)
contentFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
contentFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y
contentFrame.Parent = mainFrame

local layout = Instance.new("UIListLayout")
layout.Padding = UDim.new(0, 8)
layout.SortOrder = Enum.SortOrder.LayoutOrder
layout.Parent = contentFrame

-- Função auxiliar para criar toggle (agora sincronizada com o state)
-- Retorna: container, toggleBg, toggleCircle
local function createToggle(parent, labelText, descText, stateKey, callback)
    local container = Instance.new("Frame")
    container.Size = UDim2.new(1, 0, 0, descText and 55 or 40)
    container.BackgroundColor3 = Color3.fromRGB(18, 18, 25)
    container.BorderSizePixel = 0
    container.Parent = parent

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = container

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(0.7, -10, 0, 20)
    label.Position = UDim2.new(0, 10, 0, descText and 8 or 10)
    label.BackgroundTransparency = 1
    label.Text = labelText
    label.Font = Enum.Font.Gotham
    label.TextSize = 12
    label.TextColor3 = Color3.fromRGB(255, 255, 255)
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = container

    if descText then
        local desc = Instance.new("TextLabel")
        desc.Size = UDim2.new(1, -20, 0, 18)
        desc.Position = UDim2.new(0, 10, 0, 28)
        desc.BackgroundTransparency = 1
        desc.Text = descText
        desc.Font = Enum.Font.Gotham
        desc.TextSize = 9
        desc.TextColor3 = Color3.fromRGB(120, 120, 140)
        desc.TextXAlignment = Enum.TextXAlignment.Left
        desc.TextWrapped = true
        desc.Parent = container
    end

    local toggleBg = Instance.new("Frame")
    toggleBg.Size = UDim2.new(0, 38, 0, 20)
    toggleBg.Position = UDim2.new(1, -45, 0, 10)
    toggleBg.BorderSizePixel = 0
    toggleBg.Parent = container

    local bgCorner = Instance.new("UICorner")
    bgCorner.CornerRadius = UDim.new(1, 0)
    bgCorner.Parent = toggleBg

    local toggleCircle = Instance.new("Frame")
    toggleCircle.Size = UDim2.new(0, 16, 0, 16)
    toggleCircle.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    toggleCircle.BorderSizePixel = 0
    toggleCircle.Parent = toggleBg

    local circleCorner = Instance.new("UICorner")
    circleCorner.CornerRadius = UDim.new(1, 0)
    circleCorner.Parent = toggleCircle

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, 0, 1, 0)
    btn.BackgroundTransparency = 1
    btn.Text = ""
    btn.Parent = toggleBg

    local function updateVisual(enabled)
        TweenService:Create(toggleBg, TweenInfo.new(0.2), {
            BackgroundColor3 = enabled and Color3.fromRGB(100, 180, 255) or Color3.fromRGB(40, 40, 55)
        }):Play()
        TweenService:Create(toggleCircle, TweenInfo.new(0.2), {
            Position = enabled and UDim2.new(1, -18, 0.5, -8) or UDim2.new(0, 2, 0.5, -8)
        }):Play()
    end

    -- Inicializa visual de acordo com o estado atual
    updateVisual(state[stateKey])

    btn.MouseButton1Click:Connect(function()
        local newState = not state[stateKey]
        state[stateKey] = newState
        updateVisual(newState)
        if callback then callback(newState) end
    end)

    return container, toggleBg, toggleCircle
end

-- Função para criar slider (estilo FFO)
local function createSlider(parent, labelText, minVal, maxVal, defaultVal, callback)
    local container = Instance.new("Frame")
    container.Size = UDim2.new(1, 0, 0, 50)
    container.BackgroundColor3 = Color3.fromRGB(18, 18, 25)
    container.BorderSizePixel = 0
    container.Parent = parent

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = container

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(0.6, 0, 0, 18)
    label.Position = UDim2.new(0, 10, 0, 8)
    label.BackgroundTransparency = 1
    label.Text = labelText
    label.Font = Enum.Font.Gotham
    label.TextSize = 11
    label.TextColor3 = Color3.fromRGB(255, 255, 255)
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = container

    local valueLabel = Instance.new("TextLabel")
    valueLabel.Size = UDim2.new(0, 50, 0, 18)
    valueLabel.Position = UDim2.new(1, -55, 0, 8)
    valueLabel.BackgroundTransparency = 1
    valueLabel.Text = tostring(defaultVal)
    valueLabel.Font = Enum.Font.GothamBold
    valueLabel.TextSize = 11
    valueLabel.TextColor3 = Color3.fromRGB(100, 180, 255)
    valueLabel.TextXAlignment = Enum.TextXAlignment.Right
    valueLabel.Parent = container

    local sliderBg = Instance.new("Frame")
    sliderBg.Size = UDim2.new(1, -20, 0, 4)
    sliderBg.Position = UDim2.new(0, 10, 1, -15)
    sliderBg.BackgroundColor3 = Color3.fromRGB(30, 30, 42)
    sliderBg.BorderSizePixel = 0
    sliderBg.Parent = container

    local bgCorner = Instance.new("UICorner")
    bgCorner.CornerRadius = UDim.new(1, 0)
    bgCorner.Parent = sliderBg

    local fill = Instance.new("Frame")
    local pct = (defaultVal - minVal) / (maxVal - minVal)
    fill.Size = UDim2.new(pct, 0, 1, 0)
    fill.BackgroundColor3 = Color3.fromRGB(100, 180, 255)
    fill.BorderSizePixel = 0
    fill.Parent = sliderBg

    local fillCorner = Instance.new("UICorner")
    fillCorner.CornerRadius = UDim.new(1, 0)
    fillCorner.Parent = fill

    local sliderBtn = Instance.new("TextButton")
    sliderBtn.Size = UDim2.new(1, 0, 1, 10)
    sliderBtn.Position = UDim2.new(0, 0, 0, -5)
    sliderBtn.BackgroundTransparency = 1
    sliderBtn.Text = ""
    sliderBtn.Parent = sliderBg

    local dragging = false
    sliderBtn.MouseButton1Down:Connect(function() dragging = true end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            local mouseX = UserInputService:GetMouseLocation().X
            local pct = math.clamp((mouseX - sliderBg.AbsolutePosition.X) / sliderBg.AbsoluteSize.X, 0, 1)
            local value = minVal + pct * (maxVal - minVal)
            value = math.floor(value * 10) / 10
            valueLabel.Text = tostring(value)
            fill.Size = UDim2.new(pct, 0, 1, 0)
            if callback then callback(value) end
        end
    end)

    return container
end

-- Seção "Game Info" (nome da experiência)
local infoSection = Instance.new("Frame")
infoSection.Size = UDim2.new(1, 0, 0, 65)
infoSection.BackgroundColor3 = Color3.fromRGB(18, 18, 25)
infoSection.BorderSizePixel = 0
infoSection.Parent = contentFrame

local infoCorner = Instance.new("UICorner")
infoCorner.CornerRadius = UDim.new(0, 6)
infoCorner.Parent = infoSection

local gameIcon = Instance.new("TextLabel")
gameIcon.Size = UDim2.new(0, 40, 1, 0)
gameIcon.BackgroundTransparency = 1
gameIcon.Text = "🎮"
gameIcon.Font = Enum.Font.GothamBold
gameIcon.TextSize = 28
gameIcon.TextColor3 = Color3.fromRGB(100, 180, 255)
gameIcon.Parent = infoSection

local gameNameLabel = Instance.new("TextLabel")
gameNameLabel.Size = UDim2.new(1, -50, 1, 0)
gameNameLabel.Position = UDim2.new(0, 45, 0, 0)
gameNameLabel.BackgroundTransparency = 1
gameNameLabel.Text = gameName
gameNameLabel.Font = Enum.Font.GothamBold
gameNameLabel.TextSize = 16
gameNameLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
gameNameLabel.TextWrapped = true
gameNameLabel.TextScaled = true
gameNameLabel.TextXAlignment = Enum.TextXAlignment.Left
gameNameLabel.Parent = infoSection

-- Seção "Movement Settings"
local movementHeader = Instance.new("Frame")
movementHeader.Size = UDim2.new(1, 0, 0, 28)
movementHeader.BackgroundColor3 = Color3.fromRGB(13, 13, 18)
movementHeader.BorderSizePixel = 0
movementHeader.Parent = contentFrame

local headerLabel = Instance.new("TextLabel")
headerLabel.Size = UDim2.new(1, -10, 1, 0)
headerLabel.Position = UDim2.new(0, 10, 0, 0)
headerLabel.BackgroundTransparency = 1
headerLabel.Text = "⚡ MOVEMENT SETTINGS"
headerLabel.Font = Enum.Font.GothamBold
headerLabel.TextSize = 11
headerLabel.TextColor3 = Color3.fromRGB(180, 180, 200)
headerLabel.TextXAlignment = Enum.TextXAlignment.Left
headerLabel.Parent = movementHeader

-- Criar toggles com as novas funções (agora sincronizadas)
local speedContainer, speedToggleBg, speedToggleCircle = createToggle(contentFrame, "Speed Boost", "Increase movement speed", "speedEnabled", setSpeed)
local speedSlider = createSlider(contentFrame, "Speed Value", 50, 300, state.speedValue, function(value)
    state.speedValue = value
    if state.speedEnabled then
        local root, hum = getCharComponents()
        if root and hum then
            local moveDir = hum.MoveDirection
            if moveDir.Magnitude > 0 then
                local curY = root.AssemblyLinearVelocity.Y
                root.AssemblyLinearVelocity = (moveDir.Unit * value) + Vector3.new(0, curY, 0)
            end
        end
    end
end)

local jumpContainer, jumpToggleBg, jumpToggleCircle = createToggle(contentFrame, "Infinite Jump", "Jump infinitely without limit", "infiniteJumpEnabled", setInfiniteJump)

-- Aplica visual do toggle externamente (para usar com atalhos)
local function updateToggleVisual(toggleBg, toggleCircle, enabled)
    TweenService:Create(toggleBg, TweenInfo.new(0.2), {
        BackgroundColor3 = enabled and Color3.fromRGB(100, 180, 255) or Color3.fromRGB(40, 40, 55)
    }):Play()
    TweenService:Create(toggleCircle, TweenInfo.new(0.2), {
        Position = enabled and UDim2.new(1, -18, 0.5, -8) or UDim2.new(0, 2, 0.5, -8)
    }):Play()
end

-- Atalhos de teclado: Keypad 1 = Speed, Keypad 2 = Infinite Jump
UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == Enum.KeyCode.KeypadOne then
        local newState = not state.speedEnabled
        state.speedEnabled = newState
        updateToggleVisual(speedToggleBg, speedToggleCircle, newState)
        setSpeed(newState)
    elseif input.KeyCode == Enum.KeyCode.KeypadTwo then
        local newState = not state.infiniteJumpEnabled
        state.infiniteJumpEnabled = newState
        updateToggleVisual(jumpToggleBg, jumpToggleCircle, newState)
        setInfiniteJump(newState)
    end
end)

-- Trata o pulo infinito via JumpRequest
UserInputService.JumpRequest:Connect(function()
    if state.infiniteJumpEnabled then
        local _, hum = getCharComponents()
        if hum then
            hum:ChangeState(Enum.HumanoidStateType.Jumping)
        end
    end
end)

-- Reconecta loops após respawn do personagem
LocalPlayer.CharacterAdded:Connect(function()
    charCache.lastUpdate = 0
    if state.speedEnabled then
        disconnect("speed")
        connections.speed = RunService.Heartbeat:Connect(speedLoop)
    end
end)

-- Pequeno texto de rodapé (opcional)
local footer = Instance.new("TextLabel")
footer.Size = UDim2.new(1, 0, 0, 20)
footer.BackgroundTransparency = 1
footer.Text = "F1 para abrir/fechar • Keypad 1 = Speed • Keypad 2 = Jump"
footer.Font = Enum.Font.Gotham
footer.TextSize = 9
footer.TextColor3 = Color3.fromRGB(80, 80, 100)
footer.Parent = contentFrame

-- Hotkey F1 para mostrar/esconder a GUI
UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == Enum.KeyCode.F1 then
        mainFrame.Visible = not mainFrame.Visible
    end
end)

-- Anima entrada
mainFrame.BackgroundTransparency = 1
TweenService:Create(mainFrame, TweenInfo.new(0.4), { BackgroundTransparency = 0 }):Play()

print("═══════════════════════════════════════════════")
print("     SIMPLE HUB - SPEED + INFINITE JUMP       ")
print("═══════════════════════════════════════════════")
print("  Experiência: " .. gameName)
print("  Features: Speed Boost (ajustável) | Infinite Jump")
print("  Atalhos: Keypad 1 = Speed | Keypad 2 = Jump")
print("  Pressione F1 para mostrar/ocultar a GUI")
print("═══════════════════════════════════════════════")
