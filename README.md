-- language: Lua, target: Roblox (Phantom Forces), runtime: exploit executor
-- ESP universal — busca ativa de characters, funciona em PF independente da estrutura

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local LocalPlayer = Players.LocalPlayer

-- cleanup anterior
if getgenv().PF_ESP and getgenv().PF_ESP.Cleanup then
    pcall(getgenv().PF_ESP.Cleanup)
    task.wait(0.2)
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
    DebugMode     = false, -- true = printa no console o que encontra
}

local AllConnections = {}
local function Track(conn)
    table.insert(AllConnections, conn)
    return conn
end

-- ══════════════════════════════════════════
--  CHARACTER FINDER — busca ativa
-- ══════════════════════════════════════════
-- cache: player -> {Model, RootPart, Humanoid}
local CharacterCache = {}

-- todos os lugares onde PF pode guardar characters
local function GetSearchFolders()
    local folders = {workspace}
    -- PF usa essas pastas frequentemente
    local names = {"Ignore", "Characters", "Chars", "Camera", "Map", "Terrain"}
    for _, name in ipairs(names) do
        local folder = workspace:FindFirstChild(name)
        if folder then
            table.insert(folders, folder)
        end
    end
    -- workspace.CurrentCamera também pode ter modelos
    pcall(function()
        if workspace.CurrentCamera then
            table.insert(folders, workspace.CurrentCamera)
        end
    end)
    return folders
end

local function FindCharacterForPlayer(player)
    -- método 1: player.Character direto
    local char = player.Character
    if char and char.Parent then
        local root = char:FindFirstChild("HumanoidRootPart")
            or char:FindFirstChild("Torso")
            or char:FindFirstChild("UpperTorso")
            or char:FindFirstChild("Head")
        if root then
            return char, root
        end
    end

    -- método 2: busca por nome do player em workspace children
    for _, child in ipairs(workspace:GetChildren()) do
        if child:IsA("Model") and child.Name == player.Name then
            local root = child:FindFirstChild("HumanoidRootPart")
                or child:FindFirstChild("Torso")
                or child:FindFirstChild("UpperTorso")
                or child:FindFirstChild("Head")
            if root then
                return child, root
            end
        end
    end

    -- método 3: busca em subpastas conhecidas do PF
    local folders = GetSearchFolders()
    for _, folder in ipairs(folders) do
        if folder ~= workspace then
            pcall(function()
                for _, child in ipairs(folder:GetChildren()) do
                    if child:IsA("Model") then
                        -- checa por nome
                        if child.Name == player.Name then
                            local root = child:FindFirstChild("HumanoidRootPart")
                                or child:FindFirstChild("Torso")
                                or child:FindFirstChild("Head")
                            if root then
                                char = child
                                -- break não funciona aqui por causa do pcall, mas tudo bem
                            end
                        end
                        -- checa se GetPlayerFromCharacter mapeia de volta
                        local mapped = Players:GetPlayerFromCharacter(child)
                        if mapped == player then
                            local root = child:FindFirstChild("HumanoidRootPart")
                                or child:FindFirstChild("Torso")
                                or child:FindFirstChild("Head")
                            if root then
                                char = child
                            end
                        end
                    end
                end
            end)
            if char and char.Parent then
                local root = char:FindFirstChild("HumanoidRootPart")
                    or char:FindFirstChild("Torso")
                    or char:FindFirstChild("Head")
                if root then
                    return char, root
                end
            end
        end
    end

    -- método 4: busca por GetPlayerFromCharacter em TODOS os models do workspace
    -- mais pesado, roda só se os outros falharam
    local found, foundRoot = nil, nil
    pcall(function()
        for _, desc in ipairs(workspace:GetDescendants()) do
            if desc:IsA("Model") then
                local mapped = Players:GetPlayerFromCharacter(desc)
                if mapped == player then
                    local root = desc:FindFirstChild("HumanoidRootPart")
                        or desc:FindFirstChild("Torso")
                        or desc:FindFirstChild("Head")
                    if root then
                        found = desc
                        foundRoot = root
                    end
                end
            end
        end
    end)

    return found, foundRoot
end

-- atualiza cache
local function RefreshCache()
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local char, root = FindCharacterForPlayer(player)
            if char and root then
                local humanoid = char:FindFirstChildOfClass("Humanoid")
                CharacterCache[player] = {
                    Model = char,
                    Root = root,
                    Humanoid = humanoid,
                    Alive = true,
                }
                if Config.DebugMode then
                    print("[ESP] Found: " .. player.Name .. " at " .. tostring(char:GetFullName()))
                end
            else
                CharacterCache[player] = nil
                if Config.DebugMode then
                    print("[ESP] NOT found: " .. player.Name)
                end
            end
        end
    end
end

-- loop de refresh (0.5s)
local refreshRunning = true
task.spawn(function()
    while refreshRunning do
        pcall(RefreshCache)
        task.wait(0.5)
    end
end)

-- também ouve CharacterAdded pra refresh imediato
for _, player in ipairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then
        pcall(function()
            Track(player.CharacterAdded:Connect(function()
                task.wait(0.3) -- espera montar
                pcall(RefreshCache)
            end))
        end)
    end
end

Track(Players.PlayerAdded:Connect(function(player)
    if player ~= LocalPlayer then
        pcall(function()
            Track(player.CharacterAdded:Connect(function()
                task.wait(0.3)
                pcall(RefreshCache)
            end))
        end)
        task.wait(1)
        pcall(RefreshCache)
    end
end))

Track(Players.PlayerRemoving:Connect(function(player)
    CharacterCache[player] = nil
end))

-- ══════════════════════════════════════════
--  DRAWING ESP
-- ══════════════════════════════════════════
local ESPDrawings = {}

local function SafeNew(class, props)
    local ok, obj = pcall(Drawing.new, class)
    if not ok or not obj then return nil end
    if props then
        for k, v in pairs(props) do
            pcall(function() obj[k] = v end)
        end
    end
    return obj
end

local function EnsureDrawings(player)
    if ESPDrawings[player] then return ESPDrawings[player] end
    local d = {}
    d.Box = {}
    for i = 1, 4 do
        d.Box[i] = SafeNew("Line", {Visible = false, Thickness = 1.4})
    end
    d.Name = SafeNew("Text", {Visible = false, Center = true, Outline = true, Size = 13})
    d.HealthBG = SafeNew("Line", {Visible = false, Thickness = 3, Color = Color3.fromRGB(0,0,0)})
    d.HealthFill = SafeNew("Line", {Visible = false, Thickness = 1.5})
    d.Dist = SafeNew("Text", {Visible = false, Center = true, Outline = true, Size = 12})
    d.Tracer = SafeNew("Line", {Visible = false, Thickness = 1})
    -- valida
    if not d.Box[1] or not d.Name then return nil end
    ESPDrawings[player] = d
    return d
end

local function HideAll(d)
    if not d then return end
    pcall(function()
        for i = 1, 4 do if d.Box[i] then d.Box[i].Visible = false end end
        if d.Name then d.Name.Visible = false end
        if d.HealthBG then d.HealthBG.Visible = false end
        if d.HealthFill then d.HealthFill.Visible = false end
        if d.Dist then d.Dist.Visible = false end
        if d.Tracer then d.Tracer.Visible = false end
    end)
end

local function DestroyDrawings(player)
    local d = ESPDrawings[player]
    if not d then return end
    pcall(function()
        for i = 1, 4 do if d.Box[i] then d.Box[i]:Remove() end end
        if d.Name then d.Name:Remove() end
        if d.HealthBG then d.HealthBG:Remove() end
        if d.HealthFill then d.HealthFill:Remove() end
        if d.Dist then d.Dist:Remove() end
        if d.Tracer then d.Tracer:Remove() end
    end)
    ESPDrawings[player] = nil
end

-- ══════════════════════════════════════════
--  TEAM / COLOR
-- ══════════════════════════════════════════
local function IsEnemy(player)
    if not Config.TeamCheck then return true end
    local ok, res = pcall(function()
        local mt = LocalPlayer.Team
        local tt = player.Team
        if not mt or not tt then return true end
        return mt ~= tt
    end)
    return ok and res or true
end

-- ══════════════════════════════════════════
--  HEALTH — múltiplas fontes
-- ══════════════════════════════════════════
local function GetHealth(cache)
    local hp, maxhp = 100, 100
    -- fonte 1: Humanoid
    if cache.Humanoid then
        pcall(function()
            hp = cache.Humanoid.Health
            maxhp = cache.Humanoid.MaxHealth
        end)
    end
    -- fonte 2: value objects dentro do character (PF usa isso às vezes)
    pcall(function()
        local hpVal = cache.Model:FindFirstChild("Health")
        if hpVal and hpVal:IsA("NumberValue") or hpVal:IsA("IntValue") then
            hp = hpVal.Value
        end
        local maxVal = cache.Model:FindFirstChild("MaxHealth")
        if maxVal then
            maxhp = maxVal.Value
        end
    end)
    if maxhp <= 0 then maxhp = 100 end
    return hp, maxhp
end

local function IsAlive(cache)
    -- se tem root part com parent, considera vivo
    if not cache.Root or not cache.Root.Parent then return false end
    if not cache.Model or not cache.Model.Parent then return false end
    -- checa humanoid se existir
    if cache.Humanoid then
        local ok, dead = pcall(function()
            return cache.Humanoid.Health <= 0
        end)
        if ok and dead then return false end
    end
    return true
end

-- ══════════════════════════════════════════
--  BOUNDING BOX
-- ══════════════════════════════════════════
local function GetBBox(rootPart)
    local cam = workspace.CurrentCamera
    if not cam then return nil end

    local pos = rootPart.Position
    local dist = (cam.CFrame.Position - pos).Magnitude

    local top = pos + Vector3.new(0, 3.2, 0)
    local bot = pos - Vector3.new(0, 3.2, 0)

    local ts, tv = cam:WorldToViewportPoint(top)
    local bs, bv = cam:WorldToViewportPoint(bot)

    if ts.Z < 0 and bs.Z < 0 then return nil end
    if not tv and not bv then return nil end

    local h = math.abs(bs.Y - ts.Y)
    if h < 3 then return nil end
    local w = h * 0.55
    local cx = (ts.X + bs.X) / 2

    return {
        TL = Vector2.new(cx - w/2, ts.Y),
        TR = Vector2.new(cx + w/2, ts.Y),
        BL = Vector2.new(cx - w/2, bs.Y),
        BR = Vector2.new(cx + w/2, bs.Y),
        CT = Vector2.new(cx, (ts.Y + bs.Y) / 2),
        W = w, H = h, D = dist,
    }
end

-- ══════════════════════════════════════════
--  RENDER
-- ══════════════════════════════════════════
local RenderConn
RenderConn = RunService.RenderStepped:Connect(function()
    local ok, err = pcall(function()
        -- limpa drawings de players que saíram
        for player, d in pairs(ESPDrawings) do
            if not player.Parent or player == LocalPlayer then
                HideAll(d)
            end
        end

        if not Config.Enabled then
            for _, d in pairs(ESPDrawings) do HideAll(d) end
            return
        end

        for player, cache in pairs(CharacterCache) do
            local d = EnsureDrawings(player)
            if not d then
                -- nada a fazer sem drawings
            elseif not IsAlive(cache) then
                HideAll(d)
            elseif Config.TeamCheck and not IsEnemy(player) then
                HideAll(d)
            else
                -- refresh root ref (pode mudar)
                local root = cache.Root
                if not root or not root.Parent then
                    HideAll(d)
                else
                    local cam = workspace.CurrentCamera
                    if not cam then
                        HideAll(d)
                    else
                        local dist = (cam.CFrame.Position - root.Position).Magnitude
                        if dist > Config.MaxDistance then
                            HideAll(d)
                        else
                            local bb = GetBBox(root)
                            if not bb then
                                HideAll(d)
                            else
                                local color = IsEnemy(player) and Config.EnemyColor or Config.TeamColor

                                -- BOX
                                if Config.ShowBox then
                                    d.Box[1].From = bb.TL; d.Box[1].To = bb.TR
                                    d.Box[2].From = bb.TR; d.Box[2].To = bb.BR
                                    d.Box[3].From = bb.BR; d.Box[3].To = bb.BL
                                    d.Box[4].From = bb.BL; d.Box[4].To = bb.TL
                                    for i = 1, 4 do
                                        d.Box[i].Color = color
                                        d.Box[i].Thickness = Config.BoxThickness
                                        d.Box[i].Visible = true
                                    end
                                else
                                    for i = 1, 4 do d.Box[i].Visible = false end
                                end

                                -- NAME
                                if Config.ShowName then
                                    d.Name.Position = Vector2.new(bb.CT.X, bb.TL.Y - Config.TextSize - 2)
                                    d.Name.Text = player.DisplayName or player.Name
                                    d.Name.Color = color
                                    d.Name.Size = Config.TextSize
                                    d.Name.Visible = true
                                else
                                    d.Name.Visible = false
                                end

                                -- HEALTH
                                if Config.ShowHealth then
                                    local hp, maxhp = GetHealth(cache)
                                    local frac = math.clamp(hp / maxhp, 0, 1)
                                    local bx = bb.TL.X - 5
                                    d.HealthBG.From = Vector2.new(bx, bb.TL.Y)
                                    d.HealthBG.To = Vector2.new(bx, bb.BL.Y)
                                    d.HealthBG.Visible = true
                                    local ft = bb.BL.Y - (bb.BL.Y - bb.TL.Y) * frac
                                    d.HealthFill.From = Vector2.new(bx, ft)
                                    d.HealthFill.To = Vector2.new(bx, bb.BL.Y)
                                    local r = 1
                                    local g = 1
                                    if frac < 0.5 then g = frac * 2
                                    else r = 1 - (frac - 0.5) * 2 end
                                    d.HealthFill.Color = Color3.new(r, g, 0)
                                    d.HealthFill.Visible = true
                                else
                                    d.HealthBG.Visible = false
                                    d.HealthFill.Visible = false
                                end

                                -- DISTANCE
                                if Config.ShowDistance then
                                    d.Dist.Position = Vector2.new(bb.CT.X, bb.BL.Y + 2)
                                    d.Dist.Text = "[" .. tostring(math.floor(bb.D)) .. " studs]"
                                    d.Dist.Color = color
                                    d.Dist.Size = Config.TextSize - 1
                                    d.Dist.Visible = true
                                else
                                    d.Dist.Visible = false
                                end

                                -- TRACER
                                if Config.ShowTracers then
                                    local vs = cam.ViewportSize
                                    d.Tracer.From = Vector2.new(vs.X / 2, vs.Y)
                                    d.Tracer.To = Vector2.new(bb.CT.X, bb.BL.Y)
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

        -- esconde drawings de players sem cache
        for player, d in pairs(ESPDrawings) do
            if not CharacterCache[player] then
                HideAll(d)
            end
        end
    end)

    if not ok and Config.DebugMode then
        warn("[ESP RENDER] " .. tostring(err))
    end
end)

-- ══════════════════════════════════════════
--  GUI (mesmo visual, compacto)
-- ══════════════════════════════════════════
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "PF_ESP_GUI"
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.ResetOnSpawn = false
pcall(function()
    if gethui then ScreenGui.Parent = gethui()
    elseif syn and syn.protect_gui then syn.protect_gui(ScreenGui); ScreenGui.Parent = game:GetService("CoreGui")
    else ScreenGui.Parent = game:GetService("CoreGui") end
end)

local Theme = {
    BG = Color3.fromRGB(20,20,28), Top = Color3.fromRGB(30,30,42),
    El = Color3.fromRGB(35,35,50), Acc = Color3.fromRGB(100,120,255),
    Tx = Color3.fromRGB(220,220,230), Dim = Color3.fromRGB(140,140,160),
    On = Color3.fromRGB(80,200,120), Off = Color3.fromRGB(70,70,90),
    SBG = Color3.fromRGB(40,40,55), SF = Color3.fromRGB(100,120,255),
    Bdr = Color3.fromRGB(50,50,70),
}

local MF = Instance.new("Frame")
MF.Size = UDim2.new(0,280,0,440); MF.Position = UDim2.new(0.5,-140,0.5,-220)
MF.BackgroundColor3 = Theme.BG; MF.BorderSizePixel = 0; MF.ClipsDescendants = true; MF.Parent = ScreenGui
Instance.new("UICorner", MF).CornerRadius = UDim.new(0,8)
local mst = Instance.new("UIStroke", MF); mst.Color = Theme.Bdr; mst.Thickness = 1

local TB = Instance.new("Frame"); TB.Size = UDim2.new(1,0,0,32); TB.BackgroundColor3 = Theme.Top; TB.BorderSizePixel = 0; TB.Parent = MF
Instance.new("UICorner", TB).CornerRadius = UDim.new(0,8)
local TP = Instance.new("Frame"); TP.Size = UDim2.new(1,0,0,10); TP.Position = UDim2.new(0,0,1,-10); TP.BackgroundColor3 = Theme.Top; TP.BorderSizePixel = 0; TP.Parent = TB

local TL = Instance.new("TextLabel"); TL.Size = UDim2.new(1,-60,1,0); TL.Position = UDim2.new(0,12,0,0)
TL.BackgroundTransparency = 1; TL.Text = "PF ESP"; TL.TextColor3 = Theme.Acc; TL.TextSize = 14; TL.Font = Enum.Font.GothamBold; TL.TextXAlignment = Enum.TextXAlignment.Left; TL.Parent = TB

local MB = Instance.new("TextButton"); MB.Size = UDim2.new(0,28,0,22); MB.Position = UDim2.new(1,-36,0.5,-11)
MB.BackgroundColor3 = Theme.El; MB.BorderSizePixel = 0; MB.Text = "-"; MB.TextColor3 = Theme.Dim; MB.TextSize = 14; MB.Font = Enum.Font.GothamBold; MB.Parent = TB
Instance.new("UICorner", MB).CornerRadius = UDim.new(0,4)

local CT = Instance.new("ScrollingFrame"); CT.Size = UDim2.new(1,-16,1,-40); CT.Position = UDim2.new(0,8,0,36)
CT.BackgroundTransparency = 1; CT.BorderSizePixel = 0; CT.ScrollBarThickness = 3; CT.ScrollBarImageColor3 = Theme.Acc
CT.CanvasSize = UDim2.new(0,0,0,0); CT.AutomaticCanvasSize = Enum.AutomaticSize.Y; CT.Parent = MF
local CL = Instance.new("UIListLayout"); CL.SortOrder = Enum.SortOrder.LayoutOrder; CL.Padding = UDim.new(0,4); CL.Parent = CT

local LO = 0
local function NO() LO = LO + 1; return LO end

local function Sec(n)
    local f = Instance.new("Frame"); f.Size = UDim2.new(1,0,0,28); f.BackgroundTransparency = 1; f.LayoutOrder = NO(); f.Parent = CT
    local l = Instance.new("TextLabel"); l.Size = UDim2.new(1,0,1,0); l.Position = UDim2.new(0,4,0,0); l.BackgroundTransparency = 1
    l.Text = string.upper(n); l.TextColor3 = Theme.Acc; l.TextSize = 11; l.Font = Enum.Font.GothamBold; l.TextXAlignment = Enum.TextXAlignment.Left; l.Parent = f
    local ln = Instance.new("Frame"); ln.Size = UDim2.new(1,-8,0,1); ln.Position = UDim2.new(0,4,1,-1); ln.BackgroundColor3 = Theme.Bdr; ln.BorderSizePixel = 0; ln.Parent = f
end

local function Tog(n, def, cb)
    local st = def
    local fr = Instance.new("Frame"); fr.Size = UDim2.new(1,0,0,32); fr.BackgroundColor3 = Theme.El; fr.BorderSizePixel = 0; fr.LayoutOrder = NO(); fr.Parent = CT
    Instance.new("UICorner", fr).CornerRadius = UDim.new(0,6)
    local lb = Instance.new("TextLabel"); lb.Size = UDim2.new(1,-60,1,0); lb.Position = UDim2.new(0,10,0,0); lb.BackgroundTransparency = 1
    lb.Text = n; lb.TextColor3 = Theme.Tx; lb.TextSize = 12; lb.Font = Enum.Font.Gotham; lb.TextXAlignment = Enum.TextXAlignment.Left; lb.Parent = fr
    local bg = Instance.new("Frame"); bg.Size = UDim2.new(0,36,0,18); bg.Position = UDim2.new(1,-46,0.5,-9)
    bg.BackgroundColor3 = st and Theme.On or Theme.Off; bg.BorderSizePixel = 0; bg.Parent = fr
    Instance.new("UICorner", bg).CornerRadius = UDim.new(1,0)
    local kn = Instance.new("Frame"); kn.Size = UDim2.new(0,14,0,14)
    kn.Position = st and UDim2.new(1,-16,0.5,-7) or UDim2.new(0,2,0.5,-7)
    kn.BackgroundColor3 = Color3.new(1,1,1); kn.BorderSizePixel = 0; kn.Parent = bg
    Instance.new("UICorner", kn).CornerRadius = UDim.new(1,0)
    local bt = Instance.new("TextButton"); bt.Size = UDim2.new(1,0,1,0); bt.BackgroundTransparency = 1; bt.Text = ""; bt.Parent = fr
    local function U()
        local ti = TweenInfo.new(0.2, Enum.EasingStyle.Quad)
        TweenService:Create(bg, ti, {BackgroundColor3 = st and Theme.On or Theme.Off}):Play()
        TweenService:Create(kn, ti, {Position = st and UDim2.new(1,-16,0.5,-7) or UDim2.new(0,2,0.5,-7)}):Play()
    end
    bt.MouseButton1Click:Connect(function() st = not st; U(); cb(st) end)
end

local function Sld(n, mn, mx, def, cb)
    local val = def
    local fr = Instance.new("Frame"); fr.Size = UDim2.new(1,0,0,46); fr.BackgroundColor3 = Theme.El; fr.BorderSizePixel = 0; fr.LayoutOrder = NO(); fr.Parent = CT
    Instance.new("UICorner", fr).CornerRadius = UDim.new(0,6)
    local lb = Instance.new("TextLabel"); lb.Size = UDim2.new(1,-60,0,20); lb.Position = UDim2.new(0,10,0,2); lb.BackgroundTransparency = 1
    lb.Text = n; lb.TextColor3 = Theme.Tx; lb.TextSize = 12; lb.Font = Enum.Font.Gotham; lb.TextXAlignment = Enum.TextXAlignment.Left; lb.Parent = fr
    local vl = Instance.new("TextLabel"); vl.Size = UDim2.new(0,50,0,20); vl.Position = UDim2.new(1,-56,0,2); vl.BackgroundTransparency = 1
    vl.Text = tostring(math.floor(val)); vl.TextColor3 = Theme.Acc; vl.TextSize = 12; vl.Font = Enum.Font.GothamBold; vl.TextXAlignment = Enum.TextXAlignment.Right; vl.Parent = fr
    local sb = Instance.new("Frame"); sb.Size = UDim2.new(1,-20,0,6); sb.Position = UDim2.new(0,10,0,30); sb.BackgroundColor3 = Theme.SBG; sb.BorderSizePixel = 0; sb.Parent = fr
    Instance.new("UICorner", sb).CornerRadius = UDim.new(1,0)
    local sf = Instance.new("Frame"); sf.Size = UDim2.new((val-mn)/(mx-mn),0,1,0); sf.BackgroundColor3 = Theme.SF; sf.BorderSizePixel = 0; sf.Parent = sb
    Instance.new("UICorner", sf).CornerRadius = UDim.new(1,0)
    local kn = Instance.new("Frame"); kn.Size = UDim2.new(0,12,0,12); kn.AnchorPoint = Vector2.new(0.5,0.5)
    kn.Position = UDim2.new((val-mn)/(mx-mn),0,0.5,0); kn.BackgroundColor3 = Color3.new(1,1,1); kn.BorderSizePixel = 0; kn.ZIndex = 2; kn.Parent = sb
    Instance.new("UICorner", kn).CornerRadius = UDim.new(1,0)
    local dr = false
    local ib = Instance.new("TextButton"); ib.Size = UDim2.new(1,0,0,20); ib.Position = UDim2.new(0,0,0,24); ib.BackgroundTransparency = 1; ib.Text = ""; ib.Parent = fr
    local function Up(x)
        local rel = math.clamp((x - sb.AbsolutePosition.X) / sb.AbsoluteSize.X, 0, 1)
        val = math.floor(mn + (mx - mn) * rel)
        sf.Size = UDim2.new(rel, 0, 1, 0); kn.Position = UDim2.new(rel, 0, 0.5, 0)
        vl.Text = tostring(val); cb(val)
    end
    ib.MouseButton1Down:Connect(function() dr = true end)
    Track(UserInputService.InputChanged:Connect(function(i) if dr and i.UserInputType == Enum.UserInputType.MouseMovement then Up(i.Position.X) end end))
    Track(UserInputService.InputEnded:Connect(function(i) if i.UserInputType == Enum.UserInputType.MouseButton1 then dr = false end end))
    ib.MouseButton1Click:Connect(function() Up(UserInputService:GetMouseLocation().X) end)
end

local function CPick(n, def, cb)
    local ps = {
        Color3.fromRGB(255,50,50), Color3.fromRGB(255,120,30), Color3.fromRGB(255,220,50),
        Color3.fromRGB(80,200,120), Color3.fromRGB(50,180,255), Color3.fromRGB(100,120,255),
        Color3.fromRGB(180,80,255), Color3.fromRGB(255,80,180), Color3.fromRGB(255,255,255), Color3.fromRGB(200,200,200),
    }
    local fr = Instance.new("Frame"); fr.Size = UDim2.new(1,0,0,52); fr.BackgroundColor3 = Theme.El; fr.BorderSizePixel = 0; fr.LayoutOrder = NO(); fr.Parent = CT
    Instance.new("UICorner", fr).CornerRadius = UDim.new(0,6)
    local lb = Instance.new("TextLabel"); lb.Size = UDim2.new(1,-20,0,20); lb.Position = UDim2.new(0,10,0,2); lb.BackgroundTransparency = 1
    lb.Text = n; lb.TextColor3 = Theme.Tx; lb.TextSize = 12; lb.Font = Enum.Font.Gotham; lb.TextXAlignment = Enum.TextXAlignment.Left; lb.Parent = fr
    local sel = Instance.new("Frame"); sel.Size = UDim2.new(0,14,0,14); sel.Position = UDim2.new(1,-28,0,5)
    sel.BackgroundColor3 = def; sel.BorderSizePixel = 0; sel.Parent = fr
    Instance.new("UICorner", sel).CornerRadius = UDim.new(1,0)
    local ss = Instance.new("UIStroke", sel); ss.Color = Color3.new(1,1,1); ss.Thickness = 1.5
    for i, c in ipairs(ps) do
        local sw = Instance.new("TextButton"); sw.Size = UDim2.new(0,20,0,16); sw.Position = UDim2.new(0,10+(i-1)*24,0,28)
        sw.BackgroundColor3 = c; sw.BorderSizePixel = 0; sw.Text = ""; sw.Parent = fr
        Instance.new("UICorner", sw).CornerRadius = UDim.new(0,4)
        sw.MouseButton1Click:Connect(function() sel.BackgroundColor3 = c; cb(c) end)
    end
end

-- drag
do
    local dg, ds, sp
    TB.InputBegan:Connect(function(i) if i.UserInputType == Enum.UserInputType.MouseButton1 then dg = true; ds = i.Position; sp = MF.Position end end)
    Track(UserInputService.InputChanged:Connect(function(i)
        if dg and i.UserInputType == Enum.UserInputType.MouseMovement then
            local d = i.Position - ds; MF.Position = UDim2.new(sp.X.Scale, sp.X.Offset+d.X, sp.Y.Scale, sp.Y.Offset+d.Y)
        end
    end))
    Track(UserInputService.InputEnded:Connect(function(i) if i.UserInputType == Enum.UserInputType.MouseButton1 then dg = false end end))
end

local exp = true
MB.MouseButton1Click:Connect(function()
    exp = not exp
    TweenService:Create(MF, TweenInfo.new(0.25, Enum.EasingStyle.Quad), {Size = exp and UDim2.new(0,280,0,440) or UDim2.new(0,280,0,32)}):Play()
    MB.Text = exp and "-" or "+"
end)

-- populate
Sec("Geral")
Tog("ESP Ativo", Config.Enabled, function(v) Config.Enabled = v end)
Tog("Team Check", Config.TeamCheck, function(v) Config.TeamCheck = v end)
Tog("Debug Console", Config.DebugMode, function(v) Config.DebugMode = v end)
Sec("Elementos")
Tog("Boxes", Config.ShowBox, function(v) Config.ShowBox = v end)
Tog("Nomes", Config.ShowName, function(v) Config.ShowName = v end)
Tog("Barra de HP", Config.ShowHealth, function(v) Config.ShowHealth = v end)
Tog("Distancia", Config.ShowDistance, function(v) Config.ShowDistance = v end)
Tog("Tracers", Config.ShowTracers, function(v) Config.ShowTracers = v end)
Sec("Ajustes")
Sld("Distancia Max", 100, 5000, Config.MaxDistance, function(v) Config.MaxDistance = v end)
Sld("Tamanho Texto", 8, 24, Config.TextSize, function(v) Config.TextSize = v end)
Sld("Espessura Box", 1, 5, math.floor(Config.BoxThickness), function(v) Config.BoxThickness = v end)
Sec("Cores")
CPick("Cor Inimigo", Config.EnemyColor, function(v) Config.EnemyColor = v end)
CPick("Cor Aliado", Config.TeamColor, function(v) Config.TeamColor = v end)

-- keybinds
Track(UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == Enum.KeyCode.Insert then Config.Enabled = not Config.Enabled
    elseif input.KeyCode == Enum.KeyCode.RightShift then ScreenGui.Enabled = not ScreenGui.Enabled end
end))

-- ══════════════════════════════════════════
--  CLEANUP
-- ══════════════════════════════════════════
local function Cleanup()
    refreshRunning = false
    if RenderConn then RenderConn:Disconnect() end
    for _, c in ipairs(AllConnections) do pcall(function() c:Disconnect() end) end
    for p in pairs(ESPDrawings) do DestroyDrawings(p) end
    CharacterCache = {}
    ESPDrawings = {}
    if ScreenGui then ScreenGui:Destroy() end
end

getgenv().PF_ESP = {Config = Config, Cleanup = Cleanup, GUI = ScreenGui, Cache = CharacterCache}
