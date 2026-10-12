--[[
	LEGO PARKOUR + VISUAL DESTRUCTION ENGINE — Roblox Mobile
	Tipo: LocalScript
	Local: StarterPlayer > StarterPlayerScripts

	ARQUITETURA: rig visual SEPARADO (somente cliente).
	- O personagem R15 original continua sendo o corpo físico (movimento, câmera, controles, pulo).
	- Uma minifigura de peças (Part) é criada dentro do Character e presa ao HumanoidRootPart por UM Motor6D raiz.
	- As peças internas se ligam por Motor6D (articulações animadas) e Weld (detalhes fixos).
	- Todas as peças são Massless, sem colisão, sem toque e sem consulta (CanQuery=false).
	- O avatar original só é ocultado (Transparency local) DEPOIS de construir, validar e verificar o rig.
	- As animações são procedurais (Motor6D.Transform, um único loop de render). Nomes "MF_*" evitam
	  qualquer conflito com o Animator/Animate padrão.
	Base V3 preservada. V4 adiciona slide, vault, agarrar bordas, subir da borda, wall-jump,
	rolamento visual, scanner local de objetos potencialmente destrutíveis e contorno cinza discreto.
	Destruição apenas visual/local: não remove peças reais nem altera colisões compartilhadas.
	Status: gerado por edição estática; NÃO foi executado no Roblox Studio.
]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local TAG = "[LegoEngine]"

----------------------------------------------------------------------
-- CONFIG
----------------------------------------------------------------------
local CONFIG = {
	StartEnabled = true,
	UnitScale = 0.13,        -- studs por "mm" LEGO (minifigura ~5 studs de altura)
	EnableDoubleJump = true,
	WaitTimeout = 12,
	MaxRigParts = 140,
	FootOffsetTweak = 0,     -- ajuste manual da altura dos pés (studs; negativo = desce)
	PunchCooldown = 0.3,
	DoubleJumpMinAirTime = 0.12,
	ButtonBottom = 200,      -- distância dos botões de ação até a base da tela (acima do botão de pulo)
	ParkourScanDistance = 3.2,
	BreakScanRadius = 34,
	BreakScanInterval = 0.9,
	BreakMaxHighlights = 60,
	BreakHitDistance = 22,
	BreakRestoreSeconds = 12,
}

local K = CONFIG.UnitScale
local rad, abs, sin, cos, exp, clamp = math.rad, math.abs, math.sin, math.cos, math.exp, math.clamp
local ST = Enum.HumanoidStateType

local function lerp(a, b, t) return a + (b - a) * t end
local function easeOut(t) return 1 - (1 - t) * (1 - t) end
local function easeIn(t) return t * t end
local function smooth(t) return t * t * (3 - 2 * t) end

local ROOT_PIVOT = CFrame.new(0, 18 * K, 0)
local ROOT_PIVOT_INV = ROOT_PIVOT:Inverse()
local HEAD_CENTER = CFrame.new(0, 32.5, 0)

-- Ganchos para o futuro sistema de combate (o soco atual é APENAS visual)
local Hooks = {
	OnPunchStrike = nil, -- function(side: number (1=direita,-1=esquerda), rootPart: BasePart)
}

----------------------------------------------------------------------
-- Utilitários
----------------------------------------------------------------------
local function newBin()
	local bin = { items = {} }
	function bin:Add(item)
		table.insert(self.items, item)
		return item
	end
	function bin:Clean()
		local items = self.items
		self.items = {}
		for i = #items, 1, -1 do
			local item = items[i]
			local kind = typeof(item)
			if kind == "RBXScriptConnection" then
				item:Disconnect()
			elseif kind == "Instance" then
				local ok, e = pcall(function() item:Destroy() end)
				if not ok then warn(TAG .. " erro ao destruir: " .. tostring(e)) end
			elseif kind == "function" then
				local ok, e = pcall(item)
				if not ok then warn(TAG .. " erro na limpeza: " .. tostring(e)) end
			end
		end
	end
	return bin
end

-- Executa uma etapa preservando a mensagem de erro original (e o traceback no console)
local function safeCall(step, fn, ...)
	local args = table.pack(...)
	local trace = nil
	local ok, res = xpcall(function()
		return fn(table.unpack(args, 1, args.n))
	end, function(e)
		trace = debug.traceback(tostring(e), 2)
		return tostring(e)
	end)
	if not ok then
		warn(TAG .. " ERRO na etapa '" .. step .. "': " .. tostring(res))
		if trace then warn(trace) end
		return false, tostring(res)
	end
	return true, res
end

local function make(class, props, parent)
	local inst = Instance.new(class)
	for k, v in pairs(props) do inst[k] = v end
	if parent then inst.Parent = parent end
	return inst
end

----------------------------------------------------------------------
-- Cores e trajes
----------------------------------------------------------------------
local COL = {
	Skin = Color3.fromRGB(245, 205, 47),
	SkinDark = Color3.fromRGB(226, 181, 30),
	Black = Color3.fromRGB(27, 42, 52),
	White = Color3.fromRGB(244, 244, 244),
	Gold = Color3.fromRGB(239, 184, 56),
	Red = Color3.fromRGB(196, 40, 28),
	DarkRed = Color3.fromRGB(140, 30, 22),
	Blue = Color3.fromRGB(13, 105, 172),
	DarkBlue = Color3.fromRGB(32, 58, 86),
	Brown = Color3.fromRGB(105, 64, 40),
	LightBrown = Color3.fromRGB(128, 84, 55),
	DarkBrown = Color3.fromRGB(70, 43, 27),
	Tan = Color3.fromRGB(215, 197, 154),
	Khaki = Color3.fromRGB(158, 138, 94),
	DarkGray = Color3.fromRGB(60, 60, 64),
}

local OUTFITS = {
	{
		Name = "Explorador", Style = "Jacket",
		Torso = COL.Brown, Arms = COL.Brown, Legs = COL.Khaki, Boots = COL.DarkBrown, Sole = COL.Black,
		Shirt = COL.Tan, Collar = COL.DarkBrown, Belt = COL.Black, Buckle = COL.Gold,
		Pocket = COL.DarkBrown, Flap = COL.LightBrown, Stitch = COL.DarkBrown,
		Strap = true, StrapColor = COL.DarkBrown,
		Headgear = "Hat", HeadgearColor = COL.Brown, HatBand = COL.Tan,
	},
	{
		Name = "Herói", Style = "Hero",
		Torso = COL.Blue, Arms = COL.Blue, Legs = COL.DarkBlue, Boots = COL.Red, Sole = COL.Black,
		Belt = COL.Gold, Buckle = COL.Red, Accent = COL.Gold, Accent2 = COL.Red,
		Headgear = "Helmet", HeadgearColor = COL.Blue,
	},
	{
		Name = "Piloto", Style = "Jacket",
		Torso = COL.Red, Arms = COL.Red, Legs = COL.DarkGray, Boots = COL.Black, Sole = COL.DarkGray,
		Shirt = COL.White, Collar = COL.DarkRed, Belt = COL.Black, Buckle = COL.Gold,
		Pocket = COL.DarkRed, Flap = COL.Red, Stitch = COL.Black,
		Strap = false,
		Headgear = "Hair", HeadgearColor = COL.DarkBrown,
	},
}

----------------------------------------------------------------------
-- Interface (mobile)
----------------------------------------------------------------------
local C_NEUTRAL = Color3.fromRGB(225, 228, 235)
local C_OK = Color3.fromRGB(120, 225, 150)
local C_WARN = Color3.fromRGB(255, 205, 90)
local C_ERR = Color3.fromRGB(255, 110, 110)

local function buildUI()
	local playerGui = player:WaitForChild("PlayerGui")
	local old = playerGui:FindFirstChild("LegoEngineUI")
	if old then old:Destroy() end

	local gui = make("ScreenGui", {
		Name = "LegoEngineUI", ResetOnSpawn = false, DisplayOrder = 50, IgnoreGuiInset = false,
	})
	pcall(function() gui.ScreenInsets = Enum.ScreenInsets.DeviceSafeInsets end)
	gui.Parent = playerGui

	local panel = make("Frame", {
		Name = "Panel", Position = UDim2.new(0, 8, 0, 6), Size = UDim2.new(0, 216, 0, 0),
		AutomaticSize = Enum.AutomaticSize.Y, BackgroundColor3 = Color3.fromRGB(24, 26, 32),
		BackgroundTransparency = 0.15, BorderSizePixel = 0,
	}, gui)
	make("UICorner", { CornerRadius = UDim.new(0, 10) }, panel)
	make("UIPadding", {
		PaddingTop = UDim.new(0, 4), PaddingBottom = UDim.new(0, 6),
		PaddingLeft = UDim.new(0, 8), PaddingRight = UDim.new(0, 8),
	}, panel)
	make("UIListLayout", { Padding = UDim.new(0, 4), SortOrder = Enum.SortOrder.LayoutOrder }, panel)

	local header = make("TextButton", {
		Size = UDim2.new(1, 0, 0, 30), BackgroundTransparency = 1, Text = "LEGO ENGINE  ▾",
		TextColor3 = Color3.fromRGB(255, 214, 10), Font = Enum.Font.GothamBold, TextSize = 14,
		TextXAlignment = Enum.TextXAlignment.Left, LayoutOrder = 1,
	}, panel)

	local body = make("Frame", {
		Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y,
		BackgroundTransparency = 1, LayoutOrder = 2,
	}, panel)
	make("UIListLayout", { Padding = UDim.new(0, 4), SortOrder = Enum.SortOrder.LayoutOrder }, body)

	local function makeLabel(order)
		return make("TextLabel", {
			Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, BackgroundTransparency = 1,
			Font = Enum.Font.GothamMedium, TextSize = 13, TextWrapped = true,
			TextXAlignment = Enum.TextXAlignment.Left, TextColor3 = C_NEUTRAL, Text = "", LayoutOrder = order,
		}, body)
	end
	local lblInit, lblBuild, lblAnim, lblErr = makeLabel(1), makeLabel(2), makeLabel(3), makeLabel(4)
	lblErr.Visible = false

	local function makeButton(text, order, height, color)
		local b = make("TextButton", {
			Size = UDim2.new(1, 0, 0, height), BackgroundColor3 = color, Font = Enum.Font.GothamBold,
			TextSize = 14, TextColor3 = Color3.new(1, 1, 1), Text = text, AutoButtonColor = true, LayoutOrder = order,
		}, body)
		make("UICorner", { CornerRadius = UDim.new(0, 8) }, b)
		return b
	end
	local btnToggle = makeButton("Transformar: ...", 5, 40, Color3.fromRGB(60, 120, 220))
	local btnOutfit = makeButton("Traje: ...", 6, 36, Color3.fromRGB(90, 90, 105))

	local function makeAction(text, xOffset, color)
		local b = make("TextButton", {
			AnchorPoint = Vector2.new(1, 1), Position = UDim2.new(1, xOffset, 1, -CONFIG.ButtonBottom),
			Size = UDim2.new(0, 58, 0, 58), BackgroundColor3 = color, Font = Enum.Font.GothamBold,
			TextSize = 11, TextColor3 = Color3.new(1, 1, 1), Text = text, Visible = false, ZIndex = 5,
		}, gui)
		make("UICorner", { CornerRadius = UDim.new(1, 0) }, b)
		make("UIStroke", {
			Thickness = 2, Color = Color3.new(1, 1, 1), Transparency = 0.4,
			ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
		}, b)
		return b
	end
	local btnPunch = makeAction("SOCO", -10, Color3.fromRGB(205, 60, 50))
	local btnGesture = makeAction("GESTO", -72, Color3.fromRGB(60, 160, 90))
	local btnSlide = makeAction("SLIDE", -134, Color3.fromRGB(46, 116, 172))
	local btnParkour = makeAction("PARKOUR", -196, Color3.fromRGB(115, 86, 185))
	local btnBreak = makeAction("QUEBRAR", -258, Color3.fromRGB(105, 105, 112))

	header.Activated:Connect(function()
		body.Visible = not body.Visible
		header.Text = body.Visible and "LEGO ENGINE  ▾" or "LEGO ENGINE  ▸"
	end)

	local ui = { Gui = gui, Toggle = btnToggle, Outfit = btnOutfit, Punch = btnPunch, Gesture = btnGesture, Slide = btnSlide, Parkour = btnParkour, Break = btnBreak }
	function ui.setInit(text, color)
		lblInit.Text = "Inicialização: " .. text
		lblInit.TextColor3 = color or C_NEUTRAL
	end
	function ui.setBuild(text, color)
		lblBuild.Text = "Construção: " .. text
		lblBuild.TextColor3 = color or C_NEUTRAL
		print(TAG, "Construção:", text)
	end
	function ui.setAnim(text, color)
		lblAnim.Text = "Animação: " .. text
		lblAnim.TextColor3 = color or C_NEUTRAL
	end
	function ui.setError(step, msg)
		if step == nil then
			lblErr.Visible = false
			lblErr.Text = ""
		else
			lblErr.Text = "ERRO [" .. step .. "]: " .. tostring(msg)
			lblErr.TextColor3 = C_ERR
			lblErr.Visible = true
		end
	end
	function ui.setToggleText(text) btnToggle.Text = text end
	function ui.setOutfitText(text) btnOutfit.Text = text end
	function ui.setActionsVisible(v)
		btnPunch.Visible = v
		btnGesture.Visible = v
		btnSlide.Visible = v
		btnParkour.Visible = v
		btnBreak.Visible = v
	end
	function ui.setParkourText(text) btnParkour.Text = text end
	return ui
end

local UI
do
	local ok, res = pcall(buildUI)
	if ok then
		UI = res
	else
		warn(TAG .. " Falha ao criar a interface: " .. tostring(res))
		local dummyBtn = { Activated = { Connect = function() return { Disconnect = function() end } end } }
		UI = setmetatable({ Toggle = dummyBtn, Outfit = dummyBtn, Punch = dummyBtn, Gesture = dummyBtn, Slide = dummyBtn, Parkour = dummyBtn, Break = dummyBtn }, {
			__index = function() return function() end end,
		})
	end
end

----------------------------------------------------------------------
-- Estado global
----------------------------------------------------------------------
local State = {
	Enabled = CONFIG.StartEnabled,
	OutfitIndex = 1,
	EmoteIdx = 0,
	Token = 0,         -- invalida pipelines antigos (respawn / toggle)
	Character = nil, Humanoid = nil, Root = nil,
	Rig = nil,
	Hidden = nil,      -- [Instance] = valores originais
	ToolHeld = false,
	LastTrace = nil,
}
local GlobalBin = newBin()
State.RigBin = newBin()

----------------------------------------------------------------------
-- Poses / animação (dados)
----------------------------------------------------------------------
local JOINT_NAMES = { "Root", "Waist", "Neck", "ShL", "ShR", "WrL", "WrR", "HipL", "HipR" }

local function newPose()
	local p = {}
	for _, n in ipairs(JOINT_NAMES) do
		p[n] = { rx = 0, ry = 0, rz = 0, tx = 0, ty = 0, tz = 0 }
	end
	return p
end

local function resetPose(p)
	for i = 1, #JOINT_NAMES do
		local j = p[JOINT_NAMES[i]]
		j.rx, j.ry, j.rz, j.tx, j.ty, j.tz = 0, 0, 0, 0, 0, 0
	end
end

local function newAnimState()
	return {
		target = newPose(), cur = newPose(),
		t = 0, phase = 0, landT = 0, landS = 0, wasAir = false, lastVy = 0, airTime = 0,
		plantW = 1, yaw = nil, yawRate = 0, action = nil, punchSide = -1, lastPunch = -10,
		doubleUsed = false, prevJumpReq = -10, flipT = nil, flipAngle = 0,
		deadT = 0, deadHidden = false, nextBlink = os.clock() + 2, blinkUntil = 0,
		label = "", lastCalib = 0, expr = "",
		peak = 0, skidT = 0, fidget = nil, nextFidget = os.clock() + 6, combo = 0, parkour = nil,
	}
end

----------------------------------------------------------------------
-- Construção do rig
----------------------------------------------------------------------
local function faceCF(x, y, thick, roll)
	local R = 5.0 -- raio da cabeça (mm)
	local sz = -math.sqrt(math.max(R * R - x * x, 0.01))
	local n = Vector3.new(x, 0, sz).Unit
	local pos = Vector3.new(x, y, sz) + n * (thick * 0.5 - 0.08)
	return CFrame.lookAt(pos, pos + n) * CFrame.Angles(0, 0, roll or 0)
end

local EXPR = {
	Happy      = { browY = 2.4, browRoll = 0,         eye = 1.0,  curve = 1.0,  halfW = 2.1, open = 0,   round = false },
	Determined = { browY = 2.2, browRoll = rad(24),   eye = 0.8,  curve = 0.2,  halfW = 1.8, open = 0,   round = false },
	Surprised  = { browY = 3.1, browRoll = rad(-6),   eye = 1.15, curve = 0,    halfW = 1.2, open = 1.9, round = true },
	Shout      = { browY = 2.1, browRoll = rad(26),   eye = 0.75, curve = -0.2, halfW = 2.0, open = 2.6, round = false },
	Laugh      = { browY = 2.7, browRoll = rad(-8),   eye = 0.6,  curve = 1.1,  halfW = 2.3, open = 1.6, round = false },
	KO         = { browY = 2.2, browRoll = rad(-18),  eye = 0.12, curve = -0.9, halfW = 1.9, open = 0,   round = false },
}

local function buildRig(character, hum, hrp, outfit)
	local rig = { Parts = {}, Rest = {}, JointOf = {}, Joints = {}, BaseC0 = {}, DriveMode = "C0", Active = false }
	local model = Instance.new("Model")
	model.Name = "LegoMinifigRig"
	rig.Model = model

	local g = hum.HipHeight + hrp.Size.Y * 0.5 + CONFIG.FootOffsetTweak
	rig.BaseG = g
	rig.GroundOffset = g
	rig.AppliedG = g
	local origin = hrp.CFrame * CFrame.new(0, -g, 0)

	local rp = RaycastParams.new()
	rp.FilterType = Enum.RaycastFilterType.Exclude
	rp.FilterDescendantsInstances = { character }
	rig.RayParams = rp

	local function toRest(c)
		local p = c.Position
		return CFrame.new(p.X * K, p.Y * K, p.Z * K) * (c - p)
	end

	local function weldTo(parentPart, childPart)
		local w = Instance.new("Weld")
		w.Name = "MF_Weld"
		w.Part0 = parentPart
		w.Part1 = childPart
		w.C0 = rig.Rest[parentPart]:Inverse() * rig.Rest[childPart]
		w.C1 = CFrame.new()
		w.Parent = childPart
		rig.JointOf[childPart] = w
		return w
	end

	-- size/cfmm em "mm". cylAxis: nil (bloco), "Y" (cilindro vertical), "Z" (disco virado p/ frente), "Face" (sem rotação extra)
	local function mk(name, size, cfmm, color, parentPart, cylAxis)
		local p = Instance.new("Part")
		p.Name = "MF_" .. name
		p.Anchored = false
		p.CanCollide = false
		p.CanTouch = false
		p.CanQuery = false
		p.Massless = true
		p.Material = Enum.Material.SmoothPlastic
		p.Color = color
		p.TopSurface = Enum.SurfaceType.Smooth
		p.BottomSurface = Enum.SurfaceType.Smooth
		local cf = cfmm
		if cylAxis then
			p.Shape = Enum.PartType.Cylinder -- eixo do cilindro = eixo X local; size = (comprimento, diâmetro, diâmetro)
			if cylAxis == "Y" then
				cf = cfmm * CFrame.Angles(0, 0, rad(90))
			elseif cylAxis == "Z" then
				cf = cfmm * CFrame.Angles(0, rad(90), 0)
			end
		end
		p.Size = size * K
		local rest = toRest(cf)
		p.CFrame = origin * rest
		p.Parent = model
		table.insert(rig.Parts, p)
		rig.Rest[p] = rest
		if parentPart then weldTo(parentPart, p) end
		return p
	end

	local function motor(key, name, parentPart, childPart, jointMM)
		local J = CFrame.new(jointMM.X * K, jointMM.Y * K, jointMM.Z * K)
		local m = Instance.new("Motor6D")
		m.Name = "MF_" .. name
		m.Part0 = parentPart
		m.Part1 = childPart
		m.C0 = rig.Rest[parentPart]:Inverse() * J
		m.C1 = rig.Rest[childPart]:Inverse() * J
		m.Parent = childPart
		rig.Joints[key] = m
		rig.BaseC0[key] = m.C0
		rig.JointOf[childPart] = m
		return m
	end

	local skin = outfit.Skin or COL.Skin

	-- Corpo principal (mm; origem = centro do chão entre os pés, frente = -Z)
	local hips = mk("Hips", Vector3.new(16, 5, 8), CFrame.new(0, 12.5, 0), outfit.Legs)
	local torso = mk("Torso", Vector3.new(16, 13, 8), CFrame.new(0, 21.5, 0), outfit.Torso)
	local head = mk("Head", Vector3.new(9, 10, 10), CFrame.new(0, 32.5, 0), skin, nil, "Y")
	rig.Hips, rig.Head, rig.Torso = hips, head, torso

	-- Braços e mãos em gancho
	local arms, hands = {}, {}
	for _, s in ipairs({ -1, 1 }) do
		local n = s < 0 and "L" or "R"
		local x = 10.4 * s
		local arm = mk("Arm" .. n, Vector3.new(7.8, 4.6, 4.6), CFrame.new(x, 23.4, 0), outfit.Arms, nil, "Y")
		local hand = mk("Hand" .. n, Vector3.new(1.6, 4.4, 4.4), CFrame.new(x, 18.7, 0), skin, nil, "Y")
		mk("HandBack" .. n, Vector3.new(3.4, 4.2, 1.2), CFrame.new(x, 15.8, 1.5), skin, hand)
		mk("HandTop" .. n, Vector3.new(3.4, 1.2, 4.2), CFrame.new(x, 17.3, 0), skin, hand)
		mk("HandBot" .. n, Vector3.new(3.4, 1.2, 4.2), CFrame.new(x, 14.3, 0), skin, hand)
		local att = Instance.new("Attachment")
		att.Name = (s > 0 and "Right" or "Left") .. "GripAttachment" -- ponto para armas/objetos futuros
		att.CFrame = rig.Rest[hand]:Inverse() * toRest(CFrame.new(x, 15.8, 0))
		att.Parent = hand
		arms[s], hands[s] = arm, hand
	end

	-- Pernas com botas, solas e biqueiras
	local legs = {}
	for _, s in ipairs({ -1, 1 }) do
		local n = s < 0 and "L" or "R"
		local x = 4.0 * s
		local leg = mk("Leg" .. n, Vector3.new(7.8, 11, 7.8), CFrame.new(x, 5.5, 0), outfit.Legs)
		mk("Boot" .. n, Vector3.new(7.95, 3.4, 8.05), CFrame.new(x, 1.7, 0), outfit.Boots, leg)
		mk("Sole" .. n, Vector3.new(7.9, 0.8, 9.6), CFrame.new(x, 0.4, -0.7), outfit.Sole, leg)
		mk("Toe" .. n, Vector3.new(7.6, 2.4, 1.5), CFrame.new(x, 1.6, -4.4), outfit.Boots, leg)
		legs[s] = leg
	end

	-- Detalhes do tronco (coordenadas locais ao tronco; frente = z -4)
	local function TCF(x, y, z, rollDeg)
		local c = CFrame.new(x, 21.5 + y, z)
		if rollDeg then c = c * CFrame.Angles(0, 0, rad(rollDeg)) end
		return c
	end

	if outfit.Style == "Jacket" then
		mk("Shirt", Vector3.new(5.2, 9.6, 0.4), TCF(0, 1.4, -4.12), outfit.Shirt, torso)
		mk("CollarL", Vector3.new(3.2, 1.0, 0.9), TCF(-2.4, 5.6, -4.3, -40), outfit.Collar, torso)
		mk("CollarR", Vector3.new(3.2, 1.0, 0.9), TCF(2.4, 5.6, -4.3, 40), outfit.Collar, torso)
		mk("StitchL", Vector3.new(0.4, 9.2, 0.25), TCF(-2.75, 1.4, -4.3), outfit.Stitch, torso)
		mk("StitchR", Vector3.new(0.4, 9.2, 0.25), TCF(2.75, 1.4, -4.3), outfit.Stitch, torso)
		for i, y in ipairs({ 4.0, 1.9, -0.2 }) do
			mk("Button" .. i, Vector3.new(0.35, 1.0, 1.0), TCF(0, y, -4.445), outfit.Stitch, torso, "Z")
		end
		for _, s in ipairs({ -1, 1 }) do
			local n = s < 0 and "L" or "R"
			mk("Pocket" .. n, Vector3.new(3.6, 3.0, 0.5), TCF(4.8 * s, -1.6, -4.17), outfit.Pocket, torso)
			mk("Flap" .. n, Vector3.new(3.8, 0.8, 0.6), TCF(4.8 * s, -0.2, -4.22), outfit.Flap, torso)
		end
		if outfit.Strap then
			mk("Strap", Vector3.new(1.8, 15.5, 0.6), TCF(0, 1.0, -4.55, -45), outfit.StrapColor, torso)
			mk("StrapBuckle", Vector3.new(2.4, 1.3, 0.35), TCF(-1.77, -0.77, -4.9, -45), outfit.Buckle, torso)
		end
	else
		mk("EmblemOuter", Vector3.new(5.6, 5.6, 0.5), TCF(0, 2.0, -4.17, 45), outfit.Accent, torso)
		mk("EmblemInner", Vector3.new(3.4, 3.4, 0.3), TCF(0, 2.0, -4.42, 45), outfit.Accent2, torso)
		mk("TrimL", Vector3.new(0.9, 10.5, 0.4), TCF(-6.2, 0.2, -4.12), outfit.Accent, torso)
		mk("TrimR", Vector3.new(0.9, 10.5, 0.4), TCF(6.2, 0.2, -4.12), outfit.Accent, torso)
		mk("CollarRing", Vector3.new(7.0, 1.0, 8.2), TCF(0, 6.0, 0), outfit.Accent2, torso)
	end
	mk("Belt", Vector3.new(16.3, 2.0, 8.3), TCF(0, -5.5, 0), outfit.Belt, torso)
	mk("Buckle", Vector3.new(3.4, 1.6, 0.5), TCF(0, -5.5, -4.35), outfit.Buckle, torso)
	mk("BuckleHole", Vector3.new(1.8, 0.7, 0.25), TCF(0, -5.5, -4.62), outfit.Belt, torso)

	-- Cabeça: pino, cabelo, chapéu ou capacete
	local hg = outfit.Headgear
	if hg == "Hair" then
		mk("HairTop", Vector3.new(3.0, 10.6, 10.6), CFrame.new(0, 37.8, 0), outfit.HeadgearColor, head, "Y")
		mk("HairBack", Vector3.new(10.4, 6.8, 3.4), CFrame.new(0, 34.4, 3.6), outfit.HeadgearColor, head)
		mk("HairStud", Vector3.new(1.6, 5.4, 5.4), CFrame.new(0, 40.1, 0), outfit.HeadgearColor, head, "Y")
	elseif hg == "Hat" then
		mk("HatBrim", Vector3.new(0.9, 15.0, 15.0), CFrame.new(0, 37.4, 0), outfit.HeadgearColor, head, "Y")
		mk("HatCrown", Vector3.new(4.4, 9.6, 9.6), CFrame.new(0, 39.8, 0), outfit.HeadgearColor, head, "Y")
		mk("HatBand", Vector3.new(1.1, 9.9, 9.9), CFrame.new(0, 38.4, 0), outfit.HatBand or COL.Tan, head, "Y")
	elseif hg == "Helmet" then
		mk("HelmCap", Vector3.new(3.2, 10.8, 10.8), CFrame.new(0, 37.6, 0), outfit.HeadgearColor, head, "Y")
		mk("HelmTop", Vector3.new(2.2, 9.6, 9.6), CFrame.new(0, 40.2, 0), outfit.HeadgearColor, head, "Y")
		mk("HelmFin", Vector3.new(1.0, 3.0, 6.4), CFrame.new(0, 42.2, 0), outfit.HeadgearColor, head)
		mk("HelmSideL", Vector3.new(1.2, 6.0, 4.4), CFrame.new(-5.6, 34.2, 0.8), outfit.HeadgearColor, head)
		mk("HelmSideR", Vector3.new(1.2, 6.0, 4.4), CFrame.new(5.6, 34.2, 0.8), outfit.HeadgearColor, head)
	else
		mk("Stud", Vector3.new(1.8, 6.0, 6.0), CFrame.new(0, 37.9, 0), skin, head, "Y")
	end

	-- Rosto (peças reaproveitadas para todas as expressões; sem decalques/IDs externos)
	local F = { Mouth = {} }
	local faceState = { expr = "Happy", blink = false }
	local function faceMk(name, size, color, disc)
		local p = mk(name, size, HEAD_CENTER, color, head, disc and "Face" or nil)
		return { Part = p, Weld = rig.JointOf[p] }
	end
	F.EyeL = faceMk("EyeL", Vector3.new(0.4, 1.6, 1.6), COL.Black, true)
	F.EyeR = faceMk("EyeR", Vector3.new(0.4, 1.6, 1.6), COL.Black, true)
	F.GlintL = faceMk("GlintL", Vector3.new(0.7, 0.55, 0.55), COL.White, true)
	F.GlintR = faceMk("GlintR", Vector3.new(0.7, 0.55, 0.55), COL.White, true)
	F.BrowL = faceMk("BrowL", Vector3.new(2.4, 0.55, 0.4), COL.Black, false)
	F.BrowR = faceMk("BrowR", Vector3.new(2.4, 0.55, 0.4), COL.Black, false)
	F.Nose = faceMk("Nose", Vector3.new(0.9, 1.2, 0.8), outfit.SkinDark or COL.SkinDark, false)
	for i = 1, 7 do
		F.Mouth[i] = faceMk("Mouth" .. i, Vector3.new(1, 0.5, 0.35), COL.Black, false)
	end
	F.MouthFill = faceMk("MouthFill", Vector3.new(0.35, 1.6, 1.6), COL.Black, true)

	local function placeFace(f, x, y, thick, roll, size, disc)
		local fcf = faceCF(x, y, thick, roll)
		if disc then fcf = fcf * CFrame.Angles(0, rad(90), 0) end
		f.Weld.C0 = rig.Rest[head]:Inverse() * toRest(HEAD_CENTER * fcf)
		f.Part.Size = size * K
	end

	local function applyFace()
		local e = EXPR[faceState.expr] or EXPR.Happy
		local eyeS = faceState.blink and 0.12 or e.eye
		local eyeSize = Vector3.new(0.4, 1.6 * eyeS, 1.6)
		placeFace(F.EyeL, -1.9, 0.7, 0.4, 0, eyeSize, true)
		placeFace(F.EyeR, 1.9, 0.7, 0.4, 0, eyeSize, true)
		placeFace(F.GlintL, -1.55, 0.7 + 0.4 * eyeS, 0.7, 0, Vector3.new(0.7, 0.55, 0.55), true)
		placeFace(F.GlintR, 2.25, 0.7 + 0.4 * eyeS, 0.7, 0, Vector3.new(0.7, 0.55, 0.55), true)
		local gt = eyeS > 0.3 and 0 or 1
		F.GlintL.Part.Transparency = gt
		F.GlintR.Part.Transparency = gt
		placeFace(F.BrowL, -2.1, e.browY, 0.4, -e.browRoll, Vector3.new(2.4, 0.55, 0.4), false)
		placeFace(F.BrowR, 2.1, e.browY, 0.4, e.browRoll, Vector3.new(2.4, 0.55, 0.4), false)
		placeFace(F.Nose, 0, -0.7, 0.8, 0, Vector3.new(0.9, 1.2, 0.8), false)
		local mouthY = -2.1
		local n = #F.Mouth
		local spacing = (2 * e.halfW) / (n - 1)
		for i = 1, n do
			local u = (i - 1) / (n - 1) * 2 - 1
			local x = u * e.halfW
			local y = mouthY + e.curve * u * u
			local slope = 2 * e.curve * u / e.halfW
			placeFace(F.Mouth[i], x, y, 0.35, math.atan(slope), Vector3.new(spacing * 1.35, 0.5, 0.35), false)
			F.Mouth[i].Part.Transparency = (e.open > 0 and e.round) and 1 or 0
		end
		if e.open > 0 then
			local w = e.round and e.open or e.open * 1.5
			placeFace(F.MouthFill, 0, mouthY - e.open * 0.15, 0.35, 0, Vector3.new(0.35, e.open, w), true)
			F.MouthFill.Part.Transparency = 0
		else
			F.MouthFill.Part.Transparency = 1
		end
	end

	function rig.SetExpression(name)
		if faceState.expr == name then return end
		faceState.expr = name
		applyFace()
	end
	function rig.SetBlink(b)
		if faceState.blink == b then return end
		faceState.blink = b
		applyFace()
	end
	rig.RefreshFace = applyFace
	applyFace()

	-- Articulações Motor6D (pivôs em mm no referencial do rig)
	motor("Waist", "Waist", hips, torso, Vector3.new(0, 15, 0))
	motor("Neck", "Neck", torso, head, Vector3.new(0, 28.2, 0))
	motor("ShL", "ShoulderL", torso, arms[-1], Vector3.new(-10.4, 25.8, 0))
	motor("ShR", "ShoulderR", torso, arms[1], Vector3.new(10.4, 25.8, 0))
	motor("WrL", "WristL", arms[-1], hands[-1], Vector3.new(-10.4, 19.5, 0))
	motor("WrR", "WristR", arms[1], hands[1], Vector3.new(10.4, 19.5, 0))
	motor("HipL", "HipL", hips, legs[-1], Vector3.new(-4.0, 10.5, 0))
	motor("HipR", "HipR", hips, legs[1], Vector3.new(4.0, 10.5, 0))

	-- Só agora o rig entra no personagem; o Motor6D raiz é criado no mesmo quadro (sem yield)
	model.Parent = character
	local rm = Instance.new("Motor6D")
	rm.Name = "MF_RootMotor"
	rm.Part0 = hrp
	rm.Part1 = hips
	rm.C0 = CFrame.new(0, -g, 0)
	rm.C1 = rig.Rest[hips]:Inverse()
	rm.Parent = hips
	rig.Joints.Root = rm
	rig.BaseC0.Root = rm.C0
	rig.JointOf[hips] = rm

	-- Aplica a pose de uma junta. Modo "C0" (padrão): o Animator não mexe em C0, então não sobrescreve.
	function rig.Drive(key, delta)
		local m = rig.Joints[key]
		if rig.DriveMode == "C0" then
			m.C0 = rig.BaseC0[key] * delta
		else
			m.Transform = delta
		end
	end

	rig.Anim = newAnimState()

	function rig.Destroy()
		rig.Active = false
		if model and model.Parent then model:Destroy() elseif model then model:Destroy() end
	end
	return rig
end

----------------------------------------------------------------------
-- Validação
----------------------------------------------------------------------
local function isFiniteCF(c)
	local x, y, z = c.X, c.Y, c.Z
	if x ~= x or y ~= y or z ~= z then return false end
	if abs(x) > 1e5 or abs(y) > 1e5 or abs(z) > 1e5 then return false end
	local l = c.LookVector
	return l.X == l.X and l.Y == l.Y and l.Z == l.Z
end

local function validateRigStatic(rig, character, hrp)
	if not rig.Model or rig.Model.Parent ~= character then
		error("Modelo da minifigura não está dentro do personagem", 0)
	end
	if #rig.Parts > CONFIG.MaxRigParts then
		error("Peças demais no rig: " .. #rig.Parts .. " (máx " .. CONFIG.MaxRigParts .. ")", 0)
	end
	for _, n in ipairs(JOINT_NAMES) do
		local m = rig.Joints[n]
		if not m or not m.Parent then error("Junta ausente: " .. n, 0) end
		if not m.Part0 or not m.Part1 then error("Junta sem Part0/Part1: " .. n, 0) end
		if not isFiniteCF(m.C0) or not isFiniteCF(m.C1) then error("C0/C1 inválido na junta " .. n, 0) end
		if not m.Part1:IsDescendantOf(rig.Model) then error("Part1 fora do rig na junta " .. n, 0) end
		if m.Part0 ~= hrp and not m.Part0:IsDescendantOf(rig.Model) then
			error("Part0 inválida na junta " .. n, 0)
		end
	end
	for _, p in ipairs(rig.Parts) do
		local j = rig.JointOf[p]
		if not j or not j.Parent or j.Part1 ~= p or not j.Part0 then error("Peça sem junta válida: " .. p.Name, 0) end
		if p.Anchored then error("Peça ancorada: " .. p.Name, 0) end
		local s = p.Size
		if not (s.X > 0 and s.Y > 0 and s.Z > 0) then error("Tamanho inválido: " .. p.Name, 0) end
		local cur, guard = p, 0
		while cur ~= hrp do
			local jj = rig.JointOf[cur]
			if not jj or not jj.Part0 then error("Cadeia de juntas quebrada em " .. cur.Name, 0) end
			cur = jj.Part0
			guard += 1
			if guard > 20 then error("Ciclo de juntas em " .. p.Name, 0) end
		end
	end
end

local function verifyRigPhysics(rig, hrp, hum)
	local rm = rig.Joints.Root
	local tol = 0.75 + hrp.AssemblyLinearVelocity.Magnitude * 0.05
	for _, part in ipairs({ rig.Hips, rig.Torso, rig.Head }) do
		local expected = hrp.CFrame * rm.C0 * rm.Transform * rig.Rest[rig.Hips]
		if part ~= rig.Hips then
			expected = hrp.CFrame * rm.C0 * rig.Rest[part]
		end
		local d = (part.Position - expected.Position).Magnitude
		if d > tol then
			error(("Peça %s fora do lugar (%.2f studs)"):format(part.Name, d), 0)
		end
	end
	if rig.Hips.AssemblyRootPart ~= hrp.AssemblyRootPart then
		error("Rig não pertence ao mesmo assembly do HumanoidRootPart", 0)
	end
	if hum.Health <= 0 or hum:GetState() == ST.Dead then
		error("Humanoid sem vida durante a validação", 0)
	end
end

----------------------------------------------------------------------
-- Ocultar / restaurar o avatar original (somente local)
----------------------------------------------------------------------
local function isRigInstance(inst)
	return inst.Name == "LegoMinifigRig" or inst:FindFirstAncestor("LegoMinifigRig") ~= nil
end

local function shouldHide(inst)
	if isRigInstance(inst) then return false end
	if inst:FindFirstAncestorWhichIsA("Tool") then return false end
	return inst:IsA("BasePart") or inst:IsA("Decal")
end

local function hideOne(inst)
	local H = State.Hidden
	if not H or H[inst] then return end
	if inst:IsA("BasePart") then
		H[inst] = { Transparency = inst.Transparency, CastShadow = inst.CastShadow }
		inst.Transparency = 1
		inst.CastShadow = false
	elseif inst:IsA("Decal") then
		H[inst] = { Transparency = inst.Transparency }
		inst.Transparency = 1
	end
end

local function hideOriginal(character)
	State.Hidden = {}
	for _, inst in ipairs(character:GetDescendants()) do
		if shouldHide(inst) then hideOne(inst) end
	end
	State.RigBin:Add(character.DescendantAdded:Connect(function(inst)
		if State.Rig and shouldHide(inst) then hideOne(inst) end
	end))
end

local function restoreOriginal()
	local H = State.Hidden
	State.Hidden = nil
	if not H then return end
	for inst, rec in pairs(H) do
		if inst.Parent then
			local ok, e = pcall(function()
				inst.Transparency = rec.Transparency
				if rec.CastShadow ~= nil then inst.CastShadow = rec.CastShadow end
			end)
			if not ok then warn(TAG .. " falha ao restaurar " .. inst.Name .. ": " .. tostring(e)) end
		end
	end
end

----------------------------------------------------------------------
-- Poses
----------------------------------------------------------------------
local function ov(j, f, v, w) j[f] = j[f] + (v - j[f]) * w end

local function legDrop(rx, rz)
	return 10.5 * cos(rx) * cos(rz) + (rx > 0 and 5.4 or 4.1) * abs(sin(rx)) + 3.9 * abs(sin(rz))
end

local function poseGround(A, T, speed, yawRate, dt)
	local t = A.t
	local runB = smooth(clamp((speed - 7) / 6, 0, 1))
	local moveW = clamp(speed / 3, 0, 1)
	local idle = 1 - moveW
	if speed > 0.6 then
		A.phase += (speed / lerp(4.0, 5.4, runB)) * 2 * math.pi * dt
	end
	local ph = A.phase
	local s = sin(ph)
	local legAmp = rad(lerp(34, 58, runB)) * moveW
	T.HipL.rx = s * legAmp
	T.HipR.rx = -s * legAmp
	T.HipL.rz = -rad(2)
	T.HipR.rz = rad(2)
	local armAmp = rad(lerp(30, 64, runB)) * moveW
	T.ShL.rx = -s * armAmp
	T.ShR.rx = s * armAmp
	local turnK = clamp(speed / 4, 0, 1)
	local abd = rad(4) + rad(10) * runB * moveW + clamp(abs(yawRate) * 0.02, 0, rad(16)) * turnK
		+ rad(1.5) * sin(t * 1.3) * idle
	T.ShL.rz = -abd
	T.ShR.rz = abd
	T.WrL.rx = -T.ShL.rx * 0.3
	T.WrR.rx = -T.ShR.rx * 0.3
	local lean = -(rad(3) + rad(12) * runB) * moveW
	local turnLean = clamp(yawRate * 0.025, -rad(12), rad(12)) * turnK
	T.Waist.rx = lean
	T.Waist.ry = s * rad(5 + 7 * runB) * moveW - clamp(yawRate * 0.02, -rad(10), rad(10)) * turnK
		+ sin(t * 0.8) * rad(1.6) * idle
	T.Waist.rz = turnLean
	T.Neck.rx = -lean * 0.7 + sin(ph * 2) * rad(2.5) * moveW + sin(t * 0.9) * rad(2) * idle
	T.Neck.ry = clamp(yawRate * 0.03, -rad(20), rad(20)) * turnK
		+ (sin(t * 0.55) * rad(14) + sin(t * 1.3) * rad(4)) * idle
	T.Neck.rz = -turnLean * 0.6
	T.Root.ty = sin(t * 2.4) * 0.015 * idle
	T.Root.rz = -s * rad(3 + 2 * runB) * moveW -- balanço lateral característico
	T.Waist.rz += s * rad(3.5) * moveW
	if State.ToolHeld then -- pose de "segurar objeto" (visual)
		T.ShR.rx = rad(78)
		T.ShR.rz = rad(4)
		T.WrR.rx = -rad(8)
	end
end

local function poseAir(A, T, vy)
	local t = A.t
	local jw = clamp(vy / 12, 0, 1)
	local fw = 1 - jw
	local flail = sin(t * 13)
	local armRx = lerp(rad(118), rad(150), jw)
	T.ShL.rx = armRx + flail * rad(16) * fw
	T.ShR.rx = armRx - flail * rad(16) * fw
	T.ShL.rz = -lerp(rad(32), rad(14), jw)
	T.ShR.rz = lerp(rad(32), rad(14), jw)
	T.WrL.rx = -rad(10)
	T.WrR.rx = -rad(10)
	local kick = sin(t * 9) * rad(24) * fw
	T.HipL.rx = kick + rad(28) * jw
	T.HipR.rx = -kick - rad(14) * jw
	T.HipL.rz = -rad(8)
	T.HipR.rz = rad(8)
	T.Waist.rx = rad(6) * jw - rad(3) * fw
	T.Neck.rx = rad(10) * jw
	T.Neck.ry = sin(t * 6) * rad(6) * fw
end

local function poseSeated(T)
	T.HipL.rx = rad(88)
	T.HipR.rx = rad(88)
	T.HipL.rz = -rad(3)
	T.HipR.rz = rad(3)
	T.ShL.rx = rad(35)
	T.ShR.rx = rad(35)
	T.ShL.rz = -rad(6)
	T.ShR.rz = rad(6)
end

local function poseClimb(A, T, vy, dt)
	A.phase += clamp(abs(vy) / 3, 0, 3) * math.pi * dt
	local s = sin(A.phase)
	T.ShL.rx = rad(130) + s * rad(25)
	T.ShR.rx = rad(130) - s * rad(25)
	T.HipL.rx = rad(15) - s * rad(30)
	T.HipR.rx = rad(15) + s * rad(30)
	T.Waist.rx = -rad(5)
end

local function poseDead(A, T)
	local u = clamp(A.deadT / 1.1, 0, 1)
	local e = easeOut(u)
	local arc = sin(u * math.pi)
	T.Neck.ty = (arc * 18 + e * 4) * K
	T.Neck.ry = u * rad(540)
	T.Neck.rx = u * rad(120)
	T.Waist.ty = arc * 5 * K
	T.Waist.rx = u * rad(40)
	for _, s in ipairs({ -1, 1 }) do
		local sh = s < 0 and T.ShL or T.ShR
		local wr = s < 0 and T.WrL or T.WrR
		local hp = s < 0 and T.HipL or T.HipR
		sh.tx = s * e * 16 * K
		sh.ty = (arc * 10 - u * 6) * K
		sh.rx = u * rad(500)
		sh.rz = s * rad(60) * u
		wr.tx = s * e * 8 * K
		wr.ty = -u * 8 * K
		wr.rx = u * rad(720)
		hp.tx = s * e * 8 * K
		hp.ty = (arc * 6 - u * 8) * K
		hp.rz = s * rad(40) * u
		hp.rx = u * rad(200)
	end
end

local function overlayLanding(T, A)
	local w = smooth(A.landT) * A.landS
	T.Root.ty += -1.6 * K * w
	T.Root.rx += -rad(6) * w
	T.HipL.rz += -rad(18) * w
	T.HipR.rz += rad(18) * w
	T.Waist.rx += -rad(16) * w
	T.Neck.rx += rad(10) * w
	T.ShL.rz += -rad(20) * w
	T.ShR.rz += rad(20) * w
	T.ShL.rx += rad(15) * w
	T.ShR.rx += rad(15) * w
end

local function overlayPunch(T, act)
	local t, side = act.t, act.side
	local armRx, yaw, lean, pw
	if t < 0.12 then -- antecipação
		local u = easeOut(t / 0.12)
		armRx = lerp(0, -rad(60), u); yaw = lerp(0, -rad(24), u); lean = lerp(0, rad(4), u); pw = t / 0.22
	elseif t < 0.22 then -- golpe
		local u = easeIn((t - 0.12) / 0.10)
		armRx = lerp(-rad(60), rad(95), u); yaw = lerp(-rad(24), rad(32), u); lean = lerp(rad(4), -rad(10), u); pw = t / 0.22
	elseif t < 0.30 then -- impacto
		armRx = rad(95); yaw = rad(32); lean = -rad(10); pw = 1
	else -- retorno
		local u = smooth(clamp((t - 0.30) / 0.22, 0, 1))
		armRx = lerp(rad(95), 0, u); yaw = lerp(rad(32), 0, u); lean = lerp(-rad(10), 0, u); pw = 1 - u
	end
	local arm = side == 1 and T.ShR or T.ShL
	local other = side == 1 and T.ShL or T.ShR
	local wr = side == 1 and T.WrR or T.WrL
	arm.rx = armRx
	arm.rz = side * rad(5)
	wr.rx = -rad(8)
	other.rx = rad(38) * pw
	other.rz = -side * rad(14)
	T.Waist.ry = yaw * side
	T.Waist.rx = lean
	T.Waist.rz = 0
	T.Neck.ry = -yaw * side * 0.6
	T.Neck.rx = -lean * 0.5
	local front = side == 1 and T.HipL or T.HipR
	local back = side == 1 and T.HipR or T.HipL
	front.rx = rad(18) * pw
	back.rx = -rad(12) * pw
	T.Root.rx = -rad(5) * pw
end

local function overlayEmote(T, act)
	local t = act.t
	local w = clamp(t / 0.12, 0, 1) * clamp((act.dur - t) / 0.2, 0, 1)
	local hop = abs(sin(t * math.pi * 3.2))
	local flap = sin(t * 18)
	ov(T.ShL, "rx", rad(158) + flap * rad(14), w)
	ov(T.ShR, "rx", rad(158) - flap * rad(14), w)
	ov(T.ShL, "rz", -rad(24), w)
	ov(T.ShR, "rz", rad(24), w)
	ov(T.WrL, "rx", 0, w)
	ov(T.WrR, "rx", 0, w)
	ov(T.HipL, "rx", 0, w)
	ov(T.HipR, "rx", 0, w)
	ov(T.HipL, "rz", -rad(14) * hop, w)
	ov(T.HipR, "rz", rad(14) * hop, w)
	ov(T.Root, "ty", hop * 7 * K, w)
	ov(T.Waist, "rx", 0, w)
	ov(T.Waist, "rz", sin(t * 9) * rad(7), w)
	ov(T.Neck, "ry", sin(t * 11) * rad(16), w)
	ov(T.Neck, "rx", rad(8), w)
end

local function overlayFlip(T, u)
	local w = sin(clamp(u, 0, 1) * math.pi)
	ov(T.HipL, "rx", rad(80), w)
	ov(T.HipR, "rx", rad(80), w)
	ov(T.ShL, "rx", rad(70), w)
	ov(T.ShR, "rx", rad(70), w)
	ov(T.Waist, "rx", -rad(15), w)
end

----------------------------------------------------------------------
-- Animações extras (estilo jogos LEGO)
----------------------------------------------------------------------
local EMOTES = { "Comemorar", "Acenar", "Dançar" }
local FIDGETS = { "look", "shrug", "stretch", "tap" }

local function overlayKick(T, act)
	local t = act.t
	local k, lean
	if t < 0.15 then
		local u = easeOut(t / 0.15)
		k = lerp(0, -rad(30), u); lean = lerp(0, rad(6), u)
	elseif t < 0.28 then
		local u = easeIn((t - 0.15) / 0.13)
		k = lerp(-rad(30), rad(88), u); lean = lerp(rad(6), rad(14), u)
	elseif t < 0.38 then
		k = rad(88); lean = rad(14)
	else
		local u = smooth(clamp((t - 0.38) / 0.24, 0, 1))
		k = lerp(rad(88), 0, u); lean = lerp(rad(14), 0, u)
	end
	T.HipR.rx = k
	T.HipL.rx = -rad(6)
	T.HipR.rz = rad(4)
	T.Waist.rx = lean
	T.Neck.rx = -lean * 0.6
	T.ShL.rz = -rad(38)
	T.ShR.rz = rad(38)
	T.ShL.rx = rad(25)
	T.ShR.rx = -rad(20)
end

local function overlayWave(T, act)
	local t = act.t
	local w = clamp(t / 0.15, 0, 1) * clamp((act.dur - t) / 0.2, 0, 1)
	ov(T.ShR, "rx", rad(20), w)
	ov(T.ShR, "rz", rad(115) + sin(t * 12) * rad(20), w)
	ov(T.WrR, "rx", sin(t * 12) * rad(10), w)
	ov(T.Neck, "rz", -rad(8), w)
	ov(T.Waist, "rz", rad(3), w)
	ov(T.Root, "ty", abs(sin(t * 6)) * 0.5 * K, w)
end

local function overlayDance(T, act)
	local t = act.t
	local w = clamp(t / 0.15, 0, 1) * clamp((act.dur - t) / 0.2, 0, 1)
	local s = sin(t * 8)
	ov(T.ShL, "rx", rad(90) + s * rad(70), w)
	ov(T.ShR, "rx", rad(90) - s * rad(70), w)
	ov(T.ShL, "rz", -rad(20), w)
	ov(T.ShR, "rz", rad(20), w)
	ov(T.HipL, "rx", s * rad(28), w)
	ov(T.HipR, "rx", -s * rad(28), w)
	ov(T.Waist, "ry", s * rad(28), w)
	ov(T.Waist, "rz", cos(t * 8) * rad(8), w)
	ov(T.Neck, "ry", -s * rad(20), w)
	ov(T.Root, "ty", abs(sin(t * 8)) * 3 * K, w)
	ov(T.Root, "rz", s * rad(5), w)
end

local function overlayEmoteKind(T, act)
	if act.name == "Acenar" then
		overlayWave(T, act)
	elseif act.name == "Dançar" then
		overlayDance(T, act)
	else
		overlayEmote(T, act)
	end
end

local function overlaySkid(T, A)
	local w = smooth(A.skidT)
	ov(T.Waist, "rx", rad(16), w)
	ov(T.Neck, "rx", -rad(8), w)
	ov(T.ShL, "rx", rad(55), w)
	ov(T.ShR, "rx", rad(55), w)
	ov(T.ShL, "rz", -rad(25), w)
	ov(T.ShR, "rz", rad(25), w)
	ov(T.HipL, "rx", rad(34), w)
	ov(T.HipR, "rx", -rad(14), w)
	ov(T.Root, "rx", rad(6), w)
end

local function overlayFidget(T, f)
	local u = clamp(f.t / f.dur, 0, 1)
	local w = clamp(sin(u * math.pi) * 2.2, 0, 1)
	if f.kind == "look" then
		ov(T.Neck, "ry", sin(u * math.pi * 2) * rad(50), w)
		ov(T.Waist, "ry", sin(u * math.pi * 2) * rad(14), w)
	elseif f.kind == "shrug" then
		ov(T.ShL, "rz", -rad(55), w)
		ov(T.ShR, "rz", rad(55), w)
		ov(T.ShL, "rx", rad(15), w)
		ov(T.ShR, "rx", rad(15), w)
		ov(T.Neck, "rz", sin(u * math.pi * 4) * rad(8), w)
	elseif f.kind == "stretch" then
		ov(T.ShL, "rx", rad(172), w)
		ov(T.ShR, "rx", rad(172), w)
		ov(T.ShL, "rz", -rad(14), w)
		ov(T.ShR, "rz", rad(14), w)
		ov(T.Waist, "rx", rad(10), w)
		ov(T.Neck, "rx", rad(12), w)
	else -- tap: bate o pé
		local tap = abs(sin(f.t * 11))
		ov(T.HipR, "rx", rad(18) * tap, w)
		ov(T.Neck, "rx", rad(4) * sin(f.t * 11), w)
		ov(T.ShL, "rx", -rad(10), w)
		ov(T.ShR, "rx", -rad(10), w)
	end
end

----------------------------------------------------------------------
-- PARKOUR + DESTRUIÇÃO VISUAL LOCAL (V4)
----------------------------------------------------------------------
local Parkour = {
	mode = "None", -- None, Slide, Hang, Mantle
	endsAt = 0, cooldown = 0, direction = nil, hangPosition = nil,
	controlHumanoid = nil, savedWalkSpeed = nil, savedAutoRotate = nil,
}
local Destruction = { Entries = {}, FX = nil, scanAt = 0, running = true }
local PROTECTED_WORDS = {
	"floor", "ground", "baseplate", "spawn", "checkpoint", "foundation", "pillar", "column",
	"support", "lobby", "terrain", "boundary", "hitbox", "walkway", "platform", "road",
	"bridge", "staircase", "stair", "building", "structure", "tower", "invisible wall",
}
local LIKELY_WORDS = {
	"breakable", "crate", "barrel", "box", "wood", "glass", "cabinet", "chair", "table",
	"lamp", "fence", "door", "window", "prop", "debris", "rock", "sign", "shelf", "panel",
}

local function destroyIfAlive(inst)
	if inst and inst.Parent then pcall(function() inst:Destroy() end) end
end

local function ensureFXFolder()
	if Destruction.FX and Destruction.FX.Parent then return Destruction.FX end
	local folder = Instance.new("Folder")
	folder.Name = "LegoLocalFX_V4"
	folder.Parent = Workspace
	Destruction.FX = folder
	return folder
end

local function clearDestruction()
	for part, entry in pairs(Destruction.Entries) do
		if part and part.Parent and entry.originalLTM ~= nil then
			pcall(function() part.LocalTransparencyModifier = entry.originalLTM end)
		end
		destroyIfAlive(entry.highlight)
		for _, crack in ipairs(entry.cracks or {}) do destroyIfAlive(crack) end
	end
	Destruction.Entries = {}
	destroyIfAlive(Destruction.FX)
	Destruction.FX = nil
	Destruction.scanAt = 0
end

local function saveMovementControl(hum)
	if Parkour.controlHumanoid == hum then return end
	Parkour.controlHumanoid = hum
	Parkour.savedWalkSpeed = hum.WalkSpeed
	Parkour.savedAutoRotate = hum.AutoRotate
end

local function restoreMovementControl()
	local hum = Parkour.controlHumanoid
	if hum and hum.Parent then
		if Parkour.savedWalkSpeed ~= nil then hum.WalkSpeed = Parkour.savedWalkSpeed end
		if Parkour.savedAutoRotate ~= nil then hum.AutoRotate = Parkour.savedAutoRotate end
	end
	Parkour.controlHumanoid = nil
	Parkour.savedWalkSpeed = nil
	Parkour.savedAutoRotate = nil
end

local function resetParkour(restore)
	if restore then restoreMovementControl() end
	Parkour.mode = "None"
	Parkour.endsAt = 0
	Parkour.direction = nil
	Parkour.hangPosition = nil
	if State.Rig and State.Rig.Anim then State.Rig.Anim.parkour = nil end
	UI.setParkourText("PARKOUR")
end

local function getRayParams()
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	local exclude = {}
	if State.Character then table.insert(exclude, State.Character) end
	if Destruction.FX then table.insert(exclude, Destruction.FX) end
	params.FilterDescendantsInstances = exclude
	params.IgnoreWater = true
	return params
end

local function flatDirection()
	local hrp, hum = State.Root, State.Humanoid
	if not hrp then return Vector3.new(0, 0, -1) end
	local dir = hum and hum.MoveDirection or Vector3.zero
	if dir.Magnitude < 0.2 then dir = hrp.CFrame.LookVector end
	dir = Vector3.new(dir.X, 0, dir.Z)
	if dir.Magnitude < 0.05 then dir = Vector3.new(0, 0, -1) end
	return dir.Unit
end

local function setParkourAnimation(kind, duration)
	local rig = State.Rig
	if not (rig and rig.Anim) then return end
	rig.Anim.action = nil
	rig.Anim.fidget = nil
	rig.Anim.parkour = { kind = kind, t = 0, dur = duration or 1 }
end

local function finishSlide()
	if Parkour.mode ~= "Slide" then return end
	restoreMovementControl()
	Parkour.mode = "None"
	Parkour.endsAt = 0
end

local function startSlide()
	local hum, hrp, rig = State.Humanoid, State.Root, State.Rig
	if not (hum and hrp and rig and rig.Active and hum.Health > 0) then return end
	local now = os.clock()
	if now < Parkour.cooldown or Parkour.mode == "Hang" or Parkour.mode == "Mantle" then return end
	local state = hum:GetState()
	local grounded = state ~= ST.Freefall and state ~= ST.Jumping and state ~= ST.FallingDown and state ~= ST.Swimming
	local direction = flatDirection()
	local velocity = hrp.AssemblyLinearVelocity
	local speed = Vector3.new(velocity.X, 0, velocity.Z).Magnitude
	if not grounded or speed < 5 or hum.Sit then
		UI.setAnim("Slide: corra e pressione novamente", C_WARN)
		return
	end
	if Parkour.mode == "Slide" then return end
	saveMovementControl(hum)
	Parkour.mode = "Slide"
	Parkour.direction = direction
	Parkour.endsAt = now + 0.72
	Parkour.cooldown = now + 1.05
	hum.WalkSpeed = math.max(7, (Parkour.savedWalkSpeed or hum.WalkSpeed) * 0.48)
	local boost = math.min(25, math.max(15, speed + 5))
	hrp.AssemblyLinearVelocity = direction * boost + Vector3.new(0, math.min(velocity.Y, 0), 0)
	setParkourAnimation("slide", 0.72)
end

local function beginMantle()
	local hum, hrp = State.Humanoid, State.Root
	if not (hum and hrp and hum.Health > 0) then return end
	local dir = Parkour.direction or flatDirection()
	restoreMovementControl()
	Parkour.mode = "Mantle"
	Parkour.endsAt = os.clock() + 0.72
	Parkour.hangPosition = nil
	local v = hrp.AssemblyLinearVelocity
	hrp.AssemblyLinearVelocity = Vector3.new(dir.X * 12, math.max(23, v.Y), dir.Z * 12)
	hum:ChangeState(ST.Jumping)
	setParkourAnimation("mantle", 0.82)
	UI.setParkourText("PARKOUR")
end

local function tryWallJump()
	local hum, hrp = State.Humanoid, State.Root
	if not (hum and hrp and hum.Health > 0) then return false end
	if os.clock() < Parkour.cooldown then return false end
	local st = hum:GetState()
	if st ~= ST.Freefall and st ~= ST.Jumping then return false end
	local dir = flatDirection()
	local params = getRayParams()
	local hit = Workspace:Raycast(hrp.Position + Vector3.new(0, 0.4, 0), dir * 2.6, params)
	if not hit or hit.Normal.Y > 0.45 then return false end
	local away = Vector3.new(hit.Normal.X, 0, hit.Normal.Z)
	if away.Magnitude < 0.1 then away = -dir else away = away.Unit end
	local v = hrp.AssemblyLinearVelocity
	hrp.AssemblyLinearVelocity = away * 20 + Vector3.new(0, math.max(22, v.Y + 9), 0)
	setParkourAnimation("walljump", 0.58)
	Parkour.cooldown = os.clock() + 0.32
	return true
end

local function performParkour()
	local hum, hrp, rig = State.Humanoid, State.Root, State.Rig
	if not (hum and hrp and rig and rig.Active and hum.Health > 0) then return end
	if Parkour.mode == "Hang" then beginMantle(); return end
	if Parkour.mode == "Slide" or Parkour.mode == "Mantle" then return end
	if tryWallJump() then return end
	local now = os.clock()
	if now < Parkour.cooldown then return end
	local dir = flatDirection()
	local params = getRayParams()
	local groundY = hrp.Position.Y - hum.HipHeight - hrp.Size.Y * 0.5
	local lowOrigin = hrp.Position - Vector3.new(0, hum.HipHeight + hrp.Size.Y * 0.5, 0) + Vector3.new(0, 0.55, 0)
	local lowHit = Workspace:Raycast(lowOrigin, dir * CONFIG.ParkourScanDistance, params)
	local highHit = Workspace:Raycast(hrp.Position + Vector3.new(0, 1.75, 0), dir * CONFIG.ParkourScanDistance, params)
	local topOrigin = hrp.Position + dir * 2.25 + Vector3.new(0, 5.5, 0)
	local topHit = Workspace:Raycast(topOrigin, Vector3.new(0, -8.0, 0), params)
	local topHeight = topHit and (topHit.Position.Y - groundY) or math.huge
	if lowHit and lowHit.Normal.Y < 0.55 and topHit and topHit.Normal.Y > 0.55 and topHeight > 0.25 and topHeight <= 5.3 then
		Parkour.direction = dir
		if topHeight <= 1.7 and not highHit then
			local v = hrp.AssemblyLinearVelocity
			hrp.AssemblyLinearVelocity = Vector3.new(dir.X * math.max(17, Vector3.new(v.X,0,v.Z).Magnitude + 5), 13, dir.Z * math.max(17, Vector3.new(v.X,0,v.Z).Magnitude + 5))
			hum:ChangeState(ST.Jumping)
			setParkourAnimation("vault", 0.62)
			Parkour.cooldown = now + 0.45
		else
			-- Pendura ligeiramente abaixo da borda e trava só o controle local enquanto segura.
			local target = topHit.Position - dir * 1.18 - Vector3.new(0, 2.25, 0)
			if (target - hrp.Position).Magnitude > 5.8 then
				UI.setAnim("Parkour: borda fora de alcance", C_WARN)
				return
			end
			saveMovementControl(hum)
			Parkour.mode = "Hang"
			Parkour.hangPosition = target
			Parkour.endsAt = 0
			hum.WalkSpeed = 0
			hum.AutoRotate = false
			hrp.AssemblyLinearVelocity = Vector3.zero
			hum:ChangeState(ST.Physics)
			setParkourAnimation("hang", 9999)
			UI.setParkourText("SUBIR")
		end
		return
	end
	if lowHit and lowHit.Normal.Y < 0.55 and not highHit then
		-- Obstáculo baixo: vault curto em vez de tentar atravessar uma parede alta.
		local v = hrp.AssemblyLinearVelocity
		local push = math.max(16, Vector3.new(v.X,0,v.Z).Magnitude + 3)
		hrp.AssemblyLinearVelocity = Vector3.new(dir.X * push, 11, dir.Z * push)
		hum:ChangeState(ST.Jumping)
		setParkourAnimation("vault", 0.58)
		Parkour.cooldown = now + 0.42
	else
		UI.setAnim("Parkour: nenhuma borda segura detectada", C_WARN)
	end
end

local function updateParkourMotion(dt)
	local hum, hrp = State.Humanoid, State.Root
	if not (hum and hrp and hum.Parent and hrp.Parent and hum.Health > 0) then
		if Parkour.mode ~= "None" then resetParkour(true) end
		return
	end
	local now = os.clock()
	if Parkour.mode == "Slide" and now >= Parkour.endsAt then
		finishSlide()
	elseif Parkour.mode == "Mantle" and now >= Parkour.endsAt then
		Parkour.mode = "None"
		Parkour.endsAt = 0
	end
	if Parkour.mode == "Hang" and Parkour.hangPosition then
		-- Correção de velocidade suave, sem ancorar nem teleportar o personagem.
		local delta = Parkour.hangPosition - hrp.Position
		if delta.Magnitude > 7 then
			resetParkour(true)
			hum:ChangeState(ST.Freefall)
			return
		end
		local correction = delta * 9 - hrp.AssemblyLinearVelocity * 0.35 + Vector3.new(0, Workspace.Gravity * dt * 0.8, 0)
		if correction.Magnitude > 17 then correction = correction.Unit * 17 end
		hrp.AssemblyLinearVelocity = correction
	end
end

local function overlayParkour(T, p)
	local t = p.t or 0
	local duration = math.max(0.01, p.dur or 0.7)
	local u = clamp(t / duration, 0, 1)
	local w = p.kind == "hang" and 1 or sin(u * math.pi)
	if p.kind == "slide" then
		ov(T.Waist, "rx", rad(24), 0.95)
		ov(T.Neck, "rx", -rad(13), 0.9)
		ov(T.HipL, "rx", rad(72), 0.95)
		ov(T.HipR, "rx", rad(54), 0.95)
		ov(T.ShL, "rx", rad(54), 0.85); ov(T.ShR, "rx", rad(48), 0.85)
		ov(T.ShL, "rz", -rad(18), 0.8); ov(T.ShR, "rz", rad(18), 0.8)
	elseif p.kind == "vault" then
		ov(T.Waist, "rx", -rad(20), w)
		ov(T.ShL, "rx", rad(145), w); ov(T.ShR, "rx", rad(145), w)
		ov(T.ShL, "rz", -rad(22), w); ov(T.ShR, "rz", rad(22), w)
		ov(T.HipL, "rx", rad(78), w); ov(T.HipR, "rx", rad(70), w)
		ov(T.Neck, "rx", rad(8), w)
	elseif p.kind == "hang" then
		local sway = sin(t * 3) * rad(3)
		ov(T.ShL, "rx", rad(160) + sway, 1); ov(T.ShR, "rx", rad(160) - sway, 1)
		ov(T.ShL, "rz", -rad(15), 1); ov(T.ShR, "rz", rad(15), 1)
		ov(T.HipL, "rx", rad(26) + sin(t * 4) * rad(5), 1)
		ov(T.HipR, "rx", rad(34) - sin(t * 4) * rad(5), 1)
		ov(T.Waist, "rx", -rad(12), 1); ov(T.Neck, "rx", rad(8), 1)
	elseif p.kind == "mantle" then
		ov(T.ShL, "rx", rad(145), w); ov(T.ShR, "rx", rad(145), w)
		ov(T.Waist, "rx", -rad(28), w)
		ov(T.HipL, "rx", rad(90) * w, w); ov(T.HipR, "rx", rad(75) * w, w)
		ov(T.Neck, "rx", -rad(8), w)
	elseif p.kind == "walljump" then
		ov(T.ShL, "rx", rad(150), w); ov(T.ShR, "rx", rad(150), w)
		ov(T.ShL, "rz", -rad(32), w); ov(T.ShR, "rz", rad(32), w)
		ov(T.HipL, "rx", rad(45), w); ov(T.HipR, "rx", -rad(25), w)
		ov(T.Waist, "rx", -rad(12), w)
	elseif p.kind == "roll" then
		ov(T.Waist, "rx", rad(42), w)
		ov(T.HipL, "rx", rad(86), w); ov(T.HipR, "rx", rad(82), w)
		ov(T.ShL, "rx", rad(95), w); ov(T.ShR, "rx", rad(95), w)
		ov(T.Neck, "rx", -rad(18), w)
	end
end

local function classifyDestructible(part)
	if not part or not part:IsA("BasePart") or not part.Parent then return false end
	if part:GetAttribute("LegoBreakable") == false then return false end
	if part:GetAttribute("LegoBreakable") == true then return true end
	if not part.Anchored or not part.CanCollide or part.Transparency > 0.35 or (State.Character and part:IsDescendantOf(State.Character)) then return false end
	local name = string.lower(part.Name)
	local explicit = string.find(name, "breakable", 1, true) ~= nil
	for _, word in ipairs(PROTECTED_WORDS) do
		if string.find(name, word, 1, true) then return false end
	end
	local ancestor = part.Parent
	for _ = 1, 3 do
		if not ancestor or ancestor == Workspace then break end
		local an = string.lower(ancestor.Name)
		if string.find(an, "breakable", 1, true) then explicit = true end
		for _, word in ipairs({"baseplate", "terrain", "foundation", "building_shell", "main_structure", "collision", "hitbox"}) do
			if string.find(an, word, 1, true) then return false end
		end
		ancestor = ancestor.Parent
	end
	local size = part.Size
	local maxSide = math.max(size.X, size.Y, size.Z)
	local minSide = math.min(size.X, size.Y, size.Z)
	local volume = size.X * size.Y * size.Z
	if maxSide > 10 or volume > 300 or minSide < 0.12 then return false end
	if maxSide > 6 and minSide < 0.55 then return false end -- evita la maioria dos pisos/placas finas
	local score = explicit and 4 or 0
	for _, word in ipairs(LIKELY_WORDS) do
		if string.find(name, word, 1, true) then score += 2; break end
	end
	local material = part.Material
	if material == Enum.Material.Wood or material == Enum.Material.WoodPlanks or material == Enum.Material.Glass
		or material == Enum.Material.Metal or material == Enum.Material.DiamondPlate or material == Enum.Material.CorrodedMetal
		or material == Enum.Material.Brick or material == Enum.Material.Concrete or material == Enum.Material.Cobblestone then
		score += 1
	end
	if maxSide <= 4.5 and volume <= 90 and minSide >= 0.2 then score += 1 end
	if part:IsA("MeshPart") and not explicit and score < 3 then return false end
	return score >= 2
end

local function makeHighlight(part)
	local folder = ensureFXFolder()
	local h = Instance.new("Highlight")
	h.Name = "LegoBreakableOutline"
	h.Adornee = part
	h.FillTransparency = 1
	h.OutlineColor = Color3.fromRGB(142, 145, 151)
	h.OutlineTransparency = 0.34
	h.DepthMode = Enum.HighlightDepthMode.Occluded
	h.Parent = folder
	return h
end

local function scanDestructibles()
	local hrp = State.Root
	if not (hrp and hrp.Parent and State.Character) then return end
	ensureFXFolder()
	local now = os.clock()
	if now < Destruction.scanAt then return end
	Destruction.scanAt = now + CONFIG.BreakScanInterval
	local params = OverlapParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = { State.Character, Destruction.FX }
	params.MaxParts = 160
	local ok, parts = pcall(function() return Workspace:GetPartBoundsInRadius(hrp.Position, CONFIG.BreakScanRadius, params) end)
	if not ok or type(parts) ~= "table" then return end
	local added = 0
	for _, part in ipairs(parts) do
		if added >= CONFIG.BreakMaxHighlights then break end
		if not Destruction.Entries[part] and classifyDestructible(part) then
			local count = 0
			for _ in pairs(Destruction.Entries) do count += 1 end
			if count < CONFIG.BreakMaxHighlights then
				local originalLTM = part.LocalTransparencyModifier
				Destruction.Entries[part] = { highlight = makeHighlight(part), damage = 0, cracks = {}, originalLTM = originalLTM, lastHit = 0 }
				added += 1
			end
		end
	end
	for part, entry in pairs(Destruction.Entries) do
		if not part.Parent or (part.Position - hrp.Position).Magnitude > CONFIG.BreakScanRadius + 12 then
			if part.Parent and entry.originalLTM ~= nil then pcall(function() part.LocalTransparencyModifier = entry.originalLTM end) end
			destroyIfAlive(entry.highlight)
			for _, crack in ipairs(entry.cracks) do destroyIfAlive(crack) end
			Destruction.Entries[part] = nil
		end
	end
end

local function spawnFragments(part, position, normal, amount)
	local folder = ensureFXFolder()
	for i = 1, amount do
		local shard = Instance.new("Part")
		shard.Name = "LocalDebrisShard"
		shard.Size = Vector3.new(0.13 + math.random() * 0.22, 0.13 + math.random() * 0.2, 0.13 + math.random() * 0.22)
		shard.Color = part.Color
		shard.Material = part.Material
		shard.Anchored = true
		shard.CanCollide = false; shard.CanTouch = false; shard.CanQuery = false
		shard.CastShadow = false
		shard.CFrame = CFrame.new(position + Vector3.new((math.random()-0.5)*0.6, math.random()*0.4, (math.random()-0.5)*0.6))
		shard.Parent = folder
		local n = Vector3.new(math.random()-0.5, 0.3 + math.random(), math.random()-0.5)
		if normal then n += normal * 0.35 end
		local goal = { CFrame = shard.CFrame + n.Unit * (0.8 + math.random()*1.3), Transparency = 1 }
		local tween = TweenService:Create(shard, TweenInfo.new(0.48 + math.random()*0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), goal)
		tween:Play()
		task.delay(0.8, function() destroyIfAlive(shard) end)
	end
end

local function createCrackMarks(part, hitPosition, normal, entry)
	if #entry.cracks >= 10 then return end
	local folder = ensureFXFolder()
	local face = CFrame.lookAt(hitPosition + normal * 0.025, hitPosition + normal)
	local available = math.min(3, 10 - #entry.cracks)
	for i = 1, available do
		local length = 0.22 + math.random() * 0.48
		local mark = Instance.new("Part")
		mark.Name = "LocalCrackMark"
		mark.Size = Vector3.new(length, 0.025, 0.025)
		mark.Color = Color3.fromRGB(48, 49, 52)
		mark.Material = Enum.Material.SmoothPlastic
		mark.Anchored = true
		mark.CanCollide = false; mark.CanTouch = false; mark.CanQuery = false
		mark.CastShadow = false
		local ox = (math.random()-0.5) * 0.52
		local oy = (math.random()-0.5) * 0.52
		mark.CFrame = face * CFrame.new(ox, oy, 0) * CFrame.Angles(0, 0, math.rad(math.random(-65,65)))
		mark.Parent = folder
		table.insert(entry.cracks, mark)
	end
end

local function restoreDamageLater(part, entry)
	local stamp = os.clock()
	entry.restoreStamp = stamp
	task.delay(CONFIG.BreakRestoreSeconds, function()
		if Destruction.Entries[part] ~= entry or entry.restoreStamp ~= stamp then return end
		if part.Parent and entry.originalLTM ~= nil then pcall(function() part.LocalTransparencyModifier = entry.originalLTM end) end
		for _, crack in ipairs(entry.cracks) do destroyIfAlive(crack) end
		entry.cracks = {}
		entry.damage = 0
		entry.lastHit = 0
		if entry.highlight and entry.highlight.Parent then
			entry.highlight.OutlineColor = Color3.fromRGB(142,145,151)
			entry.highlight.OutlineTransparency = 0.34
		end
	end)
end

local function breakTarget()
	local cam = Workspace.CurrentCamera
	if not cam then return end
	local params = getRayParams()
	local hit = Workspace:Raycast(cam.CFrame.Position, cam.CFrame.LookVector * CONFIG.BreakHitDistance, params)
	if not hit or not hit.Instance or not hit.Instance:IsA("BasePart") then
		UI.setAnim("Destruição: mire em um objeto contornado", C_WARN)
		return
	end
	local part = hit.Instance
	local entry = Destruction.Entries[part]
	if not entry and classifyDestructible(part) then
		entry = { highlight = makeHighlight(part), damage = 0, cracks = {}, originalLTM = part.LocalTransparencyModifier, lastHit = 0 }
		Destruction.Entries[part] = entry
	end
	if not entry or not classifyDestructible(part) then
		UI.setAnim("Objeto protegido/incerto — não danificado", C_WARN)
		return
	end
	local now = os.clock()
	if now - (entry.lastHit or 0) < 0.22 then return end
	entry.lastHit = now
	entry.damage += 1
	createCrackMarks(part, hit.Position, hit.Normal, entry)
	spawnFragments(part, hit.Position, hit.Normal, 4 + math.min(entry.damage, 3))
	local material = part.Material
	local threshold = (material == Enum.Material.Glass and 1) or ((material == Enum.Material.Wood or material == Enum.Material.WoodPlanks) and 2) or 3
	if entry.damage >= threshold then
		-- LocalTransparencyModifier afeta só este cliente; a peça e as colisões originais permanecem.
		part.LocalTransparencyModifier = math.max(entry.originalLTM or 0, 0.62)
		if entry.highlight and entry.highlight.Parent then
			entry.highlight.OutlineColor = Color3.fromRGB(205, 205, 205)
			entry.highlight.OutlineTransparency = 0.62
		end
		UI.setAnim("Dano visual local aplicado", C_OK)
	else
		UI.setAnim("Objeto danificado: " .. tostring(entry.damage) .. "/" .. tostring(threshold), C_OK)
	end
	restoreDamageLater(part, entry)
end

local function startLocalSubsystem()
	Destruction.running = true
	task.spawn(function()
		while Destruction.running and script.Parent do
			if State.Character and State.Root and State.Enabled then
				local ok, err = pcall(scanDestructibles)
				if not ok then warn(TAG .. " scanner de objetos falhou: " .. tostring(err)) end
			end
			task.wait(0.25)
		end
	end)
end

----------------------------------------------------------------------
-- Loop de animação (um único ciclo por quadro)
----------------------------------------------------------------------
local function setAnimLabel(A, text)
	if A.label ~= text then
		A.label = text
		UI.setAnim(text, C_OK)
	end
end

local function updateRig(dt)
	local rig = State.Rig
	if not rig or not rig.Active then return end
	local hum, hrp = State.Humanoid, State.Root
	if not (hum and hum.Parent and hrp and hrp.Parent) then return end -- personagem saindo (respawn)
	if not rig.Model.Parent then error("O modelo da minifigura foi removido do personagem", 0) end

	dt = math.min(dt, 0.1)
	local A = rig.Anim
	local now = os.clock()
	updateParkourMotion(dt)
	A.t += dt

	local vel = hrp.AssemblyLinearVelocity
	local speed = math.sqrt(vel.X * vel.X + vel.Z * vel.Z)
	local vy = vel.Y
	local state = hum:GetState()
	local dead = state == ST.Dead or hum.Health <= 0
	local inAir = state == ST.Freefall or state == ST.Jumping
	local seated = state == ST.Seated
	local climbing = state == ST.Climbing
	local airLike = inAir or state == ST.Swimming or state == ST.FallingDown or state == ST.Ragdoll
		or state == ST.Physics or state == ST.Flying
	local grounded = not (airLike or seated or climbing or dead)

	-- Pouso
	if inAir then
		A.lastVy = vy
		A.airTime += dt
	else
		if A.wasAir and not dead then
			local impact = -A.lastVy
			if impact > 6 then
				A.landT = 1
				A.landS = clamp(impact / 45, 0.4, 1)
				if impact > 30 and Parkour.mode == "None" then A.parkour = { kind = "roll", t = 0, dur = 0.62 } end
			end
		end
		A.airTime = 0
	end
	A.wasAir = inAir
	A.landT = math.max(0, A.landT - dt / 0.32)
	if grounded or seated or climbing then A.doubleUsed = false end

	-- Taxa de giro (reação em mudança de direção)
	local look = hrp.CFrame.LookVector
	local yaw = math.atan2(-look.X, -look.Z)
	if A.yaw then
		local d = yaw - A.yaw
		if d > math.pi then d -= 2 * math.pi elseif d < -math.pi then d += 2 * math.pi end
		A.yawRate += (d / math.max(dt, 1 / 240) - A.yawRate) * (1 - exp(-14 * dt))
	end
	A.yaw = yaw

	-- Calibração dos pés: raio para baixo quando parado no chão (barato: 4x por segundo)
	if state == ST.Running and speed < 1.5 and now - A.lastCalib > 0.25 then
		A.lastCalib = now
		local res = Workspace:Raycast(hrp.Position, Vector3.new(0, -(rig.BaseG + 4), 0), rig.RayParams)
		if res and abs(res.Distance - rig.BaseG) <= 1.5 then
			rig.GroundOffset = lerp(rig.GroundOffset, res.Distance + CONFIG.FootOffsetTweak, 0.5)
		end
	end
	if abs(rig.GroundOffset - rig.AppliedG) > 0.002 then
		rig.AppliedG = rig.GroundOffset
		rig.BaseC0.Root = CFrame.new(0, -rig.AppliedG, 0)
		if rig.DriveMode ~= "C0" then rig.Joints.Root.C0 = rig.BaseC0.Root end
	end

	local T = A.target
	resetPose(T)
	local expr = "Happy"
	local label = "Parado"
	local fast = false
	local plantTarget = 0
	local act = A.action

	if dead then
		A.action = nil
		act = nil
		A.deadT += dt
		label = "Desativado"
		expr = "KO"
		fast = true
		poseDead(A, T)
		if A.deadT > 1.5 and not A.deadHidden then
			A.deadHidden = true
			for _, p in ipairs(rig.Parts) do p.Transparency = 1 end
		end
	else
		if A.deadT > 0 then -- reviveu
			A.deadT = 0
			if A.deadHidden then
				A.deadHidden = false
				for _, p in ipairs(rig.Parts) do p.Transparency = 0 end
				rig.RefreshFace()
			end
		end

		if act then
			act.t += dt
			if act.kind == "punch" and not act.fired and act.t >= (act.strikeAt or 0.12) then
				act.fired = true
				if Hooks.OnPunchStrike then
					local ok, e = pcall(Hooks.OnPunchStrike, act.side, hrp, act.variant)
					if not ok then warn(TAG .. " Hooks.OnPunchStrike falhou: " .. tostring(e)) end
				end
			end
			if act.t >= act.dur or (act.kind == "emote" and (airLike or speed > (act.name == "Comemorar" and 10 or 4))) then
				A.action = nil
				act = nil
			end
		end

		if A.flipT then
			A.flipT += dt / 0.5
			if A.flipT >= 1 then
				A.flipT = nil
				A.flipAngle = 0
			else
				A.flipAngle = -smooth(A.flipT) * 2 * math.pi
			end
		end

		-- Parada brusca (derrapada) e fidgets de idle
		A.peak = math.max(speed, A.peak - 25 * dt)
		if grounded and A.skidT <= 0 and A.peak > 12 and speed < A.peak * 0.45 then
			A.skidT = 1
			A.peak = speed
		end
		if not grounded then A.skidT = 0 end
		A.skidT = math.max(0, A.skidT - dt / 0.45)
		local idleNow = grounded and speed < 0.8 and not act and A.skidT <= 0 and A.landT <= 0
		if A.fidget then
			A.fidget.t += dt
			if not idleNow or A.fidget.t >= A.fidget.dur then
				A.fidget = nil
				A.nextFidget = now + 5 + math.random() * 5
			end
		elseif idleNow and now >= A.nextFidget then
			local kind = FIDGETS[math.random(#FIDGETS)]
			A.fidget = { kind = kind, t = 0, dur = (kind == "stretch" and 2.2) or (kind == "tap" and 1.6) or 1.8 }
		elseif not idleNow then
			A.nextFidget = math.max(A.nextFidget, now + 4)
		end

		if seated then
			poseSeated(T)
			label = "Sentado"
		elseif climbing then
			poseClimb(A, T, vy, dt)
			label = "Escalando"
		elseif airLike then
			poseAir(A, T, vy)
			label = vy > 1 and "Pulando" or "Caindo"
			expr = vy < -22 and "Surprised" or "Happy"
		else
			plantTarget = 1
			poseGround(A, T, speed, A.yawRate, dt)
			if speed > 12 then label = "Correndo"
			elseif speed > 0.8 then label = "Caminhando" end
			if A.landT > 0 then
				overlayLanding(T, A)
				label = "Pouso"
			end
			if A.skidT > 0 then
				overlaySkid(T, A)
				label = "Derrapando"
			end
			if A.fidget then
				overlayFidget(T, A.fidget)
				label = "Idle: " .. A.fidget.kind
			end
		end

		if A.flipT then overlayFlip(T, A.flipT) end
		if act then
			if act.kind == "punch" then
				if act.variant == "kick" then
					overlayKick(T, act)
					label = "Chute"
				else
					overlayPunch(T, act)
					label = "Soco"
				end
				if act.t > 0.06 and act.t < 0.42 then expr = "Shout" end
			else
				overlayEmoteKind(T, act)
				label = "Gesto: " .. act.name
				expr = act.name == "Acenar" and "Happy" or "Laugh"
			end
		end
		if act or A.flipT or A.landT > 0 or A.skidT > 0 then fast = true end
	end

	if A.parkour then
		A.parkour.t += dt
		if A.parkour.kind ~= "hang" and A.parkour.t >= (A.parkour.dur or 0.7) then
			A.parkour = nil
		else
			overlayParkour(T, A.parkour)
			label = ({slide="Deslizando", vault="Vault", hang="Agarrado na borda", mantle="Subindo borda", walljump="Wall-jump", roll="Rolamento"})[A.parkour.kind] or "Parkour"
			fast = true
		end
	end

	-- Suavização exponencial
	local alpha = 1 - exp(-(fast and 34 or 16) * dt)
	local C = A.cur
	for i = 1, #JOINT_NAMES do
		local jn = JOINT_NAMES[i]
		local c, t = C[jn], T[jn]
		c.rx += (t.rx - c.rx) * alpha
		c.ry += (t.ry - c.ry) * alpha
		c.rz += (t.rz - c.rz) * alpha
		c.tx += (t.tx - c.tx) * alpha
		c.ty += (t.ty - c.ty) * alpha
		c.tz += (t.tz - c.tz) * alpha
	end

	-- Pés plantados no chão: abaixa/levanta o quadril conforme o ângulo real das pernas
	local plant = (math.max(legDrop(C.HipL.rx, C.HipL.rz), legDrop(C.HipR.rx, C.HipR.rz)) - 10.5) * K
	A.plantW += (plantTarget - A.plantW) * (1 - exp(-12 * dt))

	local rc = C.Root
	rig.Drive("Root", CFrame.new(rc.tx, rc.ty + plant * A.plantW, rc.tz) * ROOT_PIVOT
		* CFrame.Angles(rc.rx + A.flipAngle, rc.ry, rc.rz) * ROOT_PIVOT_INV)
	for i = 2, #JOINT_NAMES do
		local jn = JOINT_NAMES[i]
		local c = C[jn]
		rig.Drive(jn, CFrame.new(c.tx, c.ty, c.tz) * CFrame.Angles(c.rx, c.ry, c.rz))
	end

	-- Rosto
	if not dead then
		if A.blinkUntil > 0 then
			if now >= A.blinkUntil then
				A.blinkUntil = 0
				rig.SetBlink(false)
				A.nextBlink = now + 2 + math.random() * 3
			end
		elseif now >= A.nextBlink and expr == "Happy" then
			A.blinkUntil = now + 0.12
			rig.SetBlink(true)
		end
	end
	if not A.deadHidden then rig.SetExpression(expr) end
	setAnimLabel(A, label .. " [" .. tostring(rig.DriveMode) .. "]")
end

----------------------------------------------------------------------
-- Ações do jogador
----------------------------------------------------------------------
local function startPunch()
	local rig = State.Rig
	if not (rig and rig.Active) then return end
	local A = rig.Anim
	local now = os.clock()
	if A.deadT > 0 or now - A.lastPunch < CONFIG.PunchCooldown then return end
	if now - A.lastPunch > 1.0 then A.combo = 0 end
	A.lastPunch = now
	A.combo += 1
	if A.combo >= 3 then -- 3º golpe do combo: chute
		A.combo = 0
		A.action = { kind = "punch", variant = "kick", t = 0, dur = 0.62, side = 1, strikeAt = 0.18, fired = false }
	else
		A.punchSide = -A.punchSide -- alterna direita/esquerda
		A.action = { kind = "punch", variant = "punch", t = 0, dur = 0.55, side = A.punchSide, strikeAt = 0.12, fired = false }
	end
end

local function startEmote()
	local rig = State.Rig
	if not (rig and rig.Active) then return end
	local A = rig.Anim
	if A.deadT > 0 then return end
	State.EmoteIdx = State.EmoteIdx % #EMOTES + 1
	local name = EMOTES[State.EmoteIdx]
	A.action = { kind = "emote", name = name, t = 0, dur = name == "Dançar" and 3.2 or 2.0 }
end

local function onJumpRequest()
	if not CONFIG.EnableDoubleJump then return end
	local rig = State.Rig
	local now = os.clock()
	if not (rig and rig.Active) then return end
	local A = rig.Anim
	local gap = now - A.prevJumpReq
	A.prevJumpReq = now
	if gap < 0.18 then return end -- exige um novo toque (ignora o botão mantido)
	if Parkour.mode == "Hang" then beginMantle(); return end
	if tryWallJump() then return end
	local hum, hrp = State.Humanoid, State.Root
	if not (hum and hrp and hum.Health > 0) then return end
	local jumpState = hum:GetState()
	if jumpState ~= ST.Freefall and jumpState ~= ST.Jumping then return end
	if A.doubleUsed or A.airTime < CONFIG.DoubleJumpMinAirTime then return end
	A.doubleUsed = true
	local jv
	if hum.UseJumpPower then jv = hum.JumpPower else jv = math.sqrt(2 * Workspace.Gravity * hum.JumpHeight) end
	local v = hrp.AssemblyLinearVelocity
	hrp.AssemblyLinearVelocity = Vector3.new(v.X, jv * 0.95, v.Z)
	A.flipT = 0
end

----------------------------------------------------------------------
-- Ativação / desativação
----------------------------------------------------------------------
local function refreshToggle()
	UI.setToggleText(State.Enabled and "Transformar: ON" or "Transformar: OFF")
end
local function refreshOutfit()
	UI.setOutfitText("Traje: " .. OUTFITS[State.OutfitIndex].Name)
end

local function deactivateRig(restore)
	local rig = State.Rig
	State.Rig = nil
	State.RigBin:Clean()
	if rig then rig.Destroy() end
	if restore then restoreOriginal() else State.Hidden = nil end
	State.ToolHeld = false
	UI.setActionsVisible(false)
end

local function sweepStrayRigs(character)
	for _, c in ipairs(character:GetChildren()) do
		if c.Name == "LegoMinifigRig" and not (State.Rig and State.Rig.Model == c) then
			c:Destroy()
		end
	end
end

local function failBuild(step, msg, rig, replacing, character)
	warn(TAG .. " Falha ao construir [" .. step .. "]: " .. tostring(msg))
	local full = (not replacing) or (rig ~= nil and State.Rig == rig)
	if rig then rig.Destroy() end
	if full then deactivateRig(true) end
	if character then sweepStrayRigs(character) end
	UI.setBuild("Falha ao construir: " .. tostring(msg), C_ERR)
	UI.setError(step, msg)
	UI.setAnim("—", C_NEUTRAL)
end

local function waitForParts(character, token)
	local deadline = os.clock() + CONFIG.WaitTimeout
	local function remaining() return math.max(0.1, deadline - os.clock()) end
	local hum = character:WaitForChild("Humanoid", remaining())
	if not hum then error("Humanoid não apareceu em " .. CONFIG.WaitTimeout .. "s", 0) end
	local hrp = character:WaitForChild("HumanoidRootPart", remaining())
	if not hrp then error("HumanoidRootPart não apareceu em " .. CONFIG.WaitTimeout .. "s", 0) end
	if hum.RigType ~= Enum.HumanoidRigType.R15 then
		error("Este sistema exige R15 (RigType atual: " .. tostring(hum.RigType) .. ")", 0)
	end
	for _, name in ipairs({ "LowerTorso", "UpperTorso", "Head" }) do
		if not character:WaitForChild(name, remaining()) then error("Parte R15 ausente: " .. name, 0) end
	end
	while not player:HasAppearanceLoaded() and os.clock() < deadline do
		if token ~= State.Token then return nil end
		task.wait(0.1)
	end
	if not player:HasAppearanceLoaded() then
		warn(TAG .. " aparência não terminou de carregar em " .. CONFIG.WaitTimeout .. "s; continuando mesmo assim")
	end
	return { hum = hum, hrp = hrp }
end

local function installRig(rig, character)
	local old = State.Rig
	if not old then
		hideOriginal(character)
		State.ToolHeld = character:FindFirstChildOfClass("Tool") ~= nil
		State.RigBin:Add(character.ChildAdded:Connect(function(c)
			if c:IsA("Tool") then State.ToolHeld = true end
		end))
		State.RigBin:Add(character.ChildRemoved:Connect(function(c)
			if c:IsA("Tool") then State.ToolHeld = character:FindFirstChildOfClass("Tool") ~= nil end
		end))
		local errHandler = function(e)
			State.LastTrace = debug.traceback(tostring(e), 2)
			return tostring(e)
		end
		State.RigBin:Add(RunService.RenderStepped:Connect(function(dt)
			local ok, err = xpcall(updateRig, errHandler, dt)
			if not ok then State.OnAnimError(err) end
		end))
	end
	rig.Active = true
	State.Rig = rig
	if old and old ~= rig then old.Destroy() end
end

-- Move a perna esquerda 40° e confere se a peça realmente se moveu. Tenta C0 e depois Transform.
local function testJointDrive(rig)
	local leg = rig.Joints.HipL.Part1
	local restRel = rig.Rest[rig.Hips]:Inverse() * rig.Rest[leg]
	local test = CFrame.Angles(rad(40), 0, 0)
	for _, mode in ipairs({ "C0", "Transform" }) do
		rig.DriveMode = mode
		rig.Drive("HipL", test)
		RunService.Heartbeat:Wait()
		RunService.Heartbeat:Wait()
		local rel = rig.Hips.CFrame:ToObjectSpace(leg.CFrame)
		local moved = rel.UpVector:Dot(restRel.UpVector) < 0.95
		rig.Drive("HipL", CFrame.new())
		if moved then
			RunService.Heartbeat:Wait()
			return
		end
	end
	rig.DriveMode = nil
	error("As juntas não responderam à animação (nem por C0 nem por Transform)", 0)
end

local function finalCheck(rig, character, hum)
	if not rig.Model.Parent then error("Modelo da minifigura sumiu após instalar", 0) end
	if State.Rig ~= rig or not rig.Active then error("Rig não ficou registrado como ativo", 0) end
	if hum.Parent ~= character or hum.Health <= 0 then error("Humanoid inválido após instalar", 0) end
	local count = 0
	for _ in pairs(State.Hidden or {}) do count += 1 end
	if count == 0 then error("Nenhuma peça original foi ocultada", 0) end
end

local function runPipeline(token, replacing)
	local character = player.Character
	if not character then
		UI.setInit("Aguardando personagem", C_WARN)
		return
	end
	UI.setError(nil)

	if not replacing then
		deactivateRig(true)
		sweepStrayRigs(character)
		UI.setInit("Aguardando personagem", C_WARN)
		UI.setBuild("—", C_NEUTRAL)
		UI.setAnim("—", C_NEUTRAL)
		local ok, res = safeCall("Aguardando personagem", waitForParts, character, token)
		if token ~= State.Token then return end
		if not ok then return failBuild("Aguardando personagem", res, nil, false, character) end
		if not res then return end
		State.Character, State.Humanoid, State.Root = character, res.hum, res.hrp
		UI.setInit("Personagem detectado", C_OK)
	end

	local hum, hrp = State.Humanoid, State.Root
	if State.Character ~= character or not hum or not hrp then
		return failBuild("Aguardando personagem", "personagem inválido", nil, replacing, character)
	end

	UI.setBuild("Criando peças", C_WARN)
	local ok, rig = safeCall("Criando peças", buildRig, character, hum, hrp, OUTFITS[State.OutfitIndex])
	if not ok then return failBuild("Criando peças", rig, nil, replacing, character) end

	UI.setBuild("Validando juntas", C_WARN)
	local ok2, err2 = safeCall("Validando juntas", validateRigStatic, rig, character, hrp)
	if not ok2 then return failBuild("Validando juntas", err2, rig, replacing, character) end

	RunService.Heartbeat:Wait()
	RunService.Heartbeat:Wait()
	if token ~= State.Token then rig.Destroy() return end

	UI.setBuild("Aplicando transformação", C_WARN)
	local ok3, err3 = safeCall("Aplicando transformação", verifyRigPhysics, rig, hrp, hum)
	if not ok3 then return failBuild("Aplicando transformação", err3, rig, replacing, character) end

	UI.setBuild("Testando animação das juntas", C_WARN)
	local okT, errT = safeCall("Testando animação das juntas", testJointDrive, rig)
	if not okT then return failBuild("Testando animação das juntas", errT, rig, replacing, character) end
	if token ~= State.Token then rig.Destroy() return end

	local ok4, err4 = safeCall("Instalando rig", installRig, rig, character)
	if not ok4 then return failBuild("Instalando rig", err4, rig, replacing, character) end

	local ok5, err5 = safeCall("Confirmação final", finalCheck, rig, character, hum)
	if not ok5 then return failBuild("Confirmação final", err5, rig, replacing, character) end

	UI.setInit("Personagem OK", C_OK)
	UI.setBuild("Minifigura LEGO ativa (" .. OUTFITS[State.OutfitIndex].Name .. ", " .. #rig.Parts .. " peças)", C_OK)
	UI.setActionsVisible(true)
end

State.OnAnimError = function(msg)
	task.defer(function()
		resetParkour(true)
		clearDestruction()
		warn(TAG .. " Falha na animação: " .. tostring(msg))
		if State.LastTrace then warn(State.LastTrace) end
		State.Enabled = false
		State.Token += 1
		deactivateRig(true)
		refreshToggle()
		UI.setBuild("Desativada por erro de animação (avatar restaurado)", C_ERR)
		UI.setError("Animação", msg)
		UI.setAnim("Falha: " .. tostring(msg), C_ERR)
	end)
end

----------------------------------------------------------------------
-- Ligações de eventos
----------------------------------------------------------------------
local function onCharacterAdded(char)
	resetParkour(true)
	clearDestruction()
	State.Token += 1
	local token = State.Token
	deactivateRig(false)
	State.Character, State.Humanoid, State.Root = nil, nil, nil
	if not State.Enabled then
		UI.setBuild("Desativada (avatar original)", C_NEUTRAL)
		return
	end
	task.spawn(runPipeline, token, false)
end

GlobalBin:Add(player.CharacterAdded:Connect(onCharacterAdded))
GlobalBin:Add(player.CharacterRemoving:Connect(function()
	resetParkour(true)
	clearDestruction()
	State.Token += 1
	deactivateRig(false)
	UI.setInit("Aguardando personagem", C_WARN)
end))

GlobalBin:Add(UI.Toggle.Activated:Connect(function()
	State.Enabled = not State.Enabled
	State.Token += 1
	local token = State.Token
	refreshToggle()
	UI.setError(nil)
	if State.Enabled then
		task.spawn(runPipeline, token, false)
	else
		resetParkour(true)
		clearDestruction()
		deactivateRig(true)
		UI.setBuild("Desativada — avatar original restaurado", C_NEUTRAL)
		UI.setAnim("—", C_NEUTRAL)
	end
end))

GlobalBin:Add(UI.Outfit.Activated:Connect(function()
	State.OutfitIndex = State.OutfitIndex % #OUTFITS + 1
	refreshOutfit()
	if State.Enabled and State.Rig then
		State.Token += 1
		task.spawn(runPipeline, State.Token, true)
	end
end))

GlobalBin:Add(UI.Punch.Activated:Connect(startPunch))
GlobalBin:Add(UI.Gesture.Activated:Connect(startEmote))
GlobalBin:Add(UI.Slide.Activated:Connect(startSlide))
GlobalBin:Add(UI.Parkour.Activated:Connect(performParkour))
GlobalBin:Add(UI.Break.Activated:Connect(breakTarget))
GlobalBin:Add(UserInputService.JumpRequest:Connect(onJumpRequest))
GlobalBin:Add(UserInputService.InputBegan:Connect(function(input, processed) -- atalhos para testar no PC
	if processed then return end
	if input.KeyCode == Enum.KeyCode.F then startPunch()
	elseif input.KeyCode == Enum.KeyCode.G then startEmote() end
end))

script.Destroying:Connect(function()
	Destruction.running = false
	resetParkour(true)
	clearDestruction()
	State.Token += 1
	deactivateRig(true)
	GlobalBin:Clean()
	if UI.Gui then UI.Gui:Destroy() end
end)

-- Início
startLocalSubsystem()
refreshToggle()
refreshOutfit()
UI.setInit("Aguardando personagem", C_WARN)
if player.Character then
	task.spawn(function() onCharacterAdded(player.Character) end)
end
if not State.Enabled then
	UI.setBuild("Desativada (avatar original)", C_NEUTRAL)
end
