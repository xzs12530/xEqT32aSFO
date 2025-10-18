local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer

-- ================== CONFIGURAÇÕES ==================
local ENABLED_BY_DEFAULT = true
local SHOW_TEAM_MATES = false
local MAX_DISTANCE = 300 -- studs
local BOX_THICKNESS = 1
local HP_BAR_WIDTH = 2       -- Barra fina
local HP_BAR_PADDING = 2     -- Distância da box
local BOX_TRANSPARENCY = 0.4 -- Para box filled não atrapalhar visão
-- ===================================================

local espEnabled = ENABLED_BY_DEFAULT
local espData = {} -- [player] = { Box, HPBg, HPBar, Billboard }

-- Cria BillboardGui com o nome
local function createBillboard(player)
	local char = player.Character
	if not char then return end
	local head = char:FindFirstChild("Head")
	if not head then return end
	if head:FindFirstChild("ESPGui") then return end

	local billboard = Instance.new("BillboardGui")
	billboard.Name = "ESPGui"
	billboard.Adornee = head
	billboard.AlwaysOnTop = true
	billboard.Size = UDim2.new(6,0,1.4,0)
	billboard.StudsOffset = Vector3.new(0,2.6,0)
	billboard.MaxDistance = MAX_DISTANCE
	billboard.Parent = head

	local frame = Instance.new("Frame")
	frame.Size = UDim2.new(1,0,1,0)
	frame.BackgroundTransparency = 1
	frame.Parent = billboard

	local nameLabel = Instance.new("TextLabel")
	nameLabel.Size = UDim2.new(1,0,1,0)
	nameLabel.BackgroundTransparency = 1
	nameLabel.TextScaled = true
	nameLabel.Font = Enum.Font.SourceSansBold
	nameLabel.TextStrokeTransparency = 0.6
	nameLabel.Text = (player.DisplayName ~= "" and player.DisplayName) or player.Name
	nameLabel.Parent = frame

	espData[player].Billboard = billboard
	espData[player].BillboardLabel = nameLabel -- para atualizar RGB
	return billboard
end

-- Cria Drawing objects (Box filled + HP background + HP fill)
local function createDrawingFor(player)
	if espData[player] and espData[player].Box then
		return espData[player].Box, espData[player].HPBg, espData[player].HPBar
	end

	local box = Drawing.new("Square")
	box.Visible = false
	box.Color = Color3.new(1,1,1)
	box.Thickness = BOX_THICKNESS
	box.Filled = true
	box.Transparency = BOX_TRANSPARENCY

	local hpBg = Drawing.new("Square")
	hpBg.Visible = false
	hpBg.Filled = true
	hpBg.Size = Vector2.new(HP_BAR_WIDTH, 10)
	hpBg.Color = Color3.fromRGB(30,30,30)

	local hpBar = Drawing.new("Square")
	hpBar.Visible = false
	hpBar.Filled = true
	hpBar.Size = Vector2.new(HP_BAR_WIDTH, 10)
	hpBar.Color = Color3.new(0,1,0)

	espData[player] = {
		Box = box,
		HPBg = hpBg,
		HPBar = hpBar
	}

	return box, hpBg, hpBar
end

-- Limpa ESP de um jogador
local function clearESP(player)
	local data = espData[player]
	if not data then return end
	if data.Box then pcall(function() data.Box:Remove() end) end
	if data.HPBg then pcall(function() data.HPBg:Remove() end) end
	if data.HPBar then pcall(function() data.HPBar:Remove() end) end
	if data.Billboard and data.Billboard.Parent then
		pcall(function() data.Billboard:Destroy() end)
	end
	espData[player] = nil
end

-- Atualiza ESP para um jogador
local function updateForPlayer(player)
	if not player.Character or not player.Character.Parent then
		clearESP(player)
		return
	end

	local root = player.Character:FindFirstChild("HumanoidRootPart")
	if not root then clearESP(player) return end

	local humanoid = player.Character:FindFirstChildOfClass("Humanoid")
	if not humanoid then clearESP(player) return end

	if player == LocalPlayer then clearESP(player) return end
	if not espEnabled then
		if espData[player] then
			if espData[player].Box then espData[player].Box.Visible = false end
			if espData[player].HPBg then espData[player].HPBg.Visible = false end
			if espData[player].HPBar then espData[player].HPBar.Visible = false end
		end
		return
	end

	local lpChar = LocalPlayer.Character
	if lpChar and lpChar:FindFirstChild("HumanoidRootPart") then
		local dist = (lpChar.HumanoidRootPart.Position - root.Position).Magnitude
		if dist > MAX_DISTANCE then
			if espData[player] then
				espData[player].Box.Visible = false
				espData[player].HPBg.Visible = false
				espData[player].HPBar.Visible = false
			end
			return
		end
	end

	local sameTeam = LocalPlayer.Team and player.Team and (LocalPlayer.Team == player.Team)
	if not SHOW_TEAM_MATES and sameTeam then
		clearESP(player)
		return
	end

	if not (player.Character:FindFirstChild("ESPGui")) then
		createBillboard(player)
	end

	-- calcula bounds
	local minX, minY = math.huge, math.huge
	local maxX, maxY = -math.huge, -math.huge
	local anyOnScreen = false
	for _, part in ipairs(player.Character:GetDescendants()) do
		if part:IsA("BasePart") then
			local pos, onScreen = Camera:WorldToViewportPoint(part.Position)
			if onScreen then
				anyOnScreen = true
				if pos.X < minX then minX = pos.X end
				if pos.Y < minY then minY = pos.Y end
				if pos.X > maxX then maxX = pos.X end
				if pos.Y > maxY then maxY = pos.Y end
			end
		end
	end

	if not anyOnScreen then
		if espData[player] then
			espData[player].Box.Visible = false
			espData[player].HPBg.Visible = false
			espData[player].HPBar.Visible = false
		end
		return
	end

	if minX == math.huge then clearESP(player) return end

	local box, hpBg, hpBar = createDrawingFor(player)

	-- Box RGB dinâmica
	local t = tick() * 2
	local r = (math.sin(t) * 0.5 + 0.5)
	local g = (math.sin(t + 2) * 0.5 + 0.5)
	local b = (math.sin(t + 4) * 0.5 + 0.5)
	local rgbColor = Color3.new(r,g,b)
	box.Color = rgbColor

	local boxPos = Vector2.new(minX, minY)
	local boxSize = Vector2.new(math.max(2,maxX-minX), math.max(2,maxY-minY))
	box.Position = boxPos
	box.Size = boxSize
	box.Visible = true

	-- HP vertical à direita
	local hpX = boxPos.X + boxSize.X + HP_BAR_PADDING
	local hpY = boxPos.Y
	local hpH = boxSize.Y
	hpBg.Position = Vector2.new(hpX, hpY)
	hpBg.Size = Vector2.new(HP_BAR_WIDTH, hpH)
	hpBg.Visible = true

	local ratio = 1
	if humanoid and humanoid.MaxHealth > 0 then
		ratio = math.clamp(humanoid.Health / humanoid.MaxHealth, 0, 1)
	end

	local filledHeight = math.max(1, hpH * ratio)
	hpBar.Size = Vector2.new(HP_BAR_WIDTH, filledHeight)
	hpBar.Position = Vector2.new(hpX, hpY + (hpH - filledHeight))

	local rr, gg
	if ratio > 0.5 then
		rr = (1 - ratio) * 2
		gg = 1
	else
		rr = 1
		gg = ratio * 2
	end
	hpBar.Color = Color3.new(rr, gg, 0)
	hpBar.Visible = true

	-- Atualiza cor do Billboard
	if espData[player].BillboardLabel then
		espData[player].BillboardLabel.TextColor3 = rgbColor
	end
end

-- Loop principal
RunService.RenderStepped:Connect(function()
	if not Camera then Camera = workspace.CurrentCamera end
	for _, player in pairs(Players:GetPlayers()) do
		if player ~= LocalPlayer then
			pcall(function() updateForPlayer(player) end)
		end
	end
end)

-- Toggle com tecla K
UserInputService.InputBegan:Connect(function(input, processed)
	if processed then return end
	if input.KeyCode == Enum.KeyCode.K then
		espEnabled = not espEnabled
	end
end)

-- Botão arrastável
do
	local screenGui = Instance.new("ScreenGui")
	screenGui.Name = "ESP_Controller"
	screenGui.ResetOnSpawn = false
	screenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

	local btn = Instance.new("TextButton")
	btn.Name = "ToggleESP"
	btn.Size = UDim2.new(0, 120, 0, 40)
	btn.Position = UDim2.new(0.05, 0, 0.2, 0)
	btn.TextScaled = true
	btn.Font = Enum.Font.SourceSansBold
	btn.TextColor3 = Color3.new(1,1,1)
	btn.Active = true
	btn.Draggable = true
	btn.BackgroundColor3 = espEnabled and Color3.fromRGB(0,200,0) or Color3.fromRGB(200,0,0)
	btn.Text = espEnabled and "ESP: ON" or "ESP: OFF"
	btn.Parent = screenGui

	btn.MouseButton1Click:Connect(function()
		espEnabled = not espEnabled
		btn.BackgroundColor3 = espEnabled and Color3.fromRGB(0,200,0) or Color3.fromRGB(200,0,0)
		btn.Text = espEnabled and "ESP: ON" or "ESP: OFF"
		if not espEnabled then
			for p,data in pairs(espData) do
				if data.Box then data.Box.Visible = false end
				if data.HPBg then data.HPBg.Visible = false end
				if data.HPBar then data.HPBar.Visible = false end
			end
		end
	end)
end

-- Cleanup quando jogador sai
Players.PlayerRemoving:Connect(clearESP)

-- Atualiza displayName quando muda
Players.PlayerAdded:Connect(function(p)
	p:GetPropertyChangedSignal("DisplayName"):Connect(function()
		if p.Character and p.Character:FindFirstChild("ESPGui") and espData[p] and espData[p].BillboardLabel then
			p.Character.ESPGui.Frame.TextLabel.Text = (p.DisplayName ~= "" and p.DisplayName) or p.Name
		end
	end)
end)
