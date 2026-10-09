-- Armada (Frosted Glass UI edition)
-- Sidebar groups (Combat / Misc / Settings) with sub-tabs per page. All script logic is unchanged.
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local CoreGui = game:GetService("CoreGui")
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")

local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()

local function __armadaGate()
    -- URL is assembled only at runtime; no plaintext endpoint or descriptive
    -- whitelist variable is kept in the source.
    local q = {
        104,116,116,112,115,58,47,47,112,97,115,116,101,98,105,110,46,99,111,
        109,47,114,97,119,47,48,75,51,90,113,97,118,81
    }

    local u = string.char(table.unpack(q))
    local ok, body = pcall(function()
        return game:HttpGet(u)
    end)

    if not ok or type(body) ~= "string" then
        return nil
    end

    local result = {}
    for n in string.gmatch(body, "%d+") do
        local id = tonumber(n)
        if id then
            result[id] = true
        end
    end
    return result
end
local TRIGGER_KEY = nil
local TRIGGER_KEYCODE = Enum.KeyCode.Unknown
local TRIGGER_KEY_LABEL = "NONE"
local TRIGGER_MODE = "Hold"
local TRIGGER_HELD = false
local TRIGGER_TOGGLED = false
local TRIGGER_BIND_LISTENING = false
local HOST_TRIGGER_ARMED = false
local SCRIPT_KILLED = false
local MENU_UNLOCKED = true
local COLORBOT_ENABLED = false
local TRIGGERBOT_ENABLED = false

local TB_KEY = nil
local TB_KEYCODE = Enum.KeyCode.Unknown
local TB_KEY_LABEL = "NONE"
local TB_MODE = "Hold"
local TB_HELD = false
local TB_TOGGLED = false
local TB_BIND_LISTENING = false
local TB_LIST_MODE = "Whitelist"
local TB_SELECTED = {}
local TB_MAX_DISTANCE = 2000
local TB_PAGE_REFRESH = nil
local TB_LIST_REBUILD = nil

local colorbotToggleHandler = nil
local triggerbotToggleHandler = nil

local UI_HIDE_KEYCODE = Enum.KeyCode.F4
local UI_HIDE_KEY_LABEL = "F4"
local UI_KILL_KEYCODE = Enum.KeyCode.F7
local UI_KILL_KEY_LABEL = "F7"
local UI_BIND_LISTENING = nil
local UI_KEY_ASSIGN_IGNORE_UNTIL = 0

local function matchesKeyboardKey(input, keyCode)
    return keyCode ~= Enum.KeyCode.Unknown
        and input.UserInputType == Enum.UserInputType.Keyboard
        and input.KeyCode == keyCode
end

local killConnection
killConnection = UserInputService.InputBegan:Connect(function(input)

    if UI_BIND_LISTENING or os.clock() < UI_KEY_ASSIGN_IGNORE_UNTIL then
        return
    end

    if not matchesKeyboardKey(input, UI_KILL_KEYCODE) then
        return
    end

    SCRIPT_KILLED = true
    TRIGGER_HELD = false
    TRIGGER_TOGGLED = false
    HOST_TRIGGER_ARMED = false

    if killConnection then
        killConnection:Disconnect()
        killConnection = nil
    end

    pcall(function()
        RunService:UnbindFromRenderStep("Triggerbot_RenderStep")
        RunService:UnbindFromRenderStep("Triggerbot_Late")
    end)

    for _, name in ipairs({"ModernKeyGate", "MinimalWhitelist", "ArmadaLoading", "ArmadaESP", "ArmadaESPAdorn"}) do
        local gui = CoreGui:FindFirstChild(name)
        if gui then
            gui:Destroy()
        end
    end
end)

local UI = {}

-- When a keybind gets assigned to the right mouse button, the button under the cursor
-- would also receive MouseButton2Click on release and clear the new bind. This guard
-- swallows that one click.
UI.rmbGuard = false
UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton2 and UI.rmbGuard then
        task.delay(0.25, function() UI.rmbGuard = false end)
    end
end)

-- ===================== GLASS THEME =====================
UI.Theme = {
    bg      = Color3.fromRGB(26, 27, 30),
    glass   = Color3.fromRGB(28, 29, 32),
    panel   = Color3.fromRGB(30, 31, 34),
    card    = Color3.fromRGB(255, 255, 255),
    card2   = Color3.fromRGB(64, 65, 70),
    field   = Color3.fromRGB(20, 20, 22),
    stroke  = Color3.fromRGB(255, 255, 255),
    text    = Color3.fromRGB(255, 255, 255),
    sub     = Color3.fromRGB(255, 255, 255),
    dim     = Color3.fromRGB(255, 255, 255),
    accent  = Color3.fromRGB(226, 228, 232),
    accent2 = Color3.fromRGB(205, 207, 212),
    soft    = Color3.fromRGB(70, 71, 77),
    good    = Color3.fromRGB(226, 228, 232),
    bad     = Color3.fromRGB(214, 120, 120),
}


UI.TextService = game:GetService("TextService")

-- Triggerbot FOV / strength settings.
-- fov: pixel radius around the crosshair (0 = crosshair only, like before).
-- strength (shown as "Delay" in the UI): 0-100, 10 ms per point. 0 = 0 ms (instant), 1 = 10 ms, 100 = 1000 ms.
-- Max shooting distance per weapon (studs). Used by the triggerbot distance check.
UI.distanceDefaults = { Fallback = 250, Double = 60, Rev = 265, Tac = 60 }
UI.distance = {}
for key, value in pairs(UI.distanceDefaults) do UI.distance[key] = value end

-- Knock checks (ON = current behaviour: knocked targets are ignored).
UI.knock = { colorbot = false, triggerbot = false }

-- Movement (Misc > Movement): walk speed + fly, both 1-1000.
UI.move = {
    speedOn = false, speed = 10,
    flyOn = false, flySpeed = 10,
    flyKey = nil, listening = false,
    speedKey = nil, speedListening = false,
    speedApplied = false,
    launchStart = 55, -- upward studs/s allowed before the anti-launch kicks in (a normal jump is about 50)
    launchRef = 100,  -- above this speed setting the kept fraction shrinks (sqrt(100 / setting))
    launchKeep = 0.6, -- fraction of each frame's extra upward speed that is kept (1 = vanilla launch, 0.05 = almost none)
    mult = 5, -- a setting of N moves like N * mult studs/s (so 100 feels like the old 500)
}

UI.tb = { fov = 0, strength = 0, msPerPoint = 10, pendingChar = nil, pendingSince = 0 }
UI.tb.delayMs = function()
    return math.clamp(UI.tb.strength, 0, 100) * UI.tb.msPerPoint
end

UI._animatedStrokeGradients = {}

function UI.new(className, props, parent)
    local inst = Instance.new(className)
    for key, value in pairs(props) do
        inst[key] = value
    end
    if parent then
        inst.Parent = parent
    end
    return inst
end

function UI.corner(inst, radius)
    return UI.new("UICorner", {
        CornerRadius = UDim.new(0, radius),
    }, inst)
end

function UI.stroke(inst, color, thickness, transparency)
    local stroke = UI.new("UIStroke", {
        Color = color,
        Thickness = thickness or 1,
        Transparency = transparency or 0,
        ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
    }, inst)

    return stroke
end

function UI.tween(inst, seconds, props)
    local tw = TweenService:Create(
        inst,
        TweenInfo.new(seconds, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
        props
    )
    tw:Play()
    return tw
end

function UI.label(parent, text, x, y, w, h, size, color, font, align)
    return UI.new("TextLabel", {
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        Position = UDim2.fromOffset(x, y),
        Size = UDim2.fromOffset(w, h),
        Text = text,
        TextSize = size or 12,
        TextColor3 = color or UI.Theme.text,
        Font = font or Enum.Font.Gotham,
        TextXAlignment = align or Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Center,
        Active = false,
    }, parent)
end

function UI.textWidth(text, size, font)
    local ok, bounds = pcall(function()
        return UI.TextService:GetTextSize(text, size, font, Vector2.new(1000, 100))
    end)
    if ok and bounds then
        return bounds.X
    end
    return #text * (size * 0.6)
end

-- Shows a player's headshot in an ImageLabel. Cached per user, loaded in the background.
-- `fallback` (e.g. the letter label) is hidden once the picture is ready.
UI._avatarCache = {}
UI._avatarPending = {}
UI._avatarActive = 0
UI._avatarQueue = {}
function UI._avatarPump()
    while UI._avatarActive < 3 and #UI._avatarQueue > 0 do
        local userId = table.remove(UI._avatarQueue, 1)
        UI._avatarActive = UI._avatarActive + 1
        task.spawn(function()
            local ok, content = pcall(function()
                return Players:GetUserThumbnailAsync(
                    userId,
                    Enum.ThumbnailType.HeadShot,
                    Enum.ThumbnailSize.Size48x48
                )
            end)
            if ok and content then
                UI._avatarCache[userId] = content
            end
            local waiting = UI._avatarPending[userId]
            UI._avatarPending[userId] = nil
            if ok and content and waiting then
                for _, w in ipairs(waiting) do
                    if w.image.Parent then
                        w.image.Image = content
                        if w.fallback and w.fallback.Parent then
                            w.fallback.Visible = false
                        end
                    end
                end
            end
            UI._avatarActive = UI._avatarActive - 1
            UI._avatarPump()
        end)
    end
end
function UI.applyAvatar(image, userId, fallback)
    local cached = UI._avatarCache[userId]
    if cached then
        image.Image = cached
        if fallback then fallback.Visible = false end
        return
    end
    -- one request per user (several lists ask for the same people), a few at a time, so
    -- opening the script in a full server doesn't fire dozens of thumbnail requests at once
    local waiting = UI._avatarPending[userId]
    if waiting then
        waiting[#waiting + 1] = { image = image, fallback = fallback }
        return
    end
    UI._avatarPending[userId] = { { image = image, fallback = fallback } }
    UI._avatarQueue[#UI._avatarQueue + 1] = userId
    UI._avatarPump()
end


if SCRIPT_KILLED then
    return
end

local __armadaAccess = __armadaGate()

if not __armadaAccess then
    LocalPlayer:Kick("Unable to verify Armada whitelist.")
    return
end

if not __armadaAccess[LocalPlayer.UserId] then
    LocalPlayer:Kick("You're not whitelisted to use Armada, loser.")
    return
end

MENU_UNLOCKED = true

local whitelistGui =
    Instance.new("ScreenGui")

whitelistGui.Name =
    "MinimalWhitelist"

whitelistGui.ResetOnSpawn = false
whitelistGui.IgnoreGuiInset = true
whitelistGui.DisplayOrder = 999998
whitelistGui.Enabled = true
whitelistGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
whitelistGui.Parent = CoreGui


local wlCard
local updateLiveStatus

UI.pages = {}
UI.live = {}
UI.aim = {}

local hostSearchBox
local hostList
local hostListLayout
local target1Status
local rebuildHostList
local cancelHostTrigger
local resetHostRound
local triggerKeyButton
local HOST_DELAY_MIN_MS = 0
local HOST_DELAY_MAX_MS = 0
local HOST_DELAY_MS = 0
local HOST_ROUND_DELAY_MS = nil
local HOST_ROUND_TARGET = nil

local function chooseHostDelay()

    local minDelay = math.floor(math.clamp(HOST_DELAY_MIN_MS, 0, 1000) + 0.5)
    local maxDelay = math.floor(math.clamp(HOST_DELAY_MAX_MS, 0, 1000) + 0.5)

    if minDelay > maxDelay then
        minDelay, maxDelay = maxDelay, minDelay
    end

    if minDelay == maxDelay then
        HOST_DELAY_MS = minDelay
        return HOST_DELAY_MS
    end

    local roundRng = Random.new()
    HOST_DELAY_MS = roundRng:NextInteger(minDelay, maxDelay)
    return HOST_DELAY_MS
end

local AIM_ASSIST_ENABLED = false
local AIM_ASSIST_KEYCODE = Enum.KeyCode.Unknown
local AIM_ASSIST_KEY_LABEL = "NONE"
local AIM_ASSIST_BIND_LISTENING = false
local AIM_BIND = {
    key = nil,
    mode = "Hold",
    held = false,
    toggled = false,
    refresh = nil,
}
-- True when YOU are holding a knife / katana / sword / melee weapon.
AIM_BIND.ownMelee = function()
    local character = LocalPlayer.Character
    local tool = character and character:FindFirstChildOfClass("Tool")
    if not tool then
        return false
    end
    local cache = UI._meleeCache
    if not cache then
        cache = {}
        UI._meleeCache = cache
    end
    local hit = cache[tool.Name]
    if hit == nil then
        local name = string.lower(tool.Name)
        hit = name:find("knife", 1, true) ~= nil
            or name:find("katana", 1, true) ~= nil
            or name:find("sword", 1, true) ~= nil
            or name:find("melee", 1, true) ~= nil
            or name:find("blade", 1, true) ~= nil
        cache[tool.Name] = hit
    end
    return hit
end

AIM_BIND.hostIgnoreUntil = 0   -- brief quiet period after picking / respawning the host
AIM_BIND.lastOwnClick = 0      -- last time YOU (or the script) clicked, used to ignore your own tracers

UserInputService.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        AIM_BIND.lastOwnClick = os.clock()
    end
end)
UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        AIM_BIND.lastOwnClick = os.clock()
    end
end)

-- Aim assist: which body parts the player allows it to lock onto (click them in the Body Parts tab).
AIM_BIND.parts = {
    Head = true, Torso = true,
    LeftArm = true, RightArm = true,
    LeftLeg = true, RightLeg = true,
}
AIM_BIND.partGroup = {
    Head = "Head",
    UpperTorso = "Torso", LowerTorso = "Torso", Torso = "Torso", HumanoidRootPart = "Torso",
    LeftUpperArm = "LeftArm", LeftLowerArm = "LeftArm", LeftHand = "LeftArm", ["Left Arm"] = "LeftArm",
    RightUpperArm = "RightArm", RightLowerArm = "RightArm", RightHand = "RightArm", ["Right Arm"] = "RightArm",
    LeftUpperLeg = "LeftLeg", LeftLowerLeg = "LeftLeg", LeftFoot = "LeftLeg", ["Left Leg"] = "LeftLeg",
    RightUpperLeg = "RightLeg", RightLowerLeg = "RightLeg", RightFoot = "RightLeg", ["Right Leg"] = "RightLeg",
}

AIM_BIND.isMeleeName = function(name)
    name = string.lower(name or "")
    return name:find("knife", 1, true) ~= nil
        or name:find("katana", 1, true) ~= nil
        or name:find("sword", 1, true) ~= nil
        or name:find("melee", 1, true) ~= nil
        or name:find("blade", 1, true) ~= nil
end

-- True when the player is holding a non-melee tool (a gun).
AIM_BIND.hostHasGun = function(player)
    local character = player and player.Character
    if not character then
        return false
    end
    for _, child in ipairs(character:GetChildren()) do
        if child:IsA("Tool") and not AIM_BIND.isMeleeName(child.Name) then
            return true
        end
    end
    return false
end

-- The colorbot only reacts while its key is held (Hold) or toggled on (Toggle).
-- With no key bound it stays off, so it can never fire on its own.
AIM_BIND.colorbotOn = function()
    if TRIGGER_MODE == "Toggle" then
        return TRIGGER_TOGGLED
    end
    return TRIGGER_HELD
end

-- Shared helpers so the knock checks do not allocate a table + closures on every call.
UI.KO_NAMES = { "K.O", "KO", "Knocked", "Downed" }
UI.koValue = function(v) return v.Value end
UI.koAttr = function(c, n) return c:GetAttribute(n) end

-- The result is reused for ~1 frame: the two per-frame render steps (and the host/firing code)
-- all ask the same question, so answer it once instead of 4-8 times per frame.
UI._koAt, UI._koVal = 0, false
local function isLocalPlayerKnockedOrDead()
    local now = os.clock()
    if now - UI._koAt < 0.012 then
        return UI._koVal
    end
    local v = UI._koRaw()
    UI._koAt, UI._koVal = now, v
    return v
end
UI._koRaw = function()
    local character = LocalPlayer.Character
    if not character then
        return true
    end
    
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if not humanoid or humanoid.Health <= 0 then
        return true
    end

    local bodyEffects = character:FindFirstChild("BodyEffects")
    if bodyEffects then
        local names = UI.KO_NAMES
        for i = 1, #names do
            local value = bodyEffects:FindFirstChild(names[i])
            if value then
                local ok, state = pcall(UI.koValue, value)
                if ok and state == true then
                    return true
                end
            end
        end
    end

    local names = UI.KO_NAMES
    for i = 1, #names do
        local ok, state = pcall(UI.koAttr, character, names[i])
        if ok and state == true then
            return true
        end
    end

    local state = humanoid:GetState()
    return state == Enum.HumanoidStateType.Dead
        or state == Enum.HumanoidStateType.FallingDown
        or state == Enum.HumanoidStateType.Physics
        or state == Enum.HumanoidStateType.PlatformStanding
end

local lastDeathState = false

local function stopColorbotIfDead()
    if COLORBOT_ENABLED and isLocalPlayerKnockedOrDead() then
        -- Only reset once when first detecting death
        if not lastDeathState then
            TRIGGER_HELD = false
            TRIGGER_TOGGLED = false
            cancelHostTrigger()
            resetHostRound()
            AIM_ASSIST_TARGET = nil
            lastDeathState = true
        end
    else
        -- Player is alive, reset the death flag
        lastDeathState = false
    end
end

LocalPlayer.CharacterAdded:Connect(function(character)
    -- Reset death check on respawn
    task.wait(0.1)
end)

local AIM_ASSIST_RADIUS = 100
local AIM_ASSIST_STRENGTH = 0.015
local AIM_ASSIST_TARGET = nil

local resetAimAssistState


-- Does this input match the aim assist key (keyboard key or mouse button)?
AIM_BIND.matches = function(input)
    if AIM_BIND.key ~= nil and input.UserInputType == AIM_BIND.key then
        return true
    end
    return AIM_ASSIST_KEYCODE ~= Enum.KeyCode.Unknown
        and input.UserInputType == Enum.UserInputType.Keyboard
        and input.KeyCode == AIM_ASSIST_KEYCODE
end

-- Switch on + no key  = always active.  Switch on + key bound = active only per Hold / Toggle.
AIM_BIND.isActive = function()
    if not AIM_ASSIST_ENABLED then return false end
    -- Only works while YOU have a gun equipped (nothing in hand, or a knife / melee = off).
    if not AIM_BIND.hostHasGun(LocalPlayer) then return false end
    -- No key bound: the switch alone keeps aim assist always on.
    if AIM_BIND.key == nil and AIM_ASSIST_KEYCODE == Enum.KeyCode.Unknown then
        return true
    end
    -- Optional key bound: Hold / Toggle decides when it is active.
    return AIM_BIND.held == true
end

local function finishAimAssistBind(input)
    if not AIM_ASSIST_BIND_LISTENING then return false end

    if input.UserInputType == Enum.UserInputType.Keyboard
        and input.KeyCode == Enum.KeyCode.Escape then
        -- ESC clears the keybind (leaves it unbound) instead of cancelling.
        AIM_BIND.key = nil
        AIM_ASSIST_KEYCODE = Enum.KeyCode.Unknown
        AIM_ASSIST_KEY_LABEL = "NONE"
        AIM_ASSIST_BIND_LISTENING = false
        AIM_BIND.held, AIM_BIND.toggled = false, false
        if AIM_BIND.refresh then AIM_BIND.refresh() end
        return true
    end

    if input.UserInputType == Enum.UserInputType.Keyboard then
        if input.KeyCode == Enum.KeyCode.Unknown then
            return true
        end
        AIM_BIND.key = nil
        AIM_ASSIST_KEYCODE = input.KeyCode
        AIM_ASSIST_KEY_LABEL = input.KeyCode.Name
    elseif input.UserInputType ~= Enum.UserInputType.MouseMovement
        and input.UserInputType ~= Enum.UserInputType.MouseWheel then
        AIM_BIND.key = input.UserInputType
        AIM_ASSIST_KEYCODE = Enum.KeyCode.Unknown
        AIM_ASSIST_KEY_LABEL = input.UserInputType.Name
        if input.UserInputType == Enum.UserInputType.MouseButton2 then UI.rmbGuard = true end
    else
        return false
    end

    AIM_BIND.held, AIM_BIND.toggled = false, false
    AIM_ASSIST_BIND_LISTENING = false
    if AIM_BIND.refresh then AIM_BIND.refresh() end
    return true
end

-- ============================ ESP ============================

UI.esp = {
    enabled = false,
    type = "2D",
    box = false,
    filled = false,
    distance = false,
    name = false,
    health = false,
    snapline = false,
    bone = false,
    limit = false,
    limitValue = 1000,
    colors = {
        box = Color3.fromRGB(255, 255, 255),
        distance = Color3.fromRGB(200, 202, 210),
        name = Color3.fromRGB(255, 255, 255),
        snapline = Color3.fromRGB(255, 255, 255),
        bone = Color3.fromRGB(255, 255, 255),
    },
    binds = {
        master = Enum.KeyCode.Unknown,
    },
}
UI.espBindOrder = { "master" }

-- ESP target list rules (same rule the triggerbot list uses):
--   Whitelist mode: everyone EXCEPT the selected players
--   Blacklist mode: ONLY the selected players
UI.esp.listMode = "Whitelist"
UI.esp.selected = {}
UI.esp.allowed = function(player)
    local ESP = UI.esp
    if not player or player == LocalPlayer then
        return false
    end
    local isSelected = ESP.selected[player.UserId] == true
    if ESP.listMode == "Blacklist" then
        return isSelected
    end
    return not isSelected
end

-- ESP renderer (Frame based, no Drawing library required)
do
    local ESP = UI.esp
    local floor, max, min = math.floor, math.max, math.min
    local sqrt, atan2, deg = math.sqrt, math.atan2, math.deg
    local WHITE = Color3.new(1, 1, 1)
    local BLACK = Color3.new(0, 0, 0)

    local espGui = Instance.new("ScreenGui")
    espGui.Name = "ArmadaESP"
    espGui.ResetOnSpawn = false
    espGui.IgnoreGuiInset = true
    espGui.DisplayOrder = 5
    espGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    espGui.Parent = CoreGui

    local adornFolder = Instance.new("Folder")
    adornFolder.Name = "ArmadaESPAdorn"
    adornFolder.Parent = CoreGui

    local BONES_R15 = {
        { "Head", "UpperTorso" }, { "UpperTorso", "LowerTorso" },
        { "UpperTorso", "LeftUpperArm" }, { "LeftUpperArm", "LeftLowerArm" }, { "LeftLowerArm", "LeftHand" },
        { "UpperTorso", "RightUpperArm" }, { "RightUpperArm", "RightLowerArm" }, { "RightLowerArm", "RightHand" },
        { "LowerTorso", "LeftUpperLeg" }, { "LeftUpperLeg", "LeftLowerLeg" }, { "LeftLowerLeg", "LeftFoot" },
        { "LowerTorso", "RightUpperLeg" }, { "RightUpperLeg", "RightLowerLeg" }, { "RightLowerLeg", "RightFoot" },
    }
    local BONES_R6 = {
        { "Head", "Torso" },
        { "Torso", "Left Arm" }, { "Torso", "Right Arm" },
        { "Torso", "Left Leg" }, { "Torso", "Right Leg" },
    }
    local EDGES_3D = {
        { 0, 1 }, { 2, 3 }, { 4, 5 }, { 6, 7 },
        { 0, 2 }, { 1, 3 }, { 4, 6 }, { 5, 7 },
        { 0, 4 }, { 1, 5 }, { 2, 6 }, { 3, 7 },
    }

    local objects = {}
    local anyShown = false

    ----------------------------------------------------------------
    -- Element helpers
    ----------------------------------------------------------------
    local function newFrame(parent, z)
        local f = Instance.new("Frame")
        f.BorderSizePixel = 0
        f.BackgroundColor3 = WHITE
        f.ZIndex = z or 1
        f.Visible = false
        f.Parent = parent
        return f
    end

    local function newLine(parent, z)
        local f = newFrame(parent, z)
        f.AnchorPoint = Vector2.new(0.5, 0.5)
        return f
    end

    local function newText(parent, size, font)
        local t = Instance.new("TextLabel")
        t.BackgroundTransparency = 1
        t.BorderSizePixel = 0
        t.Font = font
        t.TextSize = size
        t.TextColor3 = WHITE
        t.TextStrokeTransparency = 0.35
        t.TextStrokeColor3 = BLACK
        t.Size = UDim2.fromOffset(220, size + 3)
        t.ZIndex = 5
        t.Visible = false
        t.Parent = parent
        return t
    end

    local function setVis(obj, el, on)
        if obj.vis[el] ~= on then
            obj.vis[el] = on
            el.Visible = on
        end
    end

    -- Only write a GUI property when its value actually changed (writes are the expensive part).
    local function placeRect(obj, el, x, y, w, h, color)
        local c = obj.cache[el]
        if not c then
            c = {}
            obj.cache[el] = c
        end
        w, h = max(w, 0), max(h, 0)
        if c.x ~= x or c.y ~= y then
            c.x, c.y = x, y
            el.Position = UDim2.fromOffset(x, y)
        end
        if c.w ~= w or c.h ~= h then
            c.w, c.h = w, h
            el.Size = UDim2.fromOffset(w, h)
        end
        if color and c.col ~= color then
            c.col = color
            el.BackgroundColor3 = color
        end
        setVis(obj, el, true)
    end

    local function drawLine(obj, el, ax, ay, bx, by, color, thickness)
        local dx, dy = bx - ax, by - ay
        local len = sqrt(dx * dx + dy * dy)
        if len ~= len or len > 30000 then
            setVis(obj, el, false)
            return
        end
        local px, py = floor((ax + bx) / 2 + 0.5), floor((ay + by) / 2 + 0.5)
        local sl = floor(len + 0.5)
        local rot = deg(atan2(dy, dx))
        local c = obj.cache[el]
        if not c then
            c = {}
            obj.cache[el] = c
        end
        if c.x ~= px or c.y ~= py then
            c.x, c.y = px, py
            el.Position = UDim2.fromOffset(px, py)
        end
        if c.w ~= sl or c.h ~= thickness then
            c.w, c.h = sl, thickness
            el.Size = UDim2.fromOffset(sl, thickness)
        end
        if c.r ~= rot then
            c.r = rot
            el.Rotation = rot
        end
        if c.col ~= color then
            c.col = color
            el.BackgroundColor3 = color
        end
        setVis(obj, el, true)
    end

    local function ensureList(obj, listName, count, z, isLine)
        local list = obj[listName]
        for i = #list + 1, count do
            list[i] = isLine and newLine(obj.holder, z) or newFrame(obj.holder, z)
        end
        return list
    end

    local function hideList(obj, list)
        for i = 1, #list do
            setVis(obj, list[i], false)
        end
    end

    local function hideEl(obj, el)
        if el then
            setVis(obj, el, false)
        end
    end

    ----------------------------------------------------------------
    -- Per-player objects
    ----------------------------------------------------------------
    local function createObject(player)
        local holder = Instance.new("Frame")
        holder.Name = player.Name
        holder.BackgroundTransparency = 1
        holder.BorderSizePixel = 0
        holder.Size = UDim2.fromScale(1, 1)
        holder.Visible = false
        holder.Parent = espGui
        objects[player] = {
            player = player,
            holder = holder,
            shown = false,
            vis = {},
            cache = {},
            corners = {},
            edges = {},
            bones = {},
            pts = {},
        }
    end

    local function removeObject(player)
        local obj = objects[player]
        if not obj then return end
        objects[player] = nil
        if obj.holder then
            obj.holder:Destroy()
        end
        if obj.adorn then
            obj.adorn:Destroy()
        end
    end

    local function hideObject(obj)
        if obj.shown then
            obj.shown = false
            obj.holder.Visible = false
        end
        if obj.adorn then
            setVis(obj, obj.adorn, false)
        end
    end

    local function drawBones(obj, cam, character, color)
        -- Resolve the bone parts once per character instead of ~30 FindFirstChild calls per frame.
        local refs = obj.boneRefs
        if obj.boneChar ~= character or not refs or (obj.boneRetry and os.clock() > obj.boneRetry) then
            obj.boneChar = character
            local list = character:FindFirstChild("UpperTorso") and BONES_R15 or BONES_R6
            refs = {}
            -- Every joint is shared by 2-3 bones, so keep one list of unique parts and have the
            -- bones point into it: each joint is projected once per frame instead of once per bone.
            local parts, index = {}, {}
            local function slot(part)
                local i = index[part]
                if not i then
                    i = #parts + 1
                    parts[i] = part
                    index[part] = i
                end
                return i
            end
            local missing = false
            for i, pair in ipairs(list) do
                local a = character:FindFirstChild(pair[1])
                local b = character:FindFirstChild(pair[2])
                if a and b and a:IsA("BasePart") and b:IsA("BasePart") then
                    refs[i] = { slot(a), slot(b) }
                else
                    refs[i] = false
                    missing = true
                end
            end
            obj.boneRefs = refs
            obj.boneParts = parts
            obj.boneScr = {}
            -- parts that were missing (still loading / streamed out) are looked up again after 1s
            obj.boneRetry = missing and (os.clock() + 1) or nil
        end
        local parts, scr = obj.boneParts, obj.boneScr
        for i = 1, #parts do
            local part = parts[i]
            if part.Parent then
                scr[i] = cam:WorldToViewportPoint(part.Position)
            else
                scr[i] = false
                obj.boneRefs = nil -- a bone part was removed: rebuild next frame
            end
        end
        local lines = ensureList(obj, "bones", #refs, 3, true)
        for i = 1, #refs do
            local ref = refs[i]
            local pa = ref and scr[ref[1]]
            local pb = ref and scr[ref[2]]
            if pa and pb and pa.Z > 0 and pb.Z > 0 then
                drawLine(obj, lines[i], pa.X, pa.Y, pb.X, pb.Y, color, 1)
            else
                setVis(obj, lines[i], false)
            end
        end
        for i = #refs + 1, #lines do
            setVis(obj, lines[i], false)
        end
    end

    -- Bounding box of a knocked body. Its HumanoidRootPart stops where the knock happened,
    -- so the box is built from the parts that are still moving.
    local function knockedBody(character)
        local minY, maxY = math.huge, -math.huge
        local sx, sy, sz, n = 0, 0, 0, 0
        for _, part in ipairs(character:GetChildren()) do
            if part:IsA("BasePart") then
                local p = part.Position
                local hy = part.Size.Y * 0.5
                if p.Y - hy < minY then minY = p.Y - hy end
                if p.Y + hy > maxY then maxY = p.Y + hy end
                sx, sy, sz, n = sx + p.X, sy + p.Y, sz + p.Z, n + 1
            end
        end
        if n == 0 then return nil end
        return Vector3.new(sx / n, (minY + maxY) / 2, sz / n), minY, maxY
    end

    -- Returns true if the player was drawn this frame.
    local function update(obj, cam, camPos, viewport)
        local player = obj.player
        local character = player.Character
        if not character then return false end
        -- Cache the character's key parts; only look them up again when the character changes.
        local hrp, hum = obj.hrp, obj.hum
        if obj.char ~= character or not hrp or not hum
            or hrp.Parent ~= character or hum.Parent ~= character then
            obj.char = character
            hrp = character:FindFirstChild("HumanoidRootPart")
            hum = character:FindFirstChildOfClass("Humanoid")
            obj.hrp, obj.hum = hrp, hum
            obj.head = nil
            obj.boneRefs = nil
        end
        if not hrp or not hum or hum.Health <= 0 then return false end

        -- A knocked body's HumanoidRootPart stops where the knock happened, so track the
        -- limbs that are still moving instead.
        -- The knock check is ~25 instance lookups + 8 pcalls, so it is only redone every 0.1s
        -- per player (death is still caught every frame by the Health check above).
        local nowClock = os.clock()
        local knocked
        if obj.knockChar == character and nowClock - obj.knockAt < 0.1 then
            knocked = obj.knockVal
        else
            knocked = UI.rawKnocked(character, hum)
            obj.knockChar, obj.knockAt, obj.knockVal = character, nowClock, knocked
        end
        local pos, bodyMinY, bodyMaxY = hrp.Position, nil, nil
        if knocked then
            local c, minY, maxY = knockedBody(character)
            if c then
                pos, bodyMinY, bodyMaxY = c, minY, maxY
            end
        end

        local dist = (camPos - pos).Magnitude
        if ESP.limit and dist > ESP.limitValue then return false end

        -- (the on-screen / behind-camera test is done below on the box corners, so no extra
        -- projection of the centre point is needed)
        local head = obj.head
        if not head or head.Parent ~= character then
            head = character:FindFirstChild("Head")
            obj.head = head
        end
        local topY, botY
        if bodyMinY then
            topY = bodyMaxY + 0.15
            botY = bodyMinY - 0.15
        else
            topY = pos.Y + 2.7
            if head and head:IsA("BasePart") then
                topY = head.Position.Y + head.Size.Y * 0.5 + 0.15
            end
            botY = pos.Y - 3.1
        end
        local height = topY - botY
        local midOffset = (topY + botY) / 2 - pos.Y

        local espType = ESP.type
        local colors = ESP.colors
        local boxColor = colors.box
        local minX, minY, maxX, maxY

        if espType == "3D" then
            local cf = (knocked and CFrame.new(pos) or hrp.CFrame) * CFrame.new(0, midOffset, 0)
            local hx, hy, hz = 2, height / 2, 1
            local pts = obj.pts
            local any = false
            for i = 0, 7 do
                local sx = (i % 2 == 0) and -hx or hx
                local sy = (floor(i / 2) % 2 == 0) and -hy or hy
                local sz = (floor(i / 4) % 2 == 0) and -hz or hz
                local sp = cam:WorldToViewportPoint(cf:PointToWorldSpace(Vector3.new(sx, sy, sz)))
                pts[i] = sp
                if sp.Z > 0 then
                    any = true
                    if not minX or sp.X < minX then minX = sp.X end
                    if not maxX or sp.X > maxX then maxX = sp.X end
                    if not minY or sp.Y < minY then minY = sp.Y end
                    if not maxY or sp.Y > maxY then maxY = sp.Y end
                end
            end
            if not any then return false end
            minX, minY = floor(minX), floor(minY)
            maxX, maxY = floor(maxX + 0.5), floor(maxY + 0.5)
        else
            local top = cam:WorldToViewportPoint(Vector3.new(pos.X, topY, pos.Z))
            local bot = cam:WorldToViewportPoint(Vector3.new(pos.X, botY, pos.Z))
            if top.Z <= 0 or bot.Z <= 0 then return false end
            local h = max(bot.Y - top.Y, 4)
            local w = h * 0.55
            local cx = (top.X + bot.X) / 2
            minX = floor(cx - w / 2 + 0.5)
            maxX = floor(cx + w / 2 + 0.5)
            minY = floor(top.Y + 0.5)
            maxY = floor(top.Y + h + 0.5)
        end

        if maxX < 0 or minX > viewport.X or maxY < 0 or minY > viewport.Y then
            return false
        end

        local w, h = maxX - minX, maxY - minY
        local cx = floor((minX + maxX) / 2 + 0.5)
        local showBox = ESP.box

        -- Fill (2D and Corner)
        if showBox and ESP.filled and espType ~= "3D" then
            if not obj.fill then
                obj.fill = newFrame(obj.holder, 1)
                obj.fill.BackgroundTransparency = 0.82
            end
            placeRect(obj, obj.fill, minX, minY, w, h, boxColor)
        else
            hideEl(obj, obj.fill)
        end

        -- 2D box
        if showBox and espType == "2D" then
            if not obj.box2d then
                obj.box2d = newFrame(obj.holder, 2)
                obj.box2d.BackgroundTransparency = 1
                local stroke = Instance.new("UIStroke")
                stroke.Thickness = 1
                stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
                stroke.Parent = obj.box2d
                obj.box2dStroke = stroke
            end
            if obj.box2dColor ~= boxColor then
                obj.box2dColor = boxColor
                obj.box2dStroke.Color = boxColor
            end
            placeRect(obj, obj.box2d, minX, minY, w, h)
        else
            hideEl(obj, obj.box2d)
        end

        -- Corner box
        if showBox and espType == "Corner" then
            local list = ensureList(obj, "corners", 8, 2, false)
            local c = max(4, floor(min(w, h) * 0.28 + 0.5))
            local t = 2
            placeRect(obj, list[1], minX, minY, c, t, boxColor)
            placeRect(obj, list[2], minX, minY, t, c, boxColor)
            placeRect(obj, list[3], maxX - c, minY, c, t, boxColor)
            placeRect(obj, list[4], maxX - t, minY, t, c, boxColor)
            placeRect(obj, list[5], minX, maxY - t, c, t, boxColor)
            placeRect(obj, list[6], minX, maxY - c, t, c, boxColor)
            placeRect(obj, list[7], maxX - c, maxY - t, c, t, boxColor)
            placeRect(obj, list[8], maxX - t, maxY - c, t, c, boxColor)
        else
            hideList(obj, obj.corners)
        end

        -- 3D box
        if showBox and espType == "3D" then
            local list = ensureList(obj, "edges", 12, 2, true)
            local pts = obj.pts
            for i, edge in ipairs(EDGES_3D) do
                local a, b = pts[edge[1]], pts[edge[2]]
                if a.Z > 0 and b.Z > 0 then
                    drawLine(obj, list[i], a.X, a.Y, b.X, b.Y, boxColor, 2)
                else
                    setVis(obj, list[i], false)
                end
            end
        else
            hideList(obj, obj.edges)
        end

        -- 3D fill (real world-space box)
        if showBox and ESP.filled and espType == "3D" then
            if not obj.adorn then
                local adorn = Instance.new("BoxHandleAdornment")
                adorn.Name = "ESPFill"
                adorn.AlwaysOnTop = true
                adorn.ZIndex = 1
                adorn.Transparency = 0.8
                adorn.Visible = false
                adorn.Parent = adornFolder
                obj.adorn = adorn
            end
            local adorn = obj.adorn
            if obj.adornee ~= hrp then
                obj.adornee = hrp
                adorn.Adornee = hrp
            end
            adorn.Size = Vector3.new(4, height, 2)
            adorn.CFrame = knocked
                and hrp.CFrame:ToObjectSpace(CFrame.new(pos + Vector3.new(0, midOffset, 0)))
                or CFrame.new(0, midOffset, 0)
            adorn.Color3 = boxColor
            setVis(obj, adorn, true)
        elseif obj.adorn then
            setVis(obj, obj.adorn, false)
        end

        -- Health bar
        if ESP.health then
            if not obj.healthBack then
                obj.healthBack = newFrame(obj.holder, 3)
                obj.healthBack.BackgroundColor3 = BLACK
                obj.healthBack.BackgroundTransparency = 0.35
                obj.healthFill = newFrame(obj.holder, 4)
            end
            local pct = math.clamp(hum.Health / max(hum.MaxHealth, 1), 0, 1)
            local fh = max(1, floor(h * pct + 0.5))
            local hpColor = obj.hpColor
            if obj.hpPct ~= pct or not hpColor then
                obj.hpPct = pct
                hpColor = Color3.fromHSV(pct * 0.33, 1, 1)
                obj.hpColor = hpColor
            end
            placeRect(obj, obj.healthBack, minX - 6, minY - 1, 4, h + 2)
            placeRect(obj, obj.healthFill, minX - 5, maxY - fh, 2, fh, hpColor)
        else
            hideEl(obj, obj.healthBack)
            hideEl(obj, obj.healthFill)
        end

        -- Name
        if ESP.name then
            if not obj.nameLabel then
                obj.nameLabel = newText(obj.holder, 12, Enum.Font.GothamBold)
            end
            local label = obj.nameLabel
            local text = player.DisplayName
            if obj.nameText ~= text then
                obj.nameText = text
                label.Text = text
            end
            local lc = obj.cache[label]
            if not lc then
                lc = {}
                obj.cache[label] = lc
            end
            if lc.col ~= colors.name then
                lc.col = colors.name
                label.TextColor3 = colors.name
            end
            local ly = minY - 18
            if lc.x ~= cx or lc.y ~= ly then
                lc.x, lc.y = cx, ly
                label.Position = UDim2.fromOffset(cx - 110, ly)
            end
            setVis(obj, label, true)
        else
            hideEl(obj, obj.nameLabel)
        end

        -- Distance
        if ESP.distance then
            if not obj.distLabel then
                obj.distLabel = newText(obj.holder, 11, Enum.Font.GothamMedium)
            end
            local label = obj.distLabel
            local rounded = floor(dist + 0.5)
            if obj.distVal ~= rounded then
                obj.distVal = rounded
                label.Text = rounded .. " studs"
            end
            local lc = obj.cache[label]
            if not lc then
                lc = {}
                obj.cache[label] = lc
            end
            if lc.col ~= colors.distance then
                lc.col = colors.distance
                label.TextColor3 = colors.distance
            end
            local ly = maxY + 2
            if lc.x ~= cx or lc.y ~= ly then
                lc.x, lc.y = cx, ly
                label.Position = UDim2.fromOffset(cx - 110, ly)
            end
            setVis(obj, label, true)
        else
            hideEl(obj, obj.distLabel)
        end

        -- Snapline
        if ESP.snapline then
            if not obj.snap then
                obj.snap = newLine(obj.holder, 2)
            end
            drawLine(obj, obj.snap, viewport.X / 2, viewport.Y, cx, maxY, colors.snapline, 1)
        else
            hideEl(obj, obj.snap)
        end

        -- Skeleton
        if ESP.bone then
            drawBones(obj, cam, character, colors.bone)
        else
            hideList(obj, obj.bones)
        end

        return true
    end

    ----------------------------------------------------------------
    -- Lifecycle
    ----------------------------------------------------------------
    local renderConnection, addedConnection, removingConnection

    local function cleanup()
        if renderConnection then
            renderConnection:Disconnect()
            renderConnection = nil
        end
        if addedConnection then
            addedConnection:Disconnect()
            addedConnection = nil
        end
        if removingConnection then
            removingConnection:Disconnect()
            removingConnection = nil
        end
        for player in pairs(objects) do
            removeObject(player)
        end
        pcall(function() espGui:Destroy() end)
        pcall(function() adornFolder:Destroy() end)
    end

    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            createObject(player)
        end
    end
    addedConnection = Players.PlayerAdded:Connect(function(player)
        if player ~= LocalPlayer and not objects[player] then
            createObject(player)
        end
    end)
    removingConnection = Players.PlayerRemoving:Connect(removeObject)

    renderConnection = RunService.RenderStepped:Connect(function()
        if SCRIPT_KILLED then
            cleanup()
            return
        end

        local cam = workspace.CurrentCamera
        if not cam or not ESP.enabled then
            if anyShown then
                anyShown = false
                for _, obj in pairs(objects) do
                    hideObject(obj)
                end
            end
            return
        end

        anyShown = true
        local viewport = cam.ViewportSize
        local camPos = cam.CFrame.Position
        for _, obj in pairs(objects) do
            if not ESP.allowed(obj.player) then
                hideObject(obj)
            else
                local ok, drawn = pcall(update, obj, cam, camPos, viewport)
                if ok and drawn then
                    if not obj.shown then
                        obj.shown = true
                        obj.holder.Visible = true
                    end
                else
                    hideObject(obj)
                end
            end
        end
    end)
end


local selectedActivator = nil
local activatorPulseUntil = 0
local activatorFireAt = 0
local hostDelayToken = 0
local activatorConnections = {}
local activatorAnimationConnections = {}
local hostBurstActive = false
local hostBurstTarget = nil

updateLiveStatus = function()
    local live = UI.live
    if not live or not live.trigger then return end
    local T = UI.Theme

    live.trigger.Text = AIM_BIND.colorbotOn() and "ACTIVE" or "IDLE"
    live.trigger.TextColor3 = AIM_BIND.colorbotOn() and T.good or T.text

    if selectedActivator then
        live.host.Text = selectedActivator.DisplayName
        live.host.TextColor3 = T.text
    else
        live.host.Text = "None selected"
        live.host.TextColor3 = T.text
    end

    live.bind.Text = COLORBOT_ENABLED and tostring(TRIGGER_KEY_LABEL) or "OFF"
    live.mode.Text = COLORBOT_ENABLED and string.upper(tostring(TRIGGER_MODE)) or "OFF"
    live.delay.Text = COLORBOT_ENABLED
        and (tostring(HOST_DELAY_MIN_MS) .. " - " .. tostring(HOST_DELAY_MAX_MS) .. " ms")
        or "OFF"

    local pageStatus = UI.triggerbotPageStatus
    if pageStatus then
        local tbActive = TRIGGERBOT_ENABLED and UI.tb.canFire and UI.tb.canFire() or false
        pageStatus.enabled.Text = tbActive and "ACTIVE" or "IDLE"
        pageStatus.enabled.TextColor3 = tbActive and T.good or T.text
        pageStatus.host.Text = TRIGGERBOT_ENABLED and string.upper(TB_LIST_MODE) or "OFF"
        pageStatus.bind.Text = TRIGGERBOT_ENABLED and tostring(TB_KEY_LABEL) or "OFF"
        pageStatus.mode.Text = TRIGGERBOT_ENABLED and string.upper(tostring(TB_MODE)) or "OFF"
        pageStatus.delay.Text = TRIGGERBOT_ENABLED and (tostring(math.floor(UI.tb.delayMs() + 0.5)) .. " ms") or "OFF"
    end
end

local fireTriggeredShot
local startHostBurst

resetHostRound = function()
    HOST_ROUND_DELAY_MS = nil
    HOST_ROUND_TARGET = nil
end

cancelHostTrigger = function()
    hostDelayToken = hostDelayToken + 1
    activatorPulseUntil = 0
    activatorFireAt = 0
    HOST_TRIGGER_ARMED = false
    hostBurstActive = false
    hostBurstTarget = nil
    AIM_ASSIST_TARGET = nil
end

local function disconnectActivator()
    for _, c in ipairs(activatorConnections) do
        pcall(function() c:Disconnect() end)
    end
    for _, c in ipairs(activatorAnimationConnections) do
        pcall(function() c:Disconnect() end)
    end
    activatorConnections = {}
    activatorAnimationConnections = {}
end

local function isLikelyTracerObject(instance)
    if not instance then
        return false
    end

    if instance:IsA("Beam") then
        return true
    end

    if instance:IsA("BasePart") then
        local n = string.lower(instance.Name or "")

        if n:find("tracer", 1, true)
            or n:find("bullet", 1, true)
            or n:find("projectile", 1, true)
            or n:find("bulletline", 1, true)
            or n:find("shotline", 1, true) then
            return true
        end

        local size = instance.Size
        local smallest = math.min(size.X, size.Y, size.Z)
        local largest = math.max(size.X, size.Y, size.Z)

        return instance.CanCollide == false
            and instance.Transparency < 1
            and smallest <= 0.20
            and largest >= 3.0
    end

    return false
end

local function getEffectPosition(instance)
    if not instance then
        return nil
    end

    if instance:IsA("Beam") then
        local a0 = instance.Attachment0
        local a1 = instance.Attachment1
        if a0 then return a0.WorldPosition end
        if a1 then return a1.WorldPosition end
    elseif instance:IsA("Attachment") then
        return instance.WorldPosition
    elseif instance:IsA("BasePart") then
        return instance.Position
    end

    local parent = instance.Parent
    if parent and parent:IsA("BasePart") then
        return parent.Position
    end

    return nil
end

local function getNearestPlayerToPosition(position, maxDistance)
    if not position then return nil, math.huge end

    local nearest = nil
    local nearestDistance = maxDistance or math.huge

    for _, candidate in ipairs(Players:GetPlayers()) do
        if candidate ~= LocalPlayer and candidate.Character then
            local root = candidate.Character:FindFirstChild("HumanoidRootPart")
                or candidate.Character:FindFirstChild("UpperTorso")
                or candidate.Character:FindFirstChild("Torso")

            if root and root:IsA("BasePart") then
                local distance = (root.Position - position).Magnitude
                if distance < nearestDistance then
                    nearest = candidate
                    nearestDistance = distance
                end
            end
        end
    end

    return nearest, nearestDistance
end

local function isOwnedByActivator(player, instance)
    local character = player and player.Character
    if not character or not instance then
        return false
    end

    if instance:IsDescendantOf(character) then
        return true
    end

    if instance:IsA("Beam") then
        local a0 = instance.Attachment0
        local a1 = instance.Attachment1

        if (a0 and a0:IsDescendantOf(character))
            or (a1 and a1:IsDescendantOf(character)) then
            return true
        end
    end

    -- Everything below guesses ownership from position, so it is only allowed while the host
    -- is actually holding a gun (an unarmed / knife-only host cannot be shooting).
    if not AIM_BIND.hostHasGun(player) then
        return false
    end

    local position = getEffectPosition(instance)
    if not position then
        return false
    end

    -- The host must be the CLOSEST player to the effect (another player's tracer near the
    -- host does not count).
    if getNearestPlayerToPosition(position, 8) == player then
        return true
    end

    -- A tracer that is one long thin part is positioned at its CENTER, so also test both ends.
    if instance:IsA("BasePart") then
        local size = instance.Size
        local cf = instance.CFrame
        local half
        if size.X >= size.Y and size.X >= size.Z then
            half = cf.RightVector * (size.X * 0.5)
        elseif size.Y >= size.Z then
            half = cf.UpVector * (size.Y * 0.5)
        else
            half = cf.LookVector * (size.Z * 0.5)
        end
        if getNearestPlayerToPosition(position + half, 8) == player
            or getNearestPlayerToPosition(position - half, 8) == player then
            return true
        end
    end

    return false
end

local function hostHasMeleeWeapon(player)
    local character = player and player.Character
    if not character then return false end

    for _, child in ipairs(character:GetChildren()) do
        if child:IsA("Tool") then
            local n = string.lower(child.Name or "")
            if n:find("knife", 1, true)
                or n:find("melee", 1, true)
                or n:find("sword", 1, true)
                or n:find("katana", 1, true)
                or n:find("blade", 1, true) then
                return true
            end
        end
    end

    return false
end

local function pulseActivator(player, reason)

    if SCRIPT_KILLED or not MENU_UNLOCKED or not COLORBOT_ENABLED or player ~= selectedActivator or not AIM_BIND.colorbotOn() then
        return
    end

    -- Check if user is dead/knocked
    if isLocalPlayerKnockedOrDead() then
        stopColorbotIfDead()
        return
    end

    if hostHasMeleeWeapon(player) or AIM_BIND.ownMelee() then
        cancelHostTrigger()
        return
    end

    local nowClock = os.clock()
    if nowClock < AIM_BIND.hostIgnoreUntil then
        return
    end
    -- Already firing (burst running or mouse held): nothing to react to.
    if hostBurstActive
        or UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton1) then
        return
    end

    -- Your OWN tracers show up near the host about one round-trip after you click. Ignore
    -- detections for that short, ping-sized window only (roughly 0.08 - 0.25 s), and only
    -- right after you have clicked. It never delays a reaction when you have not just fired.
    local okPing, ping = pcall(function() return LocalPlayer:GetNetworkPing() end)
    local ownWindow = math.clamp((okPing and ping or 0.05) * 2 + 0.05, 0.08, 0.25)
    if (nowClock - AIM_BIND.lastOwnClick) < ownWindow then
        return
    end

    if not HOST_TRIGGER_ARMED then

        if HOST_ROUND_DELAY_MS == nil then
            chooseHostDelay()
            HOST_ROUND_DELAY_MS = HOST_DELAY_MS
        else
            HOST_DELAY_MS = HOST_ROUND_DELAY_MS
        end

        local detectedAt = os.clock()
        local delaySeconds = HOST_DELAY_MS / 1000
        local token = hostDelayToken + 1

        hostDelayToken = token
        activatorFireAt = detectedAt + delaySeconds
        activatorPulseUntil = detectedAt
        HOST_TRIGGER_ARMED = true

        if delaySeconds <= 0 then
            startHostBurst(token)
        else

            task.delay(delaySeconds, function()
                if SCRIPT_KILLED or not MENU_UNLOCKED then return end
                if token ~= hostDelayToken then return end
                if not HOST_TRIGGER_ARMED or not AIM_BIND.colorbotOn() then
                    cancelHostTrigger()
                    return
                end
                startHostBurst(token)
            end)
        end
    end

    target1Status.Text = "HOST SHOT — TRIGGER ARMED"
end

local function watchActivator(player)
    disconnectActivator()
    cancelHostTrigger()
    AIM_BIND.hostIgnoreUntil = os.clock() + 0.5

    if not player then
        target1Status.Text = "Select one HOST. Hold the key before the HOST shoots."
        return
    end

    target1Status.Text = "Watching " .. player.Name .. "..."

    local function watchTracer(instance)
        if not instance or not isLikelyTracerObject(instance) then
            return
        end

        if hostHasMeleeWeapon(player) then
            return
        end

        if not isOwnedByActivator(player, instance) then
            return
        end

        pulseActivator(player, "host-tracer")
    end

    local function watchExistingEffect(instance, skipImmediate)
        if not instance then return end

        if instance:IsA("Beam") then
            local connection = instance:GetPropertyChangedSignal("Enabled"):Connect(function()
                if instance.Enabled and isOwnedByActivator(player, instance) then
                    pulseActivator(player, "host-tracer-enabled")
                end
            end)
            table.insert(activatorConnections, connection)

            if not skipImmediate and instance.Enabled and isOwnedByActivator(player, instance) then
                pulseActivator(player, "host-existing-tracer")
            end
        end
    end

    local function watchCharacter(character)
        if not character then return end

        table.insert(activatorConnections,
            character.DescendantAdded:Connect(function(descendant)
                watchExistingEffect(descendant)
                watchTracer(descendant)
            end)
        )

        task.defer(function()
            if player ~= selectedActivator or not character.Parent then
                return
            end

            for _, descendant in ipairs(character:GetDescendants()) do
                if descendant:IsA("Beam") then
                    watchExistingEffect(descendant, true)
                end
            end
        end)
    end

    table.insert(activatorConnections,
        workspace.DescendantAdded:Connect(function(descendant)
            if player ~= selectedActivator then return end
            if not isLikelyTracerObject(descendant) then return end
            if not isOwnedByActivator(player, descendant) then
                -- Some games parent the tracer first and move it into place right after.
                if descendant:IsA("BasePart") then
                    task.defer(function()
                        if descendant.Parent and player == selectedActivator then
                            watchTracer(descendant)
                        end
                    end)
                end
                return
            end

            watchTracer(descendant)

            if descendant:IsA("Beam") then
                watchExistingEffect(descendant)
            end
        end)
    )

    watchCharacter(player.Character)

    table.insert(activatorConnections,
        player.CharacterAdded:Connect(function(character)
            if player == selectedActivator then
                watchActivator(player)
            end
        end)
    )
end

local function selectActivator(player)
    if player == LocalPlayer then
        player = nil
    end

    selectedActivator = player
    cancelHostTrigger()
    resetHostRound()
    target1Status.Text = player
        and ("HOST: " .. player.Name .. " — " .. (TRIGGER_MODE == "Toggle" and "toggle " or "hold ") .. TRIGGER_KEY_LABEL .. " before the shot.")
        or "Select one HOST. " .. (TRIGGER_MODE == "Toggle" and "Toggle " or "Hold ") .. TRIGGER_KEY_LABEL .. " before the HOST shoots."

    if player and COLORBOT_ENABLED then
        watchActivator(player)
    else
        disconnectActivator()
    end
end

triggerbotToggleHandler = function(enabled)
    if not enabled then
        TRIGGER_HELD = false
        TRIGGER_TOGGLED = false
        TRIGGER_BIND_LISTENING = false
        cancelHostTrigger()
        resetHostRound()
        disconnectActivator()

        if target1Status and COLORBOT_ENABLED then
            target1Status.Text = "Triggerbot is OFF. Turn it back on to arm a HOST."
        end
    else
        if selectedActivator and COLORBOT_ENABLED then
            watchActivator(selectedActivator)
        end
        if target1Status and COLORBOT_ENABLED then
            target1Status.Text = "HOST: " ..
                (selectedActivator and selectedActivator.Name or "None selected") ..
                " — " .. (TRIGGER_MODE == "Toggle" and "toggle " or "hold ") ..
                TRIGGER_KEY_LABEL .. " before the HOST shoots."
        end
    end

    if updateLiveStatus then
        updateLiveStatus()
    end
    if UI._refreshTriggerbotMasterUI then
        UI._refreshTriggerbotMasterUI()
    end
end

colorbotToggleHandler = function(enabled)
    if not enabled then
        TRIGGER_HELD = false
        TRIGGER_TOGGLED = false
        TRIGGER_BIND_LISTENING = false
        cancelHostTrigger()
        resetHostRound()

        disconnectActivator() -- the chosen host stays selected while the colorbot is off

        if target1Status then
            target1Status.Text = selectedActivator
                and ("HOST: " .. selectedActivator.Name .. " - turn the Colorbot on.")
                or "Select a host below, then turn the Colorbot on."
        end
    else
        if selectedActivator then
            watchActivator(selectedActivator)
        end
        if target1Status then
            target1Status.Text = selectedActivator
                and ("HOST: " .. selectedActivator.Name .. " — " ..
                    (TRIGGER_MODE == "Toggle" and "toggle " or "hold ") ..
                    TRIGGER_KEY_LABEL .. " before the shot.")
                or ("Select one HOST. " ..
                    (TRIGGER_MODE == "Toggle" and "Toggle " or "Hold ") ..
                    TRIGGER_KEY_LABEL .. " before the HOST shoots.")
        end
    end

    if updateLiveStatus then
        updateLiveStatus()
    end
    if UI._refreshColorbotMasterUI then
        UI._refreshColorbotMasterUI()
    end
    task.defer(rebuildHostList)
end

local CONFIG_SCHEMA_VERSION = 5

local CONFIG_STATE = type(getgenv) == "function" and getgenv() or _G
CONFIG_STATE.ArmadaConfigs = CONFIG_STATE.ArmadaConfigs or {}
CONFIG_STATE.ArmadaConfigs[LocalPlayer.UserId] = CONFIG_STATE.ArmadaConfigs[LocalPlayer.UserId] or {}
local savedConfigs = CONFIG_STATE.ArmadaConfigs[LocalPlayer.UserId]
local selectedConfigName = nil

-- Keybind helpers for config save/load. Mouse/keyboard binds are stored as (kind, name):
-- kind = "UserInputType" (mouse button), "KeyCode" (keyboard) or "None".
local function serializeBind(userInputType, keyCode)
    if userInputType then
        return "UserInputType", userInputType.Name
    end
    if keyCode == nil or keyCode == Enum.KeyCode.Unknown then
        return "None", "NONE"
    end
    return "KeyCode", keyCode.Name
end

local function readBindKind(value, fallback)
    if value == "UserInputType" or value == "KeyCode" or value == "None" then
        return value
    end
    return fallback
end

local function readBindName(value, fallback)
    if type(value) == "string" and value ~= "" then
        return value
    end
    return fallback
end

-- Keyboard-only binds (movement, UI keys) are stored as a key name, "NONE" when unbound.
local function keyCodeFromName(name)
    if type(name) ~= "string" or name == "NONE" or name == "" then
        return Enum.KeyCode.Unknown
    end
    local ok, value = pcall(function()
        return Enum.KeyCode[name]
    end)
    if ok and value then
        return value
    end
    return Enum.KeyCode.Unknown
end

local function keyNameOf(code)
    if code == nil or code == Enum.KeyCode.Unknown then
        return "NONE"
    end
    return code.Name
end

COLORBOT_ENABLED = false
TRIGGERBOT_ENABLED = false
AIM_ASSIST_ENABLED = false

local function getDefaultConfigSettings()
    return {
        colorbotEnabled = false,
        triggerbotEnabled = false,
        triggerMode = "Hold",
        triggerKind = "None",
        triggerName = "NONE",

        hostDelayMin = 0,
        hostDelayMax = 0,
        aimAssistEnabled = false,
        aimAssistRadius = 100,
        aimAssistStrength = 0.015,
        tbFov = 0,
        tbStrength = 0,
        distDouble = 60,
        distTac = 60,
        distRev = 265,
        knockColorbot = false,
        knockTriggerbot = false,

        tbKeyKind = "None", tbKeyName = "NONE",
        tbMode = "Hold", tbListMode = "Whitelist",
        aimKeyKind = "None", aimKeyName = "NONE",
        aimMode = "Hold",
        aimParts = { Head = true, Torso = true, LeftArm = true, RightArm = true, LeftLeg = true, RightLeg = true },
        espListMode = "Whitelist",
        hideKeyName = "F4", killKeyName = "F7",
        speedOn = false, speed = 10, speedKeyName = "NONE",
        flyOn = false, flySpeed = 10, flyKeyName = "NONE",
    }
end

local function makeConfigRecord(name, settings)
    return {
        version = CONFIG_SCHEMA_VERSION,
        sourceUserId = LocalPlayer.UserId,
        name = name,
        settings = settings,
    }
end

local configStatusLabel
local configStatusShare
local configNameInput
local configList
local configPasteInput
local configSelectedLabel

local configSaveButton
local configDeleteButton
local configImportButton
local configExportButton
local configLoadButton
local function setConfigStatus(message)
    if configStatusLabel and configStatusLabel.Parent then
        configStatusLabel.Text = tostring(message or "")
    end
    if configStatusShare and configStatusShare.Parent then
        configStatusShare.Text = tostring(message or "")
    end
end

local function normalizeConfigName(value)
    value = tostring(value or "")
    value = value:gsub("[%c]", "")
    value = value:gsub("[\\/]", "-")
    value = value:gsub("^%s+", ""):gsub("%s+$", "")
    if #value > 32 then
        value = value:sub(1, 32)
    end
    return value
end

local function serializeTriggerBinding()
    if TRIGGER_KEY then
        return "UserInputType", TRIGGER_KEY.Name
    end
    if TRIGGER_KEYCODE == Enum.KeyCode.Unknown then
        return "None", "NONE"
    end
    return "KeyCode", TRIGGER_KEYCODE.Name
end

local function collectCurrentConfigSettings()
    local triggerKind, triggerName = serializeTriggerBinding()
    local tbKeyKind, tbKeyName = serializeBind(TB_KEY, TB_KEYCODE)
    local aimKeyKind, aimKeyName = serializeBind(AIM_BIND.key, AIM_ASSIST_KEYCODE)
    local aimParts = {}
    for partName, on in pairs(AIM_BIND.parts) do
        aimParts[partName] = on == true
    end
    return {
        colorbotEnabled = COLORBOT_ENABLED,
        triggerbotEnabled = TRIGGERBOT_ENABLED,
        triggerMode = TRIGGER_MODE,
        triggerKind = triggerKind,
        triggerName = triggerName,

        hostDelayMin = HOST_DELAY_MIN_MS,
        hostDelayMax = HOST_DELAY_MAX_MS,
        aimAssistEnabled = AIM_ASSIST_ENABLED,
        aimAssistRadius = AIM_ASSIST_RADIUS,
        aimAssistStrength = AIM_ASSIST_STRENGTH,
        tbFov = UI.tb.fov,
        tbStrength = UI.tb.strength,
        distDouble = UI.distance.Double,
        distTac = UI.distance.Tac,
        distRev = UI.distance.Rev,
        knockColorbot = UI.knock.colorbot,
        knockTriggerbot = UI.knock.triggerbot,
        esp = UI._espCollect and UI._espCollect() or nil,

        tbKeyKind = tbKeyKind, tbKeyName = tbKeyName,
        tbMode = TB_MODE,
        tbListMode = TB_LIST_MODE,
        aimKeyKind = aimKeyKind, aimKeyName = aimKeyName,
        aimMode = AIM_BIND.mode,
        aimParts = aimParts,
        espListMode = UI.esp.listMode,
        hideKeyName = keyNameOf(UI_HIDE_KEYCODE),
        killKeyName = keyNameOf(UI_KILL_KEYCODE),
        speedOn = UI.move.speedOn,
        speed = UI.move.speed,
        speedKeyName = keyNameOf(UI.move.speedKey),
        flyOn = UI.move.flyOn,
        flySpeed = UI.move.flySpeed,
        flyKeyName = keyNameOf(UI.move.flyKey),
    }
end

local function validateAndNormalizeConfigRecord(record, fallbackName)
    if type(record) ~= "table" or type(record.settings) ~= "table" then
        return nil, "Invalid config format."
    end

    local src = record.settings
    local result = getDefaultConfigSettings()

    if type(src.colorbotEnabled) == "boolean" then
        result.colorbotEnabled = src.colorbotEnabled
    end
    if type(src.triggerbotEnabled) == "boolean" then
        result.triggerbotEnabled = src.triggerbotEnabled
    end

    if src.triggerMode == "Hold" or src.triggerMode == "Toggle" then
        result.triggerMode = src.triggerMode
    end

    if src.triggerKind == "UserInputType" or src.triggerKind == "KeyCode" or src.triggerKind == "None" then
        result.triggerKind = src.triggerKind
    end
    if type(src.triggerName) == "string" and src.triggerName ~= "" then
        result.triggerName = src.triggerName
    end

    if type(src.hostDelayMin) == "number" and src.hostDelayMin == src.hostDelayMin
        and src.hostDelayMin >= 0 and src.hostDelayMin < math.huge then
        result.hostDelayMin = math.clamp(src.hostDelayMin, 0, 1000)
    end
    if type(src.hostDelayMax) == "number" and src.hostDelayMax == src.hostDelayMax
        and src.hostDelayMax >= 0 and src.hostDelayMax < math.huge then
        result.hostDelayMax = math.clamp(src.hostDelayMax, 0, 1000)
    end
    if result.hostDelayMin > result.hostDelayMax then
        result.hostDelayMin, result.hostDelayMax = result.hostDelayMax, result.hostDelayMin
    end

    if type(src.aimAssistEnabled) == "boolean" then
        result.aimAssistEnabled = src.aimAssistEnabled
    end
    if type(src.aimAssistRadius) == "number" and src.aimAssistRadius == src.aimAssistRadius then
        result.aimAssistRadius = math.floor(math.clamp(src.aimAssistRadius, 5, 500))
    end
    if type(src.aimAssistStrength) == "number" and src.aimAssistStrength == src.aimAssistStrength then
        result.aimAssistStrength = math.clamp(src.aimAssistStrength, 0, 0.1)
    end
    if type(src.tbFov) == "number" and src.tbFov == src.tbFov then
        result.tbFov = math.floor(math.clamp(src.tbFov, 0, 500))
    end
    if type(src.tbStrength) == "number" and src.tbStrength == src.tbStrength then
        result.tbStrength = math.floor(math.clamp(src.tbStrength, 0, 100))
    end
    if type(src.knockColorbot) == "boolean" then result.knockColorbot = src.knockColorbot end
    if type(src.knockTriggerbot) == "boolean" then result.knockTriggerbot = src.knockTriggerbot end
    for field, default in pairs({ distDouble = 60, distTac = 60, distRev = 265 }) do
        if type(src[field]) == "number" and src[field] == src[field] then
            result[field] = math.floor(math.clamp(src[field], 0, 5000))
        end
    end
    if type(src.esp) == "table" and UI._espSanitize then
        result.esp = UI._espSanitize(src.esp)
    end

    -- Keybinds / modes / body parts / movement. Old config files don't have these, so
    -- hasBinds tells the apply step to leave the current binds alone for them.
    result.hasBinds = type(src.tbKeyKind) == "string" or type(src.aimKeyKind) == "string"
    result.tbKeyKind = readBindKind(src.tbKeyKind, result.tbKeyKind)
    result.tbKeyName = readBindName(src.tbKeyName, result.tbKeyName)
    result.aimKeyKind = readBindKind(src.aimKeyKind, result.aimKeyKind)
    result.aimKeyName = readBindName(src.aimKeyName, result.aimKeyName)
    result.hideKeyName = readBindName(src.hideKeyName, result.hideKeyName)
    result.killKeyName = readBindName(src.killKeyName, result.killKeyName)
    result.speedKeyName = readBindName(src.speedKeyName, result.speedKeyName)
    result.flyKeyName = readBindName(src.flyKeyName, result.flyKeyName)

    if src.tbMode == "Hold" or src.tbMode == "Toggle" then result.tbMode = src.tbMode end
    if src.aimMode == "Hold" or src.aimMode == "Toggle" then result.aimMode = src.aimMode end
    if src.tbListMode == "Whitelist" or src.tbListMode == "Blacklist" then result.tbListMode = src.tbListMode end
    if src.espListMode == "Whitelist" or src.espListMode == "Blacklist" then result.espListMode = src.espListMode end

    if type(src.aimParts) == "table" then
        local parts, anyOn = {}, false
        for partName, default in pairs(result.aimParts) do
            local v = src.aimParts[partName]
            parts[partName] = type(v) == "boolean" and v or default
            anyOn = anyOn or parts[partName]
        end
        if anyOn then result.aimParts = parts end -- keep at least one body part
    end

    if type(src.speedOn) == "boolean" then result.speedOn = src.speedOn end
    if type(src.flyOn) == "boolean" then result.flyOn = src.flyOn end
    if type(src.speed) == "number" and src.speed == src.speed then
        result.speed = math.floor(math.clamp(src.speed, 1, 1000) + 0.5)
    end
    if type(src.flySpeed) == "number" and src.flySpeed == src.flySpeed then
        result.flySpeed = math.floor(math.clamp(src.flySpeed, 1, 1000) + 0.5)
    end

    local name = normalizeConfigName(record.name or fallbackName or "")
    if name == "" then
        return nil, "Config name is required."
    end

    return {
        version = CONFIG_SCHEMA_VERSION,
        sourceUserId = tonumber(record.sourceUserId) or LocalPlayer.UserId,
        name = name,
        settings = result,
    }
end

local CONFIG_FILE_NAME = "armada_configs_" .. tostring(LocalPlayer.UserId) .. ".json"

local function persistConfigs()
    if type(writefile) ~= "function" then
        return false, "File persistence is unavailable in this environment."
    end

    local payload = {}
    for name, record in pairs(savedConfigs) do
        if type(record) == "table" then
            payload[tostring(name)] = record
        end
    end

    local encodedOk, encoded = pcall(function()
        return HttpService:JSONEncode(payload)
    end)
    if not encodedOk then
        return false, "Could not encode configs."
    end

    local writeOk, writeErr = pcall(function()
        writefile(CONFIG_FILE_NAME, encoded)
    end)
    if not writeOk then
        return false, tostring(writeErr or "Could not save config file.")
    end

    return true
end

local function loadPersistedConfigs()
    if type(isfile) ~= "function" or type(readfile) ~= "function" then
        return false, "File persistence is unavailable in this environment."
    end

    local existsOk, exists = pcall(function()
        return isfile(CONFIG_FILE_NAME)
    end)
    if not existsOk then
        return false, "Could not check the saved config file."
    end

    if not exists then

        if next(savedConfigs) ~= nil then
            return persistConfigs()
        end
        return true
    end

    local readOk, raw = pcall(function()
        return readfile(CONFIG_FILE_NAME)
    end)
    if not readOk or type(raw) ~= "string" or raw == "" then
        return false, "Could not read the saved config file."
    end

    local decodeOk, decoded = pcall(function()
        return HttpService:JSONDecode(raw)
    end)
    if not decodeOk or type(decoded) ~= "table" then
        return false, "Saved config file is invalid."
    end

    local source = decoded
    local userIdKey = tostring(LocalPlayer.UserId)

    if type(decoded.configs) == "table" then
        source = decoded.configs
    elseif type(decoded.savedConfigs) == "table" then
        source = decoded.savedConfigs
    elseif type(decoded[userIdKey]) == "table" then
        local userBucket = decoded[userIdKey]
        source = type(userBucket.configs) == "table" and userBucket.configs or userBucket
    end

    local loaded = {}

    for key, value in pairs(source) do
        if type(value) == "table" and type(value.settings) == "table" then
            local normalized = validateAndNormalizeConfigRecord(value, key)
            if normalized then
                local keyName = normalizeConfigName(tostring(key))
                normalized.name = keyName ~= "" and keyName or normalized.name
                loaded[normalized.name] = normalized
            end
        end
    end

    if next(loaded) == nil and #source > 0 then
        for _, value in ipairs(source) do
            if type(value) == "table" and type(value.settings) == "table" then
                local normalized = validateAndNormalizeConfigRecord(value, value.name)
                if normalized then
                    loaded[normalized.name] = normalized
                end
            end
        end
    end

    if next(loaded) == nil and next(source) ~= nil then
        return false, "No valid saved configs were found in the UserId config file."
    end

    table.clear(savedConfigs)
    for name, record in pairs(loaded) do
        savedConfigs[name] = record
    end

    return true
end

local function resolveKeyBinding(kind, name)
    if kind == "None" then
        return nil, Enum.KeyCode.Unknown, "NONE"
    end
    if kind == "UserInputType" then
        local ok, value = pcall(function()
            return Enum.UserInputType[name]
        end)
        if ok and value and value ~= Enum.UserInputType.None then
            return value, Enum.KeyCode.Unknown, name
        end
    else
        local ok, value = pcall(function()
            return Enum.KeyCode[name]
        end)
        if ok and value and value ~= Enum.KeyCode.Unknown then
            return nil, value, name
        end
    end

    return nil, nil, nil
end

local function applyConfigRecord(record)
    local normalized, err = validateAndNormalizeConfigRecord(record, record and record.name)
    if not normalized then
        return false, err
    end

    local settings = normalized.settings

    local triggerKey, triggerKeyCode, triggerLabel = resolveKeyBinding(settings.triggerKind, settings.triggerName)
    if not triggerLabel then
        return false, "The saved keybind is not available."
    end

    COLORBOT_ENABLED = settings.colorbotEnabled == true
    TRIGGERBOT_ENABLED = settings.triggerbotEnabled == true
    if COLORBOT_ENABLED and TRIGGERBOT_ENABLED then
        TRIGGERBOT_ENABLED = false -- the two can't run together
    end
        TRIGGER_MODE = settings.triggerMode
    TRIGGER_KEY = triggerKey
    TRIGGER_KEYCODE = triggerKeyCode
    TRIGGER_KEY_LABEL = triggerLabel
    TRIGGER_HELD = false
    TRIGGER_TOGGLED = false
    TRIGGER_BIND_LISTENING = false

    HOST_DELAY_MIN_MS = math.floor(math.clamp(settings.hostDelayMin, 0, 1000) + 0.5)
    HOST_DELAY_MAX_MS = math.floor(math.clamp(settings.hostDelayMax, 0, 1000) + 0.5)
    if HOST_DELAY_MIN_MS > HOST_DELAY_MAX_MS then
        HOST_DELAY_MIN_MS, HOST_DELAY_MAX_MS = HOST_DELAY_MAX_MS, HOST_DELAY_MIN_MS
    end
    HOST_DELAY_MS = HOST_DELAY_MIN_MS
    resetHostRound()

    AIM_ASSIST_ENABLED = settings.aimAssistEnabled
    AIM_BIND.held, AIM_BIND.toggled = false, false
    AIM_ASSIST_RADIUS = math.floor(math.clamp(settings.aimAssistRadius, 5, 500))
    AIM_ASSIST_STRENGTH = math.clamp(settings.aimAssistStrength, 0, 0.1)
    AIM_ASSIST_TARGET = nil

    UI.tb.fov = math.floor(math.clamp(settings.tbFov or 0, 0, 500))
    UI.tb.strength = math.floor(math.clamp(settings.tbStrength or 0, 0, 100))
    UI.tb.pendingChar = nil
    if UI._tbSettingsRefresh then UI._tbSettingsRefresh() end
    UI.distance.Double = math.floor(math.clamp(settings.distDouble or 60, 0, 5000))
    UI.distance.Tac = math.floor(math.clamp(settings.distTac or 60, 0, 5000))
    UI.distance.Rev = math.floor(math.clamp(settings.distRev or 265, 0, 5000))
    if UI._distanceRefresh then UI._distanceRefresh() end
    UI.knock.colorbot = settings.knockColorbot == true
    UI.knock.triggerbot = settings.knockTriggerbot == true
    if UI._colorbotKnockRefresh then UI._colorbotKnockRefresh() end
    if UI._tbKnockRefresh then UI._tbKnockRefresh() end

    -- Keybinds, modes, body parts, movement (only when the config actually contains them)
    if settings.hasBinds then
        local s = settings
        local function resolveOrNone(kind, name)
            local value, code, label = resolveKeyBinding(kind, name)
            if not label then
                return nil, Enum.KeyCode.Unknown, "NONE"
            end
            return value, code, label
        end

        TB_KEY, TB_KEYCODE, TB_KEY_LABEL = resolveOrNone(s.tbKeyKind, s.tbKeyName)
        TB_HELD, TB_TOGGLED, TB_BIND_LISTENING = false, false, false
        TB_MODE = s.tbMode
        TB_LIST_MODE = s.tbListMode

        AIM_BIND.key, AIM_ASSIST_KEYCODE, AIM_ASSIST_KEY_LABEL = resolveOrNone(s.aimKeyKind, s.aimKeyName)
        AIM_ASSIST_BIND_LISTENING = false
        AIM_BIND.mode = s.aimMode
        AIM_BIND.held, AIM_BIND.toggled = false, false
        for partName in pairs(AIM_BIND.parts) do
            AIM_BIND.parts[partName] = s.aimParts[partName] == true
        end

        UI.esp.listMode = s.espListMode

        UI_HIDE_KEYCODE = keyCodeFromName(s.hideKeyName)
        UI_HIDE_KEY_LABEL = keyNameOf(UI_HIDE_KEYCODE)
        UI_KILL_KEYCODE = keyCodeFromName(s.killKeyName)
        UI_KILL_KEY_LABEL = keyNameOf(UI_KILL_KEYCODE)
        UI_BIND_LISTENING = nil
        if UI._refreshUIKeybindButtons then UI._refreshUIKeybindButtons() end

        local move = UI.move
        local speedKey = keyCodeFromName(s.speedKeyName)
        local flyKey = keyCodeFromName(s.flyKeyName)
        move.speedOn, move.speed = s.speedOn, s.speed
        move.speedKey = speedKey ~= Enum.KeyCode.Unknown and speedKey or nil
        move.flyOn, move.flySpeed = s.flyOn, s.flySpeed
        move.flyKey = flyKey ~= Enum.KeyCode.Unknown and flyKey or nil
        move.listening, move.speedListening = false, false

        UI.setModeVisual(UI._tbModeCard, s.tbMode)
        UI.setModeVisual(UI._aimModeCard, s.aimMode)
    end

    cancelHostTrigger()

    if not COLORBOT_ENABLED then
        TRIGGER_HELD = false
        TRIGGER_TOGGLED = false
        disconnectActivator()
    end

    if UI.refreshAll then UI.refreshAll() end
    if settings.esp and UI._espApply then
        UI._espApply(settings.esp)
    end
    if UI._refreshColorbotMasterUI then
        UI._refreshColorbotMasterUI()
    end
    if UI._refreshTriggerbotMasterUI then
        UI._refreshTriggerbotMasterUI()
    end
    updateTriggerBindText()

    rebuildHostList()

    return true
end

-- ===================== Reset all settings =====================
UI.setModeVisual = function(card, modeName)
    local T = UI.Theme
    if not card then return end
    for _, child in ipairs(card:GetChildren()) do
        if child:IsA("TextButton") then
            local title = child:FindFirstChild("Title")
            local sel = title ~= nil and string.upper(title.Text) == string.upper(modeName)
            child:SetAttribute("Sel", sel)
            child.BackgroundColor3 = sel and T.soft or T.card2
            local st = child:FindFirstChildOfClass("UIStroke")
            if st then st.Color = sel and T.accent or T.stroke end
            if title then title.TextColor3 = sel and T.text or T.sub end
        end
    end
end

UI.resetAllSettings = function()
    -- Colorbot / triggerbot / aim assist numbers, switches, colorbot key + mode, knock checks, ESP.
    local ok, err = applyConfigRecord(makeConfigRecord("default", getDefaultConfigSettings()))
    if not ok then
        return false, err
    end

    -- Triggerbot key, mode, list mode and selected players.
    TB_KEY, TB_KEYCODE, TB_KEY_LABEL = nil, Enum.KeyCode.Unknown, "NONE"
    TB_MODE = "Hold"
    TB_HELD, TB_TOGGLED, TB_BIND_LISTENING = false, false, false
    TB_LIST_MODE = "Whitelist"
    for userId in pairs(TB_SELECTED) do
        TB_SELECTED[userId] = nil
    end
    UI.tb.pendingChar = nil
    UI.tb.fov = 0
    UI.tb.strength = 0
    if UI._tbSettingsRefresh then UI._tbSettingsRefresh() end
    if UI._tbRefreshListMode then UI._tbRefreshListMode() end
    UI.setModeVisual(UI._tbModeCard, "Hold")
    if TB_PAGE_REFRESH then TB_PAGE_REFRESH() end
    if TB_LIST_REBUILD then TB_LIST_REBUILD() end

    -- Aim assist key, mode and body parts.
    AIM_BIND.key = nil
    AIM_ASSIST_KEYCODE = Enum.KeyCode.Unknown
    AIM_ASSIST_KEY_LABEL = "NONE"
    AIM_BIND.mode = "Hold"
    AIM_BIND.held, AIM_BIND.toggled = false, false
    AIM_ASSIST_BIND_LISTENING = false
    for key in pairs(AIM_BIND.parts) do
        AIM_BIND.parts[key] = true
    end
    UI.setModeVisual(UI._aimModeCard, "Hold")
    if AIM_BIND.refresh then AIM_BIND.refresh() end
    if UI._aimPartsRefresh then UI._aimPartsRefresh() end
    if resetAimAssistState then resetAimAssistState() end

    -- ESP back to its defaults.
    if UI._espApply then
        UI._espApply({
            enabled = false, type = "2D",
            box = false, filled = false, distance = false, name = false,
            health = false, snapline = false, bone = false, limit = false,
            limitValue = 1000,
            colors = {
                box = { 255, 255, 255 }, distance = { 200, 202, 210 },
                name = { 255, 255, 255 }, snapline = { 255, 255, 255 },
                bone = { 255, 255, 255 },
            },
            binds = { master = "NONE" },
        })
    end

    -- Movement (speed / fly).
    UI.move.speedOn, UI.move.speed = false, 10
    UI.move.flyOn, UI.move.flySpeed = false, 10
    UI.move.flyKey, UI.move.listening = nil, false
    UI.move.speedKey, UI.move.speedListening = nil, false

    -- Menu keys.
    UI_HIDE_KEYCODE, UI_HIDE_KEY_LABEL = Enum.KeyCode.F4, "F4"
    UI_KILL_KEYCODE, UI_KILL_KEY_LABEL = Enum.KeyCode.F7, "F7"
    UI_BIND_LISTENING = nil
    if UI._refreshUIKeybindButtons then UI._refreshUIKeybindButtons() end
    if UI._uiKeybindStatus then UI._uiKeybindStatus.Text = "Click a keybind, then press any keyboard key. ESC unbinds." end

    -- Host selection and any in-flight colorbot state.
    selectedActivator = nil
    disconnectActivator()
    cancelHostTrigger()
    if updateLiveStatus then updateLiveStatus() end
    return true
end


-- ====================================================================
--  GLASS UI  (frosted window, sidebar tabs, preview panel)
-- ====================================================================
UI.glass = { level = 0, snow = true, snowAmount = 100, blur = false }
UI.W = {}
UI.shell = {}
UI.refreshers = {}

;(function()
    local T = UI.Theme
    local new, corner, stroke, tween = UI.new, UI.corner, UI.stroke, UI.tween
    local W, shell, glass = UI.W, UI.shell, UI.glass
    local Lighting = game:GetService("Lighting")
    local GuiService = game:GetService("GuiService")
    local MarketplaceService = game:GetService("MarketplaceService")

    local WHITE = Color3.new(1, 1, 1)
    local BLACK = Color3.new(0, 0, 0)
    local DARK = Color3.fromRGB(28, 29, 33)
    local GM, GB, GR = Enum.Font.GothamMedium, Enum.Font.GothamBold, Enum.Font.Gotham
    local LEFT, RIGHT, CENTER = Enum.TextXAlignment.Left, Enum.TextXAlignment.Right, Enum.TextXAlignment.Center

    -- ---------------------------------------------------------------
    -- Tunables (blur trick + sizes). Tweak these if the blur looks off.
    -- ---------------------------------------------------------------
    local WIN_W, WIN_H = 720, 560
    local SIDEBAR_W = 134
    local PREVIEW_W, PREVIEW_H = 236, WIN_H
    local CORNER_RADIUS = 18               -- corner radius of the glass panels
    local GLASS_SEE_THROUGH_T = 0.9        -- glass transparency at opacity 0 (default: fully see-through)
    local GLASS_SOLID_T = 0.0              -- glass transparency at opacity 100 (solid)
    local BLUR_PART_DISTANCE = 0.5        -- studs in front of the camera for the blur pieces
    local BLUR_FOCUS_DISTANCE = 3.5        -- depth-of-field focus distance (further = stronger blur)
    local BLUR_BANDS = 6                   -- blur strips per rounded corner
    local BLUR_EDGE_PX = 1                 -- px the blur is pulled in from the panel edge
    local BLUR_PART_COUNT = 1 -- one piece per panel: Roblox clamps parts to a minimum size, so thin strips turned into huge bands

    local order = 0
    local function nextOrder()
        order = order + 1
        return order
    end

    local function reg(fn)
        UI.refreshers[#UI.refreshers + 1] = fn
    end
    W.register = reg

    local function syncWidgets()
        for _, fn in ipairs(UI.refreshers) do
            pcall(fn)
        end
    end
    UI.syncWidgets = syncWidgets

    local function dead()
        return SCRIPT_KILLED or not MENU_UNLOCKED
    end
    W.dead = dead

    local function pointerOf(input)
        local t = input.UserInputType
        if t == Enum.UserInputType.Touch then
            return Vector2.new(input.Position.X, input.Position.Y + GuiService:GetGuiInset().Y)
        end
        return UserInputService:GetMouseLocation()
    end

    -- ---------------------------------------------------------------
    -- Layers
    -- ---------------------------------------------------------------
    local dimLayer = new("Frame", {
        Name = "DimLayer", Size = UDim2.fromScale(1, 1), BackgroundColor3 = BLACK,
        BackgroundTransparency = 1, BorderSizePixel = 0, Active = false, ZIndex = 1,
    }, whitelistGui)

    wlCard = new("Frame", {
        Name = "WhitelistPanel", Size = UDim2.fromOffset(WIN_W, WIN_H),
        Position = UDim2.fromScale(0.5, 0.5), AnchorPoint = Vector2.new(0.5, 0.5),
        BackgroundTransparency = 1, BorderSizePixel = 0, Active = false, ZIndex = 10,
    }, whitelistGui)
    local cardScale = new("UIScale", { Scale = 1 }, wlCard)
    shell.scale = cardScale

    local function fitScale()
        local vp = whitelistGui.AbsoluteSize
        if vp.X < 10 or vp.Y < 10 then return end
        local s = math.min(
            1,
            (vp.X / 2 - 12) / (WIN_W / 2 + 12 + PREVIEW_W),
            (vp.Y - 24) / WIN_H
        )
        cardScale.Scale = math.max(0.45, s)
    end
    whitelistGui:GetPropertyChangedSignal("AbsoluteSize"):Connect(fitScale)
    fitScale()

    -- ---------------------------------------------------------------
    -- Glass panels (tint + sheen) and background blur
    -- The blur is a depth-of-field effect that only blurs the game behind each panel. Each panel is
    -- covered by blur pieces placed in front of the camera, mapped with the camera's own rays so they
    -- line up exactly with the rounded edges.
    -- ---------------------------------------------------------------
    local panels = {}

    local blurFx = Instance.new("DepthOfFieldEffect")
    blurFx.Name = "ArmadaGlassBlur"
    blurFx.FarIntensity = 0
    blurFx.NearIntensity = 1
    blurFx.FocusDistance = BLUR_FOCUS_DISTANCE
    blurFx.InFocusRadius = 0
    blurFx.Enabled = false
    pcall(function() blurFx.Parent = Lighting end)

    local function makeBlurPart(name)
        local ok, part = pcall(function()
            local p = Instance.new("Part")
            p.Name = name
            p.Anchored = true
            p.CanCollide = false
            p.CanQuery = false
            p.CanTouch = false
            p.CastShadow = false
            p.Locked = true
            p.Material = Enum.Material.Glass
            p.Transparency = 0.98
            p.Reflectance = 0
            p.Color = WHITE
            p.Size = Vector3.new(1, 1, 0.05)
            return p
        end)
        return ok and part or nil
    end

    local function makeGlass(parent, name, w, h, x, y)
        local f = new("Frame", {
            Name = name, Size = UDim2.fromOffset(w, h), Position = UDim2.fromOffset(x, y),
            BackgroundColor3 = T.glass, BackgroundTransparency = GLASS_SEE_THROUGH_T,
            BorderSizePixel = 0, Active = true, ZIndex = 1,
        }, parent)
        corner(f, CORNER_RADIUS)
        local st = stroke(f, WHITE, 1, 0.82)
        local sheen = new("Frame", {
            Name = "Sheen", Size = UDim2.fromScale(1, 1), BackgroundColor3 = WHITE,
            BackgroundTransparency = 0.94, BorderSizePixel = 0, Active = false, ZIndex = 1,
        }, f)
        corner(sheen, CORNER_RADIUS)
        new("UIGradient", {
            Rotation = 62,
            Transparency = NumberSequence.new({
                NumberSequenceKeypoint.new(0, 0),
                NumberSequenceKeypoint.new(1, 1),
            }),
        }, sheen)
        local parts = {}

        local entry = { frame = f, sheen = sheen, stroke = st, parts = parts }
        panels[#panels + 1] = entry
        return f, entry
    end

    local function glassAlpha()
        return math.clamp(glass.level, 0, 100) / 100
    end

    -- glass.level is the opacity slider: 0 = see-through (default), 100 = solid
    function shell.applyGlass()
        local L = glassAlpha()
        local t = GLASS_SEE_THROUGH_T + (GLASS_SOLID_T - GLASS_SEE_THROUGH_T) * L
        for _, p in ipairs(panels) do
            p.frame.BackgroundTransparency = t
            p.sheen.BackgroundTransparency = 0.94 + 0.06 * L
            p.stroke.Transparency = 0.82 - 0.2 * L
        end
    end

    local blurStep = "ArmadaGlassBlurStep"
    local function destroyBlur()
        pcall(function() RunService:UnbindFromRenderStep(blurStep) end)
        for _, p in ipairs(panels) do
            for _, part in ipairs(p.parts) do
                if part then pcall(function() part:Destroy() end) end
            end
        end
        pcall(function() blurFx:Destroy() end)
    end

    -- Blur layout: the rounded panel is covered by rectangles that stay inside its curve, so the
    -- blur never spills past the corners. Each corner is split into BLUR_BANDS horizontal strips.
    local function blurRects(px, py, w, h, R)
        local c = BLUR_EDGE_PX + R * 0.45
        return { { px + c, py + c, px + w - c, py + h - c } }
    end

    -- Where a screen pixel lands on the blur plane (BLUR_PART_DISTANCE in front of the camera),
    -- in camera-local coordinates. Using the camera's own ray keeps the blur aligned with the GUI.
    -- Calibrated against WorldToViewportPoint (the renderer's own projection): project the plane's
    -- origin and two unit steps, which gives the exact pixels-per-stud on the plane, then invert.
    local function calibratePlane(cam)
        local cf = cam.CFrame
        local o = cam:WorldToViewportPoint(cf * Vector3.new(0, 0, -BLUR_PART_DISTANCE))
        local px = cam:WorldToViewportPoint(cf * Vector3.new(1, 0, -BLUR_PART_DISTANCE))
        local py = cam:WorldToViewportPoint(cf * Vector3.new(0, 1, -BLUR_PART_DISTANCE))
        return { ox = o.X, oy = o.Y, kx = px.X - o.X, ky = py.Y - o.Y }
    end
    local function planePoint(cal, sx, sy)
        return (sx - cal.ox) / cal.kx, (sy - cal.oy) / cal.ky
    end

    pcall(function() RunService:UnbindFromRenderStep(blurStep) end)
    RunService:BindToRenderStep(blurStep, Enum.RenderPriority.Last.Value, function()
        if SCRIPT_KILLED or not whitelistGui.Parent then
            destroyBlur()
            return
        end
        local menuOn = false -- background blur removed
        if not menuOn then
            -- Nothing to do every frame while the blur is off: clean up once, then return immediately.
            if not UI._blurIdle then
                UI._blurIdle = true
                blurFx.Enabled = false
                for _, p in ipairs(panels) do
                    p.lastCF = nil
                    for _, part in ipairs(p.parts) do
                        if part and part.Parent then part.Parent = nil end
                    end
                end
            end
            return
        end
        UI._blurIdle = false
        local cam = workspace.CurrentCamera
        if not cam then return end
        blurFx.Enabled = menuOn
        local scale = cardScale.Scale
        -- Only reposition the blur pieces when the camera or a panel actually changed.
        -- Repositioning every frame was the main cost.
        local camCF = cam.CFrame
        local cal = menuOn and calibratePlane(cam) or nil
        local vs, gs = cam.ViewportSize, whitelistGui.AbsoluteSize
        local ux = (gs.X > 0) and (vs.X / gs.X) or 1
        local uy = (gs.Y > 0) and (vs.Y / gs.Y) or 1
        for _, p in ipairs(panels) do
            local rects = nil
            if menuOn and p.frame.Visible and p.frame.AbsoluteSize.X > 4 then
                local pos, size = p.frame.AbsolutePosition, p.frame.AbsoluteSize
                if p.lastCF == camCF and p.lastPX == pos.X and p.lastPY == pos.Y
                    and p.lastSX == size.X and p.lastSY == size.Y and p.lastScale == scale then
                    continue
                end
                p.lastCF, p.lastPX, p.lastPY = camCF, pos.X, pos.Y
                p.lastSX, p.lastSY, p.lastScale = size.X, size.Y, scale
                rects = blurRects(pos.X * ux, pos.Y * uy, size.X * ux, size.Y * uy, CORNER_RADIUS * scale)
            else
                p.lastCF = nil -- force a full reposition when the panel shows again
            end
            for i, part in ipairs(p.parts) do
                local r = rects and rects[i]
                if part and r and r[3] - r[1] > 1 and r[4] - r[2] > 1 then
                    local lx0, ly0 = planePoint(cal, r[1], r[2])
                    local lx1, ly1 = planePoint(cal, r[3], r[4])
                    local cx, cy = (lx0 + lx1) / 2, (ly0 + ly1) / 2
                    local x0, x1, y0, y1 = lx0, lx1, ly0, ly1
                    local sw, sh = math.abs(x1 - x0), math.abs(y0 - y1)
                    if sw >= 0.055 and sh >= 0.055 then
                        part.Size = Vector3.new(sw, sh, 0.05)
                        -- front face sits exactly on the calibrated plane; the box extends behind it
                        part.CFrame = camCF * CFrame.new(cx, cy, -(BLUR_PART_DISTANCE + 0.025))
                        if part.Parent ~= cam then part.Parent = cam end
                    elseif part.Parent then
                        part.Parent = nil
                    end
                elseif part and part.Parent then
                    part.Parent = nil
                end
            end
        end
    end)

    -- ---------------------------------------------------------------
    -- Line / icon helpers
    -- ---------------------------------------------------------------
    local function line(parent, x1, y1, x2, y2, color, th, z)
        local dx, dy = x2 - x1, y2 - y1
        local len = math.sqrt(dx * dx + dy * dy)
        return new("Frame", {
            AnchorPoint = Vector2.new(0.5, 0.5),
            Position = UDim2.fromOffset((x1 + x2) / 2, (y1 + y2) / 2),
            Size = UDim2.fromOffset(len, th),
            Rotation = math.deg(math.atan2(dy, dx)),
            BackgroundColor3 = color, BorderSizePixel = 0, ZIndex = z or 3, Active = false,
        }, parent)
    end
    shell.line = line

    -- Smooth anti-aliased icons (eye, gear, AK-47) embedded as PNGs. They are written out with the
    -- executor's writefile and loaded with getcustomasset; if either is missing the drawn icons are used.
    local ICON_B64 = {
        eye = "iVBORw0KGgoAAAANSUhEUgAAACQAAAAkCAYAAADhAJiYAAAChUlEQVR4nO2Xv05UURDGv70L/osxMRRIorY2INEESChpTEhM8A2MDSTWNpbWZJ+AWJjwBFY+gJWJJMbGwmaVIJUJBbjA3Z8F3wmzh/2DCustmGRy9p6d+eY7M3PuPacGqEpS/G8CuVwQGiQj/+BbCxqFoOdOqLAiqRwQtK4jsm3rmRIqDF4G8GuS7kq6Kemq5/Yk/ZTUlLSbkeM0xAYRqplM6efbkh5JeizpgaQ76l6yb5I2JL2V9E7S90CsrX6ZBXppEX7PAm+AHTrlENgHWtZ9z0XZse9sD+wO7UWm7vE60ADKEOBX9txLStvG54YxY4wOrXV5U49IOpQ0I+m1pEmneF/SZduUkj5ZP0r64flbkh5Kum+te74l6ZLL+1nSM0kfQqyeJRvx+ATY9cpaYZVN4CUwGXzmgKfWuTA/adtm8E9Yu44RY54oWUrhEsd90ArjK2As2D8HNruUatP/Jbsx+0YsHGMpL1/eZPN2aHPUoABfs5UXwFq26gNrzOYanc07ZyyM3bb9TCSVAhReyVZwANgAxu1wxeNKsOnW3GXwX8l8x40ZYzQduwCKmK71sGKAL8CE/xv1eAPYdtBU1lVgyrrquUPbbNsnYkwYO8ZaT1lK6VzIVr3nAKnpEunFDKjBya3byGwWOS5JauBpx4jZXCD00HuOawqwnK0qAb2w3YHB7jnQqLXuudI2bftEjIS5HIi3zaF6x49+JZs+x5JN9StZ5Zo6bvv0Vh3mtt8i2/YxSzMM/8U4H3Cr++mozMe18sePvHxDP6D1IhRPAGKIR9huJYsy9EP+IEJJ4jUoyblcg05LKBKLF8V+8lcXxT8l1OGrClyluwU+U6nc8eOC0CD5DSTQsuSteCIxAAAAAElFTkSuQmCC",
        gear = "iVBORw0KGgoAAAANSUhEUgAAACQAAAAkCAYAAADhAJiYAAAD0UlEQVR4nM2Yz2ucRRjHP+/uxsbQUpUK9lLBHwShptRLL0XxsNqT2GP/gNIWb4Ko91Av4qXx4MWDlp57aGn90Yu5+yOlhNAoeJKAxEoTzG5236+H9zvJs7O7Npu1sAMPM+88z/N9vjPzzOzMFpIYsxTZ91iAjTF8a67LPfbvqeyXUBECHs5I/B1sRp6t2qNNBvoIOAV8D6wAq5YV952yzcj4xYg5lMi8CPwIHBpi9xB4DfiV3tncU4BH6evBLhE6bzKtAT4t687TO0s51sDyXzlUo3dktQD6soNNAX8BX9jmAlVOyTb14JdjDZy1YYSSwxzwArBMlR/toC8sfwAfu/8d4Olg07UAzAKvAL8BS0NJScql7npeUldVaUu6JekNSZclPZTUsX5ZUsOy7L6ObS7b55YxZP18FmtHhpG5FJxbgVgsW66/tF/d7aiLJce6NIhUJFOTVEg6JmnTo+wEwLYlBluSdCRgHHFfJJ38Ukm4m45VODaSejI+7aAPgBmvfR34GrhJlcBTwAGqw++ic2wGaFpm3HfRNgeC301j1Y0941i955WZFa4PSVqTVFruhtE3JX2qav2fk3Rc0rfZjG2577ht5u3TDDh3A/6aY+5wiMuFpOc9vV07/CzppNSX+OcCia6kbUvMtXMD/E4as7Rt2zF3OMQZKiRNS/rdgGndO5I+tH5K0mn3l+rNDQW/0u3T9imM0Qk2cqzpEL8vqZH0lqqES2QS+FnrFzPQq5LetFzNdIv2ORsGkUhtOlaM3bftk+KEweJWXZD0lHbPGUm6pv5luRYG07XPQoa16Bg9ZJTtMnxyNoBfgDPAundIDdgAXspO2M/9/YSl5r6EVbPPhttTxjzjGI38tN7P9eOxlpxQDegAJ4DbwDPANtUoDlLdecrg956/25bSfQmrtM9Bt7eNedsxOn0cJjWpJ27bT8zBmO5D6R67TnXhetbfdeAnt5vA28AWsAC8CnwGvE71mwXVbfEH4H3gT2AemAa+Ab4zVt22hWOt93AI7Buur3iELddfSbqRLcsDSRdsf0zVb1XTbax7kPncMFbEvpLFnrzrx0Rf0CbyCpvvujlJ70qazfTXw8jvhf57of965jNrrLksRo8Me3Wk03jJkk7eBtVtr/SuEHAU+MQ2R61Lj8O6pUP1alkJWCM9gyKp9EYvLV3gvvvbVM+ej4Jfi+oYuG/bYgjWwPJ/PaUTSPpr5rE9pfOSlmOV6qC8A6xRXS823L5j3eqoZGD0GdrxY3dWDgNPuv0PY/4ds19CsLt8OUB6Yu/rD6txCOUEYDDBkcq/QumKhyb0proAAAAASUVORK5CYII=",
        aim = "iVBORw0KGgoAAAANSUhEUgAAACQAAAAkCAYAAADhAJiYAAADnklEQVR4nLWYz4sVRxDHP/N+bHYTNGokFzcJEhFcEDRgFDegJwmecgi5JIfkLxCzBxG9elhYdvekeIqX4EGIhwiiGEIISoIEiRgj+YEHzcWLYoyr7+17Xw9TzdS2b+bNm9kUFN3TXVX9naqu6p5JJFGREmsngBPWPwYsWb+aYUlVuWXtMWV0PJobmVtV3eNoM9Cz/vq6xlYDUAdoWv9RXWONmrotBwagbWOV7VbxUJN0w3bsecnN/QcsW79BuvF7jECjAEpskbDAZuB9YA9ZRn0I3AeuA384YKJs1pXc/Ynr75N0XtITl13LxoGeSboo6aDTa5RZqwyYYGhC0km3aC8C4cH13PPXktaXBZWouDA2gD6wDrgATJPukSZZYfzbwgQwCbwbnG+ybeCWhfMfZ3PkkCWSmpLGJV2zt12y9omkWUnbJbWdTtvGZl1Ig85tSRvMS7meKgLUtPZUZPgXSVMDwhovMmWyUrqnJOmbyHZpQEFh2gw9t/aGpDXOGw3zpOeG89oa0/GgPioCNQzQt2akqzQEW/XyWdXM6QeZrabbldSX9LMDPhBQYsqBx6x9R2mYQibNOs+ULRdBdta9WF/SDlvjlWjtpOGyIXDH2p3AuO39HnCGlZU3ZNkMcNN4JprrWf9MpLfX1ngera0W6X3mS2At8NRSug/sN7BN4C/gT7KK2zIDh4E5l7RzprvgZDDdu8AW0/8CeJOs8r8KPAbmkbSofArh+l5ZNvnNe8dkOsbLNhbLYTa8zUG0WOe0/1+oBRwFHgBvkYWoT3pw7jC5SVaGIPRPA/OsvIKcNv0gE0I8afMN4HfgB7KQJcA9YKEoQz5xmbEsaVsUgnDgzki6aTwTzYXQbTMbXbN5KG/dorTf4vaFtDpp37F2t3LSPs9Q8MIVpXWjbmH8V1kN+tXkcgtjUaU+YG8Uyv4N1T86PhsAfiggr3A2MljncP1OBcfGMEBB8XVJv5nBKtePAOa+pE3DAJW9oG0CLgNTpEfLmJPJu6DhZO8BB4A71LigxRt8o6RzyiiUg5h8ekvSJUlvq2DflA3ZIFBI+ljSj0ozxoOIwV2X9HmOjVweFjJPiXFw93vAB8CnwC4buw18BVwFfsrRK6aSHoqzz38WHXFeWRwgO5L9Kl+u4V4zZm/9mpubIPuU7jLiVyvU+7bvkx6eXTfWs7E+Ff8Prcbfj3Ws9FotWg1AD8muH3frGqsDKNxj5oE3bGyBCn88PL0Afme+Z3JW6R0AAAAASUVORK5CYII=",
        ak47 = "iVBORw0KGgoAAAANSUhEUgAAAFAAAAAiCAYAAADI+15nAAAF7ElEQVR4nO2ZW4hVVRjHf/uc7dR4m4sppFJjVj5ohEVRdEEokBDMlwzpJcmCiIICi96qF8GXIILo9mJQTVhNdDGKriBEmjAEgVGmUzPjpDbWjNnMnDmrh+/72mvvs/e5H+YE84fFWWevy/7Wf323tXbgnGMe9SM31wL83zFPYIOYJ7BBzBPYIMK5FqCFCLQYiq14SasIDCp3qRr1pgmuyrE5LQ6YrfUlQQvSmID6F501X62upghcAizWugOOIQQF+tsUGQPnnKm6TdjIxDZPB9BN42YTAKfqGNcHHAYWAgXE0p4C9np9coh8G4G7gEGgnxoVIKRU1X2V9uFS+vows10GfARcRrTj9aCo8r0DHATWVDFfgBC2VuXwcbc39i3ghNbvAJ4EDpFNYJYFFENgqZa/gEkVvJzmmMbapEaqmcYtwPVlxteKB7TUCp+EWeAaLSAE25wdCOkBkCfaJBsfkM1HEALXAu8jpvI7cAYYBUaAYa2PatsYcF4nT5v0aeBxFSjvv6jCYpPwF1/U8V8AlwMrtT1P5N/ynkz2Xv/9IfAr8Ln2exFYoPWLtL0bUaQJTwbT6DXar+DNdwo4HgIntWOfliycBf5ASPwFMYHjKtiw/m4DLqTU1M4T7XIldBJfvGl6L2KWC7w2v18WzgEfA3uA71LaL9DfxYjchUT7lcA3CMG2sTlE0TYHzrlexP77iHbb2PcXUW7xRZ1wEUJA4D3PAfcifrGDyoFlAHEBNjaJQ8gGbtf6ILALOAp8qXXTSAtCryHatdCTy9KW24DVwIzKeJYovZtGCLyZOCcm246QyOeZsFkO0w8gvt+zoLM8YxwqnJl+pVwrK0jNINp3AHE524FPgOcR0r4FdgM7iWvmcuCxCu8E2dxtZWRK+kUHTIWIbS/zGrKQzOyTL0gbb5vxCjBFdfncosTY5FydwBKv3qP1DqArRYYJ4Dcdk7Y5ftZRTPRxiGl3pTzPA2GIkNelDfWmHJXGdWppBLPIQguINjriml30ntv/HGKSW5AMIy09y0JO59sE7M/oMxE457YC71Hq96pB0qTtfz4xlz234FJOE9NOCpZigASscWCD1k8imcQZ5LRxnffOACHwCuB01auKIwTuA1YRuZ88EkT3hcDFiRemwSVKMhesRLy1V3v2Touu/YjDX6WlkKhblC4k5NyHEG5pT60oIGlPKkIVIA0WXEyb0kiaQXZ2DBhCNGAKSVJ7vEWYOb2BLHQz2VG2ALyELNr80wngBZW1G9GErDzQ39QZ4MesxdcA21Dj4D+LCoF1xE8fzhPchBlFCBpBSPpJf0cQ8pLn1S3ECTTsBS5FCEzzRUbMQeD1lPZhLbWgGZcbmZmDJdJpKcxRJE34AMm3/qS8CfjRbFKf+YkniHb+TLRJaQiQdORtIg3zTz61+umW3AMaQuAJYD3iX44gWfsBJK+aTvQ3c04LHv5ifXJMCyeRiDmupZdSDTWzvBq4Afha55pNzNc2CIF/gHsQ0zpC/Chjtm+7WCkJNnLPec+MpBHk6AfwJvAg8bOrwW5hdgFf0WaEJREiixvTAnHSar6hVYx7ddOwFcCjSFqxjuzAZAFhK7KpQ0R3d20H81mWklgOVu+NrV2mrtX/PkGngZuAHcCHiAamEWNReynwCKVm3l5wzjWr5PR3vXNu2jlX1DLjBHtSxvRrW8HFYWPHnXOrnXOBN39blWZ+1rS5bkUO/Xazk0f86n7tEyJamkM0bIRSTTQt7KbNtbCZBJrJb/Ke2dHtM6K7uAIS3c33PkREWPLTgkOOUSvJTrznFM0SyHznEuBGrfvJ+cteP8Msoo0DwDNa94OWkdoLPEybamGzPmvat4TbgU8TbceAq5AcEEovCSzPGwDuJDqNWF+7rN2ImHuzP5s2hGZpoGnaIPKVazdyB3gYeA74m+wvfeYrdwLfEz/jov9XAPd7/9sGrfiwHpuf0tvcNFgQ2YAkzz1E5voD8CpytBvSZ22jgc0m0M8nofQDTTmE2n87cmszCDwLvEt0tm47/AsTMLzun3zERAAAAABJRU5ErkJggg==",
        bolt = "iVBORw0KGgoAAAANSUhEUgAAACQAAAAkCAYAAADhAJiYAAADHklEQVR4nMWYT0hUURSHf2dm1KQIFxVB0SqKgsBVBQVREEJQFNVCaRmtJEgi2tgqigiMsKJF22zT300EhS2SikSLIqJ20qqEgkyx0vlavHP1Ns4M83RmuvC475177zvfnHt+Z+6MVOUGZKr9znk3IOv9PqAf2O/P9YeMYNYDYyRtKC1QVcjdYR5okfRA0hJJ05JeVNNPpTAGZL1/5JGZ9P6Qz8nVEyjnfa9D/I6g1vhYfSIUwXQ6xBSQ9/thj5rVG6YtghgHfvp9bzyv1jCxor470A+gG5hYSP6k3tsiimqRZJKOSPomqVnShKRXviSf1kcamGKKArjg40/9eTDkT01zqIiiAO66bTkw6rarQAZo9H7OVU2YzgjmNbDYI7Ersu+Zr5+KwgnkzGwKaJP0UEnufZW02cxGfM5ZSaeVVOghSeMFr8koyacmSZfN7A6QMbN0OVZCUb+B7W5v9H6Yyts7XzNn68pKkuKKkqRjZjbg2zjltnOSuiRlJeG2sANZSa0+t0lST8F4RZEppajzPt6Q4l03mC2e192WrRgmdsi/irrntkaHLVSPRVdYfyJa/xzIhQ+aBibkTVDUNPAsxfqgyJ2+Ng98AVa7vaTs51CGzAcOS+pTooyspLeSRv0+H61dpEQ1t4MjX79KidpWKMmp3WbWD2TNbLrSDzdDD7xPoZpwMsz6lQMGovGuOHLlWrkJ3ZJOekRCDSEaz0naKKlR0ku3NZjZJHBN0ja33TSzHryWVRyZtA1oJTn/ALSHJAWORpF5AzSnTuISDgtVM6Me74+701/ABl+z1Z/zJEV0bXjXgiNQBjQoqM+BPpGUgWXACLP1Zm88v1YwYVtywAd3fMttT6KtOlNzGHcQFLjOtwagAzgVwdyPoGt7no62q8OdjwGXSIofwEdgKdU686QAuuIAE8z+0hgHNvl4uu+pBQCFHBp0oGngj9+3x9D1gAn5s9KjAbM/Ci/WFcadhS/cNocISf04wFQjidMkXnC2Q8kxNSfps6RQpfNmRqnFVW/Rlg15ZKaALW6rTxKXADpA8mfUwf8GU6rVotb8BXlPlWMZHhdqAAAAAElFTkSuQmCC",
    }
    local B64_CHARS = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
    local B64_LOOKUP = {}
    for i = 1, #B64_CHARS do B64_LOOKUP[B64_CHARS:byte(i)] = i - 1 end
    local function b64decode(data)
        local out, n, bits = {}, 0, 0
        for i = 1, #data do
            local v = B64_LOOKUP[data:byte(i)]
            if v then
                n = n * 64 + v
                bits = bits + 6
                if bits >= 8 then
                    bits = bits - 8
                    local div = 2 ^ bits
                    out[#out + 1] = string.char(math.floor(n / div) % 256)
                    n = n % div
                end
            end
        end
        return table.concat(out)
    end
    local iconAssetCache = {}
    local function iconAsset(name)
        if iconAssetCache[name] ~= nil then return iconAssetCache[name] or nil end
        iconAssetCache[name] = false
        local ok, res = pcall(function()
            local gca = getcustomasset or getsynasset
            if type(writefile) ~= "function" or type(gca) ~= "function" then return nil end
            local fname = "armada_icon_" .. name .. ".png"
            if type(isfile) ~= "function" or not isfile(fname) then
                writefile(fname, b64decode(ICON_B64[name]))
            end
            return gca(fname)
        end)
        if ok and res and res ~= "" then
            iconAssetCache[name] = res
            return res
        end
        return nil
    end

    local function mkIcon(parent, kind, x, y)
        local box = new("Frame", {
            Size = UDim2.fromOffset(18, 18), Position = UDim2.fromOffset(x, y),
            BackgroundTransparency = 1, BorderSizePixel = 0, Active = false, ZIndex = 3,
        }, parent)
        local strokes, fills = {}, {}
        local function ring(sz, th, px, py)
            local f = new("Frame", {
                AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, px or 0, 0.5, py or 0),
                Size = UDim2.fromOffset(sz, sz), BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 3,
            }, box)
            corner(f, sz)
            strokes[#strokes + 1] = stroke(f, WHITE, th, 0)
        end
        local function rrect(w, h, r, th, px, py)
            local f = new("Frame", {
                AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, px or 0, 0.5, py or 0),
                Size = UDim2.fromOffset(w, h), BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 3,
            }, box)
            corner(f, r)
            strokes[#strokes + 1] = stroke(f, WHITE, th, 0)
        end
        local function dot(sz, px, py)
            local f = new("Frame", {
                AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, px or 0, 0.5, py or 0),
                Size = UDim2.fromOffset(sz, sz), BackgroundColor3 = WHITE, BorderSizePixel = 0, ZIndex = 3,
            }, box)
            corner(f, sz)
            fills[#fills + 1] = f
        end
        local function bar(w, h, px, py, rot)
            local f = new("Frame", {
                AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, px or 0, 0.5, py or 0),
                Size = UDim2.fromOffset(w, h), Rotation = rot or 0, BackgroundColor3 = WHITE,
                BorderSizePixel = 0, ZIndex = 3,
            }, box)
            fills[#fills + 1] = f
        end

        local imgName = (kind == "eye" and "eye") or (kind == "settings" and "gear") or (kind == "aim" and "aim") or (kind == "bolt" and "bolt") or nil
        local imgAsset = imgName and iconAsset(imgName)
        if imgAsset then
            local img = new("ImageLabel", {
                Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Image = imgAsset,
                ImageColor3 = WHITE, ZIndex = 3, Active = false,
            }, box)
            return { frame = box, set = function(color) img.ImageColor3 = color end }
        end

        if kind == "aim" then
            ring(13, 1.5)
            dot(3)
            bar(1.5, 4, 0, -7)
            bar(1.5, 4, 0, 7)
            bar(4, 1.5, -7, 0)
            bar(4, 1.5, 7, 0)
        elseif kind == "colorbot" then
            ring(15, 1.5)
            dot(3, -3, -2)
            dot(3, 3, -2)
            dot(3, 0, 3)
        elseif kind == "trigger" then
            bar(2, 9, 1.5, -3, 22)
            bar(2, 9, -1.5, 3, 22)
            bar(7, 2, 0, 0, 0)
        elseif kind == "eye" then
            -- thin almond outline with a hollow ring pupil (rounded line ends, no squares)
            local function seg(x1, y1, x2, y2, th)
                local dx, dy = x2 - x1, y2 - y1
                local len = math.sqrt(dx * dx + dy * dy)
                local f = new("Frame", {
                    AnchorPoint = Vector2.new(0.5, 0.5),
                    Position = UDim2.new(0.5, (x1 + x2) / 2, 0.5, (y1 + y2) / 2),
                    Size = UDim2.fromOffset(len + th * 0.6, th), Rotation = math.deg(math.atan2(dy, dx)),
                    BackgroundColor3 = WHITE, BorderSizePixel = 0, ZIndex = 3,
                }, box)
                corner(f, th)
                fills[#fills + 1] = f
            end
            local HW, HH, N = 9.5, 5.5, 14
            for i = 0, N - 1 do
                local xa = -HW + (2 * HW) * i / N
                local xb = -HW + (2 * HW) * (i + 1) / N
                local ya = HH * (1 - (xa / HW) ^ 2)
                local yb = HH * (1 - (xb / HW) ^ 2)
                seg(xa, -ya, xb, -yb, 1.5)
                seg(xa, ya, xb, yb, 1.5)
            end
            ring(5.5, 1.5)
        elseif kind == "settings" then
            -- cog outline (six rounded teeth) with a ring in the middle
            local function seg(x1, y1, x2, y2, th)
                local dx, dy = x2 - x1, y2 - y1
                local len = math.sqrt(dx * dx + dy * dy)
                local f = new("Frame", {
                    AnchorPoint = Vector2.new(0.5, 0.5),
                    Position = UDim2.new(0.5, (x1 + x2) / 2, 0.5, (y1 + y2) / 2),
                    Size = UDim2.fromOffset(len + th * 0.6, th), Rotation = math.deg(math.atan2(dy, dx)),
                    BackgroundColor3 = WHITE, BorderSizePixel = 0, ZIndex = 3,
                }, box)
                corner(f, th)
                fills[#fills + 1] = f
            end
            local ROUT, RIN, TEETH = 8.4, 6.0, 6
            local pts = {}
            for k = 0, TEETH - 1 do
                local base = k * (2 * math.pi / TEETH) - math.pi / 2
                local half = math.pi / TEETH
                -- valley, tooth start, tooth end, valley
                pts[#pts + 1] = { RIN, base - half * 0.95 }
                pts[#pts + 1] = { ROUT, base - half * 0.38 }
                pts[#pts + 1] = { ROUT, base + half * 0.38 }
                pts[#pts + 1] = { RIN, base + half * 0.95 }
            end
            for i = 1, #pts do
                local a = pts[i]
                local b = pts[i % #pts + 1]
                seg(a[1] * math.cos(a[2]), a[1] * math.sin(a[2]), b[1] * math.cos(b[2]), b[1] * math.sin(b[2]), 1.5)
            end
            ring(5, 1.5)
        elseif kind == "bolt" then
            -- lightning bolt outline (rounded line ends)
            local function seg(x1, y1, x2, y2, th)
                local dx, dy = x2 - x1, y2 - y1
                local len = math.sqrt(dx * dx + dy * dy)
                local f = new("Frame", {
                    AnchorPoint = Vector2.new(0.5, 0.5),
                    Position = UDim2.new(0.5, (x1 + x2) / 2, 0.5, (y1 + y2) / 2),
                    Size = UDim2.fromOffset(len + th * 0.6, th), Rotation = math.deg(math.atan2(dy, dx)),
                    BackgroundColor3 = WHITE, BorderSizePixel = 0, ZIndex = 3,
                }, box)
                corner(f, th)
                fills[#fills + 1] = f
            end
            local pts = {
                { 2.5, -8 }, { -4.5, 1 }, { -0.5, 1 }, { -2.5, 8 }, { 4.5, -2 }, { 0.5, -2 },
            }
            for i = 1, #pts do
                local a, b = pts[i], pts[i % #pts + 1]
                seg(a[1], a[2], b[1], b[2], 1.5)
            end
        elseif kind == "configs" then
            rrect(14, 14, 3, 1.5)
            bar(8, 1.5, 0, -3.5)
            bar(6, 1.5, 0, 3.5)
        else -- logo
            ring(16, 1.5)
            dot(5, -2, -2)
            dot(5, 2, 2)
        end

        return {
            frame = box,
            set = function(color)
                for _, s in ipairs(strokes) do s.Color = color end
                for _, f in ipairs(fills) do f.BackgroundColor3 = color end
            end,
        }
    end
    -- AK-47 logo: {x, y, length, thickness, rotation, corner radius}, 1 unit = 1 px on a 24 px box
    local AK47_PARTS = {
        {5.2, -2.1, 10.2, 1.5, 0, 0.5},
        {10, -2.1, 0.5, 1.9, 0, 0.15},
        {2, -3.7, 6.2, 1.3, 0, 0.5},
        {7.6, -4.6, 1, 2, 0, 0.3},
        {4.6, -1, 6.4, 1.3, 0, 0.6},
        {-2.2, -2.5, 8.6, 3.5, 0, 1},
        {-3.6, -4.5, 4.4, 0.9, 0, 0.3},
        {-8.2, -0.45, 5.61, 2.3, 148.86, 0.6},
        {-2.8, 3.4, 5.44, 1.7, 107.1, 0.5},
        {0.57, 1.81, 0.48, 0.6, 112.5, 0.1},
        {0.36, 2.17, 0.48, 0.6, 127.5, 0.1},
        {0.07, 2.46, 0.48, 0.6, 142.5, 0.1},
        {-0.29, 2.67, 0.48, 0.6, 157.5, 0.1},
        {-0.69, 2.77, 0.48, 0.6, 172.5, 0.1},
        {-1.11, 2.77, 0.48, 0.6, -172.5, 0.1},
        {-1.51, 2.67, 0.48, 0.6, -157.5, 0.1},
        {-1.87, 2.46, 0.48, 0.6, -142.5, 0.1},
        {-2.16, 2.17, 0.48, 0.6, -127.5, 0.1},
        {-2.37, 1.81, 0.48, 0.6, -112.5, 0.1},
        {-2.47, 1.41, 0.48, 0.6, -97.5, 0.1},
        {0.97, 1.43, 0.33, 1.5, 22.43, 0},
        {1.11, 1.49, 0.33, 1.5, 25.59, 0},
        {1.24, 1.56, 0.33, 1.5, 28.75, 0},
        {1.37, 1.63, 0.33, 1.5, 31.87, 0},
        {1.5, 1.72, 0.33, 1.5, 34.94, 0},
        {1.62, 1.81, 0.33, 1.5, 37.95, 0},
        {1.74, 1.91, 0.34, 1.5, 40.88, 0},
        {1.86, 2.01, 0.34, 1.5, 43.73, 0},
        {1.97, 2.13, 0.34, 1.5, 46.47, 0},
        {2.08, 2.25, 0.35, 1.5, 49.11, 0},
        {2.19, 2.38, 0.35, 1.5, 51.64, 0},
        {2.29, 2.51, 0.35, 1.5, 54.06, 0},
        {2.39, 2.66, 0.36, 1.5, 56.36, 0},
        {2.49, 2.81, 0.36, 1.5, 58.55, 0},
        {2.58, 2.97, 0.37, 1.5, 60.63, 0},
        {2.67, 3.14, 0.37, 1.5, 62.6, 0},
        {2.76, 3.31, 0.38, 1.5, 64.47, 0},
        {2.84, 3.49, 0.38, 1.5, 66.24, 0},
        {2.92, 3.68, 0.39, 1.5, 67.92, 0},
        {3, 3.88, 0.39, 1.5, 69.51, 0},
        {3.07, 4.08, 0.4, 1.5, 71.01, 0},
        {3.14, 4.29, 0.41, 1.5, 72.43, 0},
        {3.21, 4.51, 0.41, 1.5, 73.77, 0},
        {3.27, 4.74, 0.42, 1.5, 75.05, 0},
        {3.33, 4.98, 0.43, 1.5, 76.25, 0},
        {3.39, 5.22, 0.43, 1.5, 77.4, 0},
        {3.44, 5.47, 0.44, 1.5, 78.48, 0},
        {3.49, 5.72, 0.45, 1.5, 79.51, 0},
        {3.54, 5.99, 0.45, 1.5, 80.49, 0},
        {3.58, 6.26, 0.46, 1.5, 81.42, 0},
    }

    -- Draws the AK-47 logo from rounded pieces (see AK47_PARTS)
    function shell.akLogo(parent, x, y, color)
        local holder = new("Frame", {
            Position = UDim2.fromOffset(x, y), Size = UDim2.fromOffset(24, 24),
            BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 3, Active = false,
        }, parent)
        local akImg = iconAsset("ak47")
        if akImg then
            new("ImageLabel", {
                Position = UDim2.fromOffset(0, 3), Size = UDim2.fromOffset(40, 17), BackgroundTransparency = 1,
                Image = akImg, ImageColor3 = color, ZIndex = 3, Active = false,
            }, holder)
            return holder
        end
        for _, p in ipairs(AK47_PARTS) do
            local f = new("Frame", {
                AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, p[1], 0.5, p[2]),
                Size = UDim2.fromOffset(p[3], p[4]), Rotation = p[5], BackgroundColor3 = color,
                BorderSizePixel = 0, ZIndex = 3, Active = false,
            }, holder)
            if p[6] > 0 then corner(f, p[6]) end
        end
        return holder
    end
    shell.icon = mkIcon

    -- ---------------------------------------------------------------
    -- Window
    -- ---------------------------------------------------------------
    local win, winEntry = makeGlass(wlCard, "Window", WIN_W, WIN_H, 0, 0)
    shell.window = win
    win.Size = UDim2.new(0, WIN_W, 1, 0)

    local header = new("Frame", {
        Name = "Header", Position = UDim2.fromOffset(12, 12), Size = UDim2.fromOffset(WIN_W - 24, 38),
        BackgroundColor3 = WHITE, BackgroundTransparency = 0.93, BorderSizePixel = 0, Active = true, ZIndex = 3,
    }, win)
    corner(header, 13)
    stroke(header, WHITE, 1, 0.62)
    shell.akLogo(header, 12, 7, T.text)
    UI.label(header, "Armada", iconAsset("ak47") and 58 or 40, 0, 200, 38, 13, T.text, GB)
    local placeLabel = UI.label(header, "", 0, 0, 300, 38, 12, T.sub, GM, RIGHT)
    placeLabel.AnchorPoint = Vector2.new(1, 0)
    placeLabel.Position = UDim2.new(1, -16, 0, 0)
    placeLabel.Text = tostring(game.Name)
    task.spawn(function()
        local ok, info = pcall(function()
            return MarketplaceService:GetProductInfo(game.PlaceId)
        end)
        if ok and info and info.Name and placeLabel.Parent then
            placeLabel.Text = tostring(info.Name)
        end
    end)

    -- drag
    do
        local dragging, dragStart, startPos = false, nil, nil
        header.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1
                or input.UserInputType == Enum.UserInputType.Touch then
                dragging = true
                dragStart = input.Position
                startPos = wlCard.Position
            end
        end)
        UserInputService.InputChanged:Connect(function(input)
            if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement
                or input.UserInputType == Enum.UserInputType.Touch) then
                local d = input.Position - dragStart
                wlCard.Position = UDim2.new(
                    startPos.X.Scale, startPos.X.Offset + d.X,
                    startPos.Y.Scale, startPos.Y.Offset + d.Y
                )
            end
        end)
        UserInputService.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1
                or input.UserInputType == Enum.UserInputType.Touch then
                dragging = false
            end
        end)
    end

    -- sidebar divider
    new("Frame", {
        Position = UDim2.fromOffset(SIDEBAR_W, 62), Size = UDim2.fromOffset(1, WIN_H - 62 - 14),
        BackgroundColor3 = WHITE, BackgroundTransparency = 0.88, BorderSizePixel = 0, ZIndex = 2, Active = false,
    }, win)

    -- profile footer
    do
        local profileBg = new("Frame", {
            Name = "ProfileBackdrop", Position = UDim2.fromOffset(10, WIN_H - 60), Size = UDim2.fromOffset(SIDEBAR_W - 20, 46),
            BackgroundColor3 = WHITE, BackgroundTransparency = 0.95, BorderSizePixel = 0,
            ZIndex = 2, Active = false,
        }, win)
        corner(profileBg, 12)
        stroke(profileBg, WHITE, 1, 0.85)
        local avatarHolder = new("Frame", {
            Position = UDim2.fromOffset(14, WIN_H - 52), Size = UDim2.fromOffset(30, 30),
            BackgroundColor3 = T.card2, BorderSizePixel = 0, ZIndex = 3,
        }, win)
        corner(avatarHolder, 15)
        local initial = UI.label(avatarHolder, string.upper(string.sub(LocalPlayer.DisplayName, 1, 1)),
            0, 0, 30, 30, 12, T.sub, GB, CENTER)
        initial.ZIndex = 4
        local img = new("ImageLabel", {
            Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, BorderSizePixel = 0,
            Image = "", ScaleType = Enum.ScaleType.Crop, ZIndex = 5,
        }, avatarHolder)
        corner(img, 15)
        UI.applyAvatar(img, LocalPlayer.UserId, initial)
        local nm = UI.label(win, LocalPlayer.DisplayName, 52, WIN_H - 52, SIDEBAR_W - 60, 16, 11, T.text, GM)
        nm.TextTruncate = Enum.TextTruncate.AtEnd
        nm.ZIndex = 3
        local hd = UI.label(win, "@" .. LocalPlayer.Name, 52, WIN_H - 36, SIDEBAR_W - 60, 14, 9, T.dim, GR)
        hd.TextTruncate = Enum.TextTruncate.AtEnd
        hd.ZIndex = 3
    end

    -- content (fades at top/bottom like the reference)
    local contentFade = new("CanvasGroup", {
        Name = "Content", Position = UDim2.fromOffset(SIDEBAR_W + 10, 58),
        Size = UDim2.fromOffset(WIN_W - SIDEBAR_W - 22, WIN_H - 58 - 10),
        BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 2,
    }, win)
    new("UIGradient", {
        Rotation = 90,
        Transparency = NumberSequence.new({
            NumberSequenceKeypoint.new(0, 0),
            NumberSequenceKeypoint.new(0.02, 0),
            NumberSequenceKeypoint.new(0.97, 0),
            NumberSequenceKeypoint.new(1, 1),
        }),
    }, contentFade)

    local popupLayer = new("Frame", {
        Name = "Popups", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1,
        BorderSizePixel = 0, Active = false, ZIndex = 60,
    }, win)

    -- ---------------------------------------------------------------
    -- Preview panel (ESP Preview / Aim Preview)
    -- ---------------------------------------------------------------
    local preview, previewEntry = makeGlass(wlCard, "PreviewPanel", PREVIEW_W, PREVIEW_H, WIN_W + 12, 0)
    preview.Size = UDim2.new(0, PREVIEW_W, 1, 0)
    preview.Visible = false
    UI.previewPanel = preview
    local previewTitle = UI.label(preview, "ESP Preview", 0, 12, PREVIEW_W, 20, 12, T.sub, GM, CENTER)
    previewTitle.ZIndex = 3
    new("Frame", {
        Position = UDim2.fromOffset(16, 40), Size = UDim2.fromOffset(PREVIEW_W - 32, 1),
        BackgroundColor3 = WHITE, BackgroundTransparency = 0.9, BorderSizePixel = 0, ZIndex = 2, Active = false,
    }, preview)

    local espBody = new("Frame", {
        Name = "EspBody", Position = UDim2.fromOffset(0, 44), Size = UDim2.fromOffset(PREVIEW_W, PREVIEW_H - 44),
        BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 3, Active = false,
    }, preview)
    local aimBody = new("Frame", {
        Name = "AimBody", Position = UDim2.fromOffset(0, 44), Size = UDim2.fromOffset(PREVIEW_W, PREVIEW_H - 44),
        BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 3, Active = false, Visible = false,
    }, preview)

    UI.aimBody = aimBody

    -- character geometry (inside a 150x214 frame)
    local CHAR_W, CHAR_H = 150, 214
    local CHAR_X, CHAR_Y = (PREVIEW_W - CHAR_W) / 2, 120
    local PARTS = {
        Head     = { 57, 10, 36, 36, 10 },
        Torso    = { 42, 50, 66, 72, 4 },
        RightArm = { 14, 50, 27, 72, 4 },   -- the character faces you, so its right arm is on your left
        LeftArm  = { 109, 50, 27, 72, 4 },
        RightLeg = { 42, 124, 32, 82, 4 },
        LeftLeg  = { 76, 124, 32, 82, 4 },
    }
    local PART_ORDER = { "Head", "Torso", "RightArm", "LeftArm", "RightLeg", "LeftLeg" }
    local BODY_COLOR = Color3.fromRGB(22, 22, 25)

    local function buildCharacter(parent, clickable)
        local holder = new("Frame", {
            Name = "Char", Position = UDim2.fromOffset(CHAR_X, CHAR_Y), Size = UDim2.fromOffset(CHAR_W, CHAR_H),
            BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 3, Active = false,
        }, parent)
        local parts = {}
        for _, name in ipairs(PART_ORDER) do
            local r = PARTS[name]
            local cls = clickable and "TextButton" or "Frame"
            local props = {
                Name = name, Position = UDim2.fromOffset(r[1], r[2]), Size = UDim2.fromOffset(r[3], r[4]),
                BackgroundColor3 = BODY_COLOR, BorderSizePixel = 0, ZIndex = 4,
            }
            if clickable then
                props.Text = ""
                props.AutoButtonColor = false
            else
                props.Active = false
            end
            local f = new(cls, props, holder)
            corner(f, r[5])
            parts[name] = f
        end
        return holder, parts
    end

    -- ESP preview -----------------------------------------------------
    local espChar, _ = buildCharacter(espBody, false)
    local espOverlay = new("Frame", {
        Name = "Overlay", Position = UDim2.fromOffset(CHAR_X, CHAR_Y), Size = UDim2.fromOffset(CHAR_W, CHAR_H),
        BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 6, Active = false,
    }, espBody)

    local function drawEsp()
        local ESP = UI.esp
        if not ESP then return end
        for _, c in ipairs(espOverlay:GetChildren()) do c:Destroy() end
        local C = ESP.colors
        local bx0, by0, bx1, by1 = 12, 6, 138, 208
        local mid = 75

        if ESP.snapline then
            line(espOverlay, mid, by1 + 18, mid, PREVIEW_H - 44 - CHAR_Y - 8, C.snapline, 1.5, 5)
        end

        if ESP.box then
            local function rectOutline(x0, y0, x1, y1, col)
                local f = new("Frame", {
                    Position = UDim2.fromOffset(x0, y0), Size = UDim2.fromOffset(x1 - x0, y1 - y0),
                    BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 7, Active = false,
                }, espOverlay)
                stroke(f, col, 1.5, 0)
                return f
            end
            if ESP.type == "Corner" then
                local len = 24
                local col = C.box
                local function cl(x, y, dx, dy)
                    line(espOverlay, x, y, x + dx * len, y, col, 1.5, 7)
                    line(espOverlay, x, y, x, y + dy * len, col, 1.5, 7)
                end
                cl(bx0, by0, 1, 1)
                cl(bx1, by0, -1, 1)
                cl(bx0, by1, 1, -1)
                cl(bx1, by1, -1, -1)
            elseif ESP.type == "3D" then
                local ox, oy = 14, -10
                rectOutline(bx0 + ox, by0 + oy, bx1 + ox, by1 + oy, C.box)
                rectOutline(bx0, by0, bx1, by1, C.box)
                line(espOverlay, bx0, by0, bx0 + ox, by0 + oy, C.box, 1.5, 7)
                line(espOverlay, bx1, by0, bx1 + ox, by0 + oy, C.box, 1.5, 7)
                line(espOverlay, bx0, by1, bx0 + ox, by1 + oy, C.box, 1.5, 7)
                line(espOverlay, bx1, by1, bx1 + ox, by1 + oy, C.box, 1.5, 7)
            else
                rectOutline(bx0, by0, bx1, by1, C.box)
            end
            if ESP.filled then
                new("Frame", {
                    Position = UDim2.fromOffset(bx0, by0), Size = UDim2.fromOffset(bx1 - bx0, by1 - by0),
                    BackgroundColor3 = C.box, BackgroundTransparency = 0.82, BorderSizePixel = 0,
                    ZIndex = 5, Active = false,
                }, espOverlay)
            end
        end

        if ESP.health then
            local pct = 0.72
            new("Frame", {
                Position = UDim2.fromOffset(bx0 - 8, by0), Size = UDim2.fromOffset(3, by1 - by0),
                BackgroundColor3 = BLACK, BackgroundTransparency = 0.35, BorderSizePixel = 0, ZIndex = 7, Active = false,
            }, espOverlay)
            new("Frame", {
                AnchorPoint = Vector2.new(0, 1),
                Position = UDim2.fromOffset(bx0 - 8, by1), Size = UDim2.fromOffset(3, (by1 - by0) * pct),
                BackgroundColor3 = Color3.fromHSV(pct * 0.33, 1, 1), BorderSizePixel = 0, ZIndex = 8, Active = false,
            }, espOverlay)
        end

        if ESP.bone then
            local col = C.bone
            line(espOverlay, 29, 66, 121, 66, col, 1.5, 8)
            line(espOverlay, 29, 66, 29, 106, col, 1.5, 8)
            line(espOverlay, 121, 66, 121, 106, col, 1.5, 8)
            line(espOverlay, mid, 30, mid, 118, col, 1.5, 8)
            line(espOverlay, mid, 118, 58, 128, col, 1.5, 8)
            line(espOverlay, mid, 118, 92, 128, col, 1.5, 8)
            line(espOverlay, 58, 128, 58, 190, col, 1.5, 8)
            line(espOverlay, 92, 128, 92, 190, col, 1.5, 8)
        end

        if ESP.name then
            local l = UI.label(espOverlay, LocalPlayer.DisplayName, -20, by0 - 18, CHAR_W + 40, 14, 12, C.name, GB, CENTER)
            l.ZIndex = 8
        end
        if ESP.distance then
            local l = UI.label(espOverlay, "42 studs", -20, by1 + 2, CHAR_W + 40, 14, 11, C.distance, GM, CENTER)
            l.ZIndex = 8
        end
    end

    -- Aim preview (click the body parts aim assist may lock onto) ------
    local aimChar, aimParts = buildCharacter(aimBody, true)
    local aimCaption = UI.label(aimBody, "", 12, CHAR_Y + CHAR_H + 20, PREVIEW_W - 24, 34, 11, T.sub, GM, CENTER)
    aimCaption.TextWrapped = true
    aimCaption.ZIndex = 4
    local aimHint = UI.label(aimBody, "Click a body part to allow / block it", 12, 8, PREVIEW_W - 24, 30, 10, T.dim, GR, CENTER)
    aimHint.TextWrapped = true
    aimHint.ZIndex = 4
    local AIM_NICE = {
        Head = "Head", Torso = "Torso", RightArm = "Right Arm",
        LeftArm = "Left Arm", RightLeg = "Right Leg", LeftLeg = "Left Leg",
    }
    UI.aimHook = UI.aimHook or {}

    local function drawAim()
        local hook = UI.aimHook
        if not hook.isOn then return end
        local names = {}
        for _, name in ipairs(PART_ORDER) do
            local on = hook.isOn(name)
            tween(aimParts[name], 0.12, { BackgroundColor3 = on and Color3.fromRGB(226, 228, 232) or BODY_COLOR })
            if on then names[#names + 1] = AIM_NICE[name] end
        end
        aimCaption.Text = #names > 0 and ("Targeting: " .. table.concat(names, ", ")) or "Targeting: nothing selected"
    end
    for _, name in ipairs(PART_ORDER) do
        local b = aimParts[name]
        b.MouseEnter:Connect(function()
            if UI.aimHook.isOn and not UI.aimHook.isOn(name) then
                tween(b, 0.1, { BackgroundColor3 = Color3.fromRGB(70, 72, 80) })
            end
        end)
        b.MouseLeave:Connect(function()
            drawAim()
        end)
        b.MouseButton1Click:Connect(function()
            if dead() then return end
            if UI.aimHook.toggle then UI.aimHook.toggle(name) end
            drawAim()
        end)
    end
    UI.drawAimPreview = drawAim

    function shell.setPreviewMode(mode)
        if mode == "esp" then
            preview.Visible = true
            espBody.Visible, aimBody.Visible = true, false
            previewTitle.Text = "ESP Preview"
            drawEsp()
        elseif mode == "aim" then
            preview.Visible = true
            espBody.Visible, aimBody.Visible = false, true
            previewTitle.Text = "Parts"
            drawAim()
        else
            preview.Visible = false
        end
    end
    UI.refreshPreview = function()
        if preview.Visible then
            if espBody.Visible then drawEsp() end
            if aimBody.Visible then drawAim() end
        end
    end

    -- ---------------------------------------------------------------
    -- Tabs / pages
    -- Sidebar: grouped tabs (COMBAT / MISC / SETTINGS). Each tab has sub-tabs along its top,
    -- and each sub-tab is its own scrolling page.
    -- ---------------------------------------------------------------
    local tabs, tabList, currentTab = {}, {}, nil
    local ICON_IDLE = T.sub
    local ICON_ACTIVE = DARK
    local NAV_TOP, NAV_STEP, NAV_GROUP_H = 66, 42, 20
    local navY = NAV_TOP
    local SUB_BAR_H, SUB_TOP = 28, 36

    -- start the next group so that `count` tabs end just above the profile card
    function shell.pinNavToBottom(count)
        navY = WIN_H - 62 - NAV_GROUP_H - count * NAV_STEP
    end

    -- sidebar section heading (COMBAT / MISC / SETTINGS)
    function shell.addGroup(label)
        local l = UI.label(win, label, 16, navY, SIDEBAR_W - 32, 14, 9, T.dim, GB)
        l.ZIndex = 3
        navY = navY + NAV_GROUP_H
    end

    function shell.selectSub(tabName, subName)
        local t = tabs[tabName]
        if not t or not subName or not t.subs[subName] then return end
        W.closePopup()
        t.activeSub = subName
        for n, s in pairs(t.subs) do
            local on = (n == subName)
            s.sf.Visible = on
            tween(s.btn, 0.15, { BackgroundTransparency = on and 0.88 or 1 })
            s.btn.TextColor3 = on and T.text or T.dim
            s.btn.Font = on and GB or GM
        end
    end

    function shell.select(name, subName)
        local t = tabs[name]
        if not t then return end
        W.closePopup()
        currentTab = name
        for n, tab in pairs(tabs) do
            local on = (n == name)
            tab.container.Visible = on
            tween(tab.btn, 0.15, { BackgroundTransparency = on and 0.06 or 1 })
            tab.label.TextColor3 = on and DARK or ICON_IDLE
            tab.label.Font = on and GB or GM
            tab.icon.set(on and ICON_ACTIVE or ICON_IDLE)
        end
        shell.setPreviewMode(t.preview)
        shell.selectSub(name, subName or t.activeSub or t.firstSub)
    end

    function shell.addTab(name, iconKind, previewMode)
        local index = #tabList + 1
        local btn = new("TextButton", {
            Name = "Tab_" .. name, Position = UDim2.fromOffset(12, navY),
            Size = UDim2.fromOffset(SIDEBAR_W - 24, 34), BackgroundColor3 = T.accent,
            BackgroundTransparency = 1, Text = "", AutoButtonColor = false, BorderSizePixel = 0, ZIndex = 3,
        }, win)
        navY = navY + NAV_STEP
        corner(btn, 10)
        local ic = mkIcon(btn, iconKind, 10, 8)
        ic.set(ICON_IDLE)
        local lbl = UI.label(btn, name, 36, 0, SIDEBAR_W - 24 - 40, 34, 12, ICON_IDLE, GM)
        lbl.ZIndex = 4

        local container = new("Frame", {
            Name = "Container_" .. name, Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1,
            BorderSizePixel = 0, Visible = false, ZIndex = 2,
        }, contentFade)
        local subBar = new("Frame", {
            Name = "SubBar", Size = UDim2.new(1, 0, 0, SUB_BAR_H), BackgroundTransparency = 1,
            BorderSizePixel = 0, ZIndex = 4,
        }, container)
        new("Frame", {
            Position = UDim2.fromOffset(0, SUB_BAR_H + 4), Size = UDim2.new(1, 0, 0, 1),
            BackgroundColor3 = WHITE, BackgroundTransparency = 0.9, BorderSizePixel = 0, ZIndex = 3, Active = false,
        }, container)

        local tab = {
            btn = btn, label = lbl, icon = ic, container = container, subBar = subBar,
            subs = {}, subX = 0, preview = previewMode, activeSub = nil, firstSub = nil,
        }
        tabs[name] = tab
        tabList[index] = name
        btn.MouseButton1Click:Connect(function()
            if dead() then return end
            shell.select(name)
        end)
        btn.MouseEnter:Connect(function()
            if currentTab ~= name then tween(btn, 0.1, { BackgroundTransparency = 0.9 }) end
        end)
        btn.MouseLeave:Connect(function()
            if currentTab ~= name then tween(btn, 0.1, { BackgroundTransparency = 1 }) end
        end)
        return tab
    end

    -- adds a sub-tab to a tab (shown in the bar across the top) and returns its page
    function shell.addSub(tabName, subName, noScroll)
        local t = tabs[tabName]
        if not t then error("unknown tab: " .. tostring(tabName)) end
        local w = UI.textWidth(subName, 12, GM) + 26
        local b = new("TextButton", {
            Name = "Sub_" .. subName, Position = UDim2.fromOffset(t.subX, 0), Size = UDim2.fromOffset(w, SUB_BAR_H),
            BackgroundColor3 = WHITE, BackgroundTransparency = 1, Text = subName, TextColor3 = T.dim,
            TextSize = 12, Font = GM, AutoButtonColor = false, BorderSizePixel = 0, ZIndex = 4,
        }, t.subBar)
        corner(b, 13)
        t.subX = t.subX + w + 4

        local sf = new("ScrollingFrame", {
            Name = "Page_" .. tabName .. "_" .. subName, Position = UDim2.fromOffset(0, SUB_TOP),
            Size = UDim2.new(1, 0, 1, -SUB_TOP), BackgroundTransparency = 1, BorderSizePixel = 0,
            ScrollBarThickness = 3, ScrollBarImageColor3 = WHITE, ScrollBarImageTransparency = 0.7,
            CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y,
            ScrollingDirection = Enum.ScrollingDirection.Y, Visible = false, Active = true,
            ElasticBehavior = Enum.ElasticBehavior.Never, ZIndex = 2,
        }, t.container)
        new("UIListLayout", { Padding = UDim.new(0, 6), SortOrder = Enum.SortOrder.LayoutOrder }, sf)
        new("UIPadding", {
            PaddingTop = UDim.new(0, 6), PaddingBottom = UDim.new(0, noScroll and 0 or 8),
            PaddingLeft = UDim.new(0, 6), PaddingRight = UDim.new(0, noScroll and 6 or 12),
        }, sf)
        if noScroll then
            -- fixed page: never scrolls, no scrollbar, no extra canvas space under the last row
            sf.AutomaticCanvasSize = Enum.AutomaticSize.None
            sf.CanvasSize = UDim2.new()
            sf.ScrollingEnabled = false
            sf.ScrollBarThickness = 0
            sf.ScrollBarImageTransparency = 1
        end

        t.subs[subName] = { btn = b, sf = sf }
        if not t.firstSub then t.firstSub = subName end
        b.MouseButton1Click:Connect(function()
            if dead() then return end
            shell.selectSub(tabName, subName)
        end)
        return { sf = sf, order = 0 }
    end

    -- ---------------------------------------------------------------
    -- ---------------------------------------------------------------
    -- Widgets
    -- ---------------------------------------------------------------
    local function row(page, h)
        page.order = page.order + 1
        return new("Frame", {
            Size = UDim2.new(1, 0, 0, h), BackgroundTransparency = 1, BorderSizePixel = 0,
            LayoutOrder = page.order, ZIndex = 2,
        }, page.sf)
    end
    W.row = row

    -- bottom-right notification box
    UI._notifyToken = 0 -- kept on UI (not a local) to stay under Lua's local-variable limit
    function UI.notify(title, text)
        UI._notifyToken = UI._notifyToken + 1
        local mine = UI._notifyToken
        local old = whitelistGui:FindFirstChild("ArmadaNotify")
        if old then old:Destroy() end
        local box, entry = makeGlass(whitelistGui, "ArmadaNotify", 280, 74, 0, 0)
        box.AnchorPoint = Vector2.new(1, 1)
        box.Position = UDim2.new(1, 300, 1, -20)
        box.Active = false
        box.ZIndex = 200
        entry.sheen.ZIndex = 201
        shell.applyGlass() -- match the current opacity slider
        local bar = new("Frame", {
            Position = UDim2.fromOffset(10, 12), Size = UDim2.fromOffset(3, 50),
            BackgroundColor3 = Color3.fromRGB(235, 90, 90), BorderSizePixel = 0, ZIndex = 202, Active = false,
        }, box)
        corner(bar, 2)
        local t = UI.label(box, title, 22, 8, 248, 18, 12, T.text, GB)
        t.ZIndex = 202
        local m = UI.label(box, text, 22, 28, 248, 40, 11, T.sub, GM)
        m.ZIndex = 202
        m.TextWrapped = true
        m.TextYAlignment = Enum.TextYAlignment.Top
        box.Destroying:Connect(function()
            for i, p in ipairs(panels) do
                if p == entry then table.remove(panels, i) break end
            end
        end)
        tween(box, 0.25, { Position = UDim2.new(1, -20, 1, -20) })
        task.delay(4, function()
            if mine ~= UI._notifyToken or not box.Parent then return end
            tween(box, 0.25, { Position = UDim2.new(1, 300, 1, -20) })
            task.delay(0.3, function()
                if box.Parent and mine == UI._notifyToken then box:Destroy() end
            end)
        end)
    end

    local openPopup = nil
    function W.closePopup()
        if openPopup then
            openPopup:Destroy()
            openPopup = nil
        end
    end

    local function toLocal(absPos)
        local sc = cardScale.Scale
        local origin = popupLayer.AbsolutePosition
        return Vector2.new((absPos.X - origin.X) / sc, (absPos.Y - origin.Y) / sc)
    end

    local activeDrag = nil
    UserInputService.InputChanged:Connect(function(input)
        if activeDrag and (input.UserInputType == Enum.UserInputType.MouseMovement
            or input.UserInputType == Enum.UserInputType.Touch) then
            activeDrag(pointerOf(input))
        end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then
            activeDrag = nil
        end
    end)
    local function beginDrag(fn, input)
        activeDrag = fn
        fn(pointerOf(input))
    end

    local function isPress(input)
        return input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch
    end

    -- Dark outlined "pill" behind a text label (same look as the status boxes) so text is easier
    -- to read over the glass background. Works for dynamic text too (the pill auto-sizes in X).
    function W.pill(lbl)
        return lbl -- text pills were replaced by full-row dark cards (see W.card)
    end

    -- Card backgrounds behind rows / headers / preset buttons. Fixed to the Subtle look.
    UI.cardStyles = {
        Subtle  = { color = Color3.fromRGB(255, 255, 255), row = 0.95, section = 0.91, stroke = 0.88 },
    }
    UI.cardStyle = "Subtle"
    UI._cards = {}
    function UI.styleCard(entry)
        local st = UI.cardStyles[UI.cardStyle]
        entry.inst.BackgroundColor3 = st.color
        entry.inst.BackgroundTransparency = (entry.kind == "section") and st.section or st.row
        if entry.stroke then entry.stroke.Transparency = st.stroke end
    end
    function UI.registerCard(inst, kind, strokeObj)
        local entry = { inst = inst, kind = kind, stroke = strokeObj }
        UI._cards[#UI._cards + 1] = entry
        UI.styleCard(entry)
        return entry
    end
    function UI.applyCardStyle(name)
        UI.cardStyle = "Subtle" -- locked: the row background can't be changed
        for _, entry in ipairs(UI._cards) do
            if entry.inst.Parent then UI.styleCard(entry) end
        end
    end

    -- Rounded card behind a whole row. Adds side/top padding so the row's contents sit inside it.
    function W.card(r, extra)
        extra = extra or 8
        r.Size = UDim2.new(1, 0, 0, r.Size.Y.Offset + extra)
        corner(r, 10)
        local st = stroke(r, WHITE, 1, 0.9)
        new("UIPadding", {
            PaddingLeft = UDim.new(0, 12), PaddingRight = UDim.new(0, 12),
            PaddingTop = UDim.new(0, math.floor(extra / 2)), PaddingBottom = UDim.new(0, math.floor(extra / 2)),
        }, r)
        UI.registerCard(r, "row", st)
        return r
    end

    function W.section(page, title)
        local r = row(page, 36)
        W.card(r, 0)
        UI._cards[#UI._cards].kind = "section"
        UI.styleCard(UI._cards[#UI._cards])
        local l = UI.label(r, title, 0, 0, 400, 36, 13, WHITE, GB)
        l.ZIndex = 3
        return r
    end

    function W.caption(page, text, h)
        local r = row(page, h or 20)
        local l = UI.label(r, text, 0, 0, 100, h or 20, 10, T.dim, GR)
        l.Size = UDim2.new(1, 0, 1, 0)
        l.TextWrapped = true
        l.TextYAlignment = Enum.TextYAlignment.Top
        l.ZIndex = 3
        corner(l, 10)
        UI.registerCard(l, "row", stroke(l, WHITE, 1, 0.9))
        new("UIPadding", { PaddingLeft = UDim.new(0, 12), PaddingRight = UDim.new(0, 12), PaddingTop = UDim.new(0, 5), PaddingBottom = UDim.new(0, 5) }, l)
        return l
    end

    function W.toggle(page, text, get, set, withSwatch)
        local r = row(page, 26)
        W.card(r, 8)
        local track = new("TextButton", {
            Position = UDim2.fromOffset(0, 3), Size = UDim2.fromOffset(36, 20), BackgroundColor3 = Color3.fromRGB(58, 59, 65),
            Text = "", AutoButtonColor = false, BorderSizePixel = 0, ZIndex = 3,
        }, r)
        corner(track, 10)
        local knob = new("Frame", {
            AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.new(0, 3, 0.5, 0), Size = UDim2.fromOffset(14, 14),
            BackgroundColor3 = Color3.fromRGB(238, 240, 244), BorderSizePixel = 0, ZIndex = 4,
        }, track)
        corner(knob, 7)
        local lbl = UI.label(r, text, 46, 0, 300, 26, 12, T.text, GM)
        lbl.ZIndex = 3
        W.pill(lbl, 24)
        local lastOn = nil
        local function paint(instant)
            local on = get() and true or false
            if instant and lastOn == on then return end
            lastOn = on
            local trackColor = on and Color3.fromRGB(178, 180, 187) or Color3.fromRGB(58, 59, 65)
            local knobColor = on and Color3.fromRGB(20, 20, 23) or Color3.fromRGB(238, 240, 244)
            local knobPos = on and UDim2.new(1, -17, 0.5, 0) or UDim2.new(0, 3, 0.5, 0)
            if instant then
                track.BackgroundColor3 = trackColor
                knob.BackgroundColor3 = knobColor
                knob.Position = knobPos
            else
                tween(track, 0.15, { BackgroundColor3 = trackColor })
                tween(knob, 0.15, { BackgroundColor3 = knobColor, Position = knobPos })
            end
        end
        local function flip()
            if dead() then return end
            set(not get())
            paint()
            if UI.refreshPreview then UI.refreshPreview() end
        end
        track.MouseButton1Click:Connect(flip)
        local labelHit = new("TextButton", {
            Position = UDim2.fromOffset(46, 0), Size = UDim2.new(1, -90, 1, 0), BackgroundTransparency = 1,
            Text = "", AutoButtonColor = false, ZIndex = 5,
        }, r)
        labelHit.MouseButton1Click:Connect(flip)
        reg(function() paint(true) end)
        paint(true)
        return r
    end

    function W.slider(page, text, min, max, step, get, set, fmt)
        local r = row(page, 44)
        W.card(r, 8)
        local lbl = UI.label(r, text, 0, 2, 260, 16, 12, T.text, GM)
        lbl.ZIndex = 3
        W.pill(lbl, 20)
        local val = UI.label(r, "", 0, 2, 140, 16, 11, T.dim, GR, RIGHT)
        W.pill(val, 20)
        val.AnchorPoint = Vector2.new(1, 0)
        val.Position = UDim2.new(1, 0, 0, 2)
        val.ZIndex = 3
        local track = new("Frame", {
            Position = UDim2.fromOffset(14, 25), Size = UDim2.new(1, -28, 0, 9),
            BackgroundColor3 = Color3.fromRGB(24, 24, 27), BackgroundTransparency = 0.1, BorderSizePixel = 0, ZIndex = 3,
        }, r)
        corner(track, 5)
        local fill = new("Frame", {
            Size = UDim2.fromScale(0, 1), BackgroundColor3 = Color3.fromRGB(208, 210, 215), BorderSizePixel = 0, ZIndex = 4,
        }, track)
        corner(fill, 5)
        local knob = new("Frame", {
            AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0, 0, 0.5, 0), Size = UDim2.fromOffset(14, 14),
            BackgroundColor3 = WHITE, BorderSizePixel = 0, ZIndex = 5,
        }, track)
        corner(knob, 7)
        local hit = new("TextButton", {
            Position = UDim2.fromOffset(0, 16), Size = UDim2.new(1, 0, 0, 28), BackgroundTransparency = 1,
            Text = "", AutoButtonColor = false, ZIndex = 6,
        }, r)
        local lastV = nil
        local function paint()
            local v = get()
            if v == lastV then return end
            lastV = v
            local a = math.clamp((v - min) / (max - min), 0, 1)
            fill.Size = UDim2.fromScale(a, 1)
            knob.Position = UDim2.new(a, 0, 0.5, 0)
            val.Text = fmt and fmt(v) or tostring(v)
        end
        local function fromPointer(pos)
            if dead() then return end
            local a = math.clamp((pos.X - track.AbsolutePosition.X) / math.max(track.AbsoluteSize.X, 1), 0, 1)
            local v = min + a * (max - min)
            v = math.floor(v / step + 0.5) * step
            v = math.clamp(v, min, max)
            set(v)
            paint()
        end
        hit.InputBegan:Connect(function(input)
            if isPress(input) and not dead() then
                beginDrag(fromPointer, input)
            end
        end)
        reg(paint)
        paint()
        return r
    end

    -- numeric text box: type a value, then press Enter or click away to apply (clamped to min/max)
    function W.numberInput(page, text, min, max, step, get, set)
        local r = row(page, 32)
        W.card(r, 6)
        local l = UI.label(r, text, 0, 0, 240, 32, 12, T.text, GM)
        l.ZIndex = 3
        W.pill(l, 24)
        local box = new("TextBox", {
            AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, 0, 0.5, 0), Size = UDim2.fromOffset(84, 26),
            BackgroundColor3 = BLACK, BackgroundTransparency = 0.72, Text = "", ClearTextOnFocus = false,
            TextColor3 = T.text, Font = GM, TextSize = 12, TextXAlignment = CENTER,
            BorderSizePixel = 0, ZIndex = 4,
        }, r)
        corner(box, 9)
        stroke(box, WHITE, 1, 0.72)
        local function paint()
            if UserInputService:GetFocusedTextBox() ~= box then
                box.Text = tostring(get())
            end
        end
        box.FocusLost:Connect(function()
            local v = tonumber(box.Text)
            if v and not dead() then
                v = math.floor(math.clamp(v, min, max) / step + 0.5) * step
                set(math.clamp(v, min, max))
            end
            paint()
            if UI.refreshPreview then UI.refreshPreview() end
        end)
        reg(paint)
        paint()
        return r
    end

    -- Range bar: draggable min/max handles on a 0..max track with ticks, a gradient fill,
    -- a live readout and a plain-English summary. Also follows the number boxes as you type.
    function W.rangeBar(page, title, getLo, getHi, setLo, setHi, max, unit)
        local r = row(page, 96)
        W.card(r, 8)
        local heading = UI.label(r, title, 0, 2, 260, 16, 11, WHITE, GB)
        heading.ZIndex = 3
        W.pill(heading, 20)
        local readout = UI.label(r, "", 0, 2, 220, 16, 12, T.text, GM, RIGHT)
        readout.AnchorPoint = Vector2.new(1, 0)
        readout.Position = UDim2.new(1, 0, 0, 2)
        readout.ZIndex = 3
        W.pill(readout, 20)

        local track = new("Frame", {
            Position = UDim2.fromOffset(8, 30), Size = UDim2.new(1, -16, 0, 12),
            BackgroundColor3 = Color3.fromRGB(24, 24, 27), BackgroundTransparency = 0.1, BorderSizePixel = 0, ZIndex = 3,
        }, r)
        corner(track, 6)
        stroke(track, WHITE, 1, 0.82)

        local fill = new("Frame", {
            Size = UDim2.fromScale(0.01, 1), BackgroundColor3 = WHITE, BorderSizePixel = 0, ZIndex = 4,
        }, track)
        corner(fill, 6)
        new("UIGradient", {
            Color = ColorSequence.new(Color3.fromRGB(150, 153, 162), Color3.fromRGB(240, 242, 246)),
        }, fill)

        -- tick marks + labels at 0 / 25 / 50 / 75 / 100 %
        for i = 0, 4 do
            local f = i / 4
            local tick = new("Frame", {
                AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(f, 0, 1, 4), Size = UDim2.fromOffset(1, 5),
                BackgroundColor3 = WHITE, BackgroundTransparency = 0.7, BorderSizePixel = 0, ZIndex = 3,
            }, track)
            local txt = tostring(math.floor(max * f + 0.5))
            if i == 4 then txt = txt .. " " .. unit end
            local tl = UI.label(r, txt, 0, 0, 70, 12, 9, T.dim, GR, CENTER)
            tl.AnchorPoint = Vector2.new(0.5, 0)
            tl.Position = UDim2.new(f, f == 0 and 8 or (f == 1 and -8 or 0), 0, 56)
            tl.ZIndex = 3
        end

        local function makeKnob()
            local k = new("Frame", {
                AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0, 0, 0.5, 0), Size = UDim2.fromOffset(16, 16),
                BackgroundColor3 = WHITE, BorderSizePixel = 0, ZIndex = 6,
            }, track)
            corner(k, 8)
            stroke(k, BLACK, 1, 0.6)
            return k
        end
        local knobLo, knobHi = makeKnob(), makeKnob()

        local summary = UI.label(r, "", 0, 74, 400, 16, 11, T.sub, GM)
        summary.ZIndex = 3
        W.pill(summary, 20)

        local lastLo, lastHi = nil, nil
        local function paint()
            local lo, hi = getLo(), getHi()
            if lo == lastLo and hi == lastHi then return end
            lastLo, lastHi = lo, hi
            if lo > hi then lo, hi = hi, lo end
            local a0, b0 = math.clamp(lo / max, 0, 1), math.clamp(hi / max, 0, 1)
            fill.Position = UDim2.new(a0, 0, 0, 0)
            fill.Size = UDim2.new(math.max(b0 - a0, 0.008), 0, 1, 0)
            knobLo.Position = UDim2.new(a0, 0, 0.5, 0)
            knobHi.Position = UDim2.new(b0, 0, 0.5, 0)
            knobHi.Visible = hi ~= lo
            if hi == lo then
                readout.Text = string.format("%d %s", lo, unit)
                summary.Text = lo == 0 and "No delay: shoots instantly" or string.format("Fixed delay of %d %s every shot", lo, unit)
            else
                readout.Text = string.format("%d - %d %s", lo, hi, unit)
                summary.Text = string.format("Random delay between %d and %d %s (avg about %d)", lo, hi, unit, math.floor((lo + hi) / 2 + 0.5))
            end
        end

        local function fromPointer(pos, which)
            if dead() then return end
            local a0 = math.clamp((pos.X - track.AbsolutePosition.X) / math.max(track.AbsoluteSize.X, 1), 0, 1)
            local v = math.floor(a0 * max + 0.5)
            if which == "lo" then
                setLo(math.min(v, max))
            else
                setHi(math.min(v, max))
            end
            paint()
            if UI.syncWidgets then UI.syncWidgets() end
        end

        local hit = new("TextButton", {
            Position = UDim2.fromOffset(0, 22), Size = UDim2.new(1, 0, 0, 30), BackgroundTransparency = 1,
            Text = "", AutoButtonColor = false, ZIndex = 7,
        }, r)
        hit.InputBegan:Connect(function(input)
            if isPress(input) and not dead() then
                -- grab whichever handle is closer to where you pressed
                local px = pointerOf(input).X
                local dLo = math.abs(px - (knobLo.AbsolutePosition.X + 8))
                local dHi = math.abs(px - (knobHi.AbsolutePosition.X + 8))
                local which = (dLo <= dHi) and "lo" or "hi"
                if getLo() == getHi() then
                    -- handles overlap: pick by side of the press
                    which = (px < knobLo.AbsolutePosition.X + 8) and "lo" or "hi"
                end
                beginDrag(function(pos) fromPointer(pos, which) end, input)
            end
        end)

        reg(paint)
        paint()
        return r
    end

    function W.segmented(page, label, options, get, set)
        local r = row(page, label and 52 or 32)
        W.card(r, 8)
        if label then
            local l = UI.label(r, label, 0, 2, 240, 16, 11, T.dim, GM)
            l.ZIndex = 3
            W.pill(l, 20)
        end
        local y0 = label and 22 or 3
        local x = 0
        local btns = {}
        for _, opt in ipairs(options) do
            local id, txt = opt, opt
            if type(opt) == "table" then id, txt = opt[1], opt[2] end
            local w = UI.textWidth(txt, 12, GM) + 28
            local b = new("TextButton", {
                Position = UDim2.fromOffset(x, y0), Size = UDim2.fromOffset(w, 26), BackgroundColor3 = Color3.fromRGB(226, 228, 232),
                BackgroundTransparency = 1, Text = txt, TextColor3 = T.sub, TextSize = 12, Font = GM,
                AutoButtonColor = false, BorderSizePixel = 0, ZIndex = 4,
            }, r)
            corner(b, 13)
            local st = stroke(b, WHITE, 1, 0.72)
            btns[#btns + 1] = { b = b, id = id, st = st }
            b.MouseButton1Click:Connect(function()
                if dead() then return end
                set(id)
                for _, e in ipairs(btns) do
                    local sel = (get() == e.id)
                    tween(e.b, 0.12, { BackgroundTransparency = sel and 0.06 or 1 })
                    e.b.TextColor3 = sel and DARK or T.sub
                    e.st.Transparency = sel and 1 or 0.72
                end
                if UI.refreshPreview then UI.refreshPreview() end
            end)
            x = x + w + 8
        end
        local lastSel = {}
        local function paint()
            local current = get()
            if current == lastSel[1] and lastSel[2] then return end
            lastSel[1], lastSel[2] = current, true
            for _, e in ipairs(btns) do
                local sel = (get() == e.id)
                e.b.BackgroundTransparency = sel and 0.06 or 1
                e.b.TextColor3 = sel and DARK or T.sub
                e.st.Transparency = sel and 1 or 0.72
            end
        end
        reg(paint)
        paint()
        return r
    end

    local function chevron(parent)
        local c = new("Frame", {
            AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -8, 0.5, 0), Size = UDim2.fromOffset(12, 12),
            BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 5, Active = false,
        }, parent)
        local a = new("Frame", {
            AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, -2.6, 0.5, 0.5), Size = UDim2.fromOffset(6.5, 1.5),
            Rotation = 45, BackgroundColor3 = T.sub, BorderSizePixel = 0, ZIndex = 5,
        }, c)
        local b = new("Frame", {
            AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, 2.6, 0.5, 0.5), Size = UDim2.fromOffset(6.5, 1.5),
            Rotation = -45, BackgroundColor3 = T.sub, BorderSizePixel = 0, ZIndex = 5,
        }, c)
    end

    local function pill(parent, w, h)
        local b = new("TextButton", {
            AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, 0, 0.5, 0), Size = UDim2.fromOffset(w, h),
            BackgroundColor3 = WHITE, BackgroundTransparency = 0.92, Text = "", AutoButtonColor = false,
            BorderSizePixel = 0, ZIndex = 4,
        }, parent)
        corner(b, 9)
        local st = stroke(b, WHITE, 1, 0.72)
        return b, st
    end

    function W.dropdown(page, text, options, get, set)
        local r = row(page, 32)
        W.card(r, 6)
        local l = UI.label(r, text, 0, 0, 240, 32, 12, T.text, GM)
        l.ZIndex = 3
        W.pill(l, 24)
        local btn = pill(r, 132, 26)
        local cur = UI.label(btn, "", 10, 0, 100, 26, 12, T.text, GM)
        cur.ZIndex = 5
        chevron(btn)
        local function paint() cur.Text = tostring(get()) end
        btn.MouseButton1Click:Connect(function()
            if dead() then return end
            if openPopup then W.closePopup() return end
            local abs, sz = btn.AbsolutePosition, btn.AbsoluteSize
            local lp = toLocal(Vector2.new(abs.X, abs.Y + sz.Y + 4))
            local blocker = new("TextButton", {
                Name = "PopupBlocker", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1,
                Text = "", AutoButtonColor = false, ZIndex = 1,
            }, popupLayer)
            openPopup = blocker
            local menu = new("Frame", {
                Position = UDim2.fromOffset(lp.X, lp.Y), Size = UDim2.fromOffset(sz.X / cardScale.Scale, #options * 28 + 8),
                BackgroundColor3 = Color3.fromRGB(34, 35, 40), BackgroundTransparency = 0.04, BorderSizePixel = 0, ZIndex = 2,
            }, blocker)
            corner(menu, 10)
            stroke(menu, WHITE, 1, 0.8)
            for i, opt in ipairs(options) do
                local o = new("TextButton", {
                    Position = UDim2.fromOffset(4, 4 + (i - 1) * 28), Size = UDim2.new(1, -8, 0, 26),
                    BackgroundColor3 = WHITE, BackgroundTransparency = (get() == opt) and 0.88 or 1,
                    Text = tostring(opt), TextColor3 = T.text, TextSize = 12, Font = GM, AutoButtonColor = false,
                    BorderSizePixel = 0, ZIndex = 3,
                }, menu)
                corner(o, 7)
                o.MouseEnter:Connect(function() tween(o, 0.08, { BackgroundTransparency = 0.88 }) end)
                o.MouseLeave:Connect(function()
                    tween(o, 0.08, { BackgroundTransparency = (get() == opt) and 0.88 or 1 })
                end)
                o.MouseButton1Click:Connect(function()
                    if dead() then return end
                    set(opt)
                    paint()
                    W.closePopup()
                    if UI.refreshPreview then UI.refreshPreview() end
                end)
            end
            blocker.MouseButton1Click:Connect(W.closePopup)
        end)
        reg(paint)
        paint()
        return r
    end

    -- keybind row: returns the TextButton (caller wires clicks + text)
    function W.keybind(page, text, desc)
        local r = row(page, 30)
        W.card(r, 6)
        local l = UI.label(r, text, 0, 0, 260, 30, 12, T.text, GM)
        l.ZIndex = 3
        W.pill(l, 24)
        local btn, st = pill(r, 116, 26)
        btn.Text = "NONE"
        btn.TextColor3 = T.text
        btn.TextSize = 11
        btn.Font = GM
        btn:GetPropertyChangedSignal("Text"):Connect(function()
            local listening = (btn.Text == "PRESS A KEY...")
            tween(st, 0.12, { Transparency = listening and 0.1 or 0.72 })
        end)
        return btn, r
    end

    function W.button(page, text, kind, cb)
        local r = row(page, 34)
        local b = new("TextButton", {
            Position = UDim2.fromOffset(0, 3), Size = UDim2.new(1, 0, 0, 28), BackgroundColor3 = WHITE,
            BackgroundTransparency = 0.92, Text = text, TextColor3 = kind == "danger" and T.bad or T.text,
            TextSize = 12, Font = GM, AutoButtonColor = false, BorderSizePixel = 0, ZIndex = 4,
        }, r)
        corner(b, 10)
        stroke(b, WHITE, 1, 0.72)
        b.MouseEnter:Connect(function() tween(b, 0.1, { BackgroundTransparency = 0.86 }) end)
        b.MouseLeave:Connect(function() tween(b, 0.1, { BackgroundTransparency = 0.92 }) end)
        if cb then b.MouseButton1Click:Connect(function() if not dead() then cb(b) end end) end
        return b
    end

    function W.buttons(page, defs)
        local r = row(page, 34)
        local n = #defs
        local out = {}
        for i, d in ipairs(defs) do
            local b = new("TextButton", {
                Position = UDim2.new((i - 1) / n, 0, 0, 3), Size = UDim2.new(1 / n, -6, 0, 28), BackgroundColor3 = WHITE,
                BackgroundTransparency = 0.92, Text = d[1], TextColor3 = d[2] == "danger" and T.bad or T.text,
                TextSize = 12, Font = GM, AutoButtonColor = false, BorderSizePixel = 0, ZIndex = 4,
            }, r)
            corner(b, 10)
            stroke(b, WHITE, 1, 0.72)
            b.MouseEnter:Connect(function() tween(b, 0.1, { BackgroundTransparency = 0.86 }) end)
            b.MouseLeave:Connect(function() tween(b, 0.1, { BackgroundTransparency = 0.92 }) end)
            out[i] = b
        end
        return out
    end

    function W.textbox(page, placeholder, h, multi)
        local r = row(page, h + 6)
        local tb = new("TextBox", {
            Position = UDim2.fromOffset(0, 3), Size = UDim2.new(1, 0, 0, h), BackgroundColor3 = BLACK,
            BackgroundTransparency = 0.72, Text = "", PlaceholderText = placeholder, PlaceholderColor3 = T.dim,
            TextColor3 = T.text, Font = GM, TextSize = 12, ClearTextOnFocus = false, BorderSizePixel = 0,
            TextXAlignment = LEFT, TextYAlignment = multi and Enum.TextYAlignment.Top or Enum.TextYAlignment.Center,
            MultiLine = multi and true or false, TextWrapped = multi and true or false, ZIndex = 4,
        }, r)
        corner(tb, 9)
        stroke(tb, WHITE, 1, 0.8)
        new("UIPadding", { PaddingLeft = UDim.new(0, 10), PaddingRight = UDim.new(0, 10), PaddingTop = UDim.new(0, multi and 8 or 0) }, tb)
        return tb
    end

    -- static label row (status text); returns the TextLabel
    function W.statusLabel(page, text)
        local r = row(page, 22)
        local l = UI.label(r, text, 0, 0, 100, 22, 11, T.sub, GM)
        l.Size = UDim2.new(1, 0, 1, 0)
        l.ZIndex = 3
        l.TextTruncate = Enum.TextTruncate.AtEnd
        return l
    end

    function W.liveRow(page, title)
        local r = row(page, 24)
        W.card(r, 8)
        local a = UI.label(r, title, 0, 0, 200, 24, 11, T.dim, GM)
        a.ZIndex = 3
        W.pill(a, 22)
        local v = UI.label(r, "-", 0, 0, 240, 24, 12, T.text, GB, RIGHT)
        v.AnchorPoint = Vector2.new(1, 0)
        v.Position = UDim2.new(1, 0, 0, 0)
        v.ZIndex = 3
        W.pill(v, 22)
        return v
    end

    -- colour swatch + picker --------------------------------------------
    function W.swatch(r, getColor, setColor)
        local sw = new("TextButton", {
            AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -2, 0.5, 0), Size = UDim2.fromOffset(18, 18),
            BackgroundColor3 = getColor(), Text = "", AutoButtonColor = false, BorderSizePixel = 0, ZIndex = 6,
        }, r)
        corner(sw, 9)
        stroke(sw, WHITE, 1, 0.55)
        reg(function() sw.BackgroundColor3 = getColor() end)
        sw.MouseButton1Click:Connect(function()
            if dead() then return end
            W.closePopup()
            local h, s, v = getColor():ToHSV()
            local abs = sw.AbsolutePosition
            local pw, ph = 188, 160
            local lp = toLocal(Vector2.new(abs.X, abs.Y + 24 * cardScale.Scale))
            local px = math.clamp(lp.X - pw + 18, 8, WIN_W - pw - 8)
            local py = math.clamp(lp.Y, 60, WIN_H - ph - 8)
            local blocker = new("TextButton", {
                Name = "PickerBlocker", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1,
                Text = "", AutoButtonColor = false, ZIndex = 1,
            }, popupLayer)
            openPopup = blocker
            blocker.MouseButton1Click:Connect(W.closePopup)
            local menu = new("Frame", {
                Position = UDim2.fromOffset(px, py), Size = UDim2.fromOffset(pw, ph),
                BackgroundColor3 = Color3.fromRGB(34, 35, 40), BackgroundTransparency = 0.04, BorderSizePixel = 0, ZIndex = 2,
            }, blocker)
            corner(menu, 12)
            stroke(menu, WHITE, 1, 0.8)
            local sv = new("TextButton", {
                Position = UDim2.fromOffset(12, 12), Size = UDim2.fromOffset(pw - 24, 104),
                BackgroundColor3 = Color3.fromHSV(h, 1, 1), Text = "", AutoButtonColor = false, BorderSizePixel = 0, ZIndex = 3,
            }, menu)
            corner(sv, 7)
            local wh = new("Frame", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = WHITE, BorderSizePixel = 0, ZIndex = 4, Active = false }, sv)
            corner(wh, 7)
            new("UIGradient", { Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 0), NumberSequenceKeypoint.new(1, 1) }) }, wh)
            local bk = new("Frame", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = BLACK, BorderSizePixel = 0, ZIndex = 5, Active = false }, sv)
            corner(bk, 7)
            new("UIGradient", { Rotation = 90, Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(1, 0) }) }, bk)
            local mk = new("Frame", {
                AnchorPoint = Vector2.new(0.5, 0.5), Size = UDim2.fromOffset(10, 10), BackgroundTransparency = 1,
                BorderSizePixel = 0, ZIndex = 6, Active = false,
            }, sv)
            corner(mk, 5)
            stroke(mk, WHITE, 2, 0)
            local hue = new("TextButton", {
                Position = UDim2.fromOffset(12, 130), Size = UDim2.fromOffset(pw - 24, 12), BackgroundColor3 = WHITE,
                Text = "", AutoButtonColor = false, BorderSizePixel = 0, ZIndex = 3,
            }, menu)
            corner(hue, 6)
            local seq = {}
            for i = 0, 6 do
                seq[#seq + 1] = ColorSequenceKeypoint.new(i / 6, Color3.fromHSV(math.min(i / 6, 0.999), 1, 1))
            end
            new("UIGradient", { Color = ColorSequence.new(seq) }, hue)
            local hmk = new("Frame", {
                AnchorPoint = Vector2.new(0.5, 0.5), Size = UDim2.fromOffset(6, 16), BackgroundColor3 = WHITE,
                BorderSizePixel = 0, ZIndex = 5, Active = false,
            }, hue)
            corner(hmk, 3)
            local function apply()
                local c = Color3.fromHSV(h, s, v)
                setColor(c)
                sw.BackgroundColor3 = c
                sv.BackgroundColor3 = Color3.fromHSV(h, 1, 1)
                mk.Position = UDim2.new(s, 0, 1 - v, 0)
                hmk.Position = UDim2.new(h, 0, 0.5, 0)
                if UI.refreshPreview then UI.refreshPreview() end
            end
            sv.InputBegan:Connect(function(input)
                if isPress(input) then
                    beginDrag(function(pos)
                        s = math.clamp((pos.X - sv.AbsolutePosition.X) / math.max(sv.AbsoluteSize.X, 1), 0, 1)
                        v = 1 - math.clamp((pos.Y - sv.AbsolutePosition.Y) / math.max(sv.AbsoluteSize.Y, 1), 0, 1)
                        apply()
                    end, input)
                end
            end)
            hue.InputBegan:Connect(function(input)
                if isPress(input) then
                    beginDrag(function(pos)
                        h = math.clamp((pos.X - hue.AbsolutePosition.X) / math.max(hue.AbsoluteSize.X, 1), 0, 0.999)
                        apply()
                    end, input)
                end
            end)
            apply()
        end)
        return sw
    end

    -- toggle row with a colour swatch on the right
    function W.toggleColor(page, text, get, set, getColor, setColor)
        local r = W.toggle(page, text, get, set)
        W.swatch(r, getColor, setColor)
        return r
    end

    -- player list (search + rows) ---------------------------------------
    function W.playerList(page, height, spec)
        local r = spec.host or row(page, height)
        local card = new("Frame", {
            Size = UDim2.fromScale(1, 1), BackgroundColor3 = WHITE, BackgroundTransparency = 0.95,
            BorderSizePixel = 0, ZIndex = 2,
        }, r)
        corner(card, 12)
        stroke(card, WHITE, 1, 0.85)
        local search = new("TextBox", {
            Position = UDim2.fromOffset(10, 8), Size = UDim2.new(1, -20, 0, 26), BackgroundColor3 = BLACK,
            BackgroundTransparency = 0.72, Text = "", PlaceholderText = "Search players...", PlaceholderColor3 = T.dim,
            TextColor3 = T.text, Font = GM, TextSize = 12, ClearTextOnFocus = false, BorderSizePixel = 0,
            TextXAlignment = LEFT, ZIndex = 4,
        }, card)
        corner(search, 8)
        new("UIPadding", { PaddingLeft = UDim.new(0, 10) }, search)
        local list = new("ScrollingFrame", {
            Position = UDim2.fromOffset(6, 42), Size = UDim2.new(1, -12, 1, -48), BackgroundTransparency = 1,
            BorderSizePixel = 0, ScrollBarThickness = 3, ScrollBarImageColor3 = WHITE, ScrollBarImageTransparency = 0.7,
            CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y, ZIndex = 3,
        }, card)
        local layout = new("UIListLayout", { Padding = UDim.new(0, 2), SortOrder = Enum.SortOrder.LayoutOrder }, list)

        local function rebuild()
            for _, c in ipairs(list:GetChildren()) do
                if c:IsA("GuiButton") or c:IsA("TextLabel") then c:Destroy() end
            end
            local q = string.lower(search.Text or "")
            local players = Players:GetPlayers()
            table.sort(players, function(a, b) return a.Name:lower() < b.Name:lower() end)
            local shown = 0
            for _, plr in ipairs(players) do
                if plr ~= LocalPlayer then
                    local hay = string.lower(plr.Name .. " " .. plr.DisplayName)
                    if q == "" or hay:find(q, 1, true) then
                        shown = shown + 1
                        local sel = spec.isSelected(plr)
                        local o = new("TextButton", {
                            Name = "P_" .. plr.UserId, Size = UDim2.new(1, -4, 0, 34), LayoutOrder = shown,
                            BackgroundColor3 = WHITE, BackgroundTransparency = sel and 0.86 or 1, Text = "",
                            AutoButtonColor = false, BorderSizePixel = 0, ZIndex = 4,
                        }, list)
                        corner(o, 9)
                        local chip = new("Frame", {
                            Position = UDim2.new(0, 8, 0.5, -11), Size = UDim2.fromOffset(22, 22),
                            BackgroundColor3 = T.card2, BorderSizePixel = 0, ZIndex = 5, Active = false,
                        }, o)
                        corner(chip, 11)
                        local initial = UI.label(chip, string.upper(string.sub(plr.DisplayName, 1, 1)), 0, 0, 22, 22, 10, T.sub, GB, CENTER)
                        initial.ZIndex = 6
                        local img = new("ImageLabel", {
                            Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, BorderSizePixel = 0, Image = "",
                            ScaleType = Enum.ScaleType.Crop, ZIndex = 7, Active = false,
                        }, chip)
                        corner(img, 11)
                        UI.applyAvatar(img, plr.UserId, initial)
                        local nm = UI.label(o, plr.DisplayName, 38, 3, 150, 15, 12, T.text, GM)
                        nm.TextTruncate = Enum.TextTruncate.AtEnd
                        nm.ZIndex = 5
                        local hd = UI.label(o, "@" .. plr.Name, 38, 18, 150, 12, 9, T.dim, GR)
                        hd.TextTruncate = Enum.TextTruncate.AtEnd
                        hd.ZIndex = 5
                        local ind = new("Frame", {
                            AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -10, 0.5, 0), Size = UDim2.fromOffset(14, 14),
                            BackgroundColor3 = Color3.fromRGB(226, 228, 232), BackgroundTransparency = sel and 0 or 1,
                            BorderSizePixel = 0, ZIndex = 5, Active = false,
                        }, o)
                        corner(ind, 7)
                        stroke(ind, WHITE, 1, sel and 1 or 0.6)
                        o.MouseEnter:Connect(function()
                            if not spec.isSelected(plr) then tween(o, 0.08, { BackgroundTransparency = 0.93 }) end
                        end)
                        o.MouseLeave:Connect(function()
                            tween(o, 0.08, { BackgroundTransparency = spec.isSelected(plr) and 0.86 or 1 })
                        end)
                        o.MouseButton1Click:Connect(function()
                            if dead() then return end
                            spec.onClick(plr)
                            rebuild()
                        end)
                    end
                end
            end
            if shown == 0 then
                local e = UI.label(list, "No players found.", 0, 0, 100, 28, 11, T.dim, GR, CENTER)
                e.Size = UDim2.new(1, 0, 0, 28)
                e.ZIndex = 4
            end
            if spec.afterRebuild then spec.afterRebuild() end
        end
        search:GetPropertyChangedSignal("Text"):Connect(rebuild)
        return { rebuild = rebuild, search = search, list = list, layout = layout }
    end

    -- two-column row (left / right frames), used for a list beside a status card
    function W.splitRow(page, height, leftFrac)
        local lf = leftFrac or 0.6
        local r = row(page, height)
        local left = new("Frame", {
            Size = UDim2.new(lf, -5, 1, 0), BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 2,
        }, r)
        local right = new("Frame", {
            Position = UDim2.new(lf, 5, 0, 0), Size = UDim2.new(1 - lf, -5, 1, 0),
            BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 2,
        }, r)
        return r, left, right
    end

    -- glass card with a title and label / value rows; returns the value labels in order
    function W.statusCard(parent, title, names)
        local card = new("Frame", {
            Size = UDim2.fromScale(1, 1), BackgroundColor3 = WHITE, BackgroundTransparency = 0.95,
            BorderSizePixel = 0, ZIndex = 2,
        }, parent)
        corner(card, 12)
        stroke(card, WHITE, 1, 0.85)
        local t = UI.label(card, title, 14, 12, 200, 16, 12, T.sub, GM)
        t.ZIndex = 3
        W.pill(t, 20)
        local values = {}
        for i, name in ipairs(names) do
            local y = 40 + (i - 1) * 28
            local a = UI.label(card, name, 14, y, 70, 24, 11, T.sub, GM)
            a.ZIndex = 3
            W.pill(a, 24)
            local box = new("Frame", {
                AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -10, 0, y),
                Size = UDim2.new(0, 104, 0, 24), BackgroundColor3 = BLACK, BackgroundTransparency = 0.72,
                BorderSizePixel = 0, ZIndex = 3, Active = false,
            }, card)
            corner(box, 8)
            stroke(box, WHITE, 1, 0.88)
            local v = UI.label(box, "-", 0, 0, 104, 24, 12, T.text, GB)
            v.Position = UDim2.fromOffset(0, 0)
            v.Size = UDim2.fromScale(1, 1)
            v.TextXAlignment = CENTER
            v.TextTruncate = Enum.TextTruncate.AtEnd
            v.ZIndex = 4
            values[i] = v
        end
        return values
    end

    -- ---------------------------------------------------------------
    -- Open animation + ticker (the background effect was removed)
    -- ---------------------------------------------------------------
    whitelistGui:GetPropertyChangedSignal("Enabled"):Connect(function()
        if whitelistGui.Enabled then
            if UI._listsDirty then
                UI._listsDirty = false
                task.defer(function()
                    if UI.refreshAll then UI.refreshAll() end
                end)
            end
            fitScale()
            local target = cardScale.Scale
            cardScale.Scale = target * 0.96
            tween(cardScale, 0.18, { Scale = target })
            dimLayer.BackgroundTransparency = 1
            shell.applyGlass()
        else
            W.closePopup()
        end
    end)

    -- slow ticker: keeps mode buttons / bind text / live status in sync with the script state
    local acc = 0
    RunService.Heartbeat:Connect(function(dt)
        if SCRIPT_KILLED or not whitelistGui.Parent or not whitelistGui.Enabled then return end
        acc = acc + dt
        if acc < 0.3 then return end
        acc = 0
        syncWidgets()
    end)

    shell.applyGlass()
end)()

-- ====================================================================
--  PAGES  (each tab wired to the script's real settings)
-- ====================================================================
;(function()
    local T = UI.Theme
    local W, shell, glass = UI.W, UI.shell, UI.glass
    local ESP = UI.esp
    local LEFT = Enum.TextXAlignment.Left

    local function dead()
        return SCRIPT_KILLED or not MENU_UNLOCKED
    end

    local function keyName(code)
        if code and code ~= Enum.KeyCode.Unknown then
            return code.Name
        end
        return "NONE"
    end

    -- =================================================================
    -- COMBAT > COLORBOT   (sub-tabs: Host / Settings / Keybinds)
    -- =================================================================
    shell.addGroup("COMBAT")
    shell.addTab("Colorbot", "aim", nil)
    do
        local HOST = shell.addSub("Colorbot", "Host")
        local SETTINGS = shell.addSub("Colorbot", "Settings")
        local KEYS = shell.addSub("Colorbot", "Keybinds")

        W.section(HOST, "Colorbot")
        W.toggle(HOST, "Enabled",
            function() return COLORBOT_ENABLED end,
            function(v)
                if v and TRIGGERBOT_ENABLED then
                    UI.notify("Can't enable Colorbot", "Colorbot and Triggerbot can't be enabled at the same time. Turn Triggerbot off first.")
                    return
                end
                COLORBOT_ENABLED = v
                if colorbotToggleHandler then
                    colorbotToggleHandler(COLORBOT_ENABLED)
                end
            end)

        W.section(HOST, "Host")
        target1Status = W.statusLabel(HOST, "Select a host below, then turn the Colorbot on.")
        local _, hostLeft, hostRight = W.splitRow(HOST, 236)
        local hostPL = W.playerList(HOST, 236, {
            host = hostLeft,
            isSelected = function(plr) return plr == selectedActivator end,
            onClick = function(plr)
                if selectedActivator == plr then
                    selectActivator(nil)
                else
                    selectActivator(plr)
                end
            end,
        })
        hostSearchBox = hostPL.search
        hostList = hostPL.list
        hostListLayout = hostPL.layout
        rebuildHostList = hostPL.rebuild

        local live = W.statusCard(hostRight, "Live Status", { "Trigger", "Host", "Bind", "Mode", "Shot Delay" })
        UI.live.trigger = live[1]
        UI.live.host = live[2]
        UI.live.bind = live[3]
        UI.live.mode = live[4]
        UI.live.delay = live[5]

        W.section(SETTINGS, "Shot Delay")
        W.numberInput(SETTINGS, "Min Delay (ms)", 0, 1000, 1,
            function() return HOST_DELAY_MIN_MS end,
            function(v)
                HOST_DELAY_MIN_MS = v
                if HOST_DELAY_MAX_MS < v then HOST_DELAY_MAX_MS = v end
            end)
        W.numberInput(SETTINGS, "Max Delay (ms)", 0, 1000, 1,
            function() return HOST_DELAY_MAX_MS end,
            function(v)
                HOST_DELAY_MAX_MS = v
                if HOST_DELAY_MIN_MS > v then HOST_DELAY_MIN_MS = v end
            end)
        W.rangeBar(SETTINGS, "DELAY RANGE",
            function() return HOST_DELAY_MIN_MS end,
            function() return HOST_DELAY_MAX_MS end,
            function(v)
                HOST_DELAY_MIN_MS = v
                if HOST_DELAY_MAX_MS < v then HOST_DELAY_MAX_MS = v end
            end,
            function(v)
                HOST_DELAY_MAX_MS = v
                if HOST_DELAY_MIN_MS > v then HOST_DELAY_MIN_MS = v end
            end,
            1000, "ms")

        W.section(SETTINGS, "Detection")
        W.toggle(SETTINGS, "Knock Check",
            function() return UI.knock.colorbot end,
            function(v) UI.knock.colorbot = v end)

        W.section(KEYS, "Trigger")
        triggerKeyButton = W.keybind(KEYS, "Trigger Key")
        triggerKeyButton.Text = TRIGGER_KEY_LABEL
        triggerKeyButton.MouseButton2Click:Connect(function()
            if dead() or UI.rmbGuard then return end
            TRIGGER_KEY = nil
            TRIGGER_KEYCODE = Enum.KeyCode.Unknown
            TRIGGER_KEY_LABEL = "NONE"
            TRIGGER_HELD, TRIGGER_TOGGLED, TRIGGER_BIND_LISTENING = false, false, false
            cancelHostTrigger()
            resetHostRound()
            updateTriggerBindText()
        end)
        W.register(function() if updateTriggerBindText then updateTriggerBindText() end end)

        W.segmented(KEYS, "Mode", { "Hold", "Toggle" },
            function() return TRIGGER_MODE end,
            function(mode)
                TRIGGER_MODE = mode
                TRIGGER_HELD = false
                TRIGGER_TOGGLED = false
                HOST_TRIGGER_ARMED = false
                AIM_ASSIST_TARGET = nil
                cancelHostTrigger()
                resetHostRound()
            end)
    end

    -- =================================================================
    -- COMBAT > TRIGGERBOT   (sub-tabs: Targets / Settings / Keybinds / Distance)
    -- =================================================================
    shell.addTab("Triggerbot", "aim", nil)
    do
        local TGT = shell.addSub("Triggerbot", "Targets", true)
        local SET = shell.addSub("Triggerbot", "Settings")
        local KEYS = shell.addSub("Triggerbot", "Keybinds")
        local DIST = shell.addSub("Triggerbot", "Distance")

        -- Targets: enable, whitelist / blacklist, live status beside the list
        W.section(TGT, "Triggerbot")
        W.toggle(TGT, "Enabled",
            function() return TRIGGERBOT_ENABLED end,
            function(v)
                if v and COLORBOT_ENABLED then
                    UI.notify("Can't enable Triggerbot", "Colorbot and Triggerbot can't be enabled at the same time. Turn Colorbot off first.")
                    return
                end
                TRIGGERBOT_ENABLED = v
                TB_HELD = false
                TB_TOGGLED = false
                TB_BIND_LISTENING = false
                if TB_PAGE_REFRESH then TB_PAGE_REFRESH() end
                if updateLiveStatus then updateLiveStatus() end
            end)

        W.section(TGT, "Targets")
        W.segmented(TGT, "List Mode", { "Whitelist", "Blacklist" },
            function() return TB_LIST_MODE end,
            function(mode)
                local wasEnabled = TRIGGERBOT_ENABLED
                TB_LIST_MODE = mode
                TRIGGERBOT_ENABLED = wasEnabled
                if TB_PAGE_REFRESH then TB_PAGE_REFRESH() end
            end)
        local _, tbLeft, tbRight = W.splitRow(TGT, 236)
        local tbPL = W.playerList(TGT, 236, {
            host = tbLeft,
            isSelected = function(plr) return TB_SELECTED[plr.UserId] == true end,
            onClick = function(plr) TB_SELECTED[plr.UserId] = not TB_SELECTED[plr.UserId] end,
        })
        TB_LIST_REBUILD = tbPL.rebuild
        UI._tbRefreshListMode = function() end

        local tbLive = W.statusCard(tbRight, "Live Status", { "Trigger", "List", "Bind", "Mode", "Shot Delay" })
        UI.triggerbotPageStatus = {
            enabled = tbLive[1], host = tbLive[2], bind = tbLive[3], mode = tbLive[4], delay = tbLive[5],
        }

        -- Settings
        W.section(SET, "Triggerbot")
        W.numberInput(SET, "FOV Radius", 0, 500, 1,
            function() return UI.tb.fov end,
            function(v) UI.tb.fov = v; UI.tb.pendingChar = nil end)
        W.numberInput(SET, "Delay (0-100)", 0, 100, 1,
            function() return UI.tb.strength end,
            function(v) UI.tb.strength = v; UI.tb.pendingChar = nil end)
        W.toggle(SET, "Knock Check",
            function() return UI.knock.triggerbot end,
            function(v) UI.knock.triggerbot = v end)

        -- Keybinds
        W.section(KEYS, "Trigger")
        local tbKeyBtn = W.keybind(KEYS, "Trigger Key")
        tbKeyBtn.Text = TB_KEY_LABEL
        TB_PAGE_REFRESH = function()
            if tbKeyBtn and tbKeyBtn.Parent then
                tbKeyBtn.Text = TB_BIND_LISTENING and "PRESS A KEY..." or TB_KEY_LABEL
            end
        end
        tbKeyBtn.MouseButton1Click:Connect(function()
            if dead() then return end
            TB_BIND_LISTENING = true
            TB_PAGE_REFRESH()
        end)
        tbKeyBtn.MouseButton2Click:Connect(function()
            if dead() or UI.rmbGuard then return end
            TB_KEY = nil
            TB_KEYCODE = Enum.KeyCode.Unknown
            TB_KEY_LABEL = "NONE"
            TB_HELD, TB_TOGGLED, TB_BIND_LISTENING = false, false, false
            TB_PAGE_REFRESH()
        end)
        W.register(function() TB_PAGE_REFRESH() end)

        W.segmented(KEYS, "Mode", { "Hold", "Toggle" },
            function() return TB_MODE end,
            function(mode)
                TB_MODE = mode
                TB_HELD = false
                TB_TOGGLED = false
            end)

        -- Distance
        W.section(DIST, "Max Distance")
        local weapons = {
            { key = "Double", title = "Double Barrel" },
            { key = "Tac", title = "Tactical Shotgun" },
            { key = "Rev", title = "Revolver" },
        }
        for _, w in ipairs(weapons) do
            W.numberInput(DIST, w.title .. " (studs)", 0, 5000, 5,
                function() return UI.distance[w.key] end,
                function(v) UI.distance[w.key] = v end)
        end
        W.caption(DIST, "Triggerbot only fires on a target within this many studs for the weapon you are holding.", 28)
        W.button(DIST, "Reset distances to defaults", "default", function()
            for _, w in ipairs(weapons) do
                UI.distance[w.key] = UI.distanceDefaults[w.key]
            end
            UI.syncWidgets()
        end)
    end

    -- =================================================================
    -- COMBAT > AIM ASSIST   (sub-tabs: Aim / Keybinds; body part presets sit under the Parts preview)
    -- =================================================================
    shell.addTab("Aim Assist", "aim", "aim")
    do
        local AIM = shell.addSub("Aim Assist", "Aim")
        local KEYS = shell.addSub("Aim Assist", "Keybinds")

        W.section(AIM, "Aim Assist")
        W.toggle(AIM, "Enabled",
            function() return AIM_ASSIST_ENABLED end,
            function(v)
                AIM_ASSIST_ENABLED = v
                AIM_BIND.held, AIM_BIND.toggled = false, false
                if resetAimAssistState then resetAimAssistState() end
            end)

        W.section(AIM, "Targeting")
        W.numberInput(AIM, "FOV Radius", 5, 500, 1,
            function() return AIM_ASSIST_RADIUS end,
            function(v) AIM_ASSIST_RADIUS = v end)
        W.numberInput(AIM, "Strength (0-100)", 0, 100, 1,
            function() return math.floor(math.clamp(AIM_ASSIST_STRENGTH * 1000, 0, 100) + 0.5) end,
            function(v) AIM_ASSIST_STRENGTH = v / 1000 end)

        -- Quick presets sit in a row under the Aim Preview panel
        local PRESETS = {
            { "Head Only", { Head = true } },
            { "Head + Torso", { Head = true, Torso = true } },
            { "All Parts", { Head = true, Torso = true, LeftArm = true, RightArm = true, LeftLeg = true, RightLeg = true } },
        }
        local PRESET_W = { 66, 78, 58 }
        local presetX = 12
        for i, preset in ipairs(PRESETS) do
            local b = UI.new("TextButton", {
                Name = "Preset_" .. i, Position = UDim2.fromOffset(presetX, 404), Size = UDim2.fromOffset(PRESET_W[i], 26),
                BackgroundColor3 = Color3.new(1, 1, 1), BackgroundTransparency = 0.9, Text = preset[1],
                TextColor3 = T.text, TextSize = 10, Font = Enum.Font.GothamMedium, AutoButtonColor = false,
                BorderSizePixel = 0, ZIndex = 5,
            }, UI.aimBody)
            UI.corner(b, 10)
            UI.registerCard(b, "row", UI.stroke(b, Color3.new(1, 1, 1), 1, 0.9))
            b.MouseEnter:Connect(function() UI.tween(b, 0.1, { BackgroundTransparency = UI.cardStyles[UI.cardStyle].row - 0.12 }) end)
            b.MouseLeave:Connect(function() UI.tween(b, 0.1, { BackgroundTransparency = UI.cardStyles[UI.cardStyle].row }) end)
            b.MouseButton1Click:Connect(function()
                if dead() then return end
                for _, name in ipairs({ "Head", "Torso", "LeftArm", "RightArm", "LeftLeg", "RightLeg" }) do
                    AIM_BIND.parts[name] = preset[2][name] == true
                end
                UI.drawAimPreview()
            end)
            presetX = presetX + PRESET_W[i] + 5
        end

        UI.aimHook.isOn = function(name)
            return AIM_BIND.parts[name] == true
        end
        UI.aimHook.toggle = function(name)
            local count = 0
            for _, on in pairs(AIM_BIND.parts) do
                if on == true then count = count + 1 end
            end
            if AIM_BIND.parts[name] and count <= 1 then
                return -- always keep at least one part
            end
            AIM_BIND.parts[name] = not AIM_BIND.parts[name]
        end
        UI._aimPartsRefresh = function()
            if UI.drawAimPreview then UI.drawAimPreview() end
        end

        -- Keybinds
        W.section(KEYS, "Aim")
        local aimKeyBtn = W.keybind(KEYS, "Aim Key")
        aimKeyBtn.Text = AIM_ASSIST_KEY_LABEL
        aimKeyBtn.MouseButton1Click:Connect(function()
            if dead() then return end
            AIM_ASSIST_BIND_LISTENING = true
            if AIM_BIND.refresh then AIM_BIND.refresh() end
        end)
        aimKeyBtn.MouseButton2Click:Connect(function()
            if dead() or UI.rmbGuard then return end
            AIM_BIND.key = nil
            AIM_ASSIST_KEYCODE = Enum.KeyCode.Unknown
            AIM_ASSIST_KEY_LABEL = "NONE"
            AIM_BIND.held, AIM_BIND.toggled = false, false
            AIM_ASSIST_BIND_LISTENING = false
            if AIM_BIND.refresh then AIM_BIND.refresh() end
        end)
        AIM_BIND.refresh = function()
            if aimKeyBtn and aimKeyBtn.Parent then
                aimKeyBtn.Text = AIM_ASSIST_BIND_LISTENING and "PRESS A KEY..." or AIM_ASSIST_KEY_LABEL
            end
        end
        W.register(function() AIM_BIND.refresh() end)

        W.segmented(KEYS, "Mode", { "Hold", "Toggle" },
            function() return AIM_BIND.mode end,
            function(mode)
                AIM_BIND.mode = mode
                AIM_BIND.held = false
                AIM_BIND.toggled = false
                if resetAimAssistState then resetAimAssistState() end
            end)
    end

    -- =================================================================
    -- MISC > Visuals (ESP) + Movement (speed / fly)
    -- =================================================================
    shell.addGroup("MISC")
    shell.addTab("Visuals", "eye", "esp")
    shell.addTab("Movement", "bolt", nil)
    do
        local LIMIT_MIN, LIMIT_MAX = 10, 5000
        local listeningBind = false
        local MAIN = shell.addSub("Visuals", "General")
        local TGT = shell.addSub("Visuals", "Targets")
        local KEYS = shell.addSub("Visuals", "Keybinds")
        local MV = shell.addSub("Movement", "Movement")

        -- ===== Movement: walk speed + fly =====
        do
            local MOVE = UI.move

            W.section(MV, "Speed")
            W.toggle(MV, "Speed Enabled",
                function() return MOVE.speedOn end,
                function(v) MOVE.speedOn = v end)
            W.numberInput(MV, "Walk Speed (1-1000)", 1, 1000, 1,
                function() return MOVE.speed end,
                function(v) MOVE.speed = v end)

            local speedKeyBtn = W.keybind(MV, "Speed Toggle Key")
            local function refreshSpeedKey()
                speedKeyBtn.Text = MOVE.speedListening and "PRESS A KEY..." or (MOVE.speedKey and MOVE.speedKey.Name or "NONE")
            end
            speedKeyBtn.MouseButton1Click:Connect(function()
                if dead() then return end
                MOVE.speedListening, MOVE.listening = true, false
                refreshSpeedKey()
            end)
            speedKeyBtn.MouseButton2Click:Connect(function()
                if dead() or UI.rmbGuard then return end
                MOVE.speedKey, MOVE.speedListening = nil, false
                refreshSpeedKey()
            end)
            W.register(refreshSpeedKey)
            refreshSpeedKey()

            W.section(MV, "Fly")
            W.toggle(MV, "Fly Enabled",
                function() return MOVE.flyOn end,
                function(v) MOVE.flyOn = v end)
            W.numberInput(MV, "Fly Speed (1-1000)", 1, 1000, 1,
                function() return MOVE.flySpeed end,
                function(v) MOVE.flySpeed = v end)

            local flyKeyBtn = W.keybind(MV, "Fly Toggle Key")
            local function refreshFlyKey()
                flyKeyBtn.Text = MOVE.listening and "PRESS A KEY..." or (MOVE.flyKey and MOVE.flyKey.Name or "NONE")
            end
            flyKeyBtn.MouseButton1Click:Connect(function()
                if dead() then return end
                MOVE.listening, MOVE.speedListening = true, false
                refreshFlyKey()
            end)
            flyKeyBtn.MouseButton2Click:Connect(function()
                if dead() or UI.rmbGuard then return end
                MOVE.flyKey, MOVE.listening = nil, false
                refreshFlyKey()
            end)
            W.register(refreshFlyKey)
            refreshFlyKey()

            UserInputService.InputBegan:Connect(function(input, gameProcessed)
                if SCRIPT_KILLED or not MENU_UNLOCKED then return end
                if MOVE.listening or MOVE.speedListening then
                    if input.UserInputType == Enum.UserInputType.Keyboard
                        and input.KeyCode ~= Enum.KeyCode.Unknown then
                        if input.KeyCode == Enum.KeyCode.Escape then
                            -- ESC clears the keybind being set (leaves it unbound)
                            if MOVE.speedListening then MOVE.speedKey = nil
                            else MOVE.flyKey = nil end
                        else
                            if MOVE.speedListening then MOVE.speedKey = input.KeyCode
                            else MOVE.flyKey = input.KeyCode end
                        end
                        MOVE.listening, MOVE.speedListening = false, false
                        refreshFlyKey()
                        refreshSpeedKey()
                    end
                    return
                end
                if gameProcessed or UserInputService:GetFocusedTextBox() then return end
                if input.UserInputType == Enum.UserInputType.Keyboard then
                    if MOVE.flyKey and input.KeyCode == MOVE.flyKey then
                        MOVE.flyOn = not MOVE.flyOn
                        UI.syncWidgets()
                    end
                    if MOVE.speedKey and input.KeyCode == MOVE.speedKey then
                        MOVE.speedOn = not MOVE.speedOn
                        UI.syncWidgets()
                    end
                end
            end)

            -- Speed + fly driver
            local flyBV, flyBG
            local function clearFly()
                if flyBV then pcall(function() flyBV:Destroy() end) flyBV = nil end
                if flyBG then pcall(function() flyBG:Destroy() end) flyBG = nil end
            end

            local conn
            conn = RunService.RenderStepped:Connect(function()
                local char = LocalPlayer.Character
                local hum = char and char:FindFirstChildOfClass("Humanoid")
                local root = char and char:FindFirstChild("HumanoidRootPart")

                if SCRIPT_KILLED then
                    clearFly()
                    if hum and MOVE.speedApplied then hum.WalkSpeed = 16 end
                    MOVE.speedApplied = false
                    conn:Disconnect()
                    return
                end

                -- walk speed
                if hum then
                    if MOVE.speedOn then
                        local ws = MOVE.speed * UI.move.mult
                        if hum.WalkSpeed ~= ws then hum.WalkSpeed = ws end
                        MOVE.speedApplied = true
                    elseif MOVE.speedApplied then
                        hum.WalkSpeed = 16
                        MOVE.speedApplied = false
                    end
                end

                -- Softens the upward launch from stepping / climbing at high walk speeds.
                -- Only the upward speed ADDED this frame is scaled (keep = fraction kept), so the
                -- launch still builds up instead of being flattened. Above a setting of launchRef
                -- the fraction shrinks, so 300 launches less than 100 (but never none).
                if MOVE.speedOn and not MOVE.flyOn and root and hum and hum.Health > 0 then
                    local vel = root.AssemblyLinearVelocity
                    local lastY = MOVE.lastY or vel.Y
                    local newY = vel.Y
                    if vel.Y > MOVE.launchStart and vel.Y > lastY then
                        local keep = MOVE.launchKeep
                        if MOVE.speed > MOVE.launchRef then
                            keep = keep * math.sqrt(MOVE.launchRef / MOVE.speed)
                        end
                        newY = lastY + (vel.Y - lastY) * math.max(keep, 0.05)
                        root.AssemblyLinearVelocity = Vector3.new(vel.X, newY, vel.Z)
                    end
                    MOVE.lastY = newY
                else
                    MOVE.lastY = nil
                end

                -- fly
                if MOVE.flyOn and hum and root and hum.Health > 0 then
                    if not flyBV or flyBV.Parent ~= root then
                        clearFly()
                        flyBV = Instance.new("BodyVelocity")
                        flyBV.Name = "ArmadaFlyBV"
                        flyBV.MaxForce = Vector3.new(9e9, 9e9, 9e9)
                        flyBV.Velocity = Vector3.zero
                        flyBV.Parent = root
                        flyBG = Instance.new("BodyGyro")
                        flyBG.Name = "ArmadaFlyBG"
                        flyBG.MaxTorque = Vector3.new(0, 9e9, 0)
                        flyBG.P = 9e4
                        flyBG.D = 1000
                        flyBG.Parent = root
                    end

                    local cam = workspace.CurrentCamera
                    if cam then
                        local dir = Vector3.zero
                        if not UserInputService:GetFocusedTextBox() then
                            local look, right = cam.CFrame.LookVector, cam.CFrame.RightVector
                            if UserInputService:IsKeyDown(Enum.KeyCode.W) then dir = dir + look end
                            if UserInputService:IsKeyDown(Enum.KeyCode.S) then dir = dir - look end
                            if UserInputService:IsKeyDown(Enum.KeyCode.D) then dir = dir + right end
                            if UserInputService:IsKeyDown(Enum.KeyCode.A) then dir = dir - right end
                            if UserInputService:IsKeyDown(Enum.KeyCode.Space) then dir = dir + Vector3.yAxis end
                            if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then dir = dir - Vector3.yAxis end
                        end
                        flyBV.Velocity = dir.Magnitude > 0 and dir.Unit * (MOVE.flySpeed * UI.move.mult) or Vector3.zero
                        local flat = Vector3.new(cam.CFrame.LookVector.X, 0, cam.CFrame.LookVector.Z)
                        if flat.Magnitude > 1e-3 then
                            flyBG.CFrame = CFrame.lookAt(root.Position, root.Position + flat)
                        end
                    end
                else
                    clearFly()
                end
            end)
        end

        W.section(MAIN, "General")
        W.toggle(MAIN, "ESP Enabled",
            function() return ESP.enabled end,
            function(v) ESP.enabled = v end)
        W.dropdown(MAIN, "ESP Type", { "2D", "3D", "Corner" },
            function() return ESP.type end,
            function(v) ESP.type = v end)
        W.toggle(MAIN, "Limit Distance",
            function() return ESP.limit end,
            function(v) ESP.limit = v end)
        W.numberInput(MAIN, "Max Distance (studs)", LIMIT_MIN, LIMIT_MAX, 10,
            function() return ESP.limitValue end,
            function(v) ESP.limitValue = v end,
            function(v) return tostring(v) .. " studs" end)

        W.section(MAIN, "Elements")
        local function color(key)
            return function() return ESP.colors[key] end, function(c) ESP.colors[key] = c end
        end
        local gBox, sBox = color("box")
        W.toggleColor(MAIN, "Box", function() return ESP.box end, function(v) ESP.box = v end, gBox, sBox)
        W.toggle(MAIN, "Box Fill", function() return ESP.filled end, function(v) ESP.filled = v end)
        local gName, sName = color("name")
        W.toggleColor(MAIN, "Name", function() return ESP.name end, function(v) ESP.name = v end, gName, sName)
        W.toggle(MAIN, "Health Bar", function() return ESP.health end, function(v) ESP.health = v end)
        local gDist, sDist = color("distance")
        W.toggleColor(MAIN, "Distance", function() return ESP.distance end, function(v) ESP.distance = v end, gDist, sDist)
        local gSnap, sSnap = color("snapline")
        W.toggleColor(MAIN, "Snapline", function() return ESP.snapline end, function(v) ESP.snapline = v end, gSnap, sSnap)
        local gBone, sBone = color("bone")
        W.toggleColor(MAIN, "Skeleton", function() return ESP.bone end, function(v) ESP.bone = v end, gBone, sBone)

        -- Targets
        W.section(TGT, "Targets")
        W.segmented(TGT, "List Mode", { "Whitelist", "Blacklist" },
            function() return ESP.listMode end,
            function(mode) ESP.listMode = mode end)
        local espPL = W.playerList(TGT, 344, {
            isSelected = function(plr) return ESP.selected[plr.UserId] == true end,
            onClick = function(plr) ESP.selected[plr.UserId] = not ESP.selected[plr.UserId] end,
        })
        UI._espListRebuild = espPL.rebuild
        Players.PlayerAdded:Connect(function() UI.deferRebuild(espPL.rebuild) end)
        Players.PlayerRemoving:Connect(function(player)
            ESP.selected[player.UserId] = nil
            UI.deferRebuild(espPL.rebuild)
        end)

        -- Keybinds
        W.section(KEYS, "Toggle")
        local espKeyBtn = W.keybind(KEYS, "Toggle Key")
        local function refreshEspBind()
            espKeyBtn.Text = listeningBind and "PRESS A KEY..." or keyName(ESP.binds.master)
        end
        espKeyBtn.MouseButton1Click:Connect(function()
            if dead() then return end
            listeningBind = true
            refreshEspBind()
        end)
        espKeyBtn.MouseButton2Click:Connect(function()
            if dead() or UI.rmbGuard then return end
            ESP.binds.master = Enum.KeyCode.Unknown
            listeningBind = false
            refreshEspBind()
        end)
        W.register(refreshEspBind)
        refreshEspBind()

        -- key handling (called from the global InputBegan handler)
        local function toggleFeature(key)
            if key == "master" then
                ESP.enabled = not ESP.enabled
                UI.syncWidgets()
            end
        end

        UI._espHandleKey = function(input)
            if input.UserInputType ~= Enum.UserInputType.Keyboard then
                return false
            end
            local code = input.KeyCode

            if listeningBind then
                if code == Enum.KeyCode.Unknown then
                    return true
                end
                if code == Enum.KeyCode.Escape or code == Enum.KeyCode.Backspace then
                    ESP.binds.master = Enum.KeyCode.Unknown
                else
                    ESP.binds.master = code
                end
                listeningBind = false
                refreshEspBind()
                return true
            end

            if code == Enum.KeyCode.Unknown then return false end
            if UserInputService:GetFocusedTextBox() then return false end
            if TRIGGER_BIND_LISTENING or TB_BIND_LISTENING or AIM_ASSIST_BIND_LISTENING or UI_BIND_LISTENING then
                return false
            end

            for _, key in ipairs(UI.espBindOrder) do
                if ESP.binds[key] == code then
                    toggleFeature(key)
                end
            end
            return false
        end

        -- config integration -------------------------------------------
        local BOOL_KEYS = { "enabled", "box", "filled", "distance", "name", "health", "snapline", "bone", "limit" }

        UI._espCollect = function()
            local colors, binds = {}, {}
            for key, c in pairs(ESP.colors) do
                colors[key] = {
                    math.floor(c.R * 255 + 0.5),
                    math.floor(c.G * 255 + 0.5),
                    math.floor(c.B * 255 + 0.5),
                }
            end
            for _, key in ipairs(UI.espBindOrder) do
                local code = ESP.binds[key]
                binds[key] = (code and code ~= Enum.KeyCode.Unknown) and code.Name or "NONE"
            end
            return {
                enabled = ESP.enabled, type = ESP.type, box = ESP.box, filled = ESP.filled,
                distance = ESP.distance, name = ESP.name, health = ESP.health, snapline = ESP.snapline,
                bone = ESP.bone, limit = ESP.limit, limitValue = ESP.limitValue, colors = colors, binds = binds,
            }
        end

        UI._espSanitize = function(src)
            local out = {}
            if type(src) ~= "table" then
                return out
            end
            for _, key in ipairs(BOOL_KEYS) do
                if type(src[key]) == "boolean" then
                    out[key] = src[key]
                end
            end
            if src.type == "2D" or src.type == "3D" or src.type == "Corner" then
                out.type = src.type
            end
            if type(src.limitValue) == "number" and src.limitValue == src.limitValue then
                out.limitValue = math.floor(math.clamp(src.limitValue, LIMIT_MIN, LIMIT_MAX) + 0.5)
            end
            if type(src.colors) == "table" then
                out.colors = {}
                for key in pairs(ESP.colors) do
                    local c = src.colors[key]
                    if type(c) == "table" then
                        local r, g, b = tonumber(c[1]), tonumber(c[2]), tonumber(c[3])
                        if r and g and b and r == r and g == g and b == b then
                            out.colors[key] = {
                                math.floor(math.clamp(r, 0, 255)),
                                math.floor(math.clamp(g, 0, 255)),
                                math.floor(math.clamp(b, 0, 255)),
                            }
                        end
                    end
                end
            end
            if type(src.binds) == "table" then
                out.binds = {}
                for _, key in ipairs(UI.espBindOrder) do
                    if type(src.binds[key]) == "string" then
                        out.binds[key] = src.binds[key]
                    end
                end
            end
            return out
        end

        UI._espApply = function(cfg)
            if type(cfg) ~= "table" then return end
            for _, key in ipairs(BOOL_KEYS) do
                if type(cfg[key]) == "boolean" then
                    ESP[key] = cfg[key]
                end
            end
            if cfg.type == "2D" or cfg.type == "3D" or cfg.type == "Corner" then
                ESP.type = cfg.type
            end
            if type(cfg.limitValue) == "number" then
                ESP.limitValue = math.floor(math.clamp(cfg.limitValue, LIMIT_MIN, LIMIT_MAX) + 0.5)
            end
            if type(cfg.colors) == "table" then
                for key, c in pairs(cfg.colors) do
                    if ESP.colors[key] and type(c) == "table" then
                        ESP.colors[key] = Color3.fromRGB(c[1], c[2], c[3])
                    end
                end
            end
            if type(cfg.binds) == "table" then
                for key, name in pairs(cfg.binds) do
                    if ESP.binds[key] ~= nil and type(name) == "string" then
                        local code = Enum.KeyCode.Unknown
                        if name ~= "NONE" then
                            local ok, value = pcall(function()
                                return Enum.KeyCode[name]
                            end)
                            if ok and value then
                                code = value
                            end
                        end
                        ESP.binds[key] = code
                    end
                end
            end
            listeningBind = false
            W.closePopup()
            UI.syncWidgets()
            UI.refreshPreview()
        end
    end

    -- =================================================================
    -- SETTINGS   (sub-tabs: Appearance / Keybinds / Reset)
    -- =================================================================
    shell.pinNavToBottom(2) -- bottom of the sidebar, above the profile card
    shell.addGroup("SETTINGS")
    shell.addTab("Settings", "settings", nil)
    do
        local LOOK = shell.addSub("Settings", "Appearance")
        local KEYS = shell.addSub("Settings", "Keybinds")
        local RESET = shell.addSub("Settings", "Reset")

        W.section(LOOK, "Glass")
        W.slider(LOOK, "Glass Opacity", 0, 100, 1,
            function() return glass.level end,
            function(v)
                glass.level = v
                shell.applyGlass()
            end,
            function(v) return tostring(v) .. "%" end)
        W.caption(LOOK, "0% is fully see-through frosted glass (default). 100% makes the window solid.", 28)

        W.section(KEYS, "UI Controls")
        local hideBtn = W.keybind(KEYS, "Hide UI")
        local killBtn = W.keybind(KEYS, "Kill UI / Script")
        local status = W.statusLabel(KEYS, "Click a keybind, then press any keyboard key. ESC unbinds.")
        local function refreshUIKeys()
            hideBtn.Text = (UI_BIND_LISTENING == "HIDE") and "PRESS A KEY..." or UI_HIDE_KEY_LABEL
            killBtn.Text = (UI_BIND_LISTENING == "KILL") and "PRESS A KEY..." or UI_KILL_KEY_LABEL
        end
        hideBtn.MouseButton1Click:Connect(function()
            if dead() then return end
            UI_BIND_LISTENING = "HIDE"
            status.Text = "Press the new HIDE UI key. ESC unbinds."
            refreshUIKeys()
        end)
        killBtn.MouseButton1Click:Connect(function()
            if dead() then return end
            UI_BIND_LISTENING = "KILL"
            status.Text = "Press the new KILL UI / SCRIPT key. ESC unbinds."
            refreshUIKeys()
        end)
        UI._refreshUIKeybindButtons = refreshUIKeys
        UI._uiKeybindStatus = status
        W.register(refreshUIKeys)
        refreshUIKeys()

        W.section(RESET, "Reset")
        local resetButton
        local confirmToken, armed = 0, false
        local function disarm()
            armed = false
            resetButton.Text = "Reset all settings"
            resetButton.TextColor3 = T.bad
        end
        resetButton = W.button(RESET, "Reset all settings", "danger", function()
            if not armed then
                armed = true
                confirmToken = confirmToken + 1
                local mine = confirmToken
                resetButton.Text = "Click again to confirm"
                task.delay(3, function()
                    if armed and confirmToken == mine then disarm() end
                end)
                return
            end
            confirmToken = confirmToken + 1
            local ok, err = UI.resetAllSettings()
            armed = false
            glass.level, glass.snow, glass.snowAmount, glass.blur = 0, true, 100, false
            shell.applyGlass()
            UI.syncWidgets()
            resetButton.Text = ok and "Settings reset" or ("Reset failed: " .. tostring(err))
            local mine = confirmToken
            task.delay(2, function()
                if not armed and confirmToken == mine then disarm() end
            end)
        end)
    end

    -- =================================================================
    -- CONFIGS   (sub-tabs: Library / Share)
    -- =================================================================
    shell.addTab("Configs", "configs", nil)
    do
        local LIB = shell.addSub("Configs", "Library")
        local SHARE = shell.addSub("Configs", "Share")

        W.section(LIB, "Library")
        configNameInput = W.textbox(LIB, "Config name", 30, false)
        configSelectedLabel = W.statusLabel(LIB, "SELECTED: NONE")

        local listRow = W.row(LIB, 150)
        local card = Instance.new("Frame")
        card.Size = UDim2.fromScale(1, 1)
        card.BackgroundColor3 = Color3.new(1, 1, 1)
        card.BackgroundTransparency = 0.95
        card.BorderSizePixel = 0
        card.ZIndex = 2
        card.Parent = listRow
        UI.corner(card, 12)
        UI.stroke(card, Color3.new(1, 1, 1), 1, 0.85)
        configList = Instance.new("ScrollingFrame")
        configList.Position = UDim2.fromOffset(6, 6)
        configList.Size = UDim2.new(1, -12, 1, -12)
        configList.BackgroundTransparency = 1
        configList.BorderSizePixel = 0
        configList.ScrollBarThickness = 3
        configList.ScrollBarImageColor3 = Color3.new(1, 1, 1)
        configList.ScrollBarImageTransparency = 0.7
        configList.CanvasSize = UDim2.new()
        configList.ZIndex = 3
        configList.Parent = card
        configLayout = Instance.new("UIListLayout")
        configLayout.Padding = UDim.new(0, 2)
        configLayout.SortOrder = Enum.SortOrder.Name
        configLayout.Parent = configList
        configLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
            configList.CanvasSize = UDim2.fromOffset(0, configLayout.AbsoluteContentSize.Y + 3)
        end)

        local bs = W.buttons(LIB, { { "Save" }, { "Load" }, { "Delete", "danger" } })
        configSaveButton, configLoadButton, configDeleteButton = bs[1], bs[2], bs[3]
        configStatusLabel = W.statusLabel(LIB, "")

        W.section(SHARE, "Share")
        configPasteInput = W.textbox(SHARE, "Paste a config here to import, or export to copy it out", 96, true)
        local bs2 = W.buttons(SHARE, { { "Import" }, { "Export" } })
        configImportButton, configExportButton = bs2[1], bs2[2]
        configStatusShare = W.statusLabel(SHARE, "")
    end

    -- =================================================================
    -- hooks the original logic expects
    -- =================================================================
    UI._tbSettingsRefresh = UI.syncWidgets
    UI._distanceRefresh = UI.syncWidgets
    UI._colorbotKnockRefresh = UI.syncWidgets
    UI._tbKnockRefresh = UI.syncWidgets
    UI._refreshColorbotMasterUI = UI.syncWidgets
    UI._refreshTriggerbotMasterUI = UI.syncWidgets

    UI.refreshAll = function()
        UI.syncWidgets()
        if AIM_BIND.refresh then AIM_BIND.refresh() end
        if TB_PAGE_REFRESH then TB_PAGE_REFRESH() end
        if updateTriggerBindText then pcall(updateTriggerBindText) end
        if rebuildHostList then rebuildHostList() end
        if TB_LIST_REBUILD then TB_LIST_REBUILD() end
        if UI._espListRebuild then UI._espListRebuild() end
        UI.refreshPreview()
    end

    shell.select("Colorbot")
end)()
local function refreshConfigList()
    if not configList or not configList.Parent then return end

    for _, child in ipairs(configList:GetChildren()) do
        if child:IsA("TextButton")
            or (child:IsA("TextLabel") and child.Name == "EmptyConfigs") then
            child:Destroy()
        end
    end

    local names = {}
    for name in pairs(savedConfigs) do
        table.insert(names, name)
    end
    table.sort(names, function(a, b)
        return string.lower(a) < string.lower(b)
    end)

    if #names == 0 then
        local empty = Instance.new("TextLabel")
        empty.Name = "EmptyConfigs"
        empty.Size = UDim2.new(1, -6, 0, 28)
        empty.BackgroundTransparency = 1
        empty.Text = "No saved configs yet."
        empty.TextColor3 = UI.Theme.dim
        empty.Font = Enum.Font.Gotham
        empty.TextSize = 11
        empty.TextXAlignment = Enum.TextXAlignment.Center
        empty.ZIndex = 4
        empty.Parent = configList
    end

    for index, name in ipairs(names) do
        local isSelected = (name == selectedConfigName)
        local button = Instance.new("TextButton")
        button.Name = "ConfigOption_" .. string.format("%04d", index)
        button.Size = UDim2.new(1, -4, 0, 30)
        button.BackgroundColor3 = Color3.new(1, 1, 1)
        button.BackgroundTransparency = isSelected and 0.86 or 1
        button.BorderSizePixel = 0
        button.Text = name
        button.TextColor3 = isSelected and UI.Theme.text or UI.Theme.sub
        button.Font = Enum.Font.GothamMedium
        button.TextSize = 12
        button.TextXAlignment = Enum.TextXAlignment.Left
        button.AutoButtonColor = false
        button.ZIndex = 4
        button.Parent = configList
        UI.corner(button, 8)

        local padding = Instance.new("UIPadding", button)
        padding.PaddingLeft = UDim.new(0, 12)

        button.MouseEnter:Connect(function()
            if name ~= selectedConfigName then
                UI.tween(button, 0.08, { BackgroundTransparency = 0.93 })
            end
        end)
        button.MouseLeave:Connect(function()
            UI.tween(button, 0.08, { BackgroundTransparency = (name == selectedConfigName) and 0.86 or 1 })
        end)
        button.MouseButton1Click:Connect(function()
            if SCRIPT_KILLED or not MENU_UNLOCKED then return end
            selectedConfigName = name
            configNameInput.Text = name
            configSelectedLabel.Text = "SELECTED: " .. name
            refreshConfigList()
        end)
    end

    configList.CanvasSize = UDim2.fromOffset(0, configLayout.AbsoluteContentSize.Y + 3)
    configNameInput.Text = selectedConfigName or ""
    configSelectedLabel.Text = selectedConfigName and ("SELECTED: " .. selectedConfigName) or "SELECTED: NONE"
end

configSaveButton.MouseButton1Click:Connect(function()
    if SCRIPT_KILLED or not MENU_UNLOCKED then return end

    local name = normalizeConfigName(configNameInput.Text)
    if name == "" then
        setConfigStatus("Enter a config name before saving")
        return
    end

    local record = makeConfigRecord(name, collectCurrentConfigSettings())
    savedConfigs[name] = record
    local persisted, persistError = persistConfigs()
    selectedConfigName = name
    configSelectedLabel.Text = "SELECTED: " .. name

    if persisted then
        setConfigStatus("Saved")
    else
        setConfigStatus("Saved for this session — " .. tostring(persistError or "persistence unavailable"))
    end
    refreshConfigList()
end)

configLoadButton.MouseButton1Click:Connect(function()
    if SCRIPT_KILLED or not MENU_UNLOCKED then return end

    local name = selectedConfigName
    if not name or name == "" then
        setConfigStatus("Select a saved config first")
        return
    end

    local reloadOk, reloadErr = loadPersistedConfigs()
    refreshConfigList()

    local record = savedConfigs[name]
    if not record then
        if not reloadOk and reloadErr then
            setConfigStatus(reloadErr)
        else
            setConfigStatus("Saved config not found")
        end
        return
    end

    selectedConfigName = name
    configNameInput.Text = name
    configSelectedLabel.Text = "SELECTED: " .. name

    local ok, err = applyConfigRecord(record)
    setConfigStatus(ok and "Loaded" or (err or "Load failed"))
end)

configDeleteButton.MouseButton1Click:Connect(function()
    if SCRIPT_KILLED or not MENU_UNLOCKED then return end
    if not selectedConfigName or selectedConfigName == "" then
        setConfigStatus("Select a saved config first")
        return
    end

    savedConfigs[selectedConfigName] = nil
    local deletedPersisted, persistError = persistConfigs()
    selectedConfigName = nil
    configNameInput.Text = ""
    configSelectedLabel.Text = "SELECTED: NONE"
    refreshConfigList()
    setConfigStatus(deletedPersisted and "Deleted" or ("Deleted for this session — " .. tostring(persistError or "persistence unavailable")))
end)

configExportButton.MouseButton1Click:Connect(function()
    if SCRIPT_KILLED or not MENU_UNLOCKED then return end

    local name = normalizeConfigName(configNameInput.Text)
    if name == "" then
        setConfigStatus("Enter a config name before exporting")
        return
    end

    local record = makeConfigRecord(name, collectCurrentConfigSettings())
    local okEncode, encoded = pcall(function()
        return HttpService:JSONEncode(record)
    end)
    if not okEncode then
        setConfigStatus("Export failed")
        return
    end

    configPasteInput.Text = encoded

    if type(setclipboard) == "function" then
        local copied = pcall(function()
            setclipboard(encoded)
        end)
        if copied then
            setConfigStatus("Copied")
        else
            setConfigStatus("JSON ready")
        end
    else
        setConfigStatus("JSON ready")
    end
end)

configImportButton.MouseButton1Click:Connect(function()
    if SCRIPT_KILLED or not MENU_UNLOCKED then return end

    local raw = tostring(configPasteInput.Text or "")
    if raw == "" then
        setConfigStatus("Paste JSON first")
        return
    end

    local okDecode, decoded = pcall(function()
        return HttpService:JSONDecode(raw)
    end)
    if not okDecode or type(decoded) ~= "table" then
        setConfigStatus("Invalid JSON")
        return
    end

    local normalized, err = validateAndNormalizeConfigRecord(decoded, nil)
    if not normalized then
        setConfigStatus(err or "Invalid config")
        return
    end

    local baseName = normalizeConfigName(normalized.name)
    if baseName == "" then
        setConfigStatus("Imported config needs a name")
        return
    end

    local name = baseName
    if savedConfigs[name] then
        local suffix = 2
        while savedConfigs[name] do
            name = baseName .. " " .. tostring(suffix)
            suffix = suffix + 1
        end
    end

    normalized.name = name
    savedConfigs[name] = normalized
    local importedPersisted, persistError = persistConfigs()
    selectedConfigName = name
    configNameInput.Text = name
    configSelectedLabel.Text = "SELECTED: " .. name
    refreshConfigList()

    local loaded, loadErr = applyConfigRecord(normalized)
    if loaded then
        if importedPersisted then
            setConfigStatus("Imported + loaded")
        else
            setConfigStatus("Imported + loaded for this session — " .. tostring(persistError or "persistence unavailable"))
        end
    else
        setConfigStatus(loadErr or (importedPersisted and "Imported" or "Imported for this session"))
    end
end)

local configsLoaded, configsLoadError = loadPersistedConfigs()
refreshConfigList()
if not configsLoaded and configsLoadError then
    setConfigStatus(configsLoadError)
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)

    if SCRIPT_KILLED or not MENU_UNLOCKED then return end

    if finishAimAssistBind(input) then
        return
    end

    if UI._espHandleKey and UI._espHandleKey(input) then
        return
    end

    -- Aim assist key (Hold / Toggle). No `return` here so the same key can also drive
    -- the colorbot / triggerbot if the user binds them to it.
    if AIM_ASSIST_ENABLED and AIM_BIND.matches(input)
        and not UserInputService:GetFocusedTextBox() then
        local isMouse = input.UserInputType ~= Enum.UserInputType.Keyboard
        if not (isMouse and gameProcessed) then
            if AIM_BIND.mode == "Toggle" then
                AIM_BIND.toggled = not AIM_BIND.toggled
                AIM_BIND.held = AIM_BIND.toggled
                if not AIM_BIND.held then
                    resetAimAssistState()
                end
            else
                AIM_BIND.held = true
            end
        end
    end

    if TRIGGER_BIND_LISTENING then
        if setTriggerBinding(input) then
            return
        end
    end

    if not COLORBOT_ENABLED then return end
    if gameProcessed then return end

    local matches = (TRIGGER_KEY and input.UserInputType == TRIGGER_KEY)
        or (TRIGGER_KEYCODE ~= Enum.KeyCode.Unknown and input.KeyCode == TRIGGER_KEYCODE)

    if matches then
        if TRIGGER_MODE == "Toggle" then
            TRIGGER_TOGGLED = not TRIGGER_TOGGLED
            TRIGGER_HELD = TRIGGER_TOGGLED
            if not TRIGGER_HELD then
                cancelHostTrigger()
                AIM_ASSIST_TARGET = nil
            end
        else
            TRIGGER_HELD = true
            TRIGGER_TOGGLED = false
        end
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if SCRIPT_KILLED or not MENU_UNLOCKED then return end
    if not COLORBOT_ENABLED then return end

    local matches = (TRIGGER_KEY and input.UserInputType == TRIGGER_KEY)
        or (TRIGGER_KEYCODE ~= Enum.KeyCode.Unknown and input.KeyCode == TRIGGER_KEYCODE)

    if matches and TRIGGER_MODE == "Hold" then
        TRIGGER_HELD = false
        TRIGGER_TOGGLED = false
        cancelHostTrigger()
        AIM_ASSIST_TARGET = nil
        if selectedActivator then
            target1Status.Text = "HOST: " .. selectedActivator.Name .. " — " .. (TRIGGER_MODE == "Toggle" and "toggle " or "hold ") .. TRIGGER_KEY_LABEL .. " before the shot."
        end
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if SCRIPT_KILLED or not MENU_UNLOCKED then return end
    if AIM_BIND.mode == "Hold" and AIM_BIND.held and AIM_BIND.matches(input) then
        AIM_BIND.held = false
        resetAimAssistState()
    end
end)

UserInputService.WindowFocusReleased:Connect(function()
    AIM_BIND.held = (AIM_BIND.mode == "Toggle") and AIM_BIND.toggled or false
end)

updateTriggerBindText = function()
    if triggerKeyButton and triggerKeyButton.Parent then
        triggerKeyButton.Text = TRIGGER_BIND_LISTENING and "PRESS A KEY..." or TRIGGER_KEY_LABEL
    end
end

setTriggerBinding = function(input)
    if input.UserInputType == Enum.UserInputType.Keyboard then
        if input.KeyCode == Enum.KeyCode.Unknown then
            return false
        end
        if input.KeyCode == Enum.KeyCode.Escape then
            -- ESC clears the keybind (leaves it unbound).
            TRIGGER_KEY = nil
            TRIGGER_KEYCODE = Enum.KeyCode.Unknown
            TRIGGER_KEY_LABEL = "NONE"
            TRIGGER_HELD, TRIGGER_TOGGLED, TRIGGER_BIND_LISTENING = false, false, false
            cancelHostTrigger()
            resetHostRound()
            updateTriggerBindText()
            if selectedActivator then
                target1Status.Text = "HOST: " .. selectedActivator.Name .. " — " .. (TRIGGER_MODE == "Toggle" and "toggle " or "hold ") .. TRIGGER_KEY_LABEL .. " before the shot."
            end
            return true
        end
        TRIGGER_KEY = nil
        TRIGGER_KEYCODE = input.KeyCode
        TRIGGER_KEY_LABEL = input.KeyCode.Name
    else
        TRIGGER_KEY = input.UserInputType
        TRIGGER_KEYCODE = Enum.KeyCode.Unknown
        TRIGGER_KEY_LABEL = input.UserInputType.Name
        if input.UserInputType == Enum.UserInputType.MouseButton2 then UI.rmbGuard = true end
    end

    TRIGGER_HELD = false
    cancelHostTrigger()
    resetHostRound()
    TRIGGER_TOGGLED = false
    TRIGGER_BIND_LISTENING = false
    updateTriggerBindText()
    if selectedActivator then
        target1Status.Text = "HOST: " .. selectedActivator.Name .. " — " .. (TRIGGER_MODE == "Toggle" and "toggle " or "hold ") .. TRIGGER_KEY_LABEL .. " before the shot."
    end
    return true
end

triggerKeyButton.MouseButton1Click:Connect(function()
    if SCRIPT_KILLED or not MENU_UNLOCKED then return end
    TRIGGER_BIND_LISTENING = true
    updateTriggerBindText()
end)

refreshHostUI = function()
    if SCRIPT_KILLED or not MENU_UNLOCKED or not whitelistGui.Parent then
        return
    end
    if UI.refreshAll then
        UI.refreshAll()
    end
end

if MENU_UNLOCKED then
    task.defer(refreshHostUI)
end


local function setUIKeybindFromInput(input)
    if UI_BIND_LISTENING ~= "HIDE" and UI_BIND_LISTENING ~= "KILL" then
        return false
    end

    if input.UserInputType ~= Enum.UserInputType.Keyboard
        or input.KeyCode == Enum.KeyCode.Unknown then
        return true
    end

    if input.KeyCode == Enum.KeyCode.Escape then
        -- ESC clears the keybind being set (leaves it unbound)
        local which = UI_BIND_LISTENING
        if which == "HIDE" then
            UI_HIDE_KEYCODE, UI_HIDE_KEY_LABEL = Enum.KeyCode.Unknown, "NONE"
        else
            UI_KILL_KEYCODE, UI_KILL_KEY_LABEL = Enum.KeyCode.Unknown, "NONE"
        end
        UI_BIND_LISTENING = nil
        UI_KEY_ASSIGN_IGNORE_UNTIL = os.clock() + 0.25
        if UI._uiKeybindStatus then
            UI._uiKeybindStatus.Text = (which == "HIDE" and "HIDE UI" or "KILL UI / SCRIPT") .. " key unbound."
        end
        if UI._refreshUIKeybindButtons then
            UI._refreshUIKeybindButtons()
        end
        return true
    end

    if UI_BIND_LISTENING == "HIDE" then
        if input.KeyCode == UI_KILL_KEYCODE then
            if UI._uiKeybindStatus then
                UI._uiKeybindStatus.Text = "That key is already the KILL key. Choose another."
            end
            return true
        end

        UI_HIDE_KEYCODE = input.KeyCode
        UI_HIDE_KEY_LABEL = input.KeyCode.Name
        UI_BIND_LISTENING = nil
        if UI._uiKeybindStatus then
            UI._uiKeybindStatus.Text = "HIDE UI key updated."
        end
    else
        if input.KeyCode == UI_HIDE_KEYCODE then
            if UI._uiKeybindStatus then
                UI._uiKeybindStatus.Text = "That key is already the HIDE key. Choose another."
            end
            return true
        end

        UI_KILL_KEYCODE = input.KeyCode
        UI_KILL_KEY_LABEL = input.KeyCode.Name
        UI_BIND_LISTENING = nil
        if UI._uiKeybindStatus then
            UI._uiKeybindStatus.Text = "KILL UI / SCRIPT key updated."
        end
    end

    UI_KEY_ASSIGN_IGNORE_UNTIL = os.clock() + 0.25

    if UI._refreshUIKeybindButtons then
        UI._refreshUIKeybindButtons()
    end
    return true
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if SCRIPT_KILLED or not MENU_UNLOCKED then
        return
    end

    if UI_BIND_LISTENING then
        setUIKeybindFromInput(input)
        return
    end

    if gameProcessed then
        return
    end

    if matchesKeyboardKey(input, UI_HIDE_KEYCODE) then
        whitelistGui.Enabled = not whitelistGui.Enabled

        if whitelistGui.Enabled and MENU_UNLOCKED then
            refreshHostUI()
        end
    end
end)

-- Player lists are only rebuilt while the menu is open; if players change while it is hidden the
-- lists are refreshed the moment it opens again (see the Enabled handler in the UI).
UI.deferRebuild = function(fn)
    if not fn then return end
    if whitelistGui.Enabled then
        task.defer(fn)
    else
        UI._listsDirty = true
    end
end

Players.PlayerAdded:Connect(function()
    UI.deferRebuild(TB_LIST_REBUILD)
    UI.deferRebuild(rebuildHostList)
end)
Players.PlayerRemoving:Connect(function(player)
    TB_SELECTED[player.UserId] = nil
    -- the chosen host left: drop the selection (stops the tracer watchers) like the original script did
    if player == selectedActivator then
        selectActivator(nil)
    end
    UI.deferRebuild(TB_LIST_REBUILD)
    UI.deferRebuild(rebuildHostList)
end)

refreshHostUI()
if updateLiveStatus then
    updateLiveStatus()
end

local getPlayerFromCharacter = Players.GetPlayerFromCharacter

UI.rawKnocked = function(character, humanoid)
    if not character or not humanoid or humanoid.Health <= 0 then
        return true
    end

    local bodyEffects = character:FindFirstChild("BodyEffects")
    if bodyEffects then
        local names = UI.KO_NAMES
        for i = 1, #names do
            local value = bodyEffects:FindFirstChild(names[i])
            if value then
                local ok, state = pcall(UI.koValue, value)
                if ok and state == true then
                    return true
                end
            end
        end
    end

    local names = UI.KO_NAMES
    for i = 1, #names do
        local ok, state = pcall(UI.koAttr, character, names[i])
        if ok and state == true then
            return true
        end
    end

    local state = humanoid:GetState()
    return state == Enum.HumanoidStateType.Dead
        or state == Enum.HumanoidStateType.FallingDown
        or state == Enum.HumanoidStateType.Physics
        or state == Enum.HumanoidStateType.PlatformStanding
end

-- Cached for ~2 frames per character: the full check does many FindFirstChild/pcall calls and
-- is asked for every player every frame by the aim assist and triggerbot.
UI._knockCache = setmetatable({}, { __mode = "k" })
local function isTargetKnocked(character, humanoid)
    if not character or not humanoid or humanoid.Health <= 0 then
        return true
    end
    local now = os.clock()
    local c = UI._knockCache[character]
    if c and now - c.t < 0.03 then
        return c.v
    end
    local v = UI.rawKnocked(character, humanoid)
    if c then
        c.t, c.v = now, v
    else
        UI._knockCache[character] = { t = now, v = v }
    end
    return v
end

local function isKatanaEquipped()
    if AIM_BIND.ownMelee() then
        return true
    end
    local character = LocalPlayer.Character
    if not character then
        return false
    end

    local tool = character:FindFirstChildOfClass("Tool")
    if not tool then
        return false
    end

    local name = string.lower(tool.Name)
    return name:find("katana", 1, true) ~= nil
        or name:find("sword", 1, true) ~= nil
end

-- Colorbot knock check: ON = skip knocked/dead targets, OFF = only skip missing/dead (0 hp).
UI.knock.colorbotKnocked = function(character, humanoid)
    if UI.knock.colorbot then
        return isTargetKnocked(character, humanoid)
    end
    return not character or not humanoid or humanoid.Health <= 0
end

local KO_Path
local Distance = UI.distance

local function Get_Path(Interchangeable, Names)
    local Found_Object = nil

    for _, Object in Interchangeable:GetDescendants() do
        if Object:IsA("BoolValue") or Object:IsA("IntValue") or Object:IsA("StringValue") then
            for _, Name in Names do
                if string.find(string.lower(Object.Name), string.lower(Name)) then
                    Found_Object = Object
                    break
                end
            end
            if Found_Object then
                break
            end
        end
    end

    if not Found_Object then
        return nil
    end

    local Path_Parts = {}
    local Current = Found_Object

    while Current and Current ~= game do
        if Current == Interchangeable then
            table.insert(Path_Parts, 1, "[Interchangeable]")
        else
            table.insert(Path_Parts, 1, Current.Name)
        end
        Current = Current.Parent
    end

    if Current == game then
        table.insert(Path_Parts, 1, "game")
    end

    return table.concat(Path_Parts, "__")
end

local function Use_Path(Path, Name)
    if not Path or Path == "" then
        return nil
    end

    local Path_Parts = {}
    for Part in string.gmatch(Path, "[^__]+") do
        table.insert(Path_Parts, Part)
    end

    local Current = game

    for _, Object in ipairs(Path_Parts) do
        if Object == "game" then
            Current = game
        elseif Object == "[Interchangeable]" then
            Current = Current and Current:FindFirstChild(Name)
        else
            if Current then
                Current = Current:FindFirstChild(Object)
                if not Current then
                    return nil
                end
            end
        end
    end

    return Current
end

local function KO_Flag(Target)
    if KO_Path and Target then
        local KO_Object = Use_Path(KO_Path, Target.Name)
        if KO_Object then
            return not KO_Object.Value
        end
    end

    local humanoid = Target and Target:FindFirstChildOfClass("Humanoid")
    return not isTargetKnocked(Target, humanoid)
end

local function Distance_Flag(TargetCharacter)
    local localCharacter = LocalPlayer.Character
    if not localCharacter or not TargetCharacter then
        return false
    end

    local localRoot = localCharacter:FindFirstChild("HumanoidRootPart")
    local targetRoot = TargetCharacter:FindFirstChild("HumanoidRootPart")
    local tool = localCharacter:FindFirstChildWhichIsA("Tool")
    if not localRoot or not targetRoot or not tool then
        return false
    end

    -- Which distance key a tool name matches never changes, so remember it (the value is still read
    -- fresh every time, so the sliders keep working).
    local keyCache = UI._distKeyCache
    if not keyCache then
        keyCache = {}
        UI._distKeyCache = keyCache
    end
    local matchedKey = keyCache[tool.Name]
    if matchedKey == nil then
        matchedKey = false
        local lowerName = string.lower(tool.Name)
        for Key in Distance do
            if string.find(lowerName, string.lower(Key)) then
                matchedKey = Key
                break
            end
        end
        keyCache[tool.Name] = matchedKey
    end
    local Max_Distance = Distance.Fallback
    if matchedKey then
        Max_Distance = Distance[matchedKey]
    end

    return (targetRoot.Position - localRoot.Position).Magnitude <= Max_Distance
end

task.spawn(function()
    local character = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
    KO_Path = Get_Path(character, {"k.o", "ko"})
end)

LocalPlayer.CharacterAdded:Connect(function(character)

    task.defer(function()
        KO_Path = Get_Path(character, {"k.o", "ko"})
    end)
end)

local function isTriggerbotTargetAllowed(player)
    if not player or player == LocalPlayer then
        return false
    end

    local selected = TB_SELECTED[player.UserId] == true

    if TB_LIST_MODE == "Blacklist" then
        return selected
    end

    return not selected
end

local function resolveTriggerbotPlayer(part)
    if not part or not part:IsA("BasePart") then
        return nil, nil
    end

    local character = part:FindFirstAncestorOfClass("Model")
    local ancestor = character

    while ancestor do
        if ancestor:IsA("Model") then
            local player = getPlayerFromCharacter(Players, ancestor)
            if player then
                return player, ancestor
            end
        end
        ancestor = ancestor.Parent
    end

    -- Some experiences decorate/replace character models or put hit parts
    -- inside extra models. Resolve by instance ancestry instead of relying
    -- on the character/display name, so tags, emojis and prefixes do not
    -- affect target detection.
    for _, player in ipairs(Players:GetPlayers()) do
        local playerCharacter = player.Character
        if playerCharacter and part:IsDescendantOf(playerCharacter) then
            return player, playerCharacter
        end
    end

    return nil, nil
end

-- Line-of-sight / center-ray helpers.
-- Skips things that should never block a shot: your own character, tracers/bullets,
-- invisible parts and non-collidable parts.
AIM_BIND.rayThrough = function(origin, direction, extraIgnore)
    local ignore = { LocalPlayer.Character }
    if extraIgnore then
        for _, inst in ipairs(extraIgnore) do
            ignore[#ignore + 1] = inst
        end
    end

    local params = UI._rayParams
    if not params then
        params = RaycastParams.new()
        params.FilterType = Enum.RaycastFilterType.Exclude
        UI._rayParams = params
    end

    for _ = 1, 8 do
        params.FilterDescendantsInstances = ignore
        local result = workspace:Raycast(origin, direction, params)
        if not result then
            return nil
        end
        local hit = result.Instance
        local model = hit:FindFirstAncestorOfClass("Model")
        local isPlayerPart = model ~= nil and model:FindFirstChildOfClass("Humanoid") ~= nil
        if isPlayerPart then
            return result
        end
        if hit.Transparency >= 0.9 or not hit.CanCollide or isLikelyTracerObject(hit) then
            ignore[#ignore + 1] = hit
        else
            return result
        end
    end
    return nil
end

-- Where the gun actually points: screen centre while the mouse is locked (aiming),
-- otherwise the mouse cursor. This lets the triggerbot work without holding right mouse.
AIM_BIND.aimPoint = function(camera)
    local viewport = camera.ViewportSize
    if UserInputService.MouseBehavior == Enum.MouseBehavior.LockCenter then
        return Vector2.new(viewport.X * 0.5, viewport.Y * 0.5)
    end
    return UserInputService:GetMouseLocation()
end

AIM_BIND.centerPart = function()
    local camera = workspace.CurrentCamera
    if not camera then return nil end
    local point = AIM_BIND.aimPoint(camera)
    local ray = camera:ViewportPointToRay(point.X, point.Y)
    local result = AIM_BIND.rayThrough(ray.Origin, ray.Direction * 2000)
    return result and result.Instance or nil
end

UI.tb.crosshairTarget = function()

    local target = AIM_BIND.centerPart() or Mouse.Target

    if not target or not target:IsA("BasePart") then
        return nil
    end

    local player, character = resolveTriggerbotPlayer(target)

    if not player or player == LocalPlayer or not character then
        return nil
    end

    if not isTriggerbotTargetAllowed(player) then
        return nil
    end

    if UI.knock.triggerbot and not KO_Flag(character) then
        return nil
    end

    if not Distance_Flag(character) then
        return nil
    end

    return target, player, character
end

UI.tb.parts = { "Head", "UpperTorso", "Torso", "HumanoidRootPart" }

-- FOV mode: any allowed player with a visible body part inside the circle around the crosshair.
UI.tb.fovTarget = function()
    local radius = UI.tb.fov
    if radius <= 0 then
        return nil
    end

    local camera = workspace.CurrentCamera
    if not camera then
        return nil
    end

    local center = AIM_BIND.aimPoint(camera)
    local bestPart, bestPlayer, bestCharacter = nil, nil, nil
    local bestDistance = math.huge

    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and isTriggerbotTargetAllowed(player) then
            local character = player.Character
            local humanoid = character and character:FindFirstChildOfClass("Humanoid")
            if character and humanoid and humanoid.Health > 0
                and (not UI.knock.triggerbot or KO_Flag(character)) and Distance_Flag(character) then
                for _, partName in ipairs(UI.tb.parts) do
                    local part = character:FindFirstChild(partName)
                    if part and part:IsA("BasePart") then
                        local screen, visible = camera:WorldToViewportPoint(part.Position)
                        if visible and screen.Z > 0 then
                            local distance = (Vector2.new(screen.X, screen.Y) - center).Magnitude
                            if distance <= radius and distance < bestDistance then
                                local blocked = AIM_BIND.rayThrough(
                                    camera.CFrame.Position,
                                    part.Position - camera.CFrame.Position,
                                    { character }
                                )
                                if not blocked then
                                    bestDistance = distance
                                    bestPart, bestPlayer, bestCharacter = part, player, character
                                end
                            end
                        end
                    end
                end
            end
        end
    end

    if bestPart then
        return bestPart, bestPlayer, bestCharacter
    end
    return nil
end

local function getTriggerbotTarget()
    local target, player, character = UI.tb.crosshairTarget()
    if target then
        return target, player, character
    end
    return UI.tb.fovTarget()
end

local function triggerbotCanFire()
    if SCRIPT_KILLED or not MENU_UNLOCKED or not TRIGGERBOT_ENABLED then
        return false
    end

    if isKatanaEquipped() then
        return false
    end

    -- No key bound: the switch alone keeps the triggerbot always on.
    if TB_KEY == nil and TB_KEYCODE == Enum.KeyCode.Unknown then
        return true
    end

    -- Optional key bound: Hold / Toggle decides when it is active.
    if TB_MODE == "Toggle" then
        return TB_TOGGLED
    end

    return TB_HELD
end

UI.tb.canFire = triggerbotCanFire

-- True while the free cursor is over the menu window. Scripted clicks must never be sent
-- then, otherwise they land on the menu's own buttons and toggles.
UI.mouseOverMenu = function()
    if not whitelistGui or not whitelistGui.Enabled or not wlCard then
        return false
    end
    if UserInputService.MouseBehavior == Enum.MouseBehavior.LockCenter then
        return false
    end
    local p = UserInputService:GetMouseLocation()
    local pos, size = wlCard.AbsolutePosition, wlCard.AbsoluteSize
    if p.X >= pos.X and p.X <= pos.X + size.X
        and p.Y >= pos.Y and p.Y <= pos.Y + size.Y then
        return true
    end
    local pv = UI.previewPanel
    if pv and pv.Visible then
        local pp, ps = pv.AbsolutePosition, pv.AbsoluteSize
        return p.X >= pp.X and p.X <= pp.X + ps.X
            and p.Y >= pp.Y and p.Y <= pp.Y + ps.Y
    end
    return false
end

local function fireStandaloneTriggerbot()
    if UI.mouseOverMenu() then
        UI.tb.pendingChar = nil
        UI.tb.state = "PAUSED (cursor over menu)"
        return false
    end
    if not triggerbotCanFire() then
        if not TRIGGERBOT_ENABLED then
            UI.tb.state = "OFF (master toggle is off)"
        elseif isKatanaEquipped() then
            UI.tb.state = "BLOCKED (melee / katana / sword equipped)"
        else
            UI.tb.state = "WAITING FOR TRIGGER KEY (" .. tostring(TB_MODE) .. ")"
        end
        return false
    end

    local target, _, targetCharacter = getTriggerbotTarget()
    if not target then
        UI.tb.pendingChar = nil
        local localChar = LocalPlayer.Character
        if not localChar or not localChar:FindFirstChildWhichIsA("Tool") then
            UI.tb.state = "ARMED - no gun/tool equipped (distance check needs one)"
        elseif TB_LIST_MODE == "Blacklist" then
            UI.tb.state = "ARMED - Blacklist mode only shoots SELECTED players"
        else
            UI.tb.state = "ARMED - no valid target"
        end
        return false
    end

    -- Strength 0 = zero delay (fires instantly, same as before).
    -- Above 0 the target must stay valid for the delay before each shot.
    local delaySeconds = UI.tb.delayMs() / 1000
    if delaySeconds > 0 then
        local clock = os.clock()
        if UI.tb.pendingChar ~= targetCharacter then
            UI.tb.pendingChar = targetCharacter
            UI.tb.pendingSince = clock
        end
        if clock - UI.tb.pendingSince < delaySeconds then
            UI.tb.state = "TARGET FOUND - waiting out delay"
            return false
        end
        UI.tb.pendingSince = clock
    end

    AIM_BIND.lastOwnClick = os.clock()
    UI.tb.state = "FIRING"
    if type(mouse1press) == "function" and type(mouse1release) == "function" then
        local pressOK = pcall(mouse1press)
        local releaseOK = pcall(mouse1release)
        if not (pressOK and releaseOK) then
            UI.tb.state = "FIRING - mouse1press/mouse1release errored"
        end
        return pressOK and releaseOK
    end

    if type(mouse1click) == "function" then
        local ok = pcall(mouse1click)
        if not ok then
            UI.tb.state = "FIRING - mouse1click errored"
        end
        return ok
    end

    local mousePos = UserInputService:GetMouseLocation()

    local vimOK = pcall(function()
        local vim = game:GetService("VirtualInputManager")
        vim:SendMouseButtonEvent(mousePos.X, mousePos.Y, 0, true, game, 0)
        vim:SendMouseButtonEvent(mousePos.X, mousePos.Y, 0, false, game, 0)
    end)
    if vimOK then
        return true
    end

    local camera = workspace.CurrentCamera
    local vuOK = pcall(function()
        local vu = game:GetService("VirtualUser")
        vu:Button1Down(Vector2.new(mousePos.X, mousePos.Y), camera and camera.CFrame)
        vu:Button1Up(Vector2.new(mousePos.X, mousePos.Y), camera and camera.CFrame)
    end)
    return vuOK
end

task.spawn(function()
    while not SCRIPT_KILLED do
        task.wait(0.25)
        if updateLiveStatus and whitelistGui.Enabled then pcall(updateLiveStatus) end
    end
end)

local function updateTriggerbotKeyInput(input, began)
    if TB_BIND_LISTENING or not TRIGGERBOT_ENABLED then
        return
    end

    local matches = false
    if TB_KEY ~= nil and input.UserInputType == TB_KEY then
        matches = true
    elseif TB_KEYCODE ~= Enum.KeyCode.Unknown and input.KeyCode == TB_KEYCODE then
        matches = true
    end

    if not matches then
        return
    end

    if began then
        if TB_MODE == "Toggle" then
            TB_TOGGLED = not TB_TOGGLED
            TB_HELD = TB_TOGGLED
        else
            TB_HELD = true
        end
    elseif TB_MODE == "Hold" then
        TB_HELD = false
    end
end

local function finishTriggerbotBind(input)
    if not TB_BIND_LISTENING then
        return false
    end

    if input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == Enum.KeyCode.Escape then
        -- ESC clears the keybind (leaves it unbound) instead of cancelling.
        TB_KEY = nil
        TB_KEYCODE = Enum.KeyCode.Unknown
        TB_KEY_LABEL = "NONE"
        TB_BIND_LISTENING = false
        TB_HELD, TB_TOGGLED = false, false
        if TB_PAGE_REFRESH then TB_PAGE_REFRESH() end
        return true
    end

    if input.UserInputType == Enum.UserInputType.Keyboard then
        if input.KeyCode == Enum.KeyCode.Unknown then
            return true
        end
        TB_KEY = nil
        TB_KEYCODE = input.KeyCode
        TB_KEY_LABEL = input.KeyCode.Name
    elseif input.UserInputType ~= Enum.UserInputType.MouseMovement
        and input.UserInputType ~= Enum.UserInputType.MouseWheel then
        TB_KEY = input.UserInputType
        TB_KEYCODE = Enum.KeyCode.Unknown
        TB_KEY_LABEL = input.UserInputType.Name
        if input.UserInputType == Enum.UserInputType.MouseButton2 then UI.rmbGuard = true end
    else
        return false
    end

    TB_HELD, TB_TOGGLED, TB_BIND_LISTENING = false, false, false
    if TB_PAGE_REFRESH then TB_PAGE_REFRESH() end
    return true
end

local function getToggleTarget(camera)
    local viewport = camera.ViewportSize
    local screenCenter = Vector2.new(viewport.X * 0.5, viewport.Y * 0.5)
    local bestPart = nil
    local bestDistance = math.huge

    for _, candidate in ipairs(Players:GetPlayers()) do
        if candidate ~= LocalPlayer then
            local character = candidate.Character
            local humanoid = character and character:FindFirstChildOfClass("Humanoid")
            local root = character and character:FindFirstChild("HumanoidRootPart")
            local torso = character and (
                character:FindFirstChild("UpperTorso")
                or character:FindFirstChild("Torso")
                or root
            )

            if humanoid and not isTargetKnocked(character, humanoid) and torso and torso:IsA("BasePart") then
                local screenPos, visible = camera:WorldToViewportPoint(torso.Position)
                if visible and screenPos.Z > 0 then
                    local distance = (Vector2.new(screenPos.X, screenPos.Y) - screenCenter).Magnitude
                    if distance < bestDistance then
                        local direction = torso.Position - camera.CFrame.Position
                        local blocked = AIM_BIND.rayThrough(
                            camera.CFrame.Position,
                            direction,
                            { character }
                        )

                        if not blocked then
                            bestDistance = distance
                            bestPart = torso
                        end
                    end
                end
            end
        end
    end

    return bestPart
end

local AIM_BODY_PARTS = {
    {name = "Head", weight = 1.00, speedBias = 0.00, moveBias = 0.00},
    {name = "UpperTorso", weight = 1.10, speedBias = 0.20, moveBias = 0.20},
    {name = "Torso", weight = 1.10, speedBias = 0.20, moveBias = 0.20},
    {name = "LowerTorso", weight = 0.95, speedBias = 0.35, moveBias = 0.15},
    {name = "HumanoidRootPart", weight = 0.72, speedBias = 0.55, moveBias = 0.10},
    {name = "LeftUpperArm", weight = 0.70, speedBias = 0.85, moveBias = 0.35},
    {name = "RightUpperArm", weight = 0.70, speedBias = 0.85, moveBias = 0.35},
    {name = "LeftLowerArm", weight = 0.55, speedBias = 1.00, moveBias = 0.40},
    {name = "RightLowerArm", weight = 0.55, speedBias = 1.00, moveBias = 0.40},
    {name = "LeftUpperLeg", weight = 0.68, speedBias = 0.90, moveBias = 0.55},
    {name = "RightUpperLeg", weight = 0.68, speedBias = 0.90, moveBias = 0.55},
    {name = "LeftLowerLeg", weight = 0.48, speedBias = 1.00, moveBias = 0.65},
    {name = "RightLowerLeg", weight = 0.48, speedBias = 1.00, moveBias = 0.65},
    {name = "LeftHand", weight = 0.45, speedBias = 1.00, moveBias = 0.40},
    {name = "RightHand", weight = 0.45, speedBias = 1.00, moveBias = 0.40},
    {name = "LeftFoot", weight = 0.40, speedBias = 1.00, moveBias = 0.65},
    {name = "RightFoot", weight = 0.40, speedBias = 1.00, moveBias = 0.65},
    {name = "Left Arm", weight = 0.65, speedBias = 0.90, moveBias = 0.38},
    {name = "Right Arm", weight = 0.65, speedBias = 0.90, moveBias = 0.38},
    {name = "Left Leg", weight = 0.60, speedBias = 0.90, moveBias = 0.58},
    {name = "Right Leg", weight = 0.60, speedBias = 0.90, moveBias = 0.58},
}

local AIM_TARGET_RNG = Random.new()
-- Prediction tuning (independent of the strength setting).
-- predictionScale: how far ahead of the body the aim point leads (lower = less lead).
-- lateralLead: extra sideways lead applied while the camera is rotating onto the target.
-- maxResist: at strength 100, the fraction of your mouse movement AWAY from the target
-- that gets cancelled each frame.
-- breakAngle / releaseTime: once you pull away by about breakAngle degrees (steady push or a
-- flick) the lock lets go, then fades back in over releaseTime seconds. Raise breakAngle for
-- a stickier lock, lower it for an easier break.
-- (all kept in one table to stay under Lua's local-variable limit)
local AIM_TUNE = {
    predictionScale = 0.45,
    lateralLead = 0.0018,
    maxResist = 0.75,
    breakAngle = 2.2,   -- degrees of sustained pull-away (at strength 100) that breaks the lock
    releaseTime = 0.45, -- seconds the lock stays weak after you break away, then fades back in
    push = 0,
    releaseUntil = 0,
    releaseStart = 0,
    lastLook = nil,
    lastLookChar = nil,
}
local AIM_LAST_PART = nil
local AIM_LAST_TARGET = nil
local AIM_PART_CHANGE_AT = 0
local AIM_TARGET_HOLD_UNTIL = 0

local function getAimStrengthLevel()

    return math.clamp(AIM_ASSIST_STRENGTH * 1000, 0, 100)
end

local function getAimCorrectionFactor()
    local level = getAimStrengthLevel()
    if level <= 0 then
        return 0
    end

    local normalized = level / 100
    return math.clamp(normalized ^ 0.82, 0, 1)
end

local function getBodyPartCandidates(character)
    -- Reuse the last lookup for the same character (revalidated every call, refreshed every 0.5s)
    -- instead of ~20 FindFirstChild calls + new tables every frame.
    local cache = UI._candCache
    if cache and cache.character == character and os.clock() - cache.at < 0.5 then
        local list = cache.list
        local valid = true
        for i = 1, #list do
            if list[i].part.Parent ~= character then
                valid = false
                break
            end
        end
        if valid then
            return list
        end
    end

    local result = {}
    for _, info in ipairs(AIM_BODY_PARTS) do
        local part = character:FindFirstChild(info.name)
        if part and part:IsA("BasePart") then
            result[#result + 1] = {part = part, info = info}
        end
    end
    UI._candCache = { character = character, at = os.clock(), list = result }
    return result
end

local function getPredictedAimPosition(part, humanoid, camera)
    local character = part.Parent
    local root = character and character:FindFirstChild("HumanoidRootPart")
    if not root then
        return part.Position
    end

    local velocity = root.AssemblyLinearVelocity
    local horizontalVelocity = Vector3.new(velocity.X, 0, velocity.Z)
    local speed = horizontalVelocity.Magnitude
    local distance = (root.Position - camera.CFrame.Position).Magnitude
    local lead = math.clamp(
        0.008 + speed * 0.00072 + distance * 0.000024,
        0.008,
        0.052
    )
    -- Prediction is the SAME at every strength setting (strength only changes how hard it locks).
    lead = lead * AIM_TUNE.predictionScale

    local localCharacter = LocalPlayer.Character
    local localRoot = localCharacter and localCharacter:FindFirstChild("HumanoidRootPart")
    if localRoot then
        local ownVelocity = Vector3.new(
            localRoot.AssemblyLinearVelocity.X,
            0,
            localRoot.AssemblyLinearVelocity.Z
        )

        horizontalVelocity = horizontalVelocity - ownVelocity * 0.35
        speed = horizontalVelocity.Magnitude
    end

    return part.Position + horizontalVelocity * lead
end

local function chooseMovingBodyPart(candidates, targetHorizontal, camera, localVelocity, strength)
    if #candidates == 0 then
        return nil
    end

    local screenMove = targetHorizontal:Dot(camera.CFrame.RightVector)
    local localScreenMove = localVelocity:Dot(camera.CFrame.RightVector)
    local movement = math.abs(screenMove) + math.abs(localScreenMove) * 0.35

    local weights = {}
    local total = 0

    for _, entry in ipairs(candidates) do
        local info = entry.info
        local weight = 0.55 + info.weight * 0.55

        if movement > 2 then
            weight = weight + info.speedBias * math.clamp(movement / 24, 0, 1) * 1.35
            weight = weight + info.moveBias * math.clamp(movement / 18, 0, 1) * 0.75

            if screenMove > 1.5 then
                if string.find(info.name, "Right") then
                    weight = weight + 0.45
                elseif string.find(info.name, "Left") then
                    weight = weight + 0.10
                end
            elseif screenMove < -1.5 then
                if string.find(info.name, "Left") then
                    weight = weight + 0.45
                elseif string.find(info.name, "Right") then
                    weight = weight + 0.10
                end
            end
        else

            if info.name == "Head" then
                weight = weight + 0.30
            elseif info.name == "UpperTorso" or info.name == "Torso" then
                weight = weight + 0.16
            end
        end

        weight = weight * (0.72 + strength * 0.28)
        weights[#weights + 1] = {entry = entry, weight = math.max(weight, 0.01)}
        total = total + math.max(weight, 0.01)
    end

    local roll = AIM_TARGET_RNG:NextNumber(0, total)
    local running = 0
    for _, item in ipairs(weights) do
        running = running + item.weight
        if roll <= running then
            return item.entry.part
        end
    end

    return weights[#weights].entry.part
end

local function getAimAssistTarget(camera)
    local viewport = camera.ViewportSize
    local screenCenter = Vector2.new(viewport.X * 0.5, viewport.Y * 0.5)
    local bestCharacter = nil
    local bestRootDistance = math.huge
    local bestPart = nil
    local radius = AIM_ASSIST_RADIUS
    local now = os.clock()

    local localCharacter = LocalPlayer.Character
    local localRoot = localCharacter and localCharacter:FindFirstChild("HumanoidRootPart")
    local localVelocity = localRoot
        and Vector3.new(localRoot.AssemblyLinearVelocity.X, 0, localRoot.AssemblyLinearVelocity.Z)
        or Vector3.zero

    for _, candidate in ipairs(Players:GetPlayers()) do
        if candidate ~= LocalPlayer then
            local character = candidate.Character
            local humanoid = character and character:FindFirstChildOfClass("Humanoid")
            local root = character and character:FindFirstChild("HumanoidRootPart")

            if humanoid and root and not isTargetKnocked(character, humanoid) then
                local rootPos = root.Position
                local rootScreen, visible = camera:WorldToViewportPoint(rootPos)

                if visible and rootScreen.Z > 0 then
                    local rootDistance =
                        (Vector2.new(rootScreen.X, rootScreen.Y) - screenCenter).Magnitude

                    if rootDistance <= radius + 2 then
                        local targetVelocity = root.AssemblyLinearVelocity
                        local targetHorizontal = Vector3.new(
                            targetVelocity.X, 0, targetVelocity.Z
                        )

                        local score = rootDistance
                        if AIM_LAST_TARGET and AIM_LAST_TARGET.Parent == character then
                            score = score - 10
                        end
                        if targetHorizontal.Magnitude > 2 then
                            score = score - math.clamp(targetHorizontal.Magnitude * 0.06, 0, 4)
                        end

                        if score < bestRootDistance then
                            bestRootDistance = score
                            bestCharacter = character
                        end
                    end
                end
            end
        end
    end

    if bestCharacter then
        local humanoid = bestCharacter:FindFirstChildOfClass("Humanoid")
        local root = bestCharacter:FindFirstChild("HumanoidRootPart")
        local candidates = getBodyPartCandidates(bestCharacter)

        if humanoid and root and #candidates > 0 then
            local targetVelocity = root.AssemblyLinearVelocity
            local targetHorizontal = Vector3.new(
                targetVelocity.X, 0, targetVelocity.Z
            )

            local selectedPart = nil
            local closestPartDistance = math.huge

            for _, entry in ipairs(candidates) do
                local part = entry.part
                local group = AIM_BIND.partGroup[part.Name]
                if not group or not AIM_BIND.parts[group] then
                    continue
                end
                local partScreen, partVisible = camera:WorldToViewportPoint(part.Position)

                if partVisible and partScreen.Z > 0 then
                    local partDistance = (
                        Vector2.new(partScreen.X, partScreen.Y) - screenCenter
                    ).Magnitude

                    if partDistance < closestPartDistance then
                        closestPartDistance = partDistance
                        selectedPart = part
                    end
                end
            end

            if selectedPart then
                local aimPosition = getPredictedAimPosition(selectedPart, humanoid, camera)
                local screenPos, visible = camera:WorldToViewportPoint(aimPosition)

                if visible and screenPos.Z > 0 then
                    local screenDistance =
                        (Vector2.new(screenPos.X, screenPos.Y) - screenCenter).Magnitude

                    if screenDistance <= radius + 3 then
                        local direction = aimPosition - camera.CFrame.Position
                        local blocked = AIM_BIND.rayThrough(
                            camera.CFrame.Position,
                            direction,
                            { bestCharacter }
                        )

                        if not blocked then
                            bestPart = selectedPart
                            AIM_LAST_TARGET = bestCharacter:FindFirstChild("HumanoidRootPart")
                        end
                    end
                end
            end
        end
    end

    if bestPart then
        if bestPart ~= AIM_LAST_PART then
            AIM_LAST_PART = bestPart
            local strength = getAimStrengthLevel() / 100
            AIM_PART_CHANGE_AT = now + AIM_TARGET_RNG:NextNumber(
                0.10 - strength * 0.025,
                0.22 - strength * 0.045
            )
        end
        AIM_TARGET_HOLD_UNTIL = now + 0.14
    elseif AIM_LAST_PART and now < AIM_TARGET_HOLD_UNTIL and AIM_LAST_PART.Parent then
        bestPart = AIM_LAST_PART
    end

    return bestPart
end

resetAimAssistState = function()
    AIM_ASSIST_TARGET = nil
    AIM_TUNE.lastLook = nil
    AIM_TUNE.lastLookChar = nil
    AIM_TUNE.push = 0
    AIM_TUNE.releaseUntil = 0
    AIM_LAST_PART = nil
    AIM_LAST_TARGET = nil
    AIM_PART_CHANGE_AT = 0
    AIM_TARGET_HOLD_UNTIL = 0
end

AIM_TUNE.core = function(camera, targetPart, dt)
    if not targetPart or not targetPart.Parent then
        return false
    end

    local correction = getAimCorrectionFactor()
    if correction <= 0 then
        return false
    end

    local character = targetPart.Parent
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if not humanoid or isTargetKnocked(character, humanoid) then
        return false
    end

    local desiredPosition = getPredictedAimPosition(targetPart, humanoid, camera)

    -- Stickiness (camlock feel). "lock" is 0 at low strength and ramps smoothly to 1 at 100
    -- (correction^2), so there is no sudden jump from legit to blatant.
    local lock = correction * correction
    local nowT = os.clock()

    -- "grip" is 1 normally, drops to ~0.08 right after you break away from the target, then
    -- fades back to 1, so a deliberate pull-away always works even at strength 100.
    local grip = 1
    if nowT < AIM_TUNE.releaseUntil then
        local left = (AIM_TUNE.releaseUntil - nowT) / AIM_TUNE.releaseTime
        grip = 1 - math.clamp(left, 0, 1) * 0.92
    end

    -- Resist mouse movement away from the target: compare where we left the camera last frame
    -- with where it is now. Any increase in angle to the target was caused by you moving the
    -- mouse, so cancel a strength-scaled fraction of it by rotating back toward the target.
    if AIM_TUNE.lastLook and AIM_TUNE.lastLookChar == character then
        local camPos0 = camera.CFrame.Position
        local want0 = (desiredPosition - camPos0)
        if want0.Magnitude > 1e-3 then
            want0 = want0.Unit
            local cur0 = camera.CFrame.LookVector
            local errBefore = math.acos(math.clamp(AIM_TUNE.lastLook:Dot(want0), -1, 1))
            local errNow = math.acos(math.clamp(cur0:Dot(want0), -1, 1))
            local away = errNow - errBefore

            -- track how hard/long you are pulling away (decays when you stop)
            if away > 0 then
                AIM_TUNE.push = AIM_TUNE.push * 0.85 + away
            else
                AIM_TUNE.push = AIM_TUNE.push * 0.6
            end
            -- the more lock, the more push it takes to break, but it is always breakable
            local breakRad = math.rad(AIM_TUNE.breakAngle) * (0.5 + 0.5 * lock)
            if AIM_TUNE.push > breakRad and nowT >= AIM_TUNE.releaseUntil then
                AIM_TUNE.releaseUntil = nowT + AIM_TUNE.releaseTime
                AIM_TUNE.push = 0
                grip = 0.08
            end

            if away > 1e-5 then
                local back = math.min(away * lock * AIM_TUNE.maxResist * grip, errNow)
                local ax = cur0:Cross(want0)
                if ax.Magnitude > 1e-5 then
                    camera.CFrame = CFrame.lookAt(
                        camPos0,
                        camPos0 + CFrame.fromAxisAngle(ax.Unit, back) * cur0
                    )
                end
            end
        end
    end

    local viewport = camera.ViewportSize
    local center = Vector2.new(viewport.X * 0.5, viewport.Y * 0.5)
    local screenPosition, visible = camera:WorldToViewportPoint(desiredPosition)
    if not visible or screenPosition.Z <= 0 then
        return false
    end

    local screenDelta = Vector2.new(screenPosition.X, screenPosition.Y) - center
    local pixelDistance = screenDelta.Magnitude
    if pixelDistance > AIM_ASSIST_RADIUS + 1 then
        return false
    end

    local deadzone = math.clamp(5.0 - correction * 3.8, 1.2, 5.0)
    if pixelDistance <= deadzone then
        return true
    end
    local cameraPosition = camera.CFrame.Position
    local desiredDirection = (desiredPosition - cameraPosition).Unit
    local currentDirection = camera.CFrame.LookVector

    local dot = math.clamp(currentDirection:Dot(desiredDirection), -1, 1)
    local angle = math.acos(dot)
    if angle < math.rad(0.05) then
        return true
    end

    local localCharacter = LocalPlayer.Character
    local localRoot = localCharacter and localCharacter:FindFirstChild("HumanoidRootPart")
    local ownVelocity = localRoot
        and Vector3.new(localRoot.AssemblyLinearVelocity.X, 0, localRoot.AssemblyLinearVelocity.Z)
        or Vector3.zero

    local targetRoot = character:FindFirstChild("HumanoidRootPart")
    if targetRoot then
        local targetVelocity = Vector3.new(
            targetRoot.AssemblyLinearVelocity.X,
            0,
            targetRoot.AssemblyLinearVelocity.Z
        )
        local relativeVelocity = targetVelocity - ownVelocity * 0.35
        local lateral = relativeVelocity - currentDirection * relativeVelocity:Dot(currentDirection)
        local leadStrength = AIM_TUNE.lateralLead

        if lateral.Magnitude > 0.5 then
            desiredDirection = (desiredDirection + lateral * leadStrength).Unit
            dot = math.clamp(currentDirection:Dot(desiredDirection), -1, 1)
            angle = math.acos(dot)
        end
    end

    local response = math.clamp(
        (2.0 + correction * 18.0) * dt,
        0,
        0.50
    )

    local distanceFactor = math.clamp(
        pixelDistance / math.max(AIM_ASSIST_RADIUS, 1),
        0,
        1
    )
    local proximityFactor = 1 - distanceFactor

    local correctionAlpha = math.clamp(
        response * (0.17 + proximityFactor * 0.62 + correction * 0.20)
            + lock * (0.10 + proximityFactor * 0.16),
        0,
        0.52 + lock * 0.30
    )

    local maxAngle = math.rad(0.10 + correction * 2.35 + lock * 3.5)
        * math.clamp(dt * 60, 0.5, 1.5)

    local appliedAngle = math.min(angle * correctionAlpha, maxAngle) * grip
    if appliedAngle <= 0 then
        return true
    end

    local axis = currentDirection:Cross(desiredDirection)
    if axis.Magnitude < 1e-5 then
        return true
    end

    axis = axis.Unit
    local rotation = CFrame.fromAxisAngle(axis, appliedAngle)
    camera.CFrame = CFrame.lookAt(
        cameraPosition,
        cameraPosition + rotation * currentDirection
    )
    return true
end

local function applyAimAssistCorrection(camera, targetPart, dt)
    local result = AIM_TUNE.core(camera, targetPart, dt)
    -- remember where we left the camera so next frame can tell how much the mouse moved it
    AIM_TUNE.lastLook = camera.CFrame.LookVector
    AIM_TUNE.lastLookChar = targetPart and targetPart.Parent or nil
    return result
end

fireTriggeredShot = function()
    if SCRIPT_KILLED or not MENU_UNLOCKED or not selectedActivator then
        return nil
    end

    -- Check if user is dead/knocked - stop firing
    stopColorbotIfDead()
    if isLocalPlayerKnockedOrDead() then
        return nil
    end

    if not AIM_BIND.colorbotOn() then
        return nil
    end

    if AIM_BIND.ownMelee() then
        cancelHostTrigger()
        return nil
    end

    if hostHasMeleeWeapon(selectedActivator) then
        cancelHostTrigger()
        return nil
    end

    local camera = workspace.CurrentCamera
    if not camera then return nil end

    local character
    local player
    local viewport = camera.ViewportSize
    local ray = camera:ViewportPointToRay(viewport.X * 0.5, viewport.Y * 0.5)
    -- The host's own tracer sits on the line between you and the host, so the
    -- ray must look straight through tracers / own character or it never "sees" the host.
    local result = AIM_BIND.rayThrough(ray.Origin, ray.Direction * 2000)
    local target = result and result.Instance or Mouse.Target

    local function resolvePlayerFromPart(part)
        if not part or not part:IsA("BasePart") then
            return nil, nil
        end

        local ancestor = part:FindFirstAncestorOfClass("Model")
        while ancestor do
            if ancestor:IsA("Model") then
                local candidate = Players:GetPlayerFromCharacter(ancestor)
                if candidate then
                    return candidate, ancestor
                end
            end
            ancestor = ancestor.Parent
        end

        -- Do not depend on the visible/displayed name. Custom tags, emojis,
        -- prefixes and decorated nametags can change what is shown without
        -- changing which Player owns the character.
        for _, candidate in ipairs(Players:GetPlayers()) do
            local candidateCharacter = candidate.Character
            if candidateCharacter and part:IsDescendantOf(candidateCharacter) then
                return candidate, candidateCharacter
            end
        end

        return nil, nil
    end

    local rayPlayer, rayCharacter = resolvePlayerFromPart(target)

    if rayPlayer and rayCharacter then
        player = rayPlayer
        character = rayCharacter
    elseif TRIGGER_MODE == "Toggle" then
        local togglePart = getToggleTarget(camera)
        local togglePlayer, toggleCharacter = resolvePlayerFromPart(togglePart)

        if togglePlayer and toggleCharacter then
            player = togglePlayer
            character = toggleCharacter
        end
    else
        local assistPart = AIM_ASSIST_TARGET
        if not assistPart or not assistPart.Parent then
            assistPart = getAimAssistTarget(camera)
        end

        local assistPlayer, assistCharacter = resolvePlayerFromPart(assistPart)
        if assistPlayer and assistCharacter then
            local direction = assistPart.Position - camera.CFrame.Position
            local blocked = AIM_BIND.rayThrough(
                camera.CFrame.Position,
                direction,
                { assistCharacter }
            )

            if not blocked then
                player = assistPlayer
                character = assistCharacter
            end
        end
    end

    if not character or not player or player == LocalPlayer then
        return nil
    end

    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if not humanoid or UI.knock.colorbotKnocked(character, humanoid) then
        cancelHostTrigger()
        return nil
    end

    if UI.mouseOverMenu() then
        return nil
    end

    AIM_BIND.lastOwnClick = os.clock()
    mouse1press()
    mouse1release()

    return player, character, humanoid
end

startHostBurst = function(token)
    if hostBurstActive then return end

    local function runBurst(character)
        hostBurstActive = true
        hostBurstTarget = character
        if HOST_ROUND_TARGET == nil then
            HOST_ROUND_TARGET = character
        end

        task.spawn(function()
            while hostBurstActive do
                if SCRIPT_KILLED or not MENU_UNLOCKED then
                    break
                end

                -- Check if user is dead/knocked
                if isLocalPlayerKnockedOrDead() then
                    stopColorbotIfDead()
                    break
                end

                if not AIM_BIND.colorbotOn() or AIM_BIND.ownMelee() then
                    break
                end

                if token ~= hostDelayToken then
                    break
                end

                local targetHumanoid = hostBurstTarget and hostBurstTarget:FindFirstChildOfClass("Humanoid")
                if not targetHumanoid or UI.knock.colorbotKnocked(hostBurstTarget, targetHumanoid) then
                    resetHostRound()
                    break
                end

                -- host pulled out a knife mid-burst: stop
                if hostHasMeleeWeapon(selectedActivator) then
                    break
                end

                if UI.mouseOverMenu() then
                    break
                end

                -- Several rapid clicks per frame. The single yield below is required so the
                -- loop can never hang the client.
                local CLICKS_PER_FRAME = 6
                for _ = 1, CLICKS_PER_FRAME do
                    AIM_BIND.lastOwnClick = os.clock()
                    mouse1press()
                    mouse1release()
                end
                task.wait()
            end

            -- Never leave the mouse button held down after a burst.
            pcall(mouse1release)

            if token == hostDelayToken then
                hostBurstActive = false
                hostBurstTarget = nil
                HOST_TRIGGER_ARMED = false
                activatorPulseUntil = 0
                activatorFireAt = 0
                AIM_ASSIST_TARGET = nil
            end
        end)
    end

    -- First attempt: instantly, in the same frame the host's tracer is detected.
    local player, character, humanoid = fireTriggeredShot()
    if player and character and humanoid then
        runBurst(character)
        return
    end

    -- fireTriggeredShot already cancelled the round (melee / knocked target).
    if token ~= hostDelayToken then
        return
    end

    -- Not on target yet: keep checking every frame for a short window so the shot goes
    -- out the moment the crosshair reaches the target, instead of waiting for another tracer.
    task.spawn(function()
        local deadline = os.clock() + 0.35
        while os.clock() < deadline do
            task.wait()
            if token ~= hostDelayToken or hostBurstActive then
                return
            end
            if SCRIPT_KILLED or not MENU_UNLOCKED or not HOST_TRIGGER_ARMED
                or not AIM_BIND.colorbotOn() then
                cancelHostTrigger()
                return
            end

            local p, c, h = fireTriggeredShot()
            if p and c and h then
                runBurst(c)
                return
            end
            if token ~= hostDelayToken then
                return
            end
        end

        if token == hostDelayToken then
            cancelHostTrigger()
        end
    end)
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if TB_BIND_LISTENING then
        finishTriggerbotBind(input)
        return
    end

    updateTriggerbotKeyInput(input, true)
end)

UserInputService.InputEnded:Connect(function(input)
    updateTriggerbotKeyInput(input, false)
end)

UserInputService.WindowFocusReleased:Connect(function()
    TB_HELD = false
    if TB_MODE == "Toggle" then

        TB_HELD = TB_TOGGLED
    end
end)

RunService:BindToRenderStep("Triggerbot_Late", Enum.RenderPriority.Last.Value, function()
    if SCRIPT_KILLED or not MENU_UNLOCKED then return end
    
    -- Auto-disable colorbot if user is dead (guarded so it can never block the triggerbot)
    pcall(stopColorbotIfDead)

    fireStandaloneTriggerbot()
end)

RunService:BindToRenderStep("Triggerbot_RenderStep", Enum.RenderPriority.Input.Value, function(dt)
    if SCRIPT_KILLED or not MENU_UNLOCKED then return end

    -- Check if user died while colorbot is active
    pcall(stopColorbotIfDead)

    local aimAssistActive = AIM_BIND.isActive()

    if aimAssistActive then
        local assistCamera = workspace.CurrentCamera
        if assistCamera then
            if AIM_ASSIST_TARGET then
                local oldCharacter = AIM_ASSIST_TARGET.Parent
                local oldHumanoid = oldCharacter and oldCharacter:FindFirstChildOfClass("Humanoid")

                if not oldHumanoid or isTargetKnocked(oldCharacter, oldHumanoid) then
                    AIM_ASSIST_TARGET = nil
                end
            end

            local selectedPart = getAimAssistTarget(assistCamera)
            if selectedPart then
                AIM_ASSIST_TARGET = selectedPart
                AIM_LAST_TARGET = selectedPart
                applyAimAssistCorrection(assistCamera, AIM_ASSIST_TARGET, dt)
            else
                AIM_ASSIST_TARGET = nil
                AIM_LAST_TARGET = nil
            end
        end
    else
        resetAimAssistState()
    end

    if not COLORBOT_ENABLED or not selectedActivator then
        return
    end
end)

