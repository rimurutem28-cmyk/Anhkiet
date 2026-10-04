--[[
    Kiet Hub - Fishing Master
    UI mới hoàn toàn + Auto Fish / Sell / Boss / Lock
]]

if not game:IsLoaded() then game.Loaded:Wait() end

local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local CollectionService = game:GetService("CollectionService")
local RunService = game:GetService("RunService")

local lp = Players.LocalPlayer
repeat task.wait() until lp

local rv = getrenv().shared
repeat task.wait(0.1) until rv and rv.LoadingController and rv.LoadingController:IsLoaded()
while lp:GetAttribute("IsLoaded") ~= true do
    lp:GetAttributeChangedSignal("IsLoaded"):Wait()
end

-- ==================== SETTINGS ====================
local S = {
    AutoFish = false,
    AutoSell = false,
    AutoBoss = false,
    AutoLock = true,
    SellWhenFull = true,
    LockRarity = {
        Legendary = true,
        Mythical = true,
        Divine = true,
        Huge = true
    }
}

-- ==================== UTILS ====================
local function pd()
    return rv.PlayerDataV2Controller:Fetch()
end

local function isFull()
    local ok, res = pcall(function()
        return require(game.ReplicatedStorage.Shared.Lib.FishStorageRules).GetState(pd()).isFull
    end)
    return ok and res
end

local function equipRod()
    local data = pd()
    local rod = data and data.RodEquip
    if type(rod) ~= "string" or rod == "" or rod == "None" then return false end
    local char = lp.Character
    if not char then return false end
    if char:FindFirstChild(rod) and char[rod]:IsA("Tool") then return true end
    rv.HeldToolController.SetHeldSlot:Fire(1)
    local t = os.clock() + 4
    repeat task.wait(0.1) until (char:FindFirstChild(rod) and char[rod]:IsA("Tool")) or os.clock() > t
    return char:FindFirstChild(rod) ~= nil
end

local function doLock()
    if not S.AutoLock then return end
    pcall(function()
        local data = pd()
        if not (data and data.Inventory and data.Inventory.Fishes) then return end
        local catalog = require(game.ReplicatedStorage.Data.Catalog)
        local sc = rv.SellController
        for uuid, fish in pairs(data.Inventory.Fishes) do
            if fish.locked ~= true then
                local info = catalog.Fish.GetById(fish.fishId)
                if info then
                    local rarity = info.rarity
                    if S.LockRarity[rarity] or (fish.isHuge and S.LockRarity.Huge) then
                        sc:ToggleLock(uuid)
                    end
                end
            end
        end
    end)
end

-- ==================== AUTO FISH ====================
local fishing = false
local function doFish()
    if fishing then return end
    fishing = true
    pcall(function()
        if not equipRod() then return end
        local fc = rv.FishingController
        if not fc then return end

        fc.FishFirstPull:Fire()

        local start = os.clock()
        local seq = 0
        local nextReel = 0

        while S.AutoFish and os.clock() - start < 25 do
            local now = os.clock()
            if now >= nextReel and not lp:GetAttribute("IsUsingSkill") then
                seq = (seq % 65535) + 1
                fc.FishReelPull:Fire(seq)
                nextReel = now + 0.17
            end
            task.wait()
        end
    end)
    fishing = false
end

-- ==================== AUTO SELL ====================
local selling = false
local function doSell()
    if selling then return end
    selling = true
    pcall(function()
        doLock()
        task.wait(0.3)
        rv.SellController:SellAll()
    end)
    selling = false
end

-- ==================== AUTO BOSS ====================
local function doBoss()
    for _, region in ipairs(CollectionService:GetTagged("BossRegion")) do
        if region:IsA("BasePart") then
            local fx = region:FindFirstChild("BossSpawnerFX")
            if fx and fx:GetAttribute("BossSpawnerFXActive") == true then
                local root = lp.Character and lp.Character:FindFirstChild("HumanoidRootPart")
                if root then
                    local target = region.Position + Vector3.new(0, 6, 0)
                    local dist = (root.Position - target).Magnitude
                    if dist > 30 then
                        local ti = TweenInfo.new(math.clamp(dist / 50, 0.7, 3), Enum.EasingStyle.Linear)
                        local tw = TweenService:Create(root, ti, {CFrame = CFrame.new(target)})
                        tw:Play()
                        tw.Completed:Wait()
                    end
                end
                return true
            end
        end
    end
    return false
end

-- ==================== MAIN LOOP ====================
task.spawn(function()
    while true do
        task.wait(0.35)
        if S.AutoBoss then
            if doBoss() then
                task.wait(1.2)
                continue
            end
        end
        if S.AutoSell and S.SellWhenFull and isFull() then
            doSell()
            task.wait(1.8)
        end
        if S.AutoFish then
            doFish()
        end
        if S.AutoLock and math.random(1, 12) == 1 then
            doLock()
        end
    end
end)

-- ==================== NEW UI ====================
local gui = Instance.new("ScreenGui")
gui.Name = "KietHub"
gui.ResetOnSpawn = false
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
gui.Parent = lp:WaitForChild("PlayerGui")

-- Main Frame
local main = Instance.new("Frame")
main.Size = UDim2.fromOffset(380, 320)
main.Position = UDim2.new(0.5, -190, 0.5, -160)
main.BackgroundColor3 = Color3.fromRGB(16, 16, 22)
main.BorderSizePixel = 0
main.Parent = gui
Instance.new("UICorner", main).CornerRadius = UDim.new(0, 12)

local stroke = Instance.new("UIStroke")
stroke.Color = Color3.fromRGB(0, 190, 255)
stroke.Thickness = 1.5
stroke.Parent = main

-- Title Bar
local titleBar = Instance.new("Frame")
titleBar.Size = UDim2.new(1, 0, 0, 42)
titleBar.BackgroundColor3 = Color3.fromRGB(22, 24, 32)
titleBar.BorderSizePixel = 0
titleBar.Parent = main
Instance.new("UICorner", titleBar).CornerRadius = UDim.new(0, 12)

local titleFix = Instance.new("Frame")
titleFix.Size = UDim2.new(1, 0, 0, 16)
titleFix.Position = UDim2.new(0, 0, 1, -16)
titleFix.BackgroundColor3 = Color3.fromRGB(22, 24, 32)
titleFix.BorderSizePixel = 0
titleFix.Parent = titleBar

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -50, 1, 0)
title.Position = UDim2.fromOffset(16, 0)
title.BackgroundTransparency = 1
title.Text = "Kiet Hub  |  Fishing"
title.TextColor3 = Color3.fromRGB(0, 200, 255)
title.Font = Enum.Font.GothamBold
title.TextSize = 16
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = titleBar

local closeBtn = Instance.new("TextButton")
closeBtn.Size = UDim2.fromOffset(28, 28)
closeBtn.Position = UDim2.new(1, -36, 0.5, -14)
closeBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
closeBtn.Text = "×"
closeBtn.TextColor3 = Color3.fromRGB(220, 220, 230)
closeBtn.Font = Enum.Font.GothamBold
closeBtn.TextSize = 18
closeBtn.Parent = titleBar
Instance.new("UICorner", closeBtn).CornerRadius = UDim.new(0, 6)

closeBtn.MouseButton1Click:Connect(function()
    gui.Enabled = false
end)

-- Content
local content = Instance.new("Frame")
content.Size = UDim2.new(1, -24, 1, -58)
content.Position = UDim2.fromOffset(12, 50)
content.BackgroundTransparency = 1
content.Parent = main

local list = Instance.new("UIListLayout")
list.Padding = UDim.new(0, 8)
list.Parent = content

local function createToggle(name, key, default)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, 0, 0, 40)
    btn.BackgroundColor3 = Color3.fromRGB(28, 30, 40)
    btn.Text = ""
    btn.AutoButtonColor = false
    btn.Parent = content
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 8)

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -70, 1, 0)
    label.Position = UDim2.fromOffset(14, 0)
    label.BackgroundTransparency = 1
    label.Text = name
    label.TextColor3 = Color3.fromRGB(230, 235, 245)
    label.Font = Enum.Font.GothamMedium
    label.TextSize = 14
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = btn

    local switchBG = Instance.new("Frame")
    switchBG.Size = UDim2.fromOffset(44, 24)
    switchBG.Position = UDim2.new(1, -56, 0.5, -12)
    switchBG.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    switchBG.Parent = btn
    Instance.new("UICorner", switchBG).CornerRadius = UDim.new(1, 0)

    local circle = Instance.new("Frame")
    circle.Size = UDim2.fromOffset(18, 18)
    circle.Position = UDim2.fromOffset(3, 3)
    circle.BackgroundColor3 = Color3.fromRGB(200, 200, 210)
    circle.Parent = switchBG
    Instance.new("UICorner", circle).CornerRadius = UDim.new(1, 0)

    local function update(state)
        S[key] = state
        TweenService:Create(switchBG, TweenInfo.new(0.2), {
            BackgroundColor3 = state and Color3.fromRGB(0, 180, 255) or Color3.fromRGB(50, 50, 60)
        }):Play()
        TweenService:Create(circle, TweenInfo.new(0.2), {
            Position = state and UDim2.fromOffset(23, 3) or UDim2.fromOffset(3, 3),
            BackgroundColor3 = state and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(200, 200, 210)
        }):Play()
    end

    update(default or false)

    btn.MouseButton1Click:Connect(function()
        update(not S[key])
    end)
end

createToggle("Auto Fish", "AutoFish", false)
createToggle("Auto Sell (Full)", "AutoSell", false)
createToggle("Auto Boss", "AutoBoss", false)
createToggle("Auto Lock High Rarity", "AutoLock", true)

-- Status
local status = Instance.new("TextLabel")
status.Size = UDim2.new(1, 0, 0, 24)
status.BackgroundTransparency = 1
status.Text = "Status: Ready"
status.TextColor3 = Color3.fromRGB(140, 160, 180)
status.Font = Enum.Font.Gotham
status.TextSize = 12
status.Parent = content

-- Drag
local dragging, dragStart, startPos
titleBar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = main.Position
    end
end)
titleBar.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = false
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local delta = input.Position - dragStart
        main.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)

-- Toggle UI
UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.KeyCode == Enum.KeyCode.LeftControl then
        gui.Enabled = not gui.Enabled
    end
end)

print("[Kiet Hub] Loaded | UI mới | No Key")
```
