-- AutoFish Pro - Ultra Otimizado com Precisão Máxima
local VIM = game:GetService('VirtualInputManager')
local UserInputService = game:GetService('UserInputService')
local ReplicatedStorage = game:GetService('ReplicatedStorage')
local RunService = game:GetService('RunService')
local Players = game:GetService('Players')
local TweenService = game:GetService('TweenService')

local player = Players.LocalPlayer
local playerGui = player:WaitForChild('PlayerGui')
local RemoteEvent = ReplicatedStorage:WaitForChild('Modules'):WaitForChild('Events'):WaitForChild('RemoteEvent')

-- Constantes Ultra Otimizadas
local CACHE_INTERVAL = 0.01
local CHECK_INTERVAL = 0.01
local CLICK_DELAY = 0.25
local MIN_CLICK_DELAY_MG2 = 0.55
local MIN_SIZE_RATIO = 0.88
local MAX_SIZE_RATIO = 0.99
local MAX_DISTANCE_RATIO = 0.22
local ACTION_TIMEOUT = 50
local FISH_DECISION_DELAY = 1.4
local RESET_DELAY = 1.8
local MIN_BAR_MOVEMENT = 3
local CLICK_HOLD_TIME = 0.04

-- Variáveis Globais
local fishingActive = false
local fishingConnection = nil
local hasCastOnce = false
local hasExecutedFishDecision = false
local minigameActive = false
local lastClickTime = 0
local lastCheckTime = 0
local lastCacheUpdate = 0
local lastActionTime = 0
local previousRedBarPos = nil
local minigame2ClickCount = 0
local cachedDescendants = {}

-- Cache otimizado
local cache = {
    CircularIndicator = nil,
    AnimatedCircle = nil,
    Target = nil,
    RedBar = nil,
    Minigame2Active = false,
    PlayerGui = nil
}

-- Stats
local stats = {
    totalFish = 0,
    successfulCasts = 0,
    failedCasts = 0,
    minigame1Success = 0,
    minigame2Success = 0,
    totalClicks = 0,
    sessionTime = 0,
    startTime = 0,
}

-- UI Setup Pré-construído
local screenGui = Instance.new('ScreenGui')
screenGui.Name = 'AutoFishProGui'
screenGui.ResetOnSpawn = false
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
screenGui.Parent = playerGui

local mainFrame = Instance.new('Frame')
mainFrame.Name = 'MainFrame'
mainFrame.Size = UDim2.new(0, 350, 0, 450)
mainFrame.Position = UDim2.new(0.5, -175, 0.5, -225)
mainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
mainFrame.BorderSizePixel = 0
mainFrame.ClipsDescendants = true
mainFrame.Parent = screenGui

local mainCorner = Instance.new('UICorner')
mainCorner.CornerRadius = UDim.new(0, 12)
mainCorner.Parent = mainFrame

local border = Instance.new('UIStroke')
border.Color = Color3.fromRGB(52, 152, 219)
border.Thickness = 2
border.Transparency = 0.3
border.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
border.Parent = mainFrame

local header = Instance.new('Frame')
header.Name = 'Header'
header.Size = UDim2.new(1, 0, 0, 45)
header.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
header.BorderSizePixel = 0
header.Parent = mainFrame

local headerCorner = Instance.new('UICorner')
headerCorner.CornerRadius = UDim.new(0, 12)
headerCorner.Parent = header

local headerFix = Instance.new('Frame')
headerFix.Size = UDim2.new(1, 0, 0, 10)
headerFix.Position = UDim2.new(0, 0, 1, -10)
headerFix.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
headerFix.BorderSizePixel = 0
headerFix.Parent = header

local icon = Instance.new('ImageLabel')
icon.Size = UDim2.new(0, 30, 0, 30)
icon.Position = UDim2.new(0, 10, 0, 7.5)
icon.BackgroundColor3 = Color3.fromRGB(52, 152, 219)
icon.BorderSizePixel = 0
icon.Parent = header

local iconCorner = Instance.new('UICorner')
iconCorner.CornerRadius = UDim.new(0, 8)
iconCorner.Parent = icon

local iconText = Instance.new('TextLabel')
iconText.Size = UDim2.new(1, 0, 1, 0)
iconText.BackgroundTransparency = 1
iconText.Text = '🎣'
iconText.TextColor3 = Color3.fromRGB(255, 255, 255)
iconText.TextSize = 20
iconText.Font = Enum.Font.GothamBold
iconText.Parent = icon

local title = Instance.new('TextLabel')
title.Size = UDim2.new(1, -100, 1, 0)
title.Position = UDim2.new(0, 45, 0, 0)
title.BackgroundTransparency = 1
title.Text = 'AutoFish Pro (PC)'
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.TextSize = 16
title.Font = Enum.Font.GothamBold
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = header

local closeButton = Instance.new('TextButton')
closeButton.Size = UDim2.new(0, 30, 0, 30)
closeButton.Position = UDim2.new(1, -37.5, 0, 7.5)
closeButton.BackgroundColor3 = Color3.fromRGB(35, 35, 40)
closeButton.BorderSizePixel = 0
closeButton.Text = '×'
closeButton.TextColor3 = Color3.fromRGB(200, 200, 200)
closeButton.TextSize = 20
closeButton.Font = Enum.Font.GothamBold
closeButton.AutoButtonColor = false
closeButton.Parent = header

local closeCorner = Instance.new('UICorner')
closeCorner.CornerRadius = UDim.new(0, 8)
closeCorner.Parent = closeButton

local scrollFrame = Instance.new('ScrollingFrame')
scrollFrame.Size = UDim2.new(1, -20, 1, -55)
scrollFrame.Position = UDim2.new(0, 10, 0, 50)
scrollFrame.BackgroundTransparency = 1
scrollFrame.BorderSizePixel = 0
scrollFrame.ScrollBarThickness = 6
scrollFrame.ScrollBarImageColor3 = Color3.fromRGB(52, 152, 219)
scrollFrame.CanvasSize = UDim2.new(0, 0, 0, 600)
scrollFrame.Parent = mainFrame

local statusTitle = Instance.new('TextLabel')
statusTitle.Size = UDim2.new(1, 0, 0, 25)
statusTitle.Position = UDim2.new(0, 0, 0, 0)
statusTitle.BackgroundTransparency = 1
statusTitle.Text = '📊 Status'
statusTitle.TextColor3 = Color3.fromRGB(52, 152, 219)
statusTitle.TextSize = 13
statusTitle.Font = Enum.Font.GothamBold
statusTitle.TextXAlignment = Enum.TextXAlignment.Left
statusTitle.Parent = scrollFrame

local statusBox = Instance.new('Frame')
statusBox.Size = UDim2.new(1, 0, 0, 50)
statusBox.Position = UDim2.new(0, 0, 0, 30)
statusBox.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
statusBox.BorderSizePixel = 0
statusBox.Parent = scrollFrame

local statusCorner = Instance.new('UICorner')
statusCorner.CornerRadius = UDim.new(0, 8)
statusCorner.Parent = statusBox

local statusLabel = Instance.new('TextLabel')
statusLabel.Size = UDim2.new(1, -20, 1, 0)
statusLabel.Position = UDim2.new(0, 10, 0, 0)
statusLabel.BackgroundTransparency = 1
statusLabel.Text = '🔴 Desativado'
statusLabel.TextColor3 = Color3.fromRGB(231, 76, 60)
statusLabel.TextSize = 14
statusLabel.Font = Enum.Font.GothamBold
statusLabel.TextYAlignment = Enum.TextYAlignment.Center
statusLabel.Parent = statusBox

local mainButton = Instance.new('TextButton')
mainButton.Size = UDim2.new(1, 0, 0, 55)
mainButton.Position = UDim2.new(0, 0, 0, 90)
mainButton.BackgroundColor3 = Color3.fromRGB(46, 204, 113)
mainButton.BorderSizePixel = 0
mainButton.Text = ''
mainButton.AutoButtonColor = false
mainButton.Parent = scrollFrame

local mainButtonCorner = Instance.new('UICorner')
mainButtonCorner.CornerRadius = UDim.new(0, 8)
mainButtonCorner.Parent = mainButton

local mainButtonLabel = Instance.new('TextLabel')
mainButtonLabel.Size = UDim2.new(1, 0, 1, 0)
mainButtonLabel.BackgroundTransparency = 1
mainButtonLabel.Text = '▶ INICIAR AUTOFISH'
mainButtonLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
mainButtonLabel.TextSize = 16
mainButtonLabel.Font = Enum.Font.GothamBold
mainButtonLabel.Parent = mainButton

local statsTitle = Instance.new('TextLabel')
statsTitle.Size = UDim2.new(1, 0, 0, 25)
statsTitle.Position = UDim2.new(0, 0, 0, 155)
statsTitle.BackgroundTransparency = 1
statsTitle.Text = '📈 Estatísticas'
statsTitle.TextColor3 = Color3.fromRGB(52, 152, 219)
statsTitle.TextSize = 13
statsTitle.Font = Enum.Font.GothamBold
statsTitle.TextXAlignment = Enum.TextXAlignment.Left
statsTitle.Parent = scrollFrame

local statsContainer = Instance.new('Frame')
statsContainer.Size = UDim2.new(1, 0, 0, 200)
statsContainer.Position = UDim2.new(0, 0, 0, 185)
statsContainer.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
statsContainer.BorderSizePixel = 0
statsContainer.Parent = scrollFrame

local statsContainerCorner = Instance.new('UICorner')
statsContainerCorner.CornerRadius = UDim.new(0, 8)
statsContainerCorner.Parent = statsContainer

local function createStatRow(parent, yPos, icon, label, valueKey)
    local row = Instance.new('Frame')
    row.Size = UDim2.new(1, -20, 0, 30)
    row.Position = UDim2.new(0, 10, 0, yPos)
    row.BackgroundTransparency = 1
    row.Parent = parent

    local iconLabel = Instance.new('TextLabel')
    iconLabel.Size = UDim2.new(0, 30, 1, 0)
    iconLabel.BackgroundTransparency = 1
    iconLabel.Text = icon
    iconLabel.TextColor3 = Color3.fromRGB(52, 152, 219)
    iconLabel.TextSize = 14
    iconLabel.Font = Enum.Font.GothamBold
    iconLabel.Parent = row

    local textLabel = Instance.new('TextLabel')
    textLabel.Size = UDim2.new(0.6, -30, 1, 0)
    textLabel.Position = UDim2.new(0, 30, 0, 0)
    textLabel.BackgroundTransparency = 1
    textLabel.Text = label
    textLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    textLabel.TextSize = 12
    textLabel.Font = Enum.Font.Gotham
    textLabel.TextXAlignment = Enum.TextXAlignment.Left
    textLabel.Parent = row

    local valueLabel = Instance.new('TextLabel')
    valueLabel.Size = UDim2.new(0.4, 0, 1, 0)
    valueLabel.Position = UDim2.new(0.6, 0, 0, 0)
    valueLabel.BackgroundTransparency = 1
    valueLabel.Text = '0'
    valueLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    valueLabel.TextSize = 13
    valueLabel.Font = Enum.Font.GothamBold
    valueLabel.TextXAlignment = Enum.TextXAlignment.Right
    valueLabel.Parent = row

    return valueLabel
end

local fishLabel = createStatRow(statsContainer, 10, '🐟', 'Peixes Capturados', 'totalFish')
local castsLabel = createStatRow(statsContainer, 45, '🎯', 'Lançamentos', 'successfulCasts')
local clicksLabel = createStatRow(statsContainer, 80, '🖱️', 'Total de Clicks', 'totalClicks')
local minigame1Label = createStatRow(statsContainer, 115, '⭕', 'Acertos MG1', 'minigame1Success')
local minigame2Label = createStatRow(statsContainer, 150, '🎪', 'Acertos MG2', 'minigame2Success')

local resetButton = Instance.new('TextButton')
resetButton.Size = UDim2.new(1, 0, 0, 40)
resetButton.Position = UDim2.new(0, 0, 0, 395)
resetButton.BackgroundColor3 = Color3.fromRGB(40, 40, 45)
resetButton.BorderSizePixel = 0
resetButton.Text = ''
resetButton.AutoButtonColor = false
resetButton.Parent = scrollFrame

local resetCorner = Instance.new('UICorner')
resetCorner.CornerRadius = UDim.new(0, 8)
resetCorner.Parent = resetButton

local resetLabel = Instance.new('TextLabel')
resetLabel.Size = UDim2.new(1, 0, 1, 0)
resetLabel.BackgroundTransparency = 1
resetLabel.Text = '🔄 Resetar Estatísticas'
resetLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
resetLabel.TextSize = 12
resetLabel.Font = Enum.Font.GothamBold
resetLabel.Parent = resetButton

-- Funções otimizadas para máxima precisão
local function updateStats()
    fishLabel.Text = tostring(stats.totalFish)
    castsLabel.Text = tostring(stats.successfulCasts)
    clicksLabel.Text = tostring(stats.totalClicks)
    minigame1Label.Text = tostring(stats.minigame1Success)
    minigame2Label.Text = tostring(stats.minigame2Success)
end

local function clickCenter()
    local cam = workspace.CurrentCamera
    local cx = cam.ViewportSize.X * 0.5
    local cy = cam.ViewportSize.Y * 0.5
    VIM:SendMouseButtonEvent(cx, cy, 0, true, game, 0)
    task.wait(CLICK_HOLD_TIME)
    VIM:SendMouseButtonEvent(cx, cy, 0, false, game, 0)
end

local function clickPosition(x, y)
    VIM:SendMouseButtonEvent(x, y, 0, true, game, 0)
    task.wait(CLICK_HOLD_TIME)
    VIM:SendMouseButtonEvent(x, y, 0, false, game, 0)
end

-- Cache Ultra Otimizado com Descendentes Pré-computados
local function updateElementCache()
    local currentTime = tick()
    if currentTime - lastCacheUpdate < CACHE_INTERVAL then
        return cache
    end

    cache.CircularIndicator = nil
    cache.AnimatedCircle = nil
    cache.Target = nil
    cache.RedBar = nil
    cache.Minigame2Active = false

    local localPlayerGui = cache.PlayerGui or game.Players.LocalPlayer:FindFirstChild('PlayerGui')
    if localPlayerGui then
        cache.PlayerGui = localPlayerGui
    else
        return cache
    end

    if #cachedDescendants == 0 then
        cachedDescendants = localPlayerGui:GetDescendants()
    end

    local descendCount = #cachedDescendants
    
    for i = 1, descendCount do
        local gui = cachedDescendants[i]
        
        if not gui.Parent then
            cachedDescendants[i] = nil
        elseif gui:IsA('ImageButton') and gui.Visible then
            if not cache.CircularIndicator then
                local hasAnimatedCircle = gui:FindFirstChild('AnimatedWhiteCircle')
                local hasMiddleCircle = gui:FindFirstChild('MiddleCircle')
                if hasAnimatedCircle and hasMiddleCircle then
                    cache.CircularIndicator = gui
                    cache.AnimatedCircle = hasAnimatedCircle
                end
            end
        elseif gui.Name == 'Target' and gui.Visible and not cache.Target then
            cache.Target = gui
            local parent = gui.Parent
            if parent then
                for _, child in pairs(parent:GetChildren()) do
                    if child:IsA('Frame') and child.ZIndex == 3 and child.Visible then
                        cache.RedBar = child
                        break
                    end
                end
            end
        elseif gui:IsA('TextLabel') and string.find(gui.Text, 'Success rate') then
            cache.Minigame2Active = true
        end

        if cache.CircularIndicator and cache.Target and cache.Minigame2Active then
            break
        end
    end

    cachedDescendants = {}
    lastCacheUpdate = currentTime
    return cache
end

-- Lógica Principal com Precisão Máxima
local function clickFishingElements()
    local currentTime = tick()
    if currentTime - lastCheckTime < CHECK_INTERVAL then
        return false
    end
    lastCheckTime = currentTime

    updateElementCache()

    if not cache.Minigame2Active then
        previousRedBarPos = nil
        minigame2ClickCount = 0
    end

    -- Fase 1: Lançamento
    if not cache.CircularIndicator and not cache.Target and not hasCastOnce then
        pcall(function()
            RemoteEvent:FireServer('Throw', 1)
        end)
        hasCastOnce = true
        lastClickTime = currentTime
        lastActionTime = currentTime
        stats.successfulCasts = stats.successfulCasts + 1
        updateStats()
        return true
    end

    -- Fase 2: Minigame 1 (Circular)
    if cache.CircularIndicator and currentTime - lastClickTime > CLICK_DELAY then
        pcall(function()
            local gui = cache.CircularIndicator
            local animatedCircle = cache.AnimatedCircle
            if animatedCircle then
                local circleSize = animatedCircle.AbsoluteSize.X
                local buttonSize = gui.AbsoluteSize.X
                local sizeRatio = circleSize / buttonSize

                if sizeRatio >= MIN_SIZE_RATIO and sizeRatio <= MAX_SIZE_RATIO then
                    local centerX = gui.AbsolutePosition.X + gui.AbsoluteSize.X * 0.5
                    local centerY = gui.AbsolutePosition.Y + gui.AbsoluteSize.Y * 0.5

                    clickPosition(centerX, centerY)

                    lastClickTime = currentTime
                    lastActionTime = currentTime
                    stats.totalClicks = stats.totalClicks + 1
                    stats.minigame1Success = stats.minigame1Success + 1
                    updateStats()
                end
            end
        end)
        return true
    end

    -- Fase 3: Minigame 2 (Barra)
    if cache.Target and cache.RedBar and cache.Minigame2Active then
        local gui = cache.Target
        local redBar = cache.RedBar

        local targetLeft = gui.AbsolutePosition.X
        local targetRight = targetLeft + gui.AbsoluteSize.X
        local targetCenter = targetLeft + gui.AbsoluteSize.X * 0.5

        local redBarLeft = redBar.AbsolutePosition.X
        local redBarRight = redBarLeft + redBar.AbsoluteSize.X
        local redBarCenter = redBarLeft + redBar.AbsoluteSize.X * 0.5

        local isCompletelyInside = redBarLeft >= targetLeft and redBarRight <= targetRight
        local distance = math.abs(redBarCenter - targetCenter)
        local maxDistance = gui.AbsoluteSize.X * MAX_DISTANCE_RATIO
        local timeSinceLastClick = currentTime - lastClickTime

        local isMoving = false
        if previousRedBarPos then
            isMoving = math.abs(redBarCenter - previousRedBarPos) > MIN_BAR_MOVEMENT
        end

        if isCompletelyInside and distance <= maxDistance and timeSinceLastClick > MIN_CLICK_DELAY_MG2 and (isMoving or not previousRedBarPos) then
            pcall(function()
                clickCenter()
                lastClickTime = currentTime
                lastActionTime = currentTime
                minigame2ClickCount = minigame2ClickCount + 1
                stats.totalClicks = stats.totalClicks + 1
                stats.minigame2Success = stats.minigame2Success + 1
                updateStats()
            end)
            return true
        end

        previousRedBarPos = redBarCenter
        minigameActive = true
        return false
    end

    -- Fase 4: Decisão do Peixe
    if minigameActive and not cache.Target and not cache.Minigame2Active then
        minigameActive = false
        previousRedBarPos = nil
        minigame2ClickCount = 0

        if not hasExecutedFishDecision then
            task.delay(FISH_DECISION_DELAY, function()
                if not fishingActive then return end
                pcall(function()
                    RemoteEvent:FireServer('FishDecision', true)
                end)
                hasExecutedFishDecision = true
                lastActionTime = tick()
                stats.totalFish = stats.totalFish + 1
                updateStats()

                task.delay(RESET_DELAY, function()
                    if fishingActive then
                        hasCastOnce = false
                        hasExecutedFishDecision = false
                        minigameActive = false
                        previousRedBarPos = nil
                        minigame2ClickCount = 0
                    end
                end)
            end)
        end
    end

    return false
end

function startAutoFishing()
    if fishingActive then return end
    fishingActive = true
    hasCastOnce = false
    hasExecutedFishDecision = false
    minigameActive = false
    previousRedBarPos = nil
    minigame2ClickCount = 0

    if stats.startTime == 0 then
        stats.startTime = tick()
    end

    statusLabel.Text = '🟢 Ativo - Pescando...'
    statusLabel.TextColor3 = Color3.fromRGB(46, 204, 113)
    statusBox.BackgroundColor3 = Color3.fromRGB(35, 50, 40)

    fishingConnection = RunService.Heartbeat:Connect(function()
        if fishingActive then
            clickFishingElements()
            if tick() - lastActionTime > ACTION_TIMEOUT then
                stopAutoFishing()
                task.delay(1, startAutoFishing)
            end
        end
    end)
end

function stopAutoFishing()
    if not fishingActive then return end
    fishingActive = false
    if fishingConnection then
        fishingConnection:Disconnect()
        fishingConnection = nil
    end
    hasCastOnce = false
    hasExecutedFishDecision = false
    minigameActive = false
    previousRedBarPos = nil
    minigame2ClickCount = 0

    statusLabel.Text = '🔴 Desativado'
    statusLabel.TextColor3 = Color3.fromRGB(231, 76, 60)
    statusBox.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
end

local mainButtonPressed = false

mainButton.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        mainButtonPressed = true
        if not fishingActive then
            TweenService:Create(mainButton, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(56, 214, 123)}):Play()
        end
    end
end)

mainButton.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        if mainButtonPressed then
            mainButtonPressed = false
            if fishingActive then
                stopAutoFishing()
                mainButtonLabel.Text = '▶ INICIAR AUTOFISH'
                TweenService:Create(mainButton, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(46, 204, 113)}):Play()
            else
                startAutoFishing()
                mainButtonLabel.Text = '■ PARAR AUTOFISH'
                TweenService:Create(mainButton, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(231, 76, 60)}):Play()
            end
        end
    end
end)

resetButton.MouseEnter:Connect(function()
    TweenService:Create(resetButton, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(50, 50, 55)}):Play()
end)

resetButton.MouseLeave:Connect(function()
    TweenService:Create(resetButton, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(40, 40, 45)}):Play()
end)

resetButton.MouseButton1Click:Connect(function()
    stats.totalFish = 0
    stats.successfulCasts = 0
    stats.failedCasts = 0
    stats.minigame1Success = 0
    stats.minigame2Success = 0
    stats.totalClicks = 0
    stats.sessionTime = 0
    stats.startTime = 0
    updateStats()
end)

closeButton.MouseEnter:Connect(function()
    TweenService:Create(closeButton, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(231, 76, 60)}):Play()
end)

closeButton.MouseLeave:Connect(function()
    TweenService:Create(closeButton, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(35, 35, 40)}):Play()
end)

closeButton.MouseButton1Click:Connect(function()
    stopAutoFishing()
    TweenService:Create(mainFrame, TweenInfo.new(0.2, Enum.EasingStyle.Back, Enum.EasingDirection.In), {Size = UDim2.new(0, 0, 0, 0)}):Play()
    task.wait(0.2)
    screenGui:Destroy()
end)

local draggingWindow = false
local dragStart
local startPos

local function updateDrag(input)
    local delta = input.Position - dragStart
    mainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
end

header.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        draggingWindow = true
        dragStart = input.Position
        startPos = mainFrame.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                draggingWindow = false
            end
        end)
    end
end)

header.InputChanged:Connect(function(input)
    if draggingWindow and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        updateDrag(input)
    end
end)

mainFrame.Size = UDim2.new(0, 0, 0, 0)
mainFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
TweenService:Create(mainFrame, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Size = UDim2.new(0, 350, 0, 450), Position = UDim2.new(0.5, -175, 0.5, -225)}):Play()

print('🎣 AutoFish Pro Ultra Otimizado carregado!')
