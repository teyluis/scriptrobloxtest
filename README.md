-- language: Lua, target: Roblox (Phantom Forces), runtime: exploit executor
-- ESP stealth — sem GetDescendants, sem varredura, sem polling, passivo

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local LocalPlayer = Players.LocalPlayer

if getgenv().PF_ESP and getgenv().PF_ESP.Cleanup then
    pcall(getgenv().PF_ESP.Cleanup)
    task.wait(0.2)
end

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

local Conns = {}
local function Trk(c) table.insert(Conns, c) return c end

-- ══════════════════════════════════════════
--  DRAWING — criação lazy, 1x por player
-- ══════════════════════════════════════════
local Draw = {}

local function SafeDraw(class, props)
    local ok, obj = pcall(Drawing.new, class)
    if not ok then return nil end
    for k, v in pairs(props or {}) do pcall(function() obj[k] = v end) end
    return obj
end

local function MakeDraw(player)
    if Draw[player] then return Draw[player] end
    local d = {}
    d.Box = {}
    for i = 1, 4 do d.Box[i] = SafeDraw("Line", {Visible=false, Thickness=1.4}) end
    d.Name = SafeDraw("Text", {Visible=false, Center=true, Outline=true, Size=13})
    d.HpBG = SafeDraw("Line", {Visible=false, Thickness=3, Color=Color3.fromRGB(0,0,0)})
    d.HpFill = SafeDraw("Line", {Visible=false, Thickness=1.5})
    d.Dist = SafeDraw("Text", {Visible=false, Center=true, Outline=true, Size=12})
    d.Tracer = SafeDraw("Line", {Visible=false, Thickness=1})
    if not d.Box[1] then return nil end
    Draw[player] = d
    return d
end

local function Hide(d)
    if not d then return end
    pcall(function()
        for i = 1, 4 do if d.Box[i] then d.Box[i].Visible = false end end
        if d.Name then d.Name.Visible = false end
        if d.HpBG then d.HpBG.Visible = false end
        if d.HpFill then d.HpFill.Visible = false end
        if d.Dist then d.Dist.Visible = false end
        if d.Tracer then d.Tracer.Visible = false end
    end)
end

local function Kill(player)
    local d = Draw[player]
    if not d then return end
    pcall(function()
        for i = 1, 4 do if d.Box[i] then d.Box[i]:Remove() end end
        if d.Name then d.Name:Remove() end
        if d.HpBG then d.HpBG:Remove() end
        if d.HpFill then d.HpFill:Remove() end
        if d.Dist then d.Dist:Remove() end
        if d.Tracer then d.Tracer:Remove() end
    end)
    Draw[player] = nil
end

-- ══════════════════════════════════════════
--  RENDER — throttled, sem enumeração
-- ══════════════════════════════════════════
local frameCount = 0

local function IsEnemy(player)
    if not Config.TeamCheck then return true end
    local ok, r = pcall(function()
        if not LocalPlayer.Team or not player.Team then return true end
        return LocalPlayer.Team ~= player.Team
    end)
    return ok and r or true
end

local function Render()
    -- throttle: processa a cada 2 frames (30hz em 60fps)
    frameCount = frameCount + 1
    if frameCount % 2 ~= 0 then return end

    local cam = workspace.CurrentCamera
    if not cam then return end

    for player, d in pairs(Draw) do
        if not Config.Enabled or not player.Parent then
            Hide(d)
        else
            -- pega character direto, sem busca
            local char = player.Character
            local root = nil
            local hum = nil

            if char and char.Parent then
                root = char:FindFirstChild("HumanoidRootPart")
                if not root then root = char:FindFirstChild("Head") end
                hum = char:FindFirstChildOfClass("Humanoid")
            end

            if not root or not hum then
                Hide(d)
            elseif hum.Health <= 0 then
                Hide(d)
            elseif Config.TeamCheck and not IsEnemy(player) then
                Hide(d)
            else
                local pos = root.Position
                local dist = (cam.CFrame.Position - pos).Magnitude

                if dist > Config.MaxDistance then
                    Hide(d)
                else
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
                            local cx = (ts.X + bs.X) / 2
                            local color = IsEnemy(player) and Config.EnemyColor or Config.TeamColor

                            -- BOX
                            if Config.ShowBox then
                                local tl = Vector2.new(cx-w/2, ts.Y)
                                local tr = Vector2.new(cx+w/2, ts.Y)
                                local bl = Vector2.new(cx-w/2, bs.Y)
                                local br = Vector2.new(cx+w/2, bs.Y)
                                d.Box[1].From=tl; d.Box[1].To=tr
                                d.Box[2].From=tr; d.Box[2].To=br
                                d.Box[3].From=br; d.Box[3].To=bl
                                d.Box[4].From=bl; d.Box[4].To=tl
                                for i=1,4 do d.Box[i].Color=color; d.Box[i].Thickness=Config.BoxThickness; d.Box[i].Visible=true end
                            else
                                for i=1,4 do d.Box[i].Visible=false end
                            end

                            -- NAME
                            if Config.ShowName then
                                d.Name.Position = Vector2.new(cx, ts.Y - Config.TextSize - 2)
                                d.Name.Text = player.DisplayName or player.Name
                                d.Name.Color = color; d.Name.Size = Config.TextSize; d.Name.Visible = true
                            else d.Name.Visible = false end

                            -- HP
                            if Config.ShowHealth then
                                local mx = hum.MaxHealth; if mx <= 0 then mx = 100 end
                                local frac = math.clamp(hum.Health / mx, 0, 1)
                                local bx = cx - w/2 - 5
                                d.HpBG.From = Vector2.new(bx, ts.Y); d.HpBG.To = Vector2.new(bx, bs.Y); d.HpBG.Visible = true
                                local ft = bs.Y - (bs.Y - ts.Y) * frac
                                d.HpFill.From = Vector2.new(bx, ft); d.HpFill.To = Vector2.new(bx, bs.Y)
                                local r2, g2 = 1, 1
                                if frac < 0.5 then g2 = frac * 2 else r2 = 1 - (frac - 0.5) * 2 end
                                d.HpFill.Color = Color3.new(r2, g2, 0); d.HpFill.Visible = true
                            else d.HpBG.Visible = false; d.HpFill.Visible = false end

                            -- DIST
                            if Config.ShowDistance then
                                d.Dist.Position = Vector2.new(cx, bs.Y + 2)
                                d.Dist.Text = "[" .. tostring(math.floor(dist)) .. "]"
                                d.Dist.Color = color; d.Dist.Size = Config.TextSize - 1; d.Dist.Visible = true
                            else d.Dist.Visible = false end

                            -- TRACER
                            if Config.ShowTracers then
                                local vs = cam.ViewportSize
                                d.Tracer.From = Vector2.new(vs.X/2, vs.Y)
                                d.Tracer.To = Vector2.new(cx, bs.Y)
                                d.Tracer.Color = color; d.Tracer.Visible = true
                            else d.Tracer.Visible = false end
                        end
                    end
                end
            end
        end
    end
end

local RC = RunService.RenderStepped:Connect(function()
    local ok = pcall(Render)
end)

-- ══════════════════════════════════════════
--  PLAYER TRACKING — só eventos, sem polling
-- ══════════════════════════════════════════
local function SetupPlayer(player)
    if player == LocalPlayer then return end
    MakeDraw(player)
end

for _, p in ipairs(Players:GetPlayers()) do
    SetupPlayer(p)
end

Trk(Players.PlayerAdded:Connect(function(p)
    SetupPlayer(p)
end))

Trk(Players.PlayerRemoving:Connect(function(p)
    Kill(p)
end))

-- ══════════════════════════════════════════
--  GUI (compacto, mesmo visual)
-- ══════════════════════════════════════════
local SG = Instance.new("ScreenGui")
SG.Name = "PF_ESP_GUI"; SG.ZIndexBehavior = Enum.ZIndexBehavior.Sibling; SG.ResetOnSpawn = false
pcall(function()
    if gethui then SG.Parent = gethui()
    elseif syn and syn.protect_gui then syn.protect_gui(SG); SG.Parent = game:GetService("CoreGui")
    else SG.Parent = game:GetService("CoreGui") end
end)

local T = {
    BG=Color3.fromRGB(20,20,28), Top=Color3.fromRGB(30,30,42), El=Color3.fromRGB(35,35,50),
    Acc=Color3.fromRGB(100,120,255), Tx=Color3.fromRGB(220,220,230), Dim=Color3.fromRGB(140,140,160),
    On=Color3.fromRGB(80,200,120), Off=Color3.fromRGB(70,70,90),
    SBG=Color3.fromRGB(40,40,55), SF=Color3.fromRGB(100,120,255), Bd=Color3.fromRGB(50,50,70),
}

local MF=Instance.new("Frame"); MF.Size=UDim2.new(0,280,0,420); MF.Position=UDim2.new(0.5,-140,0.5,-210)
MF.BackgroundColor3=T.BG; MF.BorderSizePixel=0; MF.ClipsDescendants=true; MF.Parent=SG
Instance.new("UICorner",MF).CornerRadius=UDim.new(0,8)
local ms=Instance.new("UIStroke",MF); ms.Color=T.Bd; ms.Thickness=1

local TB=Instance.new("Frame"); TB.Size=UDim2.new(1,0,0,32); TB.BackgroundColor3=T.Top; TB.BorderSizePixel=0; TB.Parent=MF
Instance.new("UICorner",TB).CornerRadius=UDim.new(0,8)
local TP=Instance.new("Frame"); TP.Size=UDim2.new(1,0,0,10); TP.Position=UDim2.new(0,0,1,-10); TP.BackgroundColor3=T.Top; TP.BorderSizePixel=0; TP.Parent=TB
local TL=Instance.new("TextLabel"); TL.Size=UDim2.new(1,-60,1,0); TL.Position=UDim2.new(0,12,0,0); TL.BackgroundTransparency=1
TL.Text="PF ESP"; TL.TextColor3=T.Acc; TL.TextSize=14; TL.Font=Enum.Font.GothamBold; TL.TextXAlignment=Enum.TextXAlignment.Left; TL.Parent=TB
local MB=Instance.new("TextButton"); MB.Size=UDim2.new(0,28,0,22); MB.Position=UDim2.new(1,-36,0.5,-11)
MB.BackgroundColor3=T.El; MB.BorderSizePixel=0; MB.Text="-"; MB.TextColor3=T.Dim; MB.TextSize=14; MB.Font=Enum.Font.GothamBold; MB.Parent=TB
Instance.new("UICorner",MB).CornerRadius=UDim.new(0,4)

local CT=Instance.new("ScrollingFrame"); CT.Size=UDim2.new(1,-16,1,-40); CT.Position=UDim2.new(0,8,0,36)
CT.BackgroundTransparency=1; CT.BorderSizePixel=0; CT.ScrollBarThickness=3; CT.ScrollBarImageColor3=T.Acc
CT.CanvasSize=UDim2.new(0,0,0,0); CT.AutomaticCanvasSize=Enum.AutomaticSize.Y; CT.Parent=MF
Instance.new("UIListLayout",CT).SortOrder=Enum.SortOrder.LayoutOrder; CT:FindFirstChildOfClass("UIListLayout").Padding=UDim.new(0,4)

local LO=0; local function NO() LO=LO+1; return LO end

local function Sec(n)
    local f=Instance.new("Frame"); f.Size=UDim2.new(1,0,0,28); f.BackgroundTransparency=1; f.LayoutOrder=NO(); f.Parent=CT
    local l=Instance.new("TextLabel"); l.Size=UDim2.new(1,0,1,0); l.Position=UDim2.new(0,4,0,0); l.BackgroundTransparency=1
    l.Text=string.upper(n); l.TextColor3=T.Acc; l.TextSize=11; l.Font=Enum.Font.GothamBold; l.TextXAlignment=Enum.TextXAlignment.Left; l.Parent=f
    local ln=Instance.new("Frame"); ln.Size=UDim2.new(1,-8,0,1); ln.Position=UDim2.new(0,4,1,-1); ln.BackgroundColor3=T.Bd; ln.BorderSizePixel=0; ln.Parent=f
end

local function Tog(n,def,cb)
    local st=def
    local fr=Instance.new("Frame"); fr.Size=UDim2.new(1,0,0,32); fr.BackgroundColor3=T.El; fr.BorderSizePixel=0; fr.LayoutOrder=NO(); fr.Parent=CT
    Instance.new("UICorner",fr).CornerRadius=UDim.new(0,6)
    local lb=Instance.new("TextLabel"); lb.Size=UDim2.new(1,-60,1,0); lb.Position=UDim2.new(0,10,0,0); lb.BackgroundTransparency=1
    lb.Text=n; lb.TextColor3=T.Tx; lb.TextSize=12; lb.Font=Enum.Font.Gotham; lb.TextXAlignment=Enum.TextXAlignment.Left; lb.Parent=fr
    local bg=Instance.new("Frame"); bg.Size=UDim2.new(0,36,0,18); bg.Position=UDim2.new(1,-46,0.5,-9)
    bg.BackgroundColor3=st and T.On or T.Off; bg.BorderSizePixel=0; bg.Parent=fr
    Instance.new("UICorner",bg).CornerRadius=UDim.new(1,0)
    local kn=Instance.new("Frame"); kn.Size=UDim2.new(0,14,0,14)
    kn.Position=st and UDim2.new(1,-16,0.5,-7) or UDim2.new(0,2,0.5,-7)
    kn.BackgroundColor3=Color3.new(1,1,1); kn.BorderSizePixel=0; kn.Parent=bg
    Instance.new("UICorner",kn).CornerRadius=UDim.new(1,0)
    local bt=Instance.new("TextButton"); bt.Size=UDim2.new(1,0,1,0); bt.BackgroundTransparency=1; bt.Text=""; bt.Parent=fr
    bt.MouseButton1Click:Connect(function()
        st=not st
        local ti=TweenInfo.new(0.2,Enum.EasingStyle.Quad)
        TweenService:Create(bg,ti,{BackgroundColor3=st and T.On or T.Off}):Play()
        TweenService:Create(kn,ti,{Position=st and UDim2.new(1,-16,0.5,-7) or UDim2.new(0,2,0.5,-7)}):Play()
        cb(st)
    end)
end

local function Sld(n,mn,mx,def,cb)
    local val=def
    local fr=Instance.new("Frame"); fr.Size=UDim2.new(1,0,0,46); fr.BackgroundColor3=T.El; fr.BorderSizePixel=0; fr.LayoutOrder=NO(); fr.Parent=CT
    Instance.new("UICorner",fr).CornerRadius=UDim.new(0,6)
    local lb=Instance.new("TextLabel"); lb.Size=UDim2.new(1,-60,0,20); lb.Position=UDim2.new(0,10,0,2); lb.BackgroundTransparency=1
    lb.Text=n; lb.TextColor3=T.Tx; lb.TextSize=12; lb.Font=Enum.Font.Gotham; lb.TextXAlignment=Enum.TextXAlignment.Left; lb.Parent=fr
    local vl=Instance.new("TextLabel"); vl.Size=UDim2.new(0,50,0,20); vl.Position=UDim2.new(1,-56,0,2); vl.BackgroundTransparency=1
    vl.Text=tostring(math.floor(val)); vl.TextColor3=T.Acc; vl.TextSize=12; vl.Font=Enum.Font.GothamBold; vl.TextXAlignment=Enum.TextXAlignment.Right; vl.Parent=fr
    local sb=Instance.new("Frame"); sb.Size=UDim2.new(1,-20,0,6); sb.Position=UDim2.new(0,10,0,30); sb.BackgroundColor3=T.SBG; sb.BorderSizePixel=0; sb.Parent=fr
    Instance.new("UICorner",sb).CornerRadius=UDim.new(1,0)
    local sf=Instance.new("Frame"); sf.Size=UDim2.new((val-mn)/(mx-mn),0,1,0); sf.BackgroundColor3=T.SF; sf.BorderSizePixel=0; sf.Parent=sb
    Instance.new("UICorner",sf).CornerRadius=UDim.new(1,0)
    local kn=Instance.new("Frame"); kn.Size=UDim2.new(0,12,0,12); kn.AnchorPoint=Vector2.new(0.5,0.5)
    kn.Position=UDim2.new((val-mn)/(mx-mn),0,0.5,0); kn.BackgroundColor3=Color3.new(1,1,1); kn.BorderSizePixel=0; kn.ZIndex=2; kn.Parent=sb
    Instance.new("UICorner",kn).CornerRadius=UDim.new(1,0)
    local dr=false
    local ib=Instance.new("TextButton"); ib.Size=UDim2.new(1,0,0,20); ib.Position=UDim2.new(0,0,0,24); ib.BackgroundTransparency=1; ib.Text=""; ib.Parent=fr
    local function Up(x)
        local rel=math.clamp((x-sb.AbsolutePosition.X)/sb.AbsoluteSize.X,0,1)
        val=math.floor(mn+(mx-mn)*rel); sf.Size=UDim2.new(rel,0,1,0); kn.Position=UDim2.new(rel,0,0.5,0)
        vl.Text=tostring(val); cb(val)
    end
    ib.MouseButton1Down:Connect(function() dr=true end)
    Trk(UserInputService.InputChanged:Connect(function(i) if dr and i.UserInputType==Enum.UserInputType.MouseMovement then Up(i.Position.X) end end))
    Trk(UserInputService.InputEnded:Connect(function(i) if i.UserInputType==Enum.UserInputType.MouseButton1 then dr=false end end))
    ib.MouseButton1Click:Connect(function() Up(UserInputService:GetMouseLocation().X) end)
end

local function CPk(n,def,cb)
    local ps={
        Color3.fromRGB(255,50,50),Color3.fromRGB(255,120,30),Color3.fromRGB(255,220,50),
        Color3.fromRGB(80,200,120),Color3.fromRGB(50,180,255),Color3.fromRGB(100,120,255),
        Color3.fromRGB(180,80,255),Color3.fromRGB(255,80,180),Color3.fromRGB(255,255,255),Color3.fromRGB(200,200,200),
    }
    local fr=Instance.new("Frame"); fr.Size=UDim2.new(1,0,0,52); fr.BackgroundColor3=T.El; fr.BorderSizePixel=0; fr.LayoutOrder=NO(); fr.Parent=CT
    Instance.new("UICorner",fr).CornerRadius=UDim.new(0,6)
    local lb=Instance.new("TextLabel"); lb.Size=UDim2.new(1,-20,0,20); lb.Position=UDim2.new(0,10,0,2); lb.BackgroundTransparency=1
    lb.Text=n; lb.TextColor3=T.Tx; lb.TextSize=12; lb.Font=Enum.Font.Gotham; lb.TextXAlignment=Enum.TextXAlignment.Left; lb.Parent=fr
    local sel=Instance.new("Frame"); sel.Size=UDim2.new(0,14,0,14); sel.Position=UDim2.new(1,-28,0,5)
    sel.BackgroundColor3=def; sel.BorderSizePixel=0; sel.Parent=fr
    Instance.new("UICorner",sel).CornerRadius=UDim.new(1,0)
    Instance.new("UIStroke",sel).Color=Color3.new(1,1,1); sel:FindFirstChildOfClass("UIStroke").Thickness=1.5
    for i,c in ipairs(ps) do
        local sw=Instance.new("TextButton"); sw.Size=UDim2.new(0,20,0,16); sw.Position=UDim2.new(0,10+(i-1)*24,0,28)
        sw.BackgroundColor3=c; sw.BorderSizePixel=0; sw.Text=""; sw.Parent=fr
        Instance.new("UICorner",sw).CornerRadius=UDim.new(0,4)
        sw.MouseButton1Click:Connect(function() sel.BackgroundColor3=c; cb(c) end)
    end
end

-- drag
do
    local dg,ds,sp
    TB.InputBegan:Connect(function(i) if i.UserInputType==Enum.UserInputType.MouseButton1 then dg=true;ds=i.Position;sp=MF.Position end end)
    Trk(UserInputService.InputChanged:Connect(function(i)
        if dg and i.UserInputType==Enum.UserInputType.MouseMovement then
            local d=i.Position-ds; MF.Position=UDim2.new(sp.X.Scale,sp.X.Offset+d.X,sp.Y.Scale,sp.Y.Offset+d.Y)
        end
    end))
    Trk(UserInputService.InputEnded:Connect(function(i) if i.UserInputType==Enum.UserInputType.MouseButton1 then dg=false end end))
end

local exp=true
MB.MouseButton1Click:Connect(function()
    exp=not exp
    TweenService:Create(MF,TweenInfo.new(0.25,Enum.EasingStyle.Quad),{Size=exp and UDim2.new(0,280,0,420) or UDim2.new(0,280,0,32)}):Play()
    MB.Text=exp and "-" or "+"
end)

Sec("Geral")
Tog("ESP Ativo",Config.Enabled,function(v) Config.Enabled=v end)
Tog("Team Check",Config.TeamCheck,function(v) Config.TeamCheck=v end)
Sec("Elementos")
Tog("Boxes",Config.ShowBox,function(v) Config.ShowBox=v end)
Tog("Nomes",Config.ShowName,function(v) Config.ShowName=v end)
Tog("Barra de HP",Config.ShowHealth,function(v) Config.ShowHealth=v end)
Tog("Distancia",Config.ShowDistance,function(v) Config.ShowDistance=v end)
Tog("Tracers",Config.ShowTracers,function(v) Config.ShowTracers=v end)
Sec("Ajustes")
Sld("Distancia Max",100,5000,Config.MaxDistance,function(v) Config.MaxDistance=v end)
Sld("Tamanho Texto",8,24,Config.TextSize,function(v) Config.TextSize=v end)
Sld("Espessura Box",1,5,math.floor(Config.BoxThickness),function(v) Config.BoxThickness=v end)
Sec("Cores")
CPk("Cor Inimigo",Config.EnemyColor,function(v) Config.EnemyColor=v end)
CPk("Cor Aliado",Config.TeamColor,function(v) Config.TeamColor=v end)

Trk(UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode==Enum.KeyCode.Insert then Config.Enabled=not Config.Enabled
    elseif input.KeyCode==Enum.KeyCode.RightShift then SG.Enabled=not SG.Enabled end
end))

local function Cleanup()
    if RC then RC:Disconnect() end
    for _,c in ipairs(Conns) do pcall(function() c:Disconnect() end) end
    for p in pairs(Draw) do Kill(p) end
    Draw={}
    if SG then SG:Destroy() end
end

getgenv().PF_ESP = {Config=Config, Cleanup=Cleanup, GUI=SG}
