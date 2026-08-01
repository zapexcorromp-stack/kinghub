--[[
    SCRIPT ÃšNICO - Sistema Admin Completo para Roblox
    (Kill, Ice, Jail, Kick + Interface grÃ¡fica)

    Tipo: Script (normal, NÃƒO local)
    Onde colocar: ServerScriptService

    Ã‰ SÃ“ ESSE ARQUIVO. Ele mesmo cria os RemoteEvents e injeta a
    interface (GUI) automaticamente em cada jogador que entrar â€”
    nÃ£o precisa colocar nada em StarterPlayerScripts.

    Comandos (tambÃ©m funcionam pelo chat):
    :kill [nome]   -> mata o jogador
    :ice [nome]    -> congela o jogador com bloco azul
    :unice [nome]  -> descongela
    :jail [nome]   -> prende o jogador numa cela
    :unjail [nome] -> libera da cela
    :kick [nome]   -> expulsa o jogador

    ATENÃ‡ÃƒO: este script estÃ¡ SEM restriÃ§Ã£o de admin â€” qualquer jogador
    que entrar no servidor pode usar os comandos (kill, ice, jail, kick),
    inclusive contra os outros. Use com cuidado.
]]

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

-- ================================
-- SEM RESTRIÃ‡ÃƒO: todo mundo que entrar no jogo pode usar os comandos
-- ================================

-- ================================
-- RemoteEvents (comunicaÃ§Ã£o com a interface)
-- ================================
local adminCommandEvent = Instance.new("RemoteEvent")
adminCommandEvent.Name = "AdminCommandEvent"
adminCommandEvent.Parent = ReplicatedStorage

local adminAccessEvent = Instance.new("RemoteEvent")
adminAccessEvent.Name = "AdminAccessEvent"
adminAccessEvent.Parent = ReplicatedStorage

local jailedParts = {} -- guarda as partes da cela criadas para cada jogador
local icedParts = {}   -- guarda o bloco de gelo criado para cada jogador

local function isAdmin(player)
    return true -- sem restriÃ§Ã£o: qualquer jogador pode usar os comandos
end

local function getTargetPlayer(name)
    name = name:lower()
    for _, p in ipairs(Players:GetPlayers()) do
        if p.Name:lower() == name or p.Name:lower():sub(1, #name) == name then
            return p
        end
    end
    return nil
end

-- ===== KILL =====
local function killPlayer(target)
    local character = target.Character
    if character then
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if humanoid then
            humanoid.Health = 0
        end
    end
end

-- ===== ICE (congelar totalmente com bloco azul) =====
local function icePlayer(target)
    local character = target.Character
    if not character then return end
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    local hrp = character:FindFirstChild("HumanoidRootPart")
    if not humanoid or not hrp then return end

    if icedParts[target.UserId] then
        icedParts[target.UserId]:Destroy()
        icedParts[target.UserId] = nil
    end

    humanoid.WalkSpeed = 0
    humanoid.JumpPower = 0
    humanoid.JumpHeight = 0

    -- ancora o HumanoidRootPart: trava o jogador 100%, sem fÃ­sica, sem ser empurrado
    -- isso Ã© replicado pelo servidor, entÃ£o TODOS os jogadores (e ele mesmo) veem o efeito
    hrp.Anchored = true

    local iceBlock = Instance.new("Part")
    iceBlock.Name = target.Name .. "_Ice"
    iceBlock.Size = Vector3.new(6, 6, 6)
    iceBlock.Anchored = true
    iceBlock.CanCollide = false
    iceBlock.Transparency = 0.4
    iceBlock.Material = Enum.Material.Ice
    iceBlock.BrickColor = BrickColor.new("Cyan")
    iceBlock.Color = Color3.fromRGB(0, 140, 255)
    iceBlock.CFrame = hrp.CFrame
    iceBlock.Parent = workspace

    icedParts[target.UserId] = iceBlock
end

local function unicePlayer(target)
    local character = target.Character
    if character then
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        local hrp = character:FindFirstChild("HumanoidRootPart")
        if humanoid then
            humanoid.WalkSpeed = 16
            humanoid.JumpPower = 50
            humanoid.JumpHeight = 7.2
        end
        if hrp then
            hrp.Anchored = false
        end
    end

    if icedParts[target.UserId] then
        icedParts[target.UserId]:Destroy()
        icedParts[target.UserId] = nil
    end
end

-- ===== JAIL (prisÃ£o) =====
local function jailPlayer(target)
    local character = target.Character
    if not character then return end
    local hrp = character:FindFirstChild("HumanoidRootPart")
    if not hrp then return end

    if jailedParts[target.UserId] then
        jailedParts[target.UserId]:Destroy()
    end

    local cellFolder = Instance.new("Folder")
    cellFolder.Name = target.Name .. "_Jail"
    cellFolder.Parent = workspace

    local position = hrp.Position
    local size = Vector3.new(8, 8, 8)

    local offsets = {
        Vector3.new(size.X/2, 0, 0),
        Vector3.new(-size.X/2, 0, 0),
        Vector3.new(0, size.Y/2, 0),
        Vector3.new(0, -size.Y/2, 0),
        Vector3.new(0, 0, size.Z/2),
        Vector3.new(0, 0, -size.Z/2),
    }

    for i, offset in ipairs(offsets) do
        local wall = Instance.new("Part")
        wall.Anchored = true
        wall.CanCollide = true
        wall.Transparency = 0.5
        wall.Material = Enum.Material.ForceField
        wall.BrickColor = BrickColor.new("Institutional white")
        wall.Size = (i <= 2) and Vector3.new(0.5, size.Y, size.Z)
                    or (i <= 4) and Vector3.new(size.X, 0.5, size.Z)
                    or Vector3.new(size.X, size.Y, 0.5)
        wall.CFrame = CFrame.new(position + offset)
        wall.Parent = cellFolder
    end

    jailedParts[target.UserId] = cellFolder
    hrp.CFrame = CFrame.new(position)
end

local function unjailPlayer(target)
    if jailedParts[target.UserId] then
        jailedParts[target.UserId]:Destroy()
        jailedParts[target.UserId] = nil
    end
end

-- ===== KICK =====
local function kickPlayer(target, reason)
    target:Kick(reason or "VocÃª foi expulso por um administrador.")
end

-- ================================
-- Comandos via chat
-- ================================
Players.PlayerAdded:Connect(function(player)
    if isAdmin(player) then
        adminAccessEvent:FireClient(player, true)
    end

    player.CharacterAdded:Connect(function(character)
        if icedParts[player.UserId] then
            icedParts[player.UserId]:Destroy()
            icedParts[player.UserId] = nil
        end
        if jailedParts[player.UserId] then
            jailedParts[player.UserId]:Destroy()
            jailedParts[player.UserId] = nil
        end

        local humanoid = character:WaitForChild("Humanoid")
        humanoid.WalkSpeed = 16
        humanoid.JumpPower = 50
    end)

    player.Chatted:Connect(function(message)
        if not isAdmin(player) then return end

        local args = string.split(message, " ")
        local command = args[1]
        local targetName = args[2]

        if not command or not targetName then return end

        local target = getTargetPlayer(targetName)
        if not target then return end

        if command == ":kill" then
            killPlayer(target)
        elseif command == ":ice" then
            icePlayer(target)
        elseif command == ":unice" then
            unicePlayer(target)
        elseif command == ":jail" then
            jailPlayer(target)
        elseif command == ":unjail" then
            unjailPlayer(target)
        elseif command == ":kick" then
            kickPlayer(target)
        end
    end)
end)

Players.PlayerRemoving:Connect(function(player)
    if jailedParts[player.UserId] then
        jailedParts[player.UserId]:Destroy()
        jailedParts[player.UserId] = nil
    end
    if icedParts[player.UserId] then
        icedParts[player.UserId]:Destroy()
        icedParts[player.UserId] = nil
    end
end)

-- ================================
-- Comandos via interface (GUI)
-- ================================
adminCommandEvent.OnServerEvent:Connect(function(player, command, targetName)
    -- NUNCA confie no cliente: sempre revalide se Ã© admin no servidor
    if not isAdmin(player) then return end
    if type(command) ~= "string" or type(targetName) ~= "string" then return end

    local target = getTargetPlayer(targetName)
    if not target then return end

    if command == "kill" then
        killPlayer(target)
    elseif command == "ice" then
        icePlayer(target)
    elseif command == "unice" then
        unicePlayer(target)
    elseif command == "jail" then
        jailPlayer(target)
    elseif command == "unjail" then
        unjailPlayer(target)
    elseif command == "kick" then
        kickPlayer(target)
    end
end)

-- ================================
-- INTERFACE (GUI) â€” injetada automaticamente em cada jogador
-- Este Ã© o cÃ³digo-fonte de um LocalScript, criado dinamicamente pelo
-- servidor e enviado para o PlayerGui de cada jogador que entra.
-- ================================
local GUI_SOURCE = [==[
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer

local adminCommandEvent = ReplicatedStorage:WaitForChild("AdminCommandEvent")
local adminAccessEvent = ReplicatedStorage:WaitForChild("AdminAccessEvent")

local icedState = {}
local jailedState = {}

local function buildGui()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "AdminGui"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = player:WaitForChild("PlayerGui")

    local toggleButton = Instance.new("TextButton")
    toggleButton.Name = "ToggleButton"
    toggleButton.Size = UDim2.new(0, 100, 0, 40)
    toggleButton.Position = UDim2.new(1, -120, 0, 20)
    toggleButton.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    toggleButton.TextColor3 = Color3.fromRGB(0, 170, 255)
    toggleButton.Font = Enum.Font.GothamBold
    toggleButton.TextSize = 16
    toggleButton.Text = "Admin"
    toggleButton.Parent = screenGui

    local toggleCorner = Instance.new("UICorner")
    toggleCorner.CornerRadius = UDim.new(0, 8)
    toggleCorner.Parent = toggleButton

    local mainFrame = Instance.new("Frame")
    mainFrame.Name = "MainFrame"
    mainFrame.Size = UDim2.new(0, 320, 0, 400)
    mainFrame.Position = UDim2.new(1, -340, 0, 70)
    mainFrame.BackgroundColor3 = Color3.fromRGB(24, 24, 24)
    mainFrame.Visible = false
    mainFrame.Parent = screenGui

    local mainCorner = Instance.new("UICorner")
    mainCorner.CornerRadius = UDim.new(0, 10)
    mainCorner.Parent = mainFrame

    local titleBar = Instance.new("Frame")
    titleBar.Name = "TitleBar"
    titleBar.Size = UDim2.new(1, 0, 0, 36)
    titleBar.BackgroundColor3 = Color3.fromRGB(0, 140, 255)
    titleBar.Parent = mainFrame

    local titleCorner = Instance.new("UICorner")
    titleCorner.CornerRadius = UDim.new(0, 10)
    titleCorner.Parent = titleBar

    local titleLabel = Instance.new("TextLabel")
    titleLabel.BackgroundTransparency = 1
    titleLabel.Size = UDim2.new(1, -40, 1, 0)
    titleLabel.Position = UDim2.new(0, 10, 0, 0)
    titleLabel.TextXAlignment = Enum.TextXAlignment.Left
    titleLabel.Font = Enum.Font.GothamBold
    titleLabel.TextSize = 16
    titleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    titleLabel.Text = "Painel Admin"
    titleLabel.Parent = titleBar

    local closeButton = Instance.new("TextButton")
    closeButton.Size = UDim2.new(0, 30, 0, 30)
    closeButton.Position = UDim2.new(1, -33, 0, 3)
    closeButton.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
    closeButton.Font = Enum.Font.GothamBold
    closeButton.TextSize = 16
    closeButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    closeButton.Text = "X"
    closeButton.Parent = titleBar

    local closeCorner = Instance.new("UICorner")
    closeCorner.CornerRadius = UDim.new(0, 8)
    closeCorner.Parent = closeButton

    local scrollFrame = Instance.new("ScrollingFrame")
    scrollFrame.Name = "PlayerList"
    scrollFrame.Size = UDim2.new(1, -16, 1, -50)
    scrollFrame.Position = UDim2.new(0, 8, 0, 44)
    scrollFrame.BackgroundTransparency = 1
    scrollFrame.BorderSizePixel = 0
    scrollFrame.ScrollBarThickness = 6
    scrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
    scrollFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y
    scrollFrame.Parent = mainFrame

    local listLayout = Instance.new("UIListLayout")
    listLayout.Padding = UDim.new(0, 8)
    listLayout.SortOrder = Enum.SortOrder.LayoutOrder
    listLayout.Parent = scrollFrame

    local function createPlayerRow(targetPlayer)
        local row = Instance.new("Frame")
        row.Name = targetPlayer.Name
        row.Size = UDim2.new(1, 0, 0, 76)
        row.BackgroundColor3 = Color3.fromRGB(36, 36, 36)
        row.Parent = scrollFrame

        local rowCorner = Instance.new("UICorner")
        rowCorner.CornerRadius = UDim.new(0, 8)
        rowCorner.Parent = row

        local nameLabel = Instance.new("TextLabel")
        nameLabel.BackgroundTransparency = 1
        nameLabel.Size = UDim2.new(1, -10, 0, 22)
        nameLabel.Position = UDim2.new(0, 6, 0, 4)
        nameLabel.TextXAlignment = Enum.TextXAlignment.Left
        nameLabel.Font = Enum.Font.GothamBold
        nameLabel.TextSize = 14
        nameLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
        nameLabel.Text = targetPlayer.Name
        nameLabel.Parent = row

        local buttonHolder = Instance.new("Frame")
        buttonHolder.BackgroundTransparency = 1
        buttonHolder.Size = UDim2.new(1, -12, 0, 44)
        buttonHolder.Position = UDim2.new(0, 6, 0, 28)
        buttonHolder.Parent = row

        local gridLayout = Instance.new("UIGridLayout")
        gridLayout.CellSize = UDim2.new(0, 68, 0, 20)
        gridLayout.CellPadding = UDim2.new(0, 4, 0, 4)
        gridLayout.Parent = buttonHolder

        local function makeButton(text, color, order, callback)
            local btn = Instance.new("TextButton")
            btn.Size = UDim2.new(0, 68, 0, 20)
            btn.BackgroundColor3 = color
            btn.Font = Enum.Font.GothamBold
            btn.TextSize = 12
            btn.TextColor3 = Color3.fromRGB(255, 255, 255)
            btn.Text = text
            btn.LayoutOrder = order
            btn.Parent = buttonHolder

            local btnCorner = Instance.new("UICorner")
            btnCorner.CornerRadius = UDim.new(0, 6)
            btnCorner.Parent = btn

            btn.MouseButton1Click:Connect(callback)
            return btn
        end

        makeButton("Kill", Color3.fromRGB(180, 40, 40), 1, function()
            adminCommandEvent:FireServer("kill", targetPlayer.Name)
        end)

        local iceButton
        iceButton = makeButton("Ice", Color3.fromRGB(0, 140, 255), 2, function()
            local isIced = icedState[targetPlayer.UserId]
            if isIced then
                adminCommandEvent:FireServer("unice", targetPlayer.Name)
                icedState[targetPlayer.UserId] = false
                iceButton.Text = "Ice"
            else
                adminCommandEvent:FireServer("ice", targetPlayer.Name)
                icedState[targetPlayer.UserId] = true
                iceButton.Text = "Unice"
            end
        end)

        local jailButton
        jailButton = makeButton("Jail", Color3.fromRGB(120, 90, 40), 3, function()
            local isJailed = jailedState[targetPlayer.UserId]
            if isJailed then
                adminCommandEvent:FireServer("unjail", targetPlayer.Name)
                jailedState[targetPlayer.UserId] = false
                jailButton.Text = "Jail"
            else
                adminCommandEvent:FireServer("jail", targetPlayer.Name)
                jailedState[targetPlayer.UserId] = true
                jailButton.Text = "Unjail"
            end
        end)

        makeButton("Kick", Color3.fromRGB(90, 90, 90), 4, function()
            adminCommandEvent:FireServer("kick", targetPlayer.Name)
        end)

        return row
    end

    for _, p in ipairs(Players:GetPlayers()) do
        createPlayerRow(p)
    end

    Players.PlayerAdded:Connect(function(p)
        createPlayerRow(p)
    end)

    Players.PlayerRemoving:Connect(function(p)
        local row = scrollFrame:FindFirstChild(p.Name)
        if row then
            row:Destroy()
        end
    end)

    local isOpen = false

    local function setOpen(open)
        isOpen = open
        mainFrame.Visible = open
    end

    toggleButton.MouseButton1Click:Connect(function()
        setOpen(not isOpen)
    end)

    closeButton.MouseButton1Click:Connect(function()
        setOpen(false)
    end)
end

adminAccessEvent.OnClientEvent:Connect(function(hasAccess)
    if hasAccess then
        buildGui()
    end
end)
]==]

-- Injeta o LocalScript da interface em cada jogador que entrar
Players.PlayerAdded:Connect(function(player)
    local playerGui = player:WaitForChild("PlayerGui")

    local guiScript = Instance.new("LocalScript")
    guiScript.Name = "AdminGuiScript"
    guiScript.Source = GUI_SOURCE
    guiScript.Parent = playerGui
end)
