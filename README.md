-- ========================================
-- NOREPHINA - REPLAY GHOST V7 (RODAS SEPARADAS)
-- ========================================
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()
local Window = Rayfield:CreateWindow({
    Name = "Norephina - Replay Ghost V7",
    LoadingTitle = "Car Dealership Tycoon",
})

local Tab = Window:CreateTab("👻 Drift Sincronizado", 4483362458)

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer

local recording = false
local playing = false
local looping = false
local replayData = {}
local recordConn = nil
local playConn = nil
local ghostCar = nil
local ghostWheelFolder = nil
local currentCar = nil
local recordStart = 0
local playbackSpeed = 1
local ghostWheels = {} -- {part = Instance, rel = CFrame}

-- ==================== ACHAR O CARRO ====================
local function FindMyCar()
    local char = LocalPlayer.Character
    if not char or not char:FindFirstChild("Humanoid") then return nil end

    for _, car in pairs(workspace:GetDescendants()) do
        if car:IsA("Model") and car:FindFirstChild("VehicleSeat") then
            local seat = car:FindFirstChild("VehicleSeat")
            if seat and seat.Occupant and seat.Occupant == char.Humanoid then
                return car
            end
        end

        if car:IsA("Model") then
            local driveSeat = car:FindFirstChild("DriveSeat", true)
            if driveSeat and driveSeat:IsA("BasePart") then
                local occupant = driveSeat:FindFirstChild("Occupant")
                if occupant and occupant.Value == char.Humanoid then
                    return car
                end
                local root = char:FindFirstChild("HumanoidRootPart")
                if root and (driveSeat.Position - root.Position).Magnitude < 6 then
                    return car
                end
            end
        end
    end
    return nil
end

local function GetRoot(car)
    if not car then return nil end
    return car:FindFirstChild("Chassis")
        or car:FindFirstChild("Base")
        or car:FindFirstChild("Body")
        or car.PrimaryPart
        or car:FindFirstChildWhichIsA("BasePart")
end

local function IsWheel(obj)
    if not obj:IsA("BasePart") then return false end
    local n = string.lower(obj.Name)
    return n:find("wheel")
        or n:find("tire")
        or n:find("tyre")
        or n:find("rim")
        or n:find("roda")
        or n == "fl" or n == "fr" or n == "rl" or n == "rr"
        or n == "wl" or n == "wr"
end

-- ==================== GRAVAÇÃO ====================
local function StartRecording()
    if recording or playing then
        Rayfield:Notify({Title = "⚠️", Content = "Já está gravando ou reproduzindo", Duration = 3})
        return
    end

    currentCar = FindMyCar()
    if not currentCar then
        Rayfield:Notify({Title = "❌ Erro", Content = "Entre em um carro primeiro!", Duration = 3})
        return
    end
    if not GetRoot(currentCar) then
        Rayfield:Notify({Title = "❌ Erro", Content = "Não achei a peça principal", Duration = 3})
        return
    end

    replayData = {}
    recording = true
    recordStart = tick()

    recordConn = RunService.Heartbeat:Connect(function()
        if not recording or not currentCar or not currentCar.Parent then return end
        local r = GetRoot(currentCar)
        if r then
            table.insert(replayData, { t = tick() - recordStart, cf = r.CFrame })
        end
    end)

    Rayfield:Notify({Title = "🔴 GRAVANDO", Content = "Faça o drift!", Duration = 4})
end

local function StopRecording()
    if not recording then return end
    recording = false
    if recordConn then recordConn:Disconnect() recordConn = nil end
    local dur = (#replayData > 0) and replayData[#replayData].t or 0
    Rayfield:Notify({
        Title = "⏹️ PARADO",
        Content = #replayData .. " frames | " .. string.format("%.1f", dur) .. "s",
        Duration = 4
    })
end

-- ==================== LIMPAR GHOST ====================
local function DestroyGhost()
    if playConn then playConn:Disconnect() playConn = nil end
    playing = false
    looping = false

    if ghostCar then
        pcall(function() ghostCar:Destroy() end)
        ghostCar = nil
    end
    if ghostWheelFolder then
        pcall(function() ghostWheelFolder:Destroy() end)
        ghostWheelFolder = nil
    end
    ghostWheels = {}
end

-- ==================== CRIAR GHOST ====================
local function CreateGhost()
    DestroyGhost()

    currentCar = currentCar or FindMyCar()
    if not currentCar then
        Rayfield:Notify({Title = "❌", Content = "Não achei o carro", Duration = 3})
        return nil
    end

    local rootOrig = GetRoot(currentCar)
    if not rootOrig then return nil end

    -- Clona o corpo do carro
    local ok, clone = pcall(function() return currentCar:Clone() end)
    if not ok or not clone then
        Rayfield:Notify({Title = "❌", Content = "Falha ao clonar", Duration = 3})
        return nil
    end
    clone.Name = "NorephinaGhost"

    for _, obj in pairs(clone:GetDescendants()) do
        if obj:IsA("BasePart") then
            obj.Anchored = true
            obj.CanCollide = false
            obj.CanQuery = false
            obj.CanTouch = false
            obj.Massless = true
        elseif obj:IsA("VehicleSeat") or obj:IsA("Seat") then
            obj.Disabled = true
        elseif obj:IsA("Sound") or obj:IsA("Fire") or obj:IsA("Smoke")
            or obj:IsA("Sparkles") or obj:IsA("ParticleEmitter") then
            pcall(function() obj:Destroy() end)
        elseif obj:IsA("Script") or obj:IsA("LocalScript") or obj:IsA("ModuleScript") then
            pcall(function() obj:Destroy() end)
        end
    end

    local hum = clone:FindFirstChildOfClass("Humanoid")
    if hum then pcall(function() hum:Destroy() end) end

    local rootGhost = GetRoot(clone)
    if rootGhost then clone.PrimaryPart = rootGhost end
    clone.Parent = workspace
    ghostCar = clone

    -- Pasta separada só pras rodas (no workspace, não dentro do clone)
    local folder = Instance.new("Folder")
    folder.Name = "NorephinaGhostWheels"
    folder.Parent = workspace
    ghostWheelFolder = folder

    local added = 0

    -- Copia CADA roda do original como peça independente
    for _, obj in pairs(currentCar:GetDescendants()) do
        if IsWheel(obj) then
            local wOk, wClone = pcall(function() return obj:Clone() end)
            if wOk and wClone then
                -- limpa filhos problemáticos mas mantém mesh
                for _, ch in pairs(wClone:GetDescendants()) do
                    if ch:IsA("Script") or ch:IsA("LocalScript") or ch:IsA("Sound") then
                        pcall(function() ch:Destroy() end)
                    end
                end

                wClone.Name = "GW_" .. obj.Name
                wClone.Anchored = true
                wClone.CanCollide = false
                wClone.CanQuery = false
                wClone.CanTouch = false
                wClone.Massless = true
                wClone.Transparency = 0
                wClone.LocalTransparencyModifier = 0

                -- se for MeshPart, garante visível
                if wClone:IsA("MeshPart") then
                    wClone.TextureID = obj.TextureID
                end

                local rel = rootOrig.CFrame:ToObjectSpace(obj.CFrame)
                wClone.CFrame = rootOrig.CFrame * rel
                wClone.Parent = folder

                table.insert(ghostWheels, { part = wClone, rel = rel })
                added = added + 1
            end
        end
    end

    -- Fallback: se IsWheel não pegou, usa tamanho
    if added == 0 then
        for _, obj in pairs(currentCar:GetDescendants()) do
            if obj:IsA("BasePart") and obj.Size.Y < 2.5 and obj.Size.X > 1.0 and obj.Size.Z > 1.0
                and (obj.Position - rootOrig.Position).Magnitude < 12 then
                local wClone = obj:Clone()
                wClone.Name = "GW_Fallback"
                wClone.Anchored = true
                wClone.CanCollide = false
                wClone.Transparency = 0
                wClone.LocalTransparencyModifier = 0
                local rel = rootOrig.CFrame:ToObjectSpace(obj.CFrame)
                wClone.CFrame = rootOrig.CFrame * rel
                wClone.Parent = folder
                table.insert(ghostWheels, { part = wClone, rel = rel })
                added = added + 1
            end
        end
    end

    -- Último recurso: cria 4 cilindros pretos nas posições das rodas detectadas
    if added == 0 then
        local offsets = {
            CFrame.new(-1.5, -0.6, 2.2),
            CFrame.new(1.5, -0.6, 2.2),
            CFrame.new(-1.5, -0.6, -2.2),
            CFrame.new(1.5, -0.6, -2.2),
        }
        for i, off in ipairs(offsets) do
            local cyl = Instance.new("Part")
            cyl.Name = "GW_Fake" .. i
            cyl.Shape = Enum.PartType.Cylinder
            cyl.Size = Vector3.new(0.6, 1.4, 1.4)
            cyl.Color = Color3.fromRGB(20, 20, 20)
            cyl.Material = Enum.Material.SmoothPlastic
            cyl.Anchored = true
            cyl.CanCollide = false
            cyl.CFrame = rootOrig.CFrame * off * CFrame.Angles(0, 0, math.rad(90))
            cyl.Parent = folder
            table.insert(ghostWheels, { part = cyl, rel = off * CFrame.Angles(0, 0, math.rad(90)) })
            added = added + 1
        end
    end

    Rayfield:Notify({
        Title = "👻 Ghost criado",
        Content = "Rodas independentes: " .. added,
        Duration = 4
    })

    return clone
end

local function UpdateGhostWheels()
    if not ghostCar or not ghostCar.Parent then return end
    local root = GetRoot(ghostCar)
    if not root then return end

    for _, w in ipairs(ghostWheels) do
        if w.part and w.part.Parent then
            pcall(function()
                w.part.CFrame = root.CFrame * w.rel
                w.part.Transparency = 0
                w.part.LocalTransparencyModifier = 0
            end)
        end
    end
end

-- ==================== REPRODUZIR ====================
local function PlayReplay(isLoop)
    if recording then
        Rayfield:Notify({Title = "⚠️", Content = "Pare a gravação primeiro!", Duration = 3})
        return
    end
    if #replayData < 10 then
        Rayfield:Notify({Title = "❌", Content = "Grave mais um pouco", Duration = 3})
        return
    end
    if playing then
        Rayfield:Notify({Title = "⚠️", Content = "Já está reproduzindo", Duration = 2})
        return
    end

    local ghost = CreateGhost()
    if not ghost then return end

    pcall(function() ghost:PivotTo(replayData[1].cf) end)
    UpdateGhostWheels()

    playing = true
    looping = isLoop == true
    local startPlay = tick()
    local totalTime = replayData[#replayData].t

    Rayfield:Notify({
        Title = looping and "🔄 LOOP" or "▶️ REPLAY",
        Content = string.format("%.1f", totalTime) .. "s | Rodas: " .. #ghostWheels,
        Duration = 4
    })

    playConn = RunService.Heartbeat:Connect(function()
        if not playing or not ghost or not ghost.Parent then
            if playConn then playConn:Disconnect() playConn = nil end
            playing = false
            looping = false
            return
        end

        local elapsed = (tick() - startPlay) * playbackSpeed

        if elapsed >= totalTime then
            if looping then
                startPlay = tick()
                elapsed = 0
                pcall(function() ghost:PivotTo(replayData[1].cf) end)
            else
                pcall(function() ghost:PivotTo(replayData[#replayData].cf) end)
                UpdateGhostWheels()
                playing = false
                if playConn then playConn:Disconnect() playConn = nil end
                Rayfield:Notify({Title = "✅ FIM", Content = "Replay terminou", Duration = 3})
                return
            end
        end

        local i = 1
        while i < #replayData and replayData[i].t < elapsed do
            i = i + 1
        end

        local prev = replayData[math.max(1, i - 1)]
        local nextF = replayData[i]
        local alpha = 0
        if nextF.t > prev.t then
            alpha = math.clamp((elapsed - prev.t) / (nextF.t - prev.t), 0, 1)
        end

        pcall(function()
            ghost:PivotTo(prev.cf:Lerp(nextF.cf, alpha))
        end)
        UpdateGhostWheels()
    end)
end

local function StopReplay()
    DestroyGhost()
    Rayfield:Notify({Title = "⏹️", Content = "Ghost e rodas removidos", Duration = 2})
end

local function ClearReplay()
    StopReplay()
    replayData = {}
    Rayfield:Notify({Title = "🗑️", Content = "Replay apagado", Duration = 2})
end

-- ==================== INTERFACE ====================
Tab:CreateButton({ Name = "🔴 Iniciar Gravação", Callback = StartRecording })
Tab:CreateButton({ Name = "⏹️ Parar Gravação", Callback = StopRecording })
Tab:CreateButton({ Name = "▶️ Reproduzir 1x", Callback = function() PlayReplay(false) end })
Tab:CreateButton({ Name = "🔄 Loop (Repetir)", Callback = function() PlayReplay(true) end })
Tab:CreateButton({ Name = "⏹️ Parar / Remover Ghost", Callback = StopReplay })
Tab:CreateButton({ Name = "🗑️ Limpar Replay", Callback = ClearReplay })

Tab:CreateSlider({
    Name = "Velocidade do Replay",
    Range = {0.25, 3},
    Increment = 0.25,
    CurrentValue = 1,
    Callback = function(v) playbackSpeed = v end,
})

Tab:CreateButton({
    Name = "🔍 Procurar Meu Carro",
    Callback = function()
        currentCar = FindMyCar()
        if currentCar then
            local names = {}
            for _, obj in pairs(currentCar:GetDescendants()) do
                if IsWheel(obj) then table.insert(names, obj.Name) end
            end
            Rayfield:Notify({
                Title = "✅ " .. currentCar.Name,
                Content = "Rodas (" .. #names .. "): " .. table.concat(names, ", "),
                Duration = 7
            })
        else
            Rayfield:Notify({Title = "❌", Content = "Entre no carro!", Duration = 3})
        end
    end,
})

Tab:CreateParagraph({
    Title = "V7",
    Content = "Rodas agora ficam numa pasta separada no workspace e seguem o ghost todo frame.\nSe ainda falhar, cria 4 cilindros pretos como fallback."
})

Rayfield:Notify({
    Title = "👻 V7 - Rodas separadas",
    Content = "Nova forma de copiar as rodas",
    Duration = 5,
})

print("✅ Norephina Replay Ghost V7")
