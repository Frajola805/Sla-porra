local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

local Window = Rayfield:CreateWindow({
   Name = "Combat Hub",
   Icon = 0,
   LoadingTitle = "Carregando. . .",
   LoadingSubtitle = "by Richard",
   ShowText = "Hub",
   Theme = "AmberGlow",
   ToggleUIKeybind = "Z",
   DisableRayfieldPrompts = false,
   DisableBuildWarnings = false,
   ConfigurationSaving = {
      Enabled = true,
      FolderName = nil,
      FileName = "richardHub"
   },
   Discord = {
      Enabled = true,
      Invite = "awSFnVnU",
      RememberJoins = true
   },
   KeySystem = false,
   KeySettings = {
      Title = "Untitled",
      Subtitle = "Key System",
      Note = "No method of obtaining the key is provided",
      FileName = "Key",
      SaveKey = true,
      GrabKeyFromSite = false,
      Key = {"5120"}
   }
})

local Tab = Window:CreateTab("Aimbot", 4483362458)

-- Variáveis do aimbot
local circle, aiming = nil, false
local circleRadius = 70
local circleOffset = Vector2.new(0, 0)
local lockSmoothness = 0.3 -- Suavidade ao travar no alvo
local unlockSmoothness = 0.5 -- Suavidade ao sair do alvo (VALOR MAIOR = MAIS FÁCIL DE TIRAR)
local connection = nil
local currentTarget = nil
local isLocked = false
local lastTargetTime = 0
local transitionProgress = 0
local lastCFrame = nil
local aimPart = "Head" -- Parte padrão para mirar (Head ou Torso)
local useToggleMode = true -- Modo toggle ou hold

-- Funções do aimbot
local function isEnemy(plr)
    local player = game.Players.LocalPlayer
    if plr.Team and player.Team then 
        return plr.Team ~= player.Team 
    else 
        return true
    end 
end

local function isVisible(part)
    local player = game.Players.LocalPlayer
    local camera = workspace.CurrentCamera
    local origin = camera.CFrame.Position 
    local direction = (part.Position - origin).Unit * 1000 
    local ray = Ray.new(origin, direction) 
    local hit = workspace:FindPartOnRay(ray, player.Character, false, true) 
    return hit and (hit:IsDescendantOf(part.Parent) or hit == part) 
end

-- Função para obter a parte do corpo do alvo
local function getAimPart(character)
    if not character then return nil end
    
    if aimPart == "Head" then
        return character:FindFirstChild("Head")
    elseif aimPart == "Torso" then
        return character:FindFirstChild("UpperTorso") or character:FindFirstChild("Torso") or character:FindFirstChild("Head")
    end
    
    return character:FindFirstChild("Head") -- Fallback
end

local function getTarget()
    local player = game.Players.LocalPlayer
    local camera = workspace.CurrentCamera
    local players = game.Players:GetPlayers() 
    local closest = nil 
    local shortest = math.huge

    for _, plr in ipairs(players) do
        if plr ~= player and plr.Character and getAimPart(plr.Character) and isEnemy(plr) then
            local aimPartInstance = getAimPart(plr.Character)
            local pos, onScreen = camera:WorldToViewportPoint(aimPartInstance.Position)
            if onScreen then
                local dist = (Vector2.new(pos.X, pos.Y) - (Vector2.new(camera.ViewportSize.X, camera.ViewportSize.Y)/2 + circleOffset)).Magnitude
                if dist <= circleRadius and dist < shortest and isVisible(aimPartInstance) then
                    shortest = dist
                    closest = plr
                end
            end
        end
    end

    return closest
end

-- Sistema de suavidade para entrada no raio
local function applyEntrySmoothness(currentCFrame, targetPosition, smoothFactor, isEntering)
    if smoothFactor <= 0 then
        return CFrame.new(currentCFrame.Position, targetPosition)
    end
    
    -- Para entrada (travar no alvo), usa suavidade normal
    if isEntering then
        local targetDirection = (targetPosition - currentCFrame.Position).Unit
        local currentDirection = currentCFrame.LookVector
        local smoothedDirection = currentDirection:Lerp(targetDirection, 1 - smoothFactor)
        return CFrame.new(currentCFrame.Position, currentCFrame.Position + smoothedDirection)
    else
        -- Para saída (liberar do alvo), usa suavidade aumentada para facilitar
        local mouse = game.Players.LocalPlayer:GetMouse()
        local mouseDirection = (Vector3.new(mouse.X, mouse.Y, 0) - currentCFrame.Position).Unit
        local currentDirection = currentCFrame.LookVector
        local smoothedDirection = currentDirection:Lerp(mouseDirection, 1 - (smoothFactor * 1.5)) -- 50% mais suave
        return CFrame.new(currentCFrame.Position, currentCFrame.Position + smoothedDirection)
    end
end

local function startAimbot()
    local player = game.Players.LocalPlayer
    local camera = workspace.CurrentCamera
    local runService = game:GetService("RunService")
    
    -- Criar círculo de visualização
    circle = Drawing.new("Circle") 
    circle.Radius = circleRadius 
    circle.Filled = false 
    circle.Color = Color3.fromRGB(255, 0, 0) 
    circle.Visible = true 
    circle.Thickness = 1
    circle.Position = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2) + circleOffset
    
    -- Salvar CFrame inicial
    lastCFrame = camera.CFrame
    
    -- Loop do aimbot
    connection = runService.RenderStepped:Connect(function()
        if circle then
            circle.Position = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2) + circleOffset
            circle.Radius = circleRadius
        end

        if aiming then
            local newTarget = getTarget()
            
            if newTarget and newTarget.Character and getAimPart(newTarget.Character) then
                -- Novo alvo encontrado
                if currentTarget ~= newTarget then
                    transitionProgress = 0
                    currentTarget = newTarget
                end
                
                local aimPartInstance = getAimPart(newTarget.Character)
                local aimPos = aimPartInstance.Position
                lastTargetTime = tick()
                
                if not isLocked then
                    -- Entrando no alvo - aplicar suavidade de entrada
                    transitionProgress = math.min(transitionProgress + (1 - lockSmoothness) * 0.1, 1)
                    camera.CFrame = applyEntrySmoothness(camera.CFrame, aimPos, lockSmoothness, true)
                    
                    if transitionProgress >= 0.9 then
                        isLocked = true
                    end
                else
                    -- Já travado - seguir normalmente
                    camera.CFrame = applyEntrySmoothness(camera.CFrame, aimPos, 0.1, true) -- Pouca suavidade quando já travado
                end
            else
                -- Nenhum alvo encontrado
                if isLocked then
                    -- Saindo do alvo - aplicar suavidade de saída
                    transitionProgress = math.max(transitionProgress - (1 - unlockSmoothness) * 0.1, 0)
                    
                    if transitionProgress <= 0.1 then
                        isLocked = false
                        currentTarget = nil
                    else
                        -- Continuar mirando no último alvo conhecido com suavidade de saída
                        if currentTarget and currentTarget.Character and getAimPart(currentTarget.Character) then
                            local aimPartInstance = getAimPart(currentTarget.Character)
                            local aimPos = aimPartInstance.Position
                            camera.CFrame = applyEntrySmoothness(camera.CFrame, aimPos, unlockSmoothness, false)
                        end
                    end
                elseif tick() - lastTargetTime > 0.3 then
                    currentTarget = nil
                end
            end
            
            lastCFrame = camera.CFrame
        else
            isLocked = false
            currentTarget = nil
            transitionProgress = 0
        end
    end)
end

local function stopAimbot()
    aiming = false
    isLocked = false
    currentTarget = nil
    transitionProgress = 0
    if connection then
        connection:Disconnect()
        connection = nil
    end
    if circle then
        circle:Remove()
        circle = nil
    end
end

-- Keybind para ativar enquanto segura Q
local Keybind = Tab:CreateKeybind({
   Name = "Ativar Ao Segurar",
   CurrentKeybind = "Q",
   HoldToInteract = true,
   Flag = "AimbotHoldKeybind",
   Callback = function(Keybind)
      if useToggleMode then return end -- Ignora se estiver no modo toggle
      
      if Keybind then
         -- Tecla pressionada - ativar aimbot
         aiming = true
         startAimbot()
      else
         -- Tecla solta - desativar aimbot
         stopAimbot()
      end
   end,
})

-- Toggle do Aimbot (modo tradicional)
local Toggle = Tab:CreateToggle({
   Name = "Modo Toggle (Ligar/Desligar)",
   CurrentValue = false,
   Flag = "AimbotToggle",
   Callback = function(Value)
      useToggleMode = true
      aiming = Value
      if Value then
         startAimbot()
      else
         stopAimbot()
      end
   end,
})

-- Toggle para escolher entre modo toggle ou hold
local ModeToggle = Tab:CreateToggle({
   Name = "Usar Modo Segurar (Ao Invés de Toggle)",
   CurrentValue = false,
   Flag = "AimbotMode",
   Callback = function(Value)
      useToggleMode = not Value
      
      -- Se mudar para modo hold, desativa o toggle atual
      if not useToggleMode then
         if aiming then
            stopAimbot()
            Rayfield.Flags["AimbotToggle"] = false
         end
      end
   end,
})

-- Dropdown para selecionar a parte do corpo
local Dropdown = Tab:CreateDropdown({
   Name = "Parte do Corpo para Mirar",
   Options = {"Head", "Torso"},
   CurrentOption = {"Head"},
   MultipleOptions = false,
   Flag = "AimPart",
   Callback = function(Option)
      aimPart = Option[1]
   end,
})

-- Slider para suavidade de ENTRADA no raio (travar no alvo)
local LockSmoothnessSlider = Tab:CreateSlider({
   Name = "Suavidade ao Travar",
   Range = {0, 0.9},
   Increment = 0.05,
   Suffix = "",
   CurrentValue = 0.3,
   Flag = "LockSmoothness",
   Callback = function(Value)
      lockSmoothness = Value
   end,
})

-- Slider para suavidade de SAÍDA do raio (sair do alvo) - VALOR MAIOR = MAIS FÁCIL DE TIRAR
local UnlockSmoothnessSlider = Tab:CreateSlider({
   Name = "Facilidade para Sair",
   Range = {0.1, 0.9},
   Increment = 0.05,
   Suffix = "",
   CurrentValue = 0.5,
   Flag = "UnlockSmoothness",
   Callback = function(Value)
      unlockSmoothness = Value
   end,
})

-- Slider para tamanho do círculo
local SizeSlider = Tab:CreateSlider({
   Name = "Tamanho do Aimbot",
   Range = {10, 200},
   Increment = 5,
   Suffix = "pixels",
   CurrentValue = 70,
   Flag = "AimbotSize",
   Callback = function(Value)
      circleRadius = Value
   end,
})

-- Slider para espessura do círculo
local ThicknessSlider = Tab:CreateSlider({
   Name = "Espessura do Círculo",
   Range = {1, 10},
   Increment = 1,
   Suffix = "px",
   CurrentValue = 1,
   Flag = "CircleThickness",
   Callback = function(Value)
      if circle then
         circle.Thickness = Value
      end
   end,
})

-- Botões para ajustar posição do círculo
local ButtonSection = Tab:CreateSection("Ajustar Posição do Círculo")

local UpButton = Tab:CreateButton({
   Name = "Mover para Cima",
   Callback = function()
      circleOffset = circleOffset + Vector2.new(0, -5)
   end,
})

local DownButton = Tab:CreateButton({
   Name = "Mover para Baixo",
   Callback = function()
      circleOffset = circleOffset + Vector2.new(0, 5)
   end,
})

local LeftButton = Tab:CreateButton({
   Name = "Mover para Esquerda",
   Callback = function()
      circleOffset = circleOffset + Vector2.new(-5, 0)
   end,
})

local RightButton = Tab:CreateButton({
   Name = "Mover para Direita",
   Callback = function()
      circleOffset = circleOffset + Vector2.new(5, 0)
   end,
})

local ResetButton = Tab:CreateButton({
   Name = "Resetar Posição",
   Callback = function()
      circleOffset = Vector2.new(0, 0)
   end,
})

-- Seção de configurações de cor
local ColorSection = Tab:CreateSection("Configurações de Cor")

local ColorPicker = Tab:CreateColorPicker({
   Name = "Cor do Círculo",
   Color = Color3.fromRGB(255, 0, 0),
   Flag = "CircleColor",
   Callback = function(Value)
      if circle then
         circle.Color = Value
      end
   end
})

-- Toggle para preencher o círculo
local FillToggle = Tab:CreateToggle({
   Name = "Preencher Círculo",
   CurrentValue = false,
   Flag = "CircleFilled",
   Callback = function(Value)
      if circle then
         circle.Filled = Value
      end
   end,
})

-- Toggle para verificação de visibilidade
local VisibilityToggle = Tab:CreateToggle({
   Name = "Verificar Visibilidade",
   CurrentValue = true,
   Flag = "VisibilityCheck",
   Callback = function(Value)
      -- Esta configuração será usada na função isVisible
   end,
})

-- Toggle para detecção de time
local TeamToggle = Tab:CreateToggle({
   Name = "Ignorar Aliados",
   CurrentValue = true,
   Flag = "IgnoreTeam",
   Callback = function(Value)
      -- Esta configuração será usada na função isEnemy
   end,
})

-- Atualizar funções para considerar as configurações
local originalIsEnemy = isEnemy
isEnemy = function(plr)
    if not Rayfield.Flags["IgnoreTeam"] then
        return true
    end
    return originalIsEnemy(plr)
end

local originalIsVisible = isVisible
isVisible = function(part)
    if not Rayfield.Flags["VisibilityCheck"] then
        return true
    end
    return originalIsVisible(part)
end

-- Informações sobre a suavidade
local InfoLabel1 = Tab:CreateLabel("Suavidade ao Travar: 0 = Imediato, 0.9 = Muito Suave")
local InfoLabel2 = Tab:CreateLabel("Facilidade para Sair: 0.1 = Difícil, 0.9 = Muito Fácil")
local ModeLabel = Tab:CreateLabel("Modo Atual: Toggle")
local PartLabel = Tab:CreateLabel("Mirando em: Head")
local TargetLabel = Tab:CreateLabel("Alvo Atual: Nenhum")
local LockLabel = Tab:CreateLabel("Travado: Não")
local TransitionLabel = Tab:CreateLabel("Transição: 0%")

-- Atualizar labels
game:GetService("RunService").RenderStepped:Connect(function()
    if currentTarget then
        TargetLabel:Set("Alvo Atual: " .. currentTarget.Name)
    else
        TargetLabel:Set("Alvo Atual: Nenhum")
    end
    
    ModeLabel:Set("Modo Atual: " .. (useToggleMode and "Toggle" or "Segurar (Q)"))
    PartLabel:Set("Mirando em: " .. aimPart)
    LockLabel:Set("Travado: " .. (isLocked and "Sim" or "Não"))
    TransitionLabel:Set("Transição: " .. math.floor(transitionProgress * 100) .. "%")
end)

-- Função para quando o jogo fecha ou script é interrompido
game:GetService("Players").LocalPlayer.CharacterRemoving:Connect(function()
    stopAimbot()
end)

game:GetService("Players").LocalPlayer:GetPropertyChangedSignal("UserId"):Connect(function()
    stopAimbot()
end)
