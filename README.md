local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
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
    TeamCheck     = true,       -- true = só inimigos
    MaxDistance    = 2000,       -- studs
    EnemyColor    = Color3.fromRGB(255, 50, 50),
    TeamColor     = Color3.fromRGB(50, 255, 50),
    BoxThickness  = 1.4,
    TextSize      = 13,
}
-- ══════════════════════════════════════════
--  DRAWING CACHE PER PLAYER
-- ══════════════════════════════════════════
local ESPObjects = {}
local function CreateESP(player)
    local esp = {}
    -- box (4 lines)
    esp.BoxLines = {}
    for i = 1, 4 do
        local line = Drawing.new("Line")
        line.Visible = false
        line.Thickness = Config.BoxThickness
        line.Color = Config.EnemyColor
        esp.BoxLines[i] = line
    end
    -- name
    esp.Name = Drawing.new("Text")
    esp.Name.Center = true
    esp.Name.Outline = true
    esp.Name.Size = Config.TextSize
    esp.Name.Visible = false
    -- health bar (background + fill)
    esp.HealthBG = Drawing.new("Line")
    esp.HealthBG.Thickness = 3
    esp.HealthBG.Color = Color3.fromRGB(0, 0, 0)
    esp.HealthBG.Visible = false
    esp.HealthFill = Drawing.new("Line")
    esp.HealthFill.Thickness = 1.5
    esp.HealthFill.Visible = false
    -- distance
    esp.Distance = Drawing.new("Text")
    esp.Distance.Center = true
    esp.Distance.Outline = true
    esp.Distance.Size = Config.TextSize - 1
    esp.Distance.Visible = false
    -- tracer
    esp.Tracer = Drawing.new("Line")
    esp.Tracer.Thickness = 1
    esp.Tracer.Visible = false
    ESPObjects[player] = esp
end
local function RemoveESP(player)
    local esp = ESPObjects[player]
    if not esp then return end
    for _, line in ipairs(esp.BoxLines) do
        line:Remove()
    end
    esp.Name:Remove()
    esp.HealthBG:Remove()
    esp.HealthFill:Remove()
    esp.Distance:Remove()
    esp.Tracer:Remove()
    ESPObjects[player] = nil
end
local function HideESP(esp)
    for _, line in ipairs(esp.BoxLines) do
        line.Visible = false
    end
    esp.Name.Visible = false
    esp.HealthBG.Visible = false
    esp.HealthFill.Visible = false
    esp.Distance.Visible = false
    esp.Tracer.Visible = false
end
-- ══════════════════════════════════════════
--  TEAM DETECTION (Phantom Forces)
-- ══════════════════════════════════════════
local function GetPlayerTeam(player)
    -- PF usa TeamColor ou o valor do atributo
    return player.Team
end
local function IsEnemy(player)
    if not Config.TeamCheck then return true end
    local myTeam = GetPlayerTeam(LocalPlayer)
    local theirTeam = GetPlayerTeam(player)
    if not myTeam or not theirTeam then return true end
    return myTeam ~= theirTeam
end
local function GetColor(player)
    return IsEnemy(player) and Config.EnemyColor or Config.TeamColor
end
-- ══════════════════════════════════════════
--  HUMANOID / CHARACTER UTILS
-- ══════════════════════════════════════════
local function GetCharacter(player)
    local char = player.Character
    if not char then return nil end
    -- PF às vezes esconde personagens em pastas dentro de workspace
    -- fallback: busca direto
    if not char:IsDescendantOf(workspace) then return nil end
    return char
end
local function GetHumanoid(char)
    return char:FindFirstChildOfClass("Humanoid")
end
local function GetRootPart(char)
    return char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Torso") or char:FindFirstChild("Head")
end
local function GetHealthFraction(humanoid)
    if humanoid.MaxHealth <= 0 then return 0 end
    return math.clamp(humanoid.Health / humanoid.MaxHealth, 0, 1)
end
-- ══════════════════════════════════════════
--  BOUNDING BOX 2D
-- ══════════════════════════════════════════
local function GetBoundingBox(char)
    local root = GetRootPart(char)
    if not root then return nil end
    local pos = root.Position
    local dist = (Camera.CFrame.Position - pos).Magnitude
    -- escala box baseada na distância projetada
    local headPos = pos + Vector3.new(0, 3, 0)
    local footPos = pos - Vector3.new(0, 3, 0)
    local headScreen, headVisible = Camera:WorldToViewportPoint(headPos)
    local footScreen, footVisible = Camera:WorldToViewportPoint(footPos)
    if not headVisible and not footVisible then return nil end
    local height = math.abs(footScreen.Y - headScreen.Y)
    local width = height * 0.55
    local centerX = (headScreen.X + footScreen.X) / 2
    local centerY = (headScreen.Y + footScreen.Y) / 2
    return {
        TopLeft     = Vector2.new(centerX - width / 2, headScreen.Y),
        TopRight    = Vector2.new(centerX + width / 2, headScreen.Y),
        BottomLeft  = Vector2.new(centerX - width / 2, footScreen.Y),
        BottomRight = Vector2.new(centerX + width / 2, footScreen.Y),
        Center      = Vector2.new(centerX, centerY),
        Width       = width,
        Height      = height,
        Distance    = dist,
    }
end
-- ══════════════════════════════════════════
--  RENDER LOOP
-- ══════════════════════════════════════════
local Connection
Connection = RunService.RenderStepped:Connect(function()
    if not Config.Enabled then
        for _, esp in pairs(ESPObjects) do
            HideESP(esp)
        end
        return
    end
    for player, esp in pairs(ESPObjects) do
        local char = GetCharacter(player)
        local humanoid = char and GetHumanoid(char)
        local root = char and GetRootPart(char)
        if not char or not humanoid or not root or humanoid.Health <= 0 then
            HideESP(esp)
            continue
        end
        local dist = (Camera.CFrame.Position - root.Position).Magnitude
        if dist > Config.MaxDistance then
            HideESP(esp)
            continue
        end
        local bb = GetBoundingBox(char)
        if not bb then
            HideESP(esp)
            continue
        end
        local color = GetColor(player)
        -- se TeamCheck ativo e é aliado, esconde
        if Config.TeamCheck and not IsEnemy(player) then
            HideESP(esp)
            continue
        end
        -- BOX
        if Config.ShowBox then
            esp.BoxLines[1].From = bb.TopLeft
            esp.BoxLines[1].To = bb.TopRight
            esp.BoxLines[2].From = bb.TopRight
            esp.BoxLines[2].To = bb.BottomRight
            esp.BoxLines[3].From = bb.BottomRight
            esp.BoxLines[3].To = bb.BottomLeft
            esp.BoxLines[4].From = bb.BottomLeft
            esp.BoxLines[4].To = bb.TopLeft
            for i = 1, 4 do
                esp.BoxLines[i].Color = color
                esp.BoxLines[i].Visible = true
            end
        else
            for i = 1, 4 do esp.BoxLines[i].Visible = false end
        end
        -- NAME
        if Config.ShowName then
            esp.Name.Position = Vector2.new(bb.Center.X, bb.TopLeft.Y - Config.TextSize - 2)
            esp.Name.Text = player.DisplayName or player.Name
            esp.Name.Color = color
            esp.Name.Visible = true
        else
            esp.Name.Visible = false
        end
        -- HEALTH BAR (barra vertical à esquerda do box)
        if Config.ShowHealth then
            local frac = GetHealthFraction(humanoid)
            local barX = bb.TopLeft.X - 5
            local barTop = bb.TopLeft.Y
            local barBottom = bb.BottomLeft.Y
            local fillBottom = barBottom
            local fillTop = barBottom - (barBottom - barTop) * frac
            esp.HealthBG.From = Vector2.new(barX, barTop)
            esp.HealthBG.To = Vector2.new(barX, barBottom)
            esp.HealthBG.Visible = true
            esp.HealthFill.From = Vector2.new(barX, fillTop)
            esp.HealthFill.To = Vector2.new(barX, fillBottom)
            -- gradiente: verde(100%) → amarelo(50%) → vermelho(0%)
            local r = frac < 0.5 and 1 or (1 - (frac - 0.5) * 2)
            local g = frac > 0.5 and 1 or (frac * 2)
            esp.HealthFill.Color = Color3.new(r, g, 0)
            esp.HealthFill.Visible = true
        else
            esp.HealthBG.Visible = false
            esp.HealthFill.Visible = false
        end
        -- DISTANCE
        if Config.ShowDistance then
            esp.Distance.Position = Vector2.new(bb.Center.X, bb.BottomLeft.Y + 2)
            esp.Distance.Text = string.format("[%d studs]", math.floor(bb.Distance))
            esp.Distance.Color = color
            esp.Distance.Visible = true
        else
            esp.Distance.Visible = false
        end
        -- TRACER (bottom center da tela → pés do player)
        if Config.ShowTracers then
            local viewportSize = Camera.ViewportSize
            esp.Tracer.From = Vector2.new(viewportSize.X / 2, viewportSize.Y)
            esp.Tracer.To = Vector2.new(bb.Center.X, bb.BottomLeft.Y)
            esp.Tracer.Color = color
            esp.Tracer.Visible = true
        else
            esp.Tracer.Visible = false
        end
    end
end)
-- ══════════════════════════════════════════
--  PLAYER JOIN / LEAVE
-- ══════════════════════════════════════════
for _, player in ipairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then
        CreateESP(player)
    end
end
Players.PlayerAdded:Connect(function(player)
    if player ~= LocalPlayer then
        CreateESP(player)
    end
end)
Players.PlayerRemoving:Connect(function(player)
    RemoveESP(player)
end)
-- ══════════════════════════════════════════
--  TOGGLE (tecla INSERT)
-- ══════════════════════════════════════════
local UserInputService = game:GetService("UserInputService")
UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == Enum.KeyCode.Insert then
        Config.Enabled = not Config.Enabled
    end
end)
-- ══════════════════════════════════════════
--  CLEANUP
-- ══════════════════════════════════════════
-- desconecta se o script for recarregado
local function Cleanup()
    if Connection then Connection:Disconnect() end
    for player in pairs(ESPObjects) do
        RemoveESP(player)
    end
end
-- expõe globalmente pra controle externo
getgenv().PF_ESP = {
    Config  = Config,
    Cleanup = Cleanup,
    Toggle  = function() Config.Enabled = not Config.Enabled end,
}
