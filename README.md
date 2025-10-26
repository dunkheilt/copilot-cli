--// 🌌 Dead hub - Tela de Carregamento Futurista (CLEAN & OP)
-- LocalScript -> StarterPlayerScripts

local TweenService = game:GetService("TweenService")
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local StarterGui = game:GetService("StarterGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

-- CONFIG
local CONFIG = {
    Theme = {
        bgStart = Color3.fromRGB(90,0,150),
        bgEnd = Color3.fromRGB(170,0,255),
        accent = Color3.fromRGB(200,130,255),
        text = Color3.fromRGB(255,220,255)
    },
    Music = {
        SoundId = "rbxassetid://1841647093",
        Volume = 0.6,
        FadeStep = 0.02
    },
    Keylist = { "DEAD-OP-2025", "GODMODE-RODRI" }, -- local only
    ParticleCount = 28,
    ParticleUpdateRate = 0.06,
    ESPUpdateRate = 0.12
}

-- UTILS
local function safePcall(fn, ...) local ok, res = pcall(fn, ...) if not ok then warn("[DeadHub] ", res) end return ok, res end
local function clamp(v, a, b) return math.max(a, math.min(b, v)) end

-- CONNECTION TRACKING (para evitar leaks)
local Connections = {}
local function addConn(conn) if conn then table.insert(Connections, conn) end end
local function disconnectAll()
    for _, c in ipairs(Connections) do
        if c and c.Disconnect then
            pcall(function() c:Disconnect() end)
        elseif type(c) == "function" then
            -- ignore
        end
    end
    Connections = {}
end

-- GUI CREATION
local function createMainGui()
    local Gui = Instance.new("ScreenGui")
    Gui.Name = "DeadHub_OP"
    Gui.IgnoreGuiInset = true
    Gui.ResetOnSpawn = false
    Gui.Parent = PlayerGui

    local Background = Instance.new("Frame")
    Background.Size = UDim2.new(1,0,1,0)
    Background.BackgroundColor3 = CONFIG.Theme.bgStart
    Background.BorderSizePixel = 0
    Background.Parent = Gui

    local Gradient = Instance.new("UIGradient")
    Gradient.Color = ColorSequence.new{
        ColorSequenceKeypoint.new(0, CONFIG.Theme.bgStart),
        ColorSequenceKeypoint.new(1, CONFIG.Theme.bgEnd)
    }
    Gradient.Rotation = 45
    Gradient.Parent = Background

    local Title = Instance.new("TextLabel")
    Title.Size = UDim2.new(0,480,0,72)
    Title.Position = UDim2.new(0.5,-240,0.12,0)
    Title.BackgroundTransparency = 1
    Title.Text = "👑 Dead hub 👑"
    Title.Font = Enum.Font.GothamBlack
    Title.TextScaled = true
    Title.TextColor3 = CONFIG.Theme.text
    Title.TextTransparency = 1
    Title.Parent = Gui

    local Logo = Instance.new("ImageLabel")
    Logo.Size = UDim2.new(0,88,0,88)
    Logo.Position = UDim2.new(0.5,-44,0.38,-44)
    Logo.BackgroundTransparency = 1
    Logo.Image = "rbxassetid://97023202251640"
    Logo.ImageTransparency = 1
    Logo.Parent = Gui

    local LoadingText = Instance.new("TextLabel")
    LoadingText.Size = UDim2.new(0,420,0,44)
    LoadingText.Position = UDim2.new(0.5,-210,0.7,0)
    LoadingText.BackgroundTransparency = 1
    LoadingText.Text = "Inicializando..."
    LoadingText.Font = Enum.Font.GothamBold
    LoadingText.TextSize = 24
    LoadingText.TextColor3 = Color3.fromRGB(255,255,255)
    LoadingText.Parent = Gui

    local BarBG = Instance.new("Frame")
    BarBG.Size = UDim2.new(0,420,0,16)
    BarBG.Position = UDim2.new(0.5,-210,0.78,0)
    BarBG.BackgroundColor3 = Color3.fromRGB(60,0,90)
    BarBG.BorderSizePixel = 0
    BarBG.Parent = Gui

    local Bar = Instance.new("Frame")
    Bar.Size = UDim2.new(0,0,1,0)
    Bar.BackgroundColor3 = CONFIG.Theme.accent
    Bar.BorderSizePixel = 0
    Bar.Parent = BarBG

    local Stroke = Instance.new("UIStroke")
    Stroke.Thickness = 2
    Stroke.Color = CONFIG.Theme.text
    Stroke.Parent = Bar

    local Flash = Instance.new("Frame")
    Flash.Size = UDim2.new(1,0,1,0)
    Flash.BackgroundColor3 = Color3.new(1,1,1)
    Flash.BackgroundTransparency = 1
    Flash.ZIndex = 100
    Flash.Parent = Gui

    return {
        Gui = Gui,
        Background = Background,
        Title = Title,
        Logo = Logo,
        LoadingText = LoadingText,
        BarBG = BarBG,
        Bar = Bar,
        Flash = Flash
    }
end

-- AUDIO (cria e devolve sound)
local function createAndPlayMusic(parent)
    local ok, sound = pcall(function()
        local s = Instance.new("Sound")
        s.Name = "DeadHubMusic"
        s.SoundId = CONFIG.Music.SoundId
        s.Looped = true
        s.Volume = CONFIG.Music.Volume
        s.Parent = parent
        s:Play()
        return s
    end)
    if ok then return sound end
    return nil
end

local function fadeOutSound(sound)
    if not sound then return end
    task.spawn(function()
        while sound.Volume > 0 and sound.Parent do
            sound.Volume = clamp(sound.Volume - CONFIG.Music.FadeStep, 0, 1)
            task.wait(0.04)
        end
        pcall(function() sound:Stop() end)
    end)
end

-- PARTICLE POOL (UI)
local ParticlePool = { pool = {}, _t = 0 }
local function setupParticles(guiParent)
    -- limpar pool anterior
    for _, p in ipairs(ParticlePool.pool) do p:Destroy() end
    ParticlePool.pool = {}
    for i = 1, CONFIG.ParticleCount do
        local p = Instance.new("Frame")
        p.Size = UDim2.new(0, math.random(6, 12), 0, math.random(6, 12))
        p.Position = UDim2.new(math.random(), 0, math.random(), 0)
        p.BackgroundColor3 = CONFIG.Theme.text
        p.BackgroundTransparency = math.random(3, 7) / 10
        p.BorderSizePixel = 0
        p.ZIndex = 3
        p.Parent = guiParent

        local glow = Instance.new("UIStroke")
        glow.Thickness = 1
        glow.Transparency = 0.35
        glow.Color = CONFIG.Theme.text
        glow.Parent = p

        table.insert(ParticlePool.pool, p)
    end
    ParticlePool._t = 0
    -- heartbeat handler adicionado pelo init (veja abaixo)
end

-- LOADER (suave)
local function runLoader()
    local ui = createMainGui()
    local music = createAndPlayMusic(ui.Gui)

    TweenService:Create(ui.Logo, TweenInfo.new(1.4, Enum.EasingStyle.Quint), {ImageTransparency = 0}):Play()
    TweenService:Create(ui.Title, TweenInfo.new(1.0, Enum.EasingStyle.Quint), {TextTransparency = 0}):Play()

    setupParticles(ui.Gui)

    local progress = 0
    local messages = {
        {limit = 25, text = "Carregando configurações"},
        {limit = 50, text = "Carregando módulos"},
        {limit = 75, text = "Carregando UI"},
        {limit = 100, text = "Finalizando"}
    }

    while progress <= 100 do
        local currentMessage = ""
        for _, m in ipairs(messages) do if progress <= m.limit then currentMessage = m.text; break end end
        ui.LoadingText.Text = string.format("%s%s %d%%", currentMessage, string.rep(".", (progress%3)+1), progress)
        ui.Bar.Size = UDim2.new(progress/100, 0, 1, 0)
        progress = progress + (1 + (progress/100)) -- acelera com o tempo
        task.wait(0.03)
    end

    TweenService:Create(ui.Flash, TweenInfo.new(0.35, Enum.EasingStyle.Quad), {BackgroundTransparency = 0.45}):Play()
    TweenService:Create(ui.Logo, TweenInfo.new(0.8, Enum.EasingStyle.Quad), {ImageTransparency = 1}):Play()
    TweenService:Create(ui.LoadingText, TweenInfo.new(0.8, Enum.EasingStyle.Quad), {TextTransparency = 1}):Play()
    TweenService:Create(ui.Title, TweenInfo.new(0.8, Enum.EasingStyle.Quad), {TextTransparency = 1}):Play()
    TweenService:Create(ui.Background, TweenInfo.new(1.0, Enum.EasingStyle.Quad), {BackgroundTransparency = 1}):Play()

    task.wait(0.9)
    TweenService:Create(ui.Flash, TweenInfo.new(0.6), {BackgroundTransparency = 1}):Play()
    task.wait(0.4)

    -- garante desconexão de event handlers ao fechar UI
    disconnectAll()

    -- fade music
    fadeOutSound(music)
    if ui.Gui and ui.Gui.Parent then ui.Gui:Destroy() end
end

-- ESP (otimizado)
local ESP = { enabled = false, billboards = {}, conn = nil, t = 0 }
local function createOrUpdateBillboard(player)
    if player == LocalPlayer then return end
    local char = player.Character
    if not char then return end
    local head = char:FindFirstChild("Head")
    if not head then return end

    local existing = ESP.billboards[player]
    if existing and existing.billboard and existing.billboard.Parent == head then
        local lbl = existing.billboard:FindFirstChild("TextLabel")
        if lbl then lbl.Text = player.Name .. " | " .. player.AccountAge .. " dias" end
        return
    end

    if existing and existing.billboard then existing.billboard:Destroy() end

    local billboard = Instance.new("BillboardGui")
    billboard.Name = "ESP_Billboard"
    billboard.Parent = head
    billboard.Adornee = head
    billboard.Size = UDim2.new(0,160,0,34)
    billboard.StudsOffset = Vector3.new(0,3,0)
    billboard.AlwaysOnTop = true

    local textLabel = Instance.new("TextLabel")
    textLabel.BackgroundTransparency = 1
    textLabel.Size = UDim2.new(1,0,1,0)
    textLabel.Font = Enum.Font.SourceSansBold
    textLabel.TextSize = 14
    textLabel.Text = player.Name .. " | " .. player.AccountAge .. " dias"
    textLabel.TextColor3 = CONFIG.Theme.accent
    textLabel.Parent = billboard

    ESP.billboards[player] = { billboard = billboard }
end

local function removeBillboard(player)
    if ESP.billboards[player] and ESP.billboards[player].billboard then
        ESP.billboards[player].billboard:Destroy()
    end
    ESP.billboards[player] = nil
end

local function toggleESP(state)
    ESP.enabled = state
    if state then
        for _, p in ipairs(Players:GetPlayers()) do createOrUpdateBillboard(p) end
        ESP.conn = RunService.Heartbeat:Connect(function(dt)
            ESP.t = ESP.t + dt
            if ESP.t >= CONFIG.ESPUpdateRate then
                ESP.t = 0
                for _, p in ipairs(Players:GetPlayers()) do
                    if p.Character and p.Character:FindFirstChild("Head") then createOrUpdateBillboard(p) end
                end
            end
        end)
        addConn(ESP.conn)
        Players.PlayerAdded:Connect(function(p) createOrUpdateBillboard(p) end)
        Players.PlayerRemoving:Connect(function(p) removeBillboard(p) end)
    else
        if ESP.conn then ESP.conn:Disconnect() end
        for p,_ in pairs(ESP.billboards) do removeBillboard(p) end
        ESP.billboards = {}
        ESP.conn = nil
    end
end

-- HEADSIT (seguindo checks)
local function headsitOnPlayerSafely(target)
    if not target or not target.Character then return false end
    local localChar = LocalPlayer.Character
    local root = localChar and localChar:FindFirstChild("HumanoidRootPart")
    local targetHead = target.Character:FindFirstChild("Head")
    if not root or not targetHead then return false end

    -- remove welds antigos
    for _, c in ipairs(root:GetChildren()) do if c:IsA("WeldConstraint") then pcall(function() c:Destroy() end) end end
    root.CFrame = targetHead.CFrame * CFrame.new(0,2.2,0)
    local weld = Instance.new("WeldConstraint")
    weld.Part0 = root
    weld.Part1 = targetHead
    weld.Parent = root

    local hum = localChar:FindFirstChildOfClass("Humanoid")
    if hum then hum.Sit = true end
    return true
end

local function removeHeadsit()
    local char = LocalPlayer.Character
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    if root then
        for _, v in ipairs(root:GetChildren()) do
            if v:IsA("WeldConstraint") then pcall(function() v:Destroy() end) end
        end
    end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if hum then hum.Sit = false end
end

-- ULTIMATE NOCIP (rework leve)
local Noclip = { conn = nil, enabled = false }
local function setCharCollision(character, canCollide)
    if not character then return end
    for _, part in ipairs(character:GetDescendants()) do
        if part:IsA("BasePart") then
            part.CanCollide = canCollide
            pcall(function() part.Anchored = false end)
        end
    end
end

local function enableNoclip(state)
    Noclip.enabled = state
    if state then
        if Noclip.conn then Noclip.conn:Disconnect() end
        Noclip.conn = RunService.Heartbeat:Connect(function()
            local char = LocalPlayer.Character
            if char then
                setCharCollision(char, false)
                local root = char:FindFirstChild("HumanoidRootPart")
                if root and root.Position.Y < -500 then root.CFrame = CFrame.new(0,100,0) end
            end
        end)
        addConn(Noclip.conn)
    else
        if Noclip.conn then Noclip.conn:Disconnect(); Noclip.conn = nil end
        if LocalPlayer.Character then setCharCollision(LocalPlayer.Character, true) end
    end
end

-- SAFE REMOTE INVOKE (verifica existencia)
local function safeInvokeRemote(remoteName, ...)
    if not ReplicatedStorage:FindFirstChild("Remotes") then return false end
    local rem = ReplicatedStorage.Remotes:FindFirstChild(remoteName)
    if rem and rem.InvokeServer then
        local ok, res = pcall(function() return rem:InvokeServer(...) end)
        return ok, res
    end
    return false
end

-- EXTERNAL UI LOAD (mantém loadstring envolto em pcall)
local function loadExternalUILib()
    local success, lib = pcall(function()
        return loadstring(game:HttpGet("https://raw.githubusercontent.com/nickainzn/Uimeuamg/main/HubUi.lua"))()
    end)
    if success and lib then return lib end
    return nil
end

-- INICIALIZAÇÃO PRINCIPAL
task.spawn(function()
    -- opcional: key check local (não bloqueante em ambientes sem Prompt)
    -- mantemos leve: se executor não suportar Prompt, passa automaticamente
    local passed = true
    local ok, res = pcall(function()
        -- muitos executores não têm PromptInput; não bloqueamos
        return false
    end)
    --Executa o loader visual
    runLoader()

    -- registra heartbeat para partículas (adiciona à lista de conexões para cleanup)
    local heartbeatConn = RunService.Heartbeat:Connect(function(dt)
        if ParticlePool and ParticlePool.pool and #ParticlePool.pool > 0 then
            ParticlePool._t = ParticlePool._t + dt
            if ParticlePool._t >= CONFIG.ParticleUpdateRate then
                ParticlePool._t = 0
                for _, p in ipairs(ParticlePool.pool) do
                    if p and p.Parent then
                        local x = clamp(p.Position.X.Scale + (math.random(-8,8)/100), 0, 1)
                        local y = clamp(p.Position.Y.Scale + (math.random(-8,8)/100), 0, 1)
                        local s = math.random(4, 12)
                        p:TweenSizeAndPosition(UDim2.new(0, s, 0, s), UDim2.new(x,0,y,0), Enum.EasingDirection.InOut, Enum.EasingStyle.Sine, 1.6, true)
                        p.BackgroundTransparency = clamp(math.random(2,7)/10, 0, 1)
                    end
                end
            end
        end
    end)
    addConn(heartbeatConn)

    -- carrega UI lib externa (se disponível)
    local uiLib = loadExternalUILib()
    if not uiLib then
        StarterGui:SetCore("SendNotification", { Title = "Dead Hub", Text = "UI externa não carregada — fallback.", Duration = 3 })
        -- fallback: não aborta; podes criar interface local se quiseres
        return
    end

    -- instancia janela (usando pcall)
    local okW, WindowRef = pcall(function()
        return uiLib:MakeWindow({ Title = "Dead hub | OP Interface", SubTitle = "by Dunkheilt", SaveFolder = "DeadHubOP" })
    end)
    if not okW or not WindowRef then return end

    -- cria tabs básicos (mantive a tua estrutura)
    local Tab1 = WindowRef:MakeTab({"Credits","info"})
    local Tab2 = WindowRef:MakeTab({"Fun","fun"})
    local Tab3 = WindowRef:MakeTab({"Avatar","shirt"})

    Tab1:AddParagraph({"Info","Dead hub OP - Versão segura e estável (client-side)"})

    -- Fun tab: headsit textBox + button
    local selectedPlayerName
    Tab2:AddTextBox({ Name = "Nome do Jogador", Description = "Digite parte do nome", PlaceholderText = "ex: lo", Callback = function(Value)
        if not Value or Value == "" then selectedPlayerName = nil; return end
        local partial = Value:lower()
        for _, p in pairs(Players:GetPlayers()) do
            if p ~= LocalPlayer and p.Name:lower():sub(1,#partial) == partial then
                selectedPlayerName = p.Name
                pcall(function() StarterGui:SetCore("SendNotification", { Title="Dead Hub", Text = p.Name.." selecionado", Duration=3 }) end)
                return
            end
        end
        pcall(function() StarterGui:SetCore("SendNotification", { Title="Dead Hub", Text = "Nenhum jogador encontrado", Duration=3 }) end)
    end })

    local headsitActive = false
    Tab2:AddButton({ Name = "Headsit Toggle", Callback = function()
        if not selectedPlayerName then pcall(function() StarterGui:SetCore("SendNotification", { Title="Headsit", Text = "Seleciona um jogador.", Duration=3 }) end); return end
        if not headsitActive then
            local target = Players:FindFirstChild(selectedPlayerName)
            if target then local ok = pcall(headsitOnPlayerSafely, target) if ok then headsitActive = true end end
        else
            removeHeadsit(); headsitActive = false
        end
    end })

    -- Speed / Jump / Gravity sliders
    Tab2:AddSlider({ Name="Speed Player", Increase=1, MinValue=3, MaxValue=300, Default=16, Callback=function(Value)
        local char = LocalPlayer.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if hum then hum.WalkSpeed = clamp(Value,3,300) end
    end })

    Tab2:AddSlider({ Name="JumpPower", Increase=1, MinValue=30, MaxValue=500, Default=50, Callback=function(Value)
        local char = LocalPlayer.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if hum then hum.JumpPower = clamp(Value,30,500) end
    end })

    Tab2:AddSlider({ Name="Gravity", Increase=0.1, MinValue=0, MaxValue=1000, Default=196.2, Callback=function(Value)
        workspace.Gravity = clamp(Value,0,1000)
    end })

    -- Infinite Jump
    local InfiniteJumpEnabled = false
    UserInputService.JumpRequest:Connect(function()
        if InfiniteJumpEnabled then
            local char = LocalPlayer.Character
            if char and char:FindFirstChildOfClass("Humanoid") then
                char.Humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
            end
        end
    end)
    Tab2:AddToggle({ Name="Infinite Jump", Default=false, Callback=function(v) InfiniteJumpEnabled = v end })

    -- Noclip toggle
    Tab2:AddToggle({ Name="Ultimate Noclip", Default=false, Callback=function(v) enableNoclip(v) end })

    -- ESP toggle
    Tab2:AddToggle({ Name="ESP Ativado", Default=false, Callback=function(v) toggleESP(v) end })

    -- Fly GUI external (pcall)
    Tab2:AddButton({ Name="Ativar Fly GUI (externo)", Callback=function()
        local okf = pcall(function() loadstring(game:HttpGet("https://rawscripts.net/raw/Universal-Script-Fly-gui-v3-30439"))() end)
        pcall(function() StarterGui:SetCore("SendNotification", { Title = okf and "Fly GUI" or "Erro", Text = okf and "Fly GUI carregado." or "Falha ao carregar Fly GUI.", Duration = 4 }) end)
    end })

    -- Avatar tab: dropdown + copy avatar (safeInvokeRemote usado)
    Tab3:AddDropdown({ Name="Players List", Options=(function()
        local t = {}
        for _, p in pairs(Players:GetPlayers()) do if p.Name ~= LocalPlayer.Name then table.insert(t,p.Name) end end
        return t
    end)(), Callback=function(val) _G.selectedCopy = val end })

    Tab3:AddButton({ Name="Copy Avatar", Callback=function()
        local targetName = _G.selectedCopy
        if not targetName then pcall(function() StarterGui:SetCore("SendNotification",{Title="Avatar",Text="Seleciona um jogador.",Duration=3}) end); return end
        local target = Players:FindFirstChild(targetName)
        if not target or not target.Character then return end
        pcall(function()
            local THum = target.Character:FindFirstChildOfClass("Humanoid")
            if THum then
                local desc = THum:GetAppliedDescription()
                safeInvokeRemote("ChangeCharacterBody", {desc.Torso, desc.RightArm, desc.LeftArm, desc.RightLeg, desc.LeftLeg, desc.Head})
                pcall(function() StarterGui:SetCore("SendNotification",{Title="Avatar",Text="Pedido de cópia enviado (se disponível).",Duration=3}) end)
            end
        end)
    end })

    -- Admin placeholder (seguro)
    Tab1:AddSection({"Admin (safe)"})
    Tab1:AddButton({ Name="Request Server Kick (placeholder)", Callback=function()
        pcall(function() StarterGui:SetCore("SendNotification",{Title="Admin",Text="Kick não implementado client-side por segurança.",Duration=5}) end)
    end })

    -- final message
    print("✅ Dead hub OP inicializado (client-side).")
end)
