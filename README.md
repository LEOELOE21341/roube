--=====================================================================
-- ZERO // steal a brainrot
-- language: Lua (Luau) | target: Roblox client | env: Xeno / Velocity
-- modules: auto steal | esp | auto buy | auto collect | anti-afk
-- technique: teleport into prompt range + fireproximityprompt
-- no remote names hardcoded — descobre os prompts em runtime
--=====================================================================

if type(getgenv) == "function" and type(getgenv().ZERO_STEAL_CLEANUP) == "function" then
	pcall(getgenv().ZERO_STEAL_CLEANUP)
end

local Players    = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS        = game:GetService("UserInputService")
local Tween      = game:GetService("TweenService")
local Workspace  = game:GetService("Workspace")

local LP = Players.LocalPlayer

local CONN, DRAW = {}, {}
local function track(c) CONN[#CONN + 1] = c; return c end
local function bind(sig, fn) return track(sig:Connect(fn)) end

local fire_prompt
pcall(function() fire_prompt = fireproximityprompt end)
pcall(function() if not fire_prompt then fire_prompt = fire_proximity_prompt end end)

local fire_touch
pcall(function() fire_touch = firetouchinterest end)

local COL = {
	bg      = Color3.fromRGB(15, 15, 19),
	panel   = Color3.fromRGB(21, 21, 27),
	panel2  = Color3.fromRGB(29, 29, 38),
	stroke  = Color3.fromRGB(44, 44, 58),
	text    = Color3.fromRGB(236, 236, 245),
	dim     = Color3.fromRGB(138, 138, 160),
	accent  = Color3.fromRGB(138, 92, 255),
	accent2 = Color3.fromRGB(80, 205, 255),
	off     = Color3.fromRGB(48, 48, 62),
	danger  = Color3.fromRGB(255, 84, 104),
	gold    = Color3.fromRGB(255, 190, 60),
}

local RARITY = {
	{ "common",    Color3.fromRGB(185, 185, 195) },
	{ "uncommon",  Color3.fromRGB(95, 205, 125) },
	{ "rare",      Color3.fromRGB(85, 165, 255) },
	{ "epic",      Color3.fromRGB(175, 95, 255) },
	{ "legendary", Color3.fromRGB(255, 190, 60) },
	{ "mythic",    Color3.fromRGB(255, 95, 95) },
	{ "secret",    Color3.fromRGB(255, 65, 205) },
	{ "godly",     Color3.fromRGB(125, 255, 225) },
}

local DrawingLib
pcall(function() DrawingLib = Drawing end)

local function new(class, props, parent)
	local inst = Instance.new(class)
	for k, v in next, props do inst[k] = v end
	if parent then inst.Parent = parent end
	return inst
end
local function corner(inst, r) return new("UICorner", { CornerRadius = UDim.new(0, r or 8) }, inst) end
local function outline(inst, c, t, tr)
	return new("UIStroke", { Color = c or COL.stroke, Thickness = t or 1, Transparency = tr or 0.35 }, inst)
end

local function viewport()
	local cam = Workspace.CurrentCamera
	return (cam and cam.ViewportSize) or Vector2.new(1280, 720)
end
local function compute_scale()
	local vp = viewport()
	return math.clamp(math.min(vp.X / 600, vp.Y / 500), 0.5, 1)
end

local function fmt(n)
	n = math.floor(tonumber(n) or 0)
	if n >= 1e9 then return string.format("%.1fB", n / 1e9) end
	if n >= 1e6 then return string.format("%.1fM", n / 1e6) end
	if n >= 1e3 then return string.format("%.1fK", n / 1e3) end
	return tostring(n)
end

--========================================================================
-- STATE
--========================================================================
local cfg = {
	auto     = false,
	delay    = 0.30,
	tp       = true,
	legit    = true,
	own      = true,
	minval   = 0,
	keyword  = "steal",
	maxdist  = 2000,
	key      = Enum.KeyCode.F,

	esp      = false,
	esp_text = true,
	esp_hl   = true,

	buy      = false,
	buy_max  = 500000,
	collect  = false,

	antiafk  = false,
}

local status_txt = "idle"
local last_action = "-"
local running = true
local cooldown = {}
local CACHE = {}

--========================================================================
-- DISCOVERY
--========================================================================
local function find_part(obj)
	local cur = obj
	for _ = 1, 6 do
		if not cur then return nil end
		if cur:IsA("BasePart") then return cur end
		cur = cur.Parent
	end
	return nil
end

local function rarity_color(r)
	r = string.lower(tostring(r or ""))
	for _, pair in ipairs(RARITY) do
		if r:find(pair[1], 1, true) then return pair[2] end
	end
	return COL.accent
end

local function entry_of(prompt)
	if not prompt or not prompt.Parent then return nil end
	local part = find_part(prompt.Parent)
	if not part then return nil end
	local model = part:FindFirstAncestorOfClass("Model") or part

	local value = 0
	for _, key in ipairs({ "Value", "Price", "Worth", "Cost", "Cash", "Income" }) do
		local v = model:GetAttribute(key) or part:GetAttribute(key)
		if type(v) == "number" and v > 0 then value = v; break end
	end
	if value == 0 then
		local digits = model.Name:match("(%d[%d,%.]*)")
		if digits then
			value = tonumber((digits:gsub("[,%.]", ""))) or 0
		end
	end

	local rarity = model:GetAttribute("Rarity")
		or model:GetAttribute("Tier")
		or model:GetAttribute("RarityName")
		or ""

	local text = string.lower(
		tostring(prompt.ActionText or "") .. " " ..
		tostring(prompt.ObjectText or "") .. " " ..
		model.Name
	)

	return {
		prompt = prompt,
		part   = part,
		model  = model,
		text   = text,
		name   = model.Name,
		value  = value,
		rarity = tostring(rarity),
	}
end

local function scan()
	local out = {}
	local ok = pcall(function()
		for _, d in ipairs(Workspace:GetDescendants()) do
			if d:IsA("ProximityPrompt") then
				local e = entry_of(d)
				if e then out[#out + 1] = e end
			end
		end
	end)
	if not ok then return CACHE end
	CACHE = out
	return out
end

local function matches(e)
	local kw = string.lower(cfg.keyword)
	if kw == "" then return true end
	return e.text:find(kw, 1, true) ~= nil
end

local function is_own(obj)
	local names = { LP.Name, LP.DisplayName }
	local cur = obj
	for _ = 1, 8 do
		if not cur or cur == Workspace then return false end
		for _, n in ipairs(names) do
			if n and n ~= "" and cur.Name:find(n, 1, true) then return true end
		end
		cur = cur.Parent
	end
	return false
end

local function hrp()
	local char = LP.Character
	return char and char:FindFirstChild("HumanoidRootPart")
end

local function dist_of(e)
	local root = hrp()
	if not root or not e.part then return math.huge end
	return (root.Position - e.part.Position).Magnitude
end

--========================================================================
-- STEAL
--========================================================================
local function approach(part)
	local root = hrp()
	if not root or not part then return false end
	local flat = root.Position - part.Position
	flat = Vector3.new(flat.X, 0, flat.Z)
	if flat.Magnitude < 0.1 then flat = Vector3.new(3, 0, 0) end
	local pos = part.Position + flat.Unit * 3 + Vector3.new(0, 3, 0)
	pcall(function() root.AssemblyLinearVelocity = Vector3.zero end)
	root.CFrame = CFrame.new(pos)
	return true
end

local function steal(e, manual)
	if not e or not e.part or not e.prompt then return false end
	if not manual and cfg.own and is_own(e.model) then return false end
	if not manual and cfg.minval > 0 and e.value < cfg.minval then return false end
	if not fire_prompt then
		status_txt = "sem fireproximityprompt"
		return false
	end

	local stamp = os.clock()
	if not manual and cooldown[e.prompt] and stamp - cooldown[e.prompt] < 1.5 then
		return false
	end
	cooldown[e.prompt] = stamp

	if cfg.tp then approach(e.part) end
	if cfg.legit then task.wait(0.05 + math.random() * 0.10) end

	pcall(function() e.prompt.Enabled = true end)
	local ok, err = pcall(function() fire_prompt(e.prompt) end)

	if ok then
		last_action = "steal · " .. e.name
		status_txt = "roubando " .. e.name
	else
		last_action = "erro · " .. tostring(err)
		status_txt = "erro no prompt"
	end

	task.wait(math.max(cfg.delay, 0.05))
	return ok
end

local function best_entry()
	local best, best_score = nil, nil
	for _, e in ipairs(CACHE) do
		if matches(e) and (e.value >= cfg.minval) then
			if not (cfg.own and is_own(e.model)) then
				local d = dist_of(e)
				if d <= cfg.maxdist then
					local score = (e.value > 0) and e.value or (1e9 - d)
					if not best_score or score > best_score then
						best, best_score = e, score
					end
				end
			end
		end
	end
	return best
end

--========================================================================
-- AUTO BUY / COLLECT
--========================================================================
local function buy_entry()
	local best, best_score = nil, nil
	local kw = "buy"
	for _, e in ipairs(CACHE) do
		if e.text:find(kw, 1, true) then
			local price = e.value
			if price <= cfg.buy_max then
				local d = dist_of(e)
				local score = (price > 0) and (1e9 - price) or (1e9 - d)
				if not best_score or score > best_score then
					best, best_score = e, score
				end
			end
		end
	end
	return best
end

local function collect_cash()
	if not fire_touch then return end
	local root = hrp()
	if not root then return end
	local ok = pcall(function()
		for _, d in ipairs(Workspace:GetDescendants()) do
			if d:IsA("BasePart") and not d.Anchored then
				local n = string.lower(d.Name)
				if n:find("cash", 1, true) or n:find("money", 1, true) or n:find("coin", 1, true) then
					if (root.Position - d.Position).Magnitude < 80 then
						pcall(function() fire_touch(root, d, 0) end)
						task.wait(0.01)
						pcall(function() fire_touch(root, d, 1) end)
					end
				end
			end
		end
	end)
end

--========================================================================
-- ESP
--========================================================================
local esp_map = {}

local function esp_drop(model)
	local e = esp_map[model]
	if not e then return end
	if e.hl then pcall(function() e.hl:Destroy() end) end
	if e.txt then pcall(function() e.txt:Remove() end) end
	esp_map[model] = nil
end

local function esp_prune()
	local dead = {}
	for model in pairs(esp_map) do
		if not model.Parent then dead[#dead + 1] = model end
	end
	for _, m in ipairs(dead) do esp_drop(m) end
end

local function esp_tick()
	local cam = Workspace.CurrentCamera
	if not cam then return end
	esp_prune()

	if not cfg.esp then
		for model, e in pairs(esp_map) do
			if e.hl then e.hl.Enabled = false end
			if e.txt then e.txt.Visible = false end
			model = model
		end
		return
	end

	local seen = {}
	local count = 0

	for _, en in ipairs(CACHE) do
		local model = en.model
		if model and model.Parent and not seen[model] and count < 90 then
			seen[model] = true
			count = count + 1

			local d = dist_of(en)
			if d <= cfg.maxdist then
				local e = esp_map[model]
				if not e then
					e = { hl = nil, txt = nil }
					esp_map[model] = e
				end

				local col = rarity_color(en.rarity)

				if cfg.esp_hl then
					if not e.hl or e.hl.Parent ~= model then
						if e.hl then pcall(function() e.hl:Destroy() end) end
						e.hl = new("Highlight", {
							Adornee = model,
							FillColor = col,
							FillTransparency = 0.72,
							OutlineColor = col,
							OutlineTransparency = 0.15,
							DepthMode = Enum.HighlightDepthMode.AlwaysOnTop,
						}, model)
					end
					e.hl.Enabled = true
					e.hl.FillColor = col
					e.hl.OutlineColor = col
				elseif e.hl then
					e.hl.Enabled = false
				end

				if cfg.esp_text and DrawingLib then
					if not e.txt then
						local ok, t = pcall(function() return DrawingLib.new("Text") end)
						if ok and t then
							t.Center = true
							t.Outline = true
							t.OutlineColor = Color3.new(0, 0, 0)
							t.Size = 12
							t.Font = 2
							e.txt = t
							DRAW[#DRAW + 1] = t
						end
					end
					if e.txt then
						local okc, cf, size = pcall(function()
							local c, s = model:GetBoundingBox()
							return c, s
						end)
						local pos = en.part.Position
						local sp, on = cam:WorldToViewportPoint(pos + Vector3.new(0, 2.5, 0))
						if on and sp.Z > 0 then
							e.txt.Position = Vector2.new(sp.X, sp.Y)
							e.txt.Text = string.format("%s  %s  [%d]",
								en.name, fmt(en.value), math.floor(d))
							e.txt.Color = col
							e.txt.Visible = true
						else
							e.txt.Visible = false
						end
					end
				elseif e.txt then
					e.txt.Visible = false
				end
			else
				local e = esp_map[model]
				if e then
					if e.hl then e.hl.Enabled = false end
					if e.txt then e.txt.Visible = false end
				end
			end
		end
	end

	for model, e in pairs(esp_map) do
		if not seen[model] then
			if e.hl then e.hl.Enabled = false end
			if e.txt then e.txt.Visible = false end
		end
	end
end

--========================================================================
-- UI ROOT
--========================================================================
local PARENT
do
	if type(gethui) == "function" then
		local ok, h = pcall(gethui)
		if ok and h then PARENT = h end
	end
	if not PARENT then
		local ok, pg = pcall(function() return LP:FindFirstChildOfClass("PlayerGui") end)
		PARENT = (ok and pg) or LP:WaitForChild("PlayerGui")
	end
end

local gui = new("ScreenGui", {
	Name = "s" .. tostring(math.random(1000000, 9999999)),
	ResetOnSpawn = false,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
	DisplayOrder = 9999,
}, PARENT)

pcall(function() if syn and syn.protect_gui then syn.protect_gui(gui) end end)
pcall(function() if protect_gui then protect_gui(gui) end end)
pcall(function() if protectgui then protectgui(gui) end end)

local ORD = setmetatable({}, { __mode = "k" })
local function next_order(parent)
	ORD[parent] = (ORD[parent] or 0) + 1
	return ORD[parent]
end

local binding_active = false

local function draggable(frame, handle, scale_get)
	local drag, start, start_pos, start_abs = false, nil, nil, nil
	handle.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch then
			drag = true
			start = Vector2.new(input.Position.X, input.Position.Y)
			start_pos = frame.Position
			start_abs = frame.AbsolutePosition
		end
	end)
	track(UIS.InputChanged:Connect(function(input)
		if not drag then return end
		if input.UserInputType ~= Enum.UserInputType.MouseMovement
		and input.UserInputType ~= Enum.UserInputType.Touch then return end
		local s = scale_get() or 1
		local vp = viewport()
		local size = frame.AbsoluteSize
		local raw = Vector2.new(input.Position.X, input.Position.Y) - start
		local target = start_abs + raw
		local x = math.clamp(target.X, -(size.X - 60), vp.X - 60)
		local y = math.clamp(target.Y, 36, vp.Y - 46)
		local d = (Vector2.new(x, y) - start_abs) / s
		frame.Position = UDim2.new(
			start_pos.X.Scale, start_pos.X.Offset + d.X,
			start_pos.Y.Scale, start_pos.Y.Offset + d.Y
		)
	end))
	track(UIS.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch then
			drag = false
			start, start_abs = nil, nil
		end
	end))
end

local function hdr_button(header, text, x, cb)
	local b = new("TextButton", {
		Size = UDim2.fromOffset(26, 26),
		Position = UDim2.new(1, x, 0.5, -13),
		BackgroundColor3 = COL.panel2,
		Text = text, Font = Enum.Font.GothamBold,
		TextSize = 15, TextColor3 = COL.dim, AutoButtonColor = false,
	}, header)
	corner(b, 7)
	b.MouseButton1Click:Connect(cb)
	return b
end

--========================================================================
-- WINDOW
--========================================================================
local win = new("Frame", {
	Size = UDim2.fromOffset(540, 470),
	Position = UDim2.new(0.5, 0, 0.5, 0),
	AnchorPoint = Vector2.new(0.5, 0.5),
	BackgroundColor3 = COL.bg,
	BorderSizePixel = 0,
	Active = true,
}, gui)
corner(win, 14)
outline(win, COL.stroke, 1, 0.2)
local win_scale = new("UIScale", { Scale = compute_scale() }, win)
draggable(win, win, function() return win_scale.Scale end)

local header = new("Frame", { Size = UDim2.new(1, 0, 0, 46), BackgroundTransparency = 1 }, win)
local dot = new("Frame", {
	Size = UDim2.fromOffset(7, 7),
	Position = UDim2.new(0, 16, 0.5, -3),
	BackgroundColor3 = COL.accent,
}, header)
corner(dot, 4)
new("TextLabel", {
	Size = UDim2.new(0, 160, 1, 0), Position = UDim2.new(0, 30, 0, 0),
	BackgroundTransparency = 1, Text = "ZERO", Font = Enum.Font.GothamBold,
	TextSize = 16, TextColor3 = COL.text, TextXAlignment = Enum.TextXAlignment.Left,
}, header)
new("TextLabel", {
	Size = UDim2.new(0, 140, 1, 0), Position = UDim2.new(0, 80, 0, 1),
	BackgroundTransparency = 1, Text = "steal", Font = Enum.Font.GothamMedium,
	TextSize = 10, TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Left,
}, header)

local unload_dummy

hdr_button(header, "×", -36, function()
	if unload_dummy then unload_dummy() end
end)

local side = new("Frame", {
	Size = UDim2.new(0, 110, 1, -64),
	Position = UDim2.new(0, 14, 0, 52),
	BackgroundTransparency = 1,
}, win)
new("UIListLayout", { Padding = UDim.new(0, 5), SortOrder = Enum.SortOrder.LayoutOrder }, side)

local body = new("Frame", {
	Size = UDim2.new(1, -144, 1, -64),
	Position = UDim2.new(0, 132, 0, 52),
	BackgroundTransparency = 1,
}, win)

local PAGES, TABS = {}, {}

local function make_page(order)
	local p = new("Frame", {
		Size = UDim2.fromScale(1, 1),
		BackgroundTransparency = 1,
		Visible = false,
		LayoutOrder = order,
	}, body)
	PAGES[#PAGES + 1] = p
	return p
end

local function scroll_area(parent)
	local s = new("ScrollingFrame", {
		Size = UDim2.fromScale(1, 1),
		BackgroundTransparency = 1,
		BorderSizePixel = 0,
		ScrollBarThickness = 3,
		ScrollBarImageColor3 = COL.stroke,
		CanvasSize = UDim2.new(0, 0, 0, 0),
		AutomaticCanvasSize = Enum.AutomaticSize.Y,
		ScrollingDirection = Enum.ScrollingDirection.Y,
	}, parent)
	new("UIListLayout", { Padding = UDim.new(0, 4), SortOrder = Enum.SortOrder.LayoutOrder }, s)
	new("UIPadding", { PaddingRight = UDim.new(0, 6), PaddingBottom = UDim.new(0, 8) }, s)
	return s
end

local function select_page(page)
	for _, p in ipairs(PAGES) do p.Visible = (p == page) end
	for _, t in ipairs(TABS) do
		local on = (t.page == page)
		Tween:Create(t.btn, TweenInfo.new(0.14), {
			BackgroundColor3 = on and COL.panel2 or COL.panel,
		}):Play()
		t.lbl.TextColor3 = on and COL.text or COL.dim
		t.ind.Visible = on
	end
end

local function make_tab(label, order, page)
	local b = new("TextButton", {
		Size = UDim2.new(1, 0, 0, 36),
		BackgroundColor3 = COL.panel,
		Text = "", AutoButtonColor = false,
		LayoutOrder = order,
	}, side)
	corner(b, 8)
	local ind = new("Frame", {
		Size = UDim2.new(0, 3, 0, 16),
		Position = UDim2.new(0, 0, 0.5, -8),
		BackgroundColor3 = COL.accent, Visible = false,
	}, b)
	corner(ind, 2)
	local lbl = new("TextLabel", {
		Size = UDim2.new(1, -22, 1, 0),
		Position = UDim2.new(0, 14, 0, 0),
		BackgroundTransparency = 1,
		Text = label, Font = Enum.Font.GothamMedium, TextSize = 13,
		TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Left,
	}, b)
	b.MouseButton1Click:Connect(function() select_page(page) end)
	TABS[#TABS + 1] = { btn = b, lbl = lbl, ind = ind, page = page }
	return b
end

--========================================================================
-- COMPONENTS
--========================================================================
local function section(parent, text)
	return new("TextLabel", {
		Size = UDim2.new(1, 0, 0, 26),
		BackgroundTransparency = 1,
		Text = string.upper(text),
		Font = Enum.Font.GothamBold, TextSize = 10,
		TextColor3 = COL.dim,
		TextXAlignment = Enum.TextXAlignment.Left,
		LayoutOrder = next_order(parent),
	}, parent)
end

local function toggle(parent, label, default, cb)
	local row = new("Frame", {
		Size = UDim2.new(1, 0, 0, 34),
		BackgroundColor3 = COL.panel,
		LayoutOrder = next_order(parent),
	}, parent)
	corner(row, 8)
	new("TextLabel", {
		Size = UDim2.new(1, -64, 1, 0), Position = UDim2.new(0, 12, 0, 0),
		BackgroundTransparency = 1, Text = label,
		Font = Enum.Font.GothamMedium, TextSize = 13,
		TextColor3 = COL.text, TextXAlignment = Enum.TextXAlignment.Left,
	}, row)
	local btn = new("TextButton", {
		Size = UDim2.fromOffset(38, 20),
		Position = UDim2.new(1, -50, 0.5, -10),
		BackgroundColor3 = COL.off, Text = "", AutoButtonColor = false,
	}, row)
	corner(btn, 10)
	local knob = new("Frame", {
		Size = UDim2.fromOffset(14, 14),
		Position = UDim2.new(0, 3, 0.5, -7),
		BackgroundColor3 = COL.text,
	}, btn)
	corner(knob, 7)
	local state = default and true or false
	local function render()
		Tween:Create(btn, TweenInfo.new(0.14), {
			BackgroundColor3 = state and COL.accent or COL.off,
		}):Play()
		Tween:Create(knob, TweenInfo.new(0.14), {
			Position = state and UDim2.new(1, -17, 0.5, -7) or UDim2.new(0, 3, 0.5, -7),
		}):Play()
	end
	render()
	local function set(v)
		state = v and true or false
		render()
		if cb then cb(state) end
	end
	btn.MouseButton1Click:Connect(function() set(not state) end)
	return set
end

local function slider(parent, label, min, max, default, step, cb)
	local row = new("Frame", {
		Size = UDim2.new(1, 0, 0, 48),
		BackgroundColor3 = COL.panel,
		LayoutOrder = next_order(parent),
	}, parent)
	corner(row, 8)
	new("TextLabel", {
		Size = UDim2.new(1, -110, 0, 18), Position = UDim2.new(0, 12, 0, 7),
		BackgroundTransparency = 1, Text = label,
		Font = Enum.Font.GothamMedium, TextSize = 13,
		TextColor3 = COL.text, TextXAlignment = Enum.TextXAlignment.Left,
	}, row)
	local value_lbl = new("TextLabel", {
		Size = UDim2.new(0, 90, 0, 18), Position = UDim2.new(1, -102, 0, 7),
		BackgroundTransparency = 1, Text = tostring(default),
		Font = Enum.Font.GothamBold, TextSize = 12,
		TextColor3 = COL.accent, TextXAlignment = Enum.TextXAlignment.Right,
	}, row)
	local hit = new("TextButton", {
		Size = UDim2.new(1, -24, 0, 16),
		Position = UDim2.new(0, 12, 0, 27),
		BackgroundTransparency = 1, Text = "",
	}, row)
	local line = new("Frame", {
		Size = UDim2.new(1, 0, 0, 3),
		Position = UDim2.new(0, 0, 0.5, -1),
		BackgroundColor3 = COL.off,
	}, hit)
	corner(line, 2)
	local fill = new("Frame", { Size = UDim2.new(0, 0, 1, 0), BackgroundColor3 = COL.accent }, line)
	corner(fill, 2)
	local knob = new("Frame", {
		Size = UDim2.fromOffset(12, 12),
		AnchorPoint = Vector2.new(0.5, 0.5),
		Position = UDim2.new(0, 0, 0.5, 0),
		BackgroundColor3 = COL.text,
	}, hit)
	corner(knob, 6)

	local function apply(v, fire)
		v = math.clamp(v, min, max)
		if step and step > 0 then
			v = math.floor((v - min) / step + 0.5) * step + min
		end
		v = math.clamp(v, min, max)
		local a = (v - min) / (max - min)
		fill.Size = UDim2.new(a, 0, 1, 0)
		knob.Position = UDim2.new(a, 0, 0.5, 0)
		value_lbl.Text = (step and step < 1)
			and string.format("%.2f", v)
			or tostring(math.floor(v + 0.5))
		if fire and cb then cb(v) end
	end
	apply(default, false)

	local dragging = false
	local function from_x(x)
		local a = (x - hit.AbsolutePosition.X) / math.max(hit.AbsoluteSize.X, 1)
		apply(min + math.clamp(a, 0, 1) * (max - min), true)
	end
	hit.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			from_x(input.Position.X)
		end
	end)
	track(UIS.InputChanged:Connect(function(input)
		if dragging
		and (input.UserInputType == Enum.UserInputType.MouseMovement
		or input.UserInputType == Enum.UserInputType.Touch) then
			from_x(input.Position.X)
		end
	end))
	track(UIS.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch then
			dragging = false
		end
	end))
	return apply
end

local function button(parent, label, cb, color)
	local b = new("TextButton", {
		Size = UDim2.new(1, 0, 0, 34),
		BackgroundColor3 = color or COL.panel2,
		Text = label, Font = Enum.Font.GothamBold, TextSize = 11,
		TextColor3 = COL.text, AutoButtonColor = false,
		LayoutOrder = next_order(parent),
	}, parent)
	corner(b, 8)
	b.MouseButton1Click:Connect(function() if cb then cb() end end)
	return b
end

local function keybind(parent, label, default, cb)
	local row = new("Frame", {
		Size = UDim2.new(1, 0, 0, 34),
		BackgroundColor3 = COL.panel,
		LayoutOrder = next_order(parent),
	}, parent)
	corner(row, 8)
	new("TextLabel", {
		Size = UDim2.new(1, -120, 1, 0), Position = UDim2.new(0, 12, 0, 0),
		BackgroundTransparency = 1, Text = label,
		Font = Enum.Font.GothamMedium, TextSize = 13,
		TextColor3 = COL.text, TextXAlignment = Enum.TextXAlignment.Left,
	}, row)
	local btn = new("TextButton", {
		Size = UDim2.new(0, 100, 0, 24),
		Position = UDim2.new(1, -110, 0.5, -12),
		BackgroundColor3 = COL.panel2,
		Text = default.Name,
		Font = Enum.Font.GothamBold, TextSize = 11,
		TextColor3 = COL.accent, AutoButtonColor = false,
	}, row)
	corner(btn, 6)
	local current = default
	local conn
	local function stop()
		if conn then conn:Disconnect(); conn = nil end
		binding_active = false
		btn.Text = current.Name
		btn.TextColor3 = COL.accent
	end
	btn.MouseButton1Click:Connect(function()
		if conn then stop(); return end
		binding_active = true
		btn.Text = "apertar tecla"
		btn.TextColor3 = COL.text
		conn = UIS.InputBegan:Connect(function(input)
			if input.UserInputType ~= Enum.UserInputType.Keyboard then return end
			if input.KeyCode == Enum.KeyCode.Escape then stop(); return end
			current = input.KeyCode
			stop()
			if cb then cb(current) end
		end)
		task.delay(5, function() if conn then stop() end end)
	end)
	return function() return current end
end

local function textbox(parent, label, default, cb)
	local row = new("Frame", {
		Size = UDim2.new(1, 0, 0, 34),
		BackgroundColor3 = COL.panel,
		LayoutOrder = next_order(parent),
	}, parent)
	corner(row, 8)
	new("TextLabel", {
		Size = UDim2.new(1, -140, 1, 0), Position = UDim2.new(0, 12, 0, 0),
		BackgroundTransparency = 1, Text = label,
		Font = Enum.Font.GothamMedium, TextSize = 13,
		TextColor3 = COL.text, TextXAlignment = Enum.TextXAlignment.Left,
	}, row)
	local box = new("TextBox", {
		Size = UDim2.new(0, 120, 0, 24),
		Position = UDim2.new(1, -130, 0.5, -12),
		BackgroundColor3 = COL.panel2,
		Text = default,
		Font = Enum.Font.GothamBold, TextSize = 11,
		TextColor3 = COL.accent,
		ClearTextOnFocus = false,
	}, row)
	corner(box, 6)
	new("UIPadding", { PaddingLeft = UDim.new(0, 8), PaddingRight = UDim.new(0, 8) }, box)
	box.FocusLost:Connect(function()
		if cb then cb(box.Text) end
	end)
	return box
end

local function status_row(parent)
	local s = new("TextLabel", {
		Size = UDim2.new(1, 0, 0, 28),
		BackgroundColor3 = COL.panel2,
		Text = "idle",
		Font = Enum.Font.GothamBold, TextSize = 11,
		TextColor3 = COL.dim,
		TextXAlignment = Enum.TextXAlignment.Left,
		LayoutOrder = next_order(parent),
	}, parent)
	corner(s, 8)
	new("UIPadding", { PaddingLeft = UDim.new(0, 10) }, s)
	return s
end

--========================================================================
-- STEAL PAGE
--========================================================================
local steal_page = make_page(1)
make_tab("STEAL", 1, steal_page)
local sa = scroll_area(steal_page)

local status = status_row(sa)

section(sa, "auto")
toggle(sa, "Auto steal", false, function(v)
	cfg.auto = v
	status_txt = v and "auto ligado" or "auto desligado"
end)
slider(sa, "Delay", 0.05, 1.5, 0.30, 0.05, function(v) cfg.delay = v end)
textbox(sa, "Keyword", "steal", function(t) cfg.keyword = t end)
slider(sa, "Min value", 0, 100000, 0, 500, function(v) cfg.minval = v end)
slider(sa, "Max distance", 50, 5000, 2000, 50, function(v) cfg.maxdist = v end)

section(sa, "behavior")
toggle(sa, "Teleport to target", true, function(v) cfg.tp = v end)
toggle(sa, "Legit pause", true, function(v) cfg.legit = v end)
toggle(sa, "Ignore own base", true, function(v) cfg.own = v end)
keybind(sa, "Steal nearest key", Enum.KeyCode.F, function(k) cfg.key = k end)

section(sa, "actions")
button(sa, "STEAL NEAREST NOW", function()
	scan()
	local e = best_entry()
	if e then steal(e, true) end
end, COL.accent)
button(sa, "STEAL HIGHEST NOW", function()
	scan()
	local best
	for _, e in ipairs(CACHE) do
		if matches(e) and (not cfg.own or not is_own(e.model)) then
			if not best or e.value > best.value then best = e end
		end
	end
	if best then steal(best, true) end
end)

section(sa, "extra")
toggle(sa, "Auto buy", false, function(v) cfg.buy = v end)
slider(sa, "Buy max price", 0, 5000000, 500000, 5000, function(v) cfg.buy_max = v end)
toggle(sa, "Auto collect cash", false, function(v) cfg.collect = v end)

--========================================================================
-- LIST PAGE
--========================================================================
local list_page = make_page(2)
make_tab("LIST", 2, list_page)

local list_top = new("Frame", {
	Size = UDim2.new(1, 0, 0, 30),
	BackgroundTransparency = 1,
}, list_page)

local scan_btn = new("TextButton", {
	Size = UDim2.new(0, 90, 1, 0),
	BackgroundColor3 = COL.accent,
	Text = "SCAN",
	Font = Enum.Font.GothamBold, TextSize = 11,
	TextColor3 = Color3.new(1, 1, 1), AutoButtonColor = false,
}, list_top)
corner(scan_btn, 8)

local list_count = new("TextLabel", {
	Size = UDim2.new(1, -100, 1, 0),
	Position = UDim2.new(0, 100, 0, 0),
	BackgroundTransparency = 1,
	Text = "0 prompts",
	Font = Enum.Font.GothamMedium, TextSize = 11,
	TextColor3 = COL.dim,
	TextXAlignment = Enum.TextXAlignment.Left,
}, list_top)

local list_scroll = new("ScrollingFrame", {
	Size = UDim2.new(1, 0, 1, -38),
	Position = UDim2.new(0, 0, 0, 38),
	BackgroundTransparency = 1,
	BorderSizePixel = 0,
	ScrollBarThickness = 3,
	ScrollBarImageColor3 = COL.stroke,
	CanvasSize = UDim2.new(0, 0, 0, 0),
	AutomaticCanvasSize = Enum.AutomaticSize.Y,
	ScrollingDirection = Enum.ScrollingDirection.Y,
}, list_page)
new("UIListLayout", { Padding = UDim.new(0, 4), SortOrder = Enum.SortOrder.LayoutOrder }, list_scroll)

local function refresh_list()
	for _, c in ipairs(list_scroll:GetChildren()) do
		if not c:IsA("UIListLayout") then c:Destroy() end
	end
	scan()
	local n = 0
	for _, e in ipairs(CACHE) do
		n = n + 1
		local row = new("Frame", {
			Size = UDim2.new(1, -4, 0, 30),
			BackgroundColor3 = COL.panel,
			LayoutOrder = n,
		}, list_scroll)
		corner(row, 7)

		local label = string.format("%s  ·  %s  ·  %d",
			e.name, fmt(e.value), math.floor(dist_of(e)))

		new("TextLabel", {
			Size = UDim2.new(1, -60, 1, 0), Position = UDim2.new(0, 10, 0, 0),
			BackgroundTransparency = 1,
			Text = label,
			Font = Enum.Font.GothamMedium, TextSize = 11,
			TextColor3 = matches(e) and COL.text or COL.dim,
			TextXAlignment = Enum.TextXAlignment.Left,
			TextTruncate = Enum.TextTruncate.AtEnd,
		}, row)

		local go = new("TextButton", {
			Size = UDim2.fromOffset(48, 22),
			Position = UDim2.new(1, -56, 0.5, -11),
			BackgroundColor3 = COL.accent,
			Text = "GO",
			Font = Enum.Font.GothamBold, TextSize = 10,
			TextColor3 = Color3.new(1, 1, 1), AutoButtonColor = false,
		}, row)
		corner(go, 6)
		go.MouseButton1Click:Connect(function() steal(e, true) end)
	end
	list_count.Text = n .. " prompts"
end

scan_btn.MouseButton1Click:Connect(refresh_list)

--========================================================================
-- ESP PAGE
--========================================================================
local esp_page = make_page(3)
make_tab("ESP", 3, esp_page)
local ea = scroll_area(esp_page)

section(ea, "esp")
toggle(ea, "Enabled", false, function(v) cfg.esp = v end)
toggle(ea, "Text (name / value / dist)", true, function(v) cfg.esp_text = v end)
toggle(ea, "Highlight", true, function(v) cfg.esp_hl = v end)
slider(ea, "Max distance", 50, 5000, 2000, 50, function(v) cfg.maxdist = v end)

if not DrawingLib then
	local note = new("TextLabel", {
		Size = UDim2.new(1, 0, 0, 44),
		BackgroundColor3 = COL.panel,
		Text = "Drawing ausente — só Highlight disponível",
		Font = Enum.Font.GothamMedium, TextSize = 11,
		TextColor3 = COL.dim, TextWrapped = true,
		LayoutOrder = next_order(ea),
	}, ea)
	corner(note, 8)
end

--========================================================================
-- MISC PAGE
--========================================================================
local misc_page = make_page(4)
make_tab("MISC", 4, misc_page)
local ma = scroll_area(misc_page)

section(ma, "misc")
toggle(ma, "Anti-AFK", false, function(v) cfg.antiafk = v end)
button(ma, "RESCAN", function() scan() end)
button(ma, "UNLOAD / REMOVE", function()
	if unload_dummy then unload_dummy() end
end, COL.danger)

bind(LP.Idled, function()
	if not cfg.antiafk then return end
	pcall(function()
		local vu = game:GetService("VirtualUser")
		vu:CaptureController()
		vu:ClickButton2(Vector2.new())
	end)
end)

--========================================================================
-- LOOPS
--========================================================================
local scan_acc = 0
bind(RunService.Heartbeat, function(dt)
	scan_acc = scan_acc + dt
	if scan_acc < 0.6 then return end
	scan_acc = 0
	scan()
end)

local esp_acc = 0
bind(RunService.RenderStepped, function(dt)
	esp_acc = esp_acc + dt
	if esp_acc < 0.1 then return end
	esp_acc = 0
	pcall(esp_tick)
	status.Text = status_txt .. "  ·  " .. last_action .. "  ·  " .. #CACHE .. " prompts"
end)

local collect_acc = 0
bind(RunService.Heartbeat, function(dt)
	collect_acc = collect_acc + dt
	if collect_acc < 1.0 then return end
	collect_acc = 0
	if cfg.collect then pcall(collect_cash) end
end)

task.spawn(function()
	while running do
		if cfg.auto then
			local e = best_entry()
			if e then
				pcall(steal, e, false)
			else
				status_txt = "sem alvo válido"
				task.wait(0.3)
			end
		else
			task.wait(0.2)
		end
	end
end)

task.spawn(function()
	while running do
		if cfg.buy then
			scan()
			local e = buy_entry()
			if e then
				pcall(steal, e, true)
			else
				task.wait(0.5)
			end
		else
			task.wait(0.5)
		end
	end
end)

bind(UIS.InputBegan, function(input, processed)
	if processed or binding_active then return end
	if input.UserInputType ~= Enum.UserInputType.Keyboard then return end
	if input.KeyCode ~= cfg.key then return end
	scan()
	local e = best_entry()
	if e then steal(e, true) end
end)

--========================================================================
-- SCALE + UNLOAD
--========================================================================
local last_scale = 0
bind(RunService.Heartbeat, function()
	local s = compute_scale()
	if math.abs(s - last_scale) > 0.01 then
		last_scale = s
		win_scale.Scale = s
	end
end)

function unload_dummy()
	running = false
	for _, c in ipairs(CONN) do pcall(function() c:Disconnect() end) end
	for i = #CONN, 1, -1 do CONN[i] = nil end
	for _, d in ipairs(DRAW) do pcall(function() d:Remove() end) end
	for i = #DRAW, 1, -1 do DRAW[i] = nil end
	for model in pairs(esp_map) do esp_drop(model) end
	pcall(function() gui:Destroy() end)
	if type(getgenv) == "function" then getgenv().ZERO_STEAL_CLEANUP = nil end
end

if type(getgenv) == "function" then
	getgenv().ZERO_STEAL_CLEANUP = unload_dummy
end

select_page(steal_page)
print("[ZERO] steal carregado · " .. tostring(#CACHE) .. " prompts")
