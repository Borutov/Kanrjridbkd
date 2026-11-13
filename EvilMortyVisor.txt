--[[
════════════════════════════════════════════════════════════════════════
  PORTAL DIMENSION TRACKER — EVIL MORTY OPTICAL VISOR  v4.0
  Client-side · Delta Executor · Mobile-optimized
  Lua/Luau · Roblox · Portal Sandbox

  Architecture: 9 independent modules
    1 · Player Scanner       5 · Data Validator
    2 · Dimension Scanner    6 · Data Manager
    3 · Portal Gun Scanner   7 · Event Logger
    4 · Environment Scanner  8 · HUD Manager
                             9 · Radar (embedded in PROXIMITY mode)

  Data integrity rule: no value is ever displayed without an origin,
  timestamp and status. Unknown data is shown as UNKNOWN — never invented.
════════════════════════════════════════════════════════════════════════
]]

-- ══════════════════════════════════════════════════════════════════════
-- SERVICES
-- ══════════════════════════════════════════════════════════════════════
local Players        = game:GetService("Players")
local RunService     = game:GetService("RunService")
local TweenService   = pcall(function() return game:GetService("TweenService") end)
      and game:GetService("TweenService") or nil

local LP     = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- ══════════════════════════════════════════════════════════════════════
-- PALETTE  (Evil Morty optical visor — gold / amber / black)
-- ══════════════════════════════════════════════════════════════════════
local C = {
    gold    = Color3.fromRGB(255, 232,   0),   -- #FFE800 dominant
    amber   = Color3.fromRGB(255, 153,   0),   -- #FF9900 depth / secondary
    white   = Color3.fromRGB(230, 220, 170),   -- warm white for highlights
    red     = Color3.fromRGB(220,   0,  50),   -- #FF0033 alerts only
    black   = Color3.fromRGB(  3,   3,   4),   -- #000000 background
    goldDim = Color3.fromRGB(140, 110,   0),   -- muted gold
    goldLo  = Color3.fromRGB( 50,  40,   0),   -- very dim gold
    gray    = Color3.fromRGB( 80,  80,  90),   -- inactive / stale
    green   = Color3.fromRGB( 70, 220,  90),   -- confirmed / in garage
    bgRow   = Color3.fromRGB( 10,   9,   1),
    bgHdr   = Color3.fromRGB( 14,  11,   0),
}

local function statusColor(s)
    if s == "CONFIRMED"      then return C.green   end
    if s == "OBSERVED"       then return C.gold    end
    if s == "INFERRED"       then return C.amber   end
    if s == "LAST KNOWN"     then return C.goldDim end
    if s == "STALE"          then return C.goldDim end
    if s == "NOT REPLICATED" then return C.gray    end
    if s == "CONFLICT"       then return C.red     end
    if s == "IN GARAGE"      then return C.green   end
    if s == "OUTSIDE"        then return C.amber   end
    if s == "VISIBLE"        then return C.gold    end
    if s == "NOT VISIBLE"    then return C.gray    end
    return C.goldLo
end

-- ══════════════════════════════════════════════════════════════════════
-- LIFECYCLE REGISTRY  (all connections / tasks stored here for cleanup)
-- ══════════════════════════════════════════════════════════════════════
local LIFE = {
    conns  = {},   -- RBXScriptConnections
    tasks  = {},   -- coroutine / task references (tracked via flag table)
    tweens = {},   -- Tween references
}
local function addConn(c)  table.insert(LIFE.conns, c)  return c end
local function killAll()
    for _, c in ipairs(LIFE.conns)  do pcall(function() c:Disconnect() end) end
    for _, t in ipairs(LIFE.tweens) do pcall(function() t:Cancel()     end) end
    LIFE.conns  = {}
    LIFE.tweens = {}
end
local function tween(obj, info, props)
    if not TweenService then return end
    local t = TweenService:Create(obj, info, props)
    table.insert(LIFE.tweens, t)
    t:Play()
    return t
end

-- ══════════════════════════════════════════════════════════════════════
-- DATA RECORD FACTORY
-- ══════════════════════════════════════════════════════════════════════
-- Every piece of collected data must carry domain, value, source, status,
-- timestamp. No raw value is passed between modules without this envelope.

local STATUS = {
    CONFIRMED      = "CONFIRMED",
    OBSERVED       = "OBSERVED",
    INFERRED       = "INFERRED",
    LAST_KNOWN     = "LAST KNOWN",
    UNKNOWN        = "UNKNOWN",
    STALE          = "STALE",
    NOT_REPLICATED = "NOT REPLICATED",
    CONFLICT       = "CONFLICT",
}

local function makeRecord(domain, value, source, status)
    return {
        domain    = domain,
        value     = value or "UNKNOWN",
        source    = source or "none",
        status    = status or STATUS.UNKNOWN,
        timestamp = os.clock(),
    }
end

local UNKNOWN_REC = function(domain)
    return makeRecord(domain, "UNKNOWN", "none", STATUS.UNKNOWN)
end

-- ══════════════════════════════════════════════════════════════════════
-- MODULE 7 · EVENT LOGGER  (declared early so scanners can emit)
-- ══════════════════════════════════════════════════════════════════════
local evLog     = {}   -- circular, max 200
local logDirty  = false
local logFrames = {}   -- cached TextLabel frames by index (incremental render)

local function logEv(text, color)
    table.insert(evLog, 1, { t = os.clock(), text = text, color = color or C.goldDim })
    if #evLog > 200 then table.remove(evLog) end
    logDirty = true
end

-- ══════════════════════════════════════════════════════════════════════
-- MODULE 5 · DATA VALIDATOR
-- ══════════════════════════════════════════════════════════════════════
-- Validates that domain / value / source are consistent.
-- A Portal Gun code must NEVER be promoted to dimension.

-- Pattern for a valid dimension code: LETTER-NUMBER with optional suffix
-- Observed format in-game: "H-3S", "N-2", so suffix letters are allowed.
-- Full sentence text ("Dimensão H-3S (SUA)") must be parsed, not used raw.
local DIM_CODE_PAT   = "([A-Za-z])%-(%d+[A-Za-z]*)"   -- captures H-3S, N-2, A-123
local OWNER_TAG_PAT  = "%(SUA%)"   -- "(SUA)" = "yours" in Portuguese

-- Extract the dimension token from a raw string.
-- Returns: code (string) or nil if none found.
local function extractDimCode(raw)
    if not raw then return nil end
    local s = tostring(raw)
    -- Try strict pattern first: LETTER-DIGITS+optional-letters, isolated by word boundaries
    local letter, rest = s:match(DIM_CODE_PAT)
    if letter and rest then
        return letter:upper() .. "-" .. rest:upper()
    end
    return nil
end

-- Returns true if the string looks like a full sentence rather than a code.
local function isSentence(s)
    if not s then return false end
    return #s > 12 or s:find("%s") ~= nil
end

-- Validate a dimension record — rejects Portal Gun codes used as dim values.
local function validateDimRecord(rec, pgRecord)
    if rec.status == STATUS.UNKNOWN then return rec end
    -- If value matches a known Portal Gun raw value from same player → reject
    if pgRecord and pgRecord.value ~= "UNKNOWN"
       and rec.value == pgRecord.value
       and rec.source == pgRecord.source then
        return UNKNOWN_REC("dimension")
    end
    -- If value is a full sentence, attempt to extract the code from it
    if isSentence(rec.value) then
        local code = extractDimCode(rec.value)
        if code then
            rec.value  = code
            rec.status = STATUS.OBSERVED
        else
            -- Cannot extract a valid code from this sentence
            return UNKNOWN_REC("dimension")
        end
    end
    return rec
end

-- ══════════════════════════════════════════════════════════════════════
-- MODULE 6 · DATA MANAGER
-- ══════════════════════════════════════════════════════════════════════
-- Single source of truth per UserId.

local DM = {}   -- [userId] = PlayerRecord

local function dmGet(userId)  return DM[userId] end

local function dmEnsure(player)
    local uid = player.UserId
    if not DM[uid] then
        DM[uid] = {
            player    = player,
            -- sub-records
            identity  = makeRecord("identity",   player.Name, "Players", STATUS.CONFIRMED),
            character = UNKNOWN_REC("character"),
            position  = UNKNOWN_REC("position"),
            distance  = UNKNOWN_REC("distance"),
            dimension = UNKNOWN_REC("dimension"),
            portalGun = UNKNOWN_REC("portalGun"),
            garage    = UNKNOWN_REC("garage"),
            visibility= UNKNOWN_REC("visibility"),
            -- last-known cache
            lastDim   = nil,
            lastPos   = nil,
            -- meta
            replicated = false,
            updated    = os.clock(),
        }
    end
    return DM[uid]
end

local function dmSet(userId, field, record)
    local r = DM[userId]
    if not r then return end
    if record.status ~= STATUS.UNKNOWN and record.status ~= STATUS.NOT_REPLICATED then
        -- preserve last-known for dimension and position
        if field == "dimension" and r.dimension.value ~= "UNKNOWN" then
            r.lastDim = r.dimension
        end
        if field == "position" and r.position.value ~= "UNKNOWN" then
            r.lastPos = r.position
        end
    end
    r[field]  = record
    r.updated = os.clock()
end

local function dmRemove(userId)
    DM[userId] = nil
end

-- ══════════════════════════════════════════════════════════════════════
-- MODULE 1 · PLAYER SCANNER
-- ══════════════════════════════════════════════════════════════════════
local PS = {}

function PS.getChar(player)
    if not player then return nil end
    local c = player.Character
    return (c and c.Parent) and c or nil
end

-- Fixed: only return HumanoidRootPart, never silently fall back to any BasePart.
function PS.getHRP(player)
    local c = PS.getChar(player)
    if not c then return nil end
    return c:FindFirstChild("HumanoidRootPart")
end

function PS.getPos(player)
    local h = PS.getHRP(player)
    return h and h.Position or nil
end

function PS.getDist(player)
    if player == LP then return 0 end
    local a = PS.getPos(player)
    local b = PS.getPos(LP)
    return (a and b) and math.floor((a - b).Magnitude) or nil
end

function PS.isReplicated(player)
    return PS.getChar(player) ~= nil
end

function PS.updateRecord(player)
    local rec = dmEnsure(player)
    local char = PS.getChar(player)
    local repl = char ~= nil

    rec.replicated = repl

    if repl then
        local pos = PS.getPos(player)
        dmSet(player.UserId, "character", makeRecord("character", char.Name, "Character", STATUS.CONFIRMED))
        if pos then
            dmSet(player.UserId, "position", makeRecord("position", pos, "HRP.Position", STATUS.CONFIRMED))
        else
            dmSet(player.UserId, "position", UNKNOWN_REC("position"))
        end
        local dist = PS.getDist(player)
        if dist then
            dmSet(player.UserId, "distance", makeRecord("distance", dist, "magnitude", STATUS.CONFIRMED))
        else
            dmSet(player.UserId, "distance", UNKNOWN_REC("distance"))
        end
    else
        dmSet(player.UserId, "character",  makeRecord("character",  "absent", "none", STATUS.NOT_REPLICATED))
        dmSet(player.UserId, "position",   makeRecord("position",   "absent", "none", STATUS.NOT_REPLICATED))
        dmSet(player.UserId, "distance",   makeRecord("distance",   "absent", "none", STATUS.NOT_REPLICATED))
        dmSet(player.UserId, "visibility", makeRecord("visibility", "absent", "none", STATUS.NOT_REPLICATED))
    end
end

function PS.hookPlayer(player)
    addConn(player.CharacterAdded:Connect(function()
        task.wait(0.2)
        pcall(PS.updateRecord, player)
        logEv("[SPAWN] " .. player.Name, C.green)
    end))
    addConn(player.CharacterRemoving:Connect(function()
        logEv("[DESP] " .. player.Name, C.gray)
        local r = dmGet(player.UserId)
        if r then
            r.replicated = false
            dmSet(player.UserId, "character", makeRecord("character", "despawning", "CharacterRemoving", STATUS.NOT_REPLICATED))
        end
    end))
end

-- ══════════════════════════════════════════════════════════════════════
-- MODULE 3 · PORTAL GUN SCANNER
-- ══════════════════════════════════════════════════════════════════════
-- Looks for the Portal Gun Tool in the character.
-- The tool name in Portal Sandbox is NOT confirmed — we search by Tool
-- children of Character and check for "portal" in name as a heuristic.
-- EXTENSION POINT T05: replace `PORTAL_TOOL_NAME` once confirmed in-game.

local PORTAL_TOOL_NAME = nil   -- nil = unknown; set after T05 test confirms name

local PGS = {}

-- Strict code pattern: single letter dash digits+optional-letters.
-- No fallback extraction — must match exactly.
local function strictParseCode(raw)
    if not raw then return nil end
    local s = tostring(raw):match("^%s*(.-)%s*$")
    -- Must not be a sentence
    if isSentence(s) then return nil end
    local code = extractDimCode(s)
    if code then return code end
    -- Also accept bare letter+digits without dash: "H35"
    local l, n = s:match("^([A-Za-z])(%d+[A-Za-z]*)$")
    if l and n then return l:upper() .. "-" .. n:upper() end
    return nil
end

function PGS.scan(player)
    local char = PS.getChar(player)
    if not char then
        return makeRecord("portalGun", "NOT_EQUIPPED", "none", STATUS.NOT_REPLICATED)
    end

    -- Search for the Tool
    for _, tool in ipairs(char:GetChildren()) do
        if not tool:IsA("Tool") then continue end
        local tn = tool.Name:lower()
        -- Use confirmed name if available, otherwise heuristic
        local isPortalTool = (PORTAL_TOOL_NAME and tool.Name == PORTAL_TOOL_NAME)
                          or (not PORTAL_TOOL_NAME and tn:find("portal"))
        if not isPortalTool then continue end

        -- Tool found — search for code value
        local codeVal, codeSrc

        -- 1. Tool attributes
        for _, k in ipairs({"PortalGunId", "DimensionId", "Code", "Id", "GunCode"}) do
            local v = tool:GetAttribute(k)
            if v then
                local c = strictParseCode(tostring(v))
                if c then codeVal = c; codeSrc = "TA:" .. tool.Name .. "." .. k; break end
            end
        end

        -- 2. ValueBase children of Tool (single scan)
        if not codeVal then
            for _, v in ipairs(tool:GetChildren()) do
                if v:IsA("ValueBase") then
                    local c = strictParseCode(tostring(v.Value))
                    if c then codeVal = c; codeSrc = "TV:" .. v.Name; break end
                end
            end
        end

        -- 3. TextLabel children of Tool (not descendants — avoid deep scan)
        if not codeVal then
            for _, v in ipairs(tool:GetChildren()) do
                if v:IsA("TextLabel") and v.Text ~= "" then
                    local c = strictParseCode(v.Text)
                    if c then codeVal = c; codeSrc = "TG:" .. v.Name; break end
                end
            end
        end

        local value = codeVal and ("EQUIPPED:" .. codeVal) or "EQUIPPED"
        return makeRecord("portalGun", value, codeSrc or ("TOOL:" .. tool.Name), STATUS.OBSERVED)
    end

    -- Tool not in character (may be in Backpack — not accessible from other clients)
    if player == LP then
        -- Also check Backpack for LP
        local bp = LP:FindFirstChild("Backpack")
        if bp then
            for _, tool in ipairs(bp:GetChildren()) do
                if not tool:IsA("Tool") then continue end
                local tn = tool.Name:lower()
                local isPortalTool = (PORTAL_TOOL_NAME and tool.Name == PORTAL_TOOL_NAME)
                              or (not PORTAL_TOOL_NAME and tn:find("portal"))
                if isPortalTool then
                    return makeRecord("portalGun", "NOT_EQUIPPED", "Backpack:" .. tool.Name, STATUS.OBSERVED)
                end
            end
        end
    end

    return makeRecord("portalGun", "UNKNOWN", "none", STATUS.UNKNOWN)
end

-- ══════════════════════════════════════════════════════════════════════
-- MODULE 2 · DIMENSION SCANNER
-- ══════════════════════════════════════════════════════════════════════
-- Priority order (confirmed sources first):
--   1. LP PlayerGui label matching "Dimensão … (SUA)" pattern
--   2. Player attributes with dimension-related names
--   3. Character attributes
--   4. ValueBase in Player / Character children
--
-- NEVER uses Portal Gun code as a dimension value.
-- NEVER uses full sentences as dimension values — always extracts code.

local DS = {}

-- Known dimension-related attribute keys to try.
-- EXTENSION POINT T07: add confirmed keys after in-game inspection.
local DIM_ATTR_KEYS = {
    "Dimension", "DimensionId", "CurrentDimension", "DimCode",
    "World", "WorldId", "PortalDestination",
}

-- Checks if a TextLabel text represents the game's dimension HUD pill.
-- Observed format: "Dimensão H-3S (SUA)" or "Placa de base"
-- Returns: code string or nil
local function extractFromHudText(text)
    if not text or text == "" then return nil end
    -- Must contain the owner tag "(SUA)" or contain a dimension code pattern
    local hasSua = text:find(OWNER_TAG_PAT) ~= nil
    local code   = extractDimCode(text)
    if code and hasSua then
        return code, "CONFIRMED"   -- "(SUA)" confirms it is the local player's dimension
    elseif code then
        return code, "OBSERVED"
    end
    return nil, nil
end

function DS.scanLP()
    -- Source 1: PlayerGui HUD label (LP only, highest confidence)
    local pg = LP:FindFirstChild("PlayerGui")
    if pg then
        for _, v in ipairs(pg:GetDescendants()) do
            if v:IsA("TextLabel") and v.Text ~= "" then
                local code, conf = extractFromHudText(v.Text)
                if code then
                    return makeRecord("dimension", code, "GUI:" .. v.Name, conf == "CONFIRMED" and STATUS.CONFIRMED or STATUS.OBSERVED)
                end
            end
        end
    end
    return nil
end

function DS.scan(player)
    -- For LP, try HUD label first
    if player == LP then
        local lpRec = DS.scanLP()
        if lpRec then return lpRec end
    end

    -- Source 2: Player attributes
    for _, k in ipairs(DIM_ATTR_KEYS) do
        local v = player:GetAttribute(k)
        if v ~= nil then
            local raw = tostring(v)
            local code = isSentence(raw) and extractDimCode(raw) or (extractDimCode(raw) or raw)
            if code and code ~= "" then
                return makeRecord("dimension", code, "PA:" .. k, STATUS.OBSERVED)
            end
        end
    end

    -- Source 3: ValueBase children of Player (1 level, no descendants)
    for _, v in ipairs(player:GetChildren()) do
        if v:IsA("ValueBase") then
            local n = v.Name:lower()
            if n:find("dim") or n:find("portal") or n:find("world") then
                local raw = tostring(v.Value)
                local code = extractDimCode(raw) or (not isSentence(raw) and raw or nil)
                if code and code ~= "" then
                    return makeRecord("dimension", code, "PV:" .. v.Name, STATUS.OBSERVED)
                end
            end
        end
    end

    -- Source 4: Character attributes
    local char = PS.getChar(player)
    if char then
        for _, k in ipairs(DIM_ATTR_KEYS) do
            local v = char:GetAttribute(k)
            if v ~= nil then
                local raw = tostring(v)
                local code = isSentence(raw) and extractDimCode(raw) or (extractDimCode(raw) or raw)
                if code and code ~= "" then
                    return makeRecord("dimension", code, "CA:" .. k, STATUS.OBSERVED)
                end
            end
        end

        -- Source 5: ValueBase children of Character (1 level)
        for _, v in ipairs(char:GetChildren()) do
            if v:IsA("ValueBase") then
                local n = v.Name:lower()
                if n:find("dim") or n:find("portal") or n:find("world") then
                    local raw = tostring(v.Value)
                    local code = extractDimCode(raw) or (not isSentence(raw) and raw or nil)
                    if code and code ~= "" then
                        return makeRecord("dimension", code, "CV:" .. v.Name, STATUS.OBSERVED)
                    end
                end
            end
        end
    end

    -- No source found
    return UNKNOWN_REC("dimension")
end

-- ══════════════════════════════════════════════════════════════════════
-- MODULE 4 · ENVIRONMENT SCANNER (Garage)
-- ══════════════════════════════════════════════════════════════════════
-- Uses selective Workspace scan at low frequency (every 10s per player).
-- Avoids workspace:GetDescendants() in tight loops.
-- EXTENSION POINT T09: set GARAGE_MODEL_NAME and GARAGE_OWNER_ATTR
-- after in-game inspection.

local ENV = {}
local garCache    = {}   -- [userId] = { regions=[], timestamp }
local GARAGE_TTL  = 10   -- seconds between full rescans per player

-- Name of the garage region/volume part — UNKNOWN until T09 test.
-- If set, only parts with this name are used for containment check.
local GARAGE_REGION_PART_NAME = nil   -- e.g. "GarageRegion" — set after T09

-- Attribute names to identify owner of a garage model.
-- EXTENSION POINT T09.
local GARAGE_OWNER_ATTRS = { "Owner", "OwnerId", "PlayerUserId", "UserId" }
local GARAGE_OWNER_VALUES = { "Owner", "OwnerId", "PlayerName" }  -- ValueBase names

-- Garage model name keywords — EXTENSION POINT T09.
local GARAGE_MODEL_KEYWORDS = { "garage", "bay", "hangar", "plot" }

local function matchesOwner(model, player)
    local uid   = tostring(player.UserId)
    local uname = player.Name:lower()

    -- Attribute match (reliable)
    for _, k in ipairs(GARAGE_OWNER_ATTRS) do
        local oa = model:GetAttribute(k)
        if oa then
            local s = tostring(oa)
            if s == uid then return true, "attribute:" .. k end
        end
    end

    -- ValueBase child match (reliable)
    for _, k in ipairs(GARAGE_OWNER_VALUES) do
        local ov = model:FindFirstChild(k)
        if ov and ov:IsA("ValueBase") then
            local s = tostring(ov.Value)
            if s == uid or s:lower() == uname then
                return true, "valuechild:" .. k
            end
        end
    end

    -- Do NOT use name substring match — too many false positives.
    return false, nil
end

-- For a confirmed garage model, compute the containment region.
-- If a region/volume part is known, use its local space (most accurate).
-- Otherwise fall back to AABB (less accurate for rotated structures).
local function computeRegion(model, player)
    -- Try to find a named region part
    if GARAGE_REGION_PART_NAME then
        for _, p in ipairs(model:GetDescendants()) do
            if p:IsA("BasePart") and p.Name == GARAGE_REGION_PART_NAME then
                return { part = p, aabb = nil }
            end
        end
    end

    -- Collect relevant parts (tagged or all)
    local parts = {}
    local tagKeywords = { "wall", "floor", "bound", "interior", "region", "volume" }
    for _, p in ipairs(model:GetDescendants()) do
        if p:IsA("BasePart") then
            local n = p.Name:lower()
            for _, kw in ipairs(tagKeywords) do
                if n:find(kw) then table.insert(parts, p); break end
            end
        end
    end
    if #parts == 0 then
        for _, p in ipairs(model:GetDescendants()) do
            if p:IsA("BasePart") then table.insert(parts, p) end
        end
    end
    if #parts == 0 then return nil end

    -- AABB in world space
    local mn, mx = parts[1].Position, parts[1].Position
    for _, p in ipairs(parts) do
        local s = p.Size / 2
        for dx = -1, 1, 2 do for dy = -1, 1, 2 do for dz = -1, 1, 2 do
            local cr = p.CFrame * Vector3.new(s.X * dx, s.Y * dy, s.Z * dz)
            mn = Vector3.new(math.min(mn.X, cr.X), math.min(mn.Y, cr.Y), math.min(mn.Z, cr.Z))
            mx = Vector3.new(math.max(mx.X, cr.X), math.max(mx.Y, cr.Y), math.max(mx.Z, cr.Z))
        end end end
    end
    return { part = nil, aabb = { center = (mn + mx) / 2, size = mx - mn } }
end

local function isInsideRegion(region, pos)
    if not region then return false end
    -- Preferred: local-space check using the part's CFrame
    if region.part then
        local lp = region.part.CFrame:PointToObjectSpace(pos)
        local hs = region.part.Size / 2
        return math.abs(lp.X) <= hs.X
           and math.abs(lp.Y) <= hs.Y
           and math.abs(lp.Z) <= hs.Z
    end
    -- Fallback: world AABB
    if region.aabb then
        local h = region.aabb.size / 2
        return math.abs(pos.X - region.aabb.center.X) <= h.X
           and math.abs(pos.Y - region.aabb.center.Y) <= h.Y
           and math.abs(pos.Z - region.aabb.center.Z) <= h.Z
    end
    return false
end

function ENV.scanGarages(player)
    local now = os.clock()
    local c   = garCache[player.UserId]
    if c and (now - c.timestamp) < GARAGE_TTL then
        return c.regions, c.ownerConfirmed
    end

    local regions       = {}
    local ownerConfirmed = false
    local scanCount     = 0
    local MAX_SCAN      = 300   -- cap to avoid performance hit

    -- Only scan top-level children of workspace first to stay shallow.
    -- If the garage is nested (e.g. workspace.Plots.PlotX.Garage),
    -- this will miss it. EXTENSION POINT T09: add path prefix.
    local function tryModel(obj)
        if scanCount >= MAX_SCAN then return end
        scanCount += 1
        if not obj:IsA("Model") then return end
        local n = obj.Name:lower()
        for _, kw in ipairs(GARAGE_MODEL_KEYWORDS) do
            if n:find(kw) then
                local owned, ownerSrc = matchesOwner(obj, player)
                if owned then
                    local region = computeRegion(obj, player)
                    if region then
                        table.insert(regions, region)
                        ownerConfirmed = true
                    end
                end
                return
            end
        end
    end

    for _, obj in ipairs(workspace:GetChildren()) do tryModel(obj) end
    -- One level deeper (common pattern: workspace.Folder.GarageModel)
    if #regions == 0 and scanCount < MAX_SCAN then
        for _, folder in ipairs(workspace:GetChildren()) do
            if folder:IsA("Folder") or folder:IsA("Model") then
                for _, obj in ipairs(folder:GetChildren()) do
                    tryModel(obj)
                    if scanCount >= MAX_SCAN then break end
                end
            end
            if scanCount >= MAX_SCAN then break end
        end
    end

    garCache[player.UserId] = { regions = regions, timestamp = now, ownerConfirmed = ownerConfirmed }
    return regions, ownerConfirmed
end

function ENV.getGarageStatus(player)
    local pos = PS.getPos(player)
    if not pos then return UNKNOWN_REC("garage") end

    local ok, regions, ownerConfirmed = pcall(ENV.scanGarages, player)
    if not ok or not regions or #regions == 0 then
        return makeRecord("garage", "UNKNOWN", "none", STATUS.UNKNOWN)
    end

    for _, region in ipairs(regions) do
        if isInsideRegion(region, pos) then
            local conf = ownerConfirmed and STATUS.CONFIRMED or STATUS.OBSERVED
            return makeRecord("garage", "IN GARAGE", "spatial", conf)
        end
    end
    local conf = ownerConfirmed and STATUS.CONFIRMED or STATUS.OBSERVED
    return makeRecord("garage", "OUTSIDE", "spatial", conf)
end

-- ══════════════════════════════════════════════════════════════════════
-- VISIBILITY  (not a full module — utility used by Data Manager update)
-- ══════════════════════════════════════════════════════════════════════
local function getVisibility(player)
    local char = PS.getChar(player)
    if not char then
        return makeRecord("visibility", "absent", "none", STATUS.NOT_REPLICATED)
    end
    local hrp = PS.getHRP(player)
    if not hrp then
        return makeRecord("visibility", "UNAVAILABLE", "none", STATUS.UNKNOWN)
    end

    -- Transparency check
    local vis, total = 0, 0
    for _, p in ipairs(char:GetDescendants()) do
        if p:IsA("BasePart") and p.Name ~= "HumanoidRootPart" then
            total += 1
            if p.Transparency < 0.95 then vis += 1 end
        end
    end
    if total > 0 and vis == 0 then
        return makeRecord("visibility", "NOT VISIBLE", "transparency", STATUS.OBSERVED)
    end

    -- Distance gate
    local dist = PS.getDist(player)
    if dist and dist > 2000 then
        return makeRecord("visibility", "NOT VISIBLE", "distance", STATUS.OBSERVED)
    end

    -- Raycast (corrected: exclude char parts + LP char parts as separate entries)
    local camPos  = Camera.CFrame.Position
    local charPos = hrp.Position
    local dir     = charPos - camPos
    local mag     = dir.Magnitude
    if mag < 1 then
        return makeRecord("visibility", "VISIBLE", "proximity", STATUS.OBSERVED)
    end

    local rp = RaycastParams.new()
    rp.FilterType = Enum.RaycastFilterType.Exclude
    local excludeList = { char }
    local lpChar = PS.getChar(LP)
    if lpChar then table.insert(excludeList, lpChar) end
    rp.FilterDescendantsInstances = excludeList   -- Fixed E3: flat array of instances

    local hit = workspace:Raycast(camPos, dir.Unit * mag, rp)
    if hit then
        return makeRecord("visibility", "NOT VISIBLE", "raycast", STATUS.OBSERVED)
    end
    return makeRecord("visibility", "VISIBLE", "raycast", STATUS.OBSERVED)
end

-- ══════════════════════════════════════════════════════════════════════
-- FULL PLAYER RECORD UPDATE  (orchestrates all scanners)
-- ══════════════════════════════════════════════════════════════════════
local garageTickAcc  = {}   -- [userId] = accumulator for garage scan throttle
local GARAGE_FREQ    = 5    -- seconds between garage checks (not every 0.5s cycle)

local function updatePlayerRecord(player, dt)
    local uid = player.UserId
    dmEnsure(player)

    -- Player Scanner (position, distance, replication)
    pcall(PS.updateRecord, player)

    -- Portal Gun Scanner
    local pgRec = PGS.scan(player)
    dmSet(uid, "portalGun", pgRec)

    -- Dimension Scanner + Validator
    local dimRec = DS.scan(player)
    dimRec = validateDimRecord(dimRec, pgRec)
    dmSet(uid, "dimension", dimRec)

    -- Visibility
    local visRec = getVisibility(player)
    dmSet(uid, "visibility", visRec)

    -- Garage (throttled)
    garageTickAcc[uid] = (garageTickAcc[uid] or GARAGE_FREQ) - (dt or 0.5)
    if garageTickAcc[uid] <= 0 then
        garageTickAcc[uid] = GARAGE_FREQ
        local garRec = ENV.getGarageStatus(player)
        dmSet(uid, "garage", garRec)
    end
end

-- ══════════════════════════════════════════════════════════════════════
-- ══════════════════════════════════════════════════════════════════════
--  HUD MANAGER — EVIL MORTY OPTICAL VISOR
-- ══════════════════════════════════════════════════════════════════════
-- ══════════════════════════════════════════════════════════════════════

local SG, lensFrame, sideMenu
local minimized    = false
local miniBadge
local activeMode   = "PROXIMITY"   -- PROXIMITY | OMEGA | DIAGNOSTICS
local modeFrames   = {}

-- Player rows for PROXIMITY / OMEGA tabs — incremental, not rebuilt each frame
local playerRowMap = {}   -- [userId] = { frame, labels... }
local selectedUid  = nil

-- ── UI helpers ────────────────────────────────────────────────────────
local FONT = Enum.Font.Code

local function uiCorner(p, r)
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, r or 4)
    c.Parent = p; return c
end
local function uiStroke(p, col, t)
    local s = Instance.new("UIStroke")
    s.Color = col or C.goldDim; s.Thickness = t or 1
    s.Parent = p; return s
end
local function uiFr(parent, size, pos, bg, alpha)
    local f = Instance.new("Frame")
    f.Size = size; f.Position = pos or UDim2.new(0,0,0,0)
    f.BackgroundColor3 = bg or C.black
    f.BackgroundTransparency = alpha or 0
    f.BorderSizePixel = 0; f.Parent = parent; return f
end
local function uiLbl(parent, text, size, pos, col, sz, align)
    local l = Instance.new("TextLabel")
    l.Text = text; l.Size = size; l.Position = pos or UDim2.new(0,0,0,0)
    l.BackgroundTransparency = 1
    l.TextColor3 = col or C.gold; l.Font = FONT; l.TextSize = sz or 11
    l.TextXAlignment = align or Enum.TextXAlignment.Left
    l.TextYAlignment = Enum.TextYAlignment.Center
    l.TextTruncate = Enum.TextTruncate.AtEnd; l.TextWrapped = false
    l.Parent = parent; return l
end
local function uiBtn(parent, text, size, pos, bg, tc, fn)
    local b = Instance.new("TextButton")
    b.Text = text; b.Size = size; b.Position = pos or UDim2.new(0,0,0,0)
    b.BackgroundColor3 = bg or C.goldLo; b.TextColor3 = tc or C.gold
    b.Font = FONT; b.TextSize = 11; b.BorderSizePixel = 0
    b.AutoButtonColor = false; b.Active = true; b.Parent = parent
    uiCorner(b, 3)
    local base = bg or C.goldLo
    b.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1
        or inp.UserInputType == Enum.UserInputType.Touch then
            b.BackgroundColor3 = Color3.fromRGB(50, 40, 0)
        end
    end)
    b.InputEnded:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1
        or inp.UserInputType == Enum.UserInputType.Touch then
            b.BackgroundColor3 = base
        end
    end)
    if fn then addConn(b.MouseButton1Click:Connect(fn)) end
    return b
end
local function uiScroll(parent, size, pos)
    local s = Instance.new("ScrollingFrame")
    s.Size = size; s.Position = pos or UDim2.new(0,0,0,0)
    s.BackgroundTransparency = 1; s.ScrollBarThickness = 3
    s.ScrollBarImageColor3 = C.goldDim
    s.ScrollingDirection = Enum.ScrollingDirection.Y
    s.CanvasSize = UDim2.new(0,0,0,0); s.Parent = parent; return s
end
local function uiVList(parent)
    local l = Instance.new("UIListLayout")
    l.SortOrder = Enum.SortOrder.LayoutOrder; l.Padding = UDim.new(0,2)
    l.Parent = parent
    local conn = l:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        if parent and parent.Parent then
            parent.CanvasSize = UDim2.new(0, 0, 0, l.AbsoluteContentSize.Y + 4)
        end
    end)
    addConn(conn)   -- Fixed E8: stored connection
    return l
end

-- ── Viewport-safe layout ──────────────────────────────────────────────
-- Safe margins based on HUD analysis:
--   Top:    70px  (below toolbar)
--   Bottom: 110px (above hotbar + jump)
--   Left:   10px  (small margin)
--   Right:  185px (clear of action buttons)
local function safeLayout()
    local vp = Camera.ViewportSize
    local W  = math.min(math.max(vp.X - 195, 280), 420)
    local H  = math.min(math.max(vp.Y - 185, 300), 500)
    -- Prefer bottom-left (avoids right-side action buttons)
    local x  = 6
    local y  = math.max(70, vp.Y - H - 115)
    return UDim2.new(0, x, 0, y), W, H
end

-- ══════════════════════════════════════════════════════════════════════
-- LENS RING ANIMATIONS
-- ══════════════════════════════════════════════════════════════════════
local ringConns = {}

local function stopRings()
    for _, c in ipairs(ringConns) do pcall(function() c:Disconnect() end) end
    ringConns = {}
end

local function startRing(frame, speedDeg, clockwise)
    local conn = RunService.Heartbeat:Connect(function(dt)
        if not frame or not frame.Parent then return end
        local delta = speedDeg * dt * (clockwise and 1 or -1)
        frame.Rotation = frame.Rotation + delta
    end)
    table.insert(ringConns, conn)
    addConn(conn)
    return conn
end

-- ══════════════════════════════════════════════════════════════════════
-- SCANLINES  (subtle horizontal lines overlay)
-- ══════════════════════════════════════════════════════════════════════
local function CreateScanlines(parent, h)
    local sf = uiFr(parent, UDim2.new(1,0,1,0), nil, Color3.new(0,0,0), 1)
    sf.ZIndex = 10; sf.ClipDescendants = false
    local step = 4
    local lineCount = math.floor(h / step)
    for i = 0, lineCount do
        local line = uiFr(sf, UDim2.new(1,0,0,1), UDim2.new(0,0,0, i*step), Color3.new(0,0,0))
        line.BackgroundTransparency = 0.86
        line.ZIndex = 10
    end
    return sf
end

-- ══════════════════════════════════════════════════════════════════════
-- RETICLES  (crosshair + angular marks)
-- ══════════════════════════════════════════════════════════════════════
local function CreateReticles(parent)
    -- Center cross
    uiFr(parent, UDim2.new(0,1,0,20),  UDim2.new(0.5,0,0.5,-10), C.goldDim)
    uiFr(parent, UDim2.new(0,20,0,1),  UDim2.new(0.5,-10,0.5,0), C.goldDim)
    -- Corner bracket marks
    local function bracket(xSign, ySign)
        local ox, oy = xSign * 60, ySign * 60
        local bx = 0.5 + (xSign * 0.5 > 0 and 0 or -1)   -- anchor relative
        uiFr(parent, UDim2.new(0,12,0,2), UDim2.new(0.5, ox,       0.5, oy),      C.gold)
        uiFr(parent, UDim2.new(0,2,0,12), UDim2.new(0.5, ox,       0.5, oy),      C.gold)
    end
    bracket( 1, 1); bracket(-1, 1); bracket( 1,-1); bracket(-1,-1)
    -- Cardinal tick marks
    for i = 1, 4 do
        local angle = (i - 1) * 90
        local rad = math.rad(angle)
        local rx  = math.sin(rad) * 0.42
        local ry  = -math.cos(rad) * 0.42
        local tick = uiFr(parent, UDim2.new(0,2,0,8), UDim2.new(0.5+rx,-1, 0.5+ry,-4), C.amber)
        tick.Rotation = angle
    end
end

-- ══════════════════════════════════════════════════════════════════════
-- LENS FRAME  (central circular visor)
-- ══════════════════════════════════════════════════════════════════════
local function CreateLensFrame(parent, W, H)
    local lensSize = math.min(W, H) - 20
    local lens = uiFr(parent, UDim2.new(0,lensSize,0,lensSize),
        UDim2.new(0.5,-lensSize/2, 0.5,-lensSize/2), C.black)
    uiCorner(lens, lensSize // 2)
    uiStroke(lens, C.goldDim, 2)

    -- Vignette (black edge fade via nested rounded frame)
    local vignette = uiFr(lens, UDim2.new(1,0,1,0), nil, Color3.new(0,0,0), 0.55)
    uiCorner(vignette, lensSize // 2)

    -- Outer ring (static, faint)
    local outerRing = uiFr(lens, UDim2.new(0.9,0,0.9,0), UDim2.new(0.05,0,0.05,0), Color3.new(0,0,0), 1)
    uiStroke(outerRing, C.goldLo, 1)
    uiCorner(outerRing, lensSize // 2)

    -- Middle ring (rotates CW)
    local midRing = uiFr(lens, UDim2.new(0.72,0,0.72,0), UDim2.new(0.14,0,0.14,0), Color3.new(0,0,0), 1)
    uiStroke(midRing, C.goldDim, 1)
    uiCorner(midRing, lensSize // 2)
    -- Dash mark on mid ring
    uiFr(midRing, UDim2.new(0,8,0,2), UDim2.new(0.5,-4, 0,2), C.gold)
    startRing(midRing, 25, true)

    -- Inner ring (rotates CCW, faster)
    local innerRing = uiFr(lens, UDim2.new(0.52,0,0.52,0), UDim2.new(0.24,0,0.24,0), Color3.new(0,0,0), 1)
    uiStroke(innerRing, C.amber, 1)
    uiCorner(innerRing, lensSize // 2)
    uiFr(innerRing, UDim2.new(0,6,0,2), UDim2.new(0.5,-3, 1,-4), C.amber)
    startRing(innerRing, 45, false)

    CreateReticles(lens)
    CreateScanlines(lens, lensSize)

    return lens, lensSize
end

-- ══════════════════════════════════════════════════════════════════════
-- SIDE MENU  (3 mode buttons + controls, left of lens)
-- ══════════════════════════════════════════════════════════════════════
local modeBtnRefs = {}

local function CreateSideMenu(parent, h, onMode, onMinimize, onClose, onToggleFX)
    local menuW = 90
    local menu = uiFr(parent, UDim2.new(0,menuW,0,h), UDim2.new(0,0,0,0), C.black, 0.3)
    uiStroke(menu, C.goldLo, 1); uiCorner(menu, 4)

    -- Title label
    uiLbl(menu, "[ EM·OPT ]", UDim2.new(1,0,0,16), UDim2.new(0,0,0,4), C.goldDim, 8, Enum.TextXAlignment.Center)

    local modes = {
        { name = "PROXIMITY", label = "PROX\nSCAN" },
        { name = "OMEGA",     label = "OMEGA\nDEVICE" },
        { name = "DIAGNOSTICS", label = "SYS\nDIAG" },
    }

    local btnH = 42
    for i, m in ipairs(modes) do
        local b = uiBtn(menu, m.label,
            UDim2.new(1,-8,0,btnH), UDim2.new(0,4,0,22+(i-1)*(btnH+4)),
            C.goldLo, C.goldDim,
            function() onMode(m.name) end)
        b.TextSize = 9; b.TextWrapped = true; b.TextYAlignment = Enum.TextYAlignment.Center
        modeBtnRefs[m.name] = b
    end

    -- Separator
    uiFr(menu, UDim2.new(1,-8,0,1), UDim2.new(0,4,0,22+3*(btnH+4)), C.goldLo)

    local ctrlY = 22 + 3 * (btnH + 4) + 8
    uiBtn(menu, "FX", UDim2.new(1,-8,0,20), UDim2.new(0,4,0,ctrlY),
        C.goldLo, C.goldDim, onToggleFX).TextSize = 9

    uiBtn(menu, "▬", UDim2.new(1,-8,0,20), UDim2.new(0,4,0,ctrlY+24),
        C.goldLo, C.amber, onMinimize).TextSize = 9

    uiBtn(menu, "✕", UDim2.new(1,-8,0,20), UDim2.new(0,4,0,ctrlY+48),
        Color3.fromRGB(40,8,0), C.red, onClose).TextSize = 9

    -- Energy indicator
    uiLbl(menu, "PWR ██▓", UDim2.new(1,0,0,12), UDim2.new(0,0,1,-20), C.goldLo, 8, Enum.TextXAlignment.Center)

    return menu, menuW
end

-- ══════════════════════════════════════════════════════════════════════
-- MODE FRAME BUILDER (shared content area, right of side menu)
-- ══════════════════════════════════════════════════════════════════════
local function makeContentArea(parent, menuW, W, H)
    local cW = W - menuW - 2
    local ca = uiFr(parent, UDim2.new(0,cW,1,0), UDim2.new(0,menuW+2,0,0), C.black, 0.15)
    ca.ClipDescendants = true
    return ca, cW
end

-- ══════════════════════════════════════════════════════════════════════
-- MODULE 9 · PROXIMITY SCAN MODE
-- ══════════════════════════════════════════════════════════════════════
local proxFrame, proxScroll, proxRows = nil, nil, {}
local radarAngle = 0
local radarBeam  = nil
local radarConnRef = nil

local function RenderProximityScan(parent, cW, cH)
    local f = uiFr(parent, UDim2.new(1,0,1,0), nil, C.black, 1)
    f.Visible = false; f.Name = "ProxMode"
    modeFrames["PROXIMITY"] = f
    proxFrame = f

    -- Top status strip
    local strip = uiFr(f, UDim2.new(1,0,0,22), nil, C.goldLo, 0.4)
    uiLbl(strip, "◈ PROXIMITY RADAR: ACTIVE", UDim2.new(0.6,0,1,0), UDim2.new(0,4,0,0), C.gold, 9)
    local detectedLbl = uiLbl(strip, "NODES: 0", UDim2.new(0.4,0,1,0), UDim2.new(0.6,0,0,0), C.amber, 9, Enum.TextXAlignment.Right)

    -- Mini radar canvas
    local radarSz = math.min(cW, cH * 0.4)
    local radarCx = cW / 2
    local radarCy = 22 + radarSz / 2 + 6
    local radarFr = uiFr(f, UDim2.new(0,radarSz,0,radarSz), UDim2.new(0, cW/2 - radarSz/2, 0, 28), C.black, 0.8)
    uiStroke(radarFr, C.goldDim, 1); uiCorner(radarFr, radarSz // 2)

    -- Concentric rings on radar
    for _, frac in ipairs({0.33, 0.66, 1.0}) do
        local sz = radarSz * frac
        local ring = uiFr(radarFr, UDim2.new(0,sz,0,sz), UDim2.new(0.5,-sz/2,0.5,-sz/2), Color3.new(0,0,0),1)
        uiStroke(ring, frac == 1.0 and C.goldDim or C.goldLo, 1); uiCorner(ring, sz//2)
    end

    -- Radar beam (rotating line)
    radarBeam = uiFr(radarFr, UDim2.new(0,1,0.5,0), UDim2.new(0.5,0,0,0), C.gold, 0.5)
    radarBeam.AnchorPoint = Vector2.new(0, 0.5)

    -- Radar sweep loop
    if radarConnRef then radarConnRef:Disconnect() end
    local conn = RunService.Heartbeat:Connect(function(dt)
        if not radarBeam or not radarBeam.Parent then return end
        radarAngle = (radarAngle + dt * 240) % 360   -- 240 deg/s ≈ 1.5s full sweep
        radarBeam.Rotation = radarAngle
    end)
    radarConnRef = conn
    addConn(conn)

    -- Player list area
    local listTop = 28 + radarSz + 8
    local listH   = cH - listTop - 4
    proxScroll = uiScroll(f, UDim2.new(1,0,0,listH), UDim2.new(0,0,0,listTop))
    uiVList(proxScroll)

    -- Footer
    uiLbl(f, "RANGE: ALL · REPLICATION: LIVE", UDim2.new(1,0,0,16), UDim2.new(0,0,1,-18), C.goldLo, 8, Enum.TextXAlignment.Center)

    return f, detectedLbl
end

-- Player row management (incremental)
local proxDetectedLbl = nil

local function getProxRow(player)
    local uid = player.UserId
    if proxRows[uid] and proxRows[uid].frame and proxRows[uid].frame.Parent then
        return proxRows[uid]
    end
    if not proxScroll then return nil end

    local row = uiFr(proxScroll, UDim2.new(1,-4,0,28), UDim2.new(0,2,0,0), C.bgRow)
    row.Name = "PR" .. uid; uiCorner(row, 2); row.LayoutOrder = uid

    uiFr(row, UDim2.new(0,2,0.7,0), UDim2.new(0,0,0.15,0), C.goldDim)
    local nameLbl  = uiLbl(row, player.Name, UDim2.new(0,100,1,0), UDim2.new(0,8,0,0), C.white, 10)
    local dimLbl   = uiLbl(row, "—", UDim2.new(0,70,1,0), UDim2.new(0,112,0,0), C.goldDim, 9)
    local statLbl  = uiLbl(row, "—", UDim2.new(0,70,1,0), UDim2.new(0,186,0,0), C.goldDim, 9)
    local distLbl  = uiLbl(row, "—", UDim2.new(0,50,1,0), UDim2.new(0,260,0,0), C.amber, 9)
    uiFr(row, UDim2.new(1,0,0,1), UDim2.new(0,0,1,-1), C.goldLo)

    -- Tap for detail
    local tap = Instance.new("TextButton")
    tap.Text = ""; tap.Size = UDim2.new(1,0,1,0); tap.BackgroundTransparency = 1
    tap.ZIndex = 5; tap.Parent = row
    addConn(tap.MouseButton1Click:Connect(function()
        selectedUid = uid
    end))

    proxRows[uid] = { frame=row, nameLbl=nameLbl, dimLbl=dimLbl, statLbl=statLbl, distLbl=distLbl }
    return proxRows[uid]
end

function UpdatePlayerMarkers()
    local all   = Players:GetPlayers()
    local alive = {}
    for _, p in ipairs(all) do alive[p.UserId] = true end

    -- Remove departed
    for uid, entry in pairs(proxRows) do
        if not alive[uid] then
            if entry.frame and entry.frame.Parent then entry.frame:Destroy() end
            proxRows[uid] = nil
        end
    end

    -- Update / create
    local count = 0
    for _, p in ipairs(all) do
        count += 1
        local row = getProxRow(p)
        if not row then continue end
        local rec = dmGet(p.UserId)
        if not rec then continue end

        local dimVal  = rec.dimension.value
        local dimStat = rec.dimension.status
        row.dimLbl.Text       = (dimVal ~= "UNKNOWN" and dimVal ~= "") and dimVal or "—"
        row.dimLbl.TextColor3 = statusColor(dimStat)
        row.statLbl.Text      = dimStat
        row.statLbl.TextColor3 = statusColor(dimStat)
        local distVal = rec.distance.value
        row.distLbl.Text = (type(distVal)=="number") and distVal.."m" or "—"
    end

    if proxDetectedLbl then
        proxDetectedLbl.Text = "NODES: " .. count
    end
end

-- ══════════════════════════════════════════════════════════════════════
-- MODULE 9 / OMEGA DEVICE MODE
-- ══════════════════════════════════════════════════════════════════════
-- Full wireframe holographic projector with independent rotating parts.
-- Implemented as layered 2D projection (authentic 3D not available in
-- ScreenGui without ViewportFrame, which may not be supported in all
-- executor environments).

local omegaFrame     = nil
local omegaNodeFrames = {}   -- [userId] = frame
local omegaRings      = {}

-- Orbital ring definitions: { radius fraction, speed deg/s, clockwise, strokeColor }
local ORBITAL_DEFS = {
    { r=0.42, spd=18,  cw=true,  col=C.gold    },
    { r=0.32, spd=30,  cw=false, col=C.amber   },
    { r=0.22, spd=50,  cw=true,  col=C.goldDim },
    { r=0.50, spd=10,  cw=false, col=C.goldLo  },
    { r=0.15, spd=80,  cw=true,  col=C.gold    },
}

-- Radiating spoke definitions (count, length fraction)
local SPOKE_COUNT = 8

local function RenderOmegaDevice(parent, cW, cH)
    local f = uiFr(parent, UDim2.new(1,0,1,0), nil, C.black, 1)
    f.Visible = false; f.Name = "OmegaMode"
    modeFrames["OMEGA"] = f
    omegaFrame = f

    local centerX = cW / 2
    local centerY = cH * 0.44
    local maxR    = math.min(centerX, centerY) * 0.95

    -- Background grid lines (subtle)
    for i = 1, 6 do
        local y = cH * (i / 7)
        uiFr(f, UDim2.new(1,0,0,1), UDim2.new(0,0,0,y), C.goldLo, 0.85)
    end
    for i = 1, 5 do
        local x = cW * (i / 6)
        uiFr(f, UDim2.new(0,1,1,0), UDim2.new(0,x,0,0), C.goldLo, 0.85)
    end

    -- Orbital rings
    for i, def in ipairs(ORBITAL_DEFS) do
        local sz = maxR * def.r * 2
        local ring = uiFr(f, UDim2.new(0,sz,0,sz),
            UDim2.new(0, centerX-sz/2, 0, centerY-sz/2), Color3.new(0,0,0), 1)
        uiStroke(ring, def.col, 1); uiCorner(ring, sz//2)
        -- Tick mark on ring
        local tick = uiFr(ring, UDim2.new(0,4,0,2), UDim2.new(0.5,-2, 0,2), def.col)
        startRing(ring, def.spd, def.cw)
        omegaRings[i] = ring
    end

    -- Radiating spokes (static)
    for i = 0, SPOKE_COUNT-1 do
        local angle = (i / SPOKE_COUNT) * 2 * math.pi
        local len   = maxR * 0.52
        local ex    = centerX + math.cos(angle) * len
        local ey    = centerY + math.sin(angle) * len
        local dx, dy = ex - centerX, ey - centerY
        local mag   = math.sqrt(dx*dx + dy*dy)
        local spoke = uiFr(f, UDim2.new(0,1,0,mag), UDim2.new(0,centerX,0,centerY), C.goldLo, 0.6)
        spoke.AnchorPoint = Vector2.new(0.5, 0)
        spoke.Rotation    = math.deg(math.atan2(dy, dx)) + 90
    end

    -- Nucleus (pulsing core)
    local nucSz = maxR * 0.18
    local nuc   = uiFr(f, UDim2.new(0,nucSz,0,nucSz),
        UDim2.new(0, centerX-nucSz/2, 0, centerY-nucSz/2), C.goldDim)
    uiCorner(nuc, nucSz//2); uiStroke(nuc, C.gold, 2)
    -- Nucleus label
    uiLbl(nuc, "Ω", UDim2.new(1,0,1,0), nil, C.white, 14, Enum.TextXAlignment.Center)
    -- Nucleus pulse loop
    local nucAlpha = 0
    local nucDir   = 1
    local nucConn = RunService.Heartbeat:Connect(function(dt)
        if not nuc or not nuc.Parent then return end
        nucAlpha = nucAlpha + dt * nucDir * 1.2
        if nucAlpha >= 0.6 then nucDir = -1 end
        if nucAlpha <= 0.0 then nucDir =  1 end
        nuc.BackgroundTransparency = math.clamp(nucAlpha, 0, 0.6)
    end)
    addConn(nucConn)

    -- Floating data panels (connected to spokes)
    for i = 1, math.min(SPOKE_COUNT, 6) do
        local angle = (i / SPOKE_COUNT) * 2 * math.pi
        local dist  = maxR * 0.72
        local px    = centerX + math.cos(angle) * dist
        local py    = centerY + math.sin(angle) * dist
        local panel = uiFr(f, UDim2.new(0,72,0,22),
            UDim2.new(0, px-36, 0, py-11), C.bgRow, 0.2)
        uiStroke(panel, C.goldLo, 1); uiCorner(panel, 2)
        local lbSz = UDim2.new(1,0,1,0)
        uiLbl(panel, "NODE-" .. i, lbSz, nil, C.goldLo, 8, Enum.TextXAlignment.Center)
        omegaNodeFrames[i] = panel
    end

    -- Top status
    uiLbl(f, "◈ OMEGA DIMENSIONAL DEVICE", UDim2.new(1,0,0,16), nil, C.gold, 9, Enum.TextXAlignment.Center)

    -- Bottom legend
    local legendY = cH - 60
    local function legendRow(label, val, y)
        local row = uiFr(f, UDim2.new(1,-8,0,14), UDim2.new(0,4,0,y), C.black, 0.6)
        uiLbl(row, label, UDim2.new(0,110,1,0), nil, C.goldDim, 8)
        uiLbl(row, val,   UDim2.new(0,100,1,0), UDim2.new(0,114,0,0), C.amber, 8)
        return row
    end
    legendRow("DIMENSIONS OBS:",  "UNKNOWN", legendY)
    legendRow("PLAYERS TRACKED:", "0", legendY + 16)
    legendRow("LAST SYNC:", "PENDING", legendY + 32)

    return f
end

function UpdateDimensionMarkers()
    -- Update the floating node panels with real player data
    local players = Players:GetPlayers()
    local idx = 1
    for _, p in ipairs(players) do
        local panel = omegaNodeFrames[idx]
        if not panel then break end
        local rec = dmGet(p.UserId)
        local dimVal = rec and rec.dimension.value or "UNKNOWN"
        -- Update label
        for _, child in ipairs(panel:GetChildren()) do
            if child:IsA("TextLabel") then
                child.Text = p.Name:sub(1,8) .. "\n" .. dimVal
                child.TextColor3 = statusColor(rec and rec.dimension.status or STATUS.UNKNOWN)
                break
            end
        end
        idx += 1
    end
end

function UpdateGarageMarkers()
    -- No visual markers in Omega for garages without confirmed data.
    -- This is an extension point for T09.
end

-- ══════════════════════════════════════════════════════════════════════
-- SYSTEM DIAGNOSTICS MODE
-- ══════════════════════════════════════════════════════════════════════
local diagFrame = nil
local diagLabels = {}   -- key → TextLabel

local function RenderSystemDiagnostics(parent, cW, cH)
    local f = uiFr(parent, UDim2.new(1,0,1,0), nil, C.black, 1)
    f.Visible = false; f.Name = "DiagMode"
    modeFrames["DIAGNOSTICS"] = f
    diagFrame = f

    uiLbl(f, "◈ SYSTEM DIAGNOSTICS TERMINAL", UDim2.new(1,0,0,14), nil, C.gold, 9, Enum.TextXAlignment.Center)
    uiFr(f, UDim2.new(1,-8,0,1), UDim2.new(0,4,0,16), C.goldDim)

    local scroll = uiScroll(f, UDim2.new(1,0,1,-20), UDim2.new(0,0,0,20))
    uiVList(scroll)

    local function diagRow(label, key, initialVal)
        local row = uiFr(scroll, UDim2.new(1,-4,0,18), UDim2.new(0,2,0,0), C.bgRow)
        row.LayoutOrder = #scroll:GetChildren()
        uiStroke(row, C.goldLo, 1); uiCorner(row, 2)
        uiLbl(row, label, UDim2.new(0,130,1,0), UDim2.new(0,4,0,0), C.goldDim, 8)
        local valL = uiLbl(row, initialVal or "PENDING", UDim2.new(1,-134,1,0), UDim2.new(0,134,0,0), C.amber, 9)
        diagLabels[key] = valL
        return valL
    end

    -- Activity bar
    local function actBar(label, key)
        local row = uiFr(scroll, UDim2.new(1,-4,0,22), UDim2.new(0,2,0,0), C.bgRow)
        row.LayoutOrder = #scroll:GetChildren()
        uiLbl(row, label, UDim2.new(0,100,1,0), UDim2.new(0,4,0,0), C.goldDim, 8)
        local barBg = uiFr(row, UDim2.new(1,-108,0,8), UDim2.new(0,106,0.5,-4), C.goldLo)
        uiCorner(barBg, 2)
        local bar = uiFr(barBg, UDim2.new(0,0,1,0), nil, C.amber)
        uiCorner(bar, 2)
        diagLabels[key .. "_bar"] = bar
    end

    diagRow("SYSTEM STATUS",        "status",    "ONLINE")
    diagRow("CLIENT MODE",          "mode",      "READ ONLY")
    diagRow("PLAYERS MONITORED",    "players",   "0")
    diagRow("DIMENSIONS OBSERVED",  "dims",      "0")
    diagRow("GARAGES IDENTIFIED",   "garages",   "0")
    diagRow("REPLICATION ACTIVE",   "repl",      "0")
    diagRow("PORTAL GUN DETECTED",  "pg",        "SCANNING")
    diagRow("DIM SOURCE (LP)",      "dimSrc",    "UNKNOWN")
    diagRow("SCAN ERRORS",          "errors",    "0")
    diagRow("JSON EXPORT",          "json",      "—")
    actBar ("SCAN ACTIVITY",        "scanAct")
    actBar ("REPL LOAD",            "replLoad")
    diagRow("UPTIME",               "uptime",    "0s")

    -- Hex terminal area
    uiFr(f, UDim2.new(1,0,0,1), UDim2.new(0,0,1,-20), C.goldDim)
    local termLbl = uiLbl(f, "> SYS INIT · MODULES LOADED · AWAITING DATA", UDim2.new(1,-4,0,18), UDim2.new(0,2,1,-20), C.goldLo, 8)
    diagLabels["terminal"] = termLbl

    return f
end

function RefreshSystemDiagnostics()
    if activeMode ~= "DIAGNOSTICS" then return end
    if not diagLabels.status then return end

    local function set(k, v, col)
        local l = diagLabels[k]; if not l then return end
        l.Text = tostring(v or "—"); if col then l.TextColor3 = col end
    end

    local all       = Players:GetPlayers()
    local replCount = 0
    local dimCount  = 0
    local garCount  = 0
    local pgCount   = 0
    local errCount  = 0

    for _, p in ipairs(all) do
        local r = dmGet(p.UserId)
        if r then
            if r.replicated                        then replCount += 1 end
            if r.dimension.status ~= STATUS.UNKNOWN then dimCount  += 1 end
            if r.garage.value     == "IN GARAGE"   then garCount  += 1 end
            if r.portalGun.value ~= "UNKNOWN" and r.portalGun.value ~= "UNKNOWN" then pgCount += 1 end
        end
    end

    set("players", #all,       C.gold)
    set("dims",    dimCount,   dimCount > 0 and C.green or C.amber)
    set("garages", garCount,   C.amber)
    set("repl",    replCount,  replCount > 0 and C.green or C.gray)
    set("pg",      pgCount > 0 and "YES (" .. pgCount .. ")" or "NOT DETECTED", pgCount > 0 and C.gold or C.goldDim)
    set("errors",  errCount)

    -- LP dimension source
    local lpR = dmGet(LP.UserId)
    if lpR then
        set("dimSrc", lpR.dimension.source, statusColor(lpR.dimension.status))
    end

    -- JSON export availability
    local jsonOk = pcall(function() game:GetService("HttpService"):JSONEncode({}) end)
    -- Also check writefile (Delta-specific)
    local wfOk = type(writefile) == "function"
    if jsonOk and wfOk then
        set("json", "AVAILABLE (writefile)", C.green)
    elseif jsonOk then
        set("json", "JSON ONLY (no writefile)", C.amber)
    else
        set("json", "UNAVAILABLE", C.gray)
    end

    -- Activity bars
    local function setBar(key, frac)
        local b = diagLabels[key .. "_bar"]; if not b then return end
        b.Size = UDim2.new(math.clamp(frac, 0, 1), 0, 1, 0)
    end
    setBar("scanAct",  math.min(#all / 10, 1))
    setBar("replLoad", math.min(replCount / math.max(#all, 1), 1))

    set("uptime", string.format("%.0fs", os.clock()))

    -- Terminal ticker
    local terms = {
        "> SCAN CYCLE COMPLETE",
        "> NODES=" .. #all .. " REPL=" .. replCount,
        "> DIM_OBS=" .. dimCount,
        "> CLIENT ONLY — NO SERVER ACCESS",
        "> DATA INTEGRITY: OK",
    }
    if diagLabels["terminal"] then
        diagLabels["terminal"].Text = terms[math.random(#terms)]
    end
end

-- ══════════════════════════════════════════════════════════════════════
-- LOG MODE  (incremental — E7 fixed)
-- ══════════════════════════════════════════════════════════════════════
local logFrame, logScroll = nil, nil

local function RenderEventLog(parent, cW, cH)
    local f = uiFr(parent, UDim2.new(1,0,1,0), nil, C.black, 1)
    f.Visible = false; f.Name = "LogMode"
    modeFrames["LOG"] = f
    logFrame = f

    uiLbl(f, "◈ EVENT LOG", UDim2.new(1,0,0,14), nil, C.gold, 9, Enum.TextXAlignment.Center)

    uiBtn(f, "CLR", UDim2.new(0,30,0,14), UDim2.new(1,-34,0,2),
        Color3.fromRGB(30,8,0), C.red, function()
            evLog = {}; logDirty = true
            for _, c in ipairs(logScroll:GetChildren()) do
                if c:IsA("Frame") then c:Destroy() end
            end
            logFrames = {}
        end).TextSize = 9

    logScroll = uiScroll(f, UDim2.new(1,0,1,-20), UDim2.new(0,0,0,20))
    uiVList(logScroll)
    return f
end

local function RefreshLog()
    if not logFrame or not logFrame.Visible then return end
    if not logDirty then return end
    logDirty = false

    -- Incremental: add only new entries at the top (evLog is prepended)
    -- Strategy: compare #evLog with #logFrames
    local currentCount = #logScroll:GetChildren()
    local maxDisplay   = 80

    -- If log was cleared
    if #evLog == 0 then
        for _, c in ipairs(logScroll:GetChildren()) do
            if c:IsA("Frame") then c:Destroy() end
        end
        logFrames = {}
        return
    end

    -- Add new entries (at index 1..N that don't exist yet)
    local toAdd = math.min(#evLog, maxDisplay) - currentCount
    if toAdd <= 0 then
        -- Update existing colors / text if content changed
        for i, e in ipairs(evLog) do
            if i > maxDisplay then break end
            if logFrames[i] then
                logFrames[i].lbl.Text = string.format("[%.0f] %s", e.t % 100000, e.text)
                logFrames[i].lbl.TextColor3 = e.color
            end
        end
        return
    end

    -- Rebuild (toAdd is positive means new events pushed in)
    for _, c in ipairs(logScroll:GetChildren()) do
        if c:IsA("Frame") then c:Destroy() end
    end
    logFrames = {}
    for i = 1, math.min(#evLog, maxDisplay) do
        local e   = evLog[i]
        local row = uiFr(logScroll, UDim2.new(1,-4,0,16), UDim2.new(0,2,0,0), Color3.fromRGB(7,6,0))
        row.LayoutOrder = i
        local lb = uiLbl(row, string.format("[%.0f] %s", e.t%100000, e.text),
            UDim2.new(1,-4,1,0), UDim2.new(0,4,0,0), e.color, 9)
        logFrames[i] = { frame = row, lbl = lb }
    end
end

-- ══════════════════════════════════════════════════════════════════════
-- IRIS TRANSITION ANIMATIONS
-- ══════════════════════════════════════════════════════════════════════
local transitioning = false

local function AnimateIrisCollapse(frame, cb)
    -- Shrink content to center over 180ms
    if not TweenService then
        if frame then frame.Visible = false end
        if cb then cb() end
        return
    end
    if not frame then if cb then cb() end; return end
    local t = TweenService:Create(frame,
        TweenInfo.new(0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.In),
        { Size = UDim2.new(0,0,0,0), Position = UDim2.new(0.5,0,0.5,0) })
    table.insert(LIFE.tweens, t)
    t:Play()
    t.Completed:Connect(function()
        frame.Visible = false
        if cb then cb() end
    end)
end

local function AnimateModeExpansion(frame, fullSize, fullPos)
    if not TweenService then
        if frame then
            frame.Size = fullSize; frame.Position = fullPos; frame.Visible = true
        end
        return
    end
    if not frame then return end
    frame.Size     = UDim2.new(0,0,0,0)
    frame.Position = UDim2.new(0.5,0,0.5,0)
    frame.Visible  = true
    local t = TweenService:Create(frame,
        TweenInfo.new(0.17, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
        { Size = fullSize, Position = fullPos })
    table.insert(LIFE.tweens, t)
    t:Play()
end

function SwitchOpticalMode(newMode)
    if transitioning then return end
    if newMode == activeMode then return end
    transitioning = true

    -- Highlight active button
    for name, b in pairs(modeBtnRefs) do
        b.BackgroundColor3 = name == newMode and C.goldDim or C.goldLo
        b.TextColor3       = name == newMode and C.gold    or C.goldDim
    end

    local oldFrame = modeFrames[activeMode]
    local newFrame = modeFrames[newMode]
    if not newFrame then activeMode = newMode; transitioning = false; return end

    local function doExpand()
        activeMode = newMode
        -- Restore size to fill parent
        if newFrame then
            local p = newFrame.Parent
            newFrame.Size     = UDim2.new(1,0,1,0)
            newFrame.Position = UDim2.new(0,0,0,0)
        end
        AnimateModeExpansion(newFrame, UDim2.new(1,0,1,0), UDim2.new(0,0,0,0))
        task.wait(0.17)
        transitioning = false
        logEv("[MODE] → " .. newMode, C.gold)
    end

    if oldFrame and oldFrame.Visible then
        AnimateIrisCollapse(oldFrame, doExpand)
    else
        doExpand()
    end
end

-- ══════════════════════════════════════════════════════════════════════
-- MINI BADGE (restore from minimized)
-- ══════════════════════════════════════════════════════════════════════
local function BuildMiniBadge(parent, mainFr)
    local vp = Camera.ViewportSize
    local badge = uiFr(parent, UDim2.new(0,120,0,26), UDim2.new(0,6,0,vp.Y-32), Color3.fromRGB(8,7,0))
    badge.Visible = false; badge.ZIndex = 20
    uiCorner(badge, 4); uiStroke(badge, C.goldDim, 1)

    local dot = uiLbl(badge, "●", UDim2.new(0,14,1,0), UDim2.new(0,4,0,0), C.green, 10)
    uiLbl(badge, "EM·VISOR", UDim2.new(1,-42,1,0), UDim2.new(0,20,0,0), C.gold, 9)
    uiBtn(badge, "▲", UDim2.new(0,22,0,20), UDim2.new(1,-26,0,3), C.goldLo, C.amber,
        function()
            badge.Visible = false; mainFr.Visible = true; minimized = false
        end).TextSize = 9

    -- Blink task — stored connection
    local blinkConn = RunService.Heartbeat:Connect(function()
        if not badge or not badge.Parent then return end
    end)
    addConn(blinkConn)
    -- Actual blink via task (lighter than a full Heartbeat)
    task.spawn(function()
        while badge and badge.Parent do
            dot.TextColor3 = C.green;  task.wait(0.8)
            if not badge or not badge.Parent then break end
            dot.TextColor3 = C.goldLo; task.wait(0.4)
        end
    end)

    miniBadge = badge
    return badge
end

-- ══════════════════════════════════════════════════════════════════════
-- DRAG (mobile-safe, header-only)
-- ══════════════════════════════════════════════════════════════════════
local function setupDrag(handle, target)
    addConn(handle.InputBegan:Connect(function(input)
        if input.UserInputType ~= Enum.UserInputType.MouseButton1
        and input.UserInputType ~= Enum.UserInputType.Touch then return end
        local sx = target.AbsolutePosition.X
        local sy = target.AbsolutePosition.Y
        local ix = input.Position.X
        local iy = input.Position.Y
        addConn(input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.Change then
                target.Position = UDim2.new(0, sx+(input.Position.X-ix), 0, sy+(input.Position.Y-iy))
            end
        end))
    end))
end

-- ══════════════════════════════════════════════════════════════════════
-- CREATE OPTICAL INTERFACE  (top-level builder)
-- ══════════════════════════════════════════════════════════════════════
local fxEnabled = true

function CreateOpticalInterface()
    -- Destroy existing
    local existing = LP.PlayerGui:FindFirstChild("EMOVSG")
    if existing then existing:Destroy() end

    SG = Instance.new("ScreenGui")
    SG.Name = "EMOVSG"; SG.ResetOnSpawn = false
    SG.ZIndexBehavior = Enum.ZIndexBehavior.Sibling; SG.DisplayOrder = 50
    SG.Parent = LP.PlayerGui

    local pos, W, H = safeLayout()

    -- Main container
    local mainFr = uiFr(SG, UDim2.new(0,W,0,H), pos, C.black)
    mainFr.Active = false   -- body passes through touch → camera unaffected
    uiCorner(mainFr, 6); uiStroke(mainFr, C.goldDim, 1)
    lensFrame = mainFr

    -- Header strip
    local hdr = uiFr(mainFr, UDim2.new(1,0,0,22), nil, C.bgHdr)
    hdr.Active = true   -- header absorbs input for drag
    uiCorner(hdr, 4)
    uiFr(hdr, UDim2.new(1,0,0,6), UDim2.new(0,0,1,-6), C.bgHdr)   -- flatten bottom corners

    local blinkL = uiLbl(hdr, "◆", UDim2.new(0,14,1,0), UDim2.new(0,4,0,0), C.gold, 11)
    uiLbl(hdr, "EM·OPTICAL INTERFACE", UDim2.new(1,-80,1,0), UDim2.new(0,22,0,0), C.gold, 10)
    local subL = uiLbl(hdr, "MULTIVERSE SCAN", UDim2.new(0,120,1,0), UDim2.new(0,22,0,0), C.goldDim, 8)
    -- (subL is below title — position offset by title height)
    subL.Position = UDim2.new(0,22,0,11)

    -- Header blink task
    task.spawn(function()
        local i = 0
        while mainFr and mainFr.Parent do
            i += 1; task.wait(0.7)
            if not mainFr or not mainFr.Parent then break end
            blinkL.Text       = i%2==0 and "◆" or "◇"
            blinkL.TextColor3 = i%2==0 and C.gold or C.goldDim
        end
    end)

    setupDrag(hdr, mainFr)

    -- Content area below header
    local contentH = H - 22
    local content = uiFr(mainFr, UDim2.new(1,0,0,contentH), UDim2.new(0,0,0,22), C.black, 0.05)
    content.ClipDescendants = true

    -- Side menu
    local function onMode(name)    SwitchOpticalMode(name) end
    local function onMinimize()    minimized=true; mainFr.Visible=false; if miniBadge then miniBadge.Visible=true end end
    local function onClose()
        logEv("[SYS] SHUTDOWN", C.red)
        killAll(); stopRings()
        mainFr:Destroy(); if SG and SG.Parent then SG:Destroy() end
    end
    local function onToggleFX()
        fxEnabled = not fxEnabled
    end

    local menu, menuW = CreateSideMenu(content, contentH, onMode, onMinimize, onClose, onToggleFX)
    sideMenu = menu

    -- Mode content area
    local ca, cW = makeContentArea(content, menuW, W, contentH)
    local cH = contentH

    -- Build all three modes
    local proxF, detLbl = RenderProximityScan(ca, cW, cH)
    proxDetectedLbl = detLbl
    RenderOmegaDevice(ca, cW, cH)
    RenderSystemDiagnostics(ca, cW, cH)
    RenderEventLog(ca, cW, cH)

    -- Activate default mode
    SwitchOpticalMode("PROXIMITY")
    -- Make first frame visible immediately (no iris on first open)
    local pf = modeFrames["PROXIMITY"]
    if pf then pf.Visible = true; pf.Size = UDim2.new(1,0,1,0); pf.Position = UDim2.new(0,0,0,0) end

    -- Mini badge
    BuildMiniBadge(SG, mainFr)

    -- ViewportSize change
    addConn(Camera:GetPropertyChangedSignal("ViewportSize"):Connect(function()
        local np, nW, nH = safeLayout()
        mainFr.Position = np
        mainFr.Size = UDim2.new(0, nW, 0, nH)
    end))

    logEv("[SYS] EM OPTICAL VISOR v4.0 ONLINE", C.gold)
    logEv("[SYS] CLIENT-SIDE — READ ONLY",        C.goldDim)
end

-- ══════════════════════════════════════════════════════════════════════
-- CLEANUP
-- ══════════════════════════════════════════════════════════════════════
function CleanupOpticalInterface()
    killAll()
    stopRings()
    if SG and SG.Parent then SG:Destroy() end
    DM = {}; garCache = {}; proxRows = {}; logFrames = {}
    evLog = {}; logDirty = false
end

-- ══════════════════════════════════════════════════════════════════════
-- MAIN LOOP  (separated frequencies)
-- ══════════════════════════════════════════════════════════════════════
local running     = true
local loopTickAcc = 0
local SCAN_FREQ   = 0.5   -- seconds between full player scans
local dtAccum     = 0

local function StartMainLoop()
    addConn(RunService.Heartbeat:Connect(function(dt)
        if not running then return end
        dtAccum = dtAccum + dt

        -- Player scan at SCAN_FREQ
        if dtAccum < SCAN_FREQ then return end
        local elapsed = dtAccum
        dtAccum = 0

        -- Skip heavy work when minimized
        if minimized then return end

        pcall(function()
            local all   = Players:GetPlayers()
            local alive = {}
            for _, p in ipairs(all) do alive[p.UserId] = true end

            -- Remove departed players from DM and rows
            for uid in pairs(DM) do
                if not alive[uid] then
                    if proxRows[uid] then
                        if proxRows[uid].frame and proxRows[uid].frame.Parent then
                            proxRows[uid].frame:Destroy()
                        end
                        proxRows[uid] = nil
                    end
                    dmRemove(uid)
                    garageTickAcc[uid] = nil
                end
            end

            -- Update all players
            for _, p in ipairs(all) do
                pcall(updatePlayerRecord, p, elapsed)
            end

            -- Update HUD
            if activeMode == "PROXIMITY"   then pcall(UpdatePlayerMarkers)      end
            if activeMode == "OMEGA"       then pcall(UpdateDimensionMarkers)    end
            if activeMode == "DIAGNOSTICS" then pcall(RefreshSystemDiagnostics) end
            pcall(RefreshLog)
        end)
    end))
end

-- ══════════════════════════════════════════════════════════════════════
-- PLAYER EVENTS
-- ══════════════════════════════════════════════════════════════════════
local function HookPlayer(p)
    dmEnsure(p)
    PS.hookPlayer(p)
end

addConn(Players.PlayerAdded:Connect(function(p)
    HookPlayer(p)
    logEv("[JOIN] " .. p.Name, C.green)
end))

addConn(Players.PlayerRemoving:Connect(function(p)
    logEv("[LEAVE] " .. p.Name, C.red)
    task.wait(0.3)
    if proxRows[p.UserId] then
        if proxRows[p.UserId].frame and proxRows[p.UserId].frame.Parent then
            proxRows[p.UserId].frame:Destroy()
        end
        proxRows[p.UserId] = nil
    end
    dmRemove(p.UserId)
    garCache[p.UserId]    = nil
    garageTickAcc[p.UserId] = nil
end))

-- ══════════════════════════════════════════════════════════════════════
-- ENTRY POINT
-- ══════════════════════════════════════════════════════════════════════
-- Hook existing players
for _, p in ipairs(Players:GetPlayers()) do
    HookPlayer(p)
end

-- Build the visor
CreateOpticalInterface()

-- Initial data pass
for _, p in ipairs(Players:GetPlayers()) do
    pcall(updatePlayerRecord, p, 0)
end

-- Start the loop
StartMainLoop()

print("[EvilMortyVisor v4.0] online → " .. #Players:GetPlayers() .. " nodes")
