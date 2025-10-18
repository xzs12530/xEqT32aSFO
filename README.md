local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer

-- == CONFIGURAÇÕES ==
local ENABLED_BY_DEFAULT = true
local SHOW_TEAM_MATES = false
local MAX_DISTANCE = 250
local BUTTON_SIZE = UDim2.new(0, 120, 0, 40)
local BUTTON_COLOR_ON = Color3.fromRGB(0, 200, 0)
local BUTTON_COLOR_OFF = Color3.fromRGB(200, 0, 0)
-- ====================

local espEnabled = ENABLED_BY_DEFAULT
local espData = {}

------------------------------------------------
-- Interface (botão arrastável)
------------------------------------------------
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "ESP_Controller"
screenGui.ResetOnSpawn = false
screenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

local button = Instance.new("TextButton")
button.Name = "ToggleESP"
button.Text = "ESP: ON"
button.BackgroundColor3 = BUTTON_COLOR_ON
button.Size = BUTTON_SIZE
button.Position = UDim2.new(0.05, 0, 0.2, 0)
button.TextScaled = true
button.TextColor3 = Color3.new(1, 1, 1)
button.Font = Enum.Font.SourceSansBold
button.Active = true
button.Draggable = true
button.Parent = screenGui

-- Ligar/desligar ESP via clique
button.MouseButton1Click:Connect(function()
	espEnabled = not espEnabled
	if espEnabled then
		button.BackgroundColor3 = BUTTON_COLOR_ON
		button.Text = "ESP: ON"
	else
		button.BackgroundColor3 = BUTTON_COLOR_OFF
		button.Text = "ESP: OFF"
	end
end)

------------------------------------------------
-- Criação de BillboardGui (nome + vida)
------------------------------------------------
local function createBillboard(player)
	local char = player.Character
	if not char or not char:FindFirstChild("Head") then return end

	local billboard = Instance.new("BillboardGui")
	billboard.Name = "ESPGui"
	billboard.Adornee = char:FindFirstChild("Head")
	billboard.AlwaysOnTop = true
	billboard.Size = UDim2.new(6, 0, 2.4, 0)
	billboard.StudsOffset = Vector3.new(0, 2.6, 0)
	billboard.MaxDistance = MAX_DISTANCE
	billboard.Parent = char

	local frame = Instance.new("Frame")
	frame.Size = UDim2.new(1, 0, 1, 0)
	frame.BackgroundTransparency = 1
	frame.Parent = billboard

	local nameLabel = Instance.new("TextLabel")
	nameLabel.BackgroundTransparency = 1
	nameLabel.Size = UDim2.new(1, 0, 0.5, 0)
	nameLabel.TextScaled = true
	nameLabel.Font = Enum.Font.SourceSansBold
	nameLabel.TextColor3 = Color3.new(1, 1, 1)
	nameLabel.TextStrokeTransparency = 0.6
	nameLabel.Text = player.DisplayName or player.Name
	nameLabel.Parent = frame

	local hpBg = Instance.new("Frame")
	hpBg.Size = UDim2.new(0.9, 0, 0.2, 0)
	hpBg.Position = UDim2.new(0.05, 0, 0.7, 0)
	hpBg.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
	hpBg.BorderSizePixel = 0
	hpBg.BackgroundTransparency = 0.4
	hpBg.Parent = frame

	local hpBar = Instance.new("Frame")
	hpBar.BackgroundColor3 = Color3.new(0, 1, 0)
	hpBar.BorderSizePixel = 0
	hpBar.Size = UDim2.new(1, 0, 1, 0)
	hpBar.Parent = hpBg

	return {
		Billboard = billboard,
		Name = nameLabel,
		HPBg = hpBg,
		HPBar = hpBar,
	}
end

------------------------------------------------
-- Funções auxiliares
------------------------------------------------
local function updateHealth(gui, humanoid)
	if not gui or not humanoid then return end
	local ratio = math.clamp(humanoid.Health / humanoid.MaxHealth, 0, 1)
	gui.HPBar.Size = UDim2.new(ratio, 0, 1, 0)
	local r, g
	if ratio > 0.5 then
		r = (1 - ratio) * 2
		g = 1
	else
		r = 1
		g = ratio * 2
	end
	gui.HPBar.BackgroundColor3 = Color3.new(r, g, 0)
end

local function getBox(player)
	if espData[player] and espData[player].Box then
		return espData[player].Box
	end
	local box = Drawing.new("Square")
	box.Visible = false
	box.Color = Color3.new(1, 1, 1)
	box.Thickness = 1
	box.Filled = false
	if not espData[player] then espData[player] = {} end
	espData[player].Box = box
	return box
end

local function clearESP(player)
	if espData[player] then
		if espData[player].Box then espData[player].Box:Remove() end
		if espData[player].Billboard then espData[player].Billboard:Destroy() end
		espData[player] = nil
	end
end

------------------------------------------------
-- Loop principal (RenderStepped)
------------------------------------------------
RunService.RenderStepped:Connect(function()
	if not espEnabled then
		for _, data in pairs(espData) do
			if data.Box then data.Box.Visible = false end
		end
		return
	end

	for _, player in pairs(Players:GetPlayers()) do
		if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
			local hrp = player.Character.HumanoidRootPart
			local humanoid = player.Character:FindFirstChild("Humanoid")

			local headPos, onScreen = Camera:WorldToViewportPoint(hrp.Position)
			local dist = (LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart"))
				and (LocalPlayer.Character.HumanoidRootPart.Position - hrp.Position).Magnitude
				or math.huge

			if onScreen and dist <= MAX_DISTANCE then
				local sameTeam = LocalPlayer.Team and player.Team and (LocalPlayer.Team == player.Team)
				if not SHOW_TEAM_MATES and sameTeam then
					clearESP(player)
				else
					local box = getBox(player)
					local min, max = Vector3.new(math.huge, math.huge, 0), Vector3.new(-math.huge, -math.huge, 0)
					for _, part in ipairs(player.Character:GetChildren()) do
						if part:IsA("BasePart") then
							local pos, vis = Camera:WorldToViewportPoint(part.Position)
							if vis then
								min = Vector3.new(math.min(min.X, pos.X), math.min(min.Y, pos.Y), 0)
								max = Vector3.new(math.max(max.X, pos.X), math.max(max.Y, pos.Y), 0)
							end
						end
					end
					box.Size = Vector2.new(max.X - min.X, max.Y - min.Y)
					box.Position = Vector2.new(min.X, min.Y)
					box.Color = sameTeam and Color3.new(0, 1, 1) or Color3.new(1, 0, 0)
					box.Visible = true

					if not espData[player] or not espData[player].Billboard then
						local gui = createBillboard(player)
						if gui then
							espData[player] = espData[player] or {}
							espData[player].Billboard = gui.Billboard
							espData[player].HPBar = gui.HPBar
						end
					end
					updateHealth(espData[player], humanoid)
				end
			else
				if espData[player] and espData[player].Box then
					espData[player].Box.Visible = false
				end
			end
		else
			clearESP(player)
		end
	end
end)

-- Limpa quando jogador sai
Players.PlayerRemoving:Connect(function(p)
	clearESP(p)
end)
