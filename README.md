local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer

-- CONFIGURAÇÕES
local SHOW_FOR_TEAMMATES = false
local BASE_TRANSPARENCY = 0.25 -- mais sólido
local MAX_DISTANCE = 250
local HEALTHBAR_WIDTH = 5
local HEALTHBAR_GAP = 3
local TWEEN_TIME = 0.25

-- GUI
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "DevESP_TeamDynamic"
screenGui.IgnoreGuiInset = true
screenGui.ResetOnSpawn = false
screenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

-----------------------------------------------------
-- 🌈 BOTÃO ESP PROFISSIONAL (Glow RGB + som + animação)
-----------------------------------------------------

local toggleBtn = Instance.new("TextButton")
toggleBtn.Name = "ESP_Toggle"
toggleBtn.Size = UDim2.new(0, 140, 0, 38)
toggleBtn.Position = UDim2.new(0, 10, 0, 10)
toggleBtn.BackgroundColor3 = Color3.fromRGB(15, 15, 20)
toggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
toggleBtn.Text = "🌟 ESP: ON"
toggleBtn.Font = Enum.Font.GothamBold
toggleBtn.TextSize = 17
toggleBtn.AutoButtonColor = false
toggleBtn.Active = true
toggleBtn.Draggable = true
toggleBtn.BorderSizePixel = 0
toggleBtn.SelectionImageObject = nil
toggleBtn.ZIndex = 3
toggleBtn.Parent = screenGui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 10)
corner.Parent = toggleBtn

local gradient = Instance.new("UIGradient")
gradient.Rotation = 45
gradient.Color = ColorSequence.new{
	ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 60, 90)),
	ColorSequenceKeypoint.new(0.5, Color3.fromRGB(180, 90, 255)),
	ColorSequenceKeypoint.new(1, Color3.fromRGB(0, 180, 255))
}
gradient.Parent = toggleBtn

local glow = Instance.new("ImageLabel")
glow.Name = "RGB_Glow"
glow.AnchorPoint = Vector2.new(0.5, 0.5)
glow.Position = UDim2.new(0.5, 0, 0.5, 0)
glow.Size = UDim2.new(1.3, 0, 1.8, 0)
glow.Image = "rbxassetid://4996891970"
glow.ImageTransparency = 0.7
glow.BackgroundTransparency = 1
glow.ZIndex = 1
glow.Parent = toggleBtn

task.spawn(function()
	while task.wait(0.05) do
		local t = tick() * 0.15
		local c1 = Color3.fromHSV((t) % 1, 1, 1)
		local c2 = Color3.fromHSV((t + 0.33) % 1, 1, 1)
		local c3 = Color3.fromHSV((t + 0.66) % 1, 1, 1)
		
		gradient.Color = ColorSequence.new{
			ColorSequenceKeypoint.new(0, c1),
			ColorSequenceKeypoint.new(0.5, c2),
			ColorSequenceKeypoint.new(1, c3)
		}
		glow.ImageColor3 = c2
	end
end)

local function tween(obj, props, dur)
	TweenService:Create(obj, TweenInfo.new(dur, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), props):Play()
end

toggleBtn.MouseEnter:Connect(function()
	tween(toggleBtn, {BackgroundColor3 = Color3.fromRGB(30, 30, 40)}, 0.15)
	tween(glow, {ImageTransparency = 0.55}, 0.15)
end)
toggleBtn.MouseLeave:Connect(function()
	tween(toggleBtn, {BackgroundColor3 = Color3.fromRGB(15, 15, 20)}, 0.15)
	tween(glow, {ImageTransparency = 0.7}, 0.15)
end)

local clickSound = Instance.new("Sound")
clickSound.SoundId = "rbxassetid://9118823104"
clickSound.Volume = 0.4
clickSound.PlayOnRemove = true
clickSound.Name = "ClickSound"
clickSound.Parent = toggleBtn

local espEnabled = true
toggleBtn.MouseButton1Click:Connect(function()
	espEnabled = not espEnabled
	toggleBtn.Text = espEnabled and "🥝 ESP: ON" or "🌑 ESP: OFF"

	local s = clickSound:Clone()
	s.Parent = toggleBtn
	s:Destroy()

	tween(toggleBtn, {Size = UDim2.new(0, 147, 0, 40)}, 0.1)
	task.wait(0.1)
	tween(toggleBtn, {Size = UDim2.new(0, 140, 0, 38)}, 0.1)
end)

-----------------------------------------------------
-- 🧩 ESP VISUAL
-----------------------------------------------------

local function createESPFrame(plr)
	local frame = Instance.new("Frame")
	frame.Name = "ESP_" .. plr.Name
	frame.BackgroundTransparency = 1
	frame.Visible = false
	frame.Parent = screenGui

	local box = Instance.new("Frame")
	box.Name = "Box"
	box.Size = UDim2.new(1, 0, 1, 0)
	box.BackgroundTransparency = BASE_TRANSPARENCY
	box.BorderSizePixel = 0
	box.Parent = frame

	local gradient = Instance.new("UIGradient")
	gradient.Rotation = 90
	gradient.Parent = box

	local healthBG = Instance.new("Frame")
	healthBG.Name = "HealthBG"
	healthBG.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
	healthBG.BackgroundTransparency = 0.7
	healthBG.BorderSizePixel = 0
	healthBG.Parent = box

	local healthBar = Instance.new("Frame")
	healthBar.Name = "HealthBar"
	healthBar.BorderSizePixel = 0
	healthBar.BackgroundColor3 = Color3.fromRGB(255, 105, 180)
	healthBar.Size = UDim2.new(1, 0, 1, 0)
	healthBar.BackgroundTransparency = 0.3
	healthBar.Parent = healthBG

	return {
		Player = plr,
		Frame = frame,
		Box = box,
		Gradient = gradient,
		HealthBG = healthBG,
		HealthBar = healthBar,
		LastHealth = 1
	}
end

local espObjects = {}
for _, p in ipairs(Players:GetPlayers()) do
	if p ~= LocalPlayer then
		espObjects[p] = createESPFrame(p)
	end
end

Players.PlayerAdded:Connect(function(p)
	if p ~= LocalPlayer then
		espObjects[p] = createESPFrame(p)
	end
end)

Players.PlayerRemoving:Connect(function(p)
	if espObjects[p] then
		espObjects[p].Frame:Destroy()
		espObjects[p] = nil
	end
end)

local function getHealthColor(r)
	if r >= 0.7 then
		return Color3.fromRGB(255, 105, 180)
	elseif r >= 0.35 then
		return Color3.fromRGB(180, 0, 255)
	else
		return Color3.fromRGB(0, 170, 255)
	end
end

local function getTeamGradient(plr)
	if not plr.Team then return nil end
	if string.find(plr.Team.Name:lower(), "red") or string.find(plr.Team.Name:lower(), "vermelh") then
		return {
			ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 70, 70)),
			ColorSequenceKeypoint.new(0.5, Color3.fromRGB(255, 120, 180)),
			ColorSequenceKeypoint.new(1, Color3.fromRGB(255, 190, 120))
		}
	elseif string.find(plr.Team.Name:lower(), "blue") or string.find(plr.Team.Name:lower(), "azul") then
		return {
			ColorSequenceKeypoint.new(0, Color3.fromRGB(100, 180, 255)),
			ColorSequenceKeypoint.new(0.5, Color3.fromRGB(160, 100, 255)),
			ColorSequenceKeypoint.new(1, Color3.fromRGB(0, 255, 255))
		}
	end
	return nil
end

task.spawn(function()
	while task.wait(0.05) do
		for plr, obj in pairs(espObjects) do
			if obj.Gradient then
				local custom = getTeamGradient(plr)
				if custom then
					obj.Gradient.Color = ColorSequence.new(custom)
				else
					local t = tick() * 0.15
					local c1 = Color3.fromHSV((t) % 1, 1, 1)
					local c2 = Color3.fromHSV((t + 0.33) % 1, 1, 1)
					local c3 = Color3.fromHSV((t + 0.66) % 1, 1, 1)
					obj.Gradient.Color = ColorSequence.new({
						ColorSequenceKeypoint.new(0, c1),
						ColorSequenceKeypoint.new(0.5, c2),
						ColorSequenceKeypoint.new(1, c3)
					})
				end
			end
		end
	end
end)

RunService.RenderStepped:Connect(function()
	if not espEnabled then
		for _, obj in pairs(espObjects) do obj.Frame.Visible = false end
		return
	end

	for plr, obj in pairs(espObjects) do
		local char = plr.Character
		local hrp = char and char:FindFirstChild("HumanoidRootPart")
		local hum = char and char:FindFirstChildOfClass("Humanoid")

		if char and hrp and hum and hum.Health > 0 then
			if not SHOW_FOR_TEAMMATES and plr.Team == LocalPlayer.Team then
				obj.Frame.Visible = false
			else
				local rootPos, onScreen = Camera:WorldToViewportPoint(hrp.Position)
				if onScreen then
					local dist = (Camera.CFrame.Position - hrp.Position).Magnitude
					local vis = math.clamp(1 - (dist / MAX_DISTANCE), 0, 1)
					
					-- 💡 Opacidade equilibrada
					local alpha = BASE_TRANSPARENCY + (1 - vis) * 0.3

					obj.Box.BackgroundTransparency = alpha
					obj.HealthBar.BackgroundTransparency = 0.3 + (1 - vis) * 0.4

					local height3D, width3D = 5.2, 2
					local topPos = Camera:WorldToViewportPoint(hrp.Position + Vector3.new(0, height3D/2, 0))
					local bottomPos = Camera:WorldToViewportPoint(hrp.Position - Vector3.new(0, height3D/2, 0))
					local boxHeight = math.abs(topPos.Y - bottomPos.Y)
					local boxWidth = boxHeight / (height3D / width3D)

					obj.Frame.Visible = true
					obj.Frame.Position = UDim2.new(0, rootPos.X, 0, rootPos.Y)
					obj.Frame.Size = UDim2.new(0, boxWidth, 0, boxHeight)
					obj.Frame.AnchorPoint = Vector2.new(0.5, 0.5)

					-- 🔹 Barra de vida proporcional à distância
					local barWidth = math.clamp(HEALTHBAR_WIDTH * vis + 1.5, 1.5, HEALTHBAR_WIDTH)
					obj.HealthBG.Size = UDim2.new(0, barWidth, 1, 0)
					obj.HealthBG.Position = UDim2.new(1, HEALTHBAR_GAP, 0, 0)

					local healthRatio = math.clamp(hum.Health / hum.MaxHealth, 0, 1)
					if math.abs(healthRatio - obj.LastHealth) > 0.01 then
						local color = getHealthColor(healthRatio)
						obj.LastHealth = healthRatio
						TweenService:Create(obj.HealthBar, TweenInfo.new(TWEEN_TIME, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
							Size = UDim2.new(1, 0, healthRatio, 0),
							Position = UDim2.new(0, 0, 1 - healthRatio, 0),
							BackgroundColor3 = color
						}):Play()
					end
				else
					obj.Frame.Visible = false
				end
			end
		else
			obj.Frame.Visible = false
		end
	end
end)

print("[DevESP_TeamDynamic] ✅ ESP carregada com barra de vida proporcional e botão RGB profissional.")
