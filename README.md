-- language: Lua, target: Roblox (Phantom Forces), runtime: exploit executor
-- ESP + GUI configurável — painel draggable com toggles, sliders, minimizar

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer

-- ══════════════════════════════════════════
--  CONFIG
-- ══════════════════════════════════════════
local Config = {
    Enabled       = true,
    ShowBox       = true,
    ShowName      = true,
    ShowHealth    = true,
    ShowDistance   = true,
    ShowTracers   = true,
    TeamCheck     = true,
    MaxDistance    = 2000,
    EnemyColor    = Color3.fromRGB(255, 50, 50),
    TeamColor     = Color3.fromRGB(50, 255, 50),
    BoxThickness  = 1.4,
    TextSize      = 13,
}

-- ══════════════════════════════════════════
--  GUI LIBRARY
-- ══════════════════════════════════════════
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "PF_ESP_GUI"
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.ResetOnSpawn = false

-- parent seguro: gethui() > CoreGui > PlayerGui
pcall(function()
    if gethui then
        ScreenGui.Parent = gethui()
    elseif syn and syn.protect_gui then
        syn.protect_gui(ScreenGui)
        ScreenGui.Parent = game:GetService("CoreGui")
    else
        ScreenGui.Parent = game:GetService("CoreGui")
    end
end)

-- cores do tema
local Theme = {
    Background    = Color3.fromRGB(20, 20, 28),
    TopBar        = Color3.fromRGB(30, 30, 42),
    Section       = Color3.fromRGB(25, 25, 35),
    Element       = Color3.fromRGB(35, 35, 50),
    ElementHover  = Color3.fromRGB(45, 45, 65),
    Accent        = Color3.fromRGB(100, 120, 255),
    AccentDark    = Color3.fromRGB(70, 85, 180),
    Text          = Color3.fromRGB(220, 220, 230),
    TextDim       = Color3.fromRGB(140, 140, 160),
    ToggleOn      = Color3.fromRGB(80, 200, 120),
    ToggleOff     = Color3.fromRGB(70, 70, 90),
    SliderBG      = Color3.fromRGB(40, 40, 55),
    SliderFill    = Color3.fromRGB(100, 120, 255),
    Border        = Color3.fromRGB(50, 50, 70),
}

-- ── MAIN FRAME ──
local MainFrame = Instance.new("Frame")
MainFrame.Name = "Main"
MainFrame.Size = UDim2.new(0, 280, 0, 0) -- altura dinâmica
MainFrame.Position = UDim2.new(0.5, -140, 0.5, -200)
MainFrame.BackgroundColor3 = Theme.Background
MainFrame.BorderSizePixel = 0
MainFrame.ClipsDescendants = true
MainFrame.Parent = ScreenGui

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 8)
MainCorner.Parent = MainFrame

local MainStroke = Instance.new("UIStroke")
MainStroke.Color = Theme.Border
MainStroke.Thickness = 1
MainStroke.Parent = MainFrame

-- ── TOP BAR (drag + minimize) ──
local TopBar = Instance.new("Frame")
TopBar.Name = "TopBar"
TopBar.Size = UDim2.new(1, 0, 0, 32)
TopBar.BackgroundColor3 = Theme.TopBar
TopBar.BorderSizePixel = 0
TopBar.Parent = MainFrame

local TopCorner = Instance.new("UICorner")
TopCorner.CornerRadius = UDim.new(0, 8)
TopCorner.Parent = TopBar

-- patch: canto inferior do topbar reto
local TopPatch = Instance.new("Frame")
TopPatch.Size = UDim2.new(1, 0, 0, 10)
TopPatch.Position = UDim2.new(0, 0, 1, -10)
TopPatch.BackgroundColor3 = Theme.TopBar
TopPatch.BorderSizePixel = 0
TopPatch.Parent = TopBar

local TitleLabel = Instance.new("TextLabel")
TitleLabel.Size = UDim2.new(1, -60, 1, 0)
TitleLabel.Position = UDim2.new(0, 12, 0, 0)
TitleLabel.BackgroundTransparency = 1
TitleLabel.Text = "⚡ PF ESP"
TitleLabel.TextColor3 = Theme.Accent
TitleLabel.TextSize = 14
TitleLabel.Font = Enum.Font.GothamBold
TitleLabel.TextXAlignment = Enum.TextXAlignment.Left
TitleLabel.Parent = TopBar

local MinimizeBtn = Instance.new("TextButton")
MinimizeBtn.Size = UDim2.new(0, 28, 0, 22)
MinimizeBtn.Position = UDim2.new(1, -36, 0.5, -11)
MinimizeBtn.BackgroundColor3 = Theme.Element
MinimizeBtn.BorderSizePixel = 0
MinimizeBtn.Text = "—"
MinimizeBtn.TextColor3 = Theme.TextDim
MinimizeBtn.TextSize = 14
MinimizeBtn.Font = Enum.Font.GothamBold
MinimizeBtn.Parent = TopBar
Instance.new("UICorner", MinimizeBtn).CornerRadius = UDim.new(0, 4)

-- ── CONTENT CONTAINER ──
local Content = Instance.new("ScrollingFrame")
Content.Name = "Content"
Content.Size = UDim2.new(1, -16, 1, -40)
Content.Position = UDim2.new(0, 8, 0, 36)
Content.BackgroundTransparency = 1
Content.BorderSizePixel = 0
Content.ScrollBarThickness = 3
Content.ScrollBarImageColor3 = Theme.Accent
Content.CanvasSize = UDim2.new(0, 0, 0, 0)
Content.AutomaticCanvasSize = Enum.AutomaticSize.Y
Content.Parent = MainFrame

local ContentLayout = Instance.new("UIListLayout")
ContentLayout.SortOrder = Enum.SortOrder.LayoutOrder
ContentLayout.Padding = UDim.new(0, 4)
ContentLayout.Parent = Content

-- ══════════════════════════════════════════
--  GUI COMPONENTS
-- ══════════════════════════════════════════

local LayoutOrder = 0
local function NextOrder()
    LayoutOrder = LayoutOrder + 1
    return LayoutOrder
end

-- ── SECTION HEADER ──
local function AddSection(name)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, 0, 0, 28)
    frame.BackgroundTransparency = 1
    frame.LayoutOrder = NextOrder()
    frame.Parent = Content

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, 0, 1, 0)
    label.Position = UDim2.new(0, 4, 0, 0)
    label.BackgroundTransparency = 1
    label.Text = string.upper(name)
    label.TextColor3 = Theme.Accent
    label.TextSize = 11
    label.Font = Enum.Font.GothamBold
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = frame

    local line = Instance.new("Frame")
    line.Size = UDim2.new(1, -8, 0, 1)
    line.Position = UDim2.new(0, 4, 1, -1)
    line.BackgroundColor3 = Theme.Border
    line.BorderSizePixel = 0
    line.Parent = frame
end

-- ── TOGGLE ──
local function AddToggle(name, default, callback)
    local state = default

    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, 0, 0, 32)
    frame.BackgroundColor3 = Theme.Element
    frame.BorderSizePixel = 0
    frame.LayoutOrder = NextOrder()
    frame.Parent = Content
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 6)

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -60, 1, 0)
    label.Position = UDim2.new(0, 10, 0, 0)
    label.BackgroundTransparency = 1
    label.Text = name
    label.TextColor3 = Theme.Text
    label.TextSize = 12
    label.Font = Enum.Font.Gotham
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = frame

    local toggleBG = Instance.new("Frame")
    toggleBG.Size = UDim2.new(0, 36, 0, 18)
    toggleBG.Position = UDim2.new(1, -46, 0.5, -9)
    toggleBG.BackgroundColor3 = state and Theme.ToggleOn or Theme.ToggleOff
    toggleBG.BorderSizePixel = 0
    toggleBG.Parent = frame
    Instance.new("UICorner", toggleBG).CornerRadius = UDim.new(1, 0)

    local knob = Instance.new("Frame")
    knob.Size = UDim2.new(0, 14, 0, 14)
    knob.Position = state and UDim2.new(1, -16, 0.5, -7) or UDim2.new(0, 2, 0.5, -7)
    knob.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    knob.BorderSizePixel = 0
    knob.Parent = toggleBG
    Instance.new("UICorner", knob).CornerRadius = UDim.new(1, 0)

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, 0, 1, 0)
    btn.BackgroundTransparency = 1
    btn.Text = ""
    btn.Parent = frame

    local function UpdateVisual()
        local tweenInfo = TweenInfo.new(0.2, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
        TweenService:Create(toggleBG, tweenInfo, {
            BackgroundColor3 = state and Theme.ToggleOn or Theme.ToggleOff
        }):Play()
        TweenService:Create(knob, tweenInfo, {
            Position = state and UDim2.new(1, -16, 0.5, -7) or UDim2.new(0, 2, 0.5, -7)
        }):Play()
    end

    btn.MouseButton1Click:Connect(function()
        state = not state
        UpdateVisual()
        callback(state)
    end)

    return {
        Set = function(val)
            state = val
            UpdateVisual()
            callback(state)
        end,
        Get = function() return state end,
    }
end

-- ── SLIDER ──
local function AddSlider(name, min, max, default, callback)
    local value = default

    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, 0, 0, 46)
    frame.BackgroundColor3 = Theme.Element
    frame.BorderSizePixel = 0
    frame.LayoutOrder = NextOrder()
    frame.Parent = Content
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 6)

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -60, 0, 20)
    label.Position = UDim2.new(0, 10, 0, 2)
    label.BackgroundTransparency = 1
    label.Text = name
    label.TextColor3 = Theme.Text
    label.TextSize = 12
    label.Font = Enum.Font.Gotham
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = frame

    local valueLabel = Instance.new("TextLabel")
    valueLabel.Size = UDim2.new(0, 50, 0, 20)
    valueLabel.Position = UDim2.new(1, -56, 0, 2)
    valueLabel.BackgroundTransparency = 1
    valueLabel.Text = tostring(math.floor(value))
    valueLabel.TextColor3 = Theme.Accent
    valueLabel.TextSize = 12
    valueLabel.Font = Enum.Font.GothamBold
    valueLabel.TextXAlignment = Enum.TextXAlignment.Right
    valueLabel.Parent = frame

    local sliderBG = Instance.new("Frame")
    sliderBG.Size = UDim2.new(1, -20, 0, 6)
    sliderBG.Position = UDim2.new(0, 10, 0, 30)
    sliderBG.BackgroundColor3 = Theme.SliderBG
    sliderBG.BorderSizePixel = 0
    sliderBG.Parent = frame
    Instance.new("UICorner", sliderBG).CornerRadius = UDim.new(1, 0)

    local sliderFill = Instance.new("Frame")
    sliderFill.Size = UDim2.new((value - min) / (max - min), 0, 1, 0)
    sliderFill.BackgroundColor3 = Theme.SliderFill
    sliderFill.BorderSizePixel = 0
    sliderFill.Parent = sliderBG
    Instance.new("UICorner", sliderFill).CornerRadius = UDim.new(1, 0)

    local knob = Instance.new("Frame")
    knob.Size = UDim2.new(0, 12, 0, 12)
    knob.AnchorPoint = Vector2.new(0.5, 0.5)
    knob.Position = UDim2.new((value - min) / (max - min), 0, 0.5, 0)
    knob.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    knob.BorderSizePixel = 0
    knob.ZIndex = 2
    knob.Parent = sliderBG
    Instance.new("UICorner", knob).CornerRadius = UDim.new(1, 0)

    -- interação
    local dragging = false

    local inputBtn = Instance.new("TextButton")
    inputBtn.Size = UDim2.new(1, 0, 0, 20)
    inputBtn.Position = UDim2.new(0, 0, 0, 24)
    inputBtn.BackgroundTransparency = 1
    inputBtn.Text = ""
    inputBtn.Parent = frame

    local function Update(inputX)
        local absPos = sliderBG.AbsolutePosition.X
        local absSize = sliderBG.AbsoluteSize.X
        local rel = math.clamp((inputX - absPos) / absSize, 0, 1)
        value = math.floor(min + (max - min) * rel)
        sliderFill.Size = UDim2.new(rel, 0, 1, 0)
        knob.Position = UDim2.new(rel, 0, 0.5, 0)
        valueLabel.Text = tostring(value)
        callback(value)
    end

    inputBtn.MouseButton1Down:Connect(function()
        dragging = true
    end)

    UserInputService.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            Update(input.Position.X)
        end
    end)

    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
        end
    end)

    inputBtn.MouseButton1Click:Connect(function()
        local mouse = UserInputService:GetMouseLocation()
        Update(mouse.X)
    end)
end

-- ── COLOR PICKER (simplificado: preset grid) ──
local function AddColorPicker(name, default, callback)
    local presets = {
        Color3.fromRGB(255, 50, 50),    -- vermelho
        Color3.fromRGB(255, 120, 30),   -- laranja
        Color3.fromRGB(255, 220, 50),   -- amarelo
        Color3.fromRGB(80, 200, 120),   -- verde
        Color3.fromRGB(50, 180, 255),   -- azul claro
        Color3.fromRGB(100, 120, 255),  -- azul
        Color3.fromRGB(180, 80, 255),   -- roxo
        Color3.fromRGB(255, 80, 180),   -- rosa
        Color3.fromRGB(255, 255, 255),  -- branco
        Color3.fromRGB(200, 200, 200),  -- cinza claro
    }

    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, 0, 0, 52)
    frame.BackgroundColor3 = Theme.Element
    frame.BorderSizePixel = 0
    frame.LayoutOrder = NextOrder()
    frame.Parent = Content
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 6)

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -20, 0, 20)
    label.Position = UDim2.new(0, 10, 0, 2)
    label.BackgroundTransparency = 1
    label.Text = name
    label.TextColor3 = Theme.Text
    label.TextSize = 12
    label.Font = Enum.Font.Gotham
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = frame

    local selected = Instance.new("Frame")
    selected.Size = UDim2.new(0, 14, 0, 14)
    selected.Position = UDim2.new(1, -28, 0, 5)
    selected.BackgroundColor3 = default
    selected.BorderSizePixel = 0
    selected.Parent = frame
    Instance.new("UICorner", selected).CornerRadius = UDim.new(1, 0)
    local selStroke = Instance.new("UIStroke")
    selStroke.Color = Color3.fromRGB(255, 255, 255)
    selStroke.Thickness = 1.5
    selStroke.Parent = selected

    for i, color in ipairs(presets) do
        local swatch = Instance.new("TextButton")
        swatch.Size = UDim2.new(0, 20, 0, 16)
        swatch.Position = UDim2.new(0, 10 + (i - 1) * 24, 0, 28)
        swatch.BackgroundColor3 = color
        swatch.BorderSizePixel = 0
        swatch.Text = ""
        swatch.Parent = frame
        Instance.new("UICorner", swatch).CornerRadius = UDim.new(0, 4)

        swatch.MouseButton1Click:Connect(function()
            selected.BackgroundColor3 = color
            callback(color)
        end)
    end
end

-- ══════════════════════════════════════════
--  DRAGGING
-- ══════════════════════════════════════════
do
    local dragging, dragStart, startPos

    TopBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStart = input.Position
            startPos = MainFrame.Position
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            local delta = input.Position - dragStart
            MainFrame.Position = UDim2.new(
                startPos.X.Scale, startPos.X.Offset + delta.X,
                startPos.Y.Scale, startPos.Y.Offset + delta.Y
            )
        end
    end)

    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
        end
    end)
end

-- ══════════════════════════════════════════
--  MINIMIZE / EXPAND
-- ══════════════════════════════════════════
local expanded = true
local expandedHeight = 420

local function SetExpanded(val)
    expanded = val
    local targetSize = expanded
        and UDim2.new(0, 280, 0, expandedHeight)
        or UDim2.new(0, 280, 0, 32)

    TweenService:Create(MainFrame, TweenInfo.new(0.25, Enum.EasingStyle.Quad), {
        Size = targetSize
    }):Play()

    MinimizeBtn.Text = expanded and "—" or "+"
end

MinimizeBtn.MouseButton1Click:Connect(function()
    SetExpanded(not expanded)
end)

-- ══════════════════════════════════════════
--  POPULATE GUI
-- ══════════════════════════════════════════

AddSection("Geral")

AddToggle("ESP Ativo", Config.Enabled, function(v) Config.Enabled = v end)
AddToggle("Team Check (só inimigos)", Config.TeamCheck, function(v) Config.TeamCheck = v end)

AddSection("Elementos")

AddToggle("Boxes", Config.ShowBox, function(v) Config.ShowBox = v end)
AddToggle("Nomes", Config.ShowName, function(v) Config.ShowName = v end)
AddToggle("Barra de HP", Config.ShowHealth, function(v) Config.ShowHealth = v end)
AddToggle("Distância", Config.ShowDistance, function(v) Config.ShowDistance = v end)
AddToggle("Tracers", Config.ShowTracers, function(v) Config.ShowTracers = v end)

AddSection("Ajustes")

AddSlider("Distância Máx", 100, 5000, Config.MaxDistance, function(v) Config.MaxDistance = v end)
AddSlider("Tamanho Texto", 8, 24, Config.TextSize, function(v) Config.TextSize = v end)
AddSlider("Espessura Box", 1, 5, math.floor(Config.BoxThickness), function(v) Config.BoxThickness = v end)

AddSection("Cores")

AddColorPicker("Cor Inimigo", Config.EnemyColor, function(v) Config.EnemyColor = v end)
AddColorPicker("Cor Aliado", Config.TeamColor, function(v) Config.TeamColor = v end)

-- calcula altura real do conteúdo
task.defer(function()
    task.wait(0.1)
    local totalH = ContentLayout.AbsoluteContentSize.Y + 44
    expandedHeight = math.min(totalH, 500)
    MainFrame.Size = UDim2.new(0, 280, 0, expandedHeight)
end)

-- ══════════════════════════════════════════
--  ESP ENGINE (mesmo de antes)
-- ══════════════════════════════════════════
local ESPObjects = {}

local function CreateESP(player)
    local esp = {}
    esp.BoxLines = {}
    for i = 1, 4 do
        local line = Drawing.new("Line")
        line.Visible = false
        line.Thickness = Config.BoxThickness
        esp.BoxLines[i] = line
    end
    esp.Name = Drawing.new("Text")
    esp.Name.Center = true
    esp.Name.Outline = true
    esp.Name.Size = Config.TextSize
    esp.Name.Visible = false

    esp.HealthBG = Drawing.new("Line")
    esp.HealthBG.Thickness = 3
    esp.HealthBG.Color = Color3.fromRGB(0, 0, 0)
    esp.HealthBG.Visible = false

    esp.HealthFill = Drawing.new("Line")
    esp.HealthFill.Thickness = 1.5
    esp.HealthFill.Visible = false

    esp.Distance = Drawing.new("Text")
    esp.Distance.Center = true
    esp.Distance.Outline = true
    esp.Distance.Size = Config.TextSize - 1
    esp.Distance.Visible = false

    esp.Tracer = Drawing.new("Line")
    esp.Tracer.Thickness = 1
    esp.Tracer.Visible = false

    ESPObjects[player] = esp
end

local function RemoveESP(player)
    local esp = ESPObjects[player]
    if not esp then return end
    for _, line in ipairs(esp.BoxLines) do line:Remove() end
    esp.Name:Remove()
    esp.HealthBG:Remove()
    esp.HealthFill:Remove()
    esp.Distance:Remove()
    esp.Tracer:Remove()
    ESPObjects[player] = nil
end

local function HideESP(esp)
    for _, line in ipairs(esp.BoxLines) do line.Visible = false end
    esp.Name.Visible = false
    esp.HealthBG.Visible = false
    esp.HealthFill.Visible = false
    esp.Distance.Visible = false
    esp.Tracer.Visible = false
end

local function IsEnemy(player)
    if not Config.TeamCheck then return true end
    local myTeam = LocalPlayer.Team
    local theirTeam = player.Team
    if not myTeam or not theirTeam then return true end
    return myTeam ~= theirTeam
end

local function GetColor(player)
    return IsEnemy(player) and Config.EnemyColor or Config.TeamColor
end

local function GetBoundingBox(char)
    local root = char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Torso") or char:FindFirstChild("Head")
    if not root then return nil end
    local pos = root.Position
    local headScreen, hv = Camera:WorldToViewportPoint(pos + Vector3.new(0, 3, 0))
    local footScreen, fv = Camera:WorldToViewportPoint(pos - Vector3.new(0, 3, 0))
    if not hv and not fv then return nil end
    local height = math.abs(footScreen.Y - headScreen.Y)
    local width = height * 0.55
    local cx = (headScreen.X + footScreen.X) / 2
    return {
        TopLeft     = Vector2.new(cx - width / 2, headScreen.Y),
        TopRight    = Vector2.new(cx + width / 2, headScreen.Y),
        BottomLeft  = Vector2.new(cx - width / 2, footScreen.Y),
        BottomRight = Vector2.new(cx + width / 2, footScreen.Y),
        Center      = Vector2.new(cx, (headScreen.Y + footScreen.Y) / 2),
        Width = width, Height = height,
        Distance = (Camera.CFrame.Position - pos).Magnitude,
    }
end

-- render loop
local Connection
Connection = RunService.RenderStepped:Connect(function()
    if not Config.Enabled then
        for _, esp in pairs(ESPObjects) do HideESP(esp) end
        return
    end
    for player, esp in pairs(ESPObjects) do
        local char = player.Character
        if char and not char:IsDescendantOf(workspace) then char = nil end
        local humanoid = char and char:FindFirstChildOfClass("Humanoid")
        local root = char and (char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Torso"))

        if not char or not humanoid or not root or humanoid.Health <= 0 then
            HideESP(esp)
            continue
        end

        local dist = (Camera.CFrame.Position - root.Position).Magnitude
        if dist > Config.MaxDistance then HideESP(esp); continue end
        if Config.TeamCheck and not IsEnemy(player) then HideESP(esp); continue end

        local bb = GetBoundingBox(char)
        if not bb then HideESP(esp); continue end

        local color = GetColor(player)

        -- box
        if Config.ShowBox then
            esp.BoxLines[1].From = bb.TopLeft;     esp.BoxLines[1].To = bb.TopRight
            esp.BoxLines[2].From = bb.TopRight;    esp.BoxLines[2].To = bb.BottomRight
            esp.BoxLines[3].From = bb.BottomRight; esp.BoxLines[3].To = bb.BottomLeft
            esp.BoxLines[4].From = bb.BottomLeft;  esp.BoxLines[4].To = bb.TopLeft
            for i = 1, 4 do
                esp.BoxLines[i].Color = color
                esp.BoxLines[i].Thickness = Config.BoxThickness
                esp.BoxLines[i].Visible = true
            end
        else
            for i = 1, 4 do esp.BoxLines[i].Visible = false end
        end

        -- name
        if Config.ShowName then
            esp.Name.Position = Vector2.new(bb.Center.X, bb.TopLeft.Y - Config.TextSize - 2)
            esp.Name.Text = player.DisplayName or player.Name
            esp.Name.Color = color
            esp.Name.Size = Config.TextSize
            esp.Name.Visible = true
        else esp.Name.Visible = false end

        -- health
        if Config.ShowHealth then
            local frac = math.clamp(humanoid.Health / math.max(humanoid.MaxHealth, 1), 0, 1)
            local barX = bb.TopLeft.X - 5
            esp.HealthBG.From = Vector2.new(barX, bb.TopLeft.Y)
            esp.HealthBG.To = Vector2.new(barX, bb.BottomLeft.Y)
            esp.HealthBG.Visible = true
            local fillTop = bb.BottomLeft.Y - (bb.BottomLeft.Y - bb.TopLeft.Y) * frac
            esp.HealthFill.From = Vector2.new(barX, fillTop)
            esp.HealthFill.To = Vector2.new(barX, bb.BottomLeft.Y)
            local r = frac < 0.5 and 1 or (1 - (frac - 0.5) * 2)
            local g = frac > 0.5 and 1 or (frac * 2)
            esp.HealthFill.Color = Color3.new(r, g, 0)
            esp.HealthFill.Visible = true
        else
            esp.HealthBG.Visible = false
            esp.HealthFill.Visible = false
        end

        -- distance
        if Config.ShowDistance then
            esp.Distance.Position = Vector2.new(bb.Center.X, bb.BottomLeft.Y + 2)
            esp.Distance.Text = string.format("[%d studs]", math.floor(bb.Distance))
            esp.Distance.Color = color
            esp.Distance.Size = Config.TextSize - 1
            esp.Distance.Visible = true
        else esp.Distance.Visible = false end

        -- tracer
        if Config.ShowTracers then
            local vs = Camera.ViewportSize
            esp.Tracer.From = Vector2.new(vs.X / 2, vs.Y)
            esp.Tracer.To = Vector2.new(bb.Center.X, bb.BottomLeft.Y)
            esp.Tracer.Color = color
            esp.Tracer.Visible = true
        else esp.Tracer.Visible = false end
    end
end)

-- player events
for _, player in ipairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then CreateESP(player) end
end
Players.PlayerAdded:Connect(function(p) if p ~= LocalPlayer then CreateESP(p) end end)
Players.PlayerRemoving:Connect(function(p) RemoveESP(p) end)

-- toggle com Insert
UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == Enum.KeyCode.Insert then
        Config.Enabled = not Config.Enabled
    elseif input.KeyCode == Enum.KeyCode.RightShift then
        ScreenGui.Enabled = not ScreenGui.Enabled
    end
end)

-- ══════════════════════════════════════════
--  CLEANUP GLOBAL
-- ══════════════════════════════════════════
local function Cleanup()
    if Connection then Connection:Disconnect() end
    for p in pairs(ESPObjects) do RemoveESP(p) end
    if ScreenGui then ScreenGui:Destroy() end
end

getgenv().PF_ESP = {
    Config  = Config,
    Cleanup = Cleanup,
    GUI     = ScreenGui,
}
