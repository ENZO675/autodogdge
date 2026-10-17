-- STREAMING_CHUNK:Configuring services and global settings...
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local VirtualInputManager = game:GetService("VirtualInputManager")

local player = Players.LocalPlayer

-- Limpa execuções anteriores para evitar duplicatas e lag
if getgenv().AutoCombatConnection then
getgenv().AutoCombatConnection:Disconnect()
end
if getgenv().AutoCombatCharConnection then
getgenv().AutoCombatCharConnection:Disconnect()
end
if getgenv().AutoCombatUI then
getgenv().AutoCombatUI:Destroy()
end

-- Tabela de configurações
local Settings = {
AutoDodge = false,
AutoParry = false,
MaxDistance = 14
}

local lastPerfect = nil
local lastParryTime = 0

-- STREAMING_CHUNK:Building the Graphical User Interface (GUI)...
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "AutoCombatHub"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = CoreGui
getgenv().AutoCombatUI = ScreenGui

local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 200, 0, 160)
MainFrame.Position = UDim2.new(0.8, 0, 0.5, 0)
MainFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
MainFrame.BorderSizePixel = 0
MainFrame.Parent = ScreenGui

local UICorner = Instance.new("UICorner")
UICorner.CornerRadius = UDim.new(0, 8)
UICorner.Parent = MainFrame

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 30)
Title.BackgroundTransparency = 1
Title.Text = "Auto Combat Hub"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.Font = Enum.Font.GothamBold
Title.TextSize = 14
Title.Parent = MainFrame

local function createToggleButton(name, yPos, settingKey)
local Button = Instance.new("TextButton")
Button.Size = UDim2.new(0.9, 0, 0, 30)
Button.Position = UDim2.new(0.05, 0, 0, yPos)
Button.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
Button.Text = name .. ": OFF"
Button.TextColor3 = Color3.fromRGB(255, 255, 255)
Button.Font = Enum.Font.GothamSemibold
Button.TextSize = 14
Button.Parent = MainFrame

local BtnCorner = Instance.new("UICorner")
BtnCorner.CornerRadius = UDim.new(0, 6)
BtnCorner.Parent = Button

Button.MouseButton1Click:Connect(function()
    Settings[settingKey] = not Settings[settingKey]
    if Settings[settingKey] then
        Button.BackgroundColor3 = Color3.fromRGB(50, 200, 50)
        Button.Text = name .. ": ON"
    else
        Button.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
        Button.Text = name .. ": OFF"
    end
end)


end

createToggleButton("Auto Dodge", 40, "AutoDodge")
createToggleButton("Auto Parry", 80, "AutoParry")

local CloseButton = Instance.new("TextButton")
CloseButton.Size = UDim2.new(0.9, 0, 0, 30)
CloseButton.Position = UDim2.new(0.05, 0, 0, 120)
CloseButton.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
CloseButton.Text = "Close Script"
CloseButton.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseButton.Font = Enum.Font.GothamSemibold
CloseButton.TextSize = 14
CloseButton.Parent = MainFrame

local CloseCorner = Instance.new("UICorner")
CloseCorner.CornerRadius = UDim.new(0, 6)
CloseCorner.Parent = CloseButton

CloseButton.MouseButton1Click:Connect(function()
ScreenGui:Destroy()
if getgenv().AutoCombatConnection then getgenv().AutoCombatConnection:Disconnect() end
if getgenv().AutoCombatCharConnection then getgenv().AutoCombatCharConnection:Disconnect() end
end)

-- STREAMING_CHUNK:Implementing state and target detection functions...
local function getState()
local states = workspace:FindFirstChild("States")
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

-- STREAMING_CHUNK:Implementing Auto Parry with Virtual Input...
local function autoParry(character)
if not Settings.AutoParry then return end

local enemy, occupied = getEnemyAndOccupied(character)
if not enemy then return end

local root = character:FindFirstChild("HumanoidRootPart")
local eroot = enemy:FindFirstChild("HumanoidRootPart")

if root and eroot and (root.Position - eroot.Position).Magnitude > Settings.MaxDistance then
    return
end

-- Verifica se o inimigo está atacando ou carregando ataque
local isAttacking = enemy:GetAttribute("IsAttacking") == true or enemy:GetAttribute("WindingUp") == true

if isAttacking then
    -- Cooldown de 0.5s para não travar o jogo spamando a tecla
    if tick() - lastParryTime > 0.5 then
        lastParryTime = tick()
        
        -- Pressiona F virtualmente
        VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.F, false, game)
        
        -- Solta F após 50 milissegundos
        task.delay(0.05, function()
            VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.F, false, game)
        end)
    end
end


end

-- STREAMING_CHUNK:Implementing Auto Dodge core logic...
local function autoDodge(character)
if not Settings.AutoDodge then return end

local perfect = character:GetAttribute("Perfect")
if perfect == nil or perfect == lastPerfect then return end

local direction = character:GetAttribute("NeededDirection")
if direction ~= "Left" and direction ~= "Right" and direction ~= "Backward" and direction ~= "Forward" then return end

local enemy, occupied = getEnemyAndOccupied(character)
if not enemy or not occupied then return end

local dashing = occupied:FindFirstChild("Dashing")
if dashing and dashing.Value then return end

if enemy:GetAttribute("IsFeinting") == true or enemy:GetAttribute("LocalIsFeinting") == true then return end

local root = character:FindFirstChild("HumanoidRootPart")
local eroot = enemy:FindFirstChild("HumanoidRootPart")
if root and eroot and (root.Position - eroot.Position).Magnitude > Settings.MaxDistance then return end

local events = ReplicatedStorage:FindFirstChild("Events")
local dash = events and events:FindFirstChild("ForceDash")
if not dash then return end

lastPerfect = perfect
dash:Fire({Direction = direction})

-- Double check para garantir que o dash foi registrado
RunService.Heartbeat:Once(function()
    local c_state = getState()
    local c_occupied = c_state and c_state:FindFirstChild("Occupied")
    local c_dashing = c_occupied and c_occupied:FindFirstChild("Dashing")

    if player.Character == character and character:GetAttribute("Perfect") == perfect and (not c_dashing or c_dashing.Value == false) then
        dash:Fire({Direction = direction})
    end
end)

RunService.Heartbeat:Once(function()
    local c_state = getState()
    local c_occupied = c_state and c_state:FindFirstChild("Occupied")
    local c_dashing = c_occupied and c_occupied:FindFirstChild("Dashing")

    if player.Character == character and character:GetAttribute("Perfect") == perfect and (not c_dashing or c_dashing.Value == false) then
        dash:Fire({Direction = direction})
    end
end)


end

-- STREAMING_CHUNK:Initializing the main gameplay loop...
getgenv().AutoCombatConnection = RunService.Heartbeat:Connect(function()
if player.Character then
autoDodge(player.Character)
autoParry(player.Character)
end
end)

getgenv().AutoCombatCharConnection = player.CharacterAdded:Connect(function()
lastPerfect = nil
end)
