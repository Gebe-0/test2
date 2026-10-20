--==============================================================--
--  Project Lazarus — combat / farm suite  (Xeno build)
--  RShift = menu   F1 = overlay
--  perf: cached scans + throttled ESP, no per-frame workspace walks
--==============================================================--

if not game:IsLoaded() then game.Loaded:Wait() end
if type(getgenv) == "function" and getgenv().LZ_UNLOAD then pcall(getgenv().LZ_UNLOAD) end

local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace        = game:GetService("Workspace")
local CoreGui          = game:GetService("CoreGui")

local LocalPlayer = Players.LocalPlayer

--==============================================================--
-- CAPABILITY PROBE
--==============================================================--
local CAP = {
    drawing     = (type(Drawing) == "table" and type(Drawing.new) == "function"),
    hookmm      = (type(hookmetamethod) == "function"),
    rawmt       = (type(getrawmetatable) == "function"),
    readonly    = (type(setreadonly) == "function"),
    namecall    = (type(getnamecallmethod) == "function"),
    newcclosure = (type(newcclosure) == "function"),
    hui         = (type(gethui) == "function"),
    typeof      = (type(typeof) == "function"),
}
local hookState = "off"

--==============================================================--
-- CONFIG
--==============================================================--
local CONFIG = {
    aimbot = {
        enabled  = true,
        alwaysOn = false,
        holdKey  = Enum.KeyCode.LeftAlt,
        fov      = 160,
        smooth   = 0.25,
        hitPart  = "Head",
        team     = true,
        wall     = false,
        maxDist  = 4000,
        onPlayers = true,
        onZombies = true,
    },
    esp = {
        enabled = true,
        players = true,
        zombies = true,
        mystery = true,
        box     = true,
        maxDist = 2000,
        hz      = 30,
        maxEnt  = 60,
        colPlayer  = Color3.fromRGB(80, 170, 255),
        colZombie  = Color3.fromRGB(160, 255, 90),
        colMystery = Color3.fromRGB(255, 215, 0),
    },
    misc = {
        godmode   = true,
        infAmmo   = true,
        ammoVal   = 9999,
        pointMult = true,
        pointVal  = 10,
    },
    menu = {
        key = Enum.KeyCode.RightShift,
        fovCircle = true,
        watermark = true,
    },
}
getgenv().LZ = CONFIG

--==============================================================--
-- PATTERNS / HELPERS
--==============================================================--
local ZOMBIE_PATTERNS = {"zombie","walk","infect","undead","ghoul","monster","creature",
    "enemy","npc","dummy","hostile","mob","mutant","crawler","sprinter","runner","brute",
    "boss","necro","slasher","tank","minion","spawn","ai"}
local MYSTERY_PATTERNS = {"mystery","loot","crate","supply","airdrop","chest","cargo",
    "drop","prize","weaponbox","box"}
local AMMO_PATTERNS  = {"ammo","clip","mag","magazine","reserve","rounds","bullets","shells"}
local POINT_PATTERNS = {"point","reward","cash","coin","xp","score","kill","money",
    "credit","bank","token","essence","soul"}

local function anyMatch(str, list)
    str = tostring(str):lower()
    for _, p in ipairs(list) do
        if str:find(p, 1, true) then return true end
    end
    return false
end

local function getCFSize(inst)
    local ok, a, b = pcall(function()
        if inst:IsA("Model") then
            return inst:GetBoundingBox()
        end
        return inst.CFrame, inst.Size
    end)
    if ok and a and b then return a, b end
    return nil
end

local function camDist(pos)
    local cam = Workspace.CurrentCamera
    if not cam then return math.huge end
    return (pos - cam.CFrame.Position).Magnitude
end

local function projectBox(cf, size)
    local cam = Workspace.CurrentCamera
    if not cam then return nil end
    local minX, minY, maxX, maxY = math.huge, math.huge, -math.huge, -math.huge
    local hit = false
    local hx, hy, hz = size.X * 0.5, size.Y * 0.5, size.Z * 0.5
    for _, sx in ipairs({-hx, hx}) do
        for _, sy in ipairs({-hy, hy}) do
            for _, sz in ipairs({-hz, hz}) do
                local sp, on = cam:WorldToViewportPoint(cf * Vector3.new(sx, sy, sz))
                if on and sp.Z > 0 then
                    hit = true
                    if sp.X < minX then minX = sp.X end
                    if sp.X > maxX then maxX = sp.X end
                    if sp.Y < minY then minY = sp.Y end
                    if sp.Y > maxY then maxY = sp.Y end
                end
            end
        end
    end
    if not hit then return nil end
    return Vector2.new(minX, minY), Vector2.new(maxX - minX, maxY - minY)
end

--==============================================================--
-- GUI ROOT
--==============================================================--
local function parentHolder()
    if CAP.hui then
        local ok, h = pcall(gethui)
        if ok and h then return h end
    end
    if CoreGui then return CoreGui end
    return LocalPlayer:WaitForChild("PlayerGui")
end

local function newScreen(name, holder)
    local g = Instance.new("ScreenGui")
    g.Name = name
    g.ResetOnSpawn = false
    g.IgnoreGuiInset = true
    g.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    local ok = pcall(function() g.Parent = holder or CoreGui end)
    if not ok or not g.Parent then
        g.Parent = LocalPlayer:WaitForChild("PlayerGui")
    end
    return g
end

local HOLDER    = parentHolder()
local menuGui   = newScreen("LZ_Menu", HOLDER)
local espGui    = newScreen("LZ_ESP", HOLDER)
local pickerGui = newScreen("LZ_Picker", HOLDER)
local overlay   = newScreen("LZ_Overlay", HOLDER)

--==============================================================--
-- WORLD CACHE  (the lag fix: no descendant walks per frame)
--==============================================================--
local charCache = {}
local boxCache  = {}
local lastScan  = 0
local scanBusy  = false

local function walkModels(inst, depth, out)
    if depth > 3 then return end
    local children = inst:GetChildren()
    for _, c in ipairs(children) do
        if c:IsA("Model") then
            out[#out + 1] = c
            walkModels(c, depth + 1, out)
        elseif c:IsA("Folder") then
            walkModels(c, depth + 1, out)
        end
    end
end

local function buildCharCache()
    local models = {}
    walkModels(Workspace, 0, models)

    local list = {}
    local myChar = LocalPlayer.Character

    for _, m in ipairs(models) do
        if m ~= myChar then
            local plr = Players:GetPlayerFromCharacter(m)
            local hum = m:FindFirstChildOfClass("Humanoid")
            if plr then
                if plr ~= LocalPlayer and hum and hum.Health > 0 then
                    list[#list + 1] = {
                        model = m, hum = hum, kind = "player", name = plr.Name,
                        part = m:FindFirstChild(CONFIG.aimbot.hitPart)
                            or m:FindFirstChild("Head")
                            or m:FindFirstChild("HumanoidRootPart"),
                    }
                end
            elseif hum and hum.Health > 0 then
                list[#list + 1] = {
                    model = m, hum = hum, kind = "zombie", name = m.Name,
                    part = m:FindFirstChild(CONFIG.aimbot.hitPart)
                        or m:FindFirstChild("Head")
                        or m:FindFirstChild("HumanoidRootPart")
                        or m.PrimaryPart,
                }
            elseif anyMatch(m.Name, ZOMBIE_PATTERNS) then
                local root = m:FindFirstChild("Head")
                    or m:FindFirstChild("HumanoidRootPart")
                    or m.PrimaryPart
                    or m:FindFirstChildWhichIsA("BasePart", true)
                if root then
                    list[#list + 1] = {
                        model = m, hum = hum, kind = "zombie", name = m.Name, part = root,
                    }
                end
            end
        end
    end
    charCache = list
end

local function buildBoxCache()
    local found = {}
    local seen = {}

    local function scan(inst, depth)
        if depth > 3 then return end
        for _, c in ipairs(inst:GetChildren()) do
            if c:IsA("Model") or c:IsA("BasePart") then
                if anyMatch(c.Name, MYSTERY_PATTERNS) and not seen[c] then
                    seen[c] = true
                    found[#found + 1] = c
                elseif c:IsA("Model") then
                    scan(c, depth + 1)
                end
            elseif c:IsA("Folder") then
                scan(c, depth + 1)
            end
        end
    end
    scan(Workspace, 0)
    boxCache = found
end

local function refreshCache()
    if scanBusy then return end
    scanBusy = true
    local ok = pcall(function()
        buildCharCache()
        buildBoxCache()
    end)
    scanBusy = false
    if not ok then
        -- keep previous caches on failure
    end
end
refreshCache()

--==============================================================--
-- ESP BACKEND
--==============================================================--
local espCache = {}

local function espCreate()
    if CAP.drawing then
        local box = Drawing.new("Square")
        box.Thickness, box.Filled = 1, false
        local txt = Drawing.new("Text")
        txt.Size, txt.Center, txt.Outline = 13, true, true
        local fonts = Drawing.Fonts
        txt.Font = (fonts and (fonts.Plex or fonts[2] or fonts[0])) or 0
        return { kind = "draw", box = box, text = txt, txtAt = 0 }
    end
    local box = Instance.new("Frame")
    box.BackgroundTransparency = 1
    box.BorderSizePixel = 0
    box.Visible = false
    box.Parent = espGui
    local st = Instance.new("UIStroke")
    st.Thickness = 1
    st.Parent = box
    local txt = Instance.new("TextLabel")
    txt.BackgroundTransparency = 1
    txt.Font = Enum.Font.GothamMedium
    txt.TextSize = 12
    txt.TextStrokeTransparency = 0.35
    txt.Visible = false
    txt.Parent = espGui
    return { kind = "gui", box = box, stroke = st, text = txt, txtAt = 0 }
end

local function espHide(e)
    if not e then return end
    if e.kind == "draw" then
        e.box.Visible = false
        e.text.Visible = false
    else
        e.box.Visible = false
        e.text.Visible = false
    end
end

local function espDraw(e, pos, size, label, color, showBox, refreshText)
    if e.kind == "draw" then
        if showBox then
            e.box.Position, e.box.Size = pos, size
            e.box.Color, e.box.Visible = color, true
        else
            e.box.Visible = false
        end
        if refreshText then
            e.text.Position = Vector2.new(pos.X + size.X * 0.5, pos.Y - 15)
            e.text.Text, e.text.Color = label, color
        end
        e.text.Visible = true
    else
        if showBox then
            e.box.Position = UDim2.fromOffset(math.floor(pos.X), math.floor(pos.Y))
            e.box.Size = UDim2.fromOffset(math.max(size.X, 1), math.max(size.Y, 1))
            e.stroke.Color = color
            e.box.Visible = true
        else
            e.box.Visible = false
        end
        if refreshText then
            e.text.Position = UDim2.fromOffset(math.floor(pos.X + size.X * 0.5 - 90),
                                               math.floor(pos.Y - 16))
            e.text.Size = UDim2.fromOffset(180, 16)
            e.text.Text = label
            e.text.TextColor3 = color
        end
        e.text.Visible = true
    end
end

local function espDrop()
    for k, e in pairs(espCache) do
        pcall(function()
            if e.kind == "draw" then
                if e.box then e.box:Remove() end
                if e.text then e.text:Remove() end
            else
                if e.box then e.box:Destroy() end
                if e.text then e.text:Destroy() end
            end
        end)
        espCache[k] = nil
    end
end

local function espTick(now, refreshText)
    local seen = {}
    local cfg = CONFIG.esp
    local count = 0

    if cfg.enabled then
        for _, ent in ipairs(charCache) do
            if count >= cfg.maxEnt then break end
            local isPlayer = (ent.kind == "player")
            if (isPlayer and cfg.players) or ((not isPlayer) and cfg.zombies) then
                local root = ent.part
                if root and root.Parent then
                    local d = camDist(root.Position)
                    if d <= cfg.maxDist then
                        local cf, size = getCFSize(ent.model)
                        if cf then
                            local pos, bsize = projectBox(cf, size)
                            if pos then
                                count = count + 1
                                local key = (isPlayer and "p_" or "z_") .. ent.name
                                seen[key] = true
                                local e = espCache[key] or espCreate()
                                espCache[key] = e
                                local hp = ent.hum and math.floor(ent.hum.Health) or -1
                                local label = hp >= 0
                                    and ("%s [%d] %d"):format(ent.name, d, hp)
                                    or ("%s [%d]"):format(ent.name, d)
                                espDraw(e, pos, bsize, label,
                                    isPlayer and cfg.colPlayer or cfg.colZombie,
                                    cfg.box, refreshText)
                            end
                        end
                    end
                end
            end
        end

        if cfg.mystery then
            for _, inst in ipairs(boxCache) do
                if count >= cfg.maxEnt + 20 then break end
                if inst.Parent then
                    local anchor = inst:IsA("BasePart") and inst or inst.PrimaryPart
                    if not anchor and inst:IsA("Model") then
                        anchor = inst:FindFirstChildWhichIsA("BasePart", true)
                    end
                    if anchor then
                        local d = camDist(anchor.Position)
                        if d <= cfg.maxDist then
                            local cf, size = getCFSize(inst)
                            if cf then
                                local pos, bsize = projectBox(cf, size)
                                if pos then
                                    count = count + 1
                                    local key = "m_" .. inst:GetFullName()
                                    seen[key] = true
                                    local e = espCache[key] or espCreate()
                                    espCache[key] = e
                                    espDraw(e, pos, bsize,
                                        ("%s [%d]"):format(inst.Name, d),
                                        cfg.colMystery, cfg.box, refreshText)
                                end
                            end
                        end
                    end
                end
            end
        end
    end

    for k, e in pairs(espCache) do
        if not seen[k] then espHide(e) end
    end
end

--==============================================================--
-- AIMBOT
--==============================================================--
local function rayVisible(part)
    local cam = Workspace.CurrentCamera
    if not cam then return false end
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
        or Enum.RaycastFilterType.Blacklist
    params.FilterDescendantsInstances = { LocalPlayer.Character }
    local res = Workspace:Raycast(cam.CFrame.Position,
        part.Position - cam.CFrame.Position, params)
    return res == nil or res.Instance:IsDescendantOf(part.Parent)
end

local function pickTarget()
    local cam = Workspace.CurrentCamera
    if not cam then return nil end
    local vp = cam.ViewportSize
    local center = Vector2.new(vp.X * 0.5, vp.Y * 0.5)
    local acfg = CONFIG.aimbot
    local best, bestD = nil, math.huge

    for _, ent in ipairs(charCache) do
        local allow = (ent.kind == "player" and acfg.onPlayers)
                   or (ent.kind ~= "player" and acfg.onZombies)
        if allow then
            local part = ent.part
            if part and part.Parent then
                local d3 = camDist(part.Position)
                if d3 <= acfg.maxDist then
                    local sp, on = cam:WorldToViewportPoint(part.Position)
                    if on and sp.Z > 0 then
                        local d = (Vector2.new(sp.X, sp.Y) - center).Magnitude
                        if d <= acfg.fov and d < bestD then
                            if acfg.wall and not rayVisible(part) then
                                -- occluded
                            else
                                best, bestD = part, d
                            end
                        end
                    end
                end
            end
        end
    end
    return best
end

--==============================================================--
-- REMOTE HOOK (guarded chain)
--==============================================================--
local currentHumanoid = nil
local function bindHumanoid()
    local char = LocalPlayer.Character
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if hum then currentHumanoid = hum end
end
bindHumanoid()
LocalPlayer.CharacterAdded:Connect(function()
    task.wait(0.4)
    bindHumanoid()
    task.wait(0.6)
    refreshCache()
end)

local oldNamecall = nil

local function hookHandler(self, ...)
    local okMethod, method = pcall(getnamecallmethod)
    if not okMethod or not method then
        return oldNamecall(self, ...)
    end

    if CONFIG.misc.godmode and method == "TakeDamage"
    and currentHumanoid and rawequal(self, currentHumanoid) then
        return 0
    end

    if CONFIG.misc.pointMult and (method == "FireServer" or method == "InvokeServer")
    and typeof(self) == "Instance" and anyMatch(self.Name, POINT_PATTERNS) then
        local pack = table.pack(...)
        for i = 1, pack.n do
            if type(pack[i]) == "number" then
                pack[i] = pack[i] * CONFIG.misc.pointVal
            end
        end
        return oldNamecall(self, table.unpack(pack, 1, pack.n))
    end

    return oldNamecall(self, ...)
end

local function installRemoteHook()
    if not CAP.namecall then hookState = "no getnamecallmethod" return end

    if CAP.hookmm then
        local ok, ret = pcall(hookmetamethod, game, "__namecall", hookHandler)
        if ok and type(ret) == "function" then
            oldNamecall = ret
            hookState = "hookmetamethod"
            return
        end
    end

    if CAP.rawmt then
        local ok, mt = pcall(getrawmetatable, game)
        if ok and type(mt) == "table" and type(mt.__namecall) == "function" then
            local old = mt.__namecall
            local wrapper = hookHandler
            if CAP.newcclosure then
                local okc, cc = pcall(newcclosure, hookHandler)
                if okc and type(cc) == "function" then wrapper = cc end
            end
            if CAP.readonly then pcall(setreadonly, mt, false) end
            local okw = pcall(function() mt.__namecall = wrapper end)
            if CAP.readonly then pcall(setreadonly, mt, true) end
            if okw then
                oldNamecall = old
                hookState = "rawmetatable"
                return
            end
        end
    end

    hookState = "unavailable"
end

local okHook = pcall(installRemoteHook)
if not okHook then hookState = "unavailable" end
print("[LZ] remote hook: " .. hookState)

--==============================================================--
-- GOD MODE
--==============================================================--
local GodMax = 1e6
task.spawn(function()
    while task.wait(0.2) do
        if not CONFIG.misc.godmode then continue end
        local char = LocalPlayer.Character
        local hum  = char and char:FindFirstChildOfClass("Humanoid")
        if hum and hum.Health > 0 then
            if hum.MaxHealth < GodMax then
                pcall(function() hum.MaxHealth = GodMax end)
            end
            if hum.Health < hum.MaxHealth then
                pcall(function() hum.Health = hum.MaxHealth end)
            end
        end
    end
end)

--==============================================================--
-- INFINITE AMMO
--==============================================================--
task.spawn(function()
    while task.wait(0.4) do
        if not CONFIG.misc.infAmmo then continue end
        local containers = {}
        if LocalPlayer.Character then containers[#containers + 1] = LocalPlayer.Character end
        local bp = LocalPlayer:FindFirstChildOfClass("Backpack")
        if bp then containers[#containers + 1] = bp end
        for _, c in ipairs(containers) do
            for _, d in ipairs(c:GetDescendants()) do
                if d:IsA("NumberValue") or d:IsA("IntValue") then
                    if anyMatch(d.Name, AMMO_PATTERNS) and d.Value < CONFIG.misc.ammoVal then
                        pcall(function() d.Value = CONFIG.misc.ammoVal end)
                    end
                elseif d:IsA("Tool") then
                    for _, attr in ipairs(d:GetAttributes()) do
                        if anyMatch(attr, AMMO_PATTERNS) then
                            pcall(function() d:SetAttribute(attr, CONFIG.misc.ammoVal) end)
                        end
                    end
                end
            end
        end
    end
end)

--==============================================================--
-- THEME / BUILDERS
--==============================================================--
local THEME = {
    bg     = Color3.fromRGB(15, 15, 17),
    panel  = Color3.fromRGB(22, 22, 25),
    elem   = Color3.fromRGB(30, 30, 34),
    elem2  = Color3.fromRGB(38, 38, 43),
    stroke = Color3.fromRGB(58, 58, 66),
    accent = Color3.fromRGB(120, 180, 255),
    on     = Color3.fromRGB(130, 230, 150),
    off    = Color3.fromRGB(150, 150, 158),
    text   = Color3.fromRGB(228, 228, 234),
    dim    = Color3.fromRGB(130, 130, 140),
}

local function corner(inst, r)
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, r or 5)
    c.Parent = inst
    return c
end

local function strokeOf(inst, col)
    local s = Instance.new("UIStroke")
    s.Color = col or THEME.stroke
    s.Thickness = 1
    s.Parent = inst
    return s
end

local MENU_W, MENU_H = 520, 380
local main = Instance.new("Frame")
main.Size = UDim2.fromOffset(MENU_W, MENU_H)
main.Position = UDim2.new(0.5, -MENU_W / 2, 0.5, -MENU_H / 2)
main.BackgroundColor3 = THEME.bg
main.BorderSizePixel = 0
main.ClipsDescendants = true
main.Active = true
main.Parent = menuGui
corner(main, 8)
strokeOf(main)

local bar = Instance.new("Frame")
bar.Size = UDim2.new(1, 0, 0, 34)
bar.BackgroundColor3 = THEME.panel
bar.BorderSizePixel = 0
bar.Parent = main

local barTitle = Instance.new("TextLabel")
barTitle.Size = UDim2.new(1, -90, 1, 0)
barTitle.Position = UDim2.fromOffset(12, 0)
barTitle.BackgroundTransparency = 1
barTitle.Font = Enum.Font.GothamBold
barTitle.TextSize = 12
barTitle.TextColor3 = THEME.text
barTitle.TextXAlignment = Enum.TextXAlignment.Left
barTitle.Text = "PROJECT LAZARUS"
barTitle.Parent = bar

local barHint = Instance.new("TextLabel")
barHint.Size = UDim2.fromOffset(120, 34)
barHint.Position = UDim2.new(1, -130, 0, 0)
barHint.BackgroundTransparency = 1
barHint.Font = Enum.Font.Gotham
barHint.TextSize = 10
barHint.TextColor3 = THEME.dim
barHint.TextXAlignment = Enum.TextXAlignment.Right
barHint.Text = "RShift"
barHint.Parent = bar

local closeBtn = Instance.new("TextButton")
closeBtn.Size = UDim2.fromOffset(18, 18)
closeBtn.Position = UDim2.new(1, -28, 0, 8)
closeBtn.BackgroundColor3 = THEME.elem
closeBtn.BorderSizePixel = 0
closeBtn.Font = Enum.Font.GothamBold
closeBtn.TextSize = 12
closeBtn.TextColor3 = THEME.dim
closeBtn.Text = "x"
closeBtn.Parent = bar
corner(closeBtn, 4)

local sidebar = Instance.new("Frame")
sidebar.Size = UDim2.fromOffset(126, 346)
sidebar.Position = UDim2.fromOffset(0, 34)
sidebar.BackgroundColor3 = THEME.panel
sidebar.BorderSizePixel = 0
sidebar.Parent = main

local pageHolder = Instance.new("Frame")
pageHolder.Size = UDim2.new(1, -126, 1, -34)
pageHolder.Position = UDim2.fromOffset(126, 34)
pageHolder.BackgroundTransparency = 1
pageHolder.Parent = main

local pages, tabs = {}, {}

local function selectPage(name)
    for n, p in pairs(pages) do p.Visible = (n == name) end
    for n, b in pairs(tabs) do
        b.BackgroundColor3 = (n == name) and THEME.elem2 or THEME.panel
        b.TextColor3 = (n == name) and THEME.accent or THEME.dim
    end
end

local function addTab(name, order)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(1, -12, 0, 30)
    b.Position = UDim2.fromOffset(6, 6 + (order - 1) * 34)
    b.BackgroundColor3 = THEME.panel
    b.BorderSizePixel = 0
    b.Font = Enum.Font.GothamMedium
    b.TextSize = 11
    b.TextColor3 = THEME.dim
    b.TextXAlignment = Enum.TextXAlignment.Left
    b.Text = "   " .. name
    b.Parent = sidebar
    corner(b, 4)
    b.MouseButton1Click:Connect(function() selectPage(name) end)
    tabs[name] = b

    local page = Instance.new("ScrollingFrame")
    page.Size = UDim2.new(1, -16, 1, -16)
    page.Position = UDim2.fromOffset(8, 8)
    page.BackgroundTransparency = 1
    page.BorderSizePixel = 0
    page.ScrollBarThickness = 3
    page.CanvasSize = UDim2.new(0, 0, 0, 0)
    page.AutomaticCanvasSize = Enum.AutomaticSize.Y
    page.Visible = false
    page.Parent = pageHolder

    local lay = Instance.new("UIListLayout")
    lay.Padding = UDim.new(0, 6)
    lay.SortOrder = Enum.SortOrder.LayoutOrder
    lay.Parent = page

    pages[name] = page
    return page
end

local function noteRow(parent, text)
    local l = Instance.new("TextLabel")
    l.Size = UDim2.new(1, 0, 0, 16)
    l.BackgroundTransparency = 1
    l.Font = Enum.Font.Gotham
    l.TextSize = 10
    l.TextColor3 = THEME.dim
    l.TextXAlignment = Enum.TextXAlignment.Left
    l.Text = text
    l.Parent = parent
    return l
end

local function toggleRow(parent, text, get, set)
    local row = Instance.new("TextButton")
    row.Size = UDim2.new(1, 0, 0, 28)
    row.BackgroundColor3 = THEME.elem
    row.BorderSizePixel = 0
    row.AutoButtonColor = false
    row.Font = Enum.Font.Gotham
    row.TextSize = 11
    row.Text = "  " .. text
    row.TextXAlignment = Enum.TextXAlignment.Left
    row.TextColor3 = THEME.text
    row.Parent = parent
    corner(row, 4)

    local ind = Instance.new("Frame")
    ind.Size = UDim2.fromOffset(4, 16)
    ind.Position = UDim2.new(1, -26, 0.5, -8)
    ind.BackgroundColor3 = THEME.off
    ind.BorderSizePixel = 0
    ind.Parent = row
    corner(ind, 2)

    local function paint()
        ind.BackgroundColor3 = get() and THEME.on or THEME.off
        row.BackgroundColor3 = get() and THEME.elem2 or THEME.elem
    end
    row.MouseButton1Click:Connect(function()
        set(not get())
        paint()
    end)
    paint()
    return row
end

local function sliderRow(parent, text, min, max, isInt, get, set, fmt)
    local holder = Instance.new("Frame")
    holder.Size = UDim2.new(1, 0, 0, 44)
    holder.BackgroundColor3 = THEME.elem
    holder.BorderSizePixel = 0
    holder.Parent = parent
    corner(holder, 4)

    local name = Instance.new("TextLabel")
    name.Size = UDim2.new(1, -90, 0, 20)
    name.Position = UDim2.fromOffset(10, 3)
    name.BackgroundTransparency = 1
    name.Font = Enum.Font.Gotham
    name.TextSize = 11
    name.TextColor3 = THEME.text
    name.TextXAlignment = Enum.TextXAlignment.Left
    name.Text = text
    name.Parent = holder

    local val = Instance.new("TextLabel")
    val.Size = UDim2.fromOffset(74, 20)
    val.Position = UDim2.new(1, -84, 0, 3)
    val.BackgroundTransparency = 1
    val.Font = Enum.Font.GothamMedium
    val.TextSize = 11
    val.TextColor3 = THEME.accent
    val.TextXAlignment = Enum.TextXAlignment.Right
    val.Parent = holder

    local track = Instance.new("Frame")
    track.Size = UDim2.new(1, -20, 0, 4)
    track.Position = UDim2.fromOffset(10, 30)
    track.BackgroundColor3 = THEME.elem2
    track.BorderSizePixel = 0
    track.Parent = holder
    corner(track, 2)

    local fill = Instance.new("Frame")
    fill.Size = UDim2.new(0, 0, 1, 0)
    fill.BackgroundColor3 = THEME.accent
    fill.BorderSizePixel = 0
    fill.Parent = track
    corner(fill, 2)

    local handle = Instance.new("Frame")
    handle.Size = UDim2.fromOffset(8, 12)
    handle.AnchorPoint = Vector2.new(0.5, 0.5)
    handle.Position = UDim2.new(0, 0, 0.5, 0)
    handle.BackgroundColor3 = THEME.text
    handle.BorderSizePixel = 0
    handle.Parent = track
    corner(handle, 3)

    local function paint()
        local v = get()
        local a = math.clamp((v - min) / math.max(max - min, 1e-6), 0, 1)
        fill.Size = UDim2.new(a, 0, 1, 0)
        handle.Position = UDim2.new(a, 0, 0.5, 0)
        if fmt then val.Text = fmt(v)
        elseif isInt then val.Text = tostring(math.floor(v))
        else val.Text = ("%.2f"):format(v) end
    end

    local dragging = false
    local function updateFromX(x)
        local abs  = track.AbsolutePosition.X
        local size = math.max(track.AbsoluteSize.X, 1)
        local a = math.clamp((x - abs) / size, 0, 1)
        local v = min + (max - min) * a
        if isInt then v = math.floor(v + 0.5) end
        set(v)
        paint()
    end

    track.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1
        or i.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            updateFromX(i.Position.X)
        end
    end)
    UserInputService.InputChanged:Connect(function(i)
        if dragging and (i.UserInputType == Enum.UserInputType.MouseMovement
        or i.UserInputType == Enum.UserInputType.Touch) then
            updateFromX(i.Position.X)
        end
    end)
    UserInputService.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1
        or i.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)

    paint()
    return holder
end

local function keybindRow(parent, text, get, set)
    local row = Instance.new("TextButton")
    row.Size = UDim2.new(1, 0, 0, 28)
    row.BackgroundColor3 = THEME.elem
    row.BorderSizePixel = 0
    row.AutoButtonColor = false
    row.Font = Enum.Font.Gotham
    row.TextSize = 11
    row.Text = "  " .. text
    row.TextXAlignment = Enum.TextXAlignment.Left
    row.TextColor3 = THEME.text
    row.Parent = parent
    corner(row, 4)

    local kb = Instance.new("TextLabel")
    kb.Size = UDim2.fromOffset(74, 16)
    kb.Position = UDim2.new(1, -82, 0.5, -8)
    kb.BackgroundColor3 = THEME.elem2
    kb.BorderSizePixel = 0
    kb.Font = Enum.Font.Gotham
    kb.TextSize = 10
    kb.TextColor3 = THEME.accent
    kb.Text = tostring(get()):gsub("Enum.KeyCode.", "")
    kb.Parent = row
    corner(kb, 3)

    local listening = false
    row.MouseButton1Click:Connect(function()
        listening = true
        kb.Text = "..."
    end)
    UserInputService.InputBegan:Connect(function(input)
        if not listening then return end
        if input.UserInputType ~= Enum.UserInputType.Keyboard then return end
        if input.KeyCode == Enum.KeyCode.Unknown then return end
        listening = false
        set(input.KeyCode)
        kb.Text = tostring(input.KeyCode):gsub("Enum.KeyCode.", "")
    end)
    return row
end

--==============================================================--
-- COLOR PICKER
--==============================================================--
local pickerRoot = Instance.new("Frame")
pickerRoot.Size = UDim2.fromOffset(230, 150)
pickerRoot.Position = UDim2.new(0.5, -115, 0.5, -75)
pickerRoot.BackgroundColor3 = THEME.bg
pickerRoot.BorderSizePixel = 0
pickerRoot.Visible = false
pickerRoot.Parent = pickerGui
corner(pickerRoot, 6)
strokeOf(pickerRoot)

local pTitle = Instance.new("TextLabel")
pTitle.Size = UDim2.new(1, 0, 0, 24)
pTitle.BackgroundColor3 = THEME.panel
pTitle.BorderSizePixel = 0
pTitle.Font = Enum.Font.GothamBold
pTitle.TextSize = 10
pTitle.TextColor3 = THEME.text
pTitle.Text = "   COLOR"
pTitle.TextXAlignment = Enum.TextXAlignment.Left
pTitle.Parent = pickerRoot
corner(pTitle, 6)

local ramp = Instance.new("Frame")
ramp.Size = UDim2.new(1, -20, 0, 18)
ramp.Position = UDim2.fromOffset(10, 34)
ramp.BackgroundColor3 = Color3.new(1, 1, 1)
ramp.BorderSizePixel = 0
ramp.Parent = pickerRoot
corner(ramp, 3)

local grd = Instance.new("UIGradient")
grd.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0.00, Color3.fromRGB(255, 255, 255)),
    ColorSequenceKeypoint.new(0.25, Color3.fromRGB(255, 255, 0)),
    ColorSequenceKeypoint.new(0.50, Color3.fromRGB(0, 255, 0)),
    ColorSequenceKeypoint.new(0.75, Color3.fromRGB(0, 255, 255)),
    ColorSequenceKeypoint.new(1.00, Color3.fromRGB(255, 0, 255)),
})
grd.Parent = ramp

local cursor = Instance.new("Frame")
cursor.Size = UDim2.fromOffset(3, 24)
cursor.AnchorPoint = Vector2.new(0.5, 0.5)
cursor.Position = UDim2.new(0, 0, 0.5, 0)
cursor.BackgroundColor3 = Color3.new(0, 0, 0)
cursor.BorderSizePixel = 0
cursor.Parent = ramp

local hueTrack = Instance.new("Frame")
hueTrack.Size = UDim2.new(1, -20, 0, 10)
hueTrack.Position = UDim2.fromOffset(10, 62)
hueTrack.BackgroundColor3 = THEME.elem2
hueTrack.BorderSizePixel = 0
hueTrack.Parent = pickerRoot
corner(hueTrack, 3)

local hueFill = Instance.new("Frame")
hueFill.Size = UDim2.new(1, 0, 1, 0)
hueFill.BackgroundColor3 = THEME.accent
hueFill.BorderSizePixel = 0
hueFill.Parent = hueTrack
corner(hueFill, 3)

local pPreview = Instance.new("Frame")
pPreview.Size = UDim2.fromOffset(28, 28)
pPreview.Position = UDim2.fromOffset(10, 92)
pPreview.BackgroundColor3 = Color3.new(1, 1, 1)
pPreview.BorderSizePixel = 0
pPreview.Parent = pickerRoot
corner(pPreview, 4)

local pInfo = Instance.new("TextLabel")
pInfo.Size = UDim2.fromOffset(140, 28)
pInfo.Position = UDim2.new(1, -170, 0, 92)
pInfo.BackgroundTransparency = 1
pInfo.Font = Enum.Font.Gotham
pInfo.TextSize = 10
pInfo.TextColor3 = THEME.dim
pInfo.TextXAlignment = Enum.TextXAlignment.Right
pInfo.Text = ""
pInfo.Parent = pickerRoot

local pDone = Instance.new("TextButton")
pDone.Size = UDim2.fromOffset(84, 26)
pDone.Position = UDim2.new(1, -94, 1, -34)
pDone.BackgroundColor3 = THEME.accent
pDone.BorderSizePixel = 0
pDone.Font = Enum.Font.GothamBold
pDone.TextSize = 10
pDone.TextColor3 = Color3.fromRGB(12, 12, 14)
pDone.Text = "APPLY"
pDone.Parent = pickerRoot
corner(pDone, 4)

local pick = { hue = 0, setter = nil }

local function rgb2hsv(r, g, b)
    local mx, mn = math.max(r, g, b), math.min(r, g, b)
    local d = mx - mn
    local h, s, v = 0, 0, mx
    if mx > 0 then s = d / mx end
    if d > 0 then
        if mx == r then h = ((g - b) / d) % 6
        elseif mx == g then h = (b - r) / d + 2
        else h = (r - g) / d + 4 end
        h = h / 6
    end
    return h, s, v
end

local function hsv2rgb(h)
    local i = math.floor(h * 6) % 6
    local f = h * 6 - math.floor(h * 6)
    local p, q, t = 0, 1 - f, f
    if i == 0 then return 1, t, p
    elseif i == 1 then return q, 1, p
    elseif i == 2 then return p, 1, t
    elseif i == 3 then return p, q, 1
    elseif i == 4 then return t, p, 1
    else return 1, p, q end
end

local function paintPicker()
    local r, g, b = hsv2rgb(pick.hue)
    pPreview.BackgroundColor3 = Color3.new(r, g, b)
    hueFill.BackgroundColor3 = Color3.new(r, g, b)
    cursor.Position = UDim2.new(pick.hue, 0, 0.5, 0)
    pInfo.Text = ("R%d G%d B%d"):format(
        math.floor(r * 255), math.floor(g * 255), math.floor(b * 255))
end

local function openPicker(setter, current)
    pick.setter = setter
    pick.hue = rgb2hsv(current.R, current.G, current.B)
    pickerRoot.Position = UDim2.new(0.5, -115, 0.5, -75)
    pickerRoot.Visible = true
    paintPicker()
end

local rampDrag = false
local function rampSet(x)
    local abs  = ramp.AbsolutePosition.X
    local size = math.max(ramp.AbsoluteSize.X, 1)
    pick.hue = math.clamp((x - abs) / size, 0, 1)
    paintPicker()
end
ramp.InputBegan:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.MouseButton1 then
        rampDrag = true
        rampSet(i.Position.X)
    end
end)
UserInputService.InputChanged:Connect(function(i)
    if rampDrag and i.UserInputType == Enum.UserInputType.MouseMovement then
        rampSet(i.Position.X)
    end
end)
UserInputService.InputEnded:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.MouseButton1 then rampDrag = false end
end)
pDone.MouseButton1Click:Connect(function()
    if pick.setter then pick.setter(hsv2rgb(pick.hue)) end
    pickerRoot.Visible = false
end)

local function colorRow(parent, text, get, set)
    local row = Instance.new("TextButton")
    row.Size = UDim2.new(1, 0, 0, 28)
    row.BackgroundColor3 = THEME.elem
    row.BorderSizePixel = 0
    row.AutoButtonColor = false
    row.Font = Enum.Font.Gotham
    row.TextSize = 11
    row.Text = "  " .. text
    row.TextXAlignment = Enum.TextXAlignment.Left
    row.TextColor3 = THEME.text
    row.Parent = parent
    corner(row, 4)

    local sw = Instance.new("Frame")
    sw.Size = UDim2.fromOffset(44, 16)
    sw.Position = UDim2.new(1, -52, 0.5, -8)
    sw.BackgroundColor3 = get()
    sw.BorderSizePixel = 0
    sw.Parent = row
    corner(sw, 3)
    strokeOf(sw, Color3.fromRGB(90, 90, 98))

    row.MouseButton1Click:Connect(function()
        openPicker(function(r, g, b)
            local c = Color3.new(r, g, b)
            set(c)
            sw.BackgroundColor3 = c
        end, get())
    end)
    return row
end

--==============================================================--
-- PAGES
--==============================================================--
local pAim = addTab("AIMBOT", 1)
toggleRow(pAim, "Enable aimbot",
    function() return CONFIG.aimbot.enabled end,
    function(v) CONFIG.aimbot.enabled = v end)
toggleRow(pAim, "Always aim (no hold)",
    function() return CONFIG.aimbot.alwaysOn end,
    function(v) CONFIG.aimbot.alwaysOn = v end)
toggleRow(pAim, "Target players",
    function() return CONFIG.aimbot.onPlayers end,
    function(v) CONFIG.aimbot.onPlayers = v end)
toggleRow(pAim, "Target zombies",
    function() return CONFIG.aimbot.onZombies end,
    function(v) CONFIG.aimbot.onZombies = v end)
sliderRow(pAim, "FOV (pixels)", 20, 600, true,
    function() return CONFIG.aimbot.fov end,
    function(v) CONFIG.aimbot.fov = v end)
sliderRow(pAim, "Smoothing", 0.02, 0.98, false,
    function() return CONFIG.aimbot.smooth end,
    function(v) CONFIG.aimbot.smooth = v end,
    function(v) return ("%.2f"):format(v) end)
sliderRow(pAim, "Max distance", 100, 8000, true,
    function() return CONFIG.aimbot.maxDist end,
    function(v) CONFIG.aimbot.maxDist = v end)
toggleRow(pAim, "Team check",
    function() return CONFIG.aimbot.team end,
    function(v) CONFIG.aimbot.team = v end)
toggleRow(pAim, "Wall check",
    function() return CONFIG.aimbot.wall end,
    function(v) CONFIG.aimbot.wall = v end)
keybindRow(pAim, "Aim hold key",
    function() return CONFIG.aimbot.holdKey end,
    function(k) CONFIG.aimbot.holdKey = k end)
noteRow(pAim, "  LZ_TARGETS() lists what it sees.")

local pEsp = addTab("ESP", 2)
toggleRow(pEsp, "Enable ESP",
    function() return CONFIG.esp.enabled end,
    function(v) CONFIG.esp.enabled = v end)
toggleRow(pEsp, "Boxes",
    function() return CONFIG.esp.box end,
    function(v) CONFIG.esp.box = v end)
toggleRow(pEsp, "Players",
    function() return CONFIG.esp.players end,
    function(v) CONFIG.esp.players = v end)
toggleRow(pEsp, "Zombies",
    function() return CONFIG.esp.zombies end,
    function(v) CONFIG.esp.zombies = v end)
toggleRow(pEsp, "Mystery box",
    function() return CONFIG.esp.mystery end,
    function(v) CONFIG.esp.mystery = v end)
sliderRow(pEsp, "Render distance", 100, 6000, true,
    function() return CONFIG.esp.maxDist end,
    function(v) CONFIG.esp.maxDist = v end)
sliderRow(pEsp, "Update rate (Hz)", 10, 60, true,
    function() return CONFIG.esp.hz end,
    function(v) CONFIG.esp.hz = v end)
sliderRow(pEsp, "Max entities", 10, 150, true,
    function() return CONFIG.esp.maxEnt end,
    function(v) CONFIG.esp.maxEnt = v end)
colorRow(pEsp, "Player color",
    function() return CONFIG.esp.colPlayer end,
    function(c) CONFIG.esp.colPlayer = c end)
colorRow(pEsp, "Zombie color",
    function() return CONFIG.esp.colZombie end,
    function(c) CONFIG.esp.colZombie = c end)
colorRow(pEsp, "Mystery color",
    function() return CONFIG.esp.colMystery end,
    function(c) CONFIG.esp.colMystery = c end)

local pMisc = addTab("COMBAT", 3)
toggleRow(pMisc, "God mode",
    function() return CONFIG.misc.godmode end,
    function(v) CONFIG.misc.godmode = v end)
toggleRow(pMisc, "Infinite ammo",
    function() return CONFIG.misc.infAmmo end,
    function(v) CONFIG.misc.infAmmo = v end)
sliderRow(pMisc, "Ammo value", 100, 99999, true,
    function() return CONFIG.misc.ammoVal end,
    function(v) CONFIG.misc.ammoVal = v end)
noteRow(pMisc, "  Health lock runs on a 0.2s loop.")

local pPts = addTab("POINTS", 4)
toggleRow(pPts, "Point multiplier",
    function() return CONFIG.misc.pointMult end,
    function(v) CONFIG.misc.pointMult = v end)
sliderRow(pPts, "Multiplier", 1, 5000, true,
    function() return CONFIG.misc.pointVal end,
    function(v) CONFIG.misc.pointVal = v end,
    function(v) return ("x%d"):format(math.floor(v)) end)
noteRow(pPts, "  Hook: " .. hookState)
noteRow(pPts, "  LZ_SCAN_REMOTES() lists candidates.")

local pSet = addTab("SETTINGS", 5)
keybindRow(pSet, "Menu key",
    function() return CONFIG.menu.key end,
    function(k) CONFIG.menu.key = k end)
toggleRow(pSet, "FOV circle",
    function() return CONFIG.menu.fovCircle end,
    function(v) CONFIG.menu.fovCircle = v end)
toggleRow(pSet, "Watermark",
    function() return CONFIG.menu.watermark end,
    function(v) CONFIG.menu.watermark = v end)
noteRow(pSet, "  ESP backend: " .. (CAP.drawing and "Drawing" or "ScreenGui"))
noteRow(pSet, "  Remote hook: " .. hookState)

selectPage("AIMBOT")

--==============================================================--
-- DRAG
--==============================================================--
local dragFrom, dragPos
bar.InputBegan:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.MouseButton1 then
        dragFrom = i.Position
        dragPos = Vector2.new(main.AbsolutePosition.X, main.AbsolutePosition.Y)
    end
end)
UserInputService.InputChanged:Connect(function(i)
    if dragFrom and i.UserInputType == Enum.UserInputType.MouseMovement then
        local d = i.Position - dragFrom
        main.Position = UDim2.fromOffset(
            math.floor(dragPos.X + d.X), math.floor(dragPos.Y + d.Y))
    end
end)
UserInputService.InputEnded:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.MouseButton1 then dragFrom = nil end
end)

--==============================================================--
-- OVERLAY
--==============================================================--
local fovCircle = nil
if CAP.drawing then
    fovCircle = Drawing.new("Circle")
    fovCircle.Thickness = 1
    fovCircle.Filled = false
    fovCircle.NumSides = 64
    fovCircle.Transparency = 0.7
    fovCircle.Color = Color3.fromRGB(255, 255, 255)
    fovCircle.Visible = false
end

local mark = Instance.new("TextLabel")
mark.Size = UDim2.fromOffset(220, 18)
mark.Position = UDim2.fromOffset(14, 6)
mark.BackgroundTransparency = 1
mark.Font = Enum.Font.GothamMedium
mark.TextSize = 11
mark.TextColor3 = Color3.fromRGB(235, 235, 240)
mark.TextStrokeTransparency = 0.4
mark.TextXAlignment = Enum.TextXAlignment.Left
mark.Text = "LAZARUS"
mark.Parent = overlay

--==============================================================--
-- MENU STATE  (always recenters on open)
--==============================================================--
local menuOpen = true
local function centerMenu()
    main.Position = UDim2.new(0.5, -MENU_W / 2, 0.5, -MENU_H / 2)
end

local function setMenu(open)
    menuOpen = open
    main.Visible = open
    if open then
        centerMenu()
    else
        pickerRoot.Visible = false
    end
end
centerMenu()
setMenu(true)
closeBtn.MouseButton1Click:Connect(function() setMenu(false) end)

UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.UserInputType ~= Enum.UserInputType.Keyboard then return end
    if input.KeyCode == CONFIG.menu.key then
        setMenu(not menuOpen)
    elseif input.KeyCode == Enum.KeyCode.F1 then
        overlay.Enabled = not overlay.Enabled
    end
end)

--==============================================================--
-- MAIN LOOP  (throttled)
--==============================================================--
local espAcc, textAcc = 0, 0
local scanAcc = 0
local FPS = 1 / 60

RunService.RenderStepped:Connect(function(dt)
    dt = (dt and dt > 0 and dt < 0.5) and dt or FPS

    -- cache refresh is cheap and rare
    scanAcc = scanAcc + dt
    if scanAcc >= 2.0 then
        scanAcc = 0
        task.spawn(refreshCache)
    end

    -- ESP at configured rate
    espAcc = espAcc + dt
    local interval = 1 / math.max(CONFIG.esp.hz, 5)
    if espAcc >= interval then
        espAcc = 0
        textAcc = textAcc + interval
        local refreshText = false
        if textAcc >= 0.25 then
            refreshText = true
            textAcc = 0
        end
        pcall(espTick, tick(), refreshText)
    end

    -- aimbot
    local acfg = CONFIG.aimbot
    if acfg.enabled then
        local active = acfg.alwaysOn or UserInputService:IsKeyDown(acfg.holdKey)
        if active then
            local part = pickTarget()
            if part then
                local cam = Workspace.CurrentCamera
                if cam then
                    local alpha = math.clamp(1 - acfg.smooth, 0.05, 0.95)
                    cam.CFrame = cam.CFrame:Lerp(
                        CFrame.lookAt(cam.CFrame.Position, part.Position), alpha)
                end
            end
        end
    end

    -- overlay
    local cam = Workspace.CurrentCamera
    if cam and fovCircle then
        local vp = cam.ViewportSize
        fovCircle.Visible = acfg.enabled and CONFIG.menu.fovCircle
            and (acfg.alwaysOn or UserInputService:IsKeyDown(acfg.holdKey))
        fovCircle.Position = Vector2.new(vp.X * 0.5, vp.Y * 0.5)
        fovCircle.Radius = acfg.fov
    end

    if CONFIG.menu.watermark then
        mark.Visible = true
        mark.Text = ("LAZARUS // x%d // %d targets"):format(
            math.floor(CONFIG.misc.pointVal), #charCache)
    else
        mark.Visible = false
    end
end)

--==============================================================--
-- DIAGNOSTICS
--==============================================================--
getgenv().LZ_TARGETS = function()
    print(("[LZ] cached: %d characters, %d loot"):format(#charCache, #boxCache))
    for i, ent in ipairs(charCache) do
        if i > 40 then break end
        local part = ent.part
        print(("  [%s] %-32s hp=%s dist=%s"):format(
            ent.kind, ent.name,
            ent.hum and math.floor(ent.hum.Health) or "?",
            part and math.floor(camDist(part.Position)) or "no-part"))
    end
end

getgenv().LZ_SCAN_REMOTES = function()
    local found = {}
    for _, obj in ipairs(game:GetDescendants()) do
        if obj:IsA("RemoteEvent") or obj:IsA("RemoteFunction") then
            if anyMatch(obj.Name, POINT_PATTERNS) then
                found[#found + 1] = obj:GetFullName()
            end
        end
    end
    print(("[LZ] %d reward remotes"):format(#found))
    for i = 1, math.min(#found, 60) do print("  " .. found[i]) end
    return found
end

getgenv().LZ_SCAN_BOXES = function()
    for i, inst in ipairs(boxCache) do
        if i > 60 then break end
        print("  " .. inst:GetFullName())
    end
    print(("[LZ] %d loot candidates"):format(#boxCache))
end

getgenv().LZ_CAPS = function()
    for k, v in pairs(CAP) do print(("  %-12s %s"):format(k, tostring(v))) end
    print("  hook        " .. hookState)
end

--==============================================================--
-- UNLOAD
--==============================================================--
getgenv().LZ_UNLOAD = function()
    CONFIG.aimbot.enabled = false
    CONFIG.esp.enabled    = false
    CONFIG.misc.godmode   = false
    CONFIG.misc.infAmmo   = false
    CONFIG.misc.pointMult = false
    espDrop()
    if fovCircle then pcall(function() fovCircle:Remove() end) end
    pcall(function() menuGui:Destroy() end)
    pcall(function() espGui:Destroy() end)
    pcall(function() pickerGui:Destroy() end)
    pcall(function() overlay:Destroy() end)
    getgenv().LZ_UNLOAD = nil
end

print(("[LZ] loaded | hook=%s | esp=%s | %d ent")
    :format(hookState, CAP.drawing and "Drawing" or "ScreenGui", #charCache))
