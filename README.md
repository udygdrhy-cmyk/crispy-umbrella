local UserInputService = game:GetService("UserInputService")
local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")
local camera = Workspace.CurrentCamera

local UI_NAME = "AdminQMenu_CircleNotification"
local myScreenGui = nil
local myFrame = nil
local notifyContainer = nil

-- [ตัวแปรสถานะระบบต่างๆ]
local clayGraphicsEnabled = _G.ClayGraphicsEnabled or false
local originalMaterials = _G.OriginalMaterials or {}
_G.OriginalMaterials = originalMaterials

local speedEnabled = false
local jumpEnabled = false
local infJumpEnabled = false
local noclipEnabled = false
local flyEnabled = false
local ramEnabled = false
local isSprinting = false  -- 🏃 สถานะกดค้างปุ่มวิ่ง
local spinEnabled = false  -- 🌪️ สถานะระบบหมุน (SS)
local spinSpeed = 5000     -- 🌪️ ความเร็วหมุน
local spinConnection = nil -- 🌪️ ตัวแปรสำหรับ Loop หมุน

-- 🎯 ตัวแปรระบบล็อคหัว (Head Lock)
local headLockEnabled = false
local headLockTarget = nil
local headLockKeybind = Enum.KeyCode.E
local headLockConnection = nil
local globalToggleHeadLockBtn = nil
local headLockTargetLabel = nil

-- [ตัวแปรระบบ ESP]
local espEnabled = false
local espMaxDistance = 500
local espShowLines = true
local espShowNames = true
local espShowDistance = true
local espContainer = nil
local espObjects = {}
local espConnection = nil

local customSpeed = 50     -- 🏃 ค่าความเร็วมาตรฐานเมื่อเปิดปุ่ม หรือกดปุ่มวิ่ง
local sprintMultiplier = 2 -- 🏃 SPEED x2! เมื่อกดปุ่มวิ่ง ความเร็วจะ x2 ของค่าที่ตั้ง
local customJump = 50
local flySpeed = 50
local normalWalkSpeed = 24 -- 🏃 ความเร็วเดินปกติ

-- [ตัวแปรปุ่มลัด]
local flyKeybind    = Enum.KeyCode.CapsLock
local clayKeybind   = Enum.KeyCode.L
local ramKeybind    = Enum.KeyCode.K
local espKeybind    = Enum.KeyCode.Z
local sprintKeybind = Enum.KeyCode.LeftControl -- 🏃 ปุ่มลัดวิ่งเร็ว (ค่าเริ่มต้น Ctrl)
local spinKeybind   = Enum.KeyCode.X          -- 🌪️ ปุ่มลัดระบบหมุน (SS)

local bodyVelocity = nil
local bodyGyro     = nil
local flyConnection = nil
local ramConnection = nil

local globalToggleClayBtn = nil
local globalToggleFlyBtn  = nil
local globalToggleRamBtn  = nil
local globalToggleEspBtn  = nil
local globalToggleSpinBtn = nil -- 🌪️ ปุ่ม UI ระบบหมุน

local function trimString(s)
	return s:match("^%s*(.-)%s*$")
end

-- 🎵 เสียงแจ้งเตือน
local function playNotificationSound(soundType)
	local sound = Instance.new("Sound")
	sound.SoundId = "rbxassetid://138070973725636"
	if soundType == "On"  then sound.Volume = 0.5
	elseif soundType == "Off" then sound.Volume = 0.4
	elseif soundType == "Hit" then sound.Volume = 0.7
	end
	sound.Parent = Workspace
	sound:Play()
	sound.Ended:Connect(function() sound:Destroy() end)
end

-- 🔔 แจ้งเตือนแบบแคปซูล
local function createNotification(text, statusType)
	if not myScreenGui then return end
	if not notifyContainer then
		notifyContainer = Instance.new("Frame", myScreenGui)
		notifyContainer.Name = "NotifyContainer"
		notifyContainer.Size = UDim2.new(0, 340, 0, 350)
		notifyContainer.Position = UDim2.new(1, -350, 1, -360)
		notifyContainer.BackgroundTransparency = 1
		local layout = Instance.new("UIListLayout", notifyContainer)
		layout.VerticalAlignment = Enum.VerticalAlignment.Bottom
		layout.SortOrder = Enum.SortOrder.LayoutOrder
		layout.Padding = UDim.new(0, 10)
	end
	local notifBox = Instance.new("Frame", notifyContainer)
	notifBox.Size = UDim2.new(1, 0, 0, 50)
	notifBox.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
	notifBox.BackgroundTransparency = 1
	Instance.new("UICorner", notifBox).CornerRadius = UDim.new(1, 0)
	local iconImage = Instance.new("ImageLabel", notifBox)
	iconImage.Size = UDim2.new(0, 36, 0, 36)
	iconImage.Position = UDim2.new(0, 8, 0.5, -18)
	iconImage.BackgroundTransparency = 1
	iconImage.ImageTransparency = 1
	Instance.new("UICorner", iconImage).CornerRadius = UDim.new(1, 0)
	if statusType == "On" or statusType == "Hit" then
		iconImage.Image = "rbxassetid://134298840674809"
	elseif statusType == "Off" then
		iconImage.Image = "rbxassetid://131342117768521"
	end
	local textLabel = Instance.new("TextLabel", notifBox)
	textLabel.Size = UDim2.new(1, -65, 1, 0)
	textLabel.Position = UDim2.new(0, 52, 0, 0)
	textLabel.BackgroundTransparency = 1
	textLabel.Text = text
	textLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
	textLabel.TextSize = 15
	textLabel.Font = Enum.Font.SourceSansBold
	textLabel.TextXAlignment = Enum.TextXAlignment.Left
	textLabel.TextTransparency = 1
	TweenService:Create(notifBox,  TweenInfo.new(0.3), {BackgroundTransparency = 0.1}):Play()
	TweenService:Create(iconImage, TweenInfo.new(0.3), {ImageTransparency = 0}):Play()
	TweenService:Create(textLabel, TweenInfo.new(0.3), {TextTransparency = 0}):Play()
	task.delay(2.2, function()
		if notifBox and notifBox.Parent then
			TweenService:Create(notifBox,  TweenInfo.new(0.4), {BackgroundTransparency = 1}):Play()
			TweenService:Create(iconImage, TweenInfo.new(0.4), {ImageTransparency = 1}):Play()
			local fo = TweenService:Create(textLabel, TweenInfo.new(0.4), {TextTransparency = 1})
			fo:Play()
			fo.Completed:Connect(function() notifBox:Destroy() end)
		end
	end)
end

-- ============================================================
-- 👁️ ระบบ ESP (Player Tracker)
-- ============================================================

local function getDistanceColor(dist)
	if dist < 50  then return Color3.fromRGB(255, 80,  80)
	elseif dist < 150 then return Color3.fromRGB(255, 200, 50)
	else               return Color3.fromRGB(80,  200, 255)
	end
end

local function removeEspForPlayer(p)
	local obj = espObjects[p]
	if obj then
		if obj.line      and obj.line.Parent      then obj.line:Destroy() end
		if obj.nameBox   and obj.nameBox.Parent   then obj.nameBox:Destroy() end
		if obj.distBox   and obj.distBox.Parent   then obj.distBox:Destroy() end
		if obj.highlight and obj.highlight.Parent  then obj.highlight:Destroy() end
		espObjects[p] = nil
	end
end

local function createEspForPlayer(p)
	if p == player then return end
	if not espContainer then return end
	removeEspForPlayer(p)

	local line = Instance.new("Frame", espContainer)
	line.Name = "ESP_Line_" .. p.Name
	line.BackgroundColor3 = Color3.fromRGB(255, 80, 80)
	line.BorderSizePixel = 0
	line.AnchorPoint = Vector2.new(0.5, 0)
	line.Size = UDim2.new(0, 2, 0, 0)

	local nameBox = Instance.new("Frame", espContainer)
	nameBox.Name = "ESP_NameBox_" .. p.Name
	nameBox.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
	nameBox.BackgroundTransparency = 0.35
	nameBox.BorderSizePixel = 0
	nameBox.Size = UDim2.new(0, 140, 0, 28)
	nameBox.AnchorPoint = Vector2.new(0.5, 1)
	Instance.new("UICorner", nameBox).CornerRadius = UDim.new(0, 4)

	local nameLabel = Instance.new("TextLabel", nameBox)
	nameLabel.Size = UDim2.new(1, -6, 1, 0)
	nameLabel.Position = UDim2.new(0, 3, 0, 0)
	nameLabel.BackgroundTransparency = 1
	nameLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
	nameLabel.TextSize = 13
	nameLabel.Font = Enum.Font.SourceSansBold
	nameLabel.TextXAlignment = Enum.TextXAlignment.Center
	nameLabel.TextStrokeTransparency = 0.4
	nameLabel.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
	nameLabel.Text = p.DisplayName

	local distBox = Instance.new("Frame", espContainer)
	distBox.Name = "ESP_DistBox_" .. p.Name
	distBox.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
	distBox.BackgroundTransparency = 0.35
	distBox.BorderSizePixel = 0
	distBox.Size = UDim2.new(0, 80, 0, 22)
	distBox.AnchorPoint = Vector2.new(0.5, 1)
	Instance.new("UICorner", distBox).CornerRadius = UDim.new(0, 4)

	local distLabel = Instance.new("TextLabel", distBox)
	distLabel.Size = UDim2.new(1, -4, 1, 0)
	distLabel.Position = UDim2.new(0, 2, 0, 0)
	distLabel.BackgroundTransparency = 1
	distLabel.TextColor3 = Color3.fromRGB(180, 255, 180)
	distLabel.TextSize = 12
	distLabel.Font = Enum.Font.SourceSansBold
	distLabel.TextXAlignment = Enum.TextXAlignment.Center
	distLabel.TextStrokeTransparency = 0.4
	distLabel.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
	distLabel.Text = "0 studs"

	local highlight = Instance.new("Highlight")
	highlight.FillTransparency = 0.75
	highlight.OutlineTransparency = 0.1
	highlight.OutlineColor = Color3.fromRGB(255, 80, 80)
	highlight.FillColor    = Color3.fromRGB(255, 80, 80)

	espObjects[p] = {
		line = line,
		nameBox = nameBox,
		nameLabel = nameLabel,
		distBox = distBox,
		distLabel = distLabel,
		highlight = highlight,
	}
end

local function updateEsp()
	if not myScreenGui then return end
	local myChar = player.Character
	local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
	if not myRoot then return end

	local vpSize = camera.ViewportSize
	local originX = vpSize.X / 2
	local originY = vpSize.Y

	for _, p in ipairs(Players:GetPlayers()) do
		if p == player then continue end
		local obj = espObjects[p]
		if not obj then continue end

		local targetChar = p.Character
		local targetRoot = targetChar and targetChar:FindFirstChild("HumanoidRootPart")
		local targetHead = targetChar and targetChar:FindFirstChild("Head")

		local shouldShow = targetRoot ~= nil
		obj.line.Visible    = shouldShow and espShowLines
		obj.nameBox.Visible = shouldShow and espShowNames
		obj.distBox.Visible = shouldShow and espShowDistance
		obj.highlight.Parent = (shouldShow and targetChar) and targetChar or nil

		if not shouldShow then continue end

		local dist = math.floor((myRoot.Position - targetRoot.Position).Magnitude)
		if dist > espMaxDistance then
			obj.line.Visible    = false
			obj.nameBox.Visible = false
			obj.distBox.Visible = false
			obj.highlight.Parent = nil
			continue
		end

		local col = getDistanceColor(dist)
		obj.line.BackgroundColor3      = col
		obj.highlight.OutlineColor     = col
		obj.highlight.FillColor        = col
		obj.distLabel.Text             = dist .. " studs"
		obj.nameLabel.Text             = p.DisplayName .. "\n[@" .. p.Name .. "]"

		local headPos = targetHead and targetHead.Position or targetRoot.Position + Vector3.new(0, 2, 0)
		local screenPos, onScreen = camera:WorldToViewportPoint(headPos)

		if onScreen then
			local sx, sy = screenPos.X, screenPos.Y

			if espShowLines then
				local dx = sx - originX
				local dy = sy - originY
				local length = math.sqrt(dx*dx + dy*dy)
				local angle  = math.atan2(dy, dx) - math.pi/2
				obj.line.Size     = UDim2.new(0, 2, 0, length)
				obj.line.Position = UDim2.new(0, originX, 0, originY)
				obj.line.Rotation = math.deg(angle)
			end

			if espShowNames then
				obj.nameBox.Position = UDim2.new(0, sx, 0, sy - 4)
				obj.nameBox.Visible = true
			end

			if espShowDistance then
				obj.distBox.Position = UDim2.new(0, sx, 0, sy + 20)
				obj.distBox.Visible = true
			end
		else
			obj.nameBox.Visible = false
			obj.distBox.Visible = false

			if espShowLines then
				local dx = sx - originX
				local dy = sy - originY
				local length = math.sqrt(dx*dx + dy*dy)
				local angle  = math.atan2(dy, dx) - math.pi/2
				obj.line.Size     = UDim2.new(0, 2, 0, math.min(length, vpSize.Y))
				obj.line.Position = UDim2.new(0, originX, 0, originY)
				obj.line.Rotation = math.deg(angle)
				obj.line.Visible  = espShowLines
			else
				obj.line.Visible = false
			end
		end
	end
end

local function startEsp()
	if not myScreenGui then return end
	if not espContainer then
		espContainer = Instance.new("Frame", myScreenGui)
		espContainer.Name = "ESP_Container"
		espContainer.Size = UDim2.new(1, 0, 1, 0)
		espContainer.BackgroundTransparency = 1
		espContainer.ZIndex = 1
	end
	for _, p in ipairs(Players:GetPlayers()) do
		if p ~= player then createEspForPlayer(p) end
	end
	Players.PlayerAdded:Connect(function(p) if espEnabled then createEspForPlayer(p) end end)
	Players.PlayerRemoving:Connect(function(p) removeEspForPlayer(p) end)
	if espConnection then espConnection:Disconnect() end
	espConnection = RunService.RenderStepped:Connect(updateEsp)
end

local function stopEsp()
	if espConnection then espConnection:Disconnect() espConnection = nil end
	for p, _ in pairs(espObjects) do removeEspForPlayer(p) end
	espObjects = {}
end

local function toggleEsp()
	espEnabled = not espEnabled
	if espEnabled then startEsp(); createNotification("👁️ ESP: เปิดใช้งานแล้ว", "On"); playNotificationSound("On")
	else stopEsp(); createNotification("🛑 ESP: ปิดใช้งานแล้ว", "Off"); playNotificationSound("Off") end
	if globalToggleEspBtn then globalToggleEspBtn.Text = espEnabled and "🟢 ESP Tracker: เปิด" or "🔴 ESP Tracker: ปิด"; globalToggleEspBtn.BackgroundColor3 = espEnabled and Color3.fromRGB(46, 139, 87) or Color3.fromRGB(178, 34, 34) end
end

-- ============================================================
-- 🌪️ ระบบ Spinbot (SS) หมุนเร็วทำให้เตะออกได้
-- ============================================================
local function toggleSpin()
	spinEnabled = not spinEnabled
	if spinEnabled then
		createNotification("🌪️ ระบบหมุน (SS): เปิดใช้งาน", "On"); playNotificationSound("On")
		if spinConnection then spinConnection:Disconnect() end
		spinConnection = RunService.RenderStepped:Connect(function()
			local character = player.Character; local rootPart = character and character:FindFirstChild("HumanoidRootPart")
			if rootPart then rootPart.CFrame = rootPart.CFrame * CFrame.Angles(0, math.rad(spinSpeed * 0.015), 0) end
		end)
	else
		createNotification("🛑 ระบบหมุน (SS): ปิดใช้งาน", "Off"); playNotificationSound("Off")
		if spinConnection then spinConnection:Disconnect(); spinConnection = nil end
	end
	if globalToggleSpinBtn then globalToggleSpinBtn.Text = spinEnabled and "🟢 ระบบหมุน SS: เปิด" or "🔴 ระบบหมุน SS: ปิด"; globalToggleSpinBtn.BackgroundColor3 = spinEnabled and Color3.fromRGB(46, 139, 87) or Color3.fromRGB(178, 34, 34) end
end

-- ============================================================
-- 🎯 ระบบล็อคหัว (Head Lock) - x-sd
-- ============================================================
local function findClosestPlayerToCenter()
	local closestPlayer = nil
	local shortestDist = math.huge
	local centerScreen = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)

	for _, p in ipairs(Players:GetPlayers()) do
		if p == player then continue end
		local char = p.Character
		local head = char and char:FindFirstChild("Head")
		local hum = char and char:FindFirstChildOfClass("Humanoid")

		if head and hum and hum.Health > 0 then
			local screenPos, onScreen = camera:WorldToViewportPoint(head.Position)
			if onScreen then
				local dist = (Vector2.new(screenPos.X, screenPos.Y) - centerScreen).Magnitude
				if dist < shortestDist then
					shortestDist = dist
					closestPlayer = p
				end
			end
		end
	end
	return closestPlayer
end

local function updateHeadLockTargetLabel()
	if headLockTargetLabel then
		headLockTargetLabel.Text = headLockTarget and ("🎯 ล็อคเป้า: " .. headLockTarget.DisplayName) or "🎯 ล็อคเป้า: ไม่มี"
	end
end

local function startHeadLockLoop()
	if headLockConnection then headLockConnection:Disconnect() end
	headLockConnection = RunService.RenderStepped:Connect(function()
		if headLockEnabled and headLockTarget then
			local char = headLockTarget.Character
			local head = char and char:FindFirstChild("Head")
			local hum = char and char:FindFirstChildOfClass("Humanoid")
			if head and hum and hum.Health > 0 then
				camera.CFrame = CFrame.new(camera.CFrame.Position, head.Position)
			else
				headLockTarget = nil
				updateHeadLockTargetLabel()
			end
		end
	end)
end

local function toggleHeadLockSystem()
	headLockEnabled = not headLockEnabled
	if headLockEnabled then
		createNotification("🎯 ระบบล็อคหัว: เปิดใช้งาน", "On"); playNotificationSound("On")
		startHeadLockLoop()
	else
		createNotification("🛑 ระบบล็อคหัว: ปิดใช้งาน", "Off"); playNotificationSound("Off")
		headLockTarget = nil
		if headLockConnection then headLockConnection:Disconnect(); headLockConnection = nil end
	end
	updateHeadLockTargetLabel()
	if globalToggleHeadLockBtn then globalToggleHeadLockBtn.Text = headLockEnabled and "🟢 ระบบล็อคหัว: เปิด" or "🔴 ระบบล็อคหัว: ปิด"; globalToggleHeadLockBtn.BackgroundColor3 = headLockEnabled and Color3.fromRGB(46, 139, 87) or Color3.fromRGB(178, 34, 34) end
end

local function handleHeadLockKey()
	if not headLockEnabled then return end
	if headLockTarget then
		headLockTarget = nil
		createNotification("🎯 ยกเลิกล็อคเป้าหมาย", "Off")
		playNotificationSound("Off")
	else
		local target = findClosestPlayerToCenter()
		if target then
			headLockTarget = target
			createNotification("🎯 ล็อคเป้าหมาย: " .. target.DisplayName, "Hit")
			playNotificationSound("Hit")
		else
			createNotification("🚫 ไม่พบผู้เล่นในจอ", "Off")
		end
	end
	updateHeadLockTargetLabel()
end

-- ============================================================
-- ระบบอื่นๆ
-- ============================================================

local function setClayGraphics(enable)
	clayGraphicsEnabled = enable; _G.ClayGraphicsEnabled = enable
	if enable then
		for _, v in ipairs(Workspace:GetDescendants()) do if v:IsA("BasePart") and not v:IsA("Terrain") then if not originalMaterials[v] then originalMaterials[v] = v.Material end; v.Material = Enum.Material.SmoothPlastic end end
		createNotification("⚙️ ภาพดินน้ำมัน: เปิดใช้งาน", "On"); playNotificationSound("On")
	else
		for part, material in pairs(originalMaterials) do if part and part.Parent then part.Material = material end end; table.clear(originalMaterials)
		createNotification("🛑 ภาพดินน้ำมัน: ปิดใช้งาน", "Off"); playNotificationSound("Off")
	end
	if globalToggleClayBtn then globalToggleClayBtn.Text = clayGraphicsEnabled and "🟢 ภาพดินน้ำมัน: เปิด" or "🔴 ภาพดินน้ำมัน: ปิด"; globalToggleClayBtn.BackgroundColor3 = clayGraphicsEnabled and Color3.fromRGB(46, 139, 87) or Color3.fromRGB(178, 34, 34) end
end

local function startRamSystem()
	if ramConnection then ramConnection:Disconnect() end
	ramConnection = RunService.Heartbeat:Connect(function()
		if not ramEnabled then return end
		local character = player.Character; local myRoot = character and character:FindFirstChild("HumanoidRootPart")
		if not myRoot then return end
		local checkRadius = 6; local overlapParams = OverlapParams.new(); overlapParams.FilterType = Enum.RaycastFilterType.Exclude; overlapParams.FilterDescendantsInstances = {character}
		local parts = Workspace:GetPartBoundsInRadius(myRoot.Position, checkRadius, overlapParams)
		for _, part in ipairs(parts) do
			if part:IsA("BasePart") and not part.Anchored then
				part:SetNetworkOwner(nil); local pushDirection = (part.Position - myRoot.Position).Unit
				local ramForce = spinEnabled and 300 or 150; part.AssemblyLinearVelocity = (pushDirection * ramForce) + Vector3.new(0, 50, 0); part.AssemblyAngularVelocity = Vector3.new(math.random(-20, 20), math.random(-20, 20), math.random(-20, 20))
			end
			local targetChar = part.Parent; local targetHumanoid = targetChar and targetChar:FindFirstChildOfClass("Humanoid"); local targetRoot = targetChar and targetChar:FindFirstChild("HumanoidRootPart")
			if targetHumanoid and targetRoot and targetChar ~= character then
				local pushDirection = (targetRoot.Position - myRoot.Position).Unit; local kickForce = spinEnabled and 400 or 200; local liftForce = spinEnabled and 150 or 80
				targetRoot.AssemblyLinearVelocity = (pushDirection * kickForce) + Vector3.new(0, liftForce, 0)
				local gyroForce = Instance.new("AngularVelocity"); gyroForce.MaxTorque = 500000; local spinForce = spinEnabled and 100 or 50; gyroForce.AngularVelocity = Vector3.new(math.random(-spinForce, spinForce), math.random(-spinForce, spinForce), math.random(-spinForce, spinForce)); gyroForce.Attachment0 = targetRoot:FindFirstChildOfClass("Attachment") or Instance.new("Attachment", targetRoot); gyroForce.Parent = targetRoot
				task.delay(0.2, function() if gyroForce then gyroForce:Destroy() end end); playNotificationSound("Hit")
			end
		end
	end)
end

local function toggleRamMode()
	ramEnabled = not ramEnabled
	if ramEnabled then createNotification("💥 เปิดโหมดชนสิ่งของปลิว!", "On"); startRamSystem() else createNotification("🛑 ปิดโหมดชนสิ่งของปลิว", "Off"); if ramConnection then ramConnection:Disconnect(); ramConnection = nil end end
	if globalToggleRamBtn then globalToggleRamBtn.Text = ramEnabled and "🟢 ระบบชนกระเด็น: เปิด" or "🔴 ระบบชนกระเด็น: ปิด"; globalToggleRamBtn.BackgroundColor3 = ramEnabled and Color3.fromRGB(46, 139, 87) or Color3.fromRGB(178, 34, 34) end
end

local function disableFly(silent)
	flyEnabled = false
	if flyConnection then flyConnection:Disconnect(); flyConnection = nil end
	if bodyVelocity then bodyVelocity:Destroy(); bodyVelocity = nil end
	if bodyGyro then bodyGyro:Destroy(); bodyGyro = nil end
	local character = player.Character; if character then local humanoid = character:FindFirstChildOfClass("Humanoid"); if humanoid then humanoid.PlatformStand = false end end
	if not silent then createNotification("🛑 โหมดบิน: ปิดใช้งาน", "Off"); playNotificationSound("Off") end
	if globalToggleFlyBtn then globalToggleFlyBtn.Text = "🔴 ระบบบิน: ปิดอยู่"; globalToggleFlyBtn.BackgroundColor3 = Color3.fromRGB(178, 34, 34) end
end

local function enableFly()
	local character = player.Character; if not character then return end; local rootPart = character:FindFirstChild("HumanoidRootPart"); local humanoid = character:FindFirstChildOfClass("Humanoid"); if not rootPart or not humanoid then return end
	flyEnabled = true; humanoid.PlatformStand = true
	bodyVelocity = Instance.new("BodyVelocity"); bodyVelocity.MaxForce = Vector3.new(1e5, 1e5, 1e5); bodyVelocity.Velocity = Vector3.new(0, 0, 0); bodyVelocity.Parent = rootPart
	bodyGyro = Instance.new("BodyGyro"); bodyGyro.MaxTorque = Vector3.new(1e5, 1e5, 1e5); bodyGyro.P = 15000; bodyGyro.D = 500; bodyGyro.CFrame = camera.CFrame; bodyGyro.Parent = rootPart
	createNotification("✈️ โหมดบิน: เปิดใช้งาน", "On"); playNotificationSound("On")
	if globalToggleFlyBtn then globalToggleFlyBtn.Text = "🟢 ระบบบิน: กำลังบิน"; globalToggleFlyBtn.BackgroundColor3 = Color3.fromRGB(46, 139, 87) end

	flyConnection = RunService.RenderStepped:Connect(function()
		if not character.Parent or not rootPart.Parent or not flyEnabled then disableFly(true); return end
		local currentFlySpeed = (speedEnabled or isSprinting) and (customSpeed * sprintMultiplier) or flySpeed; local camCF = camera.CFrame; local camLook = camCF.LookVector; local camRight = camCF.RightVector
		local moveDirection = Vector3.new(0, 0, 0)
		if UserInputService:IsKeyDown(Enum.KeyCode.W) then moveDirection += Vector3.new(camLook.X, 0, camLook.Z) end; if UserInputService:IsKeyDown(Enum.KeyCode.S) then moveDirection -= Vector3.new(camLook.X, 0, camLook.Z) end; if UserInputService:IsKeyDown(Enum.KeyCode.A) then moveDirection -= Vector3.new(camRight.X, 0, camRight.Z) end; if UserInputService:IsKeyDown(Enum.KeyCode.D) then moveDirection += Vector3.new(camRight.X, 0, camRight.Z) end
		if UserInputService:IsKeyDown(Enum.KeyCode.Space) then moveDirection += Vector3.new(0, 1, 0) end; if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then moveDirection -= Vector3.new(0, 1, 0) end

		if moveDirection.Magnitude > 0 then
			bodyVelocity.Velocity = moveDirection.Unit * currentFlySpeed; if not spinEnabled then bodyGyro.CFrame = CFrame.new(rootPart.Position, rootPart.Position + moveDirection) end
		else
			bodyVelocity.Velocity = Vector3.new(0, 0, 0); if not spinEnabled then local flatLook = Vector3.new(camLook.X, 0, camLook.Z); if flatLook.Magnitude > 0 then bodyGyro.CFrame = CFrame.new(rootPart.Position, rootPart.Position + flatLook) end end
		end
	end)
end

local function toggleFly() if flyEnabled then disableFly(false) else enableFly() end end

RunService.RenderStepped:Connect(function(dt)
	local character = player.Character; if not character then return end; local humanoid = character:FindFirstChildOfClass("Humanoid"); if not humanoid then return end
	if not flyEnabled then
		if speedEnabled then humanoid.WalkSpeed = isSprinting and (customSpeed * sprintMultiplier) or customSpeed elseif isSprinting then humanoid.WalkSpeed = customSpeed * sprintMultiplier else humanoid.WalkSpeed = normalWalkSpeed end
		if jumpEnabled then humanoid.JumpPower = customJump; humanoid.UseJumpPower = true else humanoid.JumpPower = 50 end
	end
	if noclipEnabled then for _, part in ipairs(character:GetDescendants()) do if part:IsA("BasePart") then part.CanCollide = false end end end
end)

UserInputService.JumpRequest:Connect(function()
	if infJumpEnabled and not flyEnabled then local character = player.Character; if character then local h = character:FindFirstChildOfClass("Humanoid"); if h then h:ChangeState(Enum.HumanoidStateType.Jumping) end end end
end)

-- ============================================================
-- 🏗️ UI Builder
-- ============================================================

local function createMenuButton(name, text, positionY, parent)
	local button = Instance.new("TextButton"); button.Name = name .. "Btn"; button.Size = UDim2.new(1, -10, 0, 45); button.Position = UDim2.new(0, 5, 0, positionY); button.BackgroundColor3 = Color3.fromRGB(50, 50, 50); button.Text = text; button.TextColor3 = Color3.fromRGB(255, 255, 255); button.TextSize = 16; button.Font = Enum.Font.SourceSansBold; button.BorderSizePixel = 0; button.Parent = parent; Instance.new("UICorner", button).CornerRadius = UDim.new(0, 6); return button
end

local function createPage(name, titleText, parent)
	local page = Instance.new("Frame"); page.Name = name .. "Page"; page.Size = UDim2.new(1, -20, 1, -20); page.Position = UDim2.new(0, 10, 0, 10); page.BackgroundTransparency = 1; page.Visible = false; page.Parent = parent
	local title = Instance.new("TextLabel"); title.Size = UDim2.new(1, 0, 0, 35); title.BackgroundTransparency = 1; title.Text = "📌 " .. titleText; title.TextColor3 = Color3.fromRGB(255, 255, 255); title.TextSize = 24; title.TextXAlignment = Enum.TextXAlignment.Left; title.Font = Enum.Font.SourceSansBold; title.Parent = page; return page
end

local function updatePlayerList(scrollFrame)
	for _, child in ipairs(scrollFrame:GetChildren()) do if child:IsA("TextButton") then child:Destroy() end end
	local yOffset = 0
	for _, p in ipairs(Players:GetPlayers()) do
		local pButton = Instance.new("TextButton"); pButton.Size = UDim2.new(1, -15, 0, 40); pButton.Position = UDim2.new(0, 0, 0, yOffset); pButton.BackgroundColor3 = (p == player) and Color3.fromRGB(50, 90, 50) or Color3.fromRGB(40, 40, 40); pButton.Text = "  👤 " .. p.DisplayName .. " (@" .. p.Name .. ")"; pButton.TextColor3 = Color3.fromRGB(255, 255, 255); pButton.TextSize = 16; pButton.TextXAlignment = Enum.TextXAlignment.Left; pButton.Font = Enum.Font.SourceSansBold; pButton.BorderSizePixel = 0; pButton.Parent = scrollFrame; Instance.new("UICorner", pButton).CornerRadius = UDim.new(0, 4)
		if p ~= player then pButton.MouseButton1Click:Connect(function() if player.Character and p.Character then player.Character:MoveTo(p.Character.PrimaryPart.Position + Vector3.new(0, 2, 0)) end end) end
		yOffset += 45
	end
	scrollFrame.CanvasSize = UDim2.new(0, 0, 0, yOffset)
end

local function toggleMenu() if myFrame then myFrame.Visible = not myFrame.Visible; if myFrame.Visible and myFrame.ContentContainer.PlayersPage.Visible then updatePlayerList(myFrame.ContentContainer.PlayersPage.PlayerScroll) end end end

local function makeMobileDraggable(gui)
	local dragging, dragStart, startPos, hasMoved; gui.InputBegan:Connect(function(input) if input.UserInputType == Enum.UserInputType.Touch then dragging, hasMoved, dragStart, startPos = true, false, input.Position, gui.Position end end)
	UserInputService.InputChanged:Connect(function(input) if dragging and input.UserInputType == Enum.UserInputType.Touch then if (input.Position - dragStart).Magnitude > 5 then hasMoved = true end; local delta = input.Position - dragStart; gui.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y) end end)
	gui.InputEnded:Connect(function(input) if input.UserInputType == Enum.UserInputType.Touch then dragging = false; if not hasMoved then toggleMenu() end end end)
end

local function createCirclePreview(parent)
	local bg = Instance.new("Frame", parent); bg.Size = UDim2.new(0, 80, 0, 80); bg.Position = UDim2.new(0.5, -40, 1, -145); bg.BackgroundColor3 = Color3.fromRGB(45, 45, 45); bg.ClipsDescendants = true; Instance.new("UICorner", bg).CornerRadius = UDim.new(1, 0)
	local vp = Instance.new("ViewportFrame", bg); vp.Size = UDim2.new(1, 0, 1, 0); vp.BackgroundTransparency = 1; local char = player.Character or player.CharacterAdded:Wait(); char.Archivable = true; local clone = char:Clone(); clone.Parent = vp; local hrp = clone:WaitForChild("HumanoidRootPart"); local cam = Instance.new("Camera", vp); cam.CFrame = CFrame.new(hrp.Position + hrp.CFrame.LookVector * 3.5 + Vector3.new(0,1.5,0), hrp.Position + Vector3.new(0,0.5,0)); vp.CurrentCamera = cam
	RunService.RenderStepped:Connect(function(dt) if bg.Parent then hrp.CFrame = hrp.CFrame * CFrame.Angles(0, math.rad(dt * 40), 0) end end)
end

local function createKeybindSetting(labelText, defaultKeyName, posY, parent, changeCallback)
	local label = Instance.new("TextLabel", parent); label.Size = UDim2.new(0, 200, 0, 35); label.Position = UDim2.new(0, 210, 0, posY); label.Text = labelText; label.TextColor3 = Color3.fromRGB(200, 200, 200); label.BackgroundTransparency = 1; label.TextXAlignment = Enum.TextXAlignment.Left; label.Font = Enum.Font.SourceSansBold; label.TextSize = 15
	local box = Instance.new("TextBox", parent); box.Size = UDim2.new(0, 110, 0, 35); box.Position = UDim2.new(0, 340, 0, posY); box.BackgroundColor3 = Color3.fromRGB(45, 45, 45); box.Text = defaultKeyName; box.TextColor3 = Color3.fromRGB(100, 255, 255); box.Font = Enum.Font.SourceSansBold; box.TextSize = 15; Instance.new("UICorner", box).CornerRadius = UDim.new(0, 4)
	box.FocusLost:Connect(function() local inputName = trimString(box.Text); local success, newKey = pcall(function() return Enum.KeyCode[inputName] end); if success and newKey then changeCallback(newKey); box.Text = newKey.Name; box.TextColor3 = Color3.fromRGB(100, 255, 255) else box.TextColor3 = Color3.fromRGB(255, 100, 100) end end)
end

local function createUI()
	myScreenGui = Instance.new("ScreenGui", playerGui); myScreenGui.Name = UI_NAME; myScreenGui.ResetOnSpawn = false
	local mobileBtn = Instance.new("ImageButton", myScreenGui); mobileBtn.Size = UDim2.new(0, 50, 0, 50); mobileBtn.Position = UDim2.new(0, 15, 0, 15); mobileBtn.Image = "rbxassetid://88587297244568"; mobileBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 40); Instance.new("UICorner", mobileBtn).CornerRadius = UDim.new(1, 0); makeMobileDraggable(mobileBtn); mobileBtn.MouseButton1Click:Connect(function() if UserInputService.TouchEnabled and not UserInputService.MouseEnabled then return end; toggleMenu() end)
	myFrame = Instance.new("Frame", myScreenGui); myFrame.Size = UDim2.new(0, 620, 0, 380); myFrame.Position = UDim2.new(0.5, -310, 0.5, -190); myFrame.BackgroundTransparency = 1
	local actualFrame = Instance.new("Frame", myFrame); actualFrame.Size = UDim2.new(1, 0, 1, 0); actualFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 25); Instance.new("UICorner", actualFrame).CornerRadius = UDim.new(0, 10)
	local sidebar = Instance.new("Frame", actualFrame); sidebar.Size = UDim2.new(0, 150, 1, 0); sidebar.BackgroundColor3 = Color3.fromRGB(35, 35, 35); Instance.new("UICorner", sidebar).CornerRadius = UDim.new(0, 10)
	local container = Instance.new("Frame", actualFrame); container.Name = "ContentContainer"; container.Size = UDim2.new(1, -150, 1, 0); container.Position = UDim2.new(0, 150, 0, 0); container.BackgroundTransparency = 1

	local pageGfx = createPage("Gfx", "ตั้งค่าระบบเกม & ปุ่มลัด", container)
	globalToggleClayBtn = Instance.new("TextButton", pageGfx); globalToggleClayBtn.Size = UDim2.new(0, 180, 0, 40); globalToggleClayBtn.Position = UDim2.new(0, 10, 0, 55); globalToggleClayBtn.Font = Enum.Font.SourceSansBold; globalToggleClayBtn.TextSize = 16; globalToggleClayBtn.TextColor3 = Color3.fromRGB(255,255,255); Instance.new("UICorner", globalToggleClayBtn).CornerRadius = UDim.new(0, 6); globalToggleClayBtn.Text = clayGraphicsEnabled and "🟢 ภาพดินน้ำมัน: เปิด" or "🔴 ภาพดินน้ำมัน: ปิด"; globalToggleClayBtn.BackgroundColor3 = clayGraphicsEnabled and Color3.fromRGB(46,139,87) or Color3.fromRGB(178,34,34); globalToggleClayBtn.MouseButton1Click:Connect(function() setClayGraphics(not clayGraphicsEnabled) end); createKeybindSetting("⌨️ ปุ่มลัดภาพดินน้ำมัน:", clayKeybind.Name, 57, pageGfx, function(k) clayKeybind = k end)
	globalToggleFlyBtn = Instance.new("TextButton", pageGfx); globalToggleFlyBtn.Size = UDim2.new(0, 180, 0, 40); globalToggleFlyBtn.Position = UDim2.new(0, 10, 0, 110); globalToggleFlyBtn.Font = Enum.Font.SourceSansBold; globalToggleFlyBtn.TextSize = 16; globalToggleFlyBtn.TextColor3 = Color3.fromRGB(255,255,255); Instance.new("UICorner", globalToggleFlyBtn).CornerRadius = UDim.new(0, 6); globalToggleFlyBtn.Text = "🔴 ระบบบิน: ปิดอยู่"; globalToggleFlyBtn.BackgroundColor3 = Color3.fromRGB(178,34,34); globalToggleFlyBtn.MouseButton1Click:Connect(function() toggleFly() end); createKeybindSetting("⌨️ ปุ่มลัดโหมดบิน:", flyKeybind.Name, 112, pageGfx, function(k) flyKeybind = k end)
	globalToggleRamBtn = Instance.new("TextButton", pageGfx); globalToggleRamBtn.Size = UDim2.new(0, 180, 0, 40); globalToggleRamBtn.Position = UDim2.new(0, 10, 0, 165); globalToggleRamBtn.Font = Enum.Font.SourceSansBold; globalToggleRamBtn.TextSize = 16; globalToggleRamBtn.TextColor3 = Color3.fromRGB(255,255,255); Instance.new("UICorner", globalToggleRamBtn).CornerRadius = UDim.new(0, 6); globalToggleRamBtn.Text = "🔴 ระบบชนกระเด็น: ปิด"; globalToggleRamBtn.BackgroundColor3 = Color3.fromRGB(178,34,34); globalToggleRamBtn.MouseButton1Click:Connect(function() toggleRamMode() end); createKeybindSetting("⌨️ ปุ่มลัดชนกระเด็น:", ramKeybind.Name, 167, pageGfx, function(k) ramKeybind = k end)
	globalToggleEspBtn = Instance.new("TextButton", pageGfx); globalToggleEspBtn.Size = UDim2.new(0, 180, 0, 40); globalToggleEspBtn.Position = UDim2.new(0, 10, 0, 220); globalToggleEspBtn.Font = Enum.Font.SourceSansBold; globalToggleEspBtn.TextSize = 16; globalToggleEspBtn.TextColor3 = Color3.fromRGB(255,255,255); Instance.new("UICorner", globalToggleEspBtn).CornerRadius = UDim.new(0, 6); globalToggleEspBtn.Text = "🔴 ESP Tracker: ปิด"; globalToggleEspBtn.BackgroundColor3 = Color3.fromRGB(178,34,34); globalToggleEspBtn.MouseButton1Click:Connect(function() toggleEsp() end); createKeybindSetting("⌨️ ปุ่มลัด ESP:", espKeybind.Name, 222, pageGfx, function(k) espKeybind = k end)
	globalToggleSpinBtn = Instance.new("TextButton", pageGfx); globalToggleSpinBtn.Size = UDim2.new(0, 180, 0, 40); globalToggleSpinBtn.Position = UDim2.new(0, 10, 0, 275); globalToggleSpinBtn.Font = Enum.Font.SourceSansBold; globalToggleSpinBtn.TextSize = 16; globalToggleSpinBtn.TextColor3 = Color3.fromRGB(255,255,255); Instance.new("UICorner", globalToggleSpinBtn).CornerRadius = UDim.new(0, 6); globalToggleSpinBtn.Text = "🔴 ระบบหมุน SS: ปิด"; globalToggleSpinBtn.BackgroundColor3 = Color3.fromRGB(178,34,34); globalToggleSpinBtn.MouseButton1Click:Connect(function() toggleSpin() end); createKeybindSetting("⌨️ ปุ่มลัดหมุน SS:", spinKeybind.Name, 277, pageGfx, function(k) spinKeybind = k end)

	local pagePower = createPage("Power", "ปรับค่าความสามารถตัวละคร", container)
	local function createStatRow(labelName, defaultVal, posY, toggleCallback, valueCallback)
		local label = Instance.new("TextLabel", pagePower); label.Size = UDim2.new(0, 140, 0, 35); label.Position = UDim2.new(0, 10, 0, posY); label.Text = labelName; label.TextColor3 = Color3.fromRGB(200,200,200); label.BackgroundTransparency = 1; label.TextXAlignment = Enum.TextXAlignment.Left; label.Font = Enum.Font.SourceSansBold; label.TextSize = 16
		local box = Instance.new("TextBox", pagePower); box.Size = UDim2.new(0, 70, 0, 35); box.Position = UDim2.new(0, 155, 0, posY); box.BackgroundColor3 = Color3.fromRGB(45,45,45); box.Text = tostring(defaultVal); box.TextColor3 = Color3.fromRGB(255,255,255); box.Font = Enum.Font.SourceSansBold; box.TextSize = 16; Instance.new("UICorner", box).CornerRadius = UDim.new(0, 4); box.FocusLost:Connect(function() local num = tonumber(box.Text); if num then valueCallback(num) end end)
		local tBtn = Instance.new("TextButton", pagePower); tBtn.Size = UDim2.new(0, 80, 0, 35); tBtn.Position = UDim2.new(0, 235, 0, posY); tBtn.TextColor3 = Color3.fromRGB(255,255,255); tBtn.Text = "🔴 ปิด"; tBtn.BackgroundColor3 = Color3.fromRGB(178,34,34); tBtn.Font = Enum.Font.SourceSansBold; tBtn.TextSize = 16; Instance.new("UICorner", tBtn).CornerRadius = UDim.new(0, 4); local state = false; tBtn.MouseButton1Click:Connect(function() state = not state; tBtn.Text = state and "🟢 เปิด" or "🔴 ปิด"; tBtn.BackgroundColor3 = state and Color3.fromRGB(46,139,87) or Color3.fromRGB(178,34,34); toggleCallback(state) end)
	end
	createStatRow("⚡ ความเร็ว (Speed):", customSpeed, 50, function(s) speedEnabled = s end, function(v) customSpeed = v end); createKeybindSetting("⌨️ ปุ่มลัดวิ่งเร็ว:", sprintKeybind.Name, 57, pagePower, function(k) sprintKeybind = k end)
	local sprintHint = Instance.new("TextLabel", pagePower); sprintHint.Size = UDim2.new(0, 350, 0, 20); sprintHint.Position = UDim2.new(0, 10, 0, 90); sprintHint.BackgroundTransparency = 1; sprintHint.Text = "(กดค้างปุ่มด้านบน = Sprint x2 Speed, ไม่มีหมดพลัง)"; sprintHint.TextColor3 = Color3.fromRGB(180, 180, 180); sprintHint.Font = Enum.Font.SourceSansItalic; sprintHint.TextSize = 13; sprintHint.TextXAlignment = Enum.TextXAlignment.Left
	createStatRow("🚀 แรงกระโดด (Jump):", 50, 115, function(s) jumpEnabled = s end, function(v) customJump = v end)
	createStatRow("🌪️ ความเร็วหมุน (SS):", spinSpeed, 180, function(s) if s then toggleSpin() else toggleSpin() end end, function(v) spinSpeed = v end)
	local espRangeLabel = Instance.new("TextLabel", pagePower); espRangeLabel.Size = UDim2.new(0, 140, 0, 35); espRangeLabel.Position = UDim2.new(0, 10, 0, 225); espRangeLabel.Text = "👁️ ระยะ ESP (studs):"; espRangeLabel.TextColor3 = Color3.fromRGB(200,200,200); espRangeLabel.BackgroundTransparency = 1; espRangeLabel.TextXAlignment = Enum.TextXAlignment.Left; espRangeLabel.Font = Enum.Font.SourceSansBold; espRangeLabel.TextSize = 15
	local espRangeBox = Instance.new("TextBox", pagePower); espRangeBox.Size = UDim2.new(0, 70, 0, 35); espRangeBox.Position = UDim2.new(0, 155, 0, 225); espRangeBox.BackgroundColor3 = Color3.fromRGB(45,45,45); espRangeBox.Text = tostring(espMaxDistance); espRangeBox.TextColor3 = Color3.fromRGB(255,255,255); espRangeBox.Font = Enum.Font.SourceSansBold; espRangeBox.TextSize = 16; Instance.new("UICorner", espRangeBox).CornerRadius = UDim.new(0, 4); espRangeBox.FocusLost:Connect(function() local v = tonumber(espRangeBox.Text); if v and v > 0 then espMaxDistance = v end end)
	local function makeEspToggle(labelTxt, posX, posY, getter, setter) local btn = Instance.new("TextButton", pagePower); btn.Size = UDim2.new(0, 120, 0, 32); btn.Position = UDim2.new(0, posX, 0, posY); btn.Font = Enum.Font.SourceSansBold; btn.TextSize = 14; btn.TextColor3 = Color3.fromRGB(255,255,255); Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6); local function refresh() btn.Text = getter() and ("🟢 " .. labelTxt) or ("🔴 " .. labelTxt); btn.BackgroundColor3 = getter() and Color3.fromRGB(46,139,87) or Color3.fromRGB(178,34,34) end; refresh(); btn.MouseButton1Click:Connect(function() setter(not getter()); refresh() end) end
	makeEspToggle("แสดงเส้น",  10,  270, function() return espShowLines    end, function(v) espShowLines    = v end); makeEspToggle("แสดงชื่อ",  140, 270, function() return espShowNames    end, function(v) espShowNames    = v end); makeEspToggle("แสดงระยะ",  270, 270, function() return espShowDistance end, function(v) espShowDistance = v end)
	local toggleInfJump = Instance.new("TextButton", pagePower); toggleInfJump.Size = UDim2.new(0, 150, 0, 40); toggleInfJump.Position = UDim2.new(0, 10, 0, 318); toggleInfJump.Font = Enum.Font.SourceSansBold; toggleInfJump.TextSize = 16; toggleInfJump.TextColor3 = Color3.fromRGB(255,255,255); Instance.new("UICorner", toggleInfJump).CornerRadius = UDim.new(0, 6); local function updateInfJump() toggleInfJump.Text = infJumpEnabled and "🟢 โดดไม่จำกัด" or "🔴 โดดไม่จำกัด"; toggleInfJump.BackgroundColor3 = infJumpEnabled and Color3.fromRGB(46,139,87) or Color3.fromRGB(178,34,34) end; updateInfJump(); toggleInfJump.MouseButton1Click:Connect(function() infJumpEnabled = not infJumpEnabled; updateInfJump() end)
	local toggleNoclip = Instance.new("TextButton", pagePower); toggleNoclip.Size = UDim2.new(0, 150, 0, 40); toggleNoclip.Position = UDim2.new(0, 170, 0, 318); toggleNoclip.Font = Enum.Font.SourceSansBold; toggleNoclip.TextSize = 16; toggleNoclip.TextColor3 = Color3.fromRGB(255,255,255); Instance.new("UICorner", toggleNoclip).CornerRadius = UDim.new(0, 6); local function updateNoclip() toggleNoclip.Text = noclipEnabled and "🟢 เดินทะลุ (On)" or "🔴 เดินทะลุ (Off)"; toggleNoclip.BackgroundColor3 = noclipEnabled and Color3.fromRGB(46,139,87) or Color3.fromRGB(178,34,34) end; updateNoclip(); toggleNoclip.MouseButton1Click:Connect(function() noclipEnabled = not noclipEnabled; updateNoclip() end)

	-- 🎯 หน้า x-sd (ล็อคหัว)
	local pageXsd = createPage("Xsd", "x-sd", container)

	globalToggleHeadLockBtn = Instance.new("TextButton", pageXsd)
	globalToggleHeadLockBtn.Size = UDim2.new(0, 180, 0, 40)
	globalToggleHeadLockBtn.Position = UDim2.new(0, 10, 0, 55)
	globalToggleHeadLockBtn.Font = Enum.Font.SourceSansBold
	globalToggleHeadLockBtn.TextSize = 16
	globalToggleHeadLockBtn.TextColor3 = Color3.fromRGB(255,255,255)
	Instance.new("UICorner", globalToggleHeadLockBtn).CornerRadius = UDim.new(0, 6)
	globalToggleHeadLockBtn.Text = "🔴 ระบบล็อคหัว: ปิด"
	globalToggleHeadLockBtn.BackgroundColor3 = Color3.fromRGB(178,34,34)
	globalToggleHeadLockBtn.MouseButton1Click:Connect(function() toggleHeadLockSystem() end)
	createKeybindSetting("⌨️ ปุ่มล็อค/ยกเลิก:", headLockKeybind.Name, 57, pageXsd, function(k) headLockKeybind = k end)

	headLockTargetLabel = Instance.new("TextLabel", pageXsd)
	headLockTargetLabel.Size = UDim2.new(0, 300, 0, 35)
	headLockTargetLabel.Position = UDim2.new(0, 10, 0, 110)
	headLockTargetLabel.BackgroundTransparency = 1
	headLockTargetLabel.TextColor3 = Color3.fromRGB(255, 100, 100)
	headLockTargetLabel.Font = Enum.Font.SourceSansBold
	headLockTargetLabel.TextSize = 18
	headLockTargetLabel.TextXAlignment = Enum.TextXAlignment.Left
	headLockTargetLabel.Text = "🎯 ล็อคเป้า: ไม่มี"

	local xsdHint = Instance.new("TextLabel", pageXsd)
	xsdHint.Size = UDim2.new(0, 400, 0, 50)
	xsdHint.Position = UDim2.new(0, 10, 0, 155)
	xsdHint.BackgroundTransparency = 1
	xsdHint.Text = "วิธีใช้: เปิดระบบล็อคหัวก่อน -> จากนั้นเล็งไปที่ศัตรู -> กด E เพื่อล็อค\nกด E ซ้ำอีกครั้งเพื่อยกเลิกการล็อค"
	xsdHint.TextColor3 = Color3.fromRGB(200, 200, 200)
	xsdHint.Font = Enum.Font.SourceSansItalic
	xsdHint.TextSize = 15
	xsdHint.TextXAlignment = Enum.TextXAlignment.Left
	xsdHint.TextWrapped = true

	local pagePlayers = createPage("Players", "จัดการผู้เล่น (วาร์ป)", container); local scroll = Instance.new("ScrollingFrame", pagePlayers); scroll.Name = "PlayerScroll"; scroll.Size = UDim2.new(1, 0, 1, -50); scroll.Position = UDim2.new(0, 0, 0, 50); scroll.BackgroundTransparency = 1; scroll.ScrollBarThickness = 5

	local btnGfx    = createMenuButton("Gfx",    "⚙️ ตั้งค่าระบบ",  15,  sidebar)
	local btnPower  = createMenuButton("Power",  "📌 ความสามารถ",   65,  sidebar)
	local btnPlayer = createMenuButton("Player", "👥 ดูผู้เล่น",    115, sidebar)
	local btnXsd    = createMenuButton("Xsd",    "🎯 x-sd",         165, sidebar) -- เพิ่มปุ่มเมนู x-sd

	btnGfx.MouseButton1Click:Connect(function() pageGfx.Visible = true; pagePower.Visible = false; pagePlayers.Visible = false; pageXsd.Visible = false end)
	btnPower.MouseButton1Click:Connect(function() pageGfx.Visible = false; pagePower.Visible = true; pagePlayers.Visible = false; pageXsd.Visible = false end)
	btnPlayer.MouseButton1Click:Connect(function() pageGfx.Visible = false; pagePower.Visible = false; pagePlayers.Visible = true; pageXsd.Visible = false; updatePlayerList(scroll) end)
	btnXsd.MouseButton1Click:Connect(function() pageGfx.Visible = false; pagePower.Visible = false; pagePlayers.Visible = false; pageXsd.Visible = true end)

	pageGfx.Visible = true

	local fps = Instance.new("TextLabel", sidebar); fps.Size = UDim2.new(1, 0, 0, 20); fps.Position = UDim2.new(0, 0, 1, -25); fps.TextColor3 = Color3.fromRGB(100,255,255); fps.BackgroundTransparency = 1; fps.Font = Enum.Font.SourceSansBold; fps.TextSize = 14
	local frames, lastTime = 0, os.clock(); RunService.RenderStepped:Connect(function() frames += 1; if os.clock() - lastTime >= 1 then fps.Text = "📈 FPS: " .. frames; frames, lastTime = 0, os.clock() end end)
	createCirclePreview(sidebar)
end

-- ============================================================
-- ⌨️ Keybind handler & Sprint Logic
-- ============================================================
UserInputService.InputBegan:Connect(function(input, gpe)
	if gpe then return end
	if input.KeyCode == Enum.KeyCode.Q then if not playerGui:FindFirstChild(UI_NAME) then createUI() else toggleMenu() end end
	if input.KeyCode == flyKeybind  then toggleFly() end
	if input.KeyCode == clayKeybind then setClayGraphics(not clayGraphicsEnabled) end
	if input.KeyCode == ramKeybind  then toggleRamMode() end
	if input.KeyCode == espKeybind  then toggleEsp() end
	if input.KeyCode == spinKeybind then toggleSpin() end

	-- 🎯 กดปุ่มล็อคหัว (E)
	if input.KeyCode == headLockKeybind then handleHeadLockKey() end

	if input.KeyCode == sprintKeybind then isSprinting = true end
end)

UserInputService.InputEnded:Connect(function(input, gpe)
	if input.KeyCode == sprintKeybind then isSprinting = false end
end)

if not playerGui:FindFirstChild(UI_NAME) then createUI() end
