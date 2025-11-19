-- AutoFish Pro - Ultra Otimizado com Precisão Máxima
-- Sistema completo sem auto chat
local RunService = game:GetService('RunService')
local Players = game:GetService('Players')
local ReplicatedStorage = game:GetService('ReplicatedStorage')
local UserInputService = game:GetService('UserInputService')
local GuiService = game:GetService('GuiService')
local TweenService = game:GetService('TweenService')
local VirtualUser = game:GetService('VirtualUser')
-- Detecção de Plataforma
local IS_MOBILE = false
local IS_PC = true
local PLATFORM = "PC"
local VIM = nil
pcall(function() VIM = game:GetService('VirtualInputManager') end)
-- Players & Core
local player = Players.LocalPlayer
local playerGui = player:WaitForChild('PlayerGui')
local camera = workspace.CurrentCamera
-- Remote Events
local RemoteEvent = ReplicatedStorage:WaitForChild('Modules'):WaitForChild('Events'):WaitForChild('RemoteEvent')
-- ========================================
-- CONFIGURAÇÕES PRINCIPAIS
-- ========================================
local CONFIG = {
    -- Performance
    CHECK_INTERVAL = 0.01,
    CACHE_UPDATE = 0.01,
   
    -- Minigame 1 (Circular)
    CIRCLE_MIN_RATIO = 0.88,
    CIRCLE_MAX_RATIO = 0.99,
    CIRCLE_SWEET_SPOT = 0.90,
    CIRCLE_CLICK_DELAY = 0.25,
   
    -- Minigame 2 (Barra)
    BAR_CENTER_THRESHOLD = 0.22,
    BAR_CLICK_DELAY = 0.55,
    BAR_MOVEMENT_MIN = 3,
   
    -- Sistema
    ACTION_TIMEOUT = 50,
    CAST_DELAY = 0.5,
    DECISION_DELAY = 1.4,
    RESET_DELAY = 1.8,
   
    -- UI
    UI_SCALE = 1.0
}
-- ========================================
-- SISTEMA DE INPUT UNIVERSAL
-- ========================================
local InputManager = {}
function InputManager:Initialize()
    self.inputMethod = nil
   
    if VIM then
        self.inputMethod = "VIM"
    else
        self.inputMethod = "Mouse"
    end
end
function InputManager:SendClick(x, y)
    pcall(function()
        VirtualUser:CaptureController()
        VirtualUser:Button1Down(Vector2.new(x, y))
    end)
   
    if self.inputMethod == "VIM" and VIM then
        VIM:SendMouseButtonEvent(x, y, 0, true, game, 0)
        task.wait(0.04)
        VIM:SendMouseButtonEvent(x, y, 0, false, game, 0)
       
    else
        pcall(function()
            if mouse1click then
                mouse1click()
            end
        end)
    end
end
function InputManager:ClickCenter()
    local centerX = camera.ViewportSize.X * 0.5
    local centerY = camera.ViewportSize.Y * 0.5
    self:SendClick(centerX, centerY)
end
function InputManager:ClickElement(element)
    if not element then return false end
   
    local x = element.AbsolutePosition.X + element.AbsoluteSize.X * 0.5
    local y = element.AbsolutePosition.Y + element.AbsoluteSize.Y * 0.5
   
    self:SendClick(x, y)
    return true
end
-- ========================================
-- SISTEMA DE CACHE OTIMIZADO
-- ========================================
local ElementCache = {
    elements = {},
    lastUpdate = 0,
    updateCount = 0
}
function ElementCache:FastUpdate()
    local now = tick()
    if now - self.lastUpdate < CONFIG.CACHE_UPDATE then
        return self.elements
    end
   
    self.lastUpdate = now
    self.updateCount = self.updateCount + 1
   
    -- Reset cache
    self.elements = {
        CircularIndicator = nil,
        AnimatedCircle = nil,
        Target = nil,
        RedBar = nil,
        CatchButton = nil,
        ReleaseButton = nil,
        Minigame2Active = false,
        HasDecision = false
    }
   
    -- Busca otimizada
    pcall(function()
        local descendants = playerGui:GetDescendants()
       
        for _, obj in ipairs(descendants) do
            if not obj:IsA("GuiObject") or not obj.Visible then continue end
           
            local objClass = obj.ClassName
            local objName = obj.Name
   
            -- Circular Indicator (Minigame 1)
            if objClass == "ImageButton" and not self.elements.CircularIndicator then
                local animated = obj:FindFirstChild("AnimatedWhiteCircle")
                local middle = obj:FindFirstChild("MiddleCircle")
               
                if animated and middle then
                    self.elements.CircularIndicator = obj
                    self.elements.AnimatedCircle = animated
                end
            end
           
            -- Target (Minigame 2)
            if objName == "Target" and not self.elements.Target then
                self.elements.Target = obj
               
                -- Busca RedBar
                local parent = obj.Parent
                if parent then
                    for _, sibling in ipairs(parent:GetChildren()) do
                        if sibling:IsA("Frame") and sibling.ZIndex == 3 and sibling.Visible then
                            self.elements.RedBar = sibling
                            break
                        end
                    end
                end
            end
           
            -- Botões de decisão
            if objClass == "TextButton" then
                local text = obj.Text
                if text == "Catch" then
                    self.elements.CatchButton = obj
                    self.elements.HasDecision = true
                elseif text == "Release" then
                    self.elements.ReleaseButton = obj
                    self.elements.HasDecision = true
                end
            end
           
            -- Detecta Minigame 2
            if objClass == "TextLabel" then
                local text = obj.Text
                if text and (string.find(text, "Success rate") or string.find(text, "success")) then
                    self.elements.Minigame2Active = true
                end
            end
        end
    end)
   
    return self.elements
end
-- ========================================
-- ESTADO GLOBAL
-- ========================================
local GameState = {
    active = false,
    phase = "idle",
    hasCast = false,
    hasDecided = false,
    lastAction = 0,
    lastClick = 0,
    barPosition = nil,
   
    -- Estatísticas
    stats = {
        fish = 0,
        casts = 0,
        mg1Success = 0,
        mg2Success = 0,
        clicks = 0,
        perfectCatches = 0,
        streak = 0,
        bestStreak = 0
    }
}
local statsChanged = false
-- ========================================
-- LÓGICA PRINCIPAL DO AUTOFISH
-- ========================================
local AutoFish = {}
function AutoFish:Cast()
    if GameState.hasCast then return end
   
    pcall(function()
        RemoteEvent:FireServer("Throw", 1)
    end)
   
    GameState.hasCast = true
    GameState.phase = "waiting"
    GameState.lastAction = tick()
    GameState.stats.casts = GameState.stats.casts + 1
    statsChanged = true
end
function AutoFish:ProcessCircular(indicator, circle)
    if not (indicator and circle) then return end
   
    local now = tick()
    if now - GameState.lastClick < CONFIG.CIRCLE_CLICK_DELAY then return end
   
    local circleSize = circle.AbsoluteSize.X
    local buttonSize = indicator.AbsoluteSize.X
    local ratio = circleSize / buttonSize
   
    local inRange = ratio >= CONFIG.CIRCLE_MIN_RATIO and ratio <= CONFIG.CIRCLE_MAX_RATIO
    local isPerfect = math.abs(ratio - CONFIG.CIRCLE_SWEET_SPOT) < 0.05
   
    if inRange then
        InputManager:ClickElement(indicator)
       
        GameState.lastClick = now
        GameState.lastAction = now
        GameState.phase = "minigame1"
        GameState.stats.clicks = GameState.stats.clicks + 1
        GameState.stats.mg1Success = GameState.stats.mg1Success + 1
        statsChanged = true
       
        if isPerfect then
            GameState.stats.perfectCatches = GameState.stats.perfectCatches + 1
            statsChanged = true
        end
    end
end
function AutoFish:ProcessBar(target, redBar)
    if not (target and redBar) then return end
   
    local now = tick()
    if now - GameState.lastClick < CONFIG.BAR_CLICK_DELAY then return end
   
    local targetLeft = target.AbsolutePosition.X
    local targetRight = targetLeft + target.AbsoluteSize.X
    local targetCenter = targetLeft + target.AbsoluteSize.X * 0.5

    local redBarLeft = redBar.AbsolutePosition.X
    local redBarRight = redBarLeft + redBar.AbsoluteSize.X
    local barCenter = redBarLeft + redBar.AbsoluteSize.X * 0.5

    local isCompletelyInside = redBarLeft >= targetLeft and redBarRight <= targetRight
    local distance = math.abs(targetCenter - barCenter)
    local threshold = target.AbsoluteSize.X * CONFIG.BAR_CENTER_THRESHOLD
   
    local isMoving = false
    if GameState.barPosition then
        isMoving = math.abs(barCenter - GameState.barPosition) > CONFIG.BAR_MOVEMENT_MIN
    end
    GameState.barPosition = barCenter
   
    if isCompletelyInside and distance <= threshold and (isMoving or not GameState.barPosition) then
        InputManager:ClickCenter()
       
        GameState.lastClick = now
        GameState.lastAction = now
        GameState.phase = "minigame2"
        GameState.stats.clicks = GameState.stats.clicks + 1
        GameState.stats.mg2Success = GameState.stats.mg2Success + 1
        statsChanged = true
    end
end
function AutoFish:ProcessDecision(catchBtn, releaseBtn)
    if GameState.hasDecided then return end
   
    task.wait(0.5)
   
    -- Sempre pega o peixe
    if catchBtn then
        InputManager:ClickElement(catchBtn)
       
        GameState.stats.fish = GameState.stats.fish + 1
        GameState.stats.streak = GameState.stats.streak + 1
       
        if GameState.stats.streak > GameState.stats.bestStreak then
            GameState.stats.bestStreak = GameState.stats.streak
        end
        statsChanged = true
    end
   
    GameState.hasDecided = true
    GameState.phase = "decided"
   
    task.wait(CONFIG.RESET_DELAY)
    self:Reset()
end
function AutoFish:HandleOldSystem()
    -- Sistema antigo (fallback)
    if not GameState.hasDecided and GameState.phase == "minigame2" then
        task.wait(CONFIG.DECISION_DELAY)
       
        pcall(function()
            RemoteEvent:FireServer("FishDecision", true)
        end)
       
        GameState.hasDecided = true
        GameState.stats.fish = GameState.stats.fish + 1
        GameState.stats.streak = GameState.stats.streak + 1
        if GameState.stats.streak > GameState.stats.bestStreak then
            GameState.stats.bestStreak = GameState.stats.streak
        end
        statsChanged = true
       
        task.wait(CONFIG.RESET_DELAY)
        self:Reset()
    end
end
function AutoFish:Reset()
    GameState.hasCast = false
    GameState.hasDecided = false
    GameState.phase = "idle"
    GameState.barPosition = nil
end
function AutoFish:Process()
    if not GameState.active then return end
   
    local elements = ElementCache:FastUpdate()
    local now = tick()
   
    -- Timeout check
    if now - GameState.lastAction > CONFIG.ACTION_TIMEOUT then
        self:Reset()
        GameState.lastAction = now
        return
    end
   
    -- Phase 1: Cast
    if not elements.CircularIndicator and not elements.Target and
       not elements.HasDecision and not GameState.hasCast then
        return self:Cast()
    end
   
    -- Phase 2: Circular Minigame
    if elements.CircularIndicator and elements.AnimatedCircle then
        return self:ProcessCircular(elements.CircularIndicator, elements.AnimatedCircle)
    end
   
    -- Phase 3: Bar Minigame
    if elements.Target and elements.RedBar and elements.Minigame2Active then
        return self:ProcessBar(elements.Target, elements.RedBar)
    end
   
    -- Phase 4: Catch/Release Decision
    if elements.HasDecision and (elements.CatchButton or elements.ReleaseButton) then
        return self:ProcessDecision(elements.CatchButton, elements.ReleaseButton)
    end
   
    -- Phase 5: Old System Fallback
    self:HandleOldSystem()
end
-- ========================================
-- UI MODERNA E RESPONSIVA
-- ========================================
local UIManager = {}
function UIManager:Create()
    -- ScreenGui
    local gui = Instance.new("ScreenGui")
    gui.Name = "AutoFishPro"
    gui.ResetOnSpawn = false
    gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    gui.Parent = playerGui
   
    -- Main Frame
    local main = Instance.new("Frame")
    main.Size = UDim2.new(0, 380 * CONFIG.UI_SCALE, 0, 480 * CONFIG.UI_SCALE)
    main.Position = UDim2.new(0.5, -190 * CONFIG.UI_SCALE, 0.5, -240 * CONFIG.UI_SCALE)
    main.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
    main.BorderSizePixel = 0
    main.Parent = gui
   
    local gradient = Instance.new("UIGradient")
    gradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(25, 25, 30)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(15, 15, 20))
    })
    gradient.Rotation = 90
    gradient.Parent = main
   
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 16)
    corner.Parent = main
   
    local stroke = Instance.new("UIStroke")
    stroke.Color = Color3.fromRGB(52, 152, 219)
    stroke.Thickness = 2.5
    stroke.Transparency = 0.5
    stroke.Parent = main
   
    -- Header
    local header = Instance.new("Frame")
    header.Size = UDim2.new(1, 0, 0, 55)
    header.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
    header.BorderSizePixel = 0
    header.Parent = main
   
    local headerCorner = Instance.new("UICorner")
    headerCorner.CornerRadius = UDim.new(0, 16)
    headerCorner.Parent = header
   
    -- Title
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -120, 1, 0)
    title.Position = UDim2.new(0, 15, 0, 0)
    title.BackgroundTransparency = 1
    title.Text = "🎣 AutoFish Pro - v2"
    title.TextColor3 = Color3.fromRGB(255, 255, 255)
    title.TextSize = 20
    title.Font = Enum.Font.GothamBold
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = header
   
    -- Platform Badge
    local badge = Instance.new("Frame")
    badge.Size = UDim2.new(0, 70, 0, 28)
    badge.Position = UDim2.new(1, -120, 0.5, -14)
    badge.BackgroundColor3 = Color3.fromRGB(46, 204, 113)
    badge.BorderSizePixel = 0
    badge.Parent = header
   
    local badgeCorner = Instance.new("UICorner")
    badgeCorner.CornerRadius = UDim.new(0, 8)
    badgeCorner.Parent = badge
   
    local badgeText = Instance.new("TextLabel")
    badgeText.Size = UDim2.new(1, 0, 1, 0)
    badgeText.BackgroundTransparency = 1
    badgeText.Text = "💻 PC"
    badgeText.TextColor3 = Color3.fromRGB(255, 255, 255)
    badgeText.TextSize = 12
    badgeText.Font = Enum.Font.GothamBold
    badgeText.Parent = badge
   
    -- Close Button
    local closeBtn = Instance.new("TextButton")
    closeBtn.Size = UDim2.new(0, 35, 0, 35)
    closeBtn.Position = UDim2.new(1, -45, 0.5, -17.5)
    closeBtn.BackgroundColor3 = Color3.fromRGB(231, 76, 60)
    closeBtn.Text = "✖"
    closeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    closeBtn.TextSize = 18
    closeBtn.Font = Enum.Font.GothamBold
    closeBtn.BorderSizePixel = 0
    closeBtn.Parent = header
   
    local closeCorner = Instance.new("UICorner")
    closeCorner.CornerRadius = UDim.new(0, 8)
    closeCorner.Parent = closeBtn
   
    -- Content
    local content = Instance.new("ScrollingFrame")
    content.Size = UDim2.new(1, -20, 1, -65)
    content.Position = UDim2.new(0, 10, 0, 60)
    content.BackgroundTransparency = 1
    content.BorderSizePixel = 0
    content.ScrollBarThickness = 3
    content.ScrollBarImageColor3 = Color3.fromRGB(52, 152, 219)
    content.CanvasSize = UDim2.new(0, 0, 0, 600)
    content.Parent = main
   
    -- Status Card
    local statusCard = Instance.new("Frame")
    statusCard.Size = UDim2.new(1, 0, 0, 80)
    statusCard.Position = UDim2.new(0, 0, 0, 10)
    statusCard.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
    statusCard.BorderSizePixel = 0
    statusCard.Parent = content
   
    local statusCorner = Instance.new("UICorner")
    statusCorner.CornerRadius = UDim.new(0, 12)
    statusCorner.Parent = statusCard
   
    local statusDot = Instance.new("Frame")
    statusDot.Size = UDim2.new(0, 14, 0, 14)
    statusDot.Position = UDim2.new(0, 20, 0, 20)
    statusDot.BackgroundColor3 = Color3.fromRGB(231, 76, 60)
    statusDot.BorderSizePixel = 0
    statusDot.Parent = statusCard
   
    local dotCorner = Instance.new("UICorner")
    dotCorner.CornerRadius = UDim.new(1, 0)
    dotCorner.Parent = statusDot
   
    local statusText = Instance.new("TextLabel")
    statusText.Size = UDim2.new(1, -80, 0, 30)
    statusText.Position = UDim2.new(0, 45, 0, 15)
    statusText.BackgroundTransparency = 1
    statusText.Text = "Sistema Desativado"
    statusText.TextColor3 = Color3.fromRGB(255, 255, 255)
    statusText.TextSize = 16
    statusText.Font = Enum.Font.GothamBold
    statusText.TextXAlignment = Enum.TextXAlignment.Left
    statusText.Parent = statusCard
   
    local statusInfo = Instance.new("TextLabel")
    statusInfo.Size = UDim2.new(1, -40, 0, 25)
    statusInfo.Position = UDim2.new(0, 20, 0, 45)
    statusInfo.BackgroundTransparency = 1
    statusInfo.Text = "Clique para iniciar"
    statusInfo.TextColor3 = Color3.fromRGB(150, 150, 150)
    statusInfo.TextSize = 12
    statusInfo.Font = Enum.Font.Gotham
    statusInfo.TextXAlignment = Enum.TextXAlignment.Left
    statusInfo.Parent = statusCard
   
    -- Main Button
    local mainBtn = Instance.new("TextButton")
    mainBtn.Size = UDim2.new(1, 0, 0, 60)
    mainBtn.Position = UDim2.new(0, 0, 0, 100)
    mainBtn.BackgroundColor3 = Color3.fromRGB(46, 204, 113)
    mainBtn.Text = "▶ INICIAR AUTOFISH"
    mainBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    mainBtn.TextSize = 18
    mainBtn.Font = Enum.Font.GothamBold
    mainBtn.BorderSizePixel = 0
    mainBtn.Parent = content
   
    local mainCorner = Instance.new("UICorner")
    mainCorner.CornerRadius = UDim.new(0, 12)
    mainCorner.Parent = mainBtn
   
    -- Stats Container
    local statsCard = Instance.new("Frame")
    statsCard.Size = UDim2.new(1, 0, 0, 280)
    statsCard.Position = UDim2.new(0, 0, 0, 170)
    statsCard.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
    statsCard.BorderSizePixel = 0
    statsCard.Parent = content
   
    local statsCorner = Instance.new("UICorner")
    statsCorner.CornerRadius = UDim.new(0, 12)
    statsCorner.Parent = statsCard
   
    local statsTitle = Instance.new("TextLabel")
    statsTitle.Size = UDim2.new(1, -20, 0, 35)
    statsTitle.Position = UDim2.new(0, 10, 0, 10)
    statsTitle.BackgroundTransparency = 1
    statsTitle.Text = "📊 Estatísticas da Sessão"
    statsTitle.TextColor3 = Color3.fromRGB(52, 152, 219)
    statsTitle.TextSize = 15
    statsTitle.Font = Enum.Font.GothamBold
    statsTitle.TextXAlignment = Enum.TextXAlignment.Left
    statsTitle.Parent = statsCard
   
    -- Stats Items
    local statItems = {}
    local statConfig = {
        {"🐟 Peixes", "fish"},
        {"🎯 Lançamentos", "casts"},
        {"⭕ MG1 Acertos", "mg1Success"},
        {"📊 MG2 Acertos", "mg2Success"},
        {"🖱️ Total Clicks", "clicks"},
        {"⚡ Perfect", "perfectCatches"},
        {"🔥 Sequência", "streak"},
        {"👑 Recorde", "bestStreak"}
    }
   
    for i, data in ipairs(statConfig) do
        local row = math.floor((i - 1) / 2)
        local col = (i - 1) % 2
       
        local item = Instance.new("Frame")
        item.Size = UDim2.new(0.5, -15, 0, 50)
        item.Position = UDim2.new(0.5 * col, 10, 0, 50 + row * 55)
        item.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
        item.BorderSizePixel = 0
        item.Parent = statsCard
       
        local itemCorner = Instance.new("UICorner")
        itemCorner.CornerRadius = UDim.new(0, 8)
        itemCorner.Parent = item
       
        local itemText = Instance.new("TextLabel")
        itemText.Size = UDim2.new(0.6, -10, 1, 0)
        itemText.Position = UDim2.new(0, 10, 0, 0)
        itemText.BackgroundTransparency = 1
        itemText.Text = data[1]
        itemText.TextColor3 = Color3.fromRGB(180, 180, 180)
        itemText.TextSize = 12
        itemText.Font = Enum.Font.Gotham
        itemText.TextXAlignment = Enum.TextXAlignment.Left
        itemText.Parent = item
       
        local itemValue = Instance.new("TextLabel")
        itemValue.Size = UDim2.new(0.4, -10, 1, 0)
        itemValue.Position = UDim2.new(0.6, 0, 0, 0)
        itemValue.BackgroundTransparency = 1
        itemValue.Text = "0"
        itemValue.TextColor3 = Color3.fromRGB(255, 255, 255)
        itemValue.TextSize = 20
        itemValue.Font = Enum.Font.GothamBold
        itemValue.TextXAlignment = Enum.TextXAlignment.Right
        itemValue.Parent = item
       
        statItems[data[2]] = itemValue
    end
   
    -- Reset Button
    local resetBtn = Instance.new("TextButton")
    resetBtn.Size = UDim2.new(1, 0, 0, 45)
    resetBtn.Position = UDim2.new(0, 0, 0, 460)
    resetBtn.BackgroundColor3 = Color3.fromRGB(41, 41, 46)
    resetBtn.Text = "🔄 Resetar Estatísticas"
    resetBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
    resetBtn.TextSize = 14
    resetBtn.Font = Enum.Font.Gotham
    resetBtn.BorderSizePixel = 0
    resetBtn.Parent = content
   
    local resetCorner = Instance.new("UICorner")
    resetCorner.CornerRadius = UDim.new(0, 10)
    resetCorner.Parent = resetBtn
   
    -- Functions
    local updateStats = function()
        for key, label in pairs(statItems) do
            label.Text = tostring(GameState.stats[key] or 0)
        end
    end
   
    -- Main Button Handler
    mainBtn.MouseButton1Click:Connect(function()
        GameState.active = not GameState.active
       
        if GameState.active then
            -- Start
            mainBtn.Text = "■ PARAR AUTOFISH"
            mainBtn.BackgroundColor3 = Color3.fromRGB(231, 76, 60)
            statusDot.BackgroundColor3 = Color3.fromRGB(46, 204, 113)
            statusText.Text = "Sistema Ativo"
            statusInfo.Text = "Pescando automaticamente..."
           
            AutoFish:Reset()
           
            spawn(function()
                while GameState.active do
                    AutoFish:Process()
                    if statsChanged then
                        updateStats()
                        statsChanged = false
                    end
                    task.wait(CONFIG.CHECK_INTERVAL)
                end
            end)
           
        else
            -- Stop
            mainBtn.Text = "▶ INICIAR AUTOFISH"
            mainBtn.BackgroundColor3 = Color3.fromRGB(46, 204, 113)
            statusDot.BackgroundColor3 = Color3.fromRGB(231, 76, 60)
            statusText.Text = "Sistema Desativado"
            statusInfo.Text = "Clique para iniciar"
           
        end
    end)
   
    -- Reset Stats
    resetBtn.MouseButton1Click:Connect(function()
        for key in pairs(GameState.stats) do
            GameState.stats[key] = 0
        end
        updateStats()
    end)
   
    -- Close Button
    closeBtn.MouseButton1Click:Connect(function()
        GameState.active = false
        gui:Destroy()
    end)
   
    -- Dragging
    local dragging, dragStart, startPos
   
    header.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or
           input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = main.Position
           
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)
   
    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or
                        input.UserInputType == Enum.UserInputType.Touch) then
            local delta = input.Position - dragStart
            main.Position = UDim2.new(
                startPos.X.Scale,
                startPos.X.Offset + delta.X,
                startPos.Y.Scale,
                startPos.Y.Offset + delta.Y
            )
        end
    end)
   
    -- Animação de entrada
    main.Position = UDim2.new(0.5, -190 * CONFIG.UI_SCALE, 1.5, 0)
    TweenService:Create(main, TweenInfo.new(0.5, Enum.EasingStyle.Back), {
        Position = UDim2.new(0.5, -190 * CONFIG.UI_SCALE, 0.5, -240 * CONFIG.UI_SCALE)
    }):Play()
   
    return gui
end
-- ========================================
-- INICIALIZAÇÃO
-- ========================================
-- Inicializa sistemas
InputManager:Initialize()
-- Cria UI
UIManager:Create()
