-- SERVICES
local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

-- PLAYER
local plr = Players.LocalPlayer
local cam = workspace.CurrentCamera

-- SETTINGS
local freecam = false
local sensitivityX = 0.35
local sensitivityY = 0.35

-- SPEED SETTINGS
local SpeedLevel = 5.0 -- 1.0 - 10.0 (พิมพ์เองได้)
local MinSpeed = 1.0
local MaxSpeed = 10.0
local BaseSpeed = 1.5

-- CAMERA STATE
local yaw, pitch = 0, 0
local conn
local savedCamCF
local savedRootCF
local cameraTouch

-- SPEED FUNC
local function getMoveSpeed()
	return BaseSpeed * math.clamp(SpeedLevel, MinSpeed, MaxSpeed)
end

-- ================= UI =================
local gui = Instance.new("ScreenGui")
gui.Name = "SpiritOutUI"
gui.Parent = game.CoreGui

local frame = Instance.new("Frame", gui)
frame.Size = UDim2.fromScale(0.32, 0.18)
frame.Position = UDim2.fromScale(0.64, 0.05)
frame.BackgroundColor3 = Color3.fromRGB(25,25,25)
frame.BackgroundTransparency = 0.15
frame.BorderSizePixel = 0
frame.Active = true
Instance.new("UICorner", frame).CornerRadius = UDim.new(0.2,0)

local title = Instance.new("TextLabel", frame)
title.Size = UDim2.fromScale(1,0.3)
title.BackgroundTransparency = 1
title.Text = "ถอดวิญญาณ"
title.TextScaled = true
title.TextColor3 = Color3.new(1,1,1)
title.Font = Enum.Font.SourceSansBold

local btn = Instance.new("TextButton", frame)
btn.Size = UDim2.fromScale(0.9,0.3)
btn.Position = UDim2.fromScale(0.05,0.32)
btn.Text = "ยังไม่ถอด"
btn.TextScaled = true
btn.BackgroundColor3 = Color3.fromRGB(40,40,40)
btn.TextColor3 = Color3.new(1,1,1)
btn.BorderSizePixel = 0
Instance.new("UICorner", btn).CornerRadius = UDim.new(0.25,0)

-- SPEED INPUT
local speedBox = Instance.new("TextBox", frame)
speedBox.Size = UDim2.fromScale(0.9, 0.25)
speedBox.Position = UDim2.fromScale(0.05, 0.68)
speedBox.BackgroundColor3 = Color3.fromRGB(40,40,40)
speedBox.TextColor3 = Color3.new(1,1,1)
speedBox.TextScaled = true
speedBox.Font = Enum.Font.SourceSansBold
speedBox.PlaceholderText = "Speed 1.0 - 10.0"
speedBox.Text = string.format("%.1f", SpeedLevel)
speedBox.ClearTextOnFocus = false
speedBox.BorderSizePixel = 0
Instance.new("UICorner", speedBox).CornerRadius = UDim.new(0.25,0)

-- INPUT SPEED
speedBox.FocusLost:Connect(function()
	local v = tonumber(speedBox.Text)
	if v then
		SpeedLevel = math.clamp(v, MinSpeed, MaxSpeed)
	end
	speedBox.Text = string.format("%.1f", SpeedLevel)
end)

-- ============ DRAG ============
local dragging, dragStart, startPos
frame.InputBegan:Connect(function(i)
	if i.UserInputType == Enum.UserInputType.Touch then
		dragging = true
		dragStart = i.Position
		startPos = frame.Position
	end
end)

frame.InputEnded:Connect(function(i)
	if i.UserInputType == Enum.UserInputType.Touch then
		dragging = false
	end
end)

UIS.InputChanged:Connect(function(i)
	if dragging and i.UserInputType == Enum.UserInputType.Touch then
		local d = i.Position - dragStart
		frame.Position = UDim2.new(
			startPos.X.Scale, startPos.X.Offset + d.X,
			startPos.Y.Scale, startPos.Y.Offset + d.Y
		)
	end
end)

-- ============ TOGGLE ============
btn.MouseButton1Click:Connect(function()
	freecam = not freecam
	btn.Text = freecam and "ถอดแล้ว" or "ยังไม่ถอด"

	local char = plr.Character
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	local root = char and char:FindFirstChild("HumanoidRootPart")
	if not hum or not root then return end

	if freecam then
		savedCamCF = cam.CFrame
		savedRootCF = root.CFrame

		local _, y, _ = root.CFrame:ToOrientation()
		yaw = math.deg(y)
		pitch = 0

		cam.CameraType = Enum.CameraType.Scriptable

		conn = RunService.RenderStepped:Connect(function()
			root.CFrame = savedRootCF
			root.AssemblyLinearVelocity = Vector3.zero
			root.AssemblyAngularVelocity = Vector3.zero

			local rot =
				CFrame.Angles(0, math.rad(yaw), 0) *
				CFrame.Angles(math.rad(pitch), 0, 0)

			local moveVec = Vector3.zero
			local dir = hum.MoveDirection

			if dir.Magnitude > 0 then
				local look = cam.CFrame.LookVector
				local right = cam.CFrame.RightVector

				local flatLook = Vector3.new(look.X,0,look.Z).Unit
				local flatRight = Vector3.new(right.X,0,right.Z).Unit

				local forward = dir:Dot(flatLook)
				local side = dir:Dot(flatRight)

				-- บินขึ้น / ลง ตามมุมกล้อง (เดินหน้า + ถอย)
				local vertical = math.sin(math.rad(pitch)) * forward

				local spd = getMoveSpeed()

				moveVec =
					(flatLook * forward + flatRight * side) * spd +
					Vector3.new(0, vertical * spd, 0)
			end

			cam.CFrame = CFrame.new(cam.CFrame.Position + moveVec) * rot
		end)
	else
		if conn then conn:Disconnect() conn = nil end
		cam.CameraType = Enum.CameraType.Custom
		cam.CFrame = savedCamCF
	end
end)

-- ============ TOUCH LOOK ============
UIS.TouchStarted:Connect(function(i)
	if freecam and i.Position.X > cam.ViewportSize.X/2 then
		cameraTouch = i
	end
end)

UIS.TouchMoved:Connect(function(i)
	if freecam and i == cameraTouch then
		yaw -= i.Delta.X * sensitivityX
