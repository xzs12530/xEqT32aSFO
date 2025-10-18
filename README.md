local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer

-- CONFIGURAÇÕES
local SHOW_FOR_TEAMMATES = false
local BOX_TRANSPARENCY = 0.35
local HEALTHBAR_WIDTH = 5
local HEALTHBAR_GAP = 3

-- GUI PRINCIPAL
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "DevESP_RGB"
screenGui.IgnoreGuiInset = true
screenGui.ResetOnSpawn = false
screenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

-- BOTÃO DE ATIVAÇÃO (ARRASTÁVEL)
local toggleBtn = Instance.new("TextButton")
toggleBtn.Name = "ESP_Toggle"
toggleBtn.Size = UDim2.new(0, 120, 0, 28)
toggleBtn.Position = UDim2.new(0, 10, 0, 10)
toggleBtn.BackgroundColor3 = Color3.new(0, 0, 0)
toggleBtn.TextColor3 = Color3.new(1, 1, 1)
toggleBtn.Text = "ESP: ON"
toggleBtn.Font = Enum.Font.SourceSansBold
toggleBtn.TextSize = 14
toggleBtn.AutoButtonColor = false
toggleBtn.Active = true
toggleBtn.Draggable = true -- ✅ agora é totalmente arrastável
toggleBtn.Parent = screenGui

local espEnabled = true
toggleBtn.MouseButton1Click:Connect(function()
	espEnabled = not espEnabled
	toggleBtn.Text = espEnabled and "ESP: ON" or "ESP: OFF"
end)

-- CRIAÇÃO DO ESP PARA JOGADOR
local function createESPFrame(plr)
	local frame = Instance.new("Frame")
	frame.Name = "ESP_" .. plr.Name
	frame.BackgroundTransparency = 1
	frame.Visible = false
	frame.Parent = screenGui

	-- Caixa principal translúcida
	local box = Instance.new("Frame")
	box.Name = "Box"
	box.Size = UDim2.new(1, 0, 1, 0)
	box.BackgroundTransparency = BOX_TRANSPARENCY
	box.BorderSizePixel = 0
	box.Parent = frame

	-- Gradiente fixo (amarelo → roxo → verde)
	local gradient = Instance.new("UIGradient")
	gradient.Color = ColorSequence.new({
		ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 255, 0)),   -- Amarelo topo
		ColorSequenceKeypoint.new(0.5, Color3.fromRGB(180, 0, 255)), -- Roxo meio
		ColorSequenceKeypoint.new(1, Color3.fromRGB(0, 255, 100))    -- Verde base
	})
	gradient.Rotation = 90
	gradient.Parent = box

	local outline = Instance.new("UIStroke")
	outline.Thickness = 1.4
	outline.Color = Color3.fromRGB(255, 255, 255)
	outline.Transparency = 0.4
	outline.Parent = box

	-- Barra de vida
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
	healthBar.Parent = healthBG

	return {
		Player = plr,
		Frame = frame,
		Box = box,
		Gradient = gradient,
		HealthBG = healthBG,
		HealthBar = healthBar
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

-- Cor da barra de vida conforme saúde
local function getHealthColor(r)
	if r >= 0.7 then
		return Color3.fromRGB(255, 105, 180) -- Rosa
	elseif r >= 0.35 then
		return Color3.fromRGB(255, 204, 0) -- Amarelo
	else
		return Color3.fromRGB(220, 20, 60) -- Vermelho
	end
end

-- Atualização da ESP
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
					-- Dimensões exatas para R6
					local height3D, width3D = 5.2, 2
					local topPos = Camera:WorldToViewportPoint(hrp.Position + Vector3.new(0, height3D/2, 0))
					local bottomPos = Camera:WorldToViewportPoint(hrp.Position - Vector3.new(0, height3D/2, 0))
					local boxHeight = math.abs(topPos.Y - bottomPos.Y)
					local boxWidth = boxHeight / (height3D / width3D)

					obj.Frame.Visible = true
					obj.Frame.Position = UDim2.new(0, rootPos.X, 0, rootPos.Y)
					obj.Frame.Size = UDim2.new(0, boxWidth, 0, boxHeight)
					obj.Frame.AnchorPoint = Vector2.new(0.5, 0.5)

					obj.Box.Size = UDim2.new(1, 0, 1, 0)
					obj.Box.Position = UDim2.new(0, 0, 0, 0)

					obj.HealthBG.Size = UDim2.new(0, HEALTHBAR_WIDTH, 1, 0)
					obj.HealthBG.Position = UDim2.new(1, HEALTHBAR_GAP, 0, 0)

					local healthRatio = math.clamp(hum.Health / hum.MaxHealth, 0, 1)
					local color = getHealthColor(healthRatio)
					obj.HealthBar.BackgroundColor3 = color
					obj.HealthBar.Size = UDim2.new(1, 0, healthRatio, 0)
					obj.HealthBar.Position = UDim2.new(0, 0, 1 - healthRatio, 0)
				else
					obj.Frame.Visible = false
				end
			end
		else
			obj.Frame.Visible = false
		end
	end
end)

print("[DevESP_RGB] ESP translúcida com gradiente RGB carregada (R6).")
