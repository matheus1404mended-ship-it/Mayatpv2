# Mayatpv2
Tp v2 do kiiller

tp do killer versão 2 que ele tanto ama 
feito por ia
killer fudido
by maya
local Players           = game:GetService("Players")
local RunService        = game:GetService("RunService")
local UserInputService  = game:GetService("UserInputService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local CoreGui           = game:GetService("CoreGui")

local player = Players.LocalPlayer

local EggState, AssetsData, MutationsData, EggRecords
pcall(function() EggState    = require(ReplicatedStorage.Client.EggState) end)
pcall(function() AssetsData  = require(ReplicatedStorage.Data.Assets) end)
pcall(function() MutationsData = require(ReplicatedStorage.Shared.Modules.Mutations) end)
pcall(function() EggRecords  = require(ReplicatedStorage.Shared.Util.EggRecords) end)

local RARITY_SCORE_MAP = {
    ["Light & Dark"]=1300,["Titan"]=1100,["Divine"]=1000,["Transcendent"]=1000,
    ["Superior"]=1000,["Eternal"]=900,["Limited"]=900,["Secret"]=800,
    ["Exotic"]=800,["Cosmic"]=700,["Exclusive"]=700,["Admin"]=700,
    ["Mythic"]=600,["Mythical"]=600,["Prismatic"]=600,["Rainbow"]=600,
    ["Squishy God"]=600,["BrainrotGod"]=600,["Legendary"]=500,["Epic"]=400,
    ["Rare"]=300,["SuperRare"]=200,["Celestial"]=200,["Uncommon"]=200,
    ["Basic"]=100,["Common"]=100,
}

local function getAssetInfo(cat)
    if not AssetsData then return nil end
    return (AssetsData.Directory or AssetsData)[tostring(cat)]
end

local function getRarityInfo(record)
    if record.Rarity then
        local r = record.Rarity
        local name = type(r) == "table" and (r.DisplayName or r._id or r.Name) or tostring(r)
        return name, RARITY_SCORE_MAP[name] or 100
    end

    local cat = record.AssetCategory or record.Category or record.Name
    if cat then
        local aInfo = getAssetInfo(cat)
        if aInfo and aInfo.Rarity then
            local r = aInfo.Rarity
            local name = type(r) == "table" and (r.DisplayName or r._id or r.Name) or tostring(r)
            return name, RARITY_SCORE_MAP[name] or 100
        end
    end

    return "Common", 100
end

local function getRarityColor(cat)
    local a = getAssetInfo(cat)
    local r = a and a.Rarity
    if type(r) == "table" and typeof(r.Color) == "Color3" then return r.Color end
    return Color3.fromRGB(200, 200, 200)
end

local function getEggName(cat)
    local a = getAssetInfo(cat)
    if type(a) == "table" and a.DisplayName then return tostring(a.DisplayName) end
    return tostring(cat or "?")
end

local function getEggIcon(record)
    local cat = record.AssetCategory or record.Category
    if not cat then return "" end
    local a = getAssetInfo(cat)
    if not a then return "" end
    local icon = a.Icon or a.Image or a.AssetId
    if icon == nil then return "" end
    if type(icon) == "number" or not string.match(tostring(icon), "://") then
        return "rbxassetid://" .. tostring(icon)
    end
    return tostring(icon)
end

local function getMutationMult(muts)
    if MutationsData and type(MutationsData.EarningsFor) == "function" then
        local ok, res = pcall(MutationsData.EarningsFor, muts or {})
        if ok and type(res) == "number" then return res end
    end
    return 1
end

local function scaleFactor(s)
    s = tonumber(s) or 1
    if s > 5 then return (s / 5) ^ 1.2 * 19.637875755794113 end
    return s ^ 1.85
end

local function calcValue(cat, scale, muts)
    local a = getAssetInfo(cat)
    local base = a and tonumber(a.EarningRate) or 0
    if base <= 0 then return 0 end
    return base * scaleFactor(scale) * getMutationMult(muts)
end

local function calcWeight(cat, scale)
    if EggRecords and type(EggRecords.WeightKgForScale) == "function" then
        local ok, res = pcall(EggRecords.WeightKgForScale, cat, scale)
        if ok and tonumber(res) then return tonumber(res) end
    end
    return 0
end

local function fmtRate(v)
    v = tonumber(v) or 0
    if v >= 1e12 then return string.format("%.2fT/s", v/1e12) end
    if v >= 1e9  then return string.format("%.2fB/s", v/1e9)  end
    if v >= 1e6  then return string.format("%.2fM/s", v/1e6)  end
    if v >= 1e3  then return string.format("%.1fK/s", v/1e3)  end
    return string.format("%d/s", math.floor(v))
end

local function fmtWeight(w)
    w = tonumber(w) or 0
    local s = w >= 1000 and string.format("%.0f", w) or string.format("%.2f", w)
    local int, dec = s:match("^(%-?%d+)(%.%d+)$")
    int = int or s; dec = dec or ""
    local out = int
    while true do
        local n, k = out:gsub("^(%-?%d+)(%d%d%d)", "%1,%2")
        if k == 0 then break end
        out = n
    end
    return out .. dec .. " Kg"
end

local function lerOvos()
    local lista = {}
    if not EggState or type(EggState.ReadFieldEggs) ~= "function" then
        return lista
    end

    local ok, snapshot = pcall(EggState.ReadFieldEggs)
    if not ok or not snapshot or not snapshot.Records then return lista end

    local char = player.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    local myPos = hrp and hrp.Position or Vector3.zero

    for _, record in ipairs(snapshot.Records) do
        if record.State == "Slot" and record.BoundsCFrame then
            local eggPos = record.BoundsCFrame.Position
            local dist = (eggPos - myPos).Magnitude
            local cat = record.AssetCategory or record.Category
            local scale = tonumber(record.AssetScale) or tonumber(record.Scale) or 1
            local muts = record.Mutations
            local rarName, rarScore = getRarityInfo(record)

            table.insert(lista, {
                Uid = record.Uid or tostring(record.BoundsCFrame),
                Category = cat,
                Name = getEggName(cat),
                RarityName = rarName,
                RarityScore = rarScore,
                RarityColor = getRarityColor(cat),
                Icon = getEggIcon(record),
                Scale = scale,
                Value = calcValue(cat, scale, muts),
                Weight = calcWeight(cat, scale),
                Position = eggPos,
                Distance = dist,
            })
        end
    end

    return lista
end

local TEMA = {
    fundo      = Color3.fromRGB(15, 15, 20),
    painel     = Color3.fromRGB(25, 25, 35),
    destaque   = Color3.fromRGB(46, 204, 113),
    texto      = Color3.fromRGB(240, 240, 245),
    textoFraco = Color3.fromRGB(150, 150, 160),
    verde      = Color3.fromRGB(50, 220, 100),
    vermelho   = Color3.fromRGB(231, 76, 60),
    borda      = Color3.fromRGB(45, 45, 60),
}

local SG = Instance.new("ScreenGui")
SG.Name = "EggTPPanel"
SG.ResetOnSpawn = false
SG.IgnoreGuiInset = true
SG.DisplayOrder = 999999998
SG.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
SG.Parent = (gethui and gethui()) or player:WaitForChild("PlayerGui")

local Main = Instance.new("Frame")
Main.Size = UDim2.new(0, 250, 0, 340)
Main.Position = UDim2.new(0.5, -125, 0.5, -170)
Main.BackgroundColor3 = TEMA.fundo
Main.BorderSizePixel = 0
Main.Active = true
Main.Draggable = true
Main.Visible = false
Main.Parent = SG

local MC = Instance.new("UICorner"); MC.CornerRadius = UDim.new(0, 10); MC.Parent = Main
local MS = Instance.new("UIStroke"); MS.Color = TEMA.borda; MS.Thickness = 1.5; MS.Parent = Main

local Header = Instance.new("Frame")
Header.Size = UDim2.new(1, 0, 0, 38)
Header.BackgroundColor3 = TEMA.painel
Header.BorderSizePixel = 0
Header.Parent = Main

local HC = Instance.new("UICorner"); HC.CornerRadius = UDim.new(0, 10); HC.Parent = Header

local LinhaDecor = Instance.new("Frame")
LinhaDecor.Size = UDim2.new(0, 3, 0, 22)
LinhaDecor.Position = UDim2.new(0, 10, 0.5, -11)
LinhaDecor.BackgroundColor3 = TEMA.destaque
LinhaDecor.BorderSizePixel = 0
LinhaDecor.Parent = Header

local Logo = Instance.new("TextLabel")
Logo.Size = UDim2.new(0, 180, 0, 15)
Logo.Position = UDim2.new(0, 20, 0, 5)
Logo.BackgroundTransparency = 1
Logo.Text = "AUTO TP | By KILLER"
Logo.TextColor3 = TEMA.texto
Logo.Font = Enum.Font.GothamBlack
Logo.TextSize = 13
Logo.TextXAlignment = Enum.TextXAlignment.Left
Logo.Parent = Header

local Versao = Instance.new("TextLabel")
Versao.Size = UDim2.new(0, 180, 0, 12)
Versao.Position = UDim2.new(0, 20, 0, 20)
Versao.BackgroundTransparency = 1
Versao.Text = "0 ovos"
Versao.TextColor3 = TEMA.textoFraco
Versao.Font = Enum.Font.Gotham
Versao.TextSize = 9
Versao.TextXAlignment = Enum.TextXAlignment.Left
Versao.Parent = Header

local BtnMin = Instance.new("TextButton")
BtnMin.Size = UDim2.new(0, 22, 0, 22)
BtnMin.Position = UDim2.new(1, -28, 0, 8)
BtnMin.BackgroundColor3 = TEMA.fundo
BtnMin.Text = "—"
BtnMin.TextColor3 = TEMA.texto
BtnMin.Font = Enum.Font.GothamBold
BtnMin.TextSize = 12
BtnMin.AutoButtonColor = false
BtnMin.Parent = Header

local BMC = Instance.new("UICorner"); BMC.CornerRadius = UDim.new(0, 5); BMC.Parent = BtnMin

local SpeedBar = Instance.new("Frame")
SpeedBar.Size = UDim2.new(1, -16, 0, 22)
SpeedBar.Position = UDim2.new(0, 8, 0, 44)
SpeedBar.BackgroundTransparency = 1
SpeedBar.Parent = Main

local BtnSpeed = Instance.new("TextButton")
BtnSpeed.Size = UDim2.new(1, 0, 1, 0)
BtnSpeed.Position = UDim2.new(0, 0, 0, 0)
BtnSpeed.BackgroundColor3 = TEMA.destaque
BtnSpeed.Text = "Tween Speed: 400"
BtnSpeed.TextColor3 = TEMA.texto
BtnSpeed.Font = Enum.Font.GothamBold
BtnSpeed.TextSize = 11
BtnSpeed.AutoButtonColor = false
BtnSpeed.Parent = SpeedBar

local BSC = Instance.new("UICorner"); BSC.CornerRadius = UDim.new(0, 5); BSC.Parent = BtnSpeed

local SortBar = Instance.new("Frame")
SortBar.Size = UDim2.new(1, -16, 0, 20)
SortBar.Position = UDim2.new(0, 8, 0, 70)
SortBar.BackgroundTransparency = 1
SortBar.Parent = Main

local BtnSort = Instance.new("TextButton")
BtnSort.Size = UDim2.new(1, 0, 1, 0)
BtnSort.BackgroundColor3 = Color3.fromRGB(60, 180, 70)
BtnSort.Text = "Sort: Value"
BtnSort.TextColor3 = TEMA.texto
BtnSort.Font = Enum.Font.GothamBold
BtnSort.TextSize = 11
BtnSort.AutoButtonColor = false
BtnSort.Parent = SortBar

local BSoC = Instance.new("UICorner"); BSoC.CornerRadius = UDim.new(0, 5); BSoC.Parent = BtnSort

local List = Instance.new("ScrollingFrame")
List.Size = UDim2.new(1, -16, 1, -124)
List.Position = UDim2.new(0, 8, 0, 94)
List.BackgroundTransparency = 1
List.BorderSizePixel = 0
List.ScrollBarThickness = 3
List.ScrollBarImageColor3 = TEMA.destaque
List.CanvasSize = UDim2.new(0, 0, 0, 0)
List.Parent = Main

local UIList = Instance.new("UIListLayout")
UIList.SortOrder = Enum.SortOrder.LayoutOrder
UIList.Padding = UDim.new(0, 3)
UIList.Parent = List

local Footer = Instance.new("TextLabel")
Footer.Size = UDim2.new(1, -16, 0, 20)
Footer.Position = UDim2.new(0, 8, 1, -26)
Footer.BackgroundTransparency = 1
Footer.Text = "By KILLER | AUTO TP EGG"
Footer.TextColor3 = TEMA.textoFraco
Footer.Font = Enum.Font.Gotham
Footer.TextSize = 8
Footer.Parent = Main

local BtnFloat = Instance.new("TextButton")
BtnFloat.Size = UDim2.new(0, 46, 0, 46)
BtnFloat.Position = UDim2.new(0, 12, 0.5, -23)
BtnFloat.BackgroundColor3 = TEMA.destaque
BtnFloat.Text = "K"
BtnFloat.TextColor3 = TEMA.texto
BtnFloat.Font = Enum.Font.GothamBlack
BtnFloat.TextSize = 20
BtnFloat.AutoButtonColor = false
BtnFloat.Parent = SG

local BFC = Instance.new("UICorner"); BFC.CornerRadius = UDim.new(0, 25); BFC.Parent = BtnFloat
local BFS = Instance.new("UIStroke"); BFS.Color = Color3.fromRGB(255,255,255); BFS.Thickness = 2; BFS.Transparency = 0.5; BFS.Parent = BtnFloat

local drag, mov, dStart, dPos = false, false, nil, nil
BtnFloat.InputBegan:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
        drag, mov = true, false
        dStart, dPos = i.Position, BtnFloat.Position
    end
end)
BtnFloat.InputChanged:Connect(function(i)
    if drag and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
        local d = i.Position - dStart
        if d.Magnitude > 6 then mov = true end
        if mov then
            BtnFloat.Position = UDim2.new(dPos.X.Scale, dPos.X.Offset + d.X, dPos.Y.Scale, dPos.Y.Offset + d.Y)
        end
    end
end)
UserInputService.InputEnded:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
        drag = false
    end
end)

local aberto = false
local function abrir() aberto = true; BtnFloat.Visible = false; Main.Visible = true end
local function fechar() aberto = false; Main.Visible = false; BtnFloat.Visible = true end

BtnFloat.MouseButton1Click:Connect(function() if not mov then abrir() end end)
BtnMin.MouseButton1Click:Connect(fechar)

local wD, wS, wP = false, nil, nil
Header.InputBegan:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
        wD, wS, wP = true, i.Position, Main.Position
    end
end)
UserInputService.InputChanged:Connect(function(i)
    if wD and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
        local d = i.Position - wS
        Main.Position = UDim2.new(wP.X.Scale, wP.X.Offset + d.X, wP.Y.Scale, wP.Y.Offset + d.Y)
    end
end)
UserInputService.InputEnded:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
        wD = false
    end
end)

local sortModes = { "Value", "Rarity", "Distance" }
local sortIdx = 1
local tweenSpeed = 400

BtnSort.MouseButton1Click:Connect(function()
    sortIdx = (sortIdx % #sortModes) + 1
    BtnSort.Text = "Sort: " .. sortModes[sortIdx]
    pcall(refresh)
end)

BtnSpeed.MouseButton1Click:Connect(function()
    local speeds = { 200, 400, 600, 800, 1000 }
    local i = table.find(speeds, tweenSpeed) or 2
    i = (i % #speeds) + 1
    tweenSpeed = speeds[i]
    BtnSpeed.Text = "Tween Speed: " .. tweenSpeed
end)

local tpAtivo, tpAlvo = false, nil
local tpUidAtual = nil

local function tpParar()
    tpAtivo = false
    tpAlvo = nil
    tpUidAtual = nil
    local char = player.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if hrp then
        hrp.AssemblyLinearVelocity = Vector3.zero
    end
end

local function tpPara(pos, uid)
    if not pos then return end
    
    if tpAtivo then
        tpParar()
        task.wait(0.1)
    end

    tpAtivo = true
    tpAlvo = pos
    tpUidAtual = uid

    task.spawn(function()
        local inicio = tick()
        while tpAtivo and (tick() - inicio) < 10 do
            local char = player.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            if not hrp then break end
            local pa = hrp.Position
            local diff = tpAlvo - pa
            local dist = diff.Magnitude
            if dist < 5 then
                hrp.CFrame = CFrame.new(tpAlvo)
                hrp.AssemblyLinearVelocity = Vector3.zero
                break
            end
            local dxz = Vector3.new(diff.X, 0, diff.Z)
            if dxz.Magnitude < 0.01 then break end
            local u = dxz.Unit
            
            hrp.AssemblyLinearVelocity = Vector3.new(
                u.X * tweenSpeed, 0, u.Z * tweenSpeed)
            task.wait()
        end
        tpAtivo = false
        tpAlvo = nil
        tpUidAtual = nil
    end)
end

local function criarLinha(d)
    local row = Instance.new("TextButton")
    row.Size = UDim2.new(1, -4, 0, 48)
    row.BackgroundColor3 = TEMA.painel
    row.BorderSizePixel = 0
    row.AutoButtonColor = false
    row.Text = ""
    row.Parent = List

    local c1 = Instance.new("UICorner"); c1.CornerRadius = UDim.new(0, 6); c1.Parent = row

    local icon = Instance.new("ImageLabel")
    icon.Size = UDim2.fromOffset(36, 36)
    icon.Position = UDim2.new(0, 5, 0, 6)
    icon.BackgroundColor3 = Color3.fromRGB(20, 20, 26)
    icon.BorderSizePixel = 0
    icon.Image = d.Icon
    icon.ScaleType = Enum.ScaleType.Fit
    icon.Parent = row
    local c2 = Instance.new("UICorner"); c2.CornerRadius = UDim.new(0, 5); c2.Parent = icon

    local nm = Instance.new("TextLabel")
    nm.Size = UDim2.new(1, -120, 0, 13)
    nm.Position = UDim2.new(0, 46, 0, 3)
    nm.BackgroundTransparency = 1
    nm.Text = d.Name
    nm.TextColor3 = TEMA.texto
    nm.Font = Enum.Font.GothamBold
    nm.TextSize = 11
    nm.TextXAlignment = Enum.TextXAlignment.Left
    nm.TextTruncate = Enum.TextTruncate.AtEnd
    nm.Parent = row

    local rar = Instance.new("TextLabel")
    rar.Size = UDim2.new(1, -120, 0, 11)
    rar.Position = UDim2.new(0, 46, 0, 16)
    rar.BackgroundTransparency = 1
    rar.Text = string.upper(d.RarityName)
    rar.TextColor3 = d.RarityColor
    rar.Font = Enum.Font.Gotham
    rar.TextSize = 9
    rar.TextXAlignment = Enum.TextXAlignment.Left
    rar.Parent = row

    local det = Instance.new("TextLabel")
    det.Size = UDim2.new(1, -120, 0, 13)
    det.Position = UDim2.new(0, 46, 0, 28)
    det.BackgroundTransparency = 1
    det.Text = string.format("$%s · x%.2f · %s", fmtRate(d.Value), d.Scale, fmtWeight(d.Weight))
    det.TextColor3 = Color3.fromRGB(180, 220, 180)
    det.Font = Enum.Font.Gotham
    det.TextSize = 9
    det.TextXAlignment = Enum.TextXAlignment.Left
    det.Parent = row

    local bSteal = Instance.new("TextButton")
    bSteal.AnchorPoint = Vector2.new(1, 0)
    bSteal.Position = UDim2.new(1, -5, 0, 4)
    bSteal.Size = UDim2.fromOffset(42, 18)
    bSteal.BackgroundColor3 = TEMA.destaque
    bSteal.Text = "TP"
    bSteal.TextColor3 = TEMA.texto
    bSteal.Font = Enum.Font.GothamBold
    bSteal.TextSize = 10
    bSteal.AutoButtonColor = false
    bSteal.Parent = row
    local c3 = Instance.new("UICorner"); c3.CornerRadius = UDim.new(0, 4); c3.Parent = bSteal

    local tpDesteOvo = false

    local function voltarTP()
        tpDesteOvo = false
        bSteal.Text = "TP"
        bSteal.BackgroundColor3 = TEMA.destaque
    end

    local function comecarTP()
        tpDesteOvo = true
        bSteal.Text = "❌"
        bSteal.BackgroundColor3 = TEMA.vermelho
        tpPara(d.Position, d.Uid)

        task.spawn(function()
            while tpDesteOvo do
                task.wait(0.15)
                if not tpAtivo or tpUidAtual ~= d.Uid then
                    voltarTP()
                    break
                end
            end
        end)
    end

    local dist = Instance.new("TextLabel")
    dist.AnchorPoint = Vector2.new(1, 0)
    dist.Position = UDim2.new(1, -5, 0, 26)
    dist.Size = UDim2.fromOffset(50, 14)
    dist.BackgroundTransparency = 1
    dist.TextColor3 = TEMA.textoFraco
    dist.Font = Enum.Font.Gotham
    dist.TextSize = 9
    dist.TextXAlignment = Enum.TextXAlignment.Right
    dist.Parent = row

    bSteal.MouseButton1Click:Connect(function()
        if tpDesteOvo then
            tpParar()
            voltarTP()
        else
            comecarTP()
        end
    end)

    row.MouseButton1Click:Connect(function()
        if not tpDesteOvo then
            comecarTP()
        end
    end)

    return row, dist
end

local linhasAtuais = {}
local lastStatus = ""

function refresh()
    local ovos = lerOvos()
    local vistos = {}

    local mode = sortModes[sortIdx]
    if mode == "Value" then
        table.sort(ovos, function(a, b) return a.Value > b.Value end)
    elseif mode == "Rarity" then
        table.sort(ovos, function(a, b)
            if a.RarityScore ~= b.RarityScore then return a.RarityScore > b.RarityScore end
            return a.Value > b.Value
        end)
    elseif mode == "Distance" then
        table.sort(ovos, function(a, b) return a.Distance < b.Distance end)
    end

    for i, d in ipairs(ovos) do
        vistos[d.Uid] = true
        local linha = linhasAtuais[d.Uid]
        if not linha then
            local row, distLabel = criarLinha(d)
            linhasAtuais[d.Uid] = { Row = row, Dist = distLabel }
            linha = linhasAtuais[d.Uid]
        end
        linha.Row.LayoutOrder = i
        linha.Dist.Text = string.format("%dm", math.floor(d.Distance + 0.5))
    end

    for uid, linha in pairs(linhasAtuais) do
        if not vistos[uid] then
            linha.Row:Destroy()
            linhasAtuais[uid] = nil
        end
    end

    List.CanvasSize = UDim2.new(0, 0, 0, #ovos * 52)

    local txt = #ovos .. " ovos"
    if txt ~= lastStatus then
        lastStatus = txt
        Versao.Text = txt
    end
end

task.spawn(function()
    while true do
        if aberto then pcall(refresh) end
        task.wait(1)
    end
end)

task.spawn(function()
    task.wait(2)
    print("[StealPanel] === DIAGNÓSTICO ===")
    print("EggState:", EggState ~= nil)
    print("AssetsData:", AssetsData ~= nil)
    print("MutationsData:", MutationsData ~= nil)
    print("EggRecords:", EggRecords ~= nil)
    print("Total de ovos:", #lerOvos())
    print("[StealPanel] === FIM ===")
end)

task.wait(0.5)
abrir()
print("✅ By KILLER | Auto TP carregado!")
