local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer

-- CONFIG
local BASE_TRANSPARENCY = 0.25
local MAX_DISTANCE = 250
local HEALTHBAR_WIDTH = 3
local HEALTHBAR_GAP = 3
local TWEEN_TIME = 0.20

-- STATE
local espEnabled = true
local teamCheckEnabled = false
local colorMode = "COLOR" -- "COLOR" or "STEALTH"

-- helpers
local function safeTween(obj, props, dur, style, dir)
    dur = dur or TWEEN_TIME
    style = style or Enum.EasingStyle.Quad
    dir = dir or Enum.EasingDirection.Out
    local ok, tween = pcall(function()
        return TweenService:Create(obj, TweenInfo.new(dur, style, dir), props)
    end)
    if ok and tween then
        tween:Play()
    end
end

local function clamp(v, a, b) return math.max(a, math.min(b, v)) end

-- ensure PlayerGui
local playerGui = LocalPlayer:WaitForChild("PlayerGui")

-- SCREEN GUI
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "KIWI_ESP_UI"
screenGui.ResetOnSpawn = false
screenGui.IgnoreGuiInset = true
screenGui.Parent = playerGui

-- TWEEN helper
local function tween(obj, props, dur)
    safeTween(obj, props, dur)
end

-- DRAG helper
local function makeDraggable(frame)
    local dragging, dragInput, dragStart, startPos
    frame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragInput = input
            dragStart = input.Position
            startPos = frame.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)
    frame.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and input == dragInput and dragStart and startPos then
            local delta = input.Position - dragStart
            frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end


---

-- UI: Panel

local uiFrame = Instance.new("Frame")
uiFrame.Name = "ESP_Panel"
uiFrame.Size = UDim2.new(0, 180, 0, 170)
uiFrame.Position = UDim2.new(0, 12, 0, 12)
uiFrame.BackgroundColor3 = Color3.fromRGB(22, 22, 28)
uiFrame.BorderSizePixel = 0
uiFrame.ZIndex = 50
uiFrame.Parent = screenGui
makeDraggable(uiFrame)

local uiCorner = Instance.new("UICorner", uiFrame)
uiCorner.CornerRadius = UDim.new(0, 10)
local uiGradient = Instance.new("UIGradient", uiFrame)
uiGradient.Rotation = 45

local stealthSeq = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(120,120,120)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(170,170,170))
})
task.spawn(function()
    while true do
        if colorMode == "COLOR" then
            local t = tick() * 0.16
            local c1 = Color3.fromHSV((t) % 1, 1, 1)
            local c2 = Color3.fromHSV((t + 0.33) % 1, 1, 1)
            local c3 = Color3.fromHSV((t + 0.66) % 1, 1, 1)
            uiGradient.Color = ColorSequence.new({
                ColorSequenceKeypoint.new(0, c1),
                ColorSequenceKeypoint.new(0.5, c2),
                ColorSequenceKeypoint.new(1, c3)
            })
        else
            uiGradient.Color = stealthSeq
        end
        task.wait(0.06)
    end
end)

-- Title
local title = Instance.new("TextLabel", uiFrame)
title.Size = UDim2.new(1, -16, 0, 26)
title.Position = UDim2.new(0, 8, 0, 6)
title.BackgroundTransparency = 1
title.Text = "🥝 KIWI PAINEL"
title.Font = Enum.Font.GothamBold
title.TextSize = 16
title.TextColor3 = Color3.fromRGB(255,255,255)
title.TextXAlignment = Enum.TextXAlignment.Left
title.ZIndex = 55

local minimizeBtn = Instance.new("TextButton", uiFrame)
minimizeBtn.Size = UDim2.new(0, 28, 0, 20)
minimizeBtn.Position = UDim2.new(1, -36, 0, 6)
minimizeBtn.BackgroundTransparency = 1
minimizeBtn.Text = "-"
minimizeBtn.Font = Enum.Font.GothamBold
minimizeBtn.TextSize = 20
minimizeBtn.TextColor3 = Color3.fromRGB(255,255,255)
minimizeBtn.ZIndex = 56

local function createButton(text, y)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(0, 156, 0, 36)
    b.Position = UDim2.new(0, 12, 0, y)
    b.BackgroundColor3 = Color3.fromRGB(34,34,40)
    b.BorderSizePixel = 0
    b.Font = Enum.Font.GothamBold
    b.TextSize = 14
    b.TextColor3 = Color3.fromRGB(255,255,255)
    b.Text = text
    b.ZIndex = 55
    b.Parent = uiFrame
    local c = Instance.new("UICorner", b)
    c.CornerRadius = UDim.new(0,6)
    b.MouseEnter:Connect(function() tween(b, {BackgroundColor3 = Color3.fromRGB(50,50,60)}, 0.12) end)
    b.MouseLeave:Connect(function() tween(b, {BackgroundColor3 = Color3.fromRGB(34,34,40)}, 0.12) end)
    return b
end

local espBtn = createButton("🟢 ESP: ON", 40)
local modeBtn = createButton("🎨 Modo: Color", 82)
local teamBtn = createButton("👥 Team Check: OFF", 124)

local clickSound = Instance.new("Sound", uiFrame)
clickSound.SoundId = "rbxassetid://9118823104"
clickSound.Volume = 0.5


---

-- MINI ICON (🥝 sem bola)

local miniKiwi = Instance.new("TextButton")
miniKiwi.Name = "MiniKiwi"
miniKiwi.Size = UDim2.new(0, 44, 0, 44)
miniKiwi.Position = UDim2.new(0, 20, 0, 220)
miniKiwi.BackgroundTransparency = 1
miniKiwi.BorderSizePixel = 0
miniKiwi.Visible = false
miniKiwi.Text = "🥝"
miniKiwi.Font = Enum.Font.GothamBold
miniKiwi.TextScaled = true
miniKiwi.TextColor3 = Color3.new(1,1,1)
miniKiwi.ZIndex = 80
miniKiwi.Parent = screenGui
makeDraggable(miniKiwi)

local pop = Instance.new("Sound", miniKiwi)
pop.SoundId = "rbxassetid://9118823104"
pop.Volume = 0.6

-- animação RGB no kiwi (mantive visual dinâmico para o ícone)
task.spawn(function()
    while true do
        if colorMode == "COLOR" then
            local t = tick() * 0.2
            local c1 = Color3.fromHSV((t) % 1, 1, 1)
            local c2 = Color3.fromHSV((t + 0.33) % 1, 1, 1)
            local c3 = Color3.fromHSV((t + 0.66) % 1, 1, 1)
            miniKiwi.TextColor3 = c1:Lerp(c2, 0.5):Lerp(c3, 0.5)
        else
            miniKiwi.TextColor3 = Color3.fromRGB(200,200,200)
        end
        task.wait(0.06)
    end
end)

-- minimize / restore
local minimized = false
local function minimizePanel()
    if minimized then return end
    minimized = true
    safeTween(uiFrame, {Size = UDim2.new(0, 0, 0, 0)}, 0.18)
    task.wait(0.18)
    uiFrame.Visible = false
    miniKiwi.Visible = true
    safeTween(miniKiwi, {Size = UDim2.new(0, 44, 0, 44)}, 0.3, Enum.EasingStyle.Back)
    task.spawn(function() task.wait(0.06); pcall(function() pop:Play() end) end)
end

local function restorePanel()
    if not minimized then return end
    minimized = false
    miniKiwi.Visible = false
    uiFrame.Visible = true
    safeTween(uiFrame, {Size = UDim2.new(0, 180, 0, 170)}, 0.24, Enum.EasingStyle.Back)
end

minimizeBtn.MouseButton1Click:Connect(function()
    minimizePanel()
end)
miniKiwi.MouseButton1Click:Connect(function()
    restorePanel()
end)


---

-- Buttons behavior

espBtn.MouseButton1Click:Connect(function()
    espEnabled = not espEnabled
    espBtn.Text = espEnabled and "🟢 ESP: ON" or "🔴 ESP: OFF"
end)

modeBtn.MouseButton1Click:Connect(function()
    if colorMode == "COLOR" then
        colorMode = "STEALTH"
        modeBtn.Text = "🎨 Modo: Stealth"
    else
        colorMode = "COLOR"
        modeBtn.Text = "🎨 Modo: Color"
    end
end)

teamBtn.MouseButton1Click:Connect(function()
    teamCheckEnabled = not teamCheckEnabled
    teamBtn.Text = teamCheckEnabled and "👥 Team Check: ON" or "👥 Team Check: OFF"
end)


---

-- ESP objects

local function createESPForPlayer(p)
    local container = Instance.new("Frame", screenGui)
    container.Name = "ESP_"..p.Name
    container.BackgroundTransparency = 1
    container.Visible = false
    container.ZIndex = 40

    local box = Instance.new("Frame", container)
    box.Name = "Box"
    box.Size = UDim2.new(1,0,1,0)
    box.BorderSizePixel = 0
    box.BackgroundTransparency = BASE_TRANSPARENCY
    box.Parent = container

    local grad = Instance.new("UIGradient", box)
    grad.Rotation = 90

    local stroke = Instance.new("UIStroke", box)
    stroke.Thickness = 1
    stroke.Transparency = 0.8
    stroke.Color = Color3.fromRGB(255,255,255)

    local hbBG = Instance.new("Frame", box)
    hbBG.Name = "HealthBG"
    hbBG.BackgroundColor3 = Color3.fromRGB(20,20,20)
    hbBG.BackgroundTransparency = 0.65
    hbBG.BorderSizePixel = 0
    hbBG.ZIndex = 41

    local hb = Instance.new("Frame", hbBG)
    hb.Name = "HealthBar"
    hb.BorderSizePixel = 0
    hb.BackgroundTransparency = 0.3
    hb.Size = UDim2.new(1,0,1,0)
    hb.ZIndex = 42

    return {
        Player = p,
        Container = container,
        Box = box,
        Grad = grad,
        Stroke = stroke,
        HealthBG = hbBG,
        HealthBar = hb,
        LastHealth = 1
    }

end

local espObjects = {}
for _,p in ipairs(Players:GetPlayers()) do
    if p ~= LocalPlayer then espObjects[p] = createESPForPlayer(p) end
end
Players.PlayerAdded:Connect(function(p)
    if p ~= LocalPlayer then espObjects[p] = createESPForPlayer(p) end
end)
Players.PlayerRemoving:Connect(function(p)
    if espObjects[p] then espObjects[p].Container:Destroy(); espObjects[p] = nil end
end)

local function getHealthColor(r)
    if colorMode == "STEALTH" then
        return Color3.new(1,1,1)
    else
        if r >= 0.7 then return Color3.fromRGB(0,255,0)
        elseif r >= 0.35 then return Color3.fromRGB(255,255,0)
        else return Color3.fromRGB(255,0,0) end
    end
end

RunService.RenderStepped:Connect(function()
    if not espEnabled then
        for _,obj in pairs(espObjects) do obj.Container.Visible = false end
        return
    end

    for plr, obj in pairs(espObjects) do
        local char = plr.Character
        local hrp = char and (char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Torso") or char:FindFirstChild("UpperTorso"))
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if char and hrp and hum and hum.Health > 0 then
            if teamCheckEnabled and LocalPlayer.Team and plr.Team == LocalPlayer.Team then
                obj.Container.Visible = false
            else
                local vpPos, onScreen = Camera:WorldToViewportPoint(hrp.Position)
                if onScreen then
                    local dist = (Camera.CFrame.Position - hrp.Position).Magnitude
                    if dist <= MAX_DISTANCE then
                        local height3D, width3D = 4.5, 1.5
                        local topPos = Camera:WorldToViewportPoint(hrp.Position + Vector3.new(0, height3D/2, 0))
                        local bottomPos = Camera:WorldToViewportPoint(hrp.Position - Vector3.new(0, height3D/2, 0))
                        local boxH = math.max(18, math.abs(topPos.Y - bottomPos.Y))
                        local boxW = math.max(8, boxH / (height3D / width3D))

                        obj.Container.Size = UDim2.new(0, boxW, 0, boxH)
                        obj.Container.Position = UDim2.new(0, vpPos.X, 0, vpPos.Y)
                        obj.Container.AnchorPoint = Vector2.new(0.5, 0.5)
                        obj.Container.Visible = true

                        if colorMode == "STEALTH" then
                            -- Both teams show white in stealth
                            obj.Box.BackgroundColor3 = Color3.fromRGB(255,255,255)
                            obj.Box.BackgroundTransparency = 0.62
                            obj.Stroke.Transparency = 0.75
                            obj.Stroke.Color = Color3.fromRGB(255,255,255)
                            obj.Grad.Color = ColorSequence.new({
                                ColorSequenceKeypoint.new(0, Color3.fromRGB(250,250,250)),
                                ColorSequenceKeypoint.new(1, Color3.fromRGB(245,245,245))
                            })
                        else
                            -- Modo COLOR com ajuste das cores da box e barra de vida RGB translúcida fixa

                            local isAlly = false
                            if LocalPlayer.Team and plr.Team and LocalPlayer.Team == plr.Team then
                                isAlly = true
                            end

                            -- Cores da box mais escuras para amigo e inimigo
                            local boxColor = isAlly and Color3.fromRGB(0, 100, 0) or Color3.fromRGB(139, 0, 0)
                            obj.Box.BackgroundColor3 = boxColor
                            obj.Box.BackgroundTransparency = BASE_TRANSPARENCY
                            obj.Stroke.Transparency = 0.8
                            obj.Stroke.Color = boxColor

                            -- Gradiente da box: ligeiramente mais claro que a cor base
                            local lighter = boxColor:Lerp(Color3.fromRGB(255, 255, 255), 0.15)
                            obj.Grad.Color = ColorSequence.new({
                                ColorSequenceKeypoint.new(0, boxColor),
                                ColorSequenceKeypoint.new(1, lighter)
                            })

                            -- Barra de vida com cor RGB translúcida fixa, independente do time
                            local t = tick() * 0.8
                            local rgbColor = Color3.fromHSV((t) % 1, 1, 1)
                            local translucentRgb = Color3.new(rgbColor.R, rgbColor.G, rgbColor.B)
                            obj.HealthBar.BackgroundColor3 = translucentRgb
                            obj.HealthBar.BackgroundTransparency = 0.5 -- deixa translúcida
                        end

                        local healthRatio = clamp(hum.Health / hum.MaxHealth, 0, 1)
                        safeTween(obj.HealthBar, {Size = UDim2.new(1,0,healthRatio,0), Position = UDim2.new(0,0,1-healthRatio,0)}, 0.14)
                        local barW = clamp(HEALTHBAR_WIDTH * clamp(1 - (dist / MAX_DISTANCE), 0, 1) + 1.2, 1.2, HEALTHBAR_WIDTH)
                        obj.HealthBG.Size = UDim2.new(0, barW, 1, 0)
                        obj.HealthBG.Position = UDim2.new(1, HEALTHBAR_GAP, 0, 0)
                    else
                        obj.Container.Visible = false
                    end
                else
                    obj.Container.Visible = false
                end
            end
        else
            obj.Container.Visible = false
        end
    end

end)

print("[KIWI ESP] Final script loaded.")
