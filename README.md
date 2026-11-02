-- [[ Chilli Finder + Anti-AFK / Anti-Kick / Stealth ]]
-- build mobile / 32-bit
if _G.CF_DESTROY then pcall(_G.CF_DESTROY) end

local Luna = loadstring(game:HttpGet("https://raw.githubusercontent.com/Nebula-Softworks/Luna-Interface-Suite/refs/heads/main/source.lua", true))()

local Players    = game:GetService("Players")
local HttpService= game:GetService("HttpService")
local Teleport   = game:GetService("TeleportService")
local RunService = game:GetService("RunService")
local UIS        = game:GetService("UserInputService")
local Lighting   = game:GetService("Lighting")
local VIM        = game:GetService("VirtualInputManager")
local VU         = game:GetService("VirtualUser")

local LP   = Players.LocalPlayer
local PGUI = LP:WaitForChild("PlayerGui")

local PLACE_ID, EGG_ROOT, LANE_MIN_X = 107778070777162, "AreaEggSlotsClient", 525
local LANE_Z, SAFE_X = -360, 527
local TOUCH = UIS.TouchEnabled

local alive = true

---------------------------------------------------------------------
-- CONFIG BASE
---------------------------------------------------------------------
local PERF = {
    light=TOUCH, maxMarks=TOUCH and 8 or 20, maxDist=TOUCH and 150 or 260,
    espHz=TOUCH and 8 or 15, scanHz=TOUCH and 1.2 or 0.45,
    showName=true, showInfo=not TOUCH,
}
local ROUTE = {auto=false, antiGuard=true, antiRagdoll=true, autoBase=true,
               radius=16, step=8, delay=0.09, safeX=SAFE_X, base=nil}
local S = {
    esp=false, autoHop=false, mode="até achar", minRank=3.00, webhook="",
    colorByRarity=true, onlyOnScreen=not TOUCH, noRepeat=true, maxPlayers=12,
    minScale=1.0, bigOnly=false, delay=2, targets={}, areas={}, discover=true,
}

local ANTI = {on=true, mode="misto", interval=45, jitter=true,
              antiKick=true, rejoin=true, stealth=true}
local ANTI_NOTIFY = function() end
local lastMove = os.clock()

local TIERS = {
    Common={r=0.10,c=Color3.fromRGB(165,165,170)}, Uncommon={r=0.15,c=Color3.fromRGB(105,205,110)},
    Rare={r=0.30,c=Color3.fromRGB(80,160,255)},    Epic={r=0.50,c=Color3.fromRGB(170,90,255)},
    Legendary={r=1.00,c=Color3.fromRGB(255,170,40)},Mythic={r=2.00,c=Color3.fromRGB(255,70,70)},
    Cosmic={r=3.00,c=Color3.fromRGB(120,240,255)}, Secret={r=4.00,c=Color3.fromRGB(255,60,200)},
    Eternal={r=5.00,c=Color3.fromRGB(255,240,120)},Divine={r=6.00,c=Color3.fromRGB(255,255,255)},
}

local BIOMES = {"Forest","Lake","Desert","Jungle","Snow","Volcano","Abyss","Prehistoric","Cosmic","Cherry","Titan","Light"}
local BIOME_W = {}
for i, b in ipairs(BIOMES) do BIOME_W[b] = i end
local function biomeX(i) return 528 + (i-1) * (90/(#BIOMES-1)) end

local BY_NAME = {}
local function reg(t, l) for _, n in ipairs(l) do BY_NAME[n]=TIERS[t] end end
reg("Divine",{"nightflame","unicorn","unicornegg","shatteredcolossus","kitsune","elmaja"})
reg("Eternal",{"gorillaking","lunardragon","onitiger","mosasaurus"})
reg("Secret",{"mutantshark","skeletonboss","stagegg","cosmicdragon","trex","tralaledon","kraken"})
reg("Cosmic",{"saturnita","saturno","mantaris","rhinotaur","snowyowl","koiegg","triceratops","bronto","whaleshark","belugawhale"})
reg("Mythic",{"bladehide","redpanda","cosmicgorilla","ankylosaurus","orca"})
reg("Legendary",{"spideron","crustacia","salamander","cosmicgecko","pterodactyl","sharkegg"})
reg("Epic",{"crane","centapede","swordfish"})
reg("Rare",{"dodo","parrotfish"})

local function norm(s) return (string.lower(tostring(s)):gsub("[%s_%-]","")) end

local KNOWN = {}
for n, t in pairs(BY_NAME) do KNOWN[n]=t end
local function eggNames()
    local t = {}
    for k in pairs(KNOWN) do t[#t+1]=k end
    table.sort(t)
    return t
end

local char, hrp, routeBusy, lastEgg = nil, nil, false, nil
local matches, seen, GUARDS, conns = {}, {}, {}, {}
local visited, hopping, lastHop = {}, false, 0

local function bindChar(c)
    char = c
    hrp = c:WaitForChild("HumanoidRootPart", 10)
end
if LP.Character then bindChar(LP.Character) end
LP.CharacterAdded:Connect(bindChar)

local notify = function() end

---------------------------------------------------------------------
-- ESP LAYER
---------------------------------------------------------------------
local espLayer = Instance.new("ScreenGui")
espLayer.Name="ChilliESP"; espLayer.ResetOnSpawn=false; espLayer.IgnoreGuiInset=true
espLayer.DisplayOrder=998; espLayer.Parent=(gethui and gethui()) or PGUI
if TOUCH then
    local sc = Instance.new("UIScale"); sc.Scale = 0.85; sc.Parent = espLayer
end

local POOL_N = TOUCH and 8 or 24
local pool = {}
for i=1,POOL_N do
    local f=Instance.new("Frame")
    f.Size=UDim2.fromOffset(96,22); f.AnchorPoint=Vector2.new(0.5,0.5)
    f.BackgroundColor3=Color3.fromRGB(12,12,15); f.BackgroundTransparency=0.12
    f.BorderSizePixel=0; f.Visible=false; f.Parent=espLayer
    local c=Instance.new("UICorner"); c.CornerRadius=UDim.new(0,4); c.Parent=f
    local bar=Instance.new("Frame")
    bar.Size=UDim2.fromOffset(3,22); bar.BackgroundColor3=Color3.new(1,1,1)
    bar.BorderSizePixel=0; bar.Parent=f
    local nm=Instance.new("TextLabel")
    nm.BackgroundTransparency=1; nm.Position=UDim2.fromOffset(7,-2); nm.Size=UDim2.new(1,-10,0,14)
    nm.Font=Enum.Font.GothamBold; nm.TextSize=10; nm.TextXAlignment=Enum.TextXAlignment.Left
    nm.TextTruncate=Enum.TextTruncate.AtEnd; nm.Parent=f
    local inf=Instance.new("TextLabel")
    inf.BackgroundTransparency=1; inf.Position=UDim2.fromOffset(7,10); inf.Size=UDim2.new(1,-10,0,10)
    inf.Font=Enum.Font.Gotham; inf.TextSize=8; inf.TextColor3=Color3.fromRGB(180,180,190)
    inf.TextXAlignment=Enum.TextXAlignment.Left; inf.Parent=f
    pool[i]={f=f,bar=bar,nm=nm,inf=inf}
end
local function releaseAll() for _,p in ipairs(pool) do p.f.Visible=false end end

---------------------------------------------------------------------
-- TELEPORTE / CARGA / ROTA
---------------------------------------------------------------------
local function tp(pos)
    if not hrp or not hrp.Parent then return end
    pcall(function()
        hrp.AssemblyLinearVelocity = Vector3.zero
        hrp.AssemblyAngularVelocity = Vector3.zero
        hrp.CFrame = CFrame.new(pos)
    end)
end

local function goX(x)
    if not hrp then return end
    local p = hrp.Position
    tp(Vector3.new(x, p.Y, p.Z))
end

local function goBase()
    if ROUTE.base then pcall(function() hrp.CFrame = ROUTE.base end)
    else notify("Sem base", "Use DEFINIR BASE primeiro") end
end

local CARRY_ATTRS = {"HeldEgg","CarryingEgg","EggUid","Carrying","Held","CarryUid"}
local function isCarrying()
    if not char then return false end
    for _, a in ipairs(CARRY_ATTRS) do
        local v = char:GetAttribute(a) or LP:GetAttribute(a)
        if v ~= nil and v ~= false then return true end
    end
    for _, c in ipairs(char:GetChildren()) do
        if (c:IsA("Model") or c:IsA("Tool")) and string.find(string.lower(c.Name), "egg") then return true end
    end
    return false
end

local function nearestGuard()
    local bd = 1e9
    if not hrp then return bd end
    for _, g in ipairs(GUARDS) do
        if g.Parent then
            local ok, p = pcall(function() return g:GetPivot().Position end)
            if ok and p then bd = math.min(bd, (p - hrp.Position).Magnitude) end
        end
    end
    return bd
end

local function fireRemote(path, args)
    local node = game:GetService("ReplicatedStorage"):FindFirstChild("RF")
    if not node then return false end
    for part in string.gmatch(path, "[^/]+") do
        node = node:FindFirstChild(part)
        if not node then return false end
    end
    if not node or not node:IsA("RemoteFunction") then return false end
    return pcall(function() node:InvokeServer(unpack(args or {})) end)
end

local function dropEgg()
    if fireRemote("EggWorld/AskFieldEggCarry", {}) then return true end
    return pcall(function()
        VIM:SendKeyEvent(true, "Q", false, game)
        task.wait(0.05)
        VIM:SendKeyEvent(false, "Q", false, game)
    end)
end

local function triggerPrompt(model)
    if not model then return end
    for _, d in ipairs(model:GetDescendants()) do
        if d:IsA("ProximityPrompt") then
            pcall(function()
                d:InputHoldBegin()
                task.wait(math.max(d.HoldDuration, 0.05) + 0.03)
                d:InputHoldEnd()
            end)
            return true
        end
    end
    return false
end

function startRoute(egg)
    if routeBusy or not hrp then return end
    routeBusy = true
    local y = hrp.Position.Y
    local z = (egg and egg.pos and egg.pos.Z) or hrp.Position.Z
    if z == 0 then z = LANE_Z end
    local x = hrp.Position.X

    while x > ROUTE.safeX + 0.5 and isCarrying() and alive do
        x = math.max(ROUTE.safeX, x - ROUTE.step)
        tp(Vector3.new(x, y, z))
        task.wait(ROUTE.delay)
        if ROUTE.antiGuard and nearestGuard() < ROUTE.radius then
            x = math.max(ROUTE.safeX, x - ROUTE.step * 2)
            tp(Vector3.new(x, y, z))
            task.wait(ROUTE.delay)
        end
    end

    dropEgg()
    task.wait(0.4)

    if egg and egg.model and egg.model.Parent then
        local ok, p = pcall(function() return egg.model:GetPivot().Position end)
        if ok and p then
            tp(Vector3.new(p.X, p.Y + 1, p.Z))
            task.wait(0.12)
            triggerPrompt(egg.model)
        end
        task.wait(0.35)
    end

    if ROUTE.autoBase and ROUTE.base then
        goBase()
        task.wait(0.25)
        dropEgg()
    end
    routeBusy = false
end

---------------------------------------------------------------------
-- SCAN
---------------------------------------------------------------------
local function biomeOf(m)
    local a = tostring(m:GetAttribute("AreaId") or m:GetAttribute("Area") or "")
    for _, b in ipairs(BIOMES) do
        if string.find(string.lower(a), string.lower(b), 1, true) then return b end
    end
    return a
end

local function eggInfo(m)
    local cat = m:GetAttribute("Category") or m:GetAttribute("AssetCategory")
    if not cat then return nil end
    local key = norm(cat); local tier = KNOWN[key]
    local rank = tonumber(m:GetAttribute("RarityRank") or m:GetAttribute("Rank")) or (tier and tier.r) or TIERS.Common.r
    return {model=m, name=tostring(cat), key=key, rank=rank,
            color=(tier and tier.c) or TIERS.Common.c,
            income=tonumber(m:GetAttribute("Income") or m:GetAttribute("EarningRate")) or 0,
            scale=tonumber(m:GetAttribute("Scale")) or 1, biome=biomeOf(m)}
end

local function passArea(e)
    if not next(S.areas) then return true end
    for b in pairs(S.areas) do
        if string.find(string.lower(e.biome), string.lower(b), 1, true) then return true end
    end
    return false
end

local function passes(e)
    if e.rank < S.minRank then return false end
    if S.bigOnly and e.scale < S.minScale then return false end
    if next(S.targets) and not S.targets[e.name] then return false end
    return passArea(e)
end

local function scan()
    local out = {}
    GUARDS = {}
    for _, g in ipairs(workspace:GetChildren()) do
        if g:IsA("Model") and string.find(string.lower(g.Name), "guard") then GUARDS[#GUARDS+1]=g end
    end
    local rt = workspace:FindFirstChild(EGG_ROOT)
    if not rt then return out end
    for _, m in ipairs(rt:GetChildren()) do
        if m:IsA("Model") and m.Name ~= "FirstArea" then
            local ok, p = pcall(function() return m:GetPivot().Position end)
            if ok and p and p.X >= LANE_MIN_X then
                local e = eggInfo(m)
                if e then e.pos = p; out[#out+1] = e end
            end
        end
    end
    table.sort(out, function(a,b)
        if a.rank ~= b.rank then return a.rank > b.rank end
        return (BIOME_W[a.biome] or 0) > (BIOME_W[b.biome] or 0)
    end)
    return out
end

---------------------------------------------------------------------
-- HOP
---------------------------------------------------------------------
local function fetchServers()
    local l, cur = {}, nil
    for _=1,4 do
        local url=("https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=Asc&limit=100"):format(PLACE_ID)
        if cur then url..="&cursor="..cur end
        local ok, body = pcall(function() return HttpService:JSONDecode(game:HttpGet(url, true)) end)
        if not ok or type(body)~="table" or type(body.data)~="table" then break end
        for _, s in ipairs(body.data) do
            if s.id and s.playing and s.maxPlayers and s.id~=game.JobId
               and s.playing < math.min(s.maxPlayers, S.maxPlayers) then l[#l+1]=s end
        end
        cur=body.nextPageCursor
        if not cur or cur=="" then break end
    end
    table.sort(l, function(a,b) return a.playing < b.playing end)
    return l
end

function hop(force)
    if hopping then return false end
    if not force and os.clock()-lastHop < 6 then return false end
    hopping = true
    local ok, sv = pcall(fetchServers); sv = ok and sv or {}
    local target
    for _, s in ipairs(sv) do if not (S.noRepeat and visited[s.id]) then target=s break end end
    if not target then visited={}; target=sv[1] end
    if not target then
        hopping=false
        pcall(function() Teleport:Teleport(PLACE_ID, LP) end)
        return false
    end
    visited[target.id]=true; lastHop=os.clock()
    local tOk = pcall(function() Teleport:TeleportToPlaceInstance(PLACE_ID, target.id, LP) end)
    hopping=false
    if not tOk then task.wait(1); return hop(true) end
    return true
end

Teleport.TeleportInitFailed:Connect(function() hopping=false; task.wait(2); hop(true) end)

---------------------------------------------------------------------
-- OTIMIZAÇÃO
---------------------------------------------------------------------
local saved
local function optimize(on)
    local UGS = UserSettings():GetService("UserGameSettings")
    if on then
        saved={gs=Lighting.GlobalShadows, q=UGS.SavedQualityLevel}
        pcall(function() Lighting.GlobalShadows=false end)
        pcall(function() UGS.SavedQualityLevel=Enum.SavedQualitySetting.QualityLevel1 end)
        for _, v in ipairs(workspace:GetDescendants()) do
            if v:IsA("ParticleEmitter") or v:IsA("Trail") then pcall(function() v.Enabled=false end) end
        end
    elseif saved then
        pcall(function() Lighting.GlobalShadows=saved.gs end)
        pcall(function() UGS.SavedQualityLevel=saved.q end)
    end
end

---------------------------------------------------------------------
-- ANTI-AFK / ANTI-KICK / STEALTH
---------------------------------------------------------------------
local function jit(v, pct)
    if not ANTI.jitter then return v end
    local p = pct or 0.25
    return v * (1 - p + math.random() * p * 2)
end

local function nudge()
    if not hrp then return end
    local p = hrp.Position + Vector3.new(math.random(-2,2), 0, math.random(-2,2))
    pcall(function()
        hrp.AssemblyLinearVelocity = Vector3.zero
        hrp.CFrame = CFrame.new(p, p + hrp.CFrame.LookVector)
    end)
end

local function tapKey()
    local k = (math.random() > 0.5) and "W" or "Space"
    pcall(function()
        VIM:SendKeyEvent(true, k, false, game)
        task.wait(0.06)
        VIM:SendKeyEvent(false, k, false, game)
    end)
end

local function vClick()
    pcall(function()
        VU:CaptureController()
        VU:ClickButton2(Vector2.new())
    end)
end

local function jumpOnce()
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if hum then pcall(function() hum:ChangeState(Enum.HumanoidStateType.Jumping) end) end
end

local STRATS = {nudge, tapKey, vClick, jumpOnce}

local function antiAfkTick()
    if not ANTI.on then return end
    local pick
    if ANTI.mode == "misto" then pick = STRATS[math.random(1, #STRATS)]
    elseif ANTI.mode == "click" then pick = vClick
    elseif ANTI.mode == "tecla" then pick = tapKey
    else pick = nudge end
    if TOUCH and math.random() > 0.3 then pick = vClick end
    if ANTI.stealth and math.random() > 0.6 then pick = vClick end
    pcall(pick)
    lastMove = os.clock()
end

conns[#conns+1] = task.spawn(function()
    while alive do
        task.wait(1)
        if ANTI.on and os.clock() - lastMove >= jit(ANTI.interval, 0.35) then
            antiAfkTick()
        end
    end
end)

conns[#conns+1] = LP.Idled:Connect(function()
    if ANTI.on then antiAfkTick() end
end)

conns[#conns+1] = task.spawn(function()
    local down = 0
    while alive do
        task.wait(2)
        if ANTI.antiKick then
            if LP.Parent == nil or (LP.Character == nil and not hopping and not routeBusy) then
                down += 1
                if down >= 3 and ANTI.rejoin then
                    down = 0
                    ANTI_NOTIFY("Caiu", "tentando rejoin")
                    pcall(function() Teleport:TeleportToPlaceInstance(PLACE_ID, game.JobId, LP) end)
                    task.wait(4)
                end
            else
                down = 0
            end
        end
    end
end)

conns[#conns+1] = task.spawn(function()
    while alive do
        task.wait(0.4)
        if ROUTE.antiRagdoll and char then
            local hum = char:FindFirstChildOfClass("Humanoid")
            if hum and (hum:GetState() == Enum.HumanoidStateType.Ragdoll or hum.PlatformStand) then
                pcall(function()
                    hum.PlatformStand = false
                    hum:ChangeState(Enum.HumanoidStateType.GettingUp)
                end)
            end
        end
    end
end)

---------------------------------------------------------------------
-- JANELA LUNA
---------------------------------------------------------------------
local Window = Luna:CreateWindow({
    Name = "Chilli Finder",
    Subtitle = "Steal an Egg",
    LogoID = nil,
    LoadingEnabled = true,
    LoadingTitle = "Chilli Finder",
    LoadingSubtitle = "carregando...",
    ConfigSettings = {RootFolder = nil, ConfigFolder = "ChilliSAE"},
    KeySystem = false,
})

notify = function(t, d)
    pcall(function() Window:Notify({Title=t, Text=d, Duration=3, Icon="info"}) end)
end
ANTI_NOTIFY = notify

if not TOUCH then pcall(function() Window:SetToggleKey(Enum.KeyCode.RightShift) end) end

Window:CreateHomeTab({
    SupportedExecutors = {"Delta","Fluxus","Krnl","Wave","Hydrogen","Codex"},
    DiscordInvite = "",
    Icon = 2,
})

---------------------------------------------------------------------
-- ABA FINDER
---------------------------------------------------------------------
local tabFinder = Window:CreateTab({Name="Finder", Icon="search", ImageSource="Material", ShowTitle=true})
tabFinder:CreateParagraph({Title="Filtro", Text="Só o que passa aqui vira alvo e aparece no ESP."})

local minDrop = tabFinder:CreateDropdown({
    Name="Raridade mínima",
    Options={"Rare","Epic","Legendary","Mythic","Cosmic","Secret","Eternal","Divine"},
    CurrentOption={"Cosmic"},
    MultipleOptions=false,
    Callback=function(o) S.minRank = TIERS[o[1] or o] and TIERS[o[1] or o].r or 3 end,
}, "f_min")

tabFinder:CreateToggle({Name="Só ovo grande", CurrentValue=false,
    Callback=function(v) S.bigOnly=v end}, "f_big")

tabFinder:CreateSlider({Name="Tamanho mínimo", Range={1,5}, Increment=0.25, CurrentValue=1,
    Callback=function(v) S.minScale=v end}, "f_scale")

local areaDrop = tabFinder:CreateDropdown({
    Name="Bioma", Options=BIOMES, CurrentOption={}, MultipleOptions=true,
    Callback=function(list)
        S.areas = {}
        for _, b in ipairs(list) do S.areas[b]=true end
    end,
}, "f_area")

local tgtDrop = tabFinder:CreateDropdown({
    Name="Alvos específicos", Options=eggNames(), CurrentOption={}, MultipleOptions=true,
    Callback=function(list)
        S.targets = {}
        for _, n in ipairs(list) do S.targets[n]=true end
    end,
}, "f_target")

local function refreshTargets()
    pcall(function() tgtDrop:Refresh(eggNames()) end)
end

---------------------------------------------------------------------
-- ABA ROTA
---------------------------------------------------------------------
local tabRota = Window:# P-
