local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer

-- == CONFIGURAÇÕES ==
local ENABLED_BY_DEFAULT = true
local SHOW_TEAM_MATES = false
local MAX_DISTANCE = 300 -- studs
local BOX_THICKNESS = 1
local HP_BAR_WIDTH = 1    -- pixels
local HP_BAR_PADDING = 2  -- distance between box and hp bar (pixels)
-- ======================

local espEnabled = ENABLED_BY_DEFAULT
local espData = {} -- [player] = { Box, HPBg, HPBar, Billboard }

-- Cria/atualiza Billboard com o nome (sem barra de vida)
local function createBillboard(player)
	local char = player.Character
	if not char then return end
	local head = char:FindFirstChild("Head")
	if not head then return end

	-- evita duplicar
	local exist = head:FindFirstChild("ESPGui")
	if exist then return exist end

	local billboard = Instance.new("BillboardGui")
	billboard.Name = "ESPGui"
	billboard.Adornee = head
	billboard.AlwaysOnTop = true
	billboard.Size = UDim2.new(6,0,1.4,0)
	billboard.StudsOffset = Vector3.new(0, 2.6, 0)
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
	nameLabel.TextColor3 = Color3.new(1,1,1)
	nameLabel.Text = (player.DisplayName ~= "" and player.DisplayName) or player.Name
	nameLabel.Parent = frame

	return billboard
end

-- Função para criar Drawing objects (Box + HP background + HP fill)
local function createDrawingFor(player)
	if espData[player] and espData[player].Box then
		return espData[player].Box, espData[player].HPBg, espData[player].HPBar
	end

	-- Box (outline)
	local box = Drawing.new("Square")
	box.Visible = false
	box.Color = Color3.new(1,1,1)
	box.Thickness = BOX_THICKNESS
	box.Filled = false
	-- HP background (full height)
	local hpBg = Drawing.new("Square")
	hpBg.Visible = false
	hpBg.Filled = true
	hpBg.Size = Vector2.new(HP_BAR_WIDTH, 10) -- height será ajustada por loop
	hpBg.Color = Color3.fromRGB(30,30,30)
	-- HP fill (altura proporcional)
	local hpBar = Drawing.new("Square")
	hpBar.Visible = false
	hpBar.Filled = true
	hpBar.Size = Vector2.new(HP_BAR_WIDTH, 10)
	hpBar.Color = Color3.new(0,1,0)

	espData[player] = espData[player] or {}
	espData[player].Box = box
	espData[player].HPBg = hpBg
	espData[player].HPBar = hpBar

	return box, hpBg, hpBar
end

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

-- Atualiza posição/visual da caixa e da barra lateral baseado no personagem
local function updateForPlayer(player)
	if not player.Character or not player.Character.Parent then
		clearESP(player)
		return
	end

	local root = player.Character:FindFirstChild("HumanoidRootPart")
	if not root then
		clearESP(player)
		return
	end

	local humanoid = player.Character:FindFirstChildOfClass("Humanoid")
	if not humanoid then
		clearESP(player)
		return
	end

	-- Team & distance checks
	if player == LocalPlayer then
		clearESP(player)
		return
	end

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
			-- fora do alcance
			if espData[player] and espData[player].Box then espData[player].Box.Visible = false end
			if espData[player] and espData[player].HPBg then espData[player].HPBg.Visible = false end
			if espData[player] and espData[player].HPBar then espData[player].HPBar.Visible = false end
			return
		end
	end

	if not SHOW_TEAM_MATES and LocalPlayer.Team and player.Team and LocalPlayer.Team == player.Team then
		-- teammate e não mostrar
		clearESP(player)
		return
	end

	-- cria/billboard se necessário
	if not (player.Character:FindFirstChild("ESPGui")) then
		createBillboard(player)
	end

	-- calcula bounds da personagem em viewport
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

	-- se nada na tela, esconder
	if not anyOnScreen then
		if espData[player] and espData[player].Box then espData[player].Box.Visible = false end
		if espData[player] and espData[player].HPBg then espData[player].HPBg.Visible = false end
		if espData[player] and espData[player].HPBar then espData[player].HPBar.Visible = false end
		return
	end

	-- evita casos estranhos
	if minX == math.huge then
		clearESP(player)
		return
	end

	-- cria desenhos se necessário
	local box, hpBg, hpBar = createDrawingFor(player)

	-- configura box
	local boxPos = Vector2.new(minX, minY)
	local boxSize = Vector2.new(math.max(2, maxX - minX), math.max(2, maxY - minY))
	box.Position = boxPos
	box.Size = boxSize
	-- cor por team
	local sameTeam = LocalPlayer.Team and player.Team and (LocalPlayer.Team == player.Team)
	box.Color = sameTeam and Color3.new(0, 1, 1) or Color3.new(1, 0, 0)
	box.Visible = true

	-- configura HP background (vertical, ao lado direito)
	local hpX = boxPos.X + boxSize.X + HP_BAR_PADDING
	local hpY = boxPos.Y
	local hpH = boxSize.Y
	hpBg.Position = Vector2.new(hpX, hpY)
	hpBg.Size = Vector2.new(HP_BAR_WIDTH, hpH)
	hpBg.Visible = true

	-- calcula ratio vida
	local ratio = 1
	if humanoid and humanoid.MaxHealth and humanoid.MaxHealth > 0 then
		ratio = math.clamp(humanoid.Health / humanoid.MaxHealth, 0, 1)
	end

	-- hpBar: preencher de baixo pra cima com altura proporcional
	local filledHeight = math.max(1, hpH * ratio)
	hpBar.Size = Vector2.new(HP_BAR_WIDTH, filledHeight)
	-- posicionar no fundo (baixo) do hpBg
	hpBar.Position = Vector2.new(hpX, hpY + (hpH - filledHeight))
	-- cor gradiente simples (verde->amarelo->vermelho)
	local r,g
	if ratio > 0.5 then
		r = (1 - ratio) * 2
		g = 1
	else
		r = 1
		g = ratio * 2
	end
	hpBar.Color = Color3.new(r,g,0)
	hpBar.Visible = true
end

-- Loop principal
RunService.RenderStepped:Connect(function()
	-- se jogador não tem camera (por segurança)
	if not Camera then Camera = workspace.CurrentCamera end
	for _, player in pairs(Players:GetPlayers()) do
		if player ~= LocalPlayer then
			pcall(function() updateForPlayer(player) end)
		end
	end
end)

-- Toggle com tecla K também (opcional)
UserInputService.InputBegan:Connect(function(input, processed)
	if processed then return end
	if input.KeyCode == Enum.KeyCode.K then
		espEnabled = not espEnabled
	end
end)

-- Botão arrastável na tela para ligar/desligar
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
		-- quando desligado, garantir que desenhos fiquem invisíveis
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
Players.PlayerRemoving:Connect(function(p)
	clearESP(p)
end)

-- Atualiza displayName quando muda
Players.PlayerAdded:Connect(function(p)
	p:GetPropertyChangedSignal("DisplayName"):Connect(function()
		if p.Character and p.Character:FindFirstChild("ESPGui") then
			local bb = p.Character:FindFirstChild("ESPGui")
			if bb and bb:FindFirstChildOfClass("Frame") and bb.Frame:FindFirstChildOfClass("TextLabel") then
				p.Character.ESPGui.Frame.TextLabel.Text = (p.DisplayName ~= "" and p.DisplayName) or p.Name
			end
		end
	end)
end)
