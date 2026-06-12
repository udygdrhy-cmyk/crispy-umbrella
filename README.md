local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

local oldGui = player.PlayerGui:FindFirstChild("FlySystemGUI")
if oldGui then oldGui:Destroy() end

-- ตัวแปรสถานะ
local flying = false
local runningFast = false
local jumpingHigh = false
local espPlayersEnabled = false
local espNPCsEnabled = false
local noclipEnabled = false
local spinningEnabled = false
local spinSpeed = 10
local bodyVelocity = nil
local bodyGyro = nil
local flySpeed = 80
local runSpeed = 80
local jumpPower = 150
local espObjects = {} 
local npcCache = {} 
local holdingUp = false 
local holdingDown = false 
local currentAnimTrack = nil -- เพิ่มตัวแปรเก็บแอนิเมชันที่กำลังเล่นอยู่

-- ==========================================
-- สร้าง GUI
-- ==========================================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "FlySystemGUI"
screenGui.ResetOnSpawn = false
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
screenGui.Parent = player.PlayerGui

local statusLabel = Instance.new("TextLabel")
statusLabel.Size = UDim2.new(0, 150, 0, 30)
statusLabel.Position = UDim2.new(0, -160, 0.5, 0)
statusLabel.BackgroundTransparency = 1
statusLabel.TextColor3 = Color3.fromRGB(0, 255, 120)
statusLabel.Text = "✈ กำลังบินอยู่"
statusLabel.Font = Enum.Font.GothamBold
statusLabel.TextSize = 16
statusLabel.TextXAlignment = Enum.TextXAlignment.Left
statusLabel.Parent = screenGui

local notifyLabel = Instance.new("TextLabel")
notifyLabel.Size = UDim2.new(0, 280, 0, 50)
notifyLabel.Position = UDim2.new(0.5, 0, 1, 60)
notifyLabel.AnchorPoint = Vector2.new(0.5, 1)
notifyLabel.BackgroundColor3 = Color3.fromRGB(40, 150, 40)
notifyLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
notifyLabel.Text = "✅ เปิดใช้งานระบบบิน"
notifyLabel.Font = Enum.Font.GothamBold
notifyLabel.TextSize = 16
notifyLabel.Parent = screenGui
Instance.new("UICorner", notifyLabel).CornerRadius = UDim.new(0, 8)
local notifyStroke = Instance.new("UIStroke")
notifyStroke.Color = Color3.fromRGB(100, 255, 100)
notifyStroke.Thickness = 1.5
notifyStroke.Parent = notifyLabel

local mobileBtn = Instance.new("ImageButton")
mobileBtn.Size = UDim2.new(0, 55, 0, 55) 
mobileBtn.Position = UDim2.new(0, 20, 0, 20) 
mobileBtn.BackgroundTransparency = 0.3
mobileBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
mobileBtn.Image = "rbxassetid://133493400381536" 
mobileBtn.Visible = true 
mobileBtn.Parent = screenGui
Instance.new("UICorner", mobileBtn).CornerRadius = UDim.new(0.5, 0)

local btnUp = Instance.new("TextButton") btnUp.Size = UDim2.new(0, 60, 0, 60) btnUp.Position = UDim2.new(1, -80, 1, -220) btnUp.BackgroundColor3 = Color3.fromRGB(50, 50, 50) btnUp.BackgroundTransparency = 0.5 btnUp.Text = "⬆" btnUp.TextColor3 = Color3.fromRGB(200, 200, 200) btnUp.Font = Enum.Font.GothamBold btnUp.TextSize = 30 btnUp.Visible = false btnUp.Parent = screenGui Instance.new("UICorner", btnUp).CornerRadius = UDim.new(0.5, 0)
local btnDown = Instance.new("TextButton") btnDown.Size = UDim2.new(0, 60, 0, 60) btnDown.Position = UDim2.new(1, -80, 1, -150) btnDown.BackgroundColor3 = Color3.fromRGB(50, 50, 50) btnDown.BackgroundTransparency = 0.5 btnDown.Text = "⬇" btnDown.TextColor3 = Color3.fromRGB(200, 200, 200) btnDown.Font = Enum.Font.GothamBold btnDown.TextSize = 30 btnDown.Visible = false btnDown.Parent = screenGui Instance.new("UICorner", btnDown).CornerRadius = UDim.new(0.5, 0)

btnUp.InputBegan:Connect(function(input) if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then holdingUp = true end end)
btnUp.InputEnded:Connect(function(input) if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then holdingUp = false end end)
btnDown.InputBegan:Connect(function(input) if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then holdingDown = true end end)
btnDown.InputEnded:Connect(function(input) if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then holdingDown = false end end)

-- ==========================================
-- หน้าต่าง UI หลัก
-- ==========================================
local togglePanel = Instance.new("Frame")
togglePanel.Size = UDim2.new(0, 300, 0, 500) 
togglePanel.Position = UDim2.new(0.5, 0, 0.5, 0)
togglePanel.AnchorPoint = Vector2.new(0.5, 0.5)
togglePanel.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
togglePanel.Visible = false
togglePanel.Parent = screenGui
Instance.new("UICorner", togglePanel).CornerRadius = UDim.new(0, 10)

local tabFrame = Instance.new("Frame") tabFrame.Size = UDim2.new(1, 0, 0, 35) tabFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 15) tabFrame.Parent = togglePanel Instance.new("UICorner", tabFrame).CornerRadius = UDim.new(0, 10)
local tabBtn1 = Instance.new("TextButton") tabBtn1.Size = UDim2.new(0.33, 0, 1, 0) tabBtn1.Position = UDim2.new(0,0,0,0) tabBtn1.BackgroundColor3 = Color3.fromRGB(0, 150, 80) tabBtn1.TextColor3 = Color3.fromRGB(255, 255, 255) tabBtn1.Text = "บิน/เคลื่อนที่" tabBtn1.Font = Enum.Font.GothamBold tabBtn1.TextSize = 12 tabBtn1.Parent = tabFrame
local tabBtn2 = Instance.new("TextButton") tabBtn2.Size = UDim2.new(0.34, 0, 1, 0) tabBtn2.Position = UDim2.new(0.33,0,0,0) tabBtn2.BackgroundColor3 = Color3.fromRGB(40, 40, 40) tabBtn2.TextColor3 = Color3.fromRGB(200, 200, 200) tabBtn2.Text = "ตำแหน่ง" tabBtn2.Font = Enum.Font.GothamBold tabBtn2.TextSize = 12 tabBtn2.Parent = tabFrame
local tabBtn3 = Instance.new("TextButton") tabBtn3.Size = UDim2.new(0.33, 0, 1, 0) tabBtn3.Position = UDim2.new(0.67,0,0,0) tabBtn3.BackgroundColor3 = Color3.fromRGB(40, 40, 40) tabBtn3.TextColor3 = Color3.fromRGB(200, 200, 200) tabBtn3.Text = "ตั้งค่า" tabBtn3.Font = Enum.Font.GothamBold tabBtn3.TextSize = 12 tabBtn3.Parent = tabFrame

local contentArea = Instance.new("Frame") contentArea.Size = UDim2.new(1, 0, 1, -35) contentArea.Position = UDim2.new(0, 0, 0, 35) contentArea.BackgroundTransparency = 1 contentArea.ClipsDescendants = true contentArea.Parent = togglePanel

-- ==========================================
-- ฟังก์ชันสร้าง UI
-- ==========================================
local function createSwitch(parent, posY)
	local bg = Instance.new("TextButton") bg.Size = UDim2.new(0, 50, 0, 26) bg.Position = UDim2.new(1, -15, 0, posY) bg.AnchorPoint = Vector2.new(1, 0) bg.BackgroundColor3 = Color3.fromRGB(80, 80, 80) bg.Text = "" bg.Parent = parent Instance.new("UICorner", bg).CornerRadius = UDim.new(1, 0)
	local circle = Instance.new("Frame") circle.Size = UDim2.new(0, 22, 0, 22) circle.Position = UDim2.new(0, 2, 0.5, 0) circle.AnchorPoint = Vector2.new(0, 0.5) circle.BackgroundColor3 = Color3.fromRGB(255, 255, 255) circle.Parent = bg Instance.new("UICorner", circle).CornerRadius = UDim.new(1, 0)
	return bg, circle
end

local function createToggleAndSlider(parent, posY, name, defaultVal, minVal, maxVal)
	local text = Instance.new("TextLabel") text.Size = UDim2.new(0, 150, 0, 26) text.Position = UDim2.new(0, 15, 0, posY) text.BackgroundTransparency = 1 text.TextColor3 = Color3.fromRGB(255, 255, 255) text.Text = name text.Font = Enum.Font.GothamBold text.TextSize = 14 text.TextXAlignment = Enum.TextXAlignment.Left text.Parent = parent
	local bg = Instance.new("TextButton") bg.Size = UDim2.new(0, 50, 0, 26) bg.Position = UDim2.new(1, -15, 0, posY) bg.AnchorPoint = Vector2.new(1, 0) bg.BackgroundColor3 = Color3.fromRGB(80, 80, 80) bg.Text = "" bg.Parent = parent Instance.new("UICorner", bg).CornerRadius = UDim.new(1, 0)
	local circle = Instance.new("Frame") circle.Size = UDim2.new(0, 22, 0, 22) circle.Position = UDim2.new(0, 2, 0.5, 0) circle.AnchorPoint = Vector2.new(0, 0.5) circle.BackgroundColor3 = Color3.fromRGB(255, 255, 255) circle.Parent = bg Instance.new("UICorner", circle).CornerRadius = UDim.new(1, 0)
	local valText = Instance.new("TextLabel") valText.Size = UDim2.new(0, 100, 0, 20) valText.Position = UDim2.new(0, 15, 0, posY + 28) valText.BackgroundTransparency = 1 valText.TextColor3 = Color3.fromRGB(200, 200, 200) valText.Text = tostring(defaultVal) valText.Font = Enum.Font.Gotham valText.TextSize = 12 valText.TextXAlignment = Enum.TextXAlignment.Left valText.Parent = parent
	local sliderBg = Instance.new("Frame") sliderBg.Size = UDim2.new(0, 270, 0, 8) sliderBg.Position = UDim2.new(0, 15, 0, posY + 48) sliderBg.BackgroundColor3 = Color3.fromRGB(50, 50, 50) sliderBg.Parent = parent Instance.new("UICorner", sliderBg).CornerRadius = UDim.new(1, 0)
	local sliderFill = Instance.new("Frame") local defaultRatio = (defaultVal - minVal) / (maxVal - minVal) sliderFill.Size = UDim2.new(defaultRatio, 0, 1, 0) sliderFill.BackgroundColor3 = Color3.fromRGB(0, 200, 100) sliderFill.Parent = sliderBg Instance.new("UICorner", sliderFill).CornerRadius = UDim.new(1, 0)
	local sliderBtn = Instance.new("TextButton") sliderBtn.Size = UDim2.new(0, 14, 0, 14) sliderBtn.Position = UDim2.new(defaultRatio, -7, 0.5, 0) sliderBtn.AnchorPoint = Vector2.new(0, 0.5) sliderBtn.BackgroundColor3 = Color3.fromRGB(255, 255, 255) sliderBtn.Text = "" sliderBtn.Parent = sliderBg Instance.new("UICorner", sliderBtn).CornerRadius = UDim.new(1, 0)
	return bg, circle, valText, sliderBg, sliderFill, sliderBtn
end

-- == แท็บ 1 ==
local page1 = Instance.new("Frame") page1.Size = UDim2.new(1, 0, 1, 0) page1.BackgroundTransparency = 1 page1.Visible = true page1.Parent = contentArea
local switchBg1, switchCircle1, valText1, sliderBg1, sliderFill1, sliderBtn1 = createToggleAndSlider(page1, 10, "ระบบบิน", flySpeed, 20, 200)
local switchBg2, switchCircle2, valText2, sliderBg2, sliderFill2, sliderBtn2 = createToggleAndSlider(page1, 85, "วิ่งไว (Speed)", runSpeed, 20, 200)
local switchBg3, switchCircle3, valText3, sliderBg3, sliderFill3, sliderBtn3 = createToggleAndSlider(page1, 160, "กระโดดสูง (Jump)", jumpPower, 50, 500)

-- == แท็บ 2: ตำแหน่ง (เปลี่ยนเป็น Animation ID) ==
local page2 = Instance.new("Frame") page2.Size = UDim2.new(1, 0, 1, 0) page2.BackgroundTransparency = 1 page2.Visible = false page2.Parent = contentArea
local switchBg4, switchCircle4 = createSwitch(page2, 20)
local switchBg5, switchCircle5 = createSwitch(page2, 60)
local espPText = Instance.new("TextLabel") espPText.Size = UDim2.new(0, 150, 0, 26) espPText.Position = UDim2.new(0, 15, 0, 20) espPText.BackgroundTransparency = 1 espPText.TextColor3 = Color3.fromRGB(255, 255, 255) espPText.Text = "ผู้เล่น (Players)" espPText.Font = Enum.Font.GothamBold espPText.TextSize = 14 espPText.TextXAlignment = Enum.TextXAlignment.Left espPText.Parent = page2
local espNPCText = Instance.new("TextLabel") espNPCText.Size = UDim2.new(0, 150, 0, 26) espNPCText.Position = UDim2.new(0, 15, 0, 60) espNPCText.BackgroundTransparency = 1 espNPCText.TextColor3 = Color3.fromRGB(255, 255, 255) espNPCText.Text = "มอนสเตอร์ (NPCs)" espNPCText.Font = Enum.Font.GothamBold espNPCText.TextSize = 14 espNPCText.TextXAlignment = Enum.TextXAlignment.Left espNPCText.Parent = page2

-- Noclip Switch
local switchBg6, switchCircle6 = createSwitch(page2, 105)
local noclipText = Instance.new("TextLabel") noclipText.Size = UDim2.new(0, 150, 0, 26) noclipText.Position = UDim2.new(0, 15, 0, 105) noclipText.BackgroundTransparency = 1 noclipText.TextColor3 = Color3.fromRGB(255, 255, 255) noclipText.Text = "ทะลุบล็อก (Noclip)" noclipText.Font = Enum.Font.GothamBold noclipText.TextSize = 14 noclipText.TextXAlignment = Enum.TextXAlignment.Left noclipText.Parent = page2

-- Spin Toggle + Slider
local switchBg7, switchCircle7, valText7, sliderBg7, sliderFill7, sliderBtn7 = createToggleAndSlider(page2, 145, "ระบบหมุน (Spin)", spinSpeed, 1, 50)

-- ช่องค้นหาวัตถุ/วาร์ปหาผู้เล่น
local warpHint = Instance.new("TextLabel") warpHint.Size = UDim2.new(0, 270, 0, 20) warpHint.Position = UDim2.new(0, 15, 0, 240) warpHint.BackgroundTransparency = 1 warpHint.TextColor3 = Color3.fromRGB(180, 180, 180) warpHint.Text = "กดชื่อด้านขวาเพื่อวาร์ปหาผู้เล่น" warpHint.Font = Enum.Font.Gotham warpHint.TextSize = 12 warpHint.TextXAlignment = Enum.TextXAlignment.Left warpHint.Parent = page2
local searchLabel = Instance.new("TextLabel") searchLabel.Size = UDim2.new(0, 200, 0, 20) searchLabel.Position = UDim2.new(0, 15, 0, 265) searchLabel.BackgroundTransparency = 1 searchLabel.TextColor3 = Color3.fromRGB(255, 255, 255) searchLabel.Text = "ค้นหาวัตถุ/ชื่อ แล้ววาร์ปไปหา:" searchLabel.Font = Enum.Font.GothamBold searchLabel.TextSize = 13 searchLabel.TextXAlignment = Enum.TextXAlignment.Left searchLabel.Parent = page2
local searchBox = Instance.new("TextBox") searchBox.Size = UDim2.new(0, 165, 0, 30) searchBox.Position = UDim2.new(0, 15, 0, 288) searchBox.BackgroundColor3 = Color3.fromRGB(40, 40, 40) searchBox.TextColor3 = Color3.fromRGB(255, 255, 255) searchBox.PlaceholderText = "พิมพ์ชื่อมา..." searchBox.Font = Enum.Font.Gotham searchBox.TextSize = 14 searchBox.ClearTextOnFocus = false searchBox.Parent = page2 Instance.new("UICorner", searchBox).CornerRadius = UDim.new(0, 6)
local searchBtn = Instance.new("TextButton") searchBtn.Size = UDim2.new(0, 95, 0, 30) searchBtn.Position = UDim2.new(0, 190, 0, 288) searchBtn.BackgroundColor3 = Color3.fromRGB(0, 100, 200) searchBtn.TextColor3 = Color3.fromRGB(255, 255, 255) searchBtn.Text = "วาร์ปไป" searchBtn.Font = Enum.Font.GothamBold searchBtn.TextSize = 13 searchBtn.Parent = page2 Instance.new("UICorner", searchBtn).CornerRadius = UDim.new(0, 6)

-- 🔥 ส่วน Animation ID (แก้ไขใหม่) 🔥
local animIdLabel = Instance.new("TextLabel")
animIdLabel.Size = UDim2.new(0, 200, 0, 20)
animIdLabel.Position = UDim2.new(0, 15, 0, 335)
animIdLabel.BackgroundTransparency = 1
animIdLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
animIdLabel.Text = "ใส่ ID แอนิเมชัน (Animation):"
animIdLabel.Font = Enum.Font.GothamBold
animIdLabel.TextSize = 13
animIdLabel.TextXAlignment = Enum.TextXAlignment.Left
animIdLabel.Parent = page2

local animIdBox = Instance.new("TextBox")
animIdBox.Size = UDim2.new(0, 165, 0, 30)
animIdBox.Position = UDim2.new(0, 15, 0, 358)
animIdBox.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
animIdBox.TextColor3 = Color3.fromRGB(255, 255, 255)
animIdBox.PlaceholderText = "ใส่ Animation ID..."
animIdBox.Font = Enum.Font.Gotham
animIdBox.TextSize = 14
animIdBox.ClearTextOnFocus = false
animIdBox.Parent = page2 
Instance.new("UICorner", animIdBox).CornerRadius = UDim.new(0, 6)

local playAnimBtn = Instance.new("TextButton")
playAnimBtn.Size = UDim2.new(0, 45, 0, 30)
playAnimBtn.Position = UDim2.new(0, 190, 0, 358)
playAnimBtn.BackgroundColor3 = Color3.fromRGB(0, 150, 80)
playAnimBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
playAnimBtn.Text = "เล่น"
playAnimBtn.Font = Enum.Font.GothamBold
playAnimBtn.TextSize = 13
playAnimBtn.Parent = page2 
Instance.new("UICorner", playAnimBtn).CornerRadius = UDim.new(0, 6)

local stopAnimBtn = Instance.new("TextButton")
stopAnimBtn.Size = UDim2.new(0, 45, 0, 30)
stopAnimBtn.Position = UDim2.new(0, 240, 0, 358)
stopAnimBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
stopAnimBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
stopAnimBtn.Text = "หยุด"
stopAnimBtn.Font = Enum.Font.GothamBold
stopAnimBtn.TextSize = 13
stopAnimBtn.Parent = page2 
Instance.new("UICorner", stopAnimBtn).CornerRadius = UDim.new(0, 6)

-- ฟังก์ชันเล่นแอนิเมชัน
local function playAnimation()
	local idString = animIdBox.Text
	local animId = tonumber(idString)

	if not animId or animId == 0 then
		showNotification("⚠ กรุณาใส่ตัวเลข Animation ID ให้ถูกต้อง", Color3.fromRGB(200, 50, 50), Color3.fromRGB(255, 100, 100))
		return
	end

	local char = player.Character
	local humanoid = char and char:FindFirstChildOfClass("Humanoid")
	if not humanoid then return end

	-- หยุดแอนิเมชันเก่าถ้ามี
	if currentAnimTrack and currentAnimTrack.IsPlaying then
		currentAnimTrack:Stop()
	end

	local anim = Instance.new("Animation")
	anim.AnimationId = "rbxassetid://" .. idString

	-- ใช้ pcall เพื่อดักจับ error ในกรณีที่ ID ไม่ถูกต้องหรือโดนบล็อก
	local success, track = pcall(function()
		return humanoid:LoadAnimation(anim)
	end)

	if success and track then
		currentAnimTrack = track
		currentAnimTrack:Play()
		showNotification("🕺 กำลังเล่นแอนิเมชัน: " .. idString, Color3.fromRGB(40, 150, 40), Color3.fromRGB(100, 255, 100))
	else
		showNotification("⚠ ไม่สามารถเล่น ID นี้ได้ (ผิด ID หรือถูกบล็อก)", Color3.fromRGB(200, 50, 50), Color3.fromRGB(255, 100, 100))
	end
end

-- ฟังก์ชันหยุดแอนิเมชัน
local function stopAnimation()
	if currentAnimTrack and currentAnimTrack.IsPlaying then
		currentAnimTrack:Stop()
		showNotification("⏹ หยุดเล่นแอนิเมชันแล้ว", Color3.fromRGB(200, 40, 40), Color3.fromRGB(255, 100, 100))
	end
end

playAnimBtn.MouseButton1Click:Connect(playAnimation)
stopAnimBtn.MouseButton1Click:Connect(stopAnimation)
animIdBox.FocusLost:Connect(function(enterPressed) 
	if enterPressed then playAnimation() end 
end)
-- 🔥 จบส่วน Animation ID 🔥

local playerListFrame = Instance.new("Frame") playerListFrame.Size = UDim2.new(0, 180, 0, 350) playerListFrame.Position = UDim2.new(1, -15, 0.5, 0) playerListFrame.AnchorPoint = Vector2.new(1, 0.5) playerListFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 25) playerListFrame.Visible = false playerListFrame.Parent = screenGui Instance.new("UICorner", playerListFrame).CornerRadius = UDim.new(0, 8) local plStroke = Instance.new("UIStroke") plStroke.Color = Color3.fromRGB(60, 60, 60) plStroke.Thickness = 1 plStroke.Parent = playerListFrame
local plTitle = Instance.new("TextLabel") plTitle.Size = UDim2.new(1, 0, 0, 30) plTitle.BackgroundColor3 = Color3.fromRGB(15, 15, 15) plTitle.TextColor3 = Color3.fromRGB(255, 255, 255) plTitle.Text = "วาร์ปหาผู้เล่น" plTitle.Font = Enum.Font.GothamBold plTitle.TextSize = 14 plTitle.Parent = playerListFrame Instance.new("UICorner", plTitle).CornerRadius = UDim.new(0, 8)
local plScroll = Instance.new("ScrollingFrame") plScroll.Size = UDim2.new(1, -10, 1, -40) plScroll.Position = UDim2.new(0, 5, 0, 35) plScroll.BackgroundTransparency = 1 plScroll.ScrollBarThickness = 4 plScroll.Parent = playerListFrame local plLayout = Instance.new("UIListLayout") plLayout.SortOrder = Enum.SortOrder.LayoutOrder plLayout.Padding = UDim.new(0, 4) plLayout.Parent = plScroll

-- == แท็บ 3 ==
local page3 = Instance.new("Frame") page3.Size = UDim2.new(1, 0, 1, 0) page3.BackgroundTransparency = 1 page3.Visible = false page3.Parent = contentArea
local resetBtn = Instance.new("TextButton") resetBtn.Size = UDim2.new(0, 270, 0, 40) resetBtn.Position = UDim2.new(0.5, 0, 0, 30) resetBtn.AnchorPoint = Vector2.new(0.5, 0) resetBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50) resetBtn.TextColor3 = Color3.fromRGB(255, 255, 255) resetBtn.Text = "รีเซ็ตตำแหน่งปุ่มเปิดเมนู" resetBtn.Font = Enum.Font.GothamBold resetBtn.TextSize = 14 resetBtn.Parent = page3 Instance.new("UICorner", resetBtn).CornerRadius = UDim.new(0, 6)
local infoText = Instance.new("TextLabel") infoText.Size = UDim2.new(0, 270, 0, 100) infoText.Position = UDim2.new(0.5, 0, 0, 90) infoText.AnchorPoint = Vector2.new(0.5, 0) infoText.BackgroundTransparency = 1 infoText.TextColor3 = Color3.fromRGB(180, 180, 180) infoText.Text = "ปุ่มลัดคีย์บอร์ด:\n[G] เปิด/ปิดเมนู\n[CapsLock] เปิด/ปิดระบบบิน\n[Z] เปิด/ปิดระบบตำแหน่ง" infoText.Font = Enum.Font.Gotham infoText.TextSize = 13 infoText.Parent = page3

-- ==========================================
-- Animation Logic
-- ==========================================
local tweenInfoSwitch = TweenInfo.new(0.25, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
local function playSwitchOn(circle, bg) TweenService:Create(circle, tweenInfoSwitch, {Position = UDim2.new(0, 26, 0.5, 0)}):Play() TweenService:Create(bg, tweenInfoSwitch, {BackgroundColor3 = Color3.fromRGB(0, 200, 100)}):Play() end
local function playSwitchOff(circle, bg) TweenService:Create(circle, tweenInfoSwitch, {Position = UDim2.new(0, 2, 0.5, 0)}):Play() TweenService:Create(bg, tweenInfoSwitch, {BackgroundColor3 = Color3.fromRGB(80, 80, 80)}):Play() end

local tweenInfoStatus = TweenInfo.new(0.5, Enum.EasingStyle.Back, Enum.EasingDirection.Out)
local showStatusTween = TweenService:Create(statusLabel, tweenInfoStatus, {Position = UDim2.new(0, 15, 0.5, 0)})
local hideStatusTween = TweenService:Create(statusLabel, tweenInfoStatus, {Position = UDim2.new(0, -160, 0.5, 0)})

local tweenInfoNotify = TweenInfo.new(0.5, Enum.EasingStyle.Back, Enum.EasingDirection.Out)
local showNotifyTween = TweenService:Create(notifyLabel, tweenInfoNotify, {Position = UDim2.new(0.5, 0, 1, -20)})
local hideNotifyTween = TweenService:Create(notifyLabel, tweenInfoNotify, {Position = UDim2.new(0.5, 0, 1, 60)})

local tweenInfoPanel = TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out)
local panelShowTween = TweenService:Create(togglePanel, tweenInfoPanel, {Size = UDim2.new(0, 300, 0, 500)})
local panelHideTween = TweenService:Create(togglePanel, tweenInfoPanel, {Size = UDim2.new(0, 0, 0, 0)})

local isNotifying = false
local function showNotification(text, bgColor, strokeColor)
	if isNotifying then return end
	isNotifying = true
	notifyLabel.Text = text; notifyLabel.BackgroundColor3 = bgColor; notifyStroke.Color = strokeColor
	showNotifyTween:Play()
	task.delay(2, function() hideNotifyTween:Play() task.wait(0.5) isNotifying = false end)
end

local isPanelOpen = false
function togglePanelUI()
	if not isPanelOpen then togglePanel.Visible = true; togglePanel.Size = UDim2.new(0, 0, 0, 0); panelShowTween:Play(); isPanelOpen = true
	else panelHideTween:Play(); playerListFrame.Visible = false; task.wait(0.3); togglePanel.Visible = false; isPanelOpen = false end
end

local function switchTab(tabNum)
	page1.Visible = tabNum == 1; page2.Visible = tabNum == 2; page3.Visible = tabNum == 3
	playerListFrame.Visible = (tabNum == 2)
	if tabNum == 2 then refreshPlayerList() end
	tabBtn1.BackgroundColor3 = tabNum == 1 and Color3.fromRGB(0, 150, 80) or Color3.fromRGB(40, 40, 40)
	tabBtn2.BackgroundColor3 = tabNum == 2 and Color3.fromRGB(0, 150, 80) or Color3.fromRGB(40, 40, 40)
	tabBtn3.BackgroundColor3 = tabNum == 3 and Color3.fromRGB(0, 150, 80) or Color3.fromRGB(40, 40, 40)
end

local dragging, dragInput, dragStart, startPos
local isDragging = false
mobileBtn.InputBegan:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dragging = true; dragStart = input.Position; startPos = mobileBtn.Position; isDragging = false input.Changed:Connect(function() if input.UserInputState == Enum.UserInputState.End then dragging = false end end) end end)
mobileBtn.InputChanged:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then dragInput = input end end)
UserInputService.InputChanged:Connect(function(input) if input == dragInput and dragging then local delta = input.Position - dragStart if delta.Magnitude > 5 then isDragging = true end mobileBtn.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y) end end)
mobileBtn.InputEnded:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then if not isDragging then togglePanelUI() end end end)

-- ==========================================
-- ระบบบิน / วิ่ง / กระโดด
-- ==========================================
local function startFly() if flying then return end local character = player.Character if not character then return end local hrp = character:FindFirstChild("HumanoidRootPart") local humanoid = character:FindFirstChild("Humanoid") if not hrp or not humanoid then return end flying = true humanoid.PlatformStand = false humanoid.AutoRotate = false humanoid.WalkSpeed = 0 bodyVelocity = Instance.new("BodyVelocity") bodyVelocity.MaxForce = Vector3.new(math.huge, math.huge, math.huge) bodyVelocity.Velocity = Vector3.new(0, 0, 0) bodyVelocity.Parent = hrp bodyGyro = Instance.new("BodyGyro") bodyGyro.MaxTorque = Vector3.new(math.huge, math.huge, math.huge) bodyGyro.P = 9000 bodyGyro.Parent = hrp showStatusTween:Play() playSwitchOn(switchCircle1, switchBg1) btnUp.Visible = true btnDown.Visible = true showNotification("✅ เปิดใช้งานระบบบิน", Color3.fromRGB(40, 150, 40), Color3.fromRGB(100, 255, 100)) end
local function stopFly() if not flying then return end local character = player.Character if not character then return end local humanoid = character:FindFirstChild("Humanoid") if humanoid then humanoid.PlatformStand = false humanoid.AutoRotate = true humanoid.WalkSpeed = runningFast and runSpeed or 16 end if bodyVelocity then bodyVelocity:Destroy() bodyVelocity = nil end if bodyGyro then bodyGyro:Destroy() bodyGyro = nil end flying = false hideStatusTween:Play() playSwitchOff(switchCircle1, switchBg1) btnUp.Visible = false btnDown.Visible = false holdingUp = false holdingDown = false showNotification("❌ ปิดใช้งานระบบบิน", Color3.fromRGB(200, 40, 40), Color3.fromRGB(255, 100, 100)) end
local function startRun() if runningFast then return end runningFast = true local character = player.Character if character then local humanoid = character:FindFirstChild("Humanoid") if humanoid then humanoid.WalkSpeed = runSpeed end end playSwitchOn(switchCircle2, switchBg2) showNotification("🏃 เปิดวิ่งไว", Color3.fromRGB(40, 150, 40), Color3.fromRGB(100, 255, 100)) end
local function stopRun() if not runningFast then return end runningFast = false local character = player.Character if character then local humanoid = character:FindFirstChild("Humanoid") if humanoid then humanoid.WalkSpeed = 16 end end playSwitchOff(switchCircle2, switchBg2) showNotification("❌ ปิดวิ่งไว", Color3.fromRGB(200, 40, 40), Color3.fromRGB(255, 100, 100)) end
local function startJump() if jumpingHigh then return end jumpingHigh = true local character = player.Character if character then local humanoid = character:FindFirstChild("Humanoid") if humanoid then humanoid.UseJumpPower = true humanoid.JumpPower = jumpPower end end playSwitchOn(switchCircle3, switchBg3) showNotification("🦘 เปิดกระโดดสูง", Color3.fromRGB(40, 150, 40), Color3.fromRGB(100, 255, 100)) end
local function stopJump() if not jumpingHigh then return end jumpingHigh = false local character = player.Character if character then local humanoid = character:FindFirstChild("Humanoid") if humanoid then humanoid.JumpPower = 50 end end playSwitchOff(switchCircle3, switchBg3) showNotification("❌ ปิดกระโดดสูง", Color3.fromRGB(200, 40, 40), Color3.fromRGB(255, 100, 100)) end

-- ==========================================
-- ระบบทะลุบล็อก (Noclip) & ระบบหมุน (Spin)
-- ==========================================
local function startNoclip() noclipEnabled = true playSwitchOn(switchCircle6, switchBg6) showNotification("👻 เปิดทะลุบล็อก", Color3.fromRGB(100, 50, 150), Color3.fromRGB(180, 100, 255)) end
local function stopNoclip() noclipEnabled = false local char = player.Character if char then for _, part in pairs(char:GetDescendants()) do if part:IsA("BasePart") then part.CanCollide = true end end end playSwitchOff(switchCircle6, switchBg6) showNotification("❌ ปิดทะลุบล็อก", Color3.fromRGB(200, 40, 40), Color3.fromRGB(255, 100, 100)) end
local function startSpin() spinningEnabled = true playSwitchOn(switchCircle7, switchBg7) showNotification("🌀 เปิดระบบหมุน", Color3.fromRGB(40, 150, 40), Color3.fromRGB(100, 255, 100)) end
local function stopSpin() spinningEnabled = false playSwitchOff(switchCircle7, switchBg7) showNotification("❌ ปิดระบบหมุน", Color3.fromRGB(200, 40, 40), Color3.fromRGB(255, 100, 100)) end

local dragState = {Fly = false, Run = false, Jump = false, Spin = false}
local function setupSliderDrag(btn, bg, fill, valText, stateKey, minVal, maxVal, callback) btn.InputBegan:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dragState[stateKey] = true input.Changed:Connect(function() if input.UserInputState == Enum.UserInputState.End then dragState[stateKey] = false end end) end end) UserInputService.InputChanged:Connect(function(input) if dragState[stateKey] and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then local relX = math.clamp((input.Position.X - bg.AbsolutePosition.X) / bg.AbsoluteSize.X, 0, 1) fill.Size = UDim2.new(relX, 0, 1, 0) btn.Position = UDim2.new(relX, -7, 0.5, 0) local value = math.floor(minVal + (relX * (maxVal - minVal))) valText.Text = tostring(value) callback(value) end end) end
setupSliderDrag(sliderBtn1, sliderBg1, sliderFill1, valText1, "Fly", 20, 200, function(val) flySpeed = val end)
setupSliderDrag(sliderBtn2, sliderBg2, sliderFill2, valText2, "Run", 20, 200, function(val) runSpeed = val if runningFast and not flying then local humanoid = player.Character and player.Character:FindFirstChild("Humanoid") if humanoid then humanoid.WalkSpeed = val end end end)
setupSliderDrag(sliderBtn3, sliderBg3, sliderFill3, valText3, "Jump", 50, 500, function(val) jumpPower = val if jumpingHigh then local humanoid = player.Character and player.Character:FindFirstChild("Humanoid") if humanoid then humanoid.JumpPower = val end end end)
setupSliderDrag(sliderBtn7, sliderBg7, sliderFill7, valText7, "Spin", 1, 50, function(val) spinSpeed = val end)

-- ==========================================
-- ระบบค้นหาและวาร์ป
-- ==========================================
local function teleportToSearchTarget()
	local targetName = searchBox.Text
	if targetName == "" then return end
	local foundTarget = nil
	for _, p in Players:GetPlayers() do if string.lower(p.Name):find(string.lower(targetName)) and p.Character then foundTarget = p.Character break end end
	if not foundTarget then for _, obj in workspace:GetDescendants() do if string.lower(obj.Name):find(string.lower(targetName)) then if obj:IsA("BasePart") then foundTarget = obj break elseif obj:IsA("Model") then foundTarget = obj break end end end end
	local myChar = player.Character
	if foundTarget and myChar and myChar:FindFirstChild("HumanoidRootPart") then
		local targetPos = nil
		if foundTarget:IsA("BasePart") then targetPos = foundTarget.CFrame
		elseif foundTarget:IsA("Model") then if foundTarget.PrimaryPart then targetPos = foundTarget.PrimaryPart.CFrame elseif foundTarget:FindFirstChildWhichIsA("BasePart") then targetPos = foundTarget:FindFirstChildWhichIsA("BasePart").CFrame end end
		if targetPos then myChar:PivotTo(targetPos * CFrame.new(0, 0, 3)) showNotification("⚡ วาร์ปไปหา: " .. foundTarget.Name, Color3.fromRGB(0, 100, 200), Color3.fromRGB(100, 180, 255)) else showNotification("⚠ ไม่พบพิกัดของ: " .. foundTarget.Name, Color3.fromRGB(200, 50, 50), Color3.fromRGB(255, 100, 100)) end
	else showNotification("⚠ ไม่พบชื่อนี้ในแมพ: " .. targetName, Color3.fromRGB(200, 50, 50), Color3.fromRGB(255, 100, 100)) end
end
searchBtn.MouseButton1Click:Connect(teleportToSearchTarget)
searchBox.FocusLost:Connect(function(enterPressed) if enterPressed then teleportToSearchTarget() end end)

-- ==========================================
-- ระบบ ESP & เส้นล็อคเป้า (Beam)
-- ==========================================
local function clearAllESP() for obj, data in pairs(espObjects) do if data then if data.BillGui then data.BillGui:Destroy() end if data.Beam then data.Beam:Destroy() end if data.Attach1 then data.Attach1:Destroy() end end end espObjects = {} end

local function updateOrCreateESP(obj, name, distance, color, myHrp)
	if not espObjects[obj] then
		local hrp = obj:FindFirstChild("HumanoidRootPart") or obj:FindFirstChild("Head")
		if not hrp then return end
		local att0 = myHrp:FindFirstChild("ESPRootAtt")
		if not att0 then att0 = Instance.new("Attachment") att0.Name = "ESPRootAtt" att0.Parent = myHrp end
		local att1 = Instance.new("Attachment") att1.Name = "ESPTargetAtt" att1.Parent = hrp
		local bb = Instance.new("BillboardGui") bb.Name = "TrackerESP" bb.Adornee = hrp; bb.Size = UDim2.new(0, 200, 0, 30); bb.StudsOffset = Vector3.new(0, 4, 0); bb.AlwaysOnTop = true 
		local txt = Instance.new("TextLabel") txt.Name = "ESPText"; txt.Size = UDim2.new(1,0,1,0) txt.BackgroundColor3 = Color3.fromRGB(0, 0, 0) txt.BackgroundTransparency = 0.4 txt.TextColor3 = color txt.TextStrokeColor3 = Color3.fromRGB(0, 0, 0) txt.TextStrokeTransparency = 0.5 txt.Font = Enum.Font.GothamBold; txt.TextSize = 14; txt.Parent = bb
		bb.Parent = obj
		local beam = Instance.new("Beam") beam.Attachment0 = att0 beam.Attachment1 = att1 beam.Color = ColorSequence.new(color) beam.Width0 = 0.15 beam.Width1 = 0.15 beam.Transparency = NumberSequence.new(0.4) beam.LightEmission = 0 beam.LightInfluence = 0 beam.Brightness = 0 beam.FaceCamera = true beam.Parent = obj
		espObjects[obj] = { BillGui = bb, Beam = beam, Attach1 = att1 }
	end
	local data = espObjects[obj]
	if data and data.BillGui and data.BillGui:FindFirstChild("ESPText") then data.BillGui.ESPText.Text = name .. " [" .. distance .. "m]" end
end

task.spawn(function() while task.wait(2) do if espNPCsEnabled then npcCache = {} for _, obj in workspace:GetDescendants() do if obj:IsA("Model") then local hum = obj:FindFirstChildOfClass("Humanoid") if hum and not Players:GetPlayerFromCharacter(obj) and obj ~= player.Character then table.insert(npcCache, obj) end end end end end end)

function refreshPlayerList()
	for _, child in plScroll:GetChildren() do if child:IsA("Frame") then child:Destroy() end end
	for _, p in Players:GetPlayers() do
		if p ~= player then
			local btnFrame = Instance.new("Frame") btnFrame.Size = UDim2.new(1, 0, 0, 35) btnFrame.BackgroundColor3 = Color3.fromRGB(35, 35, 35) btnFrame.Parent = plScroll Instance.new("UICorner", btnFrame).CornerRadius = UDim.new(0, 6)
			local avatar = Instance.new("ImageLabel") avatar.Size = UDim2.new(0, 25, 0, 25) avatar.Position = UDim2.new(0, 5, 0.5, 0) avatar.AnchorPoint = Vector2.new(0, 0.5) avatar.Image = "https://www.roblox.com/headshot-thumbnail/image?userId=" .. p.UserId .. "&width=150&height=150&format=Png" avatar.BackgroundColor3 = Color3.fromRGB(50, 50, 50) avatar.Parent = btnFrame Instance.new("UICorner", avatar).CornerRadius = UDim.new(0.5, 0)
			local nameLbl = Instance.new("TextLabel") nameLbl.Size = UDim2.new(1, -35, 1, 0) nameLbl.Position = UDim2.new(0, 35, 0, 0) nameLbl.BackgroundTransparency = 1 nameLbl.TextColor3 = Color3.fromRGB(255, 255, 255) nameLbl.Text = p.Name nameLbl.Font = Enum.Font.Gotham nameLbl.TextSize = 13 nameLbl.TextXAlignment = Enum.TextXAlignment.Left nameLbl.Parent = btnFrame
			local clickBtn = Instance.new("TextButton") clickBtn.Size = UDim2.new(1, 0, 1, 0) clickBtn.BackgroundTransparency = 1 clickBtn.Text = "" clickBtn.Parent = btnFrame
			clickBtn.MouseButton1Click:Connect(function()
				local targetChar = p.Character; local myChar = player.Character
				if myChar and myChar:FindFirstChild("HumanoidRootPart") and targetChar and targetChar:FindFirstChild("HumanoidRootPart") then
					myChar:PivotTo(targetChar.HumanoidRootPart.CFrame * CFrame.new(0, 0, 3))
					showNotification("⚡ วาร์ปไปหา: " .. p.Name, Color3.fromRGB(0, 100, 200), Color3.fromRGB(100, 180, 255))
				else showNotification("⚠ ไม่พบตัวละครปลายทาง", Color3.fromRGB(200, 50, 50), Color3.fromRGB(255, 100, 100)) end
			end)
		end
	end
	plScroll.CanvasSize = UDim2.new(0, 0, 0, plLayout.AbsoluteContentSize.Y)
end
Players.PlayerAdded:Connect(function(p) task.wait(1) if playerListFrame.Visible then refreshPlayerList() end end)
Players.PlayerRemoving:Connect(function(p) if playerListFrame.Visible then refreshPlayerList() end end)

-- ==========================================
-- ระบบควบคุมหลัก RenderStepped
-- ==========================================
RunService.RenderStepped:Connect(function()
	-- ระบบบิน
	if flying and bodyVelocity and bodyGyro then
		local moveDir = Vector3.new(0, 0, 0)
		local character = player.Character local humanoid = character and character:FindFirstChild("Humanoid")
		if UserInputService:IsKeyDown(Enum.KeyCode.W) then moveDir = moveDir + camera.CFrame.LookVector end
		if UserInputService:IsKeyDown(Enum.KeyCode.S) then moveDir = moveDir - camera.CFrame.LookVector end
		if UserInputService:IsKeyDown(Enum.KeyCode.A) then moveDir = moveDir - camera.CFrame.RightVector end
		if UserInputService:IsKeyDown(Enum.KeyCode.D) then moveDir = moveDir + camera.CFrame.RightVector end
		if UserInputService:IsKeyDown(Enum.KeyCode.Space) then moveDir = moveDir + Vector3.new(0, 1, 0) end
		if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then moveDir = moveDir - Vector3.new(0, 1, 0) end
		if humanoid and humanoid.MoveDirection.Magnitude > 0 then moveDir = moveDir + Vector3.new(humanoid.MoveDirection.X, 0, humanoid.MoveDirection.Z) end
		if holdingUp then moveDir = moveDir + Vector3.new(0, 1, 0) end
		if holdingDown then moveDir = moveDir - Vector3.new(0, 1, 0) end
		if moveDir.Magnitude > 0 then bodyVelocity.Velocity = moveDir.Unit * flySpeed else bodyVelocity.Velocity = Vector3.new(0, 0, 0) end
		bodyGyro.CFrame = camera.CFrame
	end

	-- ระบบทะลุบล็อก (Noclip)
	if noclipEnabled then
		local char = player.Character
		if char then
			for _, part in pairs(char:GetDescendants()) do
				if part:IsA("BasePart") then
					part.CanCollide = false
				end
			end
		end
	end

	-- ระบบหมุน (Spin)
	if spinningEnabled then
		local char = player.Character
		if char and char:FindFirstChild("HumanoidRootPart") then
			local hrp = char.HumanoidRootPart
			hrp.CFrame = hrp.CFrame * CFrame.Angles(0, math.rad(spinSpeed), 0)
		end
	end

	-- ระบบ ESP
	local isTrackingAny = espPlayersEnabled or espNPCsEnabled
	if isTrackingAny then
		local myChar = player.Character; local myHrp = myChar and myChar:FindFirstChild("HumanoidRootPart")
		if myHrp then
			if espPlayersEnabled then for _, p in Players:GetPlayers() do if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then local dist = math.floor((myHrp.Position - p.Character.HumanoidRootPart.Position).Magnitude) updateOrCreateESP(p.Character, p.Name, dist, Color3.fromRGB(0, 255, 100), myHrp) end end end
			if espNPCsEnabled then for _, npc in npcCache do if npc and npc:FindFirstChild("HumanoidRootPart") and npc:FindFirstChildOfClass("Humanoid") and npc:FindFirstChildOfClass("Humanoid").Health > 0 then local dist = math.floor((myHrp.Position - npc.HumanoidRootPart.Position).Magnitude) updateOrCreateESP(npc, npc.Name, dist, Color3.fromRGB(255, 165, 0), myHrp) end end end
		end
	else clearAllESP() end
end)

-- ==========================================
-- ระบบกดปุ่ม
-- ==========================================
UserInputService.InputBegan:Connect(function(input, gameProcessed)
	if gameProcessed then return end 
	if input.KeyCode == Enum.KeyCode.G then togglePanelUI() end
	if input.KeyCode == Enum.KeyCode.CapsLock then if not flying then startFly() else stopFly() end end
	if input.KeyCode == Enum.KeyCode.Z then
		local isAnyOn = espPlayersEnabled or espNPCsEnabled
		if not isAnyOn then espPlayersEnabled = true; playSwitchOn(switchCircle4, switchBg4); espNPCsEnabled = true; playSwitchOn(switchCircle5, switchBg5) showNotification("📍 เปิดมองเห็นผู้เล่น & NPC", Color3.fromRGB(40, 100, 150), Color3.fromRGB(100, 200, 255)) else espPlayersEnabled = false; playSwitchOff(switchCircle4, switchBg4); espNPCsEnabled = false; playSwitchOff(switchCircle5, switchBg5) showNotification("❌ ปิดมองเห็นผู้เล่น & NPC", Color3.fromRGB(200, 40, 40), Color3.fromRGB(255, 100, 100)) end
	end
end)

tabBtn1.MouseButton1Click:Connect(function() switchTab(1) end)
tabBtn2.MouseButton1Click:Connect(function() switchTab(2) end)
tabBtn3.MouseButton1Click:Connect(function() switchTab(3) end)
switchBg1.MouseButton1Click:Connect(function() if not flying then startFly() else stopFly() end end)
switchBg2.MouseButton1Click:Connect(function() if not runningFast then startRun() else stopRun() end end)
switchBg3.MouseButton1Click:Connect(function() if not jumpingHigh then startJump() else stopJump() end end)
switchBg4.MouseButton1Click:Connect(function() if not espPlayersEnabled then espPlayersEnabled = true; playSwitchOn(switchCircle4, switchBg4) showNotification("📍 เปิดมองเห็นผู้เล่น", Color3.fromRGB(40, 100, 150), Color3.fromRGB(100, 200, 255)) else espPlayersEnabled = false; playSwitchOff(switchCircle4, switchBg4) showNotification("❌ ปิดมองเห็นผู้เล่น", Color3.fromRGB(200, 40, 40), Color3.fromRGB(255, 100, 100)) end end)
switchBg5.MouseButton1Click:Connect(function() if not espNPCsEnabled then espNPCsEnabled = true; playSwitchOn(switchCircle5, switchBg5) showNotification("📍 เปิดมองเห็น NPC", Color3.fromRGB(40, 100, 150), Color3.fromRGB(100, 200, 255)) else espNPCsEnabled = false; playSwitchOff(switchCircle5, switchBg5) showNotification("❌ ปิดมองเห็น NPC", Color3.fromRGB(200, 40, 40), Color3.fromRGB(255, 100, 100)) end end)

switchBg6.MouseButton1Click:Connect(function() if not noclipEnabled then startNoclip() else stopNoclip() end end)
switchBg7.MouseButton1Click:Connect(function() if not spinningEnabled then startSpin() else stopSpin() end end)

resetBtn.MouseButton1Click:Connect(function()
	mobileBtn.Position = UDim2.new(0, 20, 0, 20)
	showNotification("🔄 รีเซ็ตตำแหน่งปุ่มแล้ว", Color3.fromRGB(100, 100, 100), Color3.fromRGB(200, 200, 200))
end)
