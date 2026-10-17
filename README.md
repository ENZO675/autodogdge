-- STREAMING_CHUNK:Configuring settings and services...
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local CoreGui = game:GetService("CoreGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer

-- Tabela de configurações para controle do script
local Settings = {
AutoDodge = false,
AutoParry = false,
MaxDistance = 14,
IsRunning = true
}

local lastPerfect

-- Limpa execuções anteriores para evitar duplicação
if getgenv().CombatHub_Conn then
getgenv().CombatHub_Conn:Disconnect()
end
if getgenv().CombatHub_Char then
getgenv().CombatHub_Char:Disconnect()
end
if getgenv().CombatHub_UI then
getgenv().CombatHub_UI:Destroy()
end

-- STREAMING_CHUNK:Building the interface (GUI)...
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "CombatHubUI"
ScreenGui.ResetOnSpawn = false
-- Usa CoreGui se for um executor de exploit, caso contrário PlayerGui
local success = pcall(function() ScreenGui.Parent = CoreGui end)
if not success then ScreenGui.Parent = player:WaitForChild("PlayerGui") end
getgenv().CombatHub_UI = ScreenGui

local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 250, 0, 200)
MainFrame.Position = UDim2.new(0.5, -125, 0.5, -100)
MainFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
MainFrame.BorderSizePixel = 0
MainFrame.Parent = ScreenGui

local UICorner = Instance.new("UICorner")
UICorner.CornerRadius = UDim.new(0, 8)
UICorner.Parent = MainFrame

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 40)
Title.BackgroundTransparency = 1
Title.Text = "⚡ Combat Hub"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.TextSize = 18
Title.Font = Enum.Font.GothamBold
Title.Parent = MainFrame

-- STREAMING_CHUNK:Adding draggable functionality to the UI...
local dragging, dragInput, dragStart, startPos
Title.InputBegan:Connect(function(input)
if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
dragging = true
dragStart = input.Position
startPos = MainFrame.Position
input.Changed:Connect(function()
if input.UserInputState == Enum.UserInputState.End then dragging = false end
end)
end
end)
Title.InputChanged:Connect(function(input)
if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then dragInput = input end
end)
UserInputService.InputChanged:Connect(function(input)
if input == dragInput and dragging then
local delta = input.Position - dragStart
MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
end
end)

-- STREAMING_CHUNK:Creating toggle buttons...
local function createToggle(name, yPos, settingKey)
local ToggleBtn = Instance.new("TextButton")
ToggleBtn.Size = UDim2.new(0.8, 0, 0, 35)
ToggleBtn.Position = UDim2.new(0.1, 0, 0, yPos)
ToggleBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 55)
ToggleBtn.Text = name .. " [OFF]"
ToggleBtn.TextColor3 = Color3.fromRGB(200, 50, 50)
ToggleBtn.Font = Enum.Font.GothamSemibold
ToggleBtn.TextSize = 14
ToggleBtn.Parent = MainFrame

local BtnCorner = Instance.new("UICorner")
BtnCorner.CornerRadius = UDim.new(0, 6)
BtnCorner.Parent = ToggleBtn

ToggleBtn.MouseButton1Click:Connect(function()
    Settings[settingKey] = not Settings[settingKey]
    if Settings[settingKey] then
        ToggleBtn.Text = name .. " [ON]"
        ToggleBtn.TextColor3 = Color3.fromRGB(50, 200, 50)
    else
        ToggleBtn.Text = name .. " [OFF]"
        ToggleBtn.TextColor3 = Color3.fromRGB(200, 50, 50)
    end
end)


end

createToggle("Auto Dodge", 50, "AutoDodge")
createToggle("Auto Parry", 95, "AutoParry")

local CloseBtn = Instance.new("TextButton")
CloseBtn.Size = UDim2.new(0.8, 0, 0, 30)
CloseBtn.Position = UDim2.new(0.1, 0, 0, 150)
CloseBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
CloseBtn.Text = "Destruir Interface"
CloseBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseBtn.Font = Enum.Font.GothamBold
CloseBtn.TextSize = 14
CloseBtn.Parent = MainFrame

local CloseCorner = Instance.new("UICorner")
CloseCorner.CornerRadius = UDim.new(0, 6)
CloseCorner.Parent = CloseBtn

CloseBtn.MouseButton1Click:Connect(function()
Settings.IsRunning = false
if getgenv().CombatHub_Conn then getgenv().CombatHub_Conn:Disconnect() end
if getgenv().CombatHub_Char then getgenv().CombatHub_Char:Disconnect() end
ScreenGui:Destroy()
end)

-- STREAMING_CHUNK:Implementing helper functions for combat logic...
local function getState()
local states = game.Workspace:FindFirstChild("States")
if states then
return states:FindFirstChild(player.Name)
end
end

local function getEnemyAndOccupied(character)
local state = getState()
if not state then return nil, nil end

local occupied = state:FindFirstChild("Occupied")
if not occupied then return nil, nil end

local locked = occupied:FindFirstChild("LockedOn")
if not locked or not locked.Value then return nil, nil end

return locked.Value, occupied


end

-- STREAMING_CHUNK:Implementing Auto Parry logic...
local function autoParry(character)
if not Settings.AutoParry then return end

local enemy, occupied = getEnemyAndOccupied(character)
if not enemy then return end

local root = character:FindFirstChild("HumanoidRootPart")
local eroot = enemy:FindFirstChild("HumanoidRootPart")

if root and eroot and (root.Position - eroot.Position).Magnitude > Settings.MaxDistance then
    return
end

-- Verifica se o inimigo está atacando (Mudar os atributos conforme o jogo específico)
local isAttacking = enemy:GetAttribute("IsAttacking") == true or enemy:GetAttribute("WindingUp") == true
if isAttacking then
    local events = ReplicatedStorage:FindFirstChild("Events")
    local parryEvent = events and events:FindFirstChild("Parry") or events:FindFirstChild("Block")
    if parryEvent then
        parryEvent:FireServer() -- Supondo uso de FireServer para parry
    end
end


end

-- STREAMING_CHUNK:Refactoring and improving Auto Dodge logic...
local function dodge(character)
if not Settings.AutoDodge then return end

local perfect = character:GetAttribute("Perfect")
if perfect == nil or perfect == lastPerfect then return end

local direction = character:GetAttribute("NeededDirection")
if direction ~= "Left" and direction ~= "Right" and direction ~= "Backward" and direction ~= "Forward" then
    return
end

local enemy, occupied = getEnemyAndOccupied(character)
if not enemy then return end

local dashing = occupied:FindFirstChild("Dashing")
if dashing and dashing.Value then return end

-- Ignora se for Feinting
if enemy:GetAttribute("IsFeinting") == true or enemy:GetAttribute("LocalIsFeinting") == true then
    return
end

local root = character:FindFirstChild("HumanoidRootPart")
local eroot = enemy:FindFirstChild("HumanoidRootPart")
if root and eroot and (root.Position - eroot.Position).Magnitude > Settings.MaxDistance then
    return
end

local events = ReplicatedStorage:FindFirstChild("Events")
local dash = events and events:FindFirstChild("ForceDash")
if not dash then return end

lastPerfect = perfect

-- Tenta executar o dash
dash:Fire({Direction = direction})

-- Tentativas secundárias em caso de falha de conexão (rollback do servidor)
for i = 1, 2 do
    RunService.Heartbeat:Once(function()
        local currentEnemy, currentOccupied = getEnemyAndOccupied(character)
        local currentDashing = currentOccupied and currentOccupied:FindFirstChild("Dashing")

        if player.Character == character and character:GetAttribute("Perfect") == perfect and (not currentDashing or currentDashing.Value == false) then
            dash:Fire({Direction = direction})
        end
    end)
end


end

-- STREAMING_CHUNK:Initializing global connections...
getgenv().CombatHub_Conn = RunService.Heartbeat:Connect(function()
if not Settings.IsRunning then return end

local character = player.Character
if character then
    dodge(character)
    autoParry(character)
end


end)

getgenv().CombatHub_Char = player.CharacterAdded:Connect(function()
lastPerfect = nil
end)
