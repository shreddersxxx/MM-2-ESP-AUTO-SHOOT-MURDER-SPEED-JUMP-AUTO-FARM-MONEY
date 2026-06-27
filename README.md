local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local LocalPlayer = Players.LocalPlayer

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "MM2_GUI"
ScreenGui.Parent = game:GetService("CoreGui")

local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 220, 0, 220)
MainFrame.Position = UDim2.new(0.5, -110, 0.5, -110)
MainFrame.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.Parent = ScreenGui

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 30)
Title.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
Title.Text = "MM2 Cheats"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.Font = Enum.Font.SourceSansBold
Title.TextSize = 18
Title.Parent = MainFrame

local function createToggle(text, yPos, callback)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0.9, 0, 0, 25)
    btn.Position = UDim2.new(0.05, 0, 0, yPos)
    btn.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    btn.Text = text .. ": OFF"
    btn.TextColor3 = Color3.fromRGB(255, 0, 0)
    btn.Font = Enum.Font.SourceSans
    btn.TextSize = 14
    btn.Parent = MainFrame
    btn.MouseButton1Click:Connect(function()
        local state = btn.Text:find("OFF") ~= nil
        if state then
            btn.Text = text .. ": ON"
            btn.TextColor3 = Color3.fromRGB(0, 255, 0)
        else
            btn.Text = text .. ": OFF"
            btn.TextColor3 = Color3.fromRGB(255, 0, 0)
        end
        callback(not state)
    end)
    return btn
end

local espDrawing = {}
local function getRole(player)
    local char = player.Character
    if not char then return nil end
    if player.TeamColor == BrickColor.new("Bright red") then return "Murderer"
    elseif player.TeamColor == BrickColor.new("Bright blue") then return "Sheriff"
    elseif player.TeamColor == BrickColor.new("Bright green") then return "Innocent" end
    local roleObj = char:FindFirstChild("Murderer") or char:FindFirstChild("Sheriff") or char:FindFirstChild("Innocent")
    if roleObj then return roleObj.Name end
    local tool = char:FindFirstChildOfClass("Tool")
    if tool then
        if tool.Name == "Knife" then return "Murderer"
        elseif tool.Name == "Gun" then return "Sheriff" end
    end
    return "Unknown"
end

local function createDrawing(player)
    if espDrawing[player] then return end
    local text = Drawing.new("Text")
    text.Color = Color3.fromRGB(255,255,255)
    text.Size = 14
    text.Center = true
    text.Outline = true
    text.Visible = false
    local box = Drawing.new("Square")
    box.Color = Color3.fromRGB(255,255,255)
    box.Thickness = 1
    box.Filled = false
    box.Visible = false
    espDrawing[player] = {text = text, box = box}
end

local function removeDrawing(player)
    local d = espDrawing[player]
    if d then
        d.text:Remove()
        d.box:Remove()
        espDrawing[player] = nil
    end
end

local espEnabled = false
local espConnection
createToggle("ESP", 40, function(enabled)
    espEnabled = enabled
    if not enabled then
        if espConnection then espConnection:Disconnect() end
        for plr, _ in pairs(espDrawing) do removeDrawing(plr) end
    else
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer then createDrawing(plr) end
        end
        espConnection = RunService.RenderStepped:Connect(function()
            local cam = workspace.CurrentCamera
            for _, plr in ipairs(Players:GetPlayers()) do
                if plr == LocalPlayer then continue end
                local char = plr.Character
                local role = getRole(plr)
                if char and char:FindFirstChild("Head") and role then
                    local headPos = char.Head.Position
                    local screenPos, onScreen = cam:WorldToScreenPoint(headPos)
                    local d = espDrawing[plr]
                    if not d then createDrawing(plr) d = espDrawing[plr] end
                    if onScreen then
                        local color
                        if role == "Murderer" then color = Color3.fromRGB(255,0,0)
                        elseif role == "Sheriff" then color = Color3.fromRGB(0,170,255)
                        elseif role == "Innocent" then color = Color3.fromRGB(0,255,0)
                        else color = Color3.fromRGB(255,255,255) end
                        d.text.Color = color
                        d.box.Color = color
                        d.text.Text = role .. " [" .. plr.Name .. "]"
                        d.text.Position = Vector2.new(screenPos.X, screenPos.Y - 15)
                        d.text.Visible = true
                        local root = char:FindFirstChild("HumanoidRootPart")
                        if root then
                            local rootPos, onScreen2 = cam:WorldToScreenPoint(root.Position)
                            d.box.Position = Vector2.new(rootPos.X - 20, rootPos.Y - 30)
                            d.box.Size = Vector2.new(40, 60)
                            d.box.Visible = onScreen2
                        end
                    else
                        d.text.Visible = false
                        d.box.Visible = false
                    end
                else
                    if espDrawing[plr] then removeDrawing(plr) end
                end
            end
            for plr, _ in pairs(espDrawing) do
                if not Players:FindFirstChild(plr.Name) then removeDrawing(plr) end
            end
        end)
    end
end)

local autoShootEnabled = false
local autoShootConnection
local function getMurderer()
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr == LocalPlayer then continue end
        local char = plr.Character
        if char and char:FindFirstChild("Head") then
            if getRole(plr) == "Murderer" or (char:FindFirstChildOfClass("Tool") and char:FindFirstChildOfClass("Tool").Name == "Knife") then
                return char
            end
        end
    end
    return nil
end

local function smoothTurnAndShoot()
    local murderer = getMurderer()
    if not murderer then return end
    local myChar = LocalPlayer.Character
    if not myChar or not myChar:FindFirstChild("Humanoid") then return end
    local tool = myChar:FindFirstChildOfClass("Tool")
    if not tool then return end

    local targetPos = murderer.Head.Position
    local root = myChar.PrimaryPart
    local originalCF = root.CFrame
    local lookAt = CFrame.new(root.Position, targetPos)
    local tween = TweenService:Create(root, TweenInfo.new(0.2), {CFrame = lookAt})
    tween:Play()
    tween.Completed:Wait()
    task.wait(0.05)
    tool:Activate()
    root.CFrame = originalCF
end

createToggle("AutoShoot (F)", 75, function(enabled)
    autoShootEnabled = enabled
    if autoShootEnabled then
        autoShootConnection = UserInputService.InputBegan:Connect(function(input, gameProcessed)
            if gameProcessed then return end
            if input.KeyCode == Enum.KeyCode.F then
                smoothTurnAndShoot()
            end
        end)
    else
        if autoShootConnection then autoShootConnection:Disconnect() end
    end
end)

local speedConnection
createToggle("Speed (x1.2)", 110, function(enabled)
    if enabled then
        local char = LocalPlayer.Character
        local humanoid = char and char:FindFirstChildOfClass("Humanoid")
        if humanoid then humanoid.WalkSpeed = 19 end
        speedConnection = LocalPlayer.CharacterAdded:Connect(function(newChar)
            local hum = newChar:WaitForChild("Humanoid")
            hum.WalkSpeed = 19
        end)
    else
        if speedConnection then speedConnection:Disconnect() end
        local char = LocalPlayer.Character
        local humanoid = char and char:FindFirstChildOfClass("Humanoid")
        if humanoid then humanoid.WalkSpeed = 16 end
    end
end)

local jumpConnection
createToggle("Jump (x1.2)", 145, function(enabled)
    if enabled then
        local char = LocalPlayer.Character
        local humanoid = char and char:FindFirstChildOfClass("Humanoid")
        if humanoid then humanoid.JumpPower = 60 end
        jumpConnection = LocalPlayer.CharacterAdded:Connect(function(newChar)
            local hum = newChar:WaitForChild("Humanoid")
            hum.JumpPower = 60
        end)
    else
        if jumpConnection then jumpConnection:Disconnect() end
        local char = LocalPlayer.Character
        local humanoid = char and char:FindFirstChildOfClass("Humanoid")
        if humanoid then humanoid.JumpPower = 50 end
    end
end)

local coinLoop
local function collectCoins()
    local char = LocalPlayer.Character
    if not char or not char.PrimaryPart then return end
    local root = char.PrimaryPart
    local radius = 15
    for _, obj in ipairs(workspace:GetDescendants()) do
        if obj.Name == "Coin" or obj.Name == "Coin_Server" then
            if obj:IsA("BasePart") and (obj.Position - root.Position).Magnitude < radius then
                root.CFrame = CFrame.new(obj.Position)
                task.wait(0.05)
                break
            end
        end
    end
end

createToggle("Auto Coin", 180, function(enabled)
    if enabled then
        coinLoop = RunService.Heartbeat:Connect(function()
            pcall(collectCoins)
        end)
    else
        if coinLoop then coinLoop:Disconnect() end
    end
end)
