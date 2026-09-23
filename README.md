-- language: Lua, target: Roblox (Phantom Forces), runtime: exploit executor
-- ESP + GUI — versão corrigida, sem 'continue', pcall no render, detecção PF robusta

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer

-- limpa instância anterior se reexecutar
if getgenv().PF_ESP and getgenv().PF_ESP.Cleanup then
    pcall(getgenv().PF_ESP.Cleanup)
end

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
--  GUI
-- ══════════════════════════════════════════
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "PF_ESP_GUI"
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.ResetOnSpawn = false

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

local Theme = {
    Background    = Color3.fromRGB(20, 20, 28),
    TopBar        = Color3.fromRGB(30, 30, 42),
    Element       = Color3.fromRGB(35, 35, 50),
    Accent        = Color3.fromRGB(100, 120, 255),
    Text          = Color3.fromRGB(220, 220, 230),
    TextDim       = Color3.fromRGB(140, 140, 160),
    ToggleOn      = Color3.fromRGB(80, 200, 120),
    ToggleOff     = Color3.fromRGB(70, 70, 90),
    SliderBG      = Color3.fromRGB(40, 40, 55),
    SliderFill    = Color3.fromRGB(100, 120, 255),
    Border        = Color3.fromRGB(50, 50, 70),
}

local MainFrame = Instance.new("Frame")
MainFrame.Name = "Main"
MainFrame.Size = UDim2.new(0, 280, 0, 420)
MainFrame.Position = UDim2.new(0.5, -140, 0.5, -210)
MainFrame.BackgroundColor3 = Theme.Background
MainFrame.BorderSizePixel = 0
MainFrame.ClipsDescendants = true
MainFrame.Parent = ScreenGui

Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 8)
local ms = Instance.new("UIStroke", MainFrame)
ms.Color = Theme.Border; ms.Thickness = 1

local TopBar = Instance.new("Frame")
TopBar.Size = UDim2.new(1, 0, 0, 32)
TopBar.BackgroundColor3 = Theme.TopBar
TopBar.BorderSizePixel = 0
TopBar.Parent = MainFrame
Instance.new("UICorner", TopBar).CornerRadius = UDim.new(0, 8)

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
TitleLabel.Text = "PF ESP"
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
MinimizeBtn.Text = "-"
MinimizeBtn.TextColor3 = Theme.TextDim
MinimizeBtn.TextSize = 14
MinimizeBtn.Font = Enum.Font.GothamBold
MinimizeBtn.Parent = TopBar
Instance.new("UICorner", MinimizeBtn).CornerRadius = UDim.new(0, 4)

local Content = Instance.new("ScrollingFrame")
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

-- ── GUI BUILDERS ──
local LayoutOrder = 0
local function NextOrder() LayoutOrder = LayoutOrder + 1; return LayoutOrder end

local function AddSection(name)
    local f = Instance.new("Frame")
    f.Size = UDim2.new(1, 0, 0, 28)
    f.BackgroundTransparency = 1
    f.LayoutOrder = NextOrder()
    f.Parent = Content
    local l = Instance.new("TextLabel")
    l.Size = UDim2.new(1, 0, 1, 0)
    l.Position = UDim2.new(0, 4, 0, 0)
    l.BackgroundTransparency = 1
    l.Text = string.upper(name)
    l.TextColor3 = Theme.Accent
    l.TextSize = 11
    l.Font = Enum.Font.GothamBold
    l.TextXAlignment = Enum.TextXAlignment.Left
    l.Parent = f
    local ln = Instance.new("Frame")
    ln.Size = UDim2.new(1, -8, 0, 1)
    ln.Position = UDim2.new(0, 4, 1, -1)
    ln.BackgroundColor3 = Theme.Border
    ln.BorderSizePixel = 0
    ln.Parent = f
end

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

    local function Upd()
        local ti = TweenInfo.new(0.2, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
        TweenService:Create(toggleBG, ti, {BackgroundColor3 = state and Theme.ToggleOn or Theme.ToggleOff}):Play()
        TweenService:Create(knob, ti, {Position = state and UDim2.new(1, -16, 0.5, -7) or UDim2.new(0, 2, 0.5, -7)}):Play()
    end
    btn.MouseButton1Click:Connect(function() state = not state; Upd(); callback(state) end)
end

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

    local vl = Instance.new("TextLabel")
    vl.Size = UDim2.new(0, 50, 0, 20)
    vl.Position = UDim2.new(1, -56, 0, 2)
    vl.BackgroundTransparency = 1
    vl.Text = tostring(math.floor(value))
    vl.TextColor3 = Theme.Accent
    vl.TextSize = 12
    vl.Font = Enum.Font.GothamBold
    vl.TextXAlignment = Enum.TextXAlignment.Right
    vl.Parent = frame

    local sbg = Instance.new("Frame")
    sbg.Size = UDim2.new(1, -20, 0, 6)
    sbg.Position = UDim2.new(0, 10, 0, 30)
    sbg.BackgroundColor3 = Theme.SliderBG
    sbg.BorderSizePixel = 0
    sbg.Parent = frame
    Instance.new("UICorner", sbg).CornerRadius = UDim.new(1, 0)

    local sf = Instance.new("Frame")
    sf.Size = UDim2.new((value - min) / (max - min), 0, 1, 0)
    sf.BackgroundColor3 = Theme.SliderFill
    sf.BorderSizePixel = 0
    sf.Parent = sbg
    Instance.new("UICorner", sf).CornerRadius = UDim.new(1, 0)

    local knob = Instance.new("Frame")
    knob.Size = UDim2.new(0, 12, 0, 12)
    knob.AnchorPoint = Vector2.new(0.5, 0.5)
    knob.Position = UDim2.new((value - min) / (max - min), 0, 0.5, 0)
    knob.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    knob.BorderSizePixel = 0
    knob.ZIndex = 2
    knob.Parent = sbg
    Instance.new("UICorner", knob).CornerRadius = UDim.new(1, 0)

    local dragging = false
    local ib = Instance.new("TextButton")
    ib.Size = UDim2.new(1, 0, 0, 20)
    ib.Position = UDim2.new(0, 0, 0, 24)
    ib.BackgroundTransparency = 1
    ib.Text = ""
    ib.Parent = frame

    local function Upd(x)
        local ap = sbg.AbsolutePosition.X
        local as = sbg.AbsoluteSize.X
        local rel = math.clamp((x - ap) / as, 0, 1)
        value = math.floor(min + (max - min) * rel)
        sf.Size = UDim2.new(rel, 0, 1, 0)
        knob.Position = UDim2.new(rel, 0, 0.5, 0)
        vl.Text = tostring(value)
        callback(value)
    end

    ib.MouseButton1Down:Connect(function() dragging = true end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            Upd(input.Position.X)
        end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then dragging = false end
    end)
    ib.MouseButton1Click:Connect(function()
        local m = UserInputService:GetMouseLocation()
        Upd(m.X)
    end)
end

local function AddColorPicker(name, default, callback)
    local presets = {
        Color3.fromRGB(255, 50, 50), Color3.fromRGB(255, 120, 30),
        Color3.fromRGB(255, 220, 50), Color3.fromRGB(80, 200, 120),
        Color3.fromRGB(50, 180, 255), Color3.fromRGB(100, 120, 255),
        Color3.fromRGB(180, 80, 255), Color3.fromRGB(255, 80, 180),
        Color3.fromRGB(255, 255, 255), Color3.fromRGB(200, 200, 200),
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

    local sel = Instance.new("Frame")
    sel.Size = UDim2.new(0, 14, 0, 14)
    sel.Position = UDim2.new(1, -28, 0, 5)
    sel.BackgroundColor3 = default
    sel.BorderSizePixel = 0
    sel.Parent = frame
    Instance.new("UICorner", sel).CornerRadius = UDim.new(1, 0)
    local ss = Instance.new("UIStroke", sel)
    ss.Color = Color3.fromRGB(255,255,255); ss.Thickness = 1.5

    for i, color in ipairs(presets) do
        local sw = Instance.new("TextButton")
        sw.Size = UDim2.new(0, 20, 0, 16)
        sw.Position = UDim2.new(0, 10 + (i-1) * 24, 0, 28)
        sw.BackgroundColor3 = color
        sw.BorderSizePixel = 0
        sw.Text = ""
        sw.Parent = frame
        Instance.new("UICorner", sw).CornerRadius = UDim.new(0, 4)
        sw.MouseButton1Click:Connect(function() sel.BackgroundColor3 = color; callback(color) end)
    end
end

-- ── DRAGGING ──
do
    local dragging, dragStart, startPos
    TopBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true; dragStart = input.Position; startPos = MainFrame.Position
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            local d = input.Position - dragStart
            MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X, startPos.Y.Scale, startPos.Y.Offset + d.Y)
        end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then dragging = false end
    end)
end

-- ── MINIMIZE ──
local expanded = true
MinimizeBtn.MouseButton1Click:Connect(function()
    expanded = not expanded
    TweenService:Create(MainFrame, TweenInfo.new(0.25, Enum.EasingStyle.Quad), {
        Size = expanded and UDim2.new(0, 280, 0, 420) or UDim2.new(0, 280, 0, 32)
    }):Play()
    MinimizeBtn.Text = expanded and "-" or "+"
end)

-- ── POPULATE ──
AddSection("Geral")
AddToggle("ESP Ativo", Config.Enabled, function(v) Config.Enabled = v end)
AddToggle("Team Check", Config.TeamCheck, function(v) Config.TeamCheck = v end)
AddSection("Elementos")
AddToggle("Boxes", Config.ShowBox, function(v) Config.ShowBox = v end)
AddToggle("Nomes", Config.ShowName, function(v) Config.ShowName = v end)
AddToggle("Barra de HP", Config.ShowHealth, function(v) Config.ShowHealth = v end)
AddToggle("Distancia", Config.ShowDistance, function(v) Config.ShowDistance = v end)
AddToggle("Tracers", Config.ShowTracers, function(v) Config.ShowTracers = v end)
AddSection("Ajustes")
AddSlider("Distancia Max", 100, 5000, Config.MaxDistance, function(v) Config.MaxDistance = v end)
AddSlider("Tamanho Texto", 8, 24, Config.TextSize, function(v) Config.TextSize = v end)
AddSlider("Espessura Box", 1, 5, math.floor(Config.BoxThickness), function(v) Config.BoxThickness = v end)
AddSection("Cores")
AddColorPicker("Cor Inimigo", Config.EnemyColor, function(v) Config.EnemyColor = v end)
AddColorPicker("Cor Aliado", Config.TeamColor, function(v) Config.TeamColor = v end)

-- ══════════════════════════════════════════
--  ESP ENGINE — SEM 'continue', COM pcall
-- ══════════════════════════════════════════
local ESPObjects = {}
local Connections = {}

local function CreateDrawing(class, props)
    local ok, obj = pcall(Drawing.new, class)
    if not ok then return nil end
    for k, v in pairs(props or {}) do
        obj[k] = v
    end
    return obj
end

local function CreateESP(player)
    local esp = {}
    esp.BoxLines = {}
    for i = 1, 4 do
        esp.BoxLines[i] = CreateDrawing("Line", {Visible = false, Thickness = Config.BoxThickness, Color = Config.EnemyColor})
    end
    esp.Name = CreateDrawing("Text", {Visible = false, Center = true, Outline = true, Size = Config.TextSize})
    esp.HealthBG = CreateDrawing("Line", {Visible = false, Thickness = 3, Color = Color3.fromRGB(0,0,0)})
    esp.HealthFill = CreateDrawing("Line", {Visible = false, Thickness = 1.5})
    esp.Distance = CreateDrawing("Text", {Visible = false, Center = true, Outline = true, Size = Config.TextSize - 1})
    esp.Tracer = CreateDrawing("Line", {Visible = false, Thickness = 1})

    -- verifica se Drawing funcionou
    if not esp.BoxLines[1] then return end

    ESPObjects[player] = esp
end

local function RemoveESP(player)
    local esp = ESPObjects[player]
    if not esp then return end
    pcall(function()
        for _, line in ipairs(esp.BoxLines) do if line then line:Remove() end end
        if esp.Name then esp.Name:Remove() end
        if esp.HealthBG then esp.HealthBG:Remove() end
        if esp.HealthFill then esp.HealthFill:Remove() end
        if esp.Distance then esp.Distance:Remove() end
        if esp.Tracer then esp.Tracer:Remove() end
    end)
    ESPObjects[player] = nil
end

local function HideESP(esp)
    pcall(function()
        for _, line in ipairs(esp.BoxLines) do if line then line.Visible = false end end
        if esp.Name then esp.Name.Visible = false end
        if esp.HealthBG then esp.HealthBG.Visible = false end
        if esp.HealthFill then esp.HealthFill.Visible = false end
        if esp.Distance then esp.Distance.Visible = false end
        if esp.Tracer then esp.Tracer.Visible = false end
    end)
end

local function IsEnemy(player)
    if not Config.TeamCheck then return true end
    local ok, result = pcall(function()
        local myTeam = LocalPlayer.Team
        local theirTeam = player.Team
        if not myTeam or not theirTeam then return true end
        return myTeam ~= theirTeam
    end)
    if not ok then return true end
    return result
end

local function GetColor(player)
    return IsEnemy(player) and Config.EnemyColor or Config.TeamColor
end

-- busca character de múltiplas formas (PF usa estrutura não-padrão às vezes)
local function GetCharacter(player)
    local char = player.Character
    if char and char.Parent then return char end
    return nil
end

local function GetRoot(char)
    return char:FindFirstChild("HumanoidRootPart")
        or char:FindFirstChild("Torso")
        or char:FindFirstChild("UpperTorso")
        or char:FindFirstChild("Head")
end

local function GetHumanoid(char)
    return char:FindFirstChildOfClass("Humanoid")
end

local function GetBoundingBox(char)
    local root = GetRoot(char)
    if not root then return nil end

    local pos = root.Position
    Camera = workspace.CurrentCamera -- refresh ref
    if not Camera then return nil end

    local headPos = pos + Vector3.new(0, 3, 0)
    local footPos = pos - Vector3.new(0, 3, 0)

    local headScreen, headVis = Camera:WorldToViewportPoint(headPos)
    local footScreen, footVis = Camera:WorldToViewportPoint(footPos)

    -- precisa estar na frente da câmera
    if headScreen.Z < 0 and footScreen.Z < 0 then return nil end
    if not headVis and not footVis then return nil end

    local height = math.abs(footScreen.Y - headScreen.Y)
    if height < 4 then return nil end -- muito pequeno
    local width = height * 0.55
    local cx = (headScreen.X + footScreen.X) / 2

    return {
        TopLeft     = Vector2.new(cx - width/2, headScreen.Y),
        TopRight    = Vector2.new(cx + width/2, headScreen.Y),
        BottomLeft  = Vector2.new(cx - width/2, footScreen.Y),
        BottomRight = Vector2.new(cx + width/2, footScreen.Y),
        Center      = Vector2.new(cx, (headScreen.Y + footScreen.Y) / 2),
        Width = width,
        Height = height,
        Distance = (Camera.CFrame.Position - pos).Magnitude,
    }
end

-- processa UM player por vez — chamado dentro do loop
local function ProcessPlayer(player, esp)
    -- player saiu ou é o local
    if not player or not player.Parent or player == LocalPlayer then
        HideESP(esp)
        return
    end

    -- esp desligado
    if not Config.Enabled then
        HideESP(esp)
        return
    end

    -- character válido
    local char = GetCharacter(player)
    if not char then
        HideESP(esp)
        return
    end

    local humanoid = GetHumanoid(char)
    local root = GetRoot(char)

    if not humanoid or not root then
        HideESP(esp)
        return
    end

    if humanoid.Health <= 0 then
        HideESP(esp)
        return
    end

    -- distância
    Camera = workspace.CurrentCamera
    if not Camera then
        HideESP(esp)
        return
    end

    local dist = (Camera.CFrame.Position - root.Position).Magnitude
    if dist > Config.MaxDistance then
        HideESP(esp)
        return
    end

    -- team check
    if Config.TeamCheck and not IsEnemy(player) then
        HideESP(esp)
        return
    end

    -- bounding box
    local bb = GetBoundingBox(char)
    if not bb then
        HideESP(esp)
        return
    end

    local color = GetColor(player)

    -- ── BOX ──
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

    -- ── NAME ──
    if Config.ShowName then
        esp.Name.Position = Vector2.new(bb.Center.X, bb.TopLeft.Y - Config.TextSize - 2)
        esp.Name.Text = player.DisplayName or player.Name
        esp.Name.Color = color
        esp.Name.Size = Config.TextSize
        esp.Name.Visible = true
    else
        esp.Name.Visible = false
    end

    -- ── HEALTH BAR ──
    if Config.ShowHealth then
        local maxHP = humanoid.MaxHealth
        if maxHP <= 0 then maxHP = 100 end
        local frac = math.clamp(humanoid.Health / maxHP, 0, 1)

        local barX = bb.TopLeft.X - 5
        local barTop = bb.TopLeft.Y
        local barBot = bb.BottomLeft.Y

        esp.HealthBG.From = Vector2.new(barX, barTop)
        esp.HealthBG.To = Vector2.new(barX, barBot)
        esp.HealthBG.Visible = true

        local fillTop = barBot - (barBot - barTop) * frac
        esp.HealthFill.From = Vector2.new(barX, fillTop)
        esp.HealthFill.To = Vector2.new(barX, barBot)

        local r = 1
        local g = 1
        if frac < 0.5 then
            g = frac * 2
        else
            r = 1 - (frac - 0.5) * 2
        end
        esp.HealthFill.Color = Color3.new(r, g, 0)
        esp.HealthFill.Visible = true
    else
        esp.HealthBG.Visible = false
        esp.HealthFill.Visible = false
    end

    -- ── DISTANCE ──
    if Config.ShowDistance then
        esp.Distance.Position = Vector2.new(bb.Center.X, bb.BottomLeft.Y + 2)
        esp.Distance.Text = "[" .. tostring(math.floor(bb.Distance)) .. " studs]"
        esp.Distance.Color = color
        esp.Distance.Size = Config.TextSize - 1
        esp.Distance.Visible = true
    else
        esp.Distance.Visible = false
    end

    -- ── TRACER ──
    if Config.ShowTracers then
        local vs = Camera.ViewportSize
        esp.Tracer.From = Vector2.new(vs.X / 2, vs.Y)
        esp.Tracer.To = Vector2.new(bb.Center.X, bb.BottomLeft.Y)
        esp.Tracer.Color = color
        esp.Tracer.Visible = true
    else
        esp.Tracer.Visible = false
    end
end

-- ── RENDER LOOP (pcall protegido) ──
local RenderConnection
RenderConnection = RunService.RenderStepped:Connect(function()
    local ok, err = pcall(function()
        for player, esp in pairs(ESPObjects) do
            ProcessPlayer(player, esp)
        end
    end)
    -- se der erro, printa mas NÃO desconecta — mantém tentando
    if not ok then
        -- warn("ESP err: " .. tostring(err)) -- descomenta pra debug
    end
end)

-- ── PLAYER EVENTS ──
for _, player in ipairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then
        CreateESP(player)
    end
end

table.insert(Connections, Players.PlayerAdded:Connect(function(p)
    if p ~= LocalPlayer then CreateESP(p) end
end))

table.insert(Connections, Players.PlayerRemoving:Connect(function(p)
    RemoveESP(p)
end))

-- ── KEYBINDS ──
table.insert(Connections, UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == Enum.KeyCode.Insert then
        Config.Enabled = not Config.Enabled
    elseif input.KeyCode == Enum.KeyCode.RightShift then
        ScreenGui.Enabled = not ScreenGui.Enabled
    end
end))

-- ══════════════════════════════════════════
--  CLEANUP
-- ══════════════════════════════════════════
local function Cleanup()
    if RenderConnection then RenderConnection:Disconnect() end
    for _, conn in ipairs(Connections) do
        pcall(function() conn:Disconnect() end)
    end
    for p in pairs(ESPObjects) do RemoveESP(p) end
    if ScreenGui then ScreenGui:Destroy() end
end

getgenv().PF_ESP = {
    Config  = Config,
    Cleanup = Cleanup,
    GUI     = ScreenGui,
}
