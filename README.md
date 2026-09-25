-- language: Lua, target: Roblox (Universal / PF), runtime: exploit executor
-- ESP + Aimbot: sintaxe 100% compativel Luau/5.1, sem goto/continue, fallback seguro de UI

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local LocalPlayer = Players.LocalPlayer

if getgenv().PF_ESP and getgenv().PF_ESP.Cleanup then
    pcall(getgenv().PF_ESP.Cleanup)
    task.wait(0.2)
end

-- ══════════════════════════════════════════
--  CONFIG
-- ══════════════════════════════════════════
local Config = {
    -- ESP
    Enabled         = true,
    ShowBox         = true,
    ShowName        = true,
    ShowHealth      = true,
    ShowDistance    = true,
    ShowTracers     = true,
    TeamCheck       = true,
    MaxDistance     = 2000,
    EnemyColor      = Color3.fromRGB(255, 50, 50),
    TeamColor       = Color3.fromRGB(50, 255, 50),
    BoxThickness    = 1.4,
    TextSize        = 13,
    -- Aimbot
    AimbotEnabled   = false,
    AimbotKey       = nil,
    FOV             = 150,
    ShowFOV         = true,
    Smoothing       = 15,
    TargetPart      = "Head",
    AimbotTeamCheck = true,
    Prediction      = false,
    PredFactor      = 12,
}

-- input matching: Config.AimbotKey stores the raw Enum value (KeyCode or UserInputType)
local function InputMatchesKey(input, key)
    if not key then return false end
    return input.UserInputType == key or input.KeyCode == key
end

local function GetInputName(input)
    if input.UserInputType ~= Enum.UserInputType.Keyboard then
        return input.UserInputType.Name
    end
    return input.KeyCode.Name
end

local function GetKeyEnum(input)
    if input.UserInputType ~= Enum.UserInputType.Keyboard then
        return input.UserInputType
    end
    return input.KeyCode
end

local Conns = {}
local function Trk(c) table.insert(Conns, c); return c end

-- ══════════════════════════════════════════
--  PLAYER MAP
-- ══════════════════════════════════════════
local CharToPlayer = {}
local function MapP(p)
    if p == LocalPlayer then return end
    if p.Character then CharToPlayer[p.Character] = p end
end
for _, p in ipairs(Players:GetPlayers()) do
    MapP(p)
    pcall(function() Trk(p.CharacterAdded:Connect(function(c) CharToPlayer[c] = p end)) end)
end
Trk(Players.PlayerAdded:Connect(function(p)
    if p == LocalPlayer then return end
    MapP(p)
    pcall(function() Trk(p.CharacterAdded:Connect(function(c) CharToPlayer[c] = p end)) end)
end))
Trk(Players.PlayerRemoving:Connect(function(p)
    for c, pl in pairs(CharToPlayer) do
        if pl == p then CharToPlayer[c] = nil end
    end
end))

-- ══════════════════════════════════════════
--  TEAM CHECK
-- ══════════════════════════════════════════
local function IsEnemy(player)
    if not player then return true end
    local my = LocalPlayer
    if my.Team and player.Team then return my.Team ~= player.Team end
    local mc, tc = my.TeamColor, player.TeamColor
    if mc and tc and mc ~= BrickColor.new("White") then return mc ~= tc end
    if my.Neutral then return true end
    return true
end

-- ══════════════════════════════════════════
--  DRAWING HELPERS
-- ══════════════════════════════════════════
local function SDraw(class, props)
    local ok, obj = pcall(Drawing.new, class)
    if not ok then return nil end
    for k, v in pairs(props or {}) do pcall(function() obj[k] = v end) end
    return obj
end

local function NewDraw()
    local d = {}; d.Box = {}
    for i = 1, 4 do d.Box[i] = SDraw("Line", {Visible = false, Thickness = 1.4}) end
    d.Name = SDraw("Text", {Visible = false, Center = true, Outline = true, Size = 13})
    d.HpBG = SDraw("Line", {Visible = false, Thickness = 3, Color = Color3.fromRGB(0, 0, 0)})
    d.HpFill = SDraw("Line", {Visible = false, Thickness = 1.5})
    d.Dist = SDraw("Text", {Visible = false, Center = true, Outline = true, Size = 12})
    d.Tracer = SDraw("Line", {Visible = false, Thickness = 1})
    if not d.Box[1] then return nil end
    return d
end

local function Hide(d)
    if not d then return end
    for i = 1, 4 do if d.Box[i] then d.Box[i].Visible = false end end
    if d.Name then d.Name.Visible = false end
    if d.HpBG then d.HpBG.Visible = false end
    if d.HpFill then d.HpFill.Visible = false end
    if d.Dist then d.Dist.Visible = false end
    if d.Tracer then d.Tracer.Visible = false end
end

local function DDraw(d)
    if not d then return end
    pcall(function()
        for i = 1, 4 do if d.Box[i] then d.Box[i]:Remove() end end
        if d.Name then d.Name:Remove() end
        if d.HpBG then d.HpBG:Remove() end
        if d.HpFill then d.HpFill:Remove() end
        if d.Dist then d.Dist:Remove() end
        if d.Tracer then d.Tracer:Remove() end
    end)
end

-- ══════════════════════════════════════════
--  TRACKING & CACHE
-- ══════════════════════════════════════════
local Tracked = {}

local function CacheRoot(m)
    local p = m.PrimaryPart
    if p and p:IsA("BasePart") then return p end
    local names = {"HumanoidRootPart", "RootPart", "Torso", "UpperTorso", "LowerTorso", "Head"}
    for _, n in ipairs(names) do
        p = m:FindFirstChild(n)
        if p and p:IsA("BasePart") then return p end
    end
    return m:FindFirstChildWhichIsA("BasePart", true)
end

local function UntrackModel(model)
    local data = Tracked[model]
    if not data then return end
    Hide(data.Draw)
    DDraw(data.Draw)
    Tracked[model] = nil
end

local function TrackModel(model)
    if not model or not model:IsA("Model") or Tracked[model] then return end
    local player = CharToPlayer[model]
    if not player then
        local ok, p = pcall(Players.GetPlayerFromCharacter, Players, model)
        if ok and p then player = p; CharToPlayer[model] = p end
    end
    if not player then
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LocalPlayer and (p.Name == model.Name or p.DisplayName == model.Name) then
                player = p; CharToPlayer[model] = p; break
            end
        end
    end
    if player == LocalPlayer then return end
    local d = NewDraw()
    if not d then return end
    local root = CacheRoot(model)
    local hum = model:FindFirstChildOfClass("Humanoid")
    Tracked[model] = {Player = player, Draw = d, Root = root, Hum = hum}
    Trk(model.ChildAdded:Connect(function(child)
        local data = Tracked[model]
        if not data then return end
        if child:IsA("Humanoid") then
            data.Hum = child
        elseif child:IsA("BasePart") and not data.Root then
            data.Root = child
        end
    end))
end

Trk(workspace.DescendantAdded:Connect(function(desc)
    if desc:IsA("Humanoid") then
        local m = desc.Parent
        if m and m:IsA("Model") then
            task.defer(function()
                task.wait(1)
                if m and m.Parent then TrackModel(m) end
            end)
        end
    end
end))

Trk(workspace.DescendantRemoving:Connect(function(desc)
    if desc:IsA("Humanoid") then
        local m = desc.Parent
        if m then UntrackModel(m) end
    end
end))

task.defer(function()
    for _, desc in ipairs(workspace:GetDescendants()) do
        if desc:IsA("Humanoid") then
            local m = desc.Parent
            if m and m:IsA("Model") and m.Parent then TrackModel(m) end
        end
    end
end)

-- ══════════════════════════════════════════
--  AIMBOT ENGINE
-- ══════════════════════════════════════════
local aimHeld = false
local currentTarget = nil
local FOVCircle = SDraw("Circle", {
    Visible = false,
    Radius = Config.FOV,
    Thickness = 1,
    Color = Color3.fromRGB(255, 255, 255),
    Transparency = 0.6,
    Filled = false,
    NumSides = 60,
})

local function GetAimPart(model, data)
    local name = Config.TargetPart
    if name == "Head" then
        local p = model:FindFirstChild("Head")
        if p and p:IsA("BasePart") then return p end
    elseif name == "Torso" then
        local p = model:FindFirstChild("UpperTorso") or model:FindFirstChild("Torso")
        if p and p:IsA("BasePart") then return p end
    end
    return data.Root or CacheRoot(model)
end

local function GetTargetPos(part, model)
    local pos = part.Position
    if Config.Prediction then
        local root = model:FindFirstChild("HumanoidRootPart") or model.PrimaryPart
        if root then
            local vel = Vector3.zero
            pcall(function() vel = root.AssemblyLinearVelocity or root.Velocity end)
            pos = pos + vel * (Config.PredFactor / 100)
        end
    end
    return pos
end

local function FindBestTarget(cam)
    local center = cam.ViewportSize / 2
    local bestDist = Config.FOV
    local bestModel, bestData = nil, nil

    for model, data in pairs(Tracked) do
        if model.Parent then
            local root = data.Root
            if root and root.Parent then
                local hum = data.Hum
                local alive = true
                if hum and hum.Parent and hum.Health <= 0 then alive = false end
                if alive then
                    local player = data.Player or CharToPlayer[model]
                    local enemyOk = true
                    if Config.AimbotTeamCheck then enemyOk = IsEnemy(player) end
                    if enemyOk then
                        local aimPart = GetAimPart(model, data)
                        if aimPart and aimPart.Parent then
                            local screen, vis = cam:WorldToViewportPoint(aimPart.Position)
                            if vis and screen.Z > 0 then
                                local dist2D = (Vector2.new(screen.X, screen.Y) - center).Magnitude
                                if dist2D < bestDist then
                                    bestDist = dist2D
                                    bestModel = model
                                    bestData = data
                                end
                            end
                        end
                    end
                end
            end
        end
    end

    return bestModel, bestData
end

-- resolve mousemoverel across executor APIs
local _moverel = (typeof(mousemoverel) == "function" and mousemoverel)
    or (typeof(Input) == "table" and typeof(Input.MouseMoveRel) == "function" and function(dx, dy) Input.MouseMoveRel(dx, dy) end)
    or (typeof(input) == "table" and typeof(input.mousemoverel) == "function" and function(dx, dy) input.mousemoverel(dx, dy) end)
    or nil

local function AimbotStep()
    local cam = workspace.CurrentCamera
    if not cam then return end

    if FOVCircle then
        if Config.AimbotEnabled and Config.ShowFOV then
            local center = cam.ViewportSize / 2
            FOVCircle.Position = center
            FOVCircle.Radius = Config.FOV
            FOVCircle.Visible = true
        else
            FOVCircle.Visible = false
        end
    end

    if not Config.AimbotEnabled or not aimHeld then
        currentTarget = nil
        return
    end

    local model = currentTarget
    local data = model and Tracked[model]
    if model then
        if not model.Parent or (data and data.Hum and data.Hum.Health <= 0) then
            model = nil
            currentTarget = nil
        end
    end

    if not model then
        model, data = FindBestTarget(cam)
        currentTarget = model
    end

    if not model or not data then return end

    local aimPart = GetAimPart(model, data)
    if not aimPart or not aimPart.Parent then return end

    local targetPos = GetTargetPos(aimPart, model)
    local screenPos, onScreen = cam:WorldToViewportPoint(targetPos)

    if not onScreen or screenPos.Z <= 0 then return end

    local center = cam.ViewportSize / 2
    local dx = screenPos.X - center.X
    local dy = screenPos.Y - center.Y

    -- smoothing: higher = slower/smoother, lower = snappier
    local smooth = math.clamp(Config.Smoothing, 1, 100)
    local factor = 1 / smooth

    if _moverel then
        _moverel(dx * factor, dy * factor)
    else
        -- fallback: direct CFrame (funciona em jogos sem camera override)
        local targetCF = CFrame.new(cam.CFrame.Position, targetPos)
        cam.CFrame = cam.CFrame:Lerp(targetCF, factor)
    end
end

pcall(function()
    RunService:BindToRenderStep("__aimbot", Enum.RenderPriority.Camera.Value + 1, AimbotStep)
end)


Trk(UserInputService.InputBegan:Connect(function(input, processed)
    -- no 'processed' check: in PF the game consumes mouse input when holding a weapon
    if not Config.AimbotKey then return end
    if InputMatchesKey(input, Config.AimbotKey) then
        aimHeld = true
    end
end))

Trk(UserInputService.InputEnded:Connect(function(input)
    if not Config.AimbotKey then return end
    if InputMatchesKey(input, Config.AimbotKey) then
        aimHeld = false
        currentTarget = nil
    end
end))

-- ══════════════════════════════════════════
--  ESP RENDER
-- ══════════════════════════════════════════
local frameN = 0
local RC = RunService.RenderStepped:Connect(function()
    frameN = frameN + 1
    if frameN % 3 ~= 0 then return end
    local cam = workspace.CurrentCamera
    if not cam then return end
    local camPos = cam.CFrame.Position

    for model, data in pairs(Tracked) do
        local d = data.Draw
        if not Config.Enabled or not model.Parent then
            Hide(d)
        else
            local root = data.Root
            if not root or not root.Parent then
                root = CacheRoot(model)
                data.Root = root
            end

            if not root or not root.Parent then
                Hide(d)
            else
                local hum = data.Hum
                if not hum or not hum.Parent then
                    hum = model:FindFirstChildOfClass("Humanoid")
                    data.Hum = hum
                end

                if hum and hum.Health <= 0 then
                    Hide(d)
                else
                    local player = data.Player or CharToPlayer[model]
                    data.Player = player
                    if Config.TeamCheck and not IsEnemy(player) then
                        Hide(d)
                    else
                        local pos = root.Position
                        local dx = camPos.X - pos.X
                        local dy = camPos.Y - pos.Y
                        local dz = camPos.Z - pos.Z
                        local dist2 = dx * dx + dy * dy + dz * dz
                        local maxD = Config.MaxDistance

                        if dist2 > maxD * maxD then
                            Hide(d)
                        else
                            local dist = math.sqrt(dist2)
                            local ts, tv = cam:WorldToViewportPoint(pos + Vector3.new(0, 3.2, 0))
                            local bs, bv = cam:WorldToViewportPoint(pos - Vector3.new(0, 3.2, 0))

                            if (ts.Z < 0 and bs.Z < 0) or (not tv and not bv) then
                                Hide(d)
                            else
                                local h = math.abs(bs.Y - ts.Y)
                                if h < 3 then
                                    Hide(d)
                                else
                                    local w = h * 0.55
                                    local cx = (ts.X + bs.X) * 0.5
                                    local isTarget = (model == currentTarget)
                                    local color = isTarget and Color3.fromRGB(255, 200, 0)
                                        or (IsEnemy(player) and Config.EnemyColor or Config.TeamColor)

                                    if Config.ShowBox then
                                        local tl = Vector2.new(cx - w * 0.5, ts.Y)
                                        local tr = Vector2.new(cx + w * 0.5, ts.Y)
                                        local bl = Vector2.new(cx - w * 0.5, bs.Y)
                                        local br = Vector2.new(cx + w * 0.5, bs.Y)
                                        d.Box[1].From = tl; d.Box[1].To = tr
                                        d.Box[2].From = tr; d.Box[2].To = br
                                        d.Box[3].From = br; d.Box[3].To = bl
                                        d.Box[4].From = bl; d.Box[4].To = tl
                                        for i = 1, 4 do
                                            d.Box[i].Color = color
                                            d.Box[i].Thickness = isTarget and 2.2 or Config.BoxThickness
                                            d.Box[i].Visible = true
                                        end
                                    else
                                        for i = 1, 4 do d.Box[i].Visible = false end
                                    end

                                    if Config.ShowName then
                                        d.Name.Position = Vector2.new(cx, ts.Y - Config.TextSize - 2)
                                        d.Name.Text = player and (player.DisplayName or player.Name) or model.Name
                                        d.Name.Color = color
                                        d.Name.Size = Config.TextSize
                                        d.Name.Visible = true
                                    else
                                        d.Name.Visible = false
                                    end

                                    if Config.ShowHealth and hum then
                                        local mx = hum.MaxHealth
                                        if mx <= 0 then mx = 100 end
                                        local frac = math.clamp(hum.Health / mx, 0, 1)
                                        local bx = cx - w * 0.5 - 5
                                        d.HpBG.From = Vector2.new(bx, ts.Y)
                                        d.HpBG.To = Vector2.new(bx, bs.Y)
                                        d.HpBG.Visible = true
                                        local ft = bs.Y - (bs.Y - ts.Y) * frac
                                        d.HpFill.From = Vector2.new(bx, ft)
                                        d.HpFill.To = Vector2.new(bx, bs.Y)
                                        local r2, g2 = 1, 1
                                        if frac < 0.5 then g2 = frac * 2 else r2 = 1 - (frac - 0.5) * 2 end
                                        d.HpFill.Color = Color3.new(r2, g2, 0)
                                        d.HpFill.Visible = true
                                    else
                                        d.HpBG.Visible = false
                                        d.HpFill.Visible = false
                                    end

                                    if Config.ShowDistance then
                                        d.Dist.Position = Vector2.new(cx, bs.Y + 2)
                                        d.Dist.Text = "[" .. tostring(math.floor(dist)) .. "]"
                                        d.Dist.Color = color
                                        d.Dist.Size = Config.TextSize - 1
                                        d.Dist.Visible = true
                                    else
                                        d.Dist.Visible = false
                                    end

                                    if Config.ShowTracers then
                                        local vs = cam.ViewportSize
                                        d.Tracer.From = Vector2.new(vs.X * 0.5, vs.Y)
                                        d.Tracer.To = Vector2.new(cx, bs.Y)
                                        d.Tracer.Color = color
                                        d.Tracer.Visible = true
                                    else
                                        d.Tracer.Visible = false
                                    end
                                end
                            end
                        end
                    end
                end
            end
        end
    end
end)

-- ══════════════════════════════════════════
--  GUI ROBUSTA (Parent Fallback Garantido)
-- ══════════════════════════════════════════
local SG = Instance.new("ScreenGui")
SG.Name = "UESP"
SG.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
SG.ResetOnSpawn = false

local parented = false
if gethui then
    local ok, res = pcall(gethui)
    if ok and res then
        SG.Parent = res
        parented = true
    end
end
if not parented and syn and syn.protect_gui then
    pcall(function()
        syn.protect_gui(SG)
        SG.Parent = game:GetService("CoreGui")
        parented = true
    end)
end
if not parented then
    local ok = pcall(function() SG.Parent = game:GetService("CoreGui") end)
    if not ok or not SG.Parent then
        pcall(function() SG.Parent = LocalPlayer:WaitForChild("PlayerGui") end)
    end
end

local C = {
    BG        = Color3.fromRGB(15, 15, 22),
    Top       = Color3.fromRGB(22, 22, 32),
    TopAccent = Color3.fromRGB(90, 110, 255),
    El        = Color3.fromRGB(28, 28, 40),
    Acc       = Color3.fromRGB(90, 110, 255),
    AccDim    = Color3.fromRGB(60, 75, 180),
    Tx        = Color3.fromRGB(225, 225, 235),
    Dm        = Color3.fromRGB(120, 120, 145),
    On        = Color3.fromRGB(70, 210, 110),
    Off       = Color3.fromRGB(55, 55, 75),
    SBg       = Color3.fromRGB(35, 35, 50),
    SFl       = Color3.fromRGB(90, 110, 255),
    Bd        = Color3.fromRGB(40, 40, 58),
    Tab       = Color3.fromRGB(20, 20, 30),
    TabA      = Color3.fromRGB(32, 32, 48),
}

local MF = Instance.new("Frame")
MF.Size = UDim2.new(0, 300, 0, 480)
MF.Position = UDim2.new(0.5, -150, 0.5, -240)
MF.BackgroundColor3 = C.BG
MF.BorderSizePixel = 0
MF.ClipsDescendants = true
MF.Parent = SG
Instance.new("UICorner", MF).CornerRadius = UDim.new(0, 10)
local mst = Instance.new("UIStroke", MF)
mst.Color = C.Bd
mst.Thickness = 1

local TB = Instance.new("Frame")
TB.Size = UDim2.new(1, 0, 0, 38)
TB.BackgroundColor3 = C.Top
TB.BorderSizePixel = 0
TB.Parent = MF
Instance.new("UICorner", TB).CornerRadius = UDim.new(0, 10)

local tPatch = Instance.new("Frame")
tPatch.Size = UDim2.new(1, 0, 0, 12)
tPatch.Position = UDim2.new(0, 0, 1, -12)
tPatch.BackgroundColor3 = C.Top
tPatch.BorderSizePixel = 0
tPatch.Parent = TB

local accLine = Instance.new("Frame")
accLine.Size = UDim2.new(1, 0, 0, 2)
accLine.Position = UDim2.new(0, 0, 1, -1)
accLine.BackgroundColor3 = C.TopAccent
accLine.BorderSizePixel = 0
accLine.Parent = TB

local titleL = Instance.new("TextLabel")
titleL.Size = UDim2.new(0, 80, 1, 0)
titleL.Position = UDim2.new(0, 14, 0, 0)
titleL.BackgroundTransparency = 1
titleL.Text = "PHANTOM"
titleL.TextColor3 = C.Acc
titleL.TextSize = 13
titleL.Font = Enum.Font.GothamBold
titleL.TextXAlignment = Enum.TextXAlignment.Left
titleL.Parent = TB

local statL = Instance.new("TextLabel")
statL.Size = UDim2.new(0, 80, 1, 0)
statL.Position = UDim2.new(0, 96, 0, 0)
statL.BackgroundTransparency = 1
statL.Text = "0 alvos"
statL.TextColor3 = C.Dm
statL.TextSize = 11
statL.Font = Enum.Font.Gotham
statL.TextXAlignment = Enum.TextXAlignment.Left
statL.Parent = TB

task.spawn(function()
    while task.wait(1) do
        local count = 0
        for _ in pairs(Tracked) do count = count + 1 end
        statL.Text = tostring(count) .. " alvos"
    end
end)

local mmb = Instance.new("TextButton")
mmb.Size = UDim2.new(0, 28, 0, 22)
mmb.Position = UDim2.new(1, -38, 0.5, -11)
mmb.BackgroundColor3 = C.El
mmb.BorderSizePixel = 0
mmb.Text = "-"
mmb.TextColor3 = C.Dm
mmb.TextSize = 14
mmb.Font = Enum.Font.GothamBold
mmb.Parent = TB
Instance.new("UICorner", mmb).CornerRadius = UDim.new(0, 4)

local TabBar = Instance.new("Frame")
TabBar.Size = UDim2.new(1, -16, 0, 30)
TabBar.Position = UDim2.new(0, 8, 0, 44)
TabBar.BackgroundColor3 = C.Tab
TabBar.BorderSizePixel = 0
TabBar.Parent = MF
Instance.new("UICorner", TabBar).CornerRadius = UDim.new(0, 6)

local tabNames = {"ESP", "AIMBOT"}
local tabBtns = {}
local tabFrames = {}

for i, name in ipairs(tabNames) do
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0.5, 0, 1, -4)
    btn.Position = UDim2.new((i - 1) * 0.5, 2, 0, 2)
    btn.BackgroundColor3 = (i == 1) and C.TabA or C.Tab
    btn.BorderSizePixel = 0
    btn.Text = name
    btn.TextColor3 = (i == 1) and C.Acc or C.Dm
    btn.TextSize = 12
    btn.Font = Enum.Font.GothamBold
    btn.Parent = TabBar
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 5)
    tabBtns[i] = btn

    local frame = Instance.new("ScrollingFrame")
    frame.Size = UDim2.new(1, -16, 1, -84)
    frame.Position = UDim2.new(0, 8, 0, 78)
    frame.BackgroundTransparency = 1
    frame.BorderSizePixel = 0
    frame.ScrollBarThickness = 3
    frame.ScrollBarImageColor3 = C.Acc
    frame.CanvasSize = UDim2.new(0, 0, 0, 0)
    frame.AutomaticCanvasSize = Enum.AutomaticSize.Y
    frame.Visible = (i == 1)
    frame.Parent = MF
    local lay = Instance.new("UIListLayout")
    lay.SortOrder = Enum.SortOrder.LayoutOrder
    lay.Padding = UDim.new(0, 4)
    lay.Parent = frame
    tabFrames[i] = frame
end

local function SwitchTab(idx)
    for i, btn in ipairs(tabBtns) do
        local active = (i == idx)
        TweenService:Create(btn, TweenInfo.new(0.15), {
            BackgroundColor3 = active and C.TabA or C.Tab,
            TextColor3 = active and C.Acc or C.Dm
        }):Play()
        tabFrames[i].Visible = active
    end
end
for i, btn in ipairs(tabBtns) do
    btn.MouseButton1Click:Connect(function() SwitchTab(i) end)
end

local function MakeIn(parent)
    local LO = 0
    local function NO() LO = LO + 1; return LO end

    local function Sec(n)
        local f = Instance.new("Frame")
        f.Size = UDim2.new(1, 0, 0, 24)
        f.BackgroundTransparency = 1
        f.LayoutOrder = NO()
        f.Parent = parent
        local l = Instance.new("TextLabel")
        l.Size = UDim2.new(1, 0, 1, 0)
        l.Position = UDim2.new(0, 4, 0, 0)
        l.BackgroundTransparency = 1
        l.Text = string.upper(n)
        l.TextColor3 = C.Acc
        l.TextSize = 10
        l.Font = Enum.Font.GothamBold
        l.TextXAlignment = Enum.TextXAlignment.Left
        l.Parent = f
        local ln = Instance.new("Frame")
        ln.Size = UDim2.new(1, -8, 0, 1)
        ln.Position = UDim2.new(0, 4, 1, -1)
        ln.BackgroundColor3 = C.Bd
        ln.BorderSizePixel = 0
        ln.Parent = f
    end

    local function Tog(n, def, cb)
        local st = def
        local fr = Instance.new("Frame")
        fr.Size = UDim2.new(1, 0, 0, 30)
        fr.BackgroundColor3 = C.El
        fr.BorderSizePixel = 0
        fr.LayoutOrder = NO()
        fr.Parent = parent
        Instance.new("UICorner", fr).CornerRadius = UDim.new(0, 6)
        local lb = Instance.new("TextLabel")
        lb.Size = UDim2.new(1, -60, 1, 0)
        lb.Position = UDim2.new(0, 10, 0, 0)
        lb.BackgroundTransparency = 1
        lb.Text = n
        lb.TextColor3 = C.Tx
        lb.TextSize = 12
        lb.Font = Enum.Font.Gotham
        lb.TextXAlignment = Enum.TextXAlignment.Left
        lb.Parent = fr
        local bg = Instance.new("Frame")
        bg.Size = UDim2.new(0, 34, 0, 16)
        bg.Position = UDim2.new(1, -44, 0.5, -8)
        bg.BackgroundColor3 = st and C.On or C.Off
        bg.BorderSizePixel = 0
        bg.Parent = fr
        Instance.new("UICorner", bg).CornerRadius = UDim.new(1, 0)
        local kn = Instance.new("Frame")
        kn.Size = UDim2.new(0, 12, 0, 12)
        kn.Position = st and UDim2.new(1, -14, 0.5, -6) or UDim2.new(0, 2, 0.5, -6)
        kn.BackgroundColor3 = Color3.new(1, 1, 1)
        kn.BorderSizePixel = 0
        kn.Parent = bg
        Instance.new("UICorner", kn).CornerRadius = UDim.new(1, 0)
        local bt = Instance.new("TextButton")
        bt.Size = UDim2.new(1, 0, 1, 0)
        bt.BackgroundTransparency = 1
        bt.Text = ""
        bt.Parent = fr
        bt.MouseButton1Click:Connect(function()
            st = not st
            local ti = TweenInfo.new(0.15, Enum.EasingStyle.Quad)
            TweenService:Create(bg, ti, {BackgroundColor3 = st and C.On or C.Off}):Play()
            TweenService:Create(kn, ti, {Position = st and UDim2.new(1, -14, 0.5, -6) or UDim2.new(0, 2, 0.5, -6)}):Play()
            cb(st)
        end)
    end

    local function Sld(n, mn, mx, def, cb)
        local val = def
        local fr = Instance.new("Frame")
        fr.Size = UDim2.new(1, 0, 0, 42)
        fr.BackgroundColor3 = C.El
        fr.BorderSizePixel = 0
        fr.LayoutOrder = NO()
        fr.Parent = parent
        Instance.new("UICorner", fr).CornerRadius = UDim.new(0, 6)
        local lb = Instance.new("TextLabel")
        lb.Size = UDim2.new(1, -55, 0, 18)
        lb.Position = UDim2.new(0, 10, 0, 2)
        lb.BackgroundTransparency = 1
        lb.Text = n
        lb.TextColor3 = C.Tx
        lb.TextSize = 11
        lb.Font = Enum.Font.Gotham
        lb.TextXAlignment = Enum.TextXAlignment.Left
        lb.Parent = fr
        local vl = Instance.new("TextLabel")
        vl.Size = UDim2.new(0, 45, 0, 18)
        vl.Position = UDim2.new(1, -50, 0, 2)
        vl.BackgroundTransparency = 1
        vl.Text = tostring(math.floor(val))
        vl.TextColor3 = C.Acc
        vl.TextSize = 11
        vl.Font = Enum.Font.GothamBold
        vl.TextXAlignment = Enum.TextXAlignment.Right
        vl.Parent = fr
        local sb = Instance.new("Frame")
        sb.Size = UDim2.new(1, -20, 0, 5)
        sb.Position = UDim2.new(0, 10, 0, 27)
        sb.BackgroundColor3 = C.SBg
        sb.BorderSizePixel = 0
        sb.Parent = fr
        Instance.new("UICorner", sb).CornerRadius = UDim.new(1, 0)
        local sf = Instance.new("Frame")
        sf.Size = UDim2.new((val - mn) / (mx - mn), 0, 1, 0)
        sf.BackgroundColor3 = C.SFl
        sf.BorderSizePixel = 0
        sf.Parent = sb
        Instance.new("UICorner", sf).CornerRadius = UDim.new(1, 0)
        local skn = Instance.new("Frame")
        skn.Size = UDim2.new(0, 10, 0, 10)
        skn.AnchorPoint = Vector2.new(0.5, 0.5)
        skn.Position = UDim2.new((val - mn) / (mx - mn), 0, 0.5, 0)
        skn.BackgroundColor3 = Color3.new(1, 1, 1)
        skn.BorderSizePixel = 0
        skn.ZIndex = 2
        skn.Parent = sb
        Instance.new("UICorner", skn).CornerRadius = UDim.new(1, 0)
        local dr = false
        local ib = Instance.new("TextButton")
        ib.Size = UDim2.new(1, 0, 0, 18)
        ib.Position = UDim2.new(0, 0, 0, 22)
        ib.BackgroundTransparency = 1
        ib.Text = ""
        ib.Parent = fr
        local function Up(x)
            local rel = math.clamp((x - sb.AbsolutePosition.X) / sb.AbsoluteSize.X, 0, 1)
            val = math.floor(mn + (mx - mn) * rel)
            sf.Size = UDim2.new(rel, 0, 1, 0)
            skn.Position = UDim2.new(rel, 0, 0.5, 0)
            vl.Text = tostring(val)
            cb(val)
        end
        ib.MouseButton1Down:Connect(function() dr = true end)
        Trk(UserInputService.InputChanged:Connect(function(i)
            if dr and i.UserInputType == Enum.UserInputType.MouseMovement then Up(i.Position.X) end
        end))
        Trk(UserInputService.InputEnded:Connect(function(i)
            if i.UserInputType == Enum.UserInputType.MouseButton1 then dr = false end
        end))
        ib.MouseButton1Click:Connect(function() Up(UserInputService:GetMouseLocation().X) end)
    end

    local function Cycle(n, options, def, cb)
        local idx = 1
        for i, v in ipairs(options) do if v == def then idx = i end end
        local fr = Instance.new("Frame")
        fr.Size = UDim2.new(1, 0, 0, 30)
        fr.BackgroundColor3 = C.El
        fr.BorderSizePixel = 0
        fr.LayoutOrder = NO()
        fr.Parent = parent
        Instance.new("UICorner", fr).CornerRadius = UDim.new(0, 6)
        local lb = Instance.new("TextLabel")
        lb.Size = UDim2.new(1, -100, 1, 0)
        lb.Position = UDim2.new(0, 10, 0, 0)
        lb.BackgroundTransparency = 1
        lb.Text = n
        lb.TextColor3 = C.Tx
        lb.TextSize = 12
        lb.Font = Enum.Font.Gotham
        lb.TextXAlignment = Enum.TextXAlignment.Left
        lb.Parent = fr
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(0, 80, 0, 22)
        btn.Position = UDim2.new(1, -90, 0.5, -11)
        btn.BackgroundColor3 = C.AccDim
        btn.BorderSizePixel = 0
        btn.Text = options[idx]
        btn.TextColor3 = Color3.new(1, 1, 1)
        btn.TextSize = 11
        btn.Font = Enum.Font.GothamBold
        btn.Parent = fr
        Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)
        btn.MouseButton1Click:Connect(function()
            idx = idx % #options + 1
            btn.Text = options[idx]
            cb(options[idx])
        end)
    end

    local function Bind(n, def, cb)
        local currentKey = def
        local listening = false
        local fr = Instance.new("Frame")
        fr.Size = UDim2.new(1, 0, 0, 30)
        fr.BackgroundColor3 = C.El
        fr.BorderSizePixel = 0
        fr.LayoutOrder = NO()
        fr.Parent = parent
        Instance.new("UICorner", fr).CornerRadius = UDim.new(0, 6)
        local lb = Instance.new("TextLabel")
        lb.Size = UDim2.new(1, -100, 1, 0)
        lb.Position = UDim2.new(0, 10, 0, 0)
        lb.BackgroundTransparency = 1
        lb.Text = n
        lb.TextColor3 = C.Tx
        lb.TextSize = 12
        lb.Font = Enum.Font.Gotham
        lb.TextXAlignment = Enum.TextXAlignment.Left
        lb.Parent = fr
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(0, 80, 0, 22)
        btn.Position = UDim2.new(1, -90, 0.5, -11)
        btn.BackgroundColor3 = C.AccDim
        btn.BorderSizePixel = 0
        btn.Text = currentKey and currentKey.Name or "Nenhuma"
        btn.TextColor3 = Color3.new(1, 1, 1)
        btn.TextSize = 11
        btn.Font = Enum.Font.GothamBold
        btn.Parent = fr
        Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)
        local conn = nil
        btn.MouseButton1Click:Connect(function()
            if listening then return end
            listening = true
            btn.Text = "..."
            TweenService:Create(btn, TweenInfo.new(0.15), {BackgroundColor3 = C.On}):Play()
            conn = UserInputService.InputBegan:Connect(function(input, processed)
                if input.UserInputType == Enum.UserInputType.Keyboard
                    or input.UserInputType == Enum.UserInputType.MouseButton1
                    or input.UserInputType == Enum.UserInputType.MouseButton2
                    or input.UserInputType == Enum.UserInputType.MouseButton3 then
                    local key = GetKeyEnum(input)
                    currentKey = key
                    btn.Text = GetInputName(input)
                    TweenService:Create(btn, TweenInfo.new(0.15), {BackgroundColor3 = C.AccDim}):Play()
                    listening = false
                    if conn then conn:Disconnect(); conn = nil end
                    cb(key)
                end
            end)
        end)
    end

    local function CPk(n, def, cb)
        local ps = {
            Color3.fromRGB(255, 50, 50), Color3.fromRGB(255, 120, 30), Color3.fromRGB(255, 220, 50),
            Color3.fromRGB(80, 200, 120), Color3.fromRGB(50, 180, 255), Color3.fromRGB(90, 110, 255),
            Color3.fromRGB(180, 80, 255), Color3.fromRGB(255, 80, 180), Color3.fromRGB(255, 255, 255), Color3.fromRGB(180, 180, 180)
        }
        local fr = Instance.new("Frame")
        fr.Size = UDim2.new(1, 0, 0, 48)
        fr.BackgroundColor3 = C.El
        fr.BorderSizePixel = 0
        fr.LayoutOrder = NO()
        fr.Parent = parent
        Instance.new("UICorner", fr).CornerRadius = UDim.new(0, 6)
        local lb = Instance.new("TextLabel")
        lb.Size = UDim2.new(1, -20, 0, 18)
        lb.Position = UDim2.new(0, 10, 0, 2)
        lb.BackgroundTransparency = 1
        lb.Text = n
        lb.TextColor3 = C.Tx
        lb.TextSize = 11
        lb.Font = Enum.Font.Gotham
        lb.TextXAlignment = Enum.TextXAlignment.Left
        lb.Parent = fr
        local sel = Instance.new("Frame")
        sel.Size = UDim2.new(0, 12, 0, 12)
        sel.Position = UDim2.new(1, -24, 0, 5)
        sel.BackgroundColor3 = def
        sel.BorderSizePixel = 0
        sel.Parent = fr
        Instance.new("UICorner", sel).CornerRadius = UDim.new(1, 0)
        local ss = Instance.new("UIStroke", sel)
        ss.Color = Color3.new(1, 1, 1)
        ss.Thickness = 1
        for i, c in ipairs(ps) do
            local sw = Instance.new("TextButton")
            sw.Size = UDim2.new(0, 22, 0, 14)
            sw.Position = UDim2.new(0, 8 + (i - 1) * 26, 0, 26)
            sw.BackgroundColor3 = c
            sw.BorderSizePixel = 0
            sw.Text = ""
            sw.Parent = fr
            Instance.new("UICorner", sw).CornerRadius = UDim.new(0, 3)
            sw.MouseButton1Click:Connect(function() sel.BackgroundColor3 = c; cb(c) end)
        end
    end

    return {Sec = Sec, Tog = Tog, Sld = Sld, Cycle = Cycle, CPk = CPk, Bind = Bind}
end

-- Tab 1: ESP
local E = MakeIn(tabFrames[1])
E.Sec("Geral")
E.Tog("ESP Ativo", Config.Enabled, function(v) Config.Enabled = v end)
E.Tog("Team Check", Config.TeamCheck, function(v) Config.TeamCheck = v end)
E.Sec("Elementos")
E.Tog("Boxes", Config.ShowBox, function(v) Config.ShowBox = v end)
E.Tog("Nomes", Config.ShowName, function(v) Config.ShowName = v end)
E.Tog("Barra de HP", Config.ShowHealth, function(v) Config.ShowHealth = v end)
E.Tog("Distancia", Config.ShowDistance, function(v) Config.ShowDistance = v end)
E.Tog("Tracers", Config.ShowTracers, function(v) Config.ShowTracers = v end)
E.Sec("Visual")
E.Sld("Distancia Max", 100, 5000, Config.MaxDistance, function(v) Config.MaxDistance = v end)
E.Sld("Tamanho Texto", 8, 24, Config.TextSize, function(v) Config.TextSize = v end)
E.Sld("Espessura Box", 1, 5, math.floor(Config.BoxThickness), function(v) Config.BoxThickness = v end)
E.Sec("Cores")
E.CPk("Cor Inimigo", Config.EnemyColor, function(v) Config.EnemyColor = v end)
E.CPk("Cor Aliado", Config.TeamColor, function(v) Config.TeamColor = v end)

-- Tab 2: AIMBOT
local A = MakeIn(tabFrames[2])
A.Sec("Geral")
A.Tog("Aimbot Ativo", Config.AimbotEnabled, function(v) Config.AimbotEnabled = v end)
A.Tog("Team Check", Config.AimbotTeamCheck, function(v) Config.AimbotTeamCheck = v end)
A.Tog("Mostrar FOV", Config.ShowFOV, function(v) Config.ShowFOV = v end)
A.Sec("Mira")
A.Bind("Tecla de Mira", Config.AimbotKey, function(v) Config.AimbotKey = v end)
A.Cycle("Parte Alvo", {"Head", "Torso", "Root"}, Config.TargetPart, function(v) Config.TargetPart = v end)
A.Sld("Raio FOV", 30, 500, Config.FOV, function(v) Config.FOV = v end)
A.Sld("Suavidade", 1, 100, Config.Smoothing, function(v) Config.Smoothing = v end)
A.Sec("Predicao")
A.Tog("Prediction", Config.Prediction, function(v) Config.Prediction = v end)
A.Sld("Fator", 1, 50, Config.PredFactor, function(v) Config.PredFactor = v end)

-- Dragging
do
    local dg, ds, sp
    TB.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then
            dg = true; ds = i.Position; sp = MF.Position
        end
    end)
    Trk(UserInputService.InputChanged:Connect(function(i)
        if dg and i.UserInputType == Enum.UserInputType.MouseMovement then
            local d = i.Position - ds
            MF.Position = UDim2.new(sp.X.Scale, sp.X.Offset + d.X, sp.Y.Scale, sp.Y.Offset + d.Y)
        end
    end))
    Trk(UserInputService.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then dg = false end
    end))
end

local exp = true
mmb.MouseButton1Click:Connect(function()
    exp = not exp
    TweenService:Create(MF, TweenInfo.new(0.2, Enum.EasingStyle.Quad), {
        Size = exp and UDim2.new(0, 300, 0, 480) or UDim2.new(0, 300, 0, 38)
    }):Play()
    mmb.Text = exp and "-" or "+"
end)

Trk(UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == Enum.KeyCode.Insert then
        Config.Enabled = not Config.Enabled
    elseif input.KeyCode == Enum.KeyCode.RightShift then
        SG.Enabled = not SG.Enabled
    end
end))

-- Cleanup
local function Cleanup()
    if RC then RC:Disconnect() end
    pcall(function() RunService:UnbindFromRenderStep("__aimbot") end)
    if FOVCircle then pcall(function() FOVCircle:Remove() end) end
    for _, c in ipairs(Conns) do pcall(function() c:Disconnect() end) end
    for model in pairs(Tracked) do UntrackModel(model) end
    Tracked = {}
    CharToPlayer = {}
    if SG then pcall(function() SG:Destroy() end) end
end

getgenv().PF_ESP = {Config = Config, Cleanup = Cleanup, GUI = SG}
