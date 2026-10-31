-- russo | integrated feature build (intro removed)
if _G.BloodHoundsRunning then return end
_G.BloodHoundsUnloaded = false
_G.BloodHoundsRunning = true
repeat task.wait() until game:IsLoaded()

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local TS = game:GetService("TweenService")
local Lighting = game:GetService("Lighting")
local HS = game:GetService("HttpService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")
local LP = Players.LocalPlayer
local camera = workspace.CurrentCamera

local _tick          = tick
local _clamp         = math.clamp
local _floor         = math.floor
local _abs           = math.abs
local _huge          = math.huge
local _sqrt          = math.sqrt
local _V3new         = Vector3.new
local _V3zero        = Vector3.zero
local _CFnew         = CFrame.new
local _CFlookAt      = CFrame.lookAt
local _RayParams_new = RaycastParams.new

local _GetPlayersCached
do
    local cache, cacheTime = nil, 0
    _GetPlayersCached = function()
        local now = _tick()
        if cache and now - cacheTime < 0.03 then return cache end
        cache = Players:GetPlayers()
        cacheTime = now
        return cache
    end
end

local function waitForCharReady(char, timeout)
    timeout = timeout or 5
    local deadline = _tick() + timeout
    while (not char) or (not char.Parent)
          or (not char:FindFirstChild("HumanoidRootPart"))
          or (not char:FindFirstChildOfClass("Humanoid")) do
        if _tick() > deadline then return false end
        task.wait(0.05)
    end
    return true
end

NS = 60
CS = 29
LAGGER_SPEED = 15
LAGGER_CARRY_SPEED = 24.5
MEDUSA_COOLDOWN = 25
BAT_AIMBOT_SPEED = 58
BYPASS_AIMBOT_SPEED = 60
MOBILE_PANEL_WIDTH = 128
MOBILE_PANEL_HEIGHT = 294
CONFIG_FILE = "FictionHub.json"
BAT_V2_HIT_DIST = 4.5
_isDraggingButton = false

backgroundIndex = 1
backgroundImageTransparency = 0.35
floatingButtonScale = 1
progressBarScale = 1
_floatingUIScales = {}

local CarrySystem = {
    normalSpeed = NS,
    carrySpeed = CS,
    laggerSpeed = LAGGER_SPEED,
    laggerCarrySpeed = LAGGER_CARRY_SPEED,
    speedToggled = false,
    laggerMode = 0,
    softStealEnabled = false,
    autoCarryEnabled = false,
    softStealRadius = 10,
    softStealSpeed = CS,
    softStealLatched = false,
    _isCarrying = false,
    _lastCarryCheck = 0,
    _lvBoost = nil,
    _lvAtt = nil,
    _blockedTime = 0,
    _maxForce = 2200,
    _freeForce = 500,
    _heartbeatConn = nil,
    _softStealScanner = nil,
    _softStealAnimals = {},
    _softStealScanning = false,
    _state = nil,
    _dropInProgress = false,
    _batAimbotToggled = false,
    _rayParams = nil,
    _rayFilter = nil,
    _rayFilterTime = 0,
}

function CarrySystem:isCarrying()
    local now = _tick()
    if now - (self._lastCarryCheck or 0) < 0.1 then
        return self._isCarrying
    end
    self._lastCarryCheck = now
    local char = LP.Character
    if not char then self._isCarrying = false; return false end
    local hum = char:FindFirstChildOfClass("Humanoid")
    local ws = hum and hum.WalkSpeed or 16
    local bySpeed = (ws < 25 and ws > 0)
    local byAttr = false
    local ok, v = pcall(function() return LP:GetAttribute("Stealing") end)
    if ok and v == true then byAttr = true end
    local ok2, v2 = pcall(function() return char:GetAttribute("Stealing") end)
    if ok2 and v2 == true then byAttr = true end
    if not byAttr then
        for _, name in ipairs({"Carrying","IsCarrying","Grabbed","Holding","StealHold","HasGrab"}) do
            local obj = char:FindFirstChild(name)
            if obj then
                if (obj:IsA("BoolValue") and obj.Value) or
                   (obj:IsA("ObjectValue") and obj.Value) or
                   (obj:IsA("StringValue") and obj.Value ~= "") then
                    byAttr = true; break
                end
            end
        end
    end
    self._isCarrying = bySpeed or byAttr
    return self._isCarrying
end

function CarrySystem:getActiveSpeed()
    if self._state and (self._state.autoLeftEnabled or self._state.autoRightEnabled) then
        return self.normalSpeed
    end
    -- Explicitly selected Lagger modes take priority over Auto Carry/Soft Steal.
    if self.laggerMode == 1 then return self.laggerSpeed end
    if self.laggerMode == 2 then return self.laggerCarrySpeed end
    if self.autoCarryEnabled then
        return self:isCarrying() and self.carrySpeed or self.normalSpeed
    end
    if self.softStealEnabled then
        local _, dist = self:getNearestSoftStealAnimal(self.softStealRadius)
        local inRange = dist and dist <= self.softStealRadius
        if inRange then
            self.softStealLatched = true
            return self.carrySpeed
        end
        if self.softStealLatched and self:isCarrying() then
            return self.carrySpeed
        else
            self.softStealLatched = false
        end
    end
    if self.speedToggled then return self.carrySpeed end
    return self.normalSpeed
end

function CarrySystem:getStatus()
    if self._state and (self._state.autoLeftEnabled or self._state.autoRightEnabled) then
        return "NORMAL", self.normalSpeed
    end
    if self.laggerMode == 1 then return "LAGGER", self.laggerSpeed end
    if self.laggerMode == 2 then return "LAGGER CARRY", self.laggerCarrySpeed end
    if self.autoCarryEnabled then
        if self:isCarrying() then return "AUTO CARRY", self.carrySpeed end
        return "NORMAL", self.normalSpeed
    end
    if self.softStealEnabled then
        local _, dist = self:getNearestSoftStealAnimal(self.softStealRadius)
        local inRange = dist and dist <= self.softStealRadius
        if inRange or (self.softStealLatched and self:isCarrying()) then
            return "AUTO CARRY", self.carrySpeed
        end
    end
    if self.speedToggled then return "CARRY", self.carrySpeed end
    return "NORMAL", self.normalSpeed
end

local function destroyLV()
    if CarrySystem._lvBoost and CarrySystem._lvBoost.Parent then pcall(function() CarrySystem._lvBoost:Destroy() end) end
    if CarrySystem._lvAtt and CarrySystem._lvAtt.Parent then pcall(function() CarrySystem._lvAtt:Destroy() end) end
    CarrySystem._lvBoost = nil; CarrySystem._lvAtt = nil
end

local function setupLV(hrp)
    if CarrySystem._lvBoost and CarrySystem._lvBoost.Parent == hrp then return end
    destroyLV()
    local att = Instance.new("Attachment"); att.Parent = hrp
    local lv = Instance.new("LinearVelocity")
    lv.Name = "CarryBoostLV"
    lv.Attachment0 = att
    lv.VelocityConstraintMode = Enum.VelocityConstraintMode.Plane
    lv.PrimaryTangentAxis = _V3new(1,0,0)
    lv.SecondaryTangentAxis = _V3new(0,0,1)
    lv.MaxForce = CarrySystem._maxForce
    lv.PlaneVelocity = Vector2.zero
    lv.RelativeTo = Enum.ActuatorRelativeTo.World
    lv.Parent = hrp
    CarrySystem._lvAtt = att
    CarrySystem._lvBoost = lv
    pcall(function() hrp:SetNetworkOwner(LP) end)
end

function CarrySystem:scanSoftStealAnimals()
    self._softStealAnimals = {}
    local plots = Workspace:FindFirstChild("Plots")
    if not plots then return end
    for _, plot in ipairs(plots:GetChildren()) do
        if plot:IsA("Model") then
            local podiums = plot:FindFirstChild("AnimalPodiums")
            if podiums then
                for _, podium in ipairs(podiums:GetChildren()) do
                    if podium:IsA("Model") then
                        local base = podium:FindFirstChild("Base")
                        local spawn = base and base:FindFirstChild("Spawn")
                        if spawn then
                            table.insert(self._softStealAnimals, {
                                plot = plot.Name,
                                slot = podium.Name,
                                worldPosition = spawn.Position,
                                uid = plot.Name .. "_" .. podium.Name,
                            })
                        end
                    end
                end
            end
        end
    end
end

function CarrySystem:startSoftStealScanner()
    if self._softStealScanner then return end
    self._softStealScanning = true
    self:scanSoftStealAnimals()
    self._softStealScanner = RunService.Heartbeat:Connect(function()
        if not self._softStealScanning then return end
        self:scanSoftStealAnimals()
    end)
end

function CarrySystem:stopSoftStealScanner()
    self._softStealScanning = false
    if self._softStealScanner then
        self._softStealScanner:Disconnect()
        self._softStealScanner = nil
    end
end

function CarrySystem:getNearestSoftStealAnimal(radius)
    local char = LP.Character
    local root = char and (char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("UpperTorso"))
    if not root then return nil, _huge end
    local best, bestDist = nil, _huge
    local animals = self._softStealAnimals
    local rpos = root.Position
    for i = 1, #animals do
        local data = animals[i]
        if data.worldPosition then
            local dx = rpos.X - data.worldPosition.X
            local dy = rpos.Y - data.worldPosition.Y
            local dz = rpos.Z - data.worldPosition.Z
            local dist = _sqrt(dx*dx + dy*dy + dz*dz)
            if dist < bestDist then
                best = data; bestDist = dist
            end
        end
    end
    if radius and bestDist > radius then return nil, bestDist end
    return best, bestDist
end

function CarrySystem:updateMovement(dt)
    local char = LP.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hrp or not hum then return end

    local speed = self:getActiveSpeed()
    local moveDir = hum.MoveDirection
    local moving = moveDir.Magnitude > 0.1

    local wallNormalFlat = nil
    if moving then
        if not self._rayParams then
            self._rayParams = _RayParams_new()
            self._rayParams.FilterType = Enum.RaycastFilterType.Exclude
            self._rayFilter = {}
            self._rayFilterTime = 0
        end
        local now = _tick()
        if now - self._rayFilterTime > 1 then
            self._rayFilterTime = now
            local filter = self._rayFilter
            while #filter > 0 do filter[#filter] = nil end
            filter[1] = char
            local plist = _GetPlayersCached()
            for i = 1, #plist do
                local p = plist[i]
                if p.Character then
                    filter[#filter + 1] = p.Character
                end
            end
            self._rayParams.FilterDescendantsInstances = filter
        else
            local filter = self._rayFilter
            local found = false
            for i = 1, #filter do
                if filter[i] == char then found = true; break end
            end
            if not found then
                filter[1] = char
                self._rayParams.FilterDescendantsInstances = filter
            end
        end
        local flatDir = _V3new(moveDir.X, 0, moveDir.Z).Unit
        local hit = Workspace:Raycast(hrp.Position + _V3new(0,1,0), flatDir * 2.5, self._rayParams)
        if hit and hit.Instance and hit.Instance.CanCollide then
            local nf = _V3new(hit.Normal.X, 0, hit.Normal.Z)
            if nf.Magnitude > 0.7 then wallNormalFlat = nf.Unit end
        end
    end

    local hVel = _V3new(hrp.AssemblyLinearVelocity.X, 0, hrp.AssemblyLinearVelocity.Z)
    local blocked = wallNormalFlat ~= nil or (moving and hVel.Magnitude < 2)

    if blocked then
        for _, model in ipairs(char:GetChildren()) do
            if model:IsA("Model") then
                for _, part in ipairs(model:GetDescendants()) do
                    if part:IsA("BasePart") and part.CanCollide then part.CanCollide = false end
                end
            end
        end
        for _, name in ipairs({"Carrying","IsCarrying","Grabbed","Holding","StealHold","HasGrab"}) do
            local v = char:FindFirstChild(name)
            if v and v:IsA("ObjectValue") and v.Value and v.Value:IsA("Model") then
                for _, part in ipairs(v.Value:GetDescendants()) do
                    if part:IsA("BasePart") and part.CanCollide then part.CanCollide = false end
                end
            end
        end
    end

    local state = hum:GetState()
    local ragdolled = state == Enum.HumanoidStateType.Physics
                   or state == Enum.HumanoidStateType.Ragdoll
                   or state == Enum.HumanoidStateType.FallingDown
    local moverBlocked = ragdolled or self._dropInProgress or self._batAimbotToggled

    if not moverBlocked then
        if not CarrySystem._lvBoost or CarrySystem._lvBoost.Parent ~= hrp then setupLV(hrp) end
        local lv = CarrySystem._lvBoost
        if lv then
            if not lv.Enabled then lv.Enabled = true end
            if moveDir.Magnitude > 0.1 then
                local flat = _V3new(moveDir.X, 0, moveDir.Z).Unit
                if wallNormalFlat then
                    local wanted = flat * speed
                    local along = wanted - wallNormalFlat * wanted:Dot(wallNormalFlat)
                    if along.Magnitude < 0.5 then
                        lv.PlaneVelocity = Vector2.zero
                    else
                        lv.PlaneVelocity = Vector2.new(along.X, along.Z)
                    end
                else
                    lv.PlaneVelocity = Vector2.new(flat.X * speed, flat.Z * speed)
                end
            else
                lv.PlaneVelocity = Vector2.zero
            end
            if blocked then
                self._blockedTime = self._blockedTime + (dt or 0.016)
            else
                self._blockedTime = 0
            end
            if self._blockedTime > 0.35 then
                if lv.MaxForce ~= self._freeForce then lv.MaxForce = self._freeForce end
            elseif lv.MaxForce ~= self._maxForce then
                lv.MaxForce = self._maxForce
            end
        end
    elseif CarrySystem._lvBoost then
        CarrySystem._lvBoost.PlaneVelocity = Vector2.zero
        if CarrySystem._lvBoost.Enabled then CarrySystem._lvBoost.Enabled = false end
    end
end

function CarrySystem:start()
    if self._heartbeatConn then return end
    self._heartbeatConn = RunService.Heartbeat:Connect(function(dt) self:updateMovement(dt) end)
    if self.softStealEnabled then self:startSoftStealScanner() end
end

function CarrySystem:stop()
    if self._heartbeatConn then
        self._heartbeatConn:Disconnect()
        self._heartbeatConn = nil
    end
    self:stopSoftStealScanner()
    destroyLV()
    self.softStealLatched = false
end

function CarrySystem:setNormalSpeed(v)
    self.normalSpeed = _clamp(v,1,500)
    NS = self.normalSpeed
end
function CarrySystem:setCarrySpeed(v)
    self.carrySpeed = _clamp(v,1,500)
    self.softStealSpeed = self.carrySpeed -- Soft Steal always follows the Hub Carry Speed
    CS = self.carrySpeed
end
function CarrySystem:setLaggerSpeed(v) self.laggerSpeed = _clamp(v,0.1,500) end
function CarrySystem:setLaggerCarrySpeed(v) self.laggerCarrySpeed = _clamp(v,0.1,500) end
function CarrySystem:setSoftStealSpeed(v) self:setCarrySpeed(v) end
function CarrySystem:setSoftStealRadius(v) self.softStealRadius = _clamp(v,1,200) end

function CarrySystem:toggleCarryMode() self.speedToggled = not self.speedToggled end
function CarrySystem:setLaggerMode(mode)
    if mode == 0 then self.laggerMode = 0
    elseif mode == 1 then self.laggerMode = 1
    elseif mode == 2 then self.laggerMode = 2 end
end
function CarrySystem:toggleLaggerMode()
    if self.laggerMode == 0 then self.laggerMode = 1
    elseif self.laggerMode == 1 then self.laggerMode = 2
    else self.laggerMode = 0 end
end
function CarrySystem:setSoftStealEnabled(enabled)
    self.softStealEnabled = enabled
    if enabled then self:startSoftStealScanner()
    else self:stopSoftStealScanner(); self.softStealLatched = false end
end
function CarrySystem:toggleSoftSteal() self:setSoftStealEnabled(not self.softStealEnabled) end
function CarrySystem:isRunning() return self._heartbeatConn ~= nil end
function CarrySystem:getCurrentSpeed() return self:getActiveSpeed() end

local COLOR_THEMES = {
    ["Gris"] = Color3.fromRGB(30, 30, 35),
}

currentColorTheme = "Gris"
selectedColor = COLOR_THEMES["Gris"]

function getThemeColor() return selectedColor end

local _lastThemeUpdate = 0
local _lastThemeColor = nil

function applyColorTheme(themeName)
    local color = COLOR_THEMES[themeName]
    if not color then return end
    currentColorTheme = themeName
    selectedColor = color
    updateAllUIThemeColors(color)
    saveAllSettings()
end

function updateAllUIThemeColors(color)
    local now = _tick()
    if color == _lastThemeColor and now - _lastThemeUpdate < 0.1 then return end
    _lastThemeUpdate = now
    _lastThemeColor = color

    if progressFill then
        progressFill.BackgroundColor3 = Color3.fromRGB(210, 210, 220)
        local fs = progressFill:FindFirstChild("FillStroke")
        if fs then fs.Color = Color3.fromRGB(210, 210, 220) end
        local grad = progressFill:FindFirstChildOfClass("UIGradient")
        if grad then
            grad.Color = ColorSequence.new({
                ColorSequenceKeypoint.new(0.00, Color3.fromRGB(150, 150, 160)),
                ColorSequenceKeypoint.new(0.50, Color3.fromRGB(210, 210, 220)),
                ColorSequenceKeypoint.new(1.00, Color3.fromRGB(230, 230, 235)),
            })
        end
    end
    if pbFrame then
        local pr = pbFrame:FindFirstChild("ProgressRow")
        if pr then
            local fr = pr:FindFirstChild("FillRegion")
            if fr then
                local s = fr:FindFirstChild("FillRegionStroke")
                if s then s.Color = Color3.fromRGB(210, 210, 220) end
            end
        end
        local discordLabel = pbFrame:FindFirstChild("DiscordLabel")
        if discordLabel then discordLabel.TextColor3 = Color3.fromRGB(210, 210, 220) end
        local fpsNeon = pbFrame:FindFirstChild("FPSNeon")
        if fpsNeon then fpsNeon.TextColor3 = Color3.fromRGB(210, 210, 220) end
    end
    local function searchAndUpdateText(parent)
        for _, child in ipairs(parent:GetDescendants()) do
            if child:IsA("TextLabel") then
                if child.Text:find("discord.gg") or child.Text:find("Spd:") or child.Name == "DiscordText" or
                   child.Name == "FictionHubSpeedIndicator" or child.Text:find("FPS") or child.Text:find("speed") then
                    child.TextColor3 = Color3.fromRGB(210, 210, 220)
                end
            end
            if child:IsA("UIStroke") then
                if child.Color == Color3.fromRGB(95, 95, 105) or child.Color == Color3.fromRGB(180, 180, 190) then child.Color = color end
            end
        end
    end
    if gui then searchAndUpdateText(gui) end
    if tpBatFloatingButton then
        paintFloatingBtn(tpBatFloatingButton:FindFirstChild("Frame"), batDesyncTpEnabled)
    end
    if batV2FloatingButton then
        paintFloatingBtn(batV2FloatingButton:FindFirstChild("Frame"), autoBatV2Enabled)
    end
    for _, tab in ipairs(tabButtons or {}) do
        if tab.TextColor3 == Color3.fromRGB(180, 180, 190) or tab.TextColor3 == Color3.fromRGB(95, 95, 105) then tab.TextColor3 = color end
    end
    for _, hl in pairs(espHighlightCache) do
        if hl then hl.FillColor = color; hl.OutlineColor = color end
    end
    for _, bb in pairs(espBillboardCache) do
        if bb then
            local img = bb:FindFirstChildOfClass("ImageLabel")
            if img then
                local stroke = img:FindFirstChildOfClass("UIStroke")
                if stroke then stroke.Color = color end
            end
        end
    end
    if main then
        local titleFrame = main:FindFirstChild("TitleFrame")
        if titleFrame then
            for _, child in ipairs(titleFrame:GetDescendants()) do
                if child:IsA("UIStroke") and (child.Color == Color3.fromRGB(95, 95, 105) or child.Color == Color3.fromRGB(180, 180, 190)) then
                    child.Color = color
                end
            end
        end
    end
    if miniBtn then
        miniBtn.TextColor3 = Color3.fromRGB(210, 210, 220)
    end

    local pGui = LP:FindFirstChild("PlayerGui")
    if pGui then
        local bb = pGui:FindFirstChild("RagCountdownBillboard")
        if bb then
            local lbl = bb:FindFirstChildOfClass("TextLabel")
            if lbl then
                lbl.TextColor3 = Color3.fromRGB(255, 255, 255)
                local grad = lbl:FindFirstChildOfClass("UIGradient")
                if grad then
                    grad.Color = ColorSequence.new({
                        ColorSequenceKeypoint.new(0, Color3.fromRGB(200, 200, 210)),
                        ColorSequenceKeypoint.new(0.3, Color3.new(1, 1, 1)),
                        ColorSequenceKeypoint.new(0.5, Color3.new(1, 1, 1)),
                        ColorSequenceKeypoint.new(0.7, Color3.new(1, 1, 1)),
                        ColorSequenceKeypoint.new(1, Color3.fromRGB(200, 200, 210)),
                    })
                end
            end
        end
    end

    if MobilePanel then
        local container = MobilePanel:FindFirstChild("FloatingPanel")
        if container then
            local btnContainer = container:FindFirstChild("ButtonsContainer")
            if btnContainer then
                for _, btn in ipairs(btnContainer:GetChildren()) do
                    if btn:IsA("TextButton") and btn:FindFirstChild("BtnGrad") then
                        paintFloatingBtn(btn, btn:GetAttribute("MobActive") == true)
                    end
                end
            end
        end
    end
end

-- ═══════════════════════════════════════════════════════════════
-- OUTFITS
-- ═══════════════════════════════════════════════════════════════
local ECLIPSE_SKIN_PRESETS = {
    V1 = { shirt = 74707712629633, pants = 12405320750, hair = 140188532534398 },
    V2 = { shirt = 101796619834594, pants = 18975891159, hair = 115520061093937 },
    V3 = { shirt = 18552805597,   pants = 5414143509,  hair = 84008082880128  },
}

local function eclipseAssetUrl(id)
    return "http://www.roblox.com/asset/?id=" .. tostring(id)
end

local TAG = "LocalOutfit_"

local function loadObjects(id)
    local ok, res = pcall(function()
        return game:GetObjects("rbxassetid://" .. tostring(id))
    end)
    if ok and typeof(res) == "table" and #res > 0 then return res end
    ok, res = pcall(function()
        return game:GetService("InsertService"):LoadAsset(id)
    end)
    if ok and res then return { res } end
    return nil
end

local function collectParts(objs)
    local out = {}
    for _, o in ipairs(objs) do
        if o:IsA("BasePart") then out[#out + 1] = o end
        for _, d in ipairs(o:GetDescendants()) do
            if d:IsA("BasePart") then out[#out + 1] = d end
        end
    end
    return out
end

local function findAtt(char, name)
    for _, p in ipairs(char:GetDescendants()) do
        if p:IsA("Attachment") and p.Name == name and p.Parent:IsA("BasePart") then
            return p
        end
    end
end

local function applyNoOutfit(char)
    if not char then char = LP.Character end
    if not char then return end
    for _, d in ipairs(char:GetChildren()) do
        if d.Name:sub(1, #TAG) == TAG then pcall(function() d:Destroy() end) end
    end
    local oldAcc = char:FindFirstChild("AuFfitAccessory")
    if oldAcc then pcall(function() oldAcc:Destroy() end) end
    local oldKorblox = char:FindFirstChild("Korblox_RightLeg")
    if oldKorblox then pcall(function() oldKorblox:Destroy() end) end
    for _, d in ipairs(char:GetChildren()) do
        if d:IsA("CharacterMesh") and d.BodyPart == Enum.BodyPart.Head then
            pcall(function() d:Destroy() end)
        end
    end
    local head = char:FindFirstChild("Head")
    if head then
        head.Transparency = 0
        head.CanCollide = true
        head.LocalTransparencyModifier = 0
        local face = head:FindFirstChild("face")
        if face then face.Transparency = 0 end
        local sm = head:FindFirstChildWhichIsA("SpecialMesh")
        if sm then pcall(function() sm:Destroy() end) end
    end
    pcall(function()
        local neck = char:FindFirstChild("Neck")
        if neck then neck.Enabled = true end
    end)
    for _, partName in ipairs({"RightUpperLeg", "RightLowerLeg", "RightFoot"}) do
        local limb = char:FindFirstChild(partName)
        if limb then limb.Transparency = 0 end
    end
    local shirt = char:FindFirstChildWhichIsA("Shirt")
    if shirt then pcall(function() shirt:Destroy() end) end
    local pants = char:FindFirstChildWhichIsA("Pants")
    if pants then pcall(function() pants:Destroy() end) end
    for _, a in ipairs(char:GetChildren()) do
        if a:IsA("Accessory") then
            local h = a:FindFirstChild("Handle")
            if h then h.Transparency = 0 end
        end
    end
end

local OUTFITS = {
    {
        accessory = 121097973925756,
        offset = _V3new(0, 0.9, 0),
        shirt = eclipseAssetUrl(ECLIPSE_SKIN_PRESETS.V1.shirt),
        pants = eclipseAssetUrl(ECLIPSE_SKIN_PRESETS.V1.pants),
        headMesh = "http://www.roblox.com/asset/?id=134079402",
        headTexture = "http://www.roblox.com/asset/?id=133940918",
        korblox = "none",
        label = "Eclipse V1",
        headlessKorblox = true,
    },
    {
        label = "NO",
        customApply = applyNoOutfit,
    }
}
local currentOutfitIndex = 1
local outfitSelectorLabel = nil

BACKGROUND_IMAGES = {
    { label = "Off",  id = nil },
    { label = "BG 1", id = "rbxassetid://77785523954153" },
    { label = "BG 2", id = "rbxassetid://122815453745063" },
}
backgroundSelectorLabel = nil
backgroundImage = nil
backgroundImagePB = nil

function applyBackground(index)
    backgroundIndex = _clamp(index or 1, 1, #BACKGROUND_IMAGES)
    local cfg = BACKGROUND_IMAGES[backgroundIndex]
    if not cfg then return end
    local targetImg = cfg.id or ""
    local targetTrans = cfg.id and backgroundImageTransparency or 1
    if backgroundImage and backgroundImage.Parent then
        backgroundImage.Image = targetImg
        backgroundImage.ImageTransparency = targetTrans
    end
    if backgroundImagePB and backgroundImagePB.Parent then
        backgroundImagePB.Image = targetImg
        backgroundImagePB.ImageTransparency = targetImg ~= "" and math.min(targetTrans + 0.15, 1) or 1
    end
    if backgroundSelectorLabel then
        backgroundSelectorLabel.Text = cfg.label
    end
end

local function loadObjectsStd(id)
    local ok, res = pcall(function() return game:GetObjects("rbxassetid://" .. tostring(id)) end)
    if ok and typeof(res) == "table" and #res > 0 then return res end
    ok, res = pcall(function() return game:GetService("InsertService"):LoadAsset(id) end)
    if ok and res then return {res} end
    return nil
end

local function applyHeadlessKorblox(char)
    if not char then return end
    pcall(function() LP.CharacterAvatarType = Enum.AvatarType.R6 end)
    local head = char:FindFirstChild("Head")
    if head then
        head.Transparency = 1
        head.CanCollide = false
        head.LocalTransparencyModifier = 1
        local face = head:FindFirstChild("face")
        if face then face.Transparency = 1 end
    end
    pcall(function()
        local neck = char:FindFirstChild("Neck")
        if neck then neck.Enabled = false end
    end)
    for _, v in pairs(char:GetChildren()) do
        if v:IsA("Accessory") then
            local w = v:FindFirstChildWhichIsA("Weld") or v:FindFirstChildWhichIsA("WeldConstraint") or v:FindFirstChildWhichIsA("Motor6D")
            if w then
                local p0, p1 = w.Part0, w.Part1
                if (p0 and p0.Name == "Head") or (p1 and p1.Name == "Head") then
                    v.Parent = nil
                end
            end
        end
    end
    local rightLegConfig = {
        id = "rbxassetid://139607718",
        targetBodyPart = "RightUpperLeg",
        partsToHide = {"RightUpperLeg", "RightLowerLeg", "RightFoot"},
        scale = _V3new(1, 1, 1),
        offset = _CFnew(0, 0, 0)
    }
    local targetPart = char:FindFirstChild(rightLegConfig.targetBodyPart)
    if targetPart then
        local oldAsset = char:FindFirstChild("Korblox_RightLeg")
        if oldAsset then oldAsset:Destroy() end
        for _, partName in ipairs(rightLegConfig.partsToHide) do
            local limb = char:FindFirstChild(partName)
            if limb and limb:IsA("BasePart") then limb.Transparency = 1 end
        end
        local success, objects = pcall(function() return game:GetObjects(rightLegConfig.id) end)
        if success and objects and #objects > 0 then
            local assetModel = objects[1]
            assetModel.Name = "Korblox_RightLeg"
            local mainMesh = assetModel:IsA("BasePart") and assetModel or assetModel:FindFirstChildWhichIsA("BasePart", true)
            if mainMesh then
                mainMesh.Size = mainMesh.Size * rightLegConfig.scale
                mainMesh.CanCollide = false
                mainMesh.CFrame = targetPart.CFrame * rightLegConfig.offset
                local weld = Instance.new("WeldConstraint")
                weld.Part0 = targetPart
                weld.Part1 = mainMesh
                weld.Parent = mainMesh
                assetModel.Parent = char
            end
        end
    end
end

function applyOutfitByIndex(index)
    local cfg = OUTFITS[index]
    if not cfg then return end
    local char = LP.Character
    if not char then return end
    if cfg.customApply then
        cfg.customApply(char)
        if outfitSelectorLabel then outfitSelectorLabel.Text = cfg.label end
        return
    end
    char:WaitForChild("Head", 5)
    local head = char:FindFirstChild("Head")
    if not head then return end
    for _, d in ipairs(char:GetChildren()) do
        if d:IsA("CharacterMesh") and d.BodyPart == Enum.BodyPart.Head then
            pcall(function() d:Destroy() end)
        end
    end
    local done = false
    if head:IsA("MeshPart") then
        done = pcall(function()
            head.MeshId = cfg.headMesh
            if cfg.headTexture then head.TextureID = cfg.headTexture end
        end)
    end
    if not done then
        local sm = head:FindFirstChildWhichIsA("SpecialMesh") or Instance.new("SpecialMesh")
        sm.Parent = head
        sm.MeshType = Enum.MeshType.FileMesh
        sm.MeshId = cfg.headMesh
        sm.TextureId = cfg.headTexture or ""
    end
    if cfg.shirt then
        local s = char:FindFirstChildWhichIsA("Shirt") or Instance.new("Shirt")
        s.Name = "Shirt"
        s.ShirtTemplate = cfg.shirt
        s.Parent = char
    end
    if cfg.pants then
        local p = char:FindFirstChildWhichIsA("Pants") or Instance.new("Pants")
        p.Name = "Pants"
        p.PantsTemplate = cfg.pants
        p.Parent = char
    end
    local old = char:FindFirstChild("AuFfitAccessory")
    if old then old:Destroy() end
    if cfg.accessory and head then
        local objs = loadObjectsStd(cfg.accessory)
        if objs then
            local handle
            for _, o in ipairs(objs) do
                if o:IsA("BasePart") then handle = o; break end
                local f = o:FindFirstChildWhichIsA("BasePart", true)
                if f then handle = f; break end
            end
            if handle then
                local h = handle:Clone()
                h.Name = "AuFfitAccessory"
                h.CanCollide = false
                h.Anchored = false
                h.Massless = true
                h.Parent = char
                local weld = Instance.new("Weld")
                weld.Part0 = head
                weld.Part1 = h
                weld.C0 = _CFnew(cfg.offset)
                weld.Parent = h
            end
            for _, o in ipairs(objs) do pcall(function() o:Destroy() end) end
        end
    end
    if cfg.headlessKorblox then
        applyHeadlessKorblox(char)
    else
        if char then
            local head2 = char:FindFirstChild("Head")
            if head2 then
                head2.Transparency = 0
                head2.CanCollide = true
                head2.LocalTransparencyModifier = 0
                local face2 = head2:FindFirstChild("face")
                if face2 then face2.Transparency = 0 end
            end
            pcall(function()
                local neck = char:FindFirstChild("Neck")
                if neck then neck.Enabled = true end
            end)
            for _, partName in ipairs({"RightUpperLeg", "RightLowerLeg", "RightFoot"}) do
                local limb = char:FindFirstChild(partName)
                if limb then limb.Transparency = 0 end
            end
            local oldKorblox = char:FindFirstChild("Korblox_RightLeg")
            if oldKorblox then oldKorblox:Destroy() end
        end
    end
    if outfitSelectorLabel then outfitSelectorLabel.Text = cfg.label end
end

speedMode = false
antiRagdollMode = "off"
antiDieEnabled = false
antiFlingEnabled = false
jumpEnabled = false
laggerToggled = false
laggerCarryToggled = false
batCounterEnabled = false
autoBatEnabled = false
dropMode = 1
antiLagEnabled = false
removeAccessoriesEnabled = false
stretchEnabled = false
stretchFOV = 120
uiLocked = true
editModeEnabled = false
uiScaleValue = 78
espEnabled = false
e01WarningEnabled = true
e01WarningStartedAt = nil
e01WarningGui = nil

antiKickEnabled = false
setSafeModeVisual = nil

infJumpEnabled = false
infJumpMode    = "HOLD"


savedProgressBarPos = nil
savedButtonPositions = {}
savedMobilePanelPos = nil
tpBatFloatingPos = nil
batV2FloatingPos = nil
instaResetFloatingPos = nil
instaResetFloatingButton = nil

neonWeatherEnabled = false
_originalLighting = nil
setNeonWeatherVisual = nil

currentAnimPack = "Off"
originalTryardAnims = nil
tryardHeartbeatConn = nil
animSelectorLabel = nil

-- ═══════════════════════════════════════════════════════════════
-- BAT BYPASS (antes Anti Bypass Aimbot) — Persecución con predicción de ping
-- ═══════════════════════════════════════════════════════════════
autoBatV2Enabled      = false
autoBatV2SetVisual    = nil

selectedAimbotMode    = "Normal"
autoSwingEnabled      = false

_G.AceAntiBypassAimbotSpeed       = _G.AceAntiBypassAimbotSpeed or 60
_G.AceAntiBypassLaggerAimbotSpeed = _G.AceAntiBypassLaggerAimbotSpeed or 40
_G.AceAntiBypassAimbotOn          = _G.AceAntiBypassAimbotOn or false
_G.AceNormalAimbotOn              = _G.AceNormalAimbotOn or false
_G.AceCurrentSpeedMode            = _G.AceCurrentSpeedMode or "Normal"

_G.AceAntiBypassAimbot = _G.AceAntiBypassAimbot or {
    conn           = nil,
    swingCooldown  = false,
    prevAutoRotate = nil,
}

_G.AceAntiBypassSlapList = _G.AceAntiBypassSlapList or {
    "Bat", "Slap", "Iron Slap", "Gold Slap", "Diamond Slap",
    "Emerald Slap", "Ruby Slap", "Dark Matter Slap",
    "Flame Slap", "Nuclear Slap", "Galaxy Slap", "Glitched Slap"
}

_G.AceSafeModeTryStart      = _G.AceSafeModeTryStart      or _G.AmbitiousSafeModeTryStart
_G.AceStopAutoTPForAction   = _G.AceStopAutoTPForAction   or function()
    if batDesyncTpEnabled and type(stopBatDesyncTp) == "function" then
        stopBatDesyncTp()
    end
end
_G.AceStopNormalAimbot      = _G.AceStopNormalAimbot      or function()
    if autoBatEnabled and type(disableAutoBat) == "function" then
        disableAutoBat()
    end
end

useCarrySystem = false
lastMoveDir = _V3zero

-- TP Bat legacy vars (compat)
tpBatVersion = 1
tpBatVersionLabel = nil
tpBatVersionPill = nil
tpBatDistanceBox = nil
_G.__tpBatV2Distance = 8

local CoreGui = game:GetService("CoreGui")

katanaSkinEnabled = false

local KatanaSkin = (function()
    local Players       = game:GetService("Players")
    local InsertService = game:GetService("InsertService")
    local Lighting2     = game:GetService("Lighting")
    local ReplicatedStorage2 = game:GetService("ReplicatedStorage")
    local LocalPlayer   = Players.LocalPlayer

    local KC = {
        red  = Color3.new(0.784314, 0, 0),
        red2 = Color3.new(1, 0.392157, 0.392157),
        gold = Color3.new(1, 0.72549, 0.196078),
    }

    local State = {
        Skin = "KATANA",
        SkinOrder = { "KATANA", "NONE" },
        LastBat = nil,
        LastAppliedSkin = nil,
        OriginalKatanaTemplate = nil,
        ExactTemplates = {},
        ExactTemplateSearched = {},
        ExactAssetIds = { KATANA = "" },
        Original = {},
    }

    local function findBatForKatanaSkin()
        local char = LocalPlayer.Character
        if char then
            local t = char:FindFirstChild("Bat")
            if t and t:IsA("Tool") then return t end
        end
        local bp = LocalPlayer:FindFirstChildOfClass("Backpack")
        if bp then
            local t = bp:FindFirstChild("Bat")
            if t and t:IsA("Tool") then return t end
        end
        return nil
    end

    local function setPartSafe(part)
        if not part or not part:IsA("BasePart") then return end
        part.Anchored    = false
        part.CanCollide  = false
        part.CanTouch    = false
        part.CanQuery    = false
        part.Massless    = true
    end

    local function weldToHandle(part, handle)
        if not part or not handle or not part:IsA("BasePart") or not handle:IsA("BasePart") then return end
        setPartSafe(part)
        local w = Instance.new("WeldConstraint")
        w.Name   = "FlowerSkin_AssetWeld"
        w.Part0  = handle
        w.Part1  = part
        w.Parent = part
    end

    local function rememberOriginal(tool)
        if not tool then return end
        local handle = tool:FindFirstChild("Handle")
        local slash  = tool:FindFirstChild("Slash")
        State.Original[tool] = State.Original[tool] or {}
        local o = State.Original[tool]
        if handle and handle:IsA("BasePart") and not o.Handle then
            o.Handle = {
                Transparency = handle.Transparency,
                LocalTransparencyModifier = handle.LocalTransparencyModifier,
                CastShadow = handle.CastShadow,
            }
        end
        if slash and slash:IsA("Sound") and not o.SlashSoundId then
            o.SlashSoundId = slash.SoundId
        end
    end

    local function captureKatanaTemplate()
        if State.OriginalKatanaTemplate then return end
        local function scan(container)
            if not container then return end
            local bat = container:FindFirstChild("Bat")
            if not bat then return end
            local folder = bat:FindFirstChild("FlowerSkin_KatanaRealistic")
            local asset  = folder and folder:FindFirstChild("FlowerSkin_AssetKatana")
            if asset then State.OriginalKatanaTemplate = asset:Clone() end
        end
        scan(LocalPlayer:FindFirstChildOfClass("Backpack"))
        scan(LocalPlayer.Character)
    end

    local function hideOriginalHandle(tool, hidden)
        local handle = tool and tool:FindFirstChild("Handle")
        if not handle or not handle:IsA("BasePart") then return end
        rememberOriginal(tool)
        if hidden then
            handle.LocalTransparencyModifier = 1
            handle.Transparency = 1
            handle.CastShadow = false
        else
            local o = State.Original[tool] and State.Original[tool].Handle
            handle.LocalTransparencyModifier = o and o.LocalTransparencyModifier or 0
            handle.Transparency = o and o.Transparency or 0
            handle.CastShadow = o and o.CastShadow
            if handle.CastShadow == nil then handle.CastShadow = true end
        end
    end

    local function setSlashSound(tool, soundId)
        rememberOriginal(tool)
        local slash = tool and tool:FindFirstChild("Slash")
        if slash and slash:IsA("Sound") then
            slash.SoundId = soundId or ((State.Original[tool] and State.Original[tool].SlashSoundId) or slash.SoundId)
        end
    end

    local function removeSkin(tool)
        if not tool then return end
        local oldFolder = tool:FindFirstChild("FlowerSkin_KatanaRealistic")
        if oldFolder then oldFolder:Destroy() end
        local oldLoose = tool:FindFirstChild("FlowerSkin_AssetKatana")
        if oldLoose then oldLoose:Destroy() end
        hideOriginalHandle(tool, false)
        setSlashSound(tool, nil)
    end

    local function addRedVFX(parentPart)
        if not parentPart or not parentPart:IsA("BasePart") then return end
        local existing = parentPart:FindFirstChild("FlowerSkin_ExtraRedVFX")
        if existing then existing:Destroy() end

        local top = Instance.new("Attachment")
        top.Name = "FlowerSkin_ExtraRedVFX"
        top.Position = Vector3.new(0, parentPart.Size.Y * 0.5, 0)
        top.Parent = parentPart

        local bottom = Instance.new("Attachment")
        bottom.Name = "FlowerSkin_ExtraRedVFX_End"
        bottom.Position = Vector3.new(0, -parentPart.Size.Y * 0.5, 0)
        bottom.Parent = parentPart

        local trail = Instance.new("Trail")
        trail.Name = "FlowerSkin_ExtraRedVFX"
        trail.Attachment0 = top
        trail.Attachment1 = bottom
        trail.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, KC.red2),
            ColorSequenceKeypoint.new(1, Color3.new(0.54902, 0, 0)),
        })
        trail.Transparency = NumberSequence.new({
            NumberSequenceKeypoint.new(0, 0.35),
            NumberSequenceKeypoint.new(1, 1),
        })
        trail.Lifetime = 0.18
        trail.LightEmission = 0.55
        trail.Parent = parentPart

        local emitter = Instance.new("ParticleEmitter")
        emitter.Name = "FlowerSkin_ExtraRedVFX"
        emitter.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, Color3.new(0.784314, 0, 0)),
            ColorSequenceKeypoint.new(1, KC.red2),
        })
        emitter.LightEmission = 0.55
        emitter.Rate = 14
        emitter.Lifetime = NumberRange.new(0.25, 0.45)
        emitter.Speed = NumberRange.new(0.2, 0.7)
        emitter.SpreadAngle = Vector2.new(12, 12)
        emitter.Size = NumberSequence.new({
            NumberSequenceKeypoint.new(0, 0.08),
            NumberSequenceKeypoint.new(1, 0),
        })
        emitter.Texture = "rbxasset://textures/particles/sparkles_main.dds"
        emitter.Parent = parentPart
    end

    local function newPart(parent, name, size, offset, color, material)
        local p = Instance.new("Part")
        p.Name = name
        p.Size = size
        p.Color = color
        p.Material = material or Enum.Material.Neon
        p.TopSurface = Enum.SurfaceType.Smooth
        p.BottomSurface = Enum.SurfaceType.Smooth
        setPartSafe(p)
        p.Parent = parent
        return p, offset or CFrame.identity
    end

    local EXACT_TEMPLATE_NAMES = {
        KATANA = { "FlowerSkin_AssetKatana", "Katana", "FlowerSkin_KatanaRealistic" },
    }

    local function isScriptObject(inst)
        return inst:IsA("Script") or inst:IsA("LocalScript") or inst:IsA("ModuleScript")
    end

    local function stripScripts(root)
        if not root then return end
        if isScriptObject(root) then root:Destroy(); return end
        for _, inst in ipairs(root:GetDescendants()) do
            if isScriptObject(inst) then inst:Destroy() end
        end
    end

    local function hasAnyBasePart(root)
        if not root then return false end
        if root:IsA("BasePart") then return true end
        return root:FindFirstChildWhichIsA("BasePart", true) ~= nil
    end

    local function isGeneratedProxy(root)
        local ok, v = pcall(function() return root:GetAttribute("CursedBatSkinsGenerated") end)
        return ok and v == true
    end

    local function usableTemplate(root)
        if not root or isGeneratedProxy(root) then return nil end
        if root:IsA("Model") or root:IsA("Tool") or root:IsA("Folder") or root:IsA("BasePart") then
            if hasAnyBasePart(root) then return root end
        end
        return nil
    end

    local function findUsableNamed(root, names)
        if not root then return nil end
        for _, name in ipairs(names) do
            if root.Name == name then
                local d = usableTemplate(root)
                if d then return d end
            end
        end
        for _, name in ipairs(names) do
            local found = root:FindFirstChild(name, true)
            local u = usableTemplate(found)
            if u then return u end
        end
        return nil
    end

    local function assetTextFromId(assetId)
        local raw = tostring(assetId or ""):gsub("%s+", "")
        if raw == "" then return nil end
        if raw:match("^rbxassetid://") or raw:match("^rbxasset://") then return raw end
        if raw:match("^%d+$") then return "rbxassetid://" .. raw end
        return raw
    end

    local function loadExactTemplateFromAssetId(skinName)
        local assetText = assetTextFromId(State.ExactAssetIds[skinName])
        if not assetText then return nil end
        local loaded = {}
        pcall(function() loaded = game:GetObjects(assetText) end)
        if #loaded == 0 then
            local id = tostring(assetText):match("(%d+)")
            if id then
                pcall(function()
                    table.insert(loaded, InsertService:LoadAsset(tonumber(id)))
                end)
            end
        end
        local names = EXACT_TEMPLATE_NAMES[skinName] or {}
        for _, root in ipairs(loaded) do
            local exact = findUsableNamed(root, names) or usableTemplate(root)
            if exact then
                local clone = exact:Clone()
                stripScripts(clone)
                return clone
            end
        end
        return nil
    end

    local function findExactTemplateInGame(skinName)
        local names = EXACT_TEMPLATE_NAMES[skinName]
        if not names then return nil end
        local containers = {
            LocalPlayer.Character,
            LocalPlayer:FindFirstChildOfClass("Backpack"),
            ReplicatedStorage2,
            Lighting2,
            workspace,
        }
        for _, container in ipairs(containers) do
            local source = findUsableNamed(container, names)
            if source then
                local clone = source:Clone()
                stripScripts(clone)
                return clone
            end
        end
        return nil
    end

    local function getExactTemplate(skinName)
        if skinName == "KATANA" and State.OriginalKatanaTemplate then
            return State.OriginalKatanaTemplate
        end
        if State.ExactTemplates[skinName] then return State.ExactTemplates[skinName] end
        if State.ExactTemplateSearched[skinName] then return nil end
        State.ExactTemplateSearched[skinName] = true
        local loaded = loadExactTemplateFromAssetId(skinName) or findExactTemplateInGame(skinName)
        if loaded then State.ExactTemplates[skinName] = loaded end
        return loaded
    end

    local function containerToModel(clone, folder)
        if clone:IsA("Model") then
            clone.Name = "FlowerSkin_AssetKatana"
            clone.Parent = folder
            return clone
        end
        if clone:IsA("BasePart") then
            local model = Instance.new("Model")
            model.Name = "FlowerSkin_AssetKatana"
            model.Parent = folder
            clone.Parent = model
            return model
        end
        if clone:IsA("Tool") or clone:IsA("Folder") then
            local nested = clone:FindFirstChild("FlowerSkin_AssetKatana")
                or clone:FindFirstChildWhichIsA("Model")
                or clone:FindFirstChildWhichIsA("BasePart")
            if nested and nested.Parent == clone then
                nested.Parent = nil
                clone:Destroy()
                return containerToModel(nested, folder)
            end
            local model = Instance.new("Model")
            model.Name = "FlowerSkin_AssetKatana"
            model.Parent = folder
            for _, child in ipairs(clone:GetChildren()) do
                if not isScriptObject(child) then child.Parent = model end
            end
            clone:Destroy()
            return model
        end
        return nil
    end

    local function applyExactMesh(tool, skinName)
        local handle = tool and tool:FindFirstChild("Handle")
        if not handle or not handle:IsA("BasePart") then return false end
        local template = getExactTemplate(skinName)
        if not template then return false end

        local folder = Instance.new("Folder")
        folder.Name = "FlowerSkin_KatanaRealistic"
        folder.Parent = tool

        local model = containerToModel(template:Clone(), folder)
        if not model or not hasAnyBasePart(model) then
            folder:Destroy()
            return false
        end
        stripScripts(model)
        pcall(function() model:SetAttribute("CursedBatSkinsExactMesh", true) end)

        local firstPart
        for _, inst in ipairs(model:GetDescendants()) do
            if inst:IsA("BasePart") then
                firstPart = firstPart or inst
                setPartSafe(inst)
            end
        end
        if not firstPart then folder:Destroy(); return false end

        if model:IsA("Model") then
            model.PrimaryPart = model.PrimaryPart or firstPart
            pcall(function() model:PivotTo(handle.CFrame) end)
        end
        for _, inst in ipairs(model:GetDescendants()) do
            if inst:IsA("BasePart") then weldToHandle(inst, handle) end
        end

        local vfxPart = model:FindFirstChild("SharpParts", true)
            or model:FindFirstChild("WeaponPart", true)
            or model:FindFirstChild("Handle", true)
            or firstPart
        if vfxPart and not vfxPart:FindFirstChild("FlowerSkin_ExtraRedVFX") then
            addRedVFX(vfxPart)
        end
        return true
    end

    local function buildProxyModel(tool, skinName)
        local handle = tool and tool:FindFirstChild("Handle")
        if not handle or not handle:IsA("BasePart") then return nil end

        local folder = Instance.new("Folder")
        folder.Name = "FlowerSkin_KatanaRealistic"
        folder.Parent = tool

        local model = Instance.new("Model")
        model.Name = "FlowerSkin_AssetKatana"
        model:SetAttribute("CursedBatSkinsGenerated", true)
        model.Parent = folder

        local parts = {}
        local function add(name, size, offset, color, material)
            local part, cfOffset = newPart(model, name, size, offset, color, material)
            part.CFrame = handle.CFrame * cfOffset
            weldToHandle(part, handle)
            table.insert(parts, part)
            return part
        end

        if skinName == "KATANA" then
            add("Handle2", Vector3.new(0.22, 1.0, 0.22), CFrame.new(0, -0.85, 0), Color3.new(0.04, 0.04, 0.045), Enum.Material.Metal)
            local sharp = add("SharpParts", Vector3.new(0.22, 3.35, 0.12), CFrame.new(0, 1.05, 0), KC.red2, Enum.Material.Neon)
            add("WeaponPart", Vector3.new(0.3, 2.7, 0.08), CFrame.new(0.08, 1.15, 0), KC.red, Enum.Material.Neon)
            add("NeonAccent", Vector3.new(0.75, 0.12, 0.42), CFrame.new(0, -0.28, 0), KC.gold, Enum.Material.Neon)
            addRedVFX(sharp)
        end
        model.PrimaryPart = parts[1]
        return model
    end

    local SKIN_SOUND_IDS = {
        KATANA = "rbxassetid://111808555599832",
    }

    local function applySkin(skinName)
        captureKatanaTemplate()
        local tool = findBatForKatanaSkin()
        if not tool then
            State.LastBat = nil
            State.LastAppliedSkin = nil
            return false
        end
        rememberOriginal(tool)
        removeSkin(tool)
        if skinName == "NONE" then
            State.LastBat = tool
            State.LastAppliedSkin = skinName
            return true
        end
        hideOriginalHandle(tool, true)
        setSlashSound(tool, SKIN_SOUND_IDS[skinName])
        if not applyExactMesh(tool, skinName) then
            buildProxyModel(tool, skinName)
        end
        local handle = tool:FindFirstChild("Handle")
        if handle then
            local fire = handle:FindFirstChildOfClass("Fire") or handle:FindFirstChild("Fire")
            if fire and fire:IsA("Fire") then
                fire.Enabled = true
                fire.Color = KC.red
                fire.SecondaryColor = KC.red2
            end
        end
        State.LastBat = tool
        State.LastAppliedSkin = skinName
        return true
    end

    return {
        State = State,
        ApplySkin = applySkin,
        SetExactAssetId = function(assetId)
            State.ExactAssetIds.KATANA = tostring(assetId or "")
            State.ExactTemplateSearched.KATANA = nil
            State.ExactTemplates.KATANA = nil
        end,
    }
end)()

_G.CursedBatKatana = KatanaSkin

task.spawn(function()
    while not _G.BloodHoundsUnloaded do
        if katanaSkinEnabled then
            local bat
            local c = LP.Character
            if c then
                local t = c:FindFirstChild("Bat")
                if t and t:IsA("Tool") then bat = t end
            end
            if not bat then
                local bp = LP:FindFirstChildOfClass("Backpack")
                if bp then
                    local t = bp:FindFirstChild("Bat")
                    if t and t:IsA("Tool") then bat = t end
                end
            end
            if bat and (bat ~= KatanaSkin.State.LastBat or KatanaSkin.State.LastAppliedSkin ~= "KATANA") then
                pcall(function() KatanaSkin.ApplySkin("KATANA") end)
            end
        end
        task.wait(0.5)
    end
end)

_G.AmbitiousNormalInfJump = _G.AmbitiousNormalInfJump or {
    holdPressed = false, holdActive = false,
    controllerActive = false, mobilePressed = false,
    mobileActive = false, hooked = {}
}

function _G._jumpEnsureProxy()
    local char = LP.Character
    if not char then return nil end
    return char:FindFirstChild("HumanoidRootPart")
end

function _G.AmbitiousApplyNormalInfJumpBoost(boost)
    if not infJumpEnabled then return end
    local char = LP.Character
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum or hum.Health <= 0 then return end
    local proxy = _G._jumpEnsureProxy()
    if not proxy then return end
    local curVel = proxy.Velocity
    proxy.Velocity = _V3new(curVel.X, boost or 50, curVel.Z)
end

function _G.AmbitiousStopNormalInfJumpHoldState()
    local S = _G.AmbitiousNormalInfJump
    S.holdPressed = false
    S.holdActive = false
    S.controllerActive = false
    S.mobilePressed = false
    S.mobileActive = false
end

UIS.JumpRequest:Connect(function()
    _G.AmbitiousApplyNormalInfJumpBoost(50)
end)

UIS.InputBegan:Connect(function(input)
    if UIS:GetFocusedTextBox() then return end
    local S = _G.AmbitiousNormalInfJump

    if input.UserInputType == Enum.UserInputType.Keyboard
       and input.KeyCode == Enum.KeyCode.Space then
        if infJumpMode == "MANUAL" then return end
        S.holdPressed = true
        task.delay(0.12, function()
            if _G.AmbitiousNormalInfJump.holdPressed and infJumpEnabled then
                _G.AmbitiousNormalInfJump.holdActive = true
                _G.AmbitiousApplyNormalInfJumpBoost(50)
            end
        end)
    elseif input.KeyCode == Enum.KeyCode.ButtonA
       and input.UserInputType.Name:match("^Gamepad") then
        if infJumpMode ~= "MANUAL" then S.controllerActive = true end
    end
end)

UIS.InputEnded:Connect(function(input)
    local S = _G.AmbitiousNormalInfJump
    if input.UserInputType == Enum.UserInputType.Keyboard
       and input.KeyCode == Enum.KeyCode.Space then
        S.holdPressed = false
        S.holdActive  = false
    end
    if input.KeyCode == Enum.KeyCode.ButtonA
       and input.UserInputType.Name:match("^Gamepad") then
        S.controllerActive = false
    end
end)

function _G.AmbitiousHookNormalInfMobileJumpButton(obj)
    local S = _G.AmbitiousNormalInfJump
    if not obj or obj.Name ~= "JumpButton"
       or not obj:IsA("GuiButton") or S.hooked[obj] then return end
    S.hooked[obj] = true
    obj.InputBegan:Connect(function(input)
        if input.UserInputType ~= Enum.UserInputType.Touch
           or not infJumpEnabled then return end
        if infJumpMode == "MANUAL" then return end
        S.mobilePressed = true
        task.delay(0.12, function()
            if S.mobilePressed and infJumpEnabled then
                S.mobileActive = true
                _G.AmbitiousApplyNormalInfJumpBoost(50)
            end
        end)
    end)
    obj.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch then
            S.mobilePressed = false
            S.mobileActive  = false
        end
    end)
end

do
    local pg = LP:FindFirstChildOfClass("PlayerGui")
    if pg then
        for _, obj in ipairs(pg:GetDescendants()) do
            _G.AmbitiousHookNormalInfMobileJumpButton(obj)
        end
        pg.DescendantAdded:Connect(function(obj)
            task.defer(_G.AmbitiousHookNormalInfMobileJumpButton, obj)
        end)
    end
end

RunService.Heartbeat:Connect(function()
    local S = _G.AmbitiousNormalInfJump
    if infJumpEnabled and infJumpMode == "HOLD"
       and (S.holdActive or S.mobileActive or S.controllerActive) then
        _G.AmbitiousApplyNormalInfJumpBoost(50)
    end
end)

function _G.setInfJumpInternal(on)
    infJumpEnabled = on and true or false
    if not infJumpEnabled then
        _G.AmbitiousStopNormalInfJumpHoldState()
        local ch = LP.Character
        local hrp = ch and ch:FindFirstChild("HumanoidRootPart")
        if hrp then pcall(function() hrp.Velocity = _V3zero end) end
    end
end

InfiniteJump = {
    start = function() _G.setInfJumpInternal(true)  end,
    stop  = function() _G.setInfJumpInternal(false) end,
    isRunning = function() return infJumpEnabled == true end,
    setJumpPower = function() end,
}

local function getCharParts()
    local char = LP.Character
    if not char then return nil, nil end
    local hum = char:FindFirstChildOfClass("Humanoid")
    local root = char:FindFirstChild("HumanoidRootPart")
    if not hum or not root then return nil, nil end
    return hum, root
end

local function claimOwnership(root)
    pcall(function() root:SetNetworkOwner(LP) end)
end

function getActiveMoveSpeed()
    if laggerCarryToggled then return LAGGER_CARRY_SPEED
    elseif laggerToggled then return LAGGER_SPEED
    elseif speedMode then return CS
    else return NS end
end

local _velChecked = {}
local _hookedVelParts = {}

local function _setupVelChecked(char)
    _velChecked = {}
    if not char then return end
    local hrp = char:WaitForChild("HumanoidRootPart", 5)
    if hrp then _velChecked[hrp] = true end
    return hrp
end

local _hookVelSupported = nil
local function _hookVelHRP(hrp)
    if not hrp or _hookedVelParts[hrp] then return end
    if _hookVelSupported == false then return end
    if _hookVelSupported == nil then
        _hookVelSupported = (type(getrawmetatable) == "function")
            and (type(setreadonly) == "function")
            and (type(newcclosure) == "function")
            and (type(checkcaller) == "function")
    end
    if not _hookVelSupported then return end
    _hookedVelParts[hrp] = true
    local ok = pcall(function()
        local mt = getrawmetatable(hrp)
        if not mt then return end
        setreadonly(mt, false)
        local originalVelIndex = rawget(mt, "__index")
        mt.__index = newcclosure(function(self, key)
            if not checkcaller() and _velChecked[self]
               and (key == "AssemblyLinearVelocity" or key == "Velocity") then
                local real
                if type(originalVelIndex) == "function" then
                    real = originalVelIndex(self, key)
                elseif type(originalVelIndex) == "table" then
                    real = originalVelIndex[key]
                end
                if real and real.Magnitude > 20 then return real.Unit * 20 end
                return real
            end
            if type(originalVelIndex) == "function" then
                return originalVelIndex(self, key)
            elseif type(originalVelIndex) == "table" then
                return originalVelIndex[key]
            end
        end)
        setreadonly(mt, true)
    end)
    if not ok then _hookVelSupported = false end
end

if LP.Character then
    local _hrp0 = _setupVelChecked(LP.Character)
    _hookVelHRP(_hrp0)
end

local function _isRagdollState(hum)
    if not hum then return true end
    local st = hum:GetState()
    return hum.PlatformStand
        or st == Enum.HumanoidStateType.Physics
        or st == Enum.HumanoidStateType.Ragdoll
        or st == Enum.HumanoidStateType.FallingDown
end

local function _applyVelocitySpeed(dir, speed, hrp)
    if not hrp or not hrp.Parent then return end
    if autoBatV2Enabled or batDesyncTpEnabled or autoBatEnabled then return end
    if dir and dir.Magnitude > 0.05 then
        pcall(function()
            if hrp.SetNetworkOwner then hrp:SetNetworkOwner(LP) end
        end)
        local unit = dir.Unit
        local vy = hrp.AssemblyLinearVelocity.Y
        hrp.AssemblyLinearVelocity = _V3new(unit.X * speed, vy, unit.Z * speed)
    else
        local vy = hrp.AssemblyLinearVelocity.Y
        hrp.AssemblyLinearVelocity = _V3new(0, vy, 0)
    end
end

function getAutoPathSpeed()
    if laggerCarryToggled or laggerToggled then return LAGGER_SPEED end
    return NS
end

ANIM_PACKS = {
    ["Zombie"] = { idle1="rbxassetid://616158929", idle2="rbxassetid://616160636", walk="rbxassetid://616168032", run="rbxassetid://616163682", jump="rbxassetid://616161997", fall="rbxassetid://616157476", climb="rbxassetid://616156119", swim="rbxassetid://616165109", swimidle="rbxassetid://616166655" },
    ["Ninja"] = { idle1="rbxassetid://656117400", idle2="rbxassetid://656117400", walk="rbxassetid://656121766", run="rbxassetid://656118852", jump="rbxassetid://656117878", fall="rbxassetid://656115606", climb="rbxassetid://656114359", swim="rbxassetid://656117400", swimidle="rbxassetid://656117400" },
    ["Vampire"] = { idle1="rbxassetid://1083445855", idle2="rbxassetid://1083445855", walk="rbxassetid://1083473930", run="rbxassetid://1083462077", jump="rbxassetid://1083455352", fall="rbxassetid://1083443587", climb="rbxassetid://1083439238", swim="rbxassetid://1083445855", swimidle="rbxassetid://1083445855" }
}

ANIM_PACK_ORDER = {{"Off", "Off"}, {"Zombie", "Zombie"}, {"Ninja", "Ninja"}, {"Vampire", "Vampire"}}

local function isPackAnim(id)
    for _, pack in pairs(ANIM_PACKS) do
        for _, v in pairs(pack) do
            if v == id then return true end
        end
    end
    return false
end

local function saveOriginalAnims(char)
    local animate = char:FindFirstChild("Animate")
    if not animate then return end
    local function g(obj) return obj and obj.AnimationId or nil end
    local ids = {
        idle1 = g(animate.idle and animate.idle.Animation1),
        idle2 = g(animate.idle and animate.idle.Animation2),
        walk  = g(animate.walk and animate.walk.WalkAnim),
        run   = g(animate.run  and animate.run.RunAnim),
        jump  = g(animate.jump and animate.jump.JumpAnim),
        fall  = g(animate.fall and animate.fall.FallAnim),
        climb = g(animate.climb and animate.climb.ClimbAnim),
        swim  = g(animate.swim and animate.swim.Swim),
        swimidle = g(animate.swimidle and animate.swimidle.SwimIdle),
    }
    if not isPackAnim(ids.walk) then originalTryardAnims = ids end
end

local function applyAnimPack(packName)
    currentAnimPack = packName
    if animSelectorLabel then animSelectorLabel.Text = packName end
    if packName == "Off" then
        if originalTryardAnims and LP.Character then
            local animate = LP.Character:FindFirstChild("Animate")
            if animate then
                local function s(obj,id) if obj then obj.AnimationId = id end end
                s(animate.idle and animate.idle.Animation1, originalTryardAnims.idle1)
                s(animate.idle and animate.idle.Animation2, originalTryardAnims.idle2)
                s(animate.walk and animate.walk.WalkAnim, originalTryardAnims.walk)
                s(animate.run  and animate.run.RunAnim,   originalTryardAnims.run)
                s(animate.jump and animate.jump.JumpAnim, originalTryardAnims.jump)
                s(animate.fall and animate.fall.FallAnim, originalTryardAnims.fall)
                s(animate.climb and animate.climb.ClimbAnim, originalTryardAnims.climb)
                s(animate.swim and animate.swim.Swim, originalTryardAnims.swim)
                s(animate.swimidle and animate.swimidle.SwimIdle, originalTryardAnims.swimidle)
            end
        end
        if tryardHeartbeatConn then tryardHeartbeatConn:Disconnect(); tryardHeartbeatConn = nil end
        return
    end
    local pack = ANIM_PACKS[packName]
    if not pack then return end
    if tryardHeartbeatConn then tryardHeartbeatConn:Disconnect() end
    tryardHeartbeatConn = RunService.Heartbeat:Connect(function()
        local c = LP.Character
        if not c then return end
        local animate = c:FindFirstChild("Animate")
        if not animate then return end
        local function s(obj,id) if obj then obj.AnimationId = id end end
        s(animate.idle and animate.idle.Animation1, pack.idle1)
        s(animate.idle and animate.idle.Animation2, pack.idle2)
        s(animate.walk and animate.walk.WalkAnim, pack.walk)
        s(animate.run  and animate.run.RunAnim,   pack.run)
        s(animate.jump and animate.jump.JumpAnim, pack.jump)
        s(animate.fall and animate.fall.FallAnim, pack.fall)
        s(animate.climb and animate.climb.ClimbAnim, pack.climb)
        s(animate.swim and animate.swim.Swim, pack.swim)
        s(animate.swimidle and animate.swimidle.SwimIdle, pack.swimidle)
    end)
end

local function startAnimPack(packName)
    local char = LP.Character
    if char then
        saveOriginalAnims(char)
        applyAnimPack(packName)
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then
            for _, track in ipairs(hum:GetPlayingAnimationTracks()) do track:Stop(0) end
            hum:ChangeState(Enum.HumanoidStateType.Running)
        end
    else
        applyAnimPack(packName)
    end
    currentAnimPack = packName
end

local function stopAnimPack()
    currentAnimPack = "Off"
    if animSelectorLabel then animSelectorLabel.Text = "Off" end
    applyAnimPack("Off")
end

-- ═══════════════════════════════════════════════════════════════
-- KEYBINDS: TPBat en X, DropBrainrot en J (liberamos X)
-- ═══════════════════════════════════════════════════════════════
DEFAULT_KB = {
    DropBrainrot = {kb = Enum.KeyCode.J, gp = nil},
    AutoBat      = {kb = Enum.KeyCode.C, gp = nil},
    TPFloor      = {kb = Enum.KeyCode.X, gp = nil},
    GuiHide      = {kb = Enum.KeyCode.RightShift, gp = nil},
    CarryToggle  = {kb = Enum.KeyCode.Q, gp = nil},
    LaggerMode   = {kb = Enum.KeyCode.V, gp = nil},
    BatV2        = {kb = Enum.KeyCode.B, gp = nil},
    InstaReset   = {kb = Enum.KeyCode.H, gp = nil},
}

KB = {
    DropBrainrot = {kb = DEFAULT_KB.DropBrainrot.kb, gp = DEFAULT_KB.DropBrainrot.gp},
    AutoBat      = {kb = DEFAULT_KB.AutoBat.kb, gp = DEFAULT_KB.AutoBat.gp},
    TPFloor      = {kb = DEFAULT_KB.TPFloor.kb, gp = DEFAULT_KB.TPFloor.gp},
    GuiHide      = {kb = DEFAULT_KB.GuiHide.kb, gp = DEFAULT_KB.GuiHide.gp},
    CarryToggle  = {kb = DEFAULT_KB.CarryToggle.kb, gp = DEFAULT_KB.CarryToggle.gp},
    LaggerMode   = {kb = DEFAULT_KB.LaggerMode.kb, gp = DEFAULT_KB.LaggerMode.gp},
    BatV2        = {kb = DEFAULT_KB.BatV2.kb, gp = DEFAULT_KB.BatV2.gp},
    InstaReset   = {kb = DEFAULT_KB.InstaReset.kb, gp = DEFAULT_KB.InstaReset.gp},
}

_isResetting = false
_lastSavedJSON = nil
_isLoading = false

CONFIG = {
    AUTO_STEAL_ENABLED = false,
    STEAL_RANGE = 61,
}

local plots = workspace:WaitForChild("Plots")
local stealConnection = nil

local Steal = {
    AutoStealEnabled = false,
    StealRadius = CONFIG.STEAL_RANGE,
    StealDuration = 1.3,
    StealDelay = 0.25,
    Data = {}
}

local isStealing = false
local autoGrabSetDelayRadius = 9
local autoGrabStopTime = 0.96
local autoGrabStopEnabled = true

autoStealVariant = 1

AUTO_STEAL_VARIANT_NAMES = { "Normal", "Semi", "Semi Normal" }

local function autoStealVariantName(n)
    n = _clamp(tonumber(n) or 1, 1, #AUTO_STEAL_VARIANT_NAMES)
    return AUTO_STEAL_VARIANT_NAMES[n]
end

local function autoStealVariantFromName(name)
    for i, v in ipairs(AUTO_STEAL_VARIANT_NAMES) do
        if v == tostring(name) then return i end
    end
    return 1
end

local function getAutoGrabStopTime()
    local dur = (Steal and Steal.StealDuration) or 1.3
    if autoStealVariant == 2 then return dur * 0.80 end
    if autoStealVariant == 3 then return dur * 0.90 end
    return dur * 0.73
end

local _plotsCache = nil
local _plotsCacheTime = 0
local function getPlotsRoot()
    local now = _tick()
    if _plotsCache and now - _plotsCacheTime < 2 and _plotsCache.Parent then
        return _plotsCache
    end
    _plotsCache = workspace:FindFirstChild("Plots")
    _plotsCacheTime = now
    return _plotsCache
end

local function isMyPlotByName(plotName)
    local plotsRoot = getPlotsRoot()
    if not plotsRoot then return false end
    local plot = plotsRoot:FindFirstChild(plotName)
    if not plot then return false end
    local sign = plot:FindFirstChild("PlotSign")
    if sign then
        local yb = sign:FindFirstChild("YourBase")
        if yb and yb:IsA("BillboardGui") then
            return yb.Enabled == true
        end
    end
    return false
end

local function findNearestPrompt()
    local char = LP.Character
    if not char then return nil, nil end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return nil, nil end
    local plotsRoot = getPlotsRoot()
    if not plotsRoot then return nil, nil end
    local nearestPrompt, nearestDist, nearestName = nil, _huge, nil
    local rpos = root.Position
    for _, plot in ipairs(plotsRoot:GetChildren()) do
        if isMyPlotByName(plot.Name) then continue end
        local pods = plot:FindFirstChild("AnimalPodiums")
        if not pods then continue end
        for _, pod in ipairs(pods:GetChildren()) do
            pcall(function()
                local base = pod:FindFirstChild("Base")
                local spawn = base and base:FindFirstChild("Spawn")
                if spawn then
                    local sp = spawn.Position
                    local dx = sp.X - rpos.X
                    local dy = sp.Y - rpos.Y
                    local dz = sp.Z - rpos.Z
                    local dist = _sqrt(dx*dx + dy*dy + dz*dz)
                    if dist < nearestDist and dist <= Steal.StealRadius then
                        local att = spawn:FindFirstChild("PromptAttachment")
                        if att then
                            for _, child in ipairs(att:GetChildren()) do
                                if child:IsA("ProximityPrompt") and child.ActionText and child.ActionText:find("Steal") then
                                    nearestPrompt = child
                                    nearestDist = dist
                                    nearestName = pod.Name
                                    break
                                end
                            end
                        end
                    end
                end
            end)
        end
    end
    return nearestPrompt, nearestName
end

local function executeSteal(prompt, podName)
    if isStealing then return end
    if math.random(30) == 1 then
        for p in pairs(Steal.Data) do
            if not p.Parent then Steal.Data[p] = nil end
        end
    end
    if not Steal.Data[prompt] then
        Steal.Data[prompt] = { hold = {}, trigger = {}, ready = true }
        pcall(function()
            if getconnections then
                for _, c in ipairs(getconnections(prompt.PromptButtonHoldBegan)) do
                    if c.Function then table.insert(Steal.Data[prompt].hold, c.Function) end
                end
                for _, c in ipairs(getconnections(prompt.Triggered)) do
                    if c.Function then table.insert(Steal.Data[prompt].trigger, c.Function) end
                end
            end
        end)
    end
    local data = Steal.Data[prompt]
    if not data.ready then return end
    data.ready = false
    isStealing = true
    if progressFill then progressFill.Size = UDim2.new(0, 0, 1, 0) end
    if progressPct then progressPct.Text = "0%" end
    task.spawn(function()
        for _, f in ipairs(data.hold) do task.spawn(f) end
        local startTime = _tick()
        local duration = Steal.StealDuration
        local stopTime = getAutoGrabStopTime()
        local promptFired = false
        if autoGrabStopEnabled then
            while isStealing and Steal.AutoStealEnabled do
                local elapsed = _tick() - startTime
                if elapsed >= stopTime then break end
                local progress = _clamp(elapsed / duration, 0, 1)
                if progressFill then progressFill.Size = UDim2.new(progress, 0, 1, 0) end
                if progressPct then progressPct.Text = _floor(progress * 100) .. "%" end
                if not prompt.Parent or not prompt.Parent.Parent then break end
                local char = LP.Character
                local hrp = char and char:FindFirstChild("HumanoidRootPart")
                if hrp and (hrp.Position - prompt.Parent.Parent.Position).Magnitude > Steal.StealRadius then
                    break
                end
                task.wait()
            end
            local stopProgress = _clamp(stopTime / duration, 0, 1)
            if progressFill then progressFill.Size = UDim2.new(stopProgress, 0, 1, 0) end
            if progressPct then progressPct.Text = _floor(stopProgress * 100) .. "%" end
            local phase2Timeout = math.max(2.99 - stopTime - math.max(duration - stopTime, 0), 0.05)
            local phase2Start = _tick()
            while isStealing and Steal.AutoStealEnabled do
                if _tick() - phase2Start >= phase2Timeout then
                    if progressFill then progressFill.Size = UDim2.new(0, 0, 1, 0) end
                    if progressPct then progressPct.Text = "0%" end
                    data.ready = true
                    isStealing = false
                    task.wait()
                    local newPrompt, newName = findNearestPrompt()
                    if newPrompt then executeSteal(newPrompt, newName) end
                    return
                end
                if not prompt.Parent or not prompt.Parent.Parent then
                    isStealing = false
                    if progressFill then progressFill.Size = UDim2.new(0, 0, 1, 0) end
                    if progressPct then progressPct.Text = "0%" end
                    data.ready = true
                    return
                end
                local char = LP.Character
                local hrp = char and char:FindFirstChild("HumanoidRootPart")
                if hrp then
                    local dist = (hrp.Position - prompt.Parent.Parent.Position).Magnitude
                    if dist <= autoGrabSetDelayRadius then
                        break
                    elseif dist > Steal.StealRadius then
                        isStealing = false
                        if progressFill then progressFill.Size = UDim2.new(0, 0, 1, 0) end
                        if progressPct then progressPct.Text = "0%" end
                        data.ready = true
                        return
                    end
                end
                task.wait()
            end
            if isStealing and Steal.AutoStealEnabled then
                local fillStart = _tick()
                local fillDuration = math.max(duration - stopTime, 0.05)
                while true do
                    local fp = _clamp((_tick() - fillStart) / fillDuration, 0, 1)
                    local totalProgress = stopProgress + fp * (1 - stopProgress)
                    if progressFill then progressFill.Size = UDim2.new(totalProgress, 0, 1, 0) end
                    if progressPct then progressPct.Text = _floor(totalProgress * 100) .. "%" end
                    if fp >= 1 and not promptFired then
                        promptFired = true
                        pcall(function()
                            for _, f in ipairs(data.trigger) do task.spawn(f) end
                            local remote = ReplicatedStorage:FindFirstChild("StealAnimal")
                            if remote and podName then remote:FireServer(podName) end
                            if prompt then prompt:Fire() end
                        end)
                        break
                    end
                    task.wait()
                end
            end
        else
            while isStealing and Steal.AutoStealEnabled do
                local elapsed = _tick() - startTime
                local progress = _clamp(elapsed / duration, 0, 1)
                if progressFill then progressFill.Size = UDim2.new(progress, 0, 1, 0) end
                if progressPct then progressPct.Text = _floor(progress * 100) .. "%" end
                if not prompt.Parent or not prompt.Parent.Parent then break end
                local char = LP.Character
                local hrp = char and char:FindFirstChild("HumanoidRootPart")
                if hrp and (hrp.Position - prompt.Parent.Parent.Position).Magnitude > Steal.StealRadius then break end
                if elapsed >= duration and not promptFired then
                    promptFired = true
                    pcall(function()
                        for _, f in ipairs(data.trigger) do task.spawn(f) end
                        local remote = ReplicatedStorage:FindFirstChild("StealAnimal")
                        if remote and podName then remote:FireServer(podName) end
                        if prompt then prompt:Fire() end
                    end)
                    break
                end
                task.wait()
            end
        end
        if progressFill then progressFill.Size = UDim2.new(0, 0, 1, 0) end
        if progressPct then progressPct.Text = "0%" end
        data.ready = true
        isStealing = false
    end)
end

function startAutoSteal()
    if stealConnection then
        local connected = false
        pcall(function() connected = stealConnection.Connected == true end)
        if connected then
            Steal.StealRadius = CONFIG.STEAL_RANGE
            Steal.AutoStealEnabled = true
            CONFIG.AUTO_STEAL_ENABLED = true
            return true
        end
        pcall(function() stealConnection:Disconnect() end)
        stealConnection = nil
    end
    Steal.StealRadius = CONFIG.STEAL_RANGE
    Steal.AutoStealEnabled = true
    CONFIG.AUTO_STEAL_ENABLED = true
    stealConnection = RunService.Heartbeat:Connect(function()
        if not Steal.AutoStealEnabled or isStealing then return end
        local p, n = findNearestPrompt()
        if p then executeSteal(p, n) end
    end)
    return true
end

function stopAutoSteal()
    if stealConnection then
        stealConnection:Disconnect()
        stealConnection = nil
    end
    isStealing = false
    Steal.AutoStealEnabled = false
    CONFIG.AUTO_STEAL_ENABLED = false
    if progressFill then
        TS:Create(progressFill, TweenInfo.new(0.2), { Size = UDim2.new(0, 0, 1, 0) }):Play()
    end
    if progressPct then progressPct.Text = "0%" end
end

dropActive = false
lastDropTime = 0
lastMoveDir = _V3new(0,0,0)
origFOV = nil
fovEnabled = false
fovValue = 70
customFovConn = nil
setFovVisual = nil
fovSliderSet = nil

_anyKeyListening = false
_aimbotConn = nil
_prevAutoRotate = nil
tpBatConn = nil
tpBatPrevAutoRotate = nil
tpBatHitCD = false
TP_BAT_SWING_CD = 0.08
tpBatFloatingButton = nil
batV2FloatingButton = nil

enemySpeedConn = nil
movementLoop = nil
steppedConn = nil
infJumpConn = nil
stretchConn = nil
stretchFovConn = nil
dropConnections = {}
enemySpeedLabels = {}
Conns = {autoSteal = nil, batCounter = nil, anchor = {}, progress = nil}
keyButtonRefs = {}
progressFill = nil
progressPct = nil
progressRadLbl = nil
pbFrame = nil
speedLabel = nil
modeValLbl = nil
normalBox, carryBox, laggerBox, lagger2Box, radInput, batSpeedBox, uiScaleBox = nil, nil, nil, nil, nil, nil, nil
modeSelectBtn, dropModeBtnRef = nil, nil
setJumpToggleState = nil
setAntiRagVisual, setJumpVisual, setAntiLagVisual, setLockUIVisual, setInstaGrab = nil, nil, nil, nil, nil
setAntiDieVisual = nil
setEditModeVisual = nil
setESPVIsual = nil
autoBatV2SetVisual = nil
miniBtn, main, gui = nil, nil, nil
MobilePanel = nil
instaResetFloatingButton = nil
showGui = nil
hideGui = nil
mainUIScale = nil
animSelectorLabel = nil
pbScale = nil
tabButtons = nil

carrySystemToggleSetter = nil
autoStealVariantLabel = nil
carrySysNormalBox = nil
carrySysCarryBox = nil
carrySysLaggerBox = nil
carrySysLaggerCarryBox = nil
carrySysSoftStealSpeedBox = nil
carrySysSoftStealRadiusBox = nil

setSafeModeVisual           = nil
infJumpSetVisual            = nil
infJumpModeSetVisual        = nil

-- ═══════════════════════════════════════════════════════════════
-- NUEVOS MÓDULOS: NO PLAYER COLLISION + NO CAMERA COLLISION
-- ═══════════════════════════════════════════════════════════════

GAMEPAD_KEYS = {
    [Enum.KeyCode.ButtonA] = true, [Enum.KeyCode.ButtonB] = true,
    [Enum.KeyCode.ButtonX] = true, [Enum.KeyCode.ButtonY] = true,
    [Enum.KeyCode.ButtonL1] = true, [Enum.KeyCode.ButtonR1] = true,
    [Enum.KeyCode.ButtonL2] = true, [Enum.KeyCode.ButtonR2] = true,
    [Enum.KeyCode.ButtonL3] = true, [Enum.KeyCode.ButtonR3] = true,
    [Enum.KeyCode.ButtonStart] = true, [Enum.KeyCode.ButtonSelect] = true,
    [Enum.KeyCode.DPadUp] = true, [Enum.KeyCode.DPadDown] = true,
    [Enum.KeyCode.DPadLeft] = true, [Enum.KeyCode.DPadRight] = true,
}

MOVE_KEYS = {
    [Enum.KeyCode.W] = true, [Enum.KeyCode.A] = true,
    [Enum.KeyCode.S] = true, [Enum.KeyCode.D] = true,
    [Enum.KeyCode.Up] = true, [Enum.KeyCode.Left] = true,
    [Enum.KeyCode.Down] = true, [Enum.KeyCode.Right] = true,
}

BAT_COUNTER_SLAP_LIST = {
    "Bat", "Slap", "Iron Slap", "Gold Slap", "Diamond Slap",
    "Emerald Slap", "Ruby Slap", "Dark Matter Slap", "Flame Slap",
    "Nuclear Slap", "Galaxy Slap", "Glitched Slap"
}

AP = {
    L1 = _V3new(-476.48, -6.28, 92.73),
    L2 = _V3new(-483.12, -4.95, 94.80),
    L_FACE = _V3new(-482.25, -4.96, 92.09),
    R1 = _V3new(-476.16, -6.52, 25.62),
    R2 = _V3new(-483.06, -5.03, 25.48),
    R_FACE = _V3new(-482.06, -6.93, 35.47),
}

function isGamepadInput(inp)
    return inp and inp.UserInputType and inp.UserInputType.Name:match("^Gamepad") ~= nil
end

function isBindableInput(inp)
    if not inp or inp.KeyCode == Enum.KeyCode.Unknown then return false end
    if inp.UserInputType == Enum.UserInputType.Keyboard then return true end
    return isGamepadInput(inp) and GAMEPAD_KEYS[inp.KeyCode] == true
end

function kbMatch(entry, kc)
    return kc and (kc == entry.kb or (entry.gp and kc == entry.gp))
end

function resetProgressBar()
    if progressPct then progressPct.Text = "0%" end
    if progressFill then progressFill.Size = UDim2.new(0, 0, 1, 0) end
end

local function doTpDown()
    pcall(function()
        local char = LP.Character
        if not char then return end
        local root = char:FindFirstChild("HumanoidRootPart")
        if not root then return end
        root.CFrame = _CFnew(root.Position.X, -7, root.Position.Z) * CFrame.Angles(0, select(2, root.CFrame:ToEulerAnglesYXZ()), 0)
        root.Velocity = _V3zero
    end)
end

-- ═══════════════════════════════════════════════════════════════
-- SAFE MODE
-- ═══════════════════════════════════════════════════════════════
function _G.AmbitiousSafeModeGetCountdownLabel()
    local ok, label = pcall(function()
        return LP.PlayerGui
          and LP.PlayerGui:FindFirstChild("DuelsMachineTopFrame")
          and LP.PlayerGui.DuelsMachineTopFrame:FindFirstChild("DuelsMachineTopFrame")
          and LP.PlayerGui.DuelsMachineTopFrame.DuelsMachineTopFrame:FindFirstChild("Timer")
          and LP.PlayerGui.DuelsMachineTopFrame.DuelsMachineTopFrame.Timer:FindFirstChild("Label")
    end)
    return (ok and label) or nil
end

function _G.AmbitiousSafeModeCountdownNumber(text)
    local t = tostring(text or ""):upper():gsub("^%s+", ""):gsub("%s+$", "")
    if t == "GO" or t == "START" or t == "READY" then return true end
    local n = tonumber(t)
    return n ~= nil and n >= 0 and n <= 10
end

function _G.AmbitiousSafeModeCountdownValue()
    local label = _G.AmbitiousSafeModeGetCountdownLabel()
    if not label then return nil end
    local t = tostring(label.Text or ""):upper():gsub("^%s+", ""):gsub("%s+$", "")
    if t == "GO" or t == "START" or t == "READY" then return 0 end
    local n = tonumber(t)
    if n ~= nil and n >= 0 and n <= 10 then return n end
    return nil
end

function _G.AmbitiousSafeModeInDuelCountdown()
    local label = _G.AmbitiousSafeModeGetCountdownLabel()
    return label and _G.AmbitiousSafeModeCountdownNumber(label.Text) or false
end

_G.AmbitiousSafeModeBlockedTools = {
    bat=true, slap=true, sword=true, gun=true, pistol=true, rifle=true,
    medusa=true, hammer=true, axe=true, knife=true, katana=true, blade=true, fist=true,
}

function _G.AmbitiousSafeModeIsCarryableTool(tool)
    if not tool or not tool:IsA("Tool") then return false end
    local name = tool.Name:lower()
    for word in pairs(_G.AmbitiousSafeModeBlockedTools) do
        if name:find(word, 1, true) then return false end
    end
    return true
end

function _G.AmbitiousSafeModeHoldingBrainrot()
    local ok, val = pcall(function() return LP:GetAttribute("Stealing") end)
    if ok and val == true then return true end
    local ok2, val2 = pcall(function() return LP:GetAttribute("AntiKick") end)
    if ok2 and val2 == true then return true end
    local char = LP.Character
    if not char then return false end
    local ok3, val3 = pcall(function() return char:GetAttribute("Stealing") end)
    if ok3 and val3 == true then return true end
    for _, name in ipairs({"Carrying","IsCarrying","Grabbed","Holding","StealHold","HasGrab"}) do
        local v = char:FindFirstChild(name, true)
        if v then
            if v:IsA("BoolValue")   and v.Value then return true end
            if v:IsA("ObjectValue") and v.Value then return true end
            if v:IsA("StringValue") and v.Value ~= "" then return true end
        end
    end
    for _, child in ipairs(char:GetChildren()) do
        if child:IsA("Model") and child:FindFirstChildWhichIsA("BasePart", true) then
            local n = child.Name:lower()
            if n:find("brainrot") or n:find("animal") or n:find("carry")
               or n:find("grab") or n:find("steal") or n:find("hold") then
                return true
            end
        end
    end
    return false
end

function _G.AmbitiousSafeModeIsLocked()
    if not antiKickEnabled then return false end
    return _G.AmbitiousSafeModeInDuelCountdown()
        or _G.AmbitiousSafeModeHoldingBrainrot()
end

function _G.AmbitiousSafeModeForceStop(reason)
    local stopped = false
    if autoBatEnabled and disableAutoBat then disableAutoBat(); stopped = true end
    if autoBatV2Enabled and disableBatV2 then disableBatV2(); stopped = true end
    if batDesyncTpEnabled and stopBatDesyncTp then stopBatDesyncTp(); stopped = true end
    if stopped then
    end
end

function _G.AmbitiousSafeModeTryStart()
    if _G.AmbitiousSafeModeIsLocked and _G.AmbitiousSafeModeIsLocked() then
        _G.AmbitiousSafeModeForceStop("SAFE MODE LOCK")
        return false
    end
    return true
end

_G.AmbitiousSafeModeMonitorStarted = _G.AmbitiousSafeModeMonitorStarted or false
if not _G.AmbitiousSafeModeMonitorStarted then
    _G.AmbitiousSafeModeMonitorStarted = true
    RunService.Heartbeat:Connect(function()
        if antiKickEnabled
           and _G.AmbitiousSafeModeIsLocked
           and _G.AmbitiousSafeModeIsLocked() then
            _G.AmbitiousSafeModeForceStop("SAFE MODE LOCK")
        end
    end)
end

local AntiRagdollV1 = {}
AntiRagdollV1.__index = AntiRagdollV1

local BOOST_SPEED = 400
local AR_DEFAULT_SPEED = 16

local stateV1 = {
    active = false,
    isBoosting = false,
    cachedChar = nil,
    ragdollConnections = {},
    _lastBoostTime = 0,
}

local function getAntiRagdollRecoverySpeed()
    if laggerCarryToggled then return LAGGER_CARRY_SPEED
    elseif laggerToggled then return LAGGER_SPEED
    elseif speedMode then return CS
    else return NS end
end

local function disconnectAllV1()
    for _, conn in ipairs(stateV1.ragdollConnections) do
        pcall(function() conn:Disconnect() end)
    end
    stateV1.ragdollConnections = {}
end

local function cacheCharacterV1()
    local char = LP.Character
    if not char then return false end
    local hum = char:FindFirstChildOfClass("Humanoid")
    local root = char:FindFirstChild("HumanoidRootPart")
    if not hum or not root then return false end
    stateV1.cachedChar = { character = char, humanoid = hum, root = root }
    return true
end

local function isRagdolledV1()
    if not stateV1.cachedChar or not stateV1.cachedChar.humanoid then return false end
    local hum = stateV1.cachedChar.humanoid
    if not hum.Parent then return false end
    local st = hum:GetState()
    local ragdollStates = {
        [Enum.HumanoidStateType.Physics] = true,
        [Enum.HumanoidStateType.Ragdoll] = true,
        [Enum.HumanoidStateType.FallingDown] = true,
    }
    return ragdollStates[st] or false
end

local function forceExitRagdollV1()
    if not stateV1.cachedChar or not stateV1.cachedChar.humanoid or not stateV1.cachedChar.root then return end
    local hum = stateV1.cachedChar.humanoid
    local root = stateV1.cachedChar.root
    if not hum.Parent or not root.Parent then return end
    pcall(function()
        LP:SetAttribute("RagdollEndTime", workspace:GetServerTimeNow())
    end)
    for _, descendant in ipairs(stateV1.cachedChar.character:GetDescendants()) do
        if descendant:IsA("BallSocketConstraint") or
           (descendant:IsA("Attachment") and descendant.Name:find("RagdollAttachment")) then
            pcall(function() descendant:Destroy() end)
        end
    end
    local spd = getAntiRagdollRecoverySpeed()
    if not stateV1.isBoosting then
        stateV1.isBoosting = true
        stateV1._lastBoostTime = _tick()
    end
    pcall(function() hum.WalkSpeed = spd end)
    if hum.Health > 0 then
        pcall(function() hum:ChangeState(Enum.HumanoidStateType.Running) end)
    end
    pcall(function() root.Anchored = false end)
end

local function heartbeatLoopV1()
    while stateV1.active do
        task.wait(0.05)
        if isRagdolledV1() then
            forceExitRagdollV1()
        elseif stateV1.isBoosting then
            local elapsed = _tick() - (stateV1._lastBoostTime or 0)
            if elapsed > 0.35 or not isRagdolledV1() then
                stateV1.isBoosting = false
                if stateV1.cachedChar and stateV1.cachedChar.humanoid then
                    pcall(function()
                        stateV1.cachedChar.humanoid.WalkSpeed = getAntiRagdollRecoverySpeed()
                    end)
                end
            end
        end
    end
end

function AntiRagdollV1.start()
    if stateV1.active then return end
    AntiRagdollV1.stop()
    if not cacheCharacterV1() then
        warn("[AntiRagdollV1] No se pudo cachear el personaje")
        return
    end
    stateV1.active = true
    stateV1.isBoosting = false
    local camConn = RunService.RenderStepped:Connect(function()
        local cam = workspace.CurrentCamera
        if cam and stateV1.cachedChar and stateV1.cachedChar.humanoid then
            cam.CameraSubject = stateV1.cachedChar.humanoid
        end
    end)
    table.insert(stateV1.ragdollConnections, camConn)
    local respawnConn = LP.CharacterAdded:Connect(function()
        stateV1.isBoosting = false
        task.wait(0.5)
        cacheCharacterV1()
    end)
    table.insert(stateV1.ragdollConnections, respawnConn)
    task.spawn(heartbeatLoopV1)
end

function AntiRagdollV1.stop()
    stateV1.active = false
    if stateV1.isBoosting and stateV1.cachedChar and stateV1.cachedChar.humanoid then
        pcall(function()
            stateV1.cachedChar.humanoid.WalkSpeed = getAntiRagdollRecoverySpeed()
        end)
    end
    stateV1.isBoosting = false
    disconnectAllV1()
    stateV1.cachedChar = nil
end

function AntiRagdollV1.isRunning() return stateV1.active end

local AntiRagdollV2 = {
    Enabled = false,
    Connection = nil,
    ResetCooldown = 0,
}

local function startAntiRagdollV2()
    if AntiRagdollV2.Connection then return end
    AntiRagdollV2.Enabled = true
    AntiRagdollV2.Connection = RunService.Heartbeat:Connect(function()
        if not AntiRagdollV2.Enabled then return end
        local char = LP.Character
        if not char then return end
        local hum = char:FindFirstChildOfClass("Humanoid")
        local root = char:FindFirstChild("HumanoidRootPart")
        if not hum or not root then return end
        if hum.Health <= 0 or hum:GetState() == Enum.HumanoidStateType.Dead then return end
        local state = hum:GetState()
        local now = _tick()
        if state == Enum.HumanoidStateType.Physics or
           state == Enum.HumanoidStateType.Ragdoll or
           state == Enum.HumanoidStateType.FallingDown then
            if now - AntiRagdollV2.ResetCooldown > 0.15 then
                AntiRagdollV2.ResetCooldown = now
                pcall(function()
                    if hum:GetState() == Enum.HumanoidStateType.GettingUp then return end
                    if hum.Health <= 0 or hum:GetState() == Enum.HumanoidStateType.Dead then return end
                    hum:ChangeState(Enum.HumanoidStateType.GettingUp)
                    root.Velocity = _V3zero
                    root.RotVelocity = _V3zero
                    root.AssemblyLinearVelocity = _V3zero
                    root.AssemblyAngularVelocity = _V3zero
                    for _, obj in ipairs(char:GetDescendants()) do
                        if obj:IsA("Motor6D") then obj.Enabled = true end
                        if obj:IsA("Constraint") then obj.Enabled = true end
                    end
                    workspace.CurrentCamera.CameraSubject = hum
                    local PM = LP.PlayerScripts:FindFirstChild("PlayerModule")
                    if PM then
                        local CM = require(PM:FindFirstChild("ControlModule"))
                        if CM then CM:Enable() end
                    end
                    hum.AutoRotate = true
                    hum.PlatformStand = false
                    hum.Sit = false
                end)
            end
        end
    end)
end

local function stopAntiRagdollV2()
    AntiRagdollV2.Enabled = false
    if AntiRagdollV2.Connection then
        AntiRagdollV2.Connection:Disconnect()
        AntiRagdollV2.Connection = nil
    end
    AntiRagdollV2.ResetCooldown = 0
end

function setAntiRagdollMode(mode)
    if AntiRagdollV1.isRunning() then AntiRagdollV1.stop() end
    if AntiRagdollV2.Enabled then stopAntiRagdollV2() end
    antiRagdollMode = mode
    if mode == "v1" then AntiRagdollV1.start()
    elseif mode == "v2" then startAntiRagdollV2() end
    if _G.updateAntiRagdollUI then _G.updateAntiRagdollUI(mode) end
    saveAllSettings()
end

local AntiDieModule = {
    enabled      = false,
    heartConn    = nil,
    deathConns   = {},
    charConn     = nil,
    humanoid     = nil,
}

local function disconnectAntiDieAll()
    for _, c in ipairs(AntiDieModule.deathConns) do
        pcall(function() c:Disconnect() end)
    end
    AntiDieModule.deathConns = {}
    if AntiDieModule.heartConn then
        pcall(function() AntiDieModule.heartConn:Disconnect() end)
        AntiDieModule.heartConn = nil
    end
end

local function protectAntiDieChar(char)
    if not char then return end
    local hum = char:WaitForChild("Humanoid", 5)
    if not hum then return end

    hum.MaxHealth = math.huge
    hum.Health    = math.huge
    pcall(function()
        hum:SetStateEnabled(Enum.HumanoidStateType.Dead, false)
        hum.BreakJointsOnDeath = false
    end)
    AntiDieModule.humanoid = hum

    table.insert(AntiDieModule.deathConns, hum.StateChanged:Connect(function(_, new)
        if not AntiDieModule.enabled then return end
        if new == Enum.HumanoidStateType.Dead then
            hum.Health = math.huge
            pcall(function()
                hum:SetStateEnabled(Enum.HumanoidStateType.Dead, false)
            end)
        end
    end))

    table.insert(AntiDieModule.deathConns, hum:GetPropertyChangedSignal("Health"):Connect(function()
        if AntiDieModule.enabled and hum.Health < hum.MaxHealth then
            hum.Health = math.huge
        end
    end))

    if AntiDieModule.heartConn then
        pcall(function() AntiDieModule.heartConn:Disconnect() end)
    end
    AntiDieModule.heartConn = RunService.Heartbeat:Connect(function()
        if AntiDieModule.enabled and hum and hum.Parent and hum.Health < hum.MaxHealth then
            hum.Health = math.huge
        end
    end)
end

local function activateOnCharacter(char)
    if not AntiDieModule.enabled then return end
    protectAntiDieChar(char or LP.Character)
end

function AntiDieModule.start()
    AntiDieModule.enabled = true
    disconnectAntiDieAll()
    protectAntiDieChar(LP.Character)

    if AntiDieModule.charConn then
        pcall(function() AntiDieModule.charConn:Disconnect() end)
    end
    AntiDieModule.charConn = LP.CharacterAdded:Connect(function(char)
        if not AntiDieModule.enabled then return end
        task.wait(0.1)
        disconnectAntiDieAll()
        protectAntiDieChar(char)
    end)
end

function AntiDieModule.stop()
    AntiDieModule.enabled = false
    disconnectAntiDieAll()
    if AntiDieModule.charConn then
        pcall(function() AntiDieModule.charConn:Disconnect() end)
        AntiDieModule.charConn = nil
    end

    local char = LP.Character
    local hum  = char and char:FindFirstChildOfClass("Humanoid")
    if hum then
        pcall(function()
            hum:SetStateEnabled(Enum.HumanoidStateType.Dead, true)
            hum.MaxHealth = 100
            hum.Health    = 100
        end)
    end
    AntiDieModule.humanoid = nil
end

_G.AntiDie = AntiDieModule

local AntiFlingShieldModule = {
    enabled = false,
    loop = nil,
    velocityThreshold = 80,
}

local function stabilizeRoot(root)
    if not root or not root.Parent then return end
    if batDesyncTpEnabled then return end
    local velocity
    local ok = pcall(function() velocity = root.AssemblyLinearVelocity end)
    if not ok or typeof(velocity) ~= "Vector3" then
        local legacyOk
        legacyOk, velocity = pcall(function() return root.Velocity end)
        if not legacyOk or typeof(velocity) ~= "Vector3" then return end
    end
    if velocity.Magnitude <= AntiFlingShieldModule.velocityThreshold then return end
    local stabilized = _V3new(0, velocity.Y, 0)
    pcall(function() root.AssemblyLinearVelocity = stabilized end)
    pcall(function() root.AssemblyAngularVelocity = _V3zero end)
    pcall(function() root.Velocity = stabilized end)
    pcall(function() root.RotVelocity = _V3zero end)
end

function AntiFlingShieldModule.start()
    AntiFlingShieldModule.enabled = true
    if AntiFlingShieldModule.loop then AntiFlingShieldModule.loop:Disconnect() end
    AntiFlingShieldModule.loop = RunService.Heartbeat:Connect(function()
        if not AntiFlingShieldModule.enabled then return end
        local char = LP.Character
        stabilizeRoot(char and char:FindFirstChild("HumanoidRootPart"))
    end)
end

function AntiFlingShieldModule.stop()
    AntiFlingShieldModule.enabled = false
    if AntiFlingShieldModule.loop then
        AntiFlingShieldModule.loop:Disconnect()
        AntiFlingShieldModule.loop = nil
    end
end

_G.AntiFlingShield = AntiFlingShieldModule

do
    local _ragCountdownRunning = false
    local function _getRagBillboard()
        local char = LP.Character
        if not char then return nil, nil end
        local head = char:FindFirstChild("Head")
        if not head then return nil, nil end
        local pGui = LP.PlayerGui
        local existing = pGui:FindFirstChild("RagCountdownBillboard")
        if existing then existing:Destroy() end
        local bb = Instance.new("BillboardGui")
        bb.Name = "RagCountdownBillboard"
        bb.Size = UDim2.new(0, 84, 0, 42)
        bb.StudsOffset = _V3new(0, 7.0, 0)
        bb.AlwaysOnTop = true
        bb.Adornee = head
        bb.Parent = pGui
        local lbl = Instance.new("TextLabel")
        lbl.Size = UDim2.new(1, 0, 1, 0)
        lbl.AnchorPoint = Vector2.new(0.5, 0.5)
        lbl.Position = UDim2.new(0.5, 0, 0.5, 0)
        lbl.BackgroundTransparency = 1
        lbl.Font = Enum.Font.GothamBlack
        lbl.TextScaled = true
        lbl.TextColor3 = Color3.fromRGB(255, 255, 255)
        lbl.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
        lbl.TextStrokeTransparency = 1
        lbl.Text = ""
        lbl.Parent = bb
        local grad = Instance.new("UIGradient", lbl)
        grad.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, Color3.fromRGB(200, 200, 210)),
            ColorSequenceKeypoint.new(0.3, Color3.new(1, 1, 1)),
            ColorSequenceKeypoint.new(0.5, Color3.new(1, 1, 1)),
            ColorSequenceKeypoint.new(0.7, Color3.new(1, 1, 1)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(200, 200, 210)),
        })
        grad.Rotation = 45
        grad.Offset = Vector2.new(0,0)
        return bb, lbl
    end

    local function _ragPunch(lbl, text)
        if not (lbl and lbl.Parent) then return end
        lbl.Text = text
    end

    local function _startRagCountdown()
        if _ragCountdownRunning then return end
        _ragCountdownRunning = true
        task.spawn(function()
            local bb, lbl = _getRagBillboard()
            if not bb then _ragCountdownRunning = false; return end
            local timeLeft = 2.5
            local step = 0.1
            while timeLeft > 0 and bb.Parent do
                _ragPunch(lbl, string.format("%.1f", timeLeft))
                task.wait(step)
                timeLeft = timeLeft - step
            end
            if bb and bb.Parent then
                _ragPunch(lbl, "READY!")
                task.wait(0.5)
                if bb and bb.Parent then bb:Destroy() end
            end
            _ragCountdownRunning = false
        end)
    end

    local _wasRagdolled = false
    RunService.Heartbeat:Connect(function()
        local char = LP.Character
        if not char then _wasRagdolled = false; return end
        local hum = char:FindFirstChildOfClass("Humanoid")
        if not hum or hum.Health <= 0 then _wasRagdolled = false; return end
        local st = hum:GetState()
        local inRag = st == Enum.HumanoidStateType.Physics
                   or st == Enum.HumanoidStateType.Ragdoll
                   or st == Enum.HumanoidStateType.FallingDown
        if inRag and not _wasRagdolled then
            _wasRagdolled = true
            _startRagCountdown()
        elseif not inRag then
            _wasRagdolled = false
        end
    end)
end

local espHighlightCache = {}
local espBillboardCache = {}
local espConn = nil
local _espLastRun = 0
profileImageCache = {}

local function clearESP()
    for plr in pairs(espHighlightCache) do
        pcall(function() espHighlightCache[plr]:Destroy() end)
    end
    for plr in pairs(espBillboardCache) do
        pcall(function() espBillboardCache[plr]:Destroy() end)
    end
    espHighlightCache = {}
    espBillboardCache = {}
end

local function updateESP()
    local now = _tick()
    if now - _espLastRun < 0.05 then return end
    _espLastRun = now
    if not espEnabled then clearESP(); return end
    local myChar = LP.Character
    local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
    if not myRoot then return end
    local myPos = myRoot.Position
    local currentPlayers = _GetPlayersCached()
    local plrSet = {}
    for _, p in ipairs(currentPlayers) do plrSet[p] = true end
    for plr in pairs(espHighlightCache) do
        if not plrSet[plr] then
            pcall(function() espHighlightCache[plr]:Destroy() end)
            espHighlightCache[plr] = nil
        end
    end
    for plr in pairs(espBillboardCache) do
        if not plrSet[plr] then
            pcall(function() espBillboardCache[plr]:Destroy() end)
            espBillboardCache[plr] = nil
        end
    end
    local color = getThemeColor()
    for _, plr in ipairs(currentPlayers) do
        if plr == LP then continue end
        local char = plr.Character
        if not char then
            if espHighlightCache[plr] then
                pcall(function() espHighlightCache[plr]:Destroy() end)
                espHighlightCache[plr] = nil
            end
            if espBillboardCache[plr] then
                pcall(function() espBillboardCache[plr]:Destroy() end)
                espBillboardCache[plr] = nil
            end
            continue
        end
        local tRoot = char:FindFirstChild("HumanoidRootPart")
        local tHead = char:FindFirstChild("Head")
        local tHum = char:FindFirstChildOfClass("Humanoid")
        local alive = tRoot and tHead and tHum and tHum.Health > 0
        if alive then
            local hl = espHighlightCache[plr]
            if not hl or not hl.Parent or hl.Parent ~= char then
                if hl then pcall(function() hl:Destroy() end) end
                hl = Instance.new("Highlight")
                hl.Name = "FictionHubESP"
                hl.FillColor = color
                hl.FillTransparency = 0.72
                hl.OutlineColor = color
                hl.OutlineTransparency = 0.05
                hl.Adornee = char
                hl.Parent = char
                espHighlightCache[plr] = hl
            end
            local bb = espBillboardCache[plr]
            if not bb or not bb.Parent then
                if bb then pcall(function() bb:Destroy() end) end
                bb = Instance.new("BillboardGui")
                bb.Name = "ProfilePic"
                bb.Size = UDim2.new(0, 56, 0, 56)
                bb.StudsOffset = _V3new(0, 3.8, 0)
                bb.Adornee = tHead
                bb.AlwaysOnTop = true
                bb.Parent = tHead
                local img = Instance.new("ImageLabel", bb)
                img.Size = UDim2.new(1, -6, 1, -6)
                img.Position = UDim2.new(0, 3, 0, 3)
                img.BackgroundTransparency = 1
                img.Image = "rbxassetid://0"
                img.ScaleType = Enum.ScaleType.Fit
                local circle = Instance.new("UICorner", img)
                circle.CornerRadius = UDim.new(1, 0)
                local stroke = Instance.new("UIStroke", img)
                stroke.Color = color
                stroke.Thickness = 1.5
                espBillboardCache[plr] = bb
                task.spawn(function()
                    local userId = plr.UserId
                    local url = profileImageCache[userId]
                    if not url then
                        local success, u = pcall(function()
                            return Players:GetUserThumbnailAsync(userId, Enum.ThumbnailType.HeadShot, Enum.ThumbnailSize.Size420x420)
                        end)
                        if success and u and u ~= "" then
                            url = u
                            profileImageCache[userId] = url
                        else
                            url = "rbxassetid://0"
                        end
                    end
                    if img then img.Image = url end
                end)
            else
                if bb.Adornee ~= tHead then bb.Adornee = tHead end
                bb.Enabled = true
            end
        else
            if espHighlightCache[plr] then
                pcall(function() espHighlightCache[plr]:Destroy() end)
                espHighlightCache[plr] = nil
            end
            if espBillboardCache[plr] then
                pcall(function() espBillboardCache[plr]:Destroy() end)
                espBillboardCache[plr] = nil
            end
        end
    end
end

local function startESPLoop()
    if espConn then espConn:Disconnect() end
    espConn = RunService.RenderStepped:Connect(updateESP)
end

local function stopESPLoop()
    if espConn then espConn:Disconnect(); espConn = nil end
    clearESP()
end

function toggleESP(on)
    espEnabled = on
    if on then startESPLoop() else stopESPLoop() end
    if setESPVIsual then setESPVIsual(on) end
end

local _enemySpeedAcc = 0
function updateEnemySpeedLabels()
    _enemySpeedAcc = _enemySpeedAcc + 1
    if _enemySpeedAcc < 6 then return end
    _enemySpeedAcc = 0
    local color = getThemeColor()
    local players = _GetPlayersCached()
    for i = 1, #players do
        local player = players[i]
        if player ~= LP then
            local char = player.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            if hrp and hum and hum.Health > 0 then
                local v = hrp.AssemblyLinearVelocity
                local speed = _sqrt(v.X*v.X + v.Z*v.Z)
                local label = enemySpeedLabels[player]
                if not label then
                    local head = char:FindFirstChild("Head")
                    if head then
                        local bb = Instance.new("BillboardGui")
                        bb.Size = UDim2.new(0, 100, 0, 25)
                        bb.StudsOffset = _V3new(0, 5.5, 0)
                        bb.AlwaysOnTop = true
                        bb.Name = "EnemySpeedGui"
                        bb.Parent = head
                        local tl = Instance.new("TextLabel", bb)
                        tl.Size = UDim2.new(1, 0, 1, 0)
                        tl.BackgroundTransparency = 1
                        tl.TextColor3 = color
                        tl.Font = Enum.Font.GothamBold
                        tl.TextScaled = true
                        tl.TextStrokeTransparency = 1
                        tl.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
                        enemySpeedLabels[player] = tl
                        label = tl
                    end
                elseif label.Parent and label.Parent.Parent ~= char then
                    local head = char:FindFirstChild("Head")
                    if head then label.Parent.Parent = head end
                end
                if label then
                    label.Text = string.format("%.1f", speed)
                    if label.TextColor3 ~= color then label.TextColor3 = color end
                end
            else
                local label = enemySpeedLabels[player]
                if label and label.Parent and label.Parent.Parent then label.Parent.Parent = nil end
                enemySpeedLabels[player] = nil
            end
        end
    end
end

function startEnemySpeed()
    if enemySpeedConn then enemySpeedConn:Disconnect() end
    enemySpeedConn = RunService.Heartbeat:Connect(updateEnemySpeedLabels)
end

function stopEnemySpeed()
    if enemySpeedConn then enemySpeedConn:Disconnect(); enemySpeedConn = nil end
end

local function getClosestTargetBody()
    local char = LP.Character
    if not char then return nil end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return nil end
    local rpos = root.Position
    local closest, minDist = nil, _huge
    local plist = _GetPlayersCached()
    for i = 1, #plist do
        local plr = plist[i]
        if plr ~= LP then
            local c = plr.Character
            if c then
                local tRoot = c:FindFirstChild("HumanoidRootPart")
                if tRoot then
                    local hum = c:FindFirstChildOfClass("Humanoid")
                    if hum and hum.Health > 0 then
                        local dx = tRoot.Position.X - rpos.X
                        local dy = tRoot.Position.Y - rpos.Y
                        local dz = tRoot.Position.Z - rpos.Z
                        local d = dx*dx + dy*dy + dz*dz
                        if d < minDist then minDist = d; closest = tRoot end
                    end
                end
            end
        end
    end
    return closest
end

function _suppressBodyLock() end
function _unsuppressBodyLock(delayed) end

function setupSpeedIndicator(char)
    local head = char:WaitForChild("Head", 5)
    if not head then return end

    local oldBB = head:FindFirstChild("FictionHubSpeedIndicator")
    if oldBB then oldBB:Destroy() end
    local oldDiscord = head:FindFirstChild("DiscordText")
    if oldDiscord then oldDiscord:Destroy() end

    local bb = Instance.new("BillboardGui", head)
    bb.Name = "FictionHubSpeedIndicator"
    bb.Size = UDim2.new(0, 240, 0, 46)
    bb.StudsOffset = _V3new(0, 2.35, 0)
    bb.AlwaysOnTop = true
    bb.LightInfluence = 0

    speedLabel = Instance.new("TextLabel", bb)
    speedLabel.Name = "Speed"
    speedLabel.Size = UDim2.new(1, 0, 0, 24)
    speedLabel.Position = UDim2.new(0, 0, 0, 0)
    speedLabel.BackgroundTransparency = 1
    speedLabel.Text = "0 speed"
    speedLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    speedLabel.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
    speedLabel.TextStrokeTransparency = 0.08
    speedLabel.Font = Enum.Font.GothamBlack
    speedLabel.TextSize = 18
    speedLabel.TextXAlignment = Enum.TextXAlignment.Center
    speedLabel.ZIndex = 10
end

local unwalkSavedAnimate = nil

function startUnwalk()
    local c = LP.Character
    if not c then return end
    local hum = c:FindFirstChildOfClass("Humanoid")
    if hum then
        for _, t in ipairs(hum:GetPlayingAnimationTracks()) do pcall(function() t:Stop() end) end
    end
    local anim = c:FindFirstChild("Animate")
    if anim then
        unwalkSavedAnimate = anim:Clone()
        anim:Destroy()
    end
end

function stopUnwalk()
    local c = LP.Character
    if c then
        local existing = c:FindFirstChild("Animate")
        if not existing then
            local src = game:GetService("StarterPlayer"):FindFirstChildOfClass("StarterCharacterScripts")
            local starterAnim = src and src:FindFirstChild("Animate")
            if starterAnim then
                starterAnim:Clone().Parent = c
            elseif unwalkSavedAnimate then
                unwalkSavedAnimate:Clone().Parent = c
            end
        end
    end
    unwalkSavedAnimate = nil
end

function refreshSpeedModeLabel()
    if modeValLbl then
        if laggerCarryToggled then modeValLbl.Text = "Lagger Carry"
        elseif laggerToggled then modeValLbl.Text = "Lagger"
        elseif speedMode then modeValLbl.Text = "Carry"
        else modeValLbl.Text = "Normal" end
    end
    if laggerCarryToggled then _G.AceCurrentSpeedMode = "Lagger Carry"
    elseif laggerToggled then _G.AceCurrentSpeedMode = "Lagger"
    elseif speedMode then _G.AceCurrentSpeedMode = "Carry"
    else _G.AceCurrentSpeedMode = "Normal" end

    if setCarryModeVisual then setCarryModeVisual(speedMode) end
    if setLaggerModeVisual then setLaggerModeVisual(laggerToggled) end
    if setLaggerCarryVisual then setLaggerCarryVisual(laggerCarryToggled) end

    if CarrySystem then
        CarrySystem.speedToggled = speedMode
        if laggerCarryToggled then
            CarrySystem:setLaggerMode(2)
        elseif laggerToggled then
            CarrySystem:setLaggerMode(1)
        else
            CarrySystem:setLaggerMode(0)
        end
    end
end

function resetMovementState()
    refreshSpeedModeLabel()
    if mobSetCarry then mobSetCarry(speedMode) end
    if setLaggerModeVisual then setLaggerModeVisual(laggerToggled) end
    if setLaggerCarryVisual then setLaggerCarryVisual(laggerCarryToggled) end
    if CarrySystem then
        CarrySystem.speedToggled = speedMode
        if laggerCarryToggled then
            CarrySystem:setLaggerMode(2)
        elseif laggerToggled then
            CarrySystem:setLaggerMode(1)
        else
            CarrySystem:setLaggerMode(0)
        end
    end
end

function toggleCarryMode()
    if laggerToggled or laggerCarryToggled then
        laggerToggled = false; laggerCarryToggled = false; speedMode = true
    else speedMode = not speedMode end
    resetMovementState()
end

function toggleLaggerMode()
    if laggerCarryToggled then laggerCarryToggled = false end
    speedMode = false; laggerToggled = not laggerToggled
    resetMovementState()
end
function toggleLaggerCarryMode()
    if laggerToggled then laggerToggled = false end
    speedMode = false; laggerCarryToggled = not laggerCarryToggled
    resetMovementState()
end

function toggleLaggerCycle()
    if speedMode then
        speedMode = false
        laggerToggled = true
        laggerCarryToggled = false
    elseif laggerToggled then
        speedMode = false
        laggerToggled = false
        laggerCarryToggled = true
    else
        speedMode = true
        laggerToggled = false
        laggerCarryToggled = false
    end
    resetMovementState()
end

function getClosestTarget()
    local char = LP.Character
    if not char then return nil end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return nil end
    local rpos = root.Position
    local closest, minDist = nil, _huge
    local plist = _GetPlayersCached()
    for i = 1, #plist do
        local plr = plist[i]
        if plr ~= LP then
            local c = plr.Character
            if c then
                local tRoot = c:FindFirstChild("HumanoidRootPart")
                if tRoot then
                    local hum = c:FindFirstChildOfClass("Humanoid")
                    if hum and hum.Health > 0 then
                        local dx = tRoot.Position.X - rpos.X
                        local dy = tRoot.Position.Y - rpos.Y
                        local dz = tRoot.Position.Z - rpos.Z
                        local d = dx*dx + dy*dy + dz*dz
                        if d < minDist then minDist = d; closest = tRoot end
                    end
                end
            end
        end
    end
    return closest
end

function trySwing()
    pcall(function()
        local char = LP.Character
        if not char then return end
        local currentTool = char:FindFirstChildOfClass("Tool")
        if currentTool and not isBatTool(currentTool) then return end
        local bat = findBat()
        if bat then
            if bat.Parent ~= char then
                local hum = char:FindFirstChildOfClass("Humanoid")
                if hum then pcall(function() hum:EquipTool(bat) end) end
            end
            pcall(function() bat:Activate() end)
        end
    end)
end

function stopAimbotAdapt()
    if _aimbotConn then
        pcall(function() _aimbotConn:Disconnect() end)
        _aimbotConn = nil
    end
    local char = LP.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if hum then
        hum.AutoRotate = (_prevAutoRotate == nil) and true or _prevAutoRotate
        hum.PlatformStand = false
        pcall(function() hum:ChangeState(Enum.HumanoidStateType.Running) end)
    end
    if root then
        root.AssemblyLinearVelocity = _V3new(0, -0.1, 0)
        root.AssemblyAngularVelocity = _V3zero
    end
    _prevAutoRotate = nil
    lastMoveDir = _V3zero
    _unsuppressBodyLock(true)
end

function startAimbotAdapt()
    if _aimbotConn then return end
    _suppressBodyLock()
    local hum0 = LP.Character and LP.Character:FindFirstChildOfClass("Humanoid")
    if hum0 then
        if _prevAutoRotate == nil then _prevAutoRotate = hum0.AutoRotate end
        hum0.AutoRotate = false
    end
    _aimbotConn = RunService.RenderStepped:Connect(function()
        if not autoBatEnabled then return end
        local char = LP.Character
        if not char then return end
        local root = char:FindFirstChild("HumanoidRootPart")
        local hum = char:FindFirstChildOfClass("Humanoid")
        if not root or not hum then return end
        if not char:FindFirstChildOfClass("Tool") then
            local bat = findBat()
            if bat then pcall(function() hum:EquipTool(bat) end) end
        end
        local target = getClosestTarget()
        if not target then return end
        local targetVel = target.AssemblyLinearVelocity
        local myPos = root.Position
        local targetPos = target.Position
        local predictPos = targetPos + targetVel * 0.14
        predictPos = predictPos + target.CFrame.LookVector * 0.3
        local direction = predictPos - myPos
        local flatDir = _V3new(direction.X, 0, direction.Z)
        if flatDir.Magnitude > 0 then flatDir = flatDir.Unit else flatDir = _V3new(0,0,0) end
        local desiredHeight = targetPos.Y + 3.7
        local yVel = (desiredHeight - myPos.Y) * 19.5 + targetVel.Y * 0.8
        if hum.FloorMaterial ~= Enum.Material.Air then yVel = math.max(yVel, 13) end
        yVel = _clamp(yVel, -70, 110)
        local desiredVel = _V3new(flatDir.X * BAT_AIMBOT_SPEED, yVel, flatDir.Z * BAT_AIMBOT_SPEED)
        root.AssemblyLinearVelocity = root.AssemblyLinearVelocity:Lerp(desiredVel, 0.8)
        local speed3 = targetVel.Magnitude
        local predictTime = _clamp(speed3 / 150, 0.05, 0.2)
        local predictedPos = targetPos + targetVel * predictTime
        local toPredict = predictedPos - myPos
        if toPredict.Magnitude > 0.1 then
            local goalCF = _CFlookAt(myPos, predictedPos)
            local diffCF = root.CFrame:Inverse() * goalCF
            local rx, ry, rz = diffCF:ToEulerAnglesXYZ()
            rx = _clamp(rx, -2.5, 2.5)
            ry = _clamp(ry, -2.5, 2.5)
            rz = _clamp(rz, -2.5, 2.5)
            root.AssemblyAngularVelocity = root.CFrame:VectorToWorldSpace(_V3new(rx * 42, ry * 42, rz * 42))
        end
        local distToTarget = (root.Position - target.Position).Magnitude
        if distToTarget <= 8 then trySwing() end
    end)
end

function disableAutoBat()
    autoBatEnabled = false
    if autoBatSetVisual then autoBatSetVisual(false) end
    if mobSetAutoBat then mobSetAutoBat(false) end
    stopAimbotAdapt()
end

function enableAutoBat()
    if not _G.AmbitiousSafeModeTryStart() then
        autoBatEnabled = false
        if autoBatSetVisual then autoBatSetVisual(false) end
        return
    end
    if batDesyncTpEnabled then toggleBatDesyncTp() end
    if autoBatV2Enabled then disableBatV2() end
    autoBatEnabled = true
    if autoBatSetVisual then autoBatSetVisual(true) end
    if mobSetAutoBat then mobSetAutoBat(true) end
    startAimbotAdapt()
end

-- ═══════════════════════════════════════════════════════════════
-- BAT BYPASS — Persecución autónoma con predicción de ping
-- (Reemplaza el módulo Anti Bypass anterior. Antes BAT V2.)
-- ═══════════════════════════════════════════════════════════════

BAT_V2_SPEED           = BAT_V2_SPEED           or 55
BAT_V2_HIT_DIST        = BAT_V2_HIT_DIST        or 13
BAT_V2_LEAD_STUDS      = BAT_V2_LEAD_STUDS      or 3
BAT_V2_BODY_LOCK_RANGE = BAT_V2_BODY_LOCK_RANGE or 60

local SnowVS = _G.SnowVS or {}
_G.SnowVS = SnowVS

SnowVS.BatBypass = SnowVS.BatBypass or {}
-- Alias legado para no romper referencias externas existentes
SnowVS.AimbotBypassV2 = SnowVS.BatBypass

local BB = SnowVS.BatBypass

BB.Enabled    = BB.Enabled    or false
BB.Connection = BB.Connection or nil

BB.Z = BB.Z or {
    targetPlayer          = nil,
    lastTargetPos         = nil,
    targetVelocity        = Vector3.zero,
    smoothedVelocity      = Vector3.zero,
    velocityHistory       = {},
    accelerationHistory   = {},
    aerialVelocityHistory = {},
    previousDirection     = nil,
    lastDirectionChangeTime = 0,
    airborneTime          = 0,
    lastActivationTime    = 0,
    currentPing           = 0.1,
    realPingMs            = 0,
}

BB.ZCFG = BB.ZCFG or {
    FOLLOW_SPEED                    = BAT_V2_SPEED,
    ACTIVATE_DISTANCE               = BAT_V2_HIT_DIST,
    MIN_FOLLOW_DISTANCE             = 1,
    PREDICTION_TIME                 = 0.22,
    PREDICT_AHEAD                   = BAT_V2_LEAD_STUDS,
    MAX_VELOCITY_CHANGE             = 150,
    VELOCITY_SMOOTHING              = 0.2,
    MAX_HORIZONTAL_VELOCITY         = 80,
    SERVER_TICKRATE                 = 1 / 60,
    MIN_PING_COMPENSATION           = 0.03,
    MAX_PING_COMPENSATION           = 0.25,
    ACCELERATION_PREDICTION_WEIGHT  = 0.3,
    DIRECTION_CHANGE_DETECTION_TIME = 0.12,
    QUICK_DIRECTION_CHANGE_MULTIPLIER = 1.5,
    GRAVITY                         = 196.2,
    AIR_CONTROL_FACTOR              = 0.8,
    MIN_AIRBORNE_TIME               = 0.08,
}

local function bbAverageVector(history)
    if #history == 0 then return Vector3.zero end
    local total = Vector3.zero
    for _, v in ipairs(history) do total += v end
    return total / #history
end

local function bbPushHistory(history, value, maximum)
    table.insert(history, value)
    if #history > maximum then table.remove(history, 1) end
end

function BB.FindBat()
    local char = LP.Character
    if not char then return nil end
    local equipped = char:FindFirstChildOfClass("Tool")
    if equipped then
        local n = equipped.Name:lower()
        if n:find("bat", 1, true) or n:find("slap", 1, true) then
            return equipped
        end
    end
    local bp = LP:FindFirstChildOfClass("Backpack")
    if bp then
        for _, tool in ipairs(bp:GetChildren()) do
            if tool:IsA("Tool") then
                local n = tool.Name:lower()
                if n:find("bat", 1, true) or n:find("slap", 1, true) then
                    return tool
                end
            end
        end
    end
    return nil
end

local function bbResetTarget()
    local Z = BB.Z
    Z.targetPlayer          = nil
    Z.lastTargetPos         = nil
    Z.targetVelocity        = Vector3.zero
    Z.smoothedVelocity      = Vector3.zero
    Z.velocityHistory       = {}
    Z.accelerationHistory   = {}
    Z.aerialVelocityHistory = {}
    Z.previousDirection     = nil
    Z.airborneTime          = 0
end

local function bbNearestTarget(root)
    local nearest, nearestDistance = nil, math.huge
    for _, candidate in ipairs(Players:GetPlayers()) do
        if candidate ~= LP and candidate.Character then
            local targetRoot     = candidate.Character:FindFirstChild("HumanoidRootPart")
            local targetHumanoid = candidate.Character:FindFirstChildOfClass("Humanoid")
            if targetRoot and targetHumanoid and targetHumanoid.Health > 0 then
                local distance = (root.Position - targetRoot.Position).Magnitude
                if distance < nearestDistance then
                    nearest, nearestDistance = candidate, distance
                end
            end
        end
    end
    return nearest
end

local function bbRotateRoot(root, direction)
    if direction.Magnitude < 0.01 then return end
    local axis  = root.CFrame.LookVector:Cross(direction.Unit)
    local angle = math.asin(math.clamp(axis.Magnitude, -1, 1))
    root.AssemblyAngularVelocity = axis.Magnitude > 0.01
        and axis.Unit * angle * 80
        or  Vector3.zero
end

local function bbSamplePing()
    local ok, value = pcall(function()
        return game:GetService("Stats").Network.ServerStatsItem["Data Ping"]:GetValue()
    end)
    local Z = BB.Z
    if ok and type(value) == "number" then Z.realPingMs = math.floor(value) end
    Z.currentPing = math.clamp(
        Z.realPingMs / 1000,
        BB.ZCFG.MIN_PING_COMPENSATION,
        BB.ZCFG.MAX_PING_COMPENSATION
    )
end

BB.PingLoopStarted = BB.PingLoopStarted or false
if not BB.PingLoopStarted then
    BB.PingLoopStarted = true
    task.spawn(function()
        while not _G.BloodHoundsUnloaded do
            pcall(bbSamplePing)
            task.wait(0.5)
        end
    end)
end

-- Stubs para compatibilidad con la API antigua (Body Lock ya no se usa aparte)
function BB.StartBodyLock() end
function BB.StopBodyLock()  end

function BB.Stop()
    local Z = BB.Z
    BB.Enabled = false
    if BB.Connection then
        BB.Connection:Disconnect()
        BB.Connection = nil
    end
    local char     = LP.Character
    local humanoid = char and char:FindFirstChildOfClass("Humanoid")
    local root     = char and char:FindFirstChild("HumanoidRootPart")
    if humanoid then humanoid.AutoRotate = true end
    if root     then root.AssemblyAngularVelocity = Vector3.zero end
    bbResetTarget()
    Z.lastActivationTime = 0
end

function BB.Start()
    if BB.Connection then return end

    local char     = LP.Character
    local humanoid = char and char:FindFirstChildOfClass("Humanoid")
    local root     = char and char:FindFirstChild("HumanoidRootPart")
    if not humanoid or not root then return end

    BB.Enabled = true
    humanoid.AutoRotate = false

    local bat = BB.FindBat()
    if bat and bat.Parent ~= char then
        pcall(function() humanoid:EquipTool(bat) end)
    end

    BB.Connection = RunService.RenderStepped:Connect(function(dt)
        if not BB.Enabled then BB.Stop(); return end
        local Z     = BB.Z
        local ZCFG  = BB.ZCFG

        -- Refresca velocidad según modo actual (Normal/Lagger/Lagger Carry)
        if _G.AceGetAntiBypassAimbotSpeed then
            ZCFG.FOLLOW_SPEED = _G.AceGetAntiBypassAimbotSpeed()
        end

        local currentChar     = LP.Character
        local currentRoot     = currentChar and currentChar:FindFirstChild("HumanoidRootPart")
        local currentHumanoid = currentChar and currentChar:FindFirstChildOfClass("Humanoid")
        if not currentRoot or not currentHumanoid or currentHumanoid.Health <= 0 then return end

        currentHumanoid.AutoRotate = false
        root, humanoid = currentRoot, currentHumanoid

        bat = currentChar:FindFirstChildOfClass("Tool") or BB.FindBat()
        if bat and bat.Parent ~= currentChar then
            pcall(currentHumanoid.EquipTool, currentHumanoid, bat)
        end

        Z.targetPlayer  = bbNearestTarget(root)
        local targetChar = Z.targetPlayer and Z.targetPlayer.Character
        local targetRoot = targetChar and targetChar:FindFirstChild("HumanoidRootPart")
        local targetHumanoid = targetChar and targetChar:FindFirstChildOfClass("Humanoid")
        if not targetRoot or not targetHumanoid or targetHumanoid.Health <= 0 then
            bbResetTarget(); return
        end

        local targetPos = targetRoot.Position
        local safeDt    = math.max(dt, 1 / 240)

        -- Estima velocidad del rival filtrando outliers
        if Z.lastTargetPos then
            local rawVelocity   = (targetPos - Z.lastTargetPos) / safeDt
            local velocityDelta = rawVelocity - Z.targetVelocity
            if velocityDelta.Magnitude > ZCFG.MAX_VELOCITY_CHANGE then
                rawVelocity = Z.targetVelocity + velocityDelta.Unit * ZCFG.MAX_VELOCITY_CHANGE
            end
            local horizontal = Vector3.new(rawVelocity.X, 0, rawVelocity.Z)
            if horizontal.Magnitude > ZCFG.MAX_HORIZONTAL_VELOCITY then
                horizontal  = horizontal.Unit * ZCFG.MAX_HORIZONTAL_VELOCITY
                rawVelocity = Vector3.new(horizontal.X, rawVelocity.Y, horizontal.Z)
            end
            bbPushHistory(Z.accelerationHistory, (rawVelocity - Z.targetVelocity) / safeDt, 4)
            bbPushHistory(Z.velocityHistory,     rawVelocity, 8)
            Z.targetVelocity   = rawVelocity
            Z.smoothedVelocity = Z.smoothedVelocity:Lerp(rawVelocity, ZCFG.VELOCITY_SMOOTHING)
        end
        Z.lastTargetPos = targetPos

        -- Estado aéreo
        local airborne = targetHumanoid.FloorMaterial == Enum.Material.Air
        Z.airborneTime = airborne and (Z.airborneTime + safeDt) or 0
        if airborne and Z.airborneTime >= ZCFG.MIN_AIRBORNE_TIME then
            bbPushHistory(Z.aerialVelocityHistory, Z.targetVelocity, 6)
        elseif not airborne then
            Z.aerialVelocityHistory = {}
        end

        local predictionVelocity = Z.smoothedVelocity
        if airborne and #Z.aerialVelocityHistory > 0 then
            local aerial = bbAverageVector(Z.aerialVelocityHistory)
            predictionVelocity = Vector3.new(aerial.X, Z.targetVelocity.Y, aerial.Z) * ZCFG.AIR_CONTROL_FACTOR
        end

        -- Detecta cambios bruscos de dirección
        local quickTurn = false
        local horizontalVelocity = Vector3.new(Z.targetVelocity.X, 0, Z.targetVelocity.Z)
        if horizontalVelocity.Magnitude > 5 then
            local direction = horizontalVelocity.Unit
            if Z.previousDirection and Z.previousDirection:Dot(direction) < 0.5 then
                quickTurn = tick() - Z.lastDirectionChangeTime < ZCFG.DIRECTION_CHANGE_DETECTION_TIME
                Z.lastDirectionChangeTime = tick()
            end
            Z.previousDirection = direction
        end

        local serverDelay = Z.currentPing + ZCFG.SERVER_TICKRATE
        if quickTurn then serverDelay *= ZCFG.QUICK_DIRECTION_CHANGE_MULTIPLIER end

        local predicted    = targetPos + predictionVelocity * serverDelay
        local acceleration = bbAverageVector(Z.accelerationHistory)
        predicted += acceleration * ZCFG.ACCELERATION_PREDICTION_WEIGHT * (serverDelay * serverDelay * 0.5)

        local predictionTime = ZCFG.PREDICTION_TIME * 1.1
        if airborne then
            predicted += predictionVelocity * predictionTime
            predicted += Vector3.new(0, -0.5 * ZCFG.GRAVITY * predictionTime * predictionTime, 0)
        else
            predicted += predictionVelocity * predictionTime
        end

        local flatPrediction = Vector3.new(predictionVelocity.X, 0, predictionVelocity.Z)
        if flatPrediction.Magnitude > 1 then
            predicted += flatPrediction.Unit * ZCFG.PREDICT_AHEAD
        end

        local toTarget = predicted - root.Position
        bbRotateRoot(root, toTarget)

        if (targetPos - root.Position).Magnitude <= ZCFG.ACTIVATE_DISTANCE
        and tick() - Z.lastActivationTime >= 0.3 then
            if bat then pcall(bat.Activate, bat) end
            Z.lastActivationTime = tick()
        end

        if toTarget.Magnitude > ZCFG.MIN_FOLLOW_DISTANCE then
            root.AssemblyLinearVelocity = toTarget.Unit * ZCFG.FOLLOW_SPEED
        else
            root.AssemblyLinearVelocity = Vector3.new(0, root.AssemblyLinearVelocity.Y * 0.5, 0)
        end
    end)
end

function BB.Toggle()
    if BB.Enabled then BB.Stop() else BB.Start() end
end

-- ═══════════════════════════════════════════════════════════════
-- Compat wrappers (API externa usada por UI / config / visuales)
-- ═══════════════════════════════════════════════════════════════

local function _aceSafeSave()
    if type(saveAllSettings) == "function" then pcall(saveAllSettings) end
end

local function _aceCurrentSpeedMode()
    if laggerCarryToggled then return "Lagger Carry" end
    if laggerToggled      then return "Lagger" end
    if speedMode          then return "Carry"  end
    return "Normal"
end

_G.AceGetAntiBypassAimbotSpeed = function()
    local mode = _aceCurrentSpeedMode()
    if mode == "Lagger" or mode == "Lagger Carry" then
        return tonumber(_G.AceAntiBypassLaggerAimbotSpeed) or 40
    end
    return tonumber(_G.AceAntiBypassAimbotSpeed) or 60
end

_G.AceRefreshAimbotVisual = function()
    if _G.AceAimbotSetVisual then
        _G.AceAimbotSetVisual(_G.AceAntiBypassAimbotOn == true)
    end
end

_G.AceStartAntiBypassAimbot = function()
    if _G.AceSafeModeTryStart and not _G.AceSafeModeTryStart() then
        return false
    end
    if _G.AceStopAutoTPForAction then _G.AceStopAutoTPForAction() end
    if _G.AceStopNormalAimbot    then _G.AceStopNormalAimbot()    end

    selectedAimbotMode = "Bat Bypass"
    _G.AceAntiBypassAimbotOn = true
    SnowVS.BatBypass.ZCFG.FOLLOW_SPEED = _G.AceGetAntiBypassAimbotSpeed()
    SnowVS.BatBypass.Start()

    if _G.AceRefreshAimbotVisual then _G.AceRefreshAimbotVisual() end
    _aceSafeSave()
    return true
end

_G.AceStopAntiBypassAimbot = function(keepVisual)
    SnowVS.BatBypass.Stop()
    _G.AceAntiBypassAimbotOn = false
    if keepVisual ~= false and _G.AceRefreshAimbotVisual then
        _G.AceRefreshAimbotVisual()
    end
    _aceSafeSave()
end

_G.AceToggleSelectedAimbot = function()
    if selectedAimbotMode == "Bat Bypass" then
        if _G.AceAntiBypassAimbotOn then
            _G.AceStopAntiBypassAimbot()
        else
            _G.AceStartAntiBypassAimbot()
        end
    end
    if _G.AceRefreshAimbotVisual then _G.AceRefreshAimbotVisual() end
    _aceSafeSave()
end

_G.AceAntiBypassStart = _G.AceStartAntiBypassAimbot
_G.AceAntiBypassStop  = _G.AceStopAntiBypassAimbot

function enableBatV2()
    selectedAimbotMode = "Bat Bypass"
    local ok = _G.AceStartAntiBypassAimbot()
    if ok ~= false then
        autoBatV2Enabled = true
        if autoBatV2SetVisual then autoBatV2SetVisual(true) end
        if batV2FloatingButton then
            local btnFrame = batV2FloatingButton:FindFirstChild("Frame")
            if btnFrame then paintFloatingBtn(btnFrame, true) end
        end
    end
end

function disableBatV2()
    _G.AceStopAntiBypassAimbot()
    autoBatV2Enabled = false
    if autoBatV2SetVisual then autoBatV2SetVisual(false) end
    if batV2FloatingButton then
        local btnFrame = batV2FloatingButton:FindFirstChild("Frame")
        if btnFrame then paintFloatingBtn(btnFrame, false) end
    end
end

function toggleBatV2()
    if autoBatV2Enabled then disableBatV2() else enableBatV2() end
end

_G.AceAntiBypassSaveToConfig = function(t)
    t = t or {}
    t.selectedAimbotMode              = selectedAimbotMode
    t.ANTI_BYPASS_AIMBOT_SPEED        = _G.AceAntiBypassAimbotSpeed
    t.ANTI_BYPASS_LAGGER_AIMBOT_SPEED = _G.AceAntiBypassLaggerAimbotSpeed
    t.antiBypassAimbotEnabled         = _G.AceAntiBypassAimbotOn == true
    return t
end

_G.AceAntiBypassLoadFromConfig = function(data)
    if type(data) ~= "table" then return end
    selectedAimbotMode = data.selectedAimbotMode or selectedAimbotMode
    -- Migración de configs antiguas "Anti Bypass" → "Bat Bypass"
    if selectedAimbotMode == "Anti Bypass" then
        selectedAimbotMode = "Bat Bypass"
    end
    if selectedAimbotMode ~= "Bat Bypass" then
        selectedAimbotMode = "Normal"
    end
    _G.AceAntiBypassAimbotSpeed =
        tonumber(data.ANTI_BYPASS_AIMBOT_SPEED) or _G.AceAntiBypassAimbotSpeed or 60

    if data.ANTI_BYPASS_LAGGER_AIMBOT_SPEED == nil
       or tonumber(data.ANTI_BYPASS_LAGGER_AIMBOT_SPEED) == 58 then
        _G.AceAntiBypassLaggerAimbotSpeed = 40
    else
        _G.AceAntiBypassLaggerAimbotSpeed =
            tonumber(data.ANTI_BYPASS_LAGGER_AIMBOT_SPEED) or 40
    end

    _G.AceAntiBypassAimbotOn = data.antiBypassAimbotEnabled == true
end

-- ═══════════════════════════════════════════════════════════════
-- TP BAT — Network Owner Teleport + Anti-Die + Lagger
-- ═══════════════════════════════════════════════════════════════
batDesyncTpEnabled = false
batDesyncTpConn    = nil
batDesyncTpSetVisual = nil
local _tpBatUnwalkForced = false

-- Compat externa (UI, safe mode, teclas, carga de config, etc.)
_G.AlvaroTP = _G.AlvaroTP or {
    conn           = nil,
    on             = false,
    h              = nil,
    hrp            = nil,
    hittingCooldown= false,
    _charConn      = nil,
}

local _tpBatHitCooldown = false
local _tpBatMaxRange    = 100

-- ──────────────────────────────────────────────────────────────
-- ANTI-DIE + LAGGER (integrados al TP BAT)
-- ──────────────────────────────────────────────────────────────
local _tpBatAntiDieThread = nil
local _tpBatLagThread     = nil
local _tpBatLagChain      = nil
local _tpBatLagPayload    = nil

local TP_LAG_KBPS     = 12000
local TP_LAG_INTERVAL = 0.15
local TP_LAG_DEPTH    = 12
local TP_LAG_REPS     = 800

local function tpBatStartAntiDie()
    if _tpBatAntiDieThread then return end
    _tpBatAntiDieThread = task.spawn(function()
        while batDesyncTpEnabled do
            pcall(function()
                local char = LP.Character
                if not char then task.wait(0.1) return end
                local hum  = char:FindFirstChildOfClass("Humanoid")
                local root = char:FindFirstChild("HumanoidRootPart")
                if hum then
                    if hum.Health <= 0 then
                        hum.Health = hum.MaxHealth
                    end
                    if hum.PlatformStand then hum.PlatformStand = false end
                    if hum.Sit then hum.Sit = false end
                    local st = hum:GetState()
                    if st == Enum.HumanoidStateType.Physics
                       or st == Enum.HumanoidStateType.Ragdoll
                       or st == Enum.HumanoidStateType.FallingDown then
                        hum:ChangeState(Enum.HumanoidStateType.GettingUp)
                        hum:ChangeState(Enum.HumanoidStateType.Running)
                    end
                end
                if root and sethiddenproperty then
                    pcall(sethiddenproperty, root, "PhysicsRepRootPart", nil)
                end
                if char then
                    for _, p in ipairs(char:GetDescendants()) do
                        if p:IsA("BasePart") then p.CanCollide = false end
                    end
                end
            end)
            task.wait(0.05)
        end
        _tpBatAntiDieThread = nil
    end)
end

local function tpBatStopAntiDie()
    if _tpBatAntiDieThread then
        pcall(task.cancel, _tpBatAntiDieThread)
        _tpBatAntiDieThread = nil
    end
end

local function tpBatBuildLagChain()
    if _tpBatLagChain then return _tpBatLagChain end
    local root = {}
    local cur = root
    for _ = 1, TP_LAG_DEPTH do
        local n = {}
        cur[1] = n
        cur = n
    end
    _tpBatLagChain = root
    return root
end

local function tpBatBuildLagPayload()
    if _tpBatLagPayload then return _tpBatLagPayload end
    local chain = tpBatBuildLagChain()
    local payload = table.create(TP_LAG_REPS)
    for i = 1, TP_LAG_REPS do payload[i] = chain end
    _tpBatLagPayload = payload
    return payload
end

local function tpBatSetKBPS(v)
    pcall(function()
        game:GetService("NetworkClient"):SetOutgoingKBPSLimit(v)
    end)
end

local function tpBatFireLagPayload()
    pcall(function()
        local rrs = game:GetService("RobloxReplicatedStorage")
        if rrs and rrs:FindFirstChild("SetPlayerBlockList") then
            rrs.SetPlayerBlockList:FireServer(tpBatBuildLagPayload())
        end
    end)
end

local function tpBatStartLagger()
    if _tpBatLagThread then return end
    _tpBatLagThread = task.spawn(function()
        task.wait(0.05)
        while batDesyncTpEnabled do
            tpBatSetKBPS(TP_LAG_KBPS)
            tpBatFireLagPayload()
            task.wait(TP_LAG_INTERVAL)
        end
        tpBatSetKBPS(0)
        _tpBatLagThread = nil
    end)
end

local function tpBatStopLagger()
    if _tpBatLagThread then
        pcall(task.cancel, _tpBatLagThread)
        _tpBatLagThread = nil
    end
    tpBatSetKBPS(0)
end

-- ══════════════════════════════════════════════════════
-- BAT
-- ══════════════════════════════════════════════════════
local function tpBatGetBat()
    local character = LP.Character
    if not character then return nil end
    for _, tool in ipairs(character:GetChildren()) do
        if tool:IsA("Tool") then
            local name = tool.Name:lower()
            if name:find("bat") or name:find("slap") then return tool end
        end
    end
    local backpack = LP:FindFirstChild("Backpack")
    if not backpack then return nil end
    for _, tool in ipairs(backpack:GetChildren()) do
        if tool:IsA("Tool") then
            local name = tool.Name:lower()
            if name:find("bat") or name:find("slap") then
                local humanoid = character:FindFirstChildOfClass("Humanoid")
                if humanoid then pcall(function() humanoid:EquipTool(tool) end) end
                return tool
            end
        end
    end
    return nil
end

local function tpBatHit()
    if _tpBatHitCooldown then return end
    _tpBatHitCooldown = true
    pcall(function()
        local bat = tpBatGetBat()
        if not bat then return end
        bat:Activate()
        local remote = bat:FindFirstChildWhichIsA("RemoteEvent")
        if remote then remote:FireServer() end
    end)
    task.delay(0.08, function() _tpBatHitCooldown = false end)
end

-- ══════════════════════════════════════════════════════
-- TARGETING
-- ══════════════════════════════════════════════════════
local function tpBatClosestPlayer(root)
    local closest, distance = nil, math.huge
    for _, player in ipairs(Players:GetPlayers()) do
        local character  = player ~= LP and player.Character
        local targetRoot = character and character:FindFirstChild("HumanoidRootPart")
        local humanoid   = character and character:FindFirstChildOfClass("Humanoid")
        if targetRoot and humanoid and humanoid.Health > 0 then
            local candidateDistance = (root.Position - targetRoot.Position).Magnitude
            if candidateDistance < distance then
                closest, distance = player, candidateDistance
            end
        end
    end
    return closest, distance
end

-- ══════════════════════════════════════════════════════
-- NETWORK OWNER TELEPORT
-- ══════════════════════════════════════════════════════
local function tpBatNetworkOwnerTeleport(root, targetRoot)
    pcall(function() if root.SetNetworkOwner then root:SetNetworkOwner(nil) end end)
    task.wait()
    local targetPosition = targetRoot.Position + _V3new(0, 0.9, 0)
    root.CFrame = _CFnew(targetPosition)
    root.AssemblyLinearVelocity = targetRoot.AssemblyLinearVelocity
    pcall(function() if root.SetNetworkOwner then root:SetNetworkOwner(LP) end end)
end

-- ══════════════════════════════════════════════════════
-- UPDATE LOOP
-- ══════════════════════════════════════════════════════
local function tpBatUpdate()
    if not batDesyncTpEnabled then return end
    local character = LP.Character
    local root      = character and character:FindFirstChild("HumanoidRootPart")
    local humanoid  = character and character:FindFirstChildOfClass("Humanoid")
    if not root or not humanoid then return end

    local animator = humanoid:FindFirstChildOfClass("Animator")
    if animator then
        pcall(function()
            for _, track in ipairs(animator:GetPlayingAnimationTracks()) do track:Stop() end
        end)
    end

    local bat = tpBatGetBat()
    if bat and bat.Parent ~= character then
        pcall(function() humanoid:EquipTool(bat) end)
    end

    local target, distance = tpBatClosestPlayer(root)
    if not target or distance > _tpBatMaxRange then return end
    local targetRoot = target.Character and target.Character:FindFirstChild("HumanoidRootPart")
    if not targetRoot then return end

    tpBatNetworkOwnerTeleport(root, targetRoot)

    local camera = workspace.CurrentCamera
    if camera then
        camera.CFrame = _CFnew(camera.CFrame.Position, targetRoot.Position)
    end
    tpBatHit()

    pcall(function()
        for _, part in ipairs(character:GetDescendants()) do
            if part:IsA("BasePart") then part.CanCollide = false end
        end
    end)
end

-- ══════════════════════════════════════════════════════
-- START / STOP / TOGGLE
-- ══════════════════════════════════════════════════════
function startAlvaroTP()
    if batDesyncTpConn then
        batDesyncTpConn:Disconnect()
        batDesyncTpConn = nil
    end

    batDesyncTpEnabled = true
    _G.AlvaroTP.on     = true
    _G.AlvaroTP.conn   = nil
    _tpBatHitCooldown  = false

    local char = LP.Character
    if char then
        _G.AlvaroTP.h   = char:FindFirstChildOfClass("Humanoid")
        _G.AlvaroTP.hrp = char:FindFirstChild("HumanoidRootPart")
    end

    batDesyncTpConn  = RunService.Heartbeat:Connect(tpBatUpdate)
    _G.AlvaroTP.conn = batDesyncTpConn

    -- Anti-Die + Lagger integrados
    tpBatStartAntiDie()
    tpBatStartLagger()

    if batDesyncTpSetVisual then batDesyncTpSetVisual(true) end
    updateTpBatButtonWithAntiDie(true)
end

function stopAlvaroTP()
    batDesyncTpEnabled = false
    _G.AlvaroTP.on     = false

    if batDesyncTpConn then
        batDesyncTpConn:Disconnect()
        batDesyncTpConn = nil
    end
    _G.AlvaroTP.conn            = nil
    _G.AlvaroTP.hittingCooldown = false
    _tpBatHitCooldown           = false

    -- Apagar Anti-Die + Lagger
    tpBatStopAntiDie()
    tpBatStopLagger()

    local meRoot = _G.AlvaroTP.hrp
    if meRoot and meRoot.Parent then
        pcall(function()
            if meRoot.SetNetworkOwner then meRoot:SetNetworkOwner(LP) end
        end)
    end

    if batDesyncTpSetVisual then batDesyncTpSetVisual(false) end
    updateTpBatButtonWithAntiDie(false)
end

function toggleAlvaroTP()
    if batDesyncTpEnabled then stopAlvaroTP() else startAlvaroTP() end
end

alvaroShiftInputConn = UIS.InputBegan:Connect(function(inp, gp)
    if gp then return end
    if _G.AlvaroTP.on then
        if inp.KeyCode == Enum.KeyCode.LeftShift or inp.KeyCode == Enum.KeyCode.RightShift then
            pcall(function()
                UIS.MouseBehavior = Enum.MouseBehavior.Default
            end)
        end
    end
end)

-- Reenganche al respawn (guarda referencias de HRP/Humanoid)
if _G.AlvaroTP._charConn then
    pcall(function() _G.AlvaroTP._charConn:Disconnect() end)
    _G.AlvaroTP._charConn = nil
end
local function tpBatSetupChar(char)
    task.wait(0.15)
    _G.AlvaroTP.h   = char and char:FindFirstChildOfClass("Humanoid") or nil
    _G.AlvaroTP.hrp = char and char:FindFirstChild("HumanoidRootPart") or nil
    if batDesyncTpEnabled and not batDesyncTpConn then
        startAlvaroTP()
    end
end
alvaroCharAddedConn = LP.CharacterAdded:Connect(function(char)
    _G.AlvaroTP._charConn = nil
    pcall(function() tpBatSetupChar(char) end)
end)
if LP.Character then
    task.spawn(function() pcall(function() tpBatSetupChar(LP.Character) end) end)
end

function startBatDesyncTp()
    if not _G.AmbitiousSafeModeTryStart() then return end
    if not _tpBatUnwalkForced then
        startUnwalk()
        _tpBatUnwalkForced = true
    end
    startAlvaroTP()
end

function stopBatDesyncTp()
    stopAlvaroTP()
    if _tpBatUnwalkForced then
        stopUnwalk()
                _tpBatUnwalkForced = false
    end
end

function toggleBatDesyncTp()
    if batDesyncTpEnabled then
        stopBatDesyncTp()
    else
        disableAllAimbots()
        startBatDesyncTp()
    end
    saveAllSettings()
end

function setTPBatVersion(_) end

function updateTpBatButtonWithAntiDie(state)
    if batDesyncTpSetVisual then batDesyncTpSetVisual(state) end
end

_G.AlvaroTPBat = {
    toggle    = toggleAlvaroTP,
    start     = startAlvaroTP,
    stop      = stopAlvaroTP,
    getStatus = function() return _G.AlvaroTP.on end,
}

-- ────────────────────────────────────────────────────────────────

function findBat()
    local char = LP.Character
    if not char then return nil end
    for _, name in ipairs(BAT_COUNTER_SLAP_LIST) do
        local t = char:FindFirstChild(name)
        if t and t:IsA("Tool") then return t end
    end
    local bp = LP:FindFirstChildOfClass("Backpack")
    if bp then
        for _, name in ipairs(BAT_COUNTER_SLAP_LIST) do
            local t = bp:FindFirstChild(name)
            if t and t:IsA("Tool") then
                local hum = char:FindFirstChildOfClass("Humanoid")
                if hum then pcall(function() hum:EquipTool(t) end) end
                return t
            end
        end
    end
    for _, ch in ipairs(char:GetChildren()) do
        if ch:IsA("Tool") and (ch.Name:lower():find("bat") or ch.Name:lower():find("slap")) then
            return ch
        end
    end
    return nil
end

function isBatTool(tool)
    if not tool then return false end
    for _, name in ipairs(BAT_COUNTER_SLAP_LIST) do
        if tool.Name == name then return true end
    end
    return tool.Name:lower():find("bat") or tool.Name:lower():find("slap")
end

function findBatForCounter()
    local char = LP.Character
    if not char then return nil end
    local backpack = LP:FindFirstChildOfClass("Backpack")
    for _, name in ipairs(BAT_COUNTER_SLAP_LIST) do
        local tool = char:FindFirstChild(name) or (backpack and backpack:FindFirstChild(name))
        if tool then return tool end
    end
    for _, child in ipairs(char:GetChildren()) do
        if child:IsA("Tool") and (child.Name:lower():find("bat") or child.Name:lower():find("slap")) then
            return child
        end
    end
    if backpack then
        for _, child in ipairs(backpack:GetChildren()) do
            if child:IsA("Tool") and (child.Name:lower():find("bat") or child.Name:lower():find("slap")) then
                return child
            end
        end
    end
    return nil
end

function swingBatForCounter(bat, character)
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if bat.Parent ~= character and humanoid then
        pcall(function() humanoid:EquipTool(bat) end)
        task.wait(0.05)
    end
    local remote = bat:FindFirstChildOfClass("RemoteEvent") or bat:FindFirstChildOfClass("RemoteFunction")
    if remote and remote:IsA("RemoteEvent") then
        pcall(function() remote:FireServer() end)
        task.wait(0.1)
        pcall(function() remote:FireServer() end)
    else
        pcall(function() bat:Activate() end)
        task.wait(0.1)
        pcall(function() bat:Activate() end)
    end
end

batCounterDebounce = false

function stopBatCounter()
    if Conns.batCounter then
        Conns.batCounter:Disconnect()
        Conns.batCounter = nil
    end
    batCounterDebounce = false
end

function startBatCounter()
    if Conns.batCounter then return end
    Conns.batCounter = RunService.Heartbeat:Connect(function()
        if not batCounterEnabled then return end
        if batCounterDebounce then return end
        local character = LP.Character
        if not character then return end
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if not humanoid then return end
        local state = humanoid:GetState()
        if state == Enum.HumanoidStateType.Physics or state == Enum.HumanoidStateType.Ragdoll or state == Enum.HumanoidStateType.FallingDown then
            batCounterDebounce = true
            _suppressBodyLock()
            task.spawn(function()
                task.wait(0.15)
                local bat = findBatForCounter()
                if bat then swingBatForCounter(bat, character) end
                task.wait(0.3)
                batCounterDebounce = false
                _unsuppressBodyLock(true)
            end)
        end
    end)
end

function findMedusa()
    local c = LP.Character
    if not c then return nil end
    for _, t in ipairs(c:GetChildren()) do
        if t:IsA("Tool") then
            local n = t.Name:lower()
            if n:find("medusa") or n:find("head") or n:find("stone") then return t end
        end
    end
    local bp = LP:FindFirstChild("Backpack")
    if bp then
        for _, t in ipairs(bp:GetChildren()) do
            if t:IsA("Tool") then
                local n = t.Name:lower()
                if n:find("medusa") or n:find("head") or n:find("stone") then return t end
            end
        end
    end
    return nil
end

local DROP_ASCEND_DURATION = 0.22
local DROP_ASCEND_SPEED = 160
local _dropConn = nil

function stopDropBrainrot()
    dropActive = false
    if _dropConn then
        _dropConn:Disconnect()
        _dropConn = nil
    end
    for _, t in ipairs(dropConnections) do
        if type(t) == "thread" then pcall(task.cancel, t)
        elseif type(t) == "RBXScriptConnection" then pcall(t.Disconnect, t) end
    end
    dropConnections = {}
    local c = LP.Character
    if c then
        local root = c:FindFirstChild("HumanoidRootPart")
        if root then root.AssemblyLinearVelocity = _V3zero end
    end
    if dropBrainrotSetVisual then dropBrainrotSetVisual(false) end
    if mobSetDropBR then mobSetDropBR(false) end
end

function runDropBrainrot()
    if dropActive then return end
    local char = LP.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if not root or not hum then return end
    if dropMode == 1 then
        local speedH = 0
        if root then
            local vel = root.AssemblyLinearVelocity
            speedH = _V3new(vel.X, 0, vel.Z).Magnitude
        end
        local cooldown = (speedH > 5) and 0.6 or 0.25
        if _tick() - lastDropTime < cooldown then return end
        lastDropTime = _tick()
        dropActive = true
        if dropBrainrotSetVisual then dropBrainrotSetVisual(true) end
        if mobSetDropBR then mobSetDropBR(true) end
        local wasAutoBat = false
        if autoBatEnabled then
            wasAutoBat = true
            disableAutoBat()
            if autoBatSetVisual then autoBatSetVisual(false) end
            if mobSetAutoBat then mobSetAutoBat(false) end
        end
        local function finishDrop(threadRef)
            if threadRef and dropConnections then
                for i = #dropConnections, 1, -1 do
                    if dropConnections[i] == threadRef then
                        table.remove(dropConnections, i)
                        break
                    end
                end
            end
            dropActive = false
            local c = LP.Character
            if c then
                local r = c:FindFirstChild("HumanoidRootPart")
                local h = c:FindFirstChildOfClass("Humanoid")
                if r then
                    r.AssemblyLinearVelocity = _V3zero
                    r.AssemblyAngularVelocity = _V3zero
                    if r.Position.Y < -100 then
                        r.CFrame = _CFnew(r.Position.X, 5, r.Position.Z)
                    end
                    local rp = RaycastParams.new()
                    rp.FilterDescendantsInstances = {c}
                    rp.FilterType = Enum.RaycastFilterType.Exclude
                    local rr = workspace:Raycast(r.Position, _V3new(0, -2000, 0), rp)
                    if rr then
                        local off = (h and h.HipHeight or 2) + (r.Size.Y / 2)
                        r.CFrame = _CFnew(r.Position.X, rr.Position.Y + off, r.Position.Z)
                    end
                    if h and h.Health > 0 then h:ChangeState(Enum.HumanoidStateType.Running) end
                end
            end
            if wasAutoBat then
                enableAutoBat()
                if autoBatSetVisual then autoBatSetVisual(true) end
                if mobSetAutoBat then mobSetAutoBat(true) end
            end
            if dropBrainrotSetVisual then dropBrainrotSetVisual(false) end
            if mobSetDropBR then mobSetDropBR(false) end
        end
        local flingThread = nil
        flingThread = task.spawn(function()
            local startTime = _tick()
            while dropActive and (_tick() - startTime) < 0.25 do
                RunService.Heartbeat:Wait()
                local c = LP.Character
                local r = c and c:FindFirstChild("HumanoidRootPart")
                if not r then break end
                local vel = r.AssemblyLinearVelocity
                vel = _V3new(0, vel.Y, 0)
                r.AssemblyLinearVelocity = vel * 10000 + _V3new(0, 10000, 0)
                RunService.RenderStepped:Wait()
                if r and r.Parent then r.AssemblyLinearVelocity = vel end
                RunService.Stepped:Wait()
                if r and r.Parent then r.AssemblyLinearVelocity = vel + _V3new(0, 0.1, 0) end
            end
            finishDrop(flingThread)
        end)
        table.insert(dropConnections, flingThread)
        task.delay(0.35, function()
            if dropActive then finishDrop(flingThread) end
        end)
        return
    end
    dropActive = true
    if dropBrainrotSetVisual then dropBrainrotSetVisual(true) end
    if mobSetDropBR then mobSetDropBR(true) end
    local t0 = _tick()
    if _dropConn then _dropConn:Disconnect() end
    _dropConn = RunService.Heartbeat:Connect(function()
        local c = LP.Character
        local r = c and c:FindFirstChild("HumanoidRootPart")
        if not r then
            if _dropConn then _dropConn:Disconnect(); _dropConn = nil end
            dropActive = false
            if dropBrainrotSetVisual then dropBrainrotSetVisual(false) end
            if mobSetDropBR then mobSetDropBR(false) end
            return
        end
        if not dropActive then
            if _dropConn then _dropConn:Disconnect(); _dropConn = nil end
            if dropBrainrotSetVisual then dropBrainrotSetVisual(false) end
            if mobSetDropBR then mobSetDropBR(false) end
            return
        end
        if _tick() - t0 >= DROP_ASCEND_DURATION then
            if _dropConn then _dropConn:Disconnect(); _dropConn = nil end
            pcall(function()
                local rp = RaycastParams.new()
                rp.FilterDescendantsInstances = {c}
                rp.FilterType = Enum.RaycastFilterType.Exclude
                local rr = workspace:Raycast(r.Position, _V3new(0, -3000, 0), rp)
                if rr then
                    local hum2 = c:FindFirstChildOfClass("Humanoid")
                    local off = ((hum2 and hum2.HipHeight) or 2) + (r.Size.Y / 2)
                    r.CFrame = _CFnew(r.Position.X, rr.Position.Y + off, r.Position.Z)
                    r.AssemblyLinearVelocity = _V3zero
                    r.AssemblyAngularVelocity = _V3zero
                end
                if hum2 and hum2.Health > 0 then hum2:ChangeState(Enum.HumanoidStateType.Running) end
            end)
            dropActive = false
            if dropBrainrotSetVisual then dropBrainrotSetVisual(false) end
            if mobSetDropBR then mobSetDropBR(false) end
            return
        end
        local lv = r.AssemblyLinearVelocity
        r.AssemblyLinearVelocity = _V3new(lv.X, DROP_ASCEND_SPEED, lv.Z)
    end)
end

function executeDropWithToggle(setVisual)
    if dropActive then return end
    task.spawn(function()
        if setVisual then setVisual(true) end
        runDropBrainrot()
        while dropActive do task.wait() end
        task.wait(0.1)
        if setVisual then setVisual(false) end
    end)
end

local function applyAntiLagDerender(obj)
    pcall(function()
        if obj:GetAttribute("_AmbitiousHubSky") ~= nil then return end
        if obj:IsA("Sky") or obj:IsA("Atmosphere") or obj:IsA("Clouds") then return end
        if skyTheme ~= "Off" and obj.Parent == Lighting then
            if obj:IsA("BlurEffect") or obj:IsA("SunRaysEffect")
            or obj:IsA("ColorCorrectionEffect") or obj:IsA("BloomEffect")
            or obj:IsA("DepthOfFieldEffect") or obj:IsA("BrightnessEffect")
            or obj:IsA("DitheringEffect") then return end
        end
        if obj:IsA("Accessory") or obj:IsA("Hat") then
            local char = LP.Character
            if char and obj:IsDescendantOf(char) then return end
            obj:Destroy()
        elseif obj:IsA("BasePart") then
            obj.Material      = Enum.Material.Plastic
            obj.Reflectance   = 0
            obj.CastShadow    = false
            obj.TopSurface    = Enum.SurfaceType.Smooth
            obj.BottomSurface = Enum.SurfaceType.Smooth
            obj.FrontSurface  = Enum.SurfaceType.Smooth
            obj.BackSurface   = Enum.SurfaceType.Smooth
            obj.LeftSurface   = Enum.SurfaceType.Smooth
            obj.RightSurface  = Enum.SurfaceType.Smooth
            for _, child in ipairs(obj:GetChildren()) do
                if child:IsA("Decal") or child:IsA("Texture") then
                    child.Transparency = 0.5
                end
            end
        elseif obj:IsA("Decal") or obj:IsA("Texture") then
            obj.Transparency = 0.5
        elseif obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Beam")
            or obj:IsA("Fire") or obj:IsA("Smoke") or obj:IsA("Sparkles") then
            obj.Enabled = false
        elseif obj:IsA("PointLight") or obj:IsA("SpotLight")
            or obj:IsA("SurfaceLight") then
            obj.Enabled = false
        elseif obj:IsA("SelectionBox") and obj.Name ~= "AmbitiousHubESP" then
            pcall(function() obj:Destroy() end)
        end
    end)
end

function enableAntiLag()
    antiLagEnabled           = true
    removeAccessoriesEnabled = true

    _G._AmbitiousDefLightBrightness = _G._AmbitiousDefLightBrightness or Lighting.Brightness
    _G._AmbitiousDefLightClock      = _G._AmbitiousDefLightClock      or Lighting.ClockTime
    _G._AmbitiousDefLightAmbient    = _G._AmbitiousDefLightAmbient    or Lighting.OutdoorAmbient

    pcall(function()
        settings().Rendering.QualityLevel        = 1
        settings().Rendering.MeshPartDetailLevel = Enum.MeshPartDetailLevel.Disabled
    end)

    if skyTheme == "Off" then
        Lighting.GlobalShadows            = false
        Lighting.FogEnd                   = 1e10
        Lighting.FogStart                 = 0
        Lighting.Brightness               = 1
        Lighting.EnvironmentDiffuseScale  = 0
        Lighting.EnvironmentSpecularScale = 0
        Lighting.ShadowSoftness           = 0
        Lighting.ExposureCompensation     = 0
        pcall(function() Lighting.Technology = Enum.Technology.Compatibility end)

        for _, e in pairs(Lighting:GetChildren()) do
            pcall(function()
                if e:GetAttribute("_AmbitiousVivid") then return end
                if e:IsA("BlurEffect") or e:IsA("SunRaysEffect")
                or e:IsA("ColorCorrectionEffect") or e:IsA("BloomEffect")
                or e:IsA("DepthOfFieldEffect") or e:IsA("BrightnessEffect")
                or e:IsA("DitheringEffect") then
                    e.Enabled = false
                end
            end)
        end
    end

    for _, obj in ipairs(workspace:GetDescendants()) do
        applyAntiLagDerender(obj)
    end

    for _, descendant in pairs(workspace:GetDescendants()) do
        pcall(function()
            if descendant:IsA("ParticleEmitter") then
                descendant.Enabled = false
            elseif descendant:IsA("Decal") then
                descendant.Transparency = 1
            elseif descendant:IsA("BasePart") then
                local char = LP.Character
                if char and descendant:IsDescendantOf(char) then return end
                descendant.Material      = Enum.Material.Plastic
                descendant.Reflectance   = 0
                descendant.CastShadow    = false
                descendant.TopSurface    = Enum.SurfaceType.Smooth
                descendant.BottomSurface = Enum.SurfaceType.Smooth
                descendant.FrontSurface  = Enum.SurfaceType.Smooth
                descendant.BackSurface   = Enum.SurfaceType.Smooth
                descendant.LeftSurface   = Enum.SurfaceType.Smooth
                descendant.RightSurface  = Enum.SurfaceType.Smooth
            end
        end)
    end

    pcall(function()
        workspace.Terrain.WaterWaveSize     = 0
        workspace.Terrain.WaterWaveSpeed    = 0
        workspace.Terrain.WaterReflectance  = 0
        workspace.Terrain.WaterTransparency = 0
        workspace.Terrain.Decoration        = false
    end)

    if antiLagDescConn then antiLagDescConn:Disconnect() end
    antiLagDescConn = workspace.DescendantAdded:Connect(function(obj)
        if obj:GetAttribute("_AmbitiousHubSky") then return end
        if obj:IsA("Sky") or obj:IsA("Atmosphere") or obj:IsA("Clouds") then return end
        if removeAccessoriesEnabled then applyAntiLagDerender(obj) end
    end)
end

function disableAntiLag()
    antiLagEnabled           = false
    removeAccessoriesEnabled = false
    if antiLagDescConn then
        antiLagDescConn:Disconnect()
        antiLagDescConn = nil
    end
    if skyTheme == "Off" then
        pcall(function()
            if _G._AmbitiousDefLightBrightness then
                Lighting.Brightness = _G._AmbitiousDefLightBrightness
            end
            if _G._AmbitiousDefLightClock then
                Lighting.ClockTime = _G._AmbitiousDefLightClock
            end
            if _G._AmbitiousDefLightAmbient then
                Lighting.OutdoorAmbient = _G._AmbitiousDefLightAmbient
            end
            Lighting.ExposureCompensation = 0
            Lighting.GlobalShadows = true
        end)
    end
end

CUSTOM_FOV_BIND = "FictionHubCustomFOV"

function enableCustomFov()
    local cam = workspace.CurrentCamera
    if cam and origFOV == nil then origFOV = cam.FieldOfView end
    fovEnabled = true
    if cam then
        pcall(function() cam.FieldOfViewMode = Enum.FieldOfViewMode.Diagonal end)
        pcall(function() cam.FieldOfView = fovValue end)
    end
    if customFovConn then customFovConn:Disconnect(); customFovConn = nil end
    pcall(function() RunService:UnbindFromRenderStep(CUSTOM_FOV_BIND) end)
    local prio = 200
    pcall(function() prio = Enum.RenderPriority.Camera.Value + 10 end)
    local ok = pcall(function()
        RunService:BindToRenderStep(CUSTOM_FOV_BIND, prio, function()
            if not fovEnabled then return end
            local c = workspace.CurrentCamera
            if c and c.FieldOfView ~= fovValue then
                c.FieldOfView = fovValue
            end
        end)
    end)
    if not ok then
        customFovConn = RunService.RenderStepped:Connect(function()
            if not fovEnabled then
                if customFovConn then customFovConn:Disconnect(); customFovConn = nil end
                return
            end
            local c = workspace.CurrentCamera
            if c then c.FieldOfView = fovValue end
        end)
    end
end

function disableCustomFov()
    fovEnabled = false
    pcall(function() RunService:UnbindFromRenderStep(CUSTOM_FOV_BIND) end)
    if customFovConn then customFovConn:Disconnect(); customFovConn = nil end
    local cam = workspace.CurrentCamera
    if cam then
        pcall(function() cam.FieldOfViewMode = Enum.FieldOfViewMode.Vertical end)
        pcall(function() cam.FieldOfView = origFOV or 70 end)
    end
end

function applyStretchFOV(val)
    local cam = workspace.CurrentCamera
    if cam then pcall(function() cam.FieldOfView = val end) end
end

function enableStretch()
    if stretchConn then return end
    stretchEnabled = true
    local cam = workspace.CurrentCamera
    if not cam then return end
    origFOV = cam.FieldOfView or 70
    applyStretchFOV(stretchFOV)
    stretchConn = RunService.RenderStepped:Connect(function()
        if not stretchEnabled then
            stretchConn:Disconnect()
            stretchConn = nil
            return
        end
        local c = workspace.CurrentCamera
        if c then c.CFrame = c.CFrame * _CFnew(0,0,0,1,0,0,0,0.7,0,0,0,1) end
    end)
    if stretchFovConn then stretchFovConn:Disconnect() end
    stretchFovConn = RunService.RenderStepped:Connect(function()
        if stretchEnabled then applyStretchFOV(stretchFOV)
        else stretchFovConn:Disconnect(); stretchFovConn = nil end
    end)
end

function disableStretch()
    stretchEnabled = false
    if stretchConn then stretchConn:Disconnect(); stretchConn = nil end
    if stretchFovConn then stretchFovConn:Disconnect(); stretchFovConn = nil end
    local cam = workspace.CurrentCamera
    if cam then pcall(function() cam.FieldOfView = origFOV or 70 end) end
end

local function saveLightingState()
    if _originalLighting then return end
    _originalLighting = {
        Brightness = Lighting.Brightness,
        ClockTime = Lighting.ClockTime,
        OutdoorAmbient = Lighting.OutdoorAmbient,
        GlobalShadows = Lighting.GlobalShadows,
        FogEnd = Lighting.FogEnd,
        FogStart = Lighting.FogStart,
        FogColor = Lighting.FogColor,
        Ambient = Lighting.Ambient,
        ColorCorrection = nil,
        Bloom = nil,
    }
    for _, e in ipairs(Lighting:GetChildren()) do
        if e:IsA("ColorCorrectionEffect") then
            _originalLighting.ColorCorrection = {
                Enabled = e.Enabled,
                Brightness = e.Brightness,
                Contrast = e.Contrast,
                Saturation = e.Saturation,
                TintColor = e.TintColor,
            }
        elseif e:IsA("BloomEffect") then
            _originalLighting.Bloom = {
                Enabled = e.Enabled,
                Intensity = e.Intensity,
                Size = e.Size,
                Threshold = e.Threshold,
            }
        end
    end
end

local function restoreLightingState()
    if not _originalLighting then return end
    local old = _originalLighting
    Lighting.Brightness = old.Brightness
    Lighting.ClockTime = old.ClockTime
    Lighting.OutdoorAmbient = old.OutdoorAmbient
    Lighting.GlobalShadows = old.GlobalShadows
    Lighting.FogEnd = old.FogEnd
    Lighting.FogStart = old.FogStart
    Lighting.FogColor = old.FogColor
    Lighting.Ambient = old.Ambient
    if old.ColorCorrection then
        local cc = Lighting:FindFirstChildOfClass("ColorCorrectionEffect")
        if cc then
            cc.Enabled = old.ColorCorrection.Enabled
            cc.Brightness = old.ColorCorrection.Brightness
            cc.Contrast = old.ColorCorrection.Contrast
            cc.Saturation = old.ColorCorrection.Saturation
            cc.TintColor = old.ColorCorrection.TintColor
        end
    end
    if old.Bloom then
        local bloom = Lighting:FindFirstChildOfClass("BloomEffect")
        if bloom then
            bloom.Enabled = old.Bloom.Enabled
            bloom.Intensity = old.Bloom.Intensity
            bloom.Size = old.Bloom.Size
            bloom.Threshold = old.Bloom.Threshold
        end
    end
end

-- Custom sky themes removed; the game keeps its default environment.

local function applyNeonWeather()
    if not neonWeatherEnabled then
        restoreLightingState()
        return
    end
    if not _originalLighting then saveLightingState() end
    Lighting.Brightness = 2.2
    Lighting.ClockTime = 20
    Lighting.OutdoorAmbient = Color3.fromRGB(60, 80, 120)
    Lighting.GlobalShadows = false
    Lighting.FogEnd = 800
    Lighting.FogStart = 0
    Lighting.FogColor = Color3.fromRGB(60, 120, 200)
    Lighting.Ambient = Color3.fromRGB(60, 90, 140)
    local cc = Lighting:FindFirstChildOfClass("ColorCorrectionEffect")
    if not cc then
        cc = Instance.new("ColorCorrectionEffect")
        cc.Parent = Lighting
    end
    cc.Enabled = true
    cc.Brightness = 0.1
    cc.Contrast = 0.08
    cc.Saturation = 0.08
    cc.TintColor = Color3.fromRGB(200, 200, 210)
    local bloom = Lighting:FindFirstChildOfClass("BloomEffect")
    if not bloom then
        bloom = Instance.new("BloomEffect")
        bloom.Parent = Lighting
    end
    bloom.Enabled = true
    bloom.Intensity = 0.4
    bloom.Size = 20
    bloom.Threshold = 0.9
end

function toggleNeonWeather(state)
    if state == nil then
        neonWeatherEnabled = not neonWeatherEnabled
    else
        neonWeatherEnabled = state
    end
    applyNeonWeather()
    if setNeonWeatherVisual then setNeonWeatherVisual(neonWeatherEnabled) end
end

function paintFloatingBtn(btnFrame, active)
    if not btnFrame then return end
    local bg = btnFrame:FindFirstChild("BtnGrad")
    local label = btnFrame:FindFirstChild("TextLabel")
    local stroke = btnFrame:FindFirstChildOfClass("UIStroke")
    btnFrame.BackgroundColor3 = Color3.fromRGB(255, 255, 255)

    if active then
        if bg then
            bg.Color = ColorSequence.new({
                ColorSequenceKeypoint.new(0.00, Color3.fromRGB(255, 255, 255)),
                ColorSequenceKeypoint.new(0.35, Color3.fromRGB(248, 248, 252)),
                ColorSequenceKeypoint.new(0.70, Color3.fromRGB(225, 225, 235)),
                ColorSequenceKeypoint.new(1.00, Color3.fromRGB(195, 195, 208)),
            })
        end
        if label then label.TextColor3 = Color3.fromRGB(15, 15, 18) end
        if stroke then
            stroke.Color = Color3.fromRGB(255, 255, 255)
            stroke.Thickness = 1.8
            stroke.Transparency = 0.05
        end
    else
        if bg then
            bg.Color = ColorSequence.new({
                ColorSequenceKeypoint.new(0.00, Color3.fromRGB(70, 70, 78)),
                ColorSequenceKeypoint.new(0.35, Color3.fromRGB(28, 28, 34)),
                ColorSequenceKeypoint.new(0.70, Color3.fromRGB(10, 10, 14)),
                ColorSequenceKeypoint.new(1.00, Color3.fromRGB(0, 0, 0)),
            })
        end
        if label then label.TextColor3 = Color3.fromRGB(255, 255, 255) end
        if stroke then
            stroke.Color = Color3.fromRGB(70, 70, 78)
            stroke.Thickness = 1
            stroke.Transparency = 0.45
        end
    end
end

function applyFloatingButtonScale()
    for _, uiScale in ipairs(_floatingUIScales) do
        if uiScale and uiScale.Parent then
            uiScale.Scale = floatingButtonScale
        end
    end
end

local function drag(f)
    local dn, ds, sp, di = false, nil, nil, nil
    local endConn = nil
    local function stopDrag()
        dn = false
        di = nil
        if endConn then
            endConn:Disconnect()
            endConn = nil
        end
        pcall(saveAllSettings)
    end
    f.InputBegan:Connect(function(i)
        if uiLocked then return end
        if _isDraggingButton then return end
        if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
            dn = true; ds = i.Position; sp = f.Position
            if endConn then endConn:Disconnect() end
            endConn = i.Changed:Connect(function()
                if i.UserInputState == Enum.UserInputState.End then stopDrag() end
            end)
        end
    end)
    f.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
            stopDrag()
        end
    end)
    UIS.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
            stopDrag()
        end
    end)
    f.InputChanged:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch then di = i end
    end)
    UIS.InputChanged:Connect(function(i)
        if i == di and dn then
            if uiLocked then stopDrag(); return end
            if _isDraggingButton then return end
            if not ds or not sp then return end
            local nX = sp.X.Offset + (i.Position.X - ds.X)
            local nY = sp.Y.Offset + (i.Position.Y - ds.Y)
            f.Position = UDim2.new(sp.X.Scale, nX, sp.Y.Scale, nY)
        end
    end)
end

function setupMovementAndIndicators(char)
    if steppedConn then steppedConn:Disconnect(); steppedConn = nil end
    if movementLoop then movementLoop:Disconnect(); movementLoop = nil end

    local ccAcc = 0
    steppedConn = RunService.Heartbeat:Connect(function(dt)
        ccAcc = ccAcc + dt
        if ccAcc < 0.05 then return end
        ccAcc = 0
        local plist = _GetPlayersCached()
        for i = 1, #plist do
            local p = plist[i]
            if p ~= LP then
                local ch = p.Character
                if ch then
                    local parts = ch:GetChildren()
                    for j = 1, #parts do
                        local part = parts[j]
                        if part:IsA("BasePart") and part.CanCollide then
                            part.CanCollide = false
                        end
                    end
                end
            end
        end
    end)

    movementLoop = RunService.RenderStepped:Connect(function()
        local char2 = LP.Character
        if not char2 then return end
        local hum = char2:FindFirstChildOfClass("Humanoid")
        local hrp = char2:FindFirstChild("HumanoidRootPart")
        if not hum or not hrp then return end

        if not autoBatEnabled
           and not autoBatV2Enabled and not batDesyncTpEnabled
           and not useCarrySystem then
            if _isRagdollState(hum) then
                lastMoveDir = _V3zero
            else
                local md = hum.MoveDirection
                local spd = getActiveMoveSpeed()
                local dir = nil
                if md.Magnitude > 0 then
                    lastMoveDir = md
                    dir = md
                elseif lastMoveDir.Magnitude > 0 then
                    for key in pairs(MOVE_KEYS) do
                        if UIS:IsKeyDown(key) then dir = lastMoveDir; break end
                    end
                end
                _applyVelocitySpeed(dir, spd, hrp)
            end
        end

        if speedLabel then
            local v = hrp.AssemblyLinearVelocity
            local s = _sqrt(v.X*v.X + v.Z*v.Z)
            if s < 0.05 then s = 0 end
            speedLabel.Text = string.format("%.1f speed", s)
        end
    end)
    setupSpeedIndicator(char)
    startEnemySpeed()
end

function toggleLockUI(state)
    if state == nil then uiLocked = not uiLocked else uiLocked = state end
    if uiLocked and editModeEnabled then
        editModeEnabled = false
        if setEditModeVisual then setEditModeVisual(false) end
    end
    if setLockUIVisual then setLockUIVisual(uiLocked) end
end

function toggleEditMode(state)
    if state == nil then state = not editModeEnabled end
    if state and uiLocked then state = false end
    editModeEnabled = state
    if setEditModeVisual then setEditModeVisual(state) end
end

function disableAllAimbots()
    if autoBatEnabled then
        disableAutoBat()
        if autoBatSetVisual then autoBatSetVisual(false) end
        if mobSetAutoBat then mobSetAutoBat(false) end
    end
    if batDesyncTpEnabled then stopBatDesyncTp() end
    if autoBatV2Enabled then
        disableBatV2()
        if autoBatV2SetVisual then autoBatV2SetVisual(false) end
        if batV2FloatingButton then
            local btnFrame = batV2FloatingButton:FindFirstChild("Frame")
            if btnFrame then paintFloatingBtn(btnFrame, false) end
        end
    end
end

function stopAllBackgroundTasks()
    if movementLoop then movementLoop:Disconnect(); movementLoop = nil end
    if steppedConn then steppedConn:Disconnect(); steppedConn = nil end
    stopEnemySpeed()
    if stretchEnabled then disableStretch() end
    if stretchConn then stretchConn:Disconnect(); stretchConn = nil end
    if stretchFovConn then stretchFovConn:Disconnect(); stretchFovConn = nil end
    if AntiRagdollV1.isRunning() then AntiRagdollV1.stop() end
    if AntiRagdollV2.Enabled then stopAntiRagdollV2() end
    if antiDieEnabled then AntiDieModule.stop() end
    if antiFlingEnabled then AntiFlingShieldModule.stop() end
    stopBatCounter()
    stopAutoSteal()
    disableAutoBat()
    if batDesyncTpEnabled then stopBatDesyncTp() end
    if autoBatV2Enabled then disableBatV2() end
    if antiLagEnabled then disableAntiLag() end
    if espEnabled then toggleESP(false) end
    if dropActive then stopDropBrainrot() end
    for _, t in ipairs(dropConnections) do
        if type(t) == "thread" then pcall(task.cancel, t)
        elseif type(t) == "RBXScriptConnection" then pcall(t.Disconnect, t) end
    end
    dropConnections = {}
    dropActive = false
    lastDropTime = 0
        end

-- ═══════════════════════════════════════════════════════════════════════════
-- CONFIG
-- ═══════════════════════════════════════════════════════════════════════════
function buildConfigTable()
    local config = {
        normalSpeed = NS,
        carrySpeed = CS,
        laggerSpeed1 = LAGGER_SPEED,
        laggerSpeed2 = LAGGER_CARRY_SPEED,
        stealRadius = CONFIG.STEAL_RANGE,
        antiRagdollMode = antiRagdollMode,
        antiDieEnabled = antiDieEnabled,
        antiFlingEnabled = antiFlingEnabled,
        autoBatEnabled = autoBatEnabled,
        autoSteal = CONFIG.AUTO_STEAL_ENABLED,
        batCounter = batCounterEnabled,
        laggerToggled = laggerToggled,
        laggerCarryToggled = laggerCarryToggled,
        carryMode = speedMode,
        batAimbotSpeed = BAT_AIMBOT_SPEED,
        dropMode = dropMode,
        stretchEnabled = stretchEnabled,
        stretchFOV = stretchFOV,
        fovEnabled = fovEnabled,
        fovValue = fovValue,
        uiScale = uiScaleValue,
        animPack = currentAnimPack,
        espEnabled = espEnabled,
        antiLag = antiLagEnabled,
        tpBatEnabled = batDesyncTpEnabled,
        neonWeather = neonWeatherEnabled,
        autoBatV2Enabled = autoBatV2Enabled,
        selectedAimbotMode              = selectedAimbotMode,
        ANTI_BYPASS_AIMBOT_SPEED        = _G.AceAntiBypassAimbotSpeed,
        ANTI_BYPASS_LAGGER_AIMBOT_SPEED = _G.AceAntiBypassLaggerAimbotSpeed,
        antiBypassAimbotEnabled         = _G.AceAntiBypassAimbotOn == true,
        katanaSkinEnabled = katanaSkinEnabled,
        cleanHubShowE01Warning = CleanHubShowE01Warning,
        holdJumpEnabled = infJumpEnabled,
        holdJumpMode    = infJumpMode,
        safeMode        = antiKickEnabled,
        mobileButtonPositions = savedButtonPositions,
        dropBrainrotKey = {kb = KB.DropBrainrot.kb and KB.DropBrainrot.kb.Name, gp = KB.DropBrainrot.gp and KB.DropBrainrot.gp.Name},
        autoBatKey = {kb = KB.AutoBat.kb and KB.AutoBat.kb.Name, gp = KB.AutoBat.gp and KB.AutoBat.gp.Name},
        tpFloorKey = {kb = KB.TPFloor.kb and KB.TPFloor.kb.Name, gp = KB.TPFloor.gp and KB.TPFloor.gp.Name},
        carryToggleKey = {kb = KB.CarryToggle.kb and KB.CarryToggle.kb.Name, gp = KB.CarryToggle.gp and KB.CarryToggle.gp.Name},
        laggerModeKey = {kb = KB.LaggerMode.kb and KB.LaggerMode.kb.Name, gp = KB.LaggerMode.gp and KB.LaggerMode.gp.Name},
        batV2Key = {kb = KB.BatV2.kb and KB.BatV2.kb.Name, gp = KB.BatV2.gp and KB.BatV2.gp.Name},
        instaResetKey = {kb = KB.InstaReset.kb and KB.InstaReset.kb.Name, gp = KB.InstaReset.gp and KB.InstaReset.gp.Name},
        tpBatFloatingPos = tpBatFloatingPos,
        batV2FloatingPos = batV2FloatingPos,
        instaResetFloatingPos = instaResetFloatingPos,
        progressBarPos = savedProgressBarPos,
        progressBarScale = progressBarScale,
        lockUI = uiLocked,
        editMode = editModeEnabled,
        floatingButtonScale = floatingButtonScale,
        outfitIndex = currentOutfitIndex,
        backgroundIndex = backgroundIndex,
        backgroundImageTransparency = backgroundImageTransparency,
        themeColor = currentColorTheme,
        useCarrySystem = useCarrySystem,
        autoStealVariant = autoStealVariant,
        carrySysNormal = CarrySystem.normalSpeed,
        carrySysCarry = CarrySystem.carrySpeed,
        carrySysLagger = CarrySystem.laggerSpeed,
        carrySysLaggerCarry = CarrySystem.laggerCarrySpeed,
        carrySysSoftStealSpeed = CarrySystem.carrySpeed,
        carrySysSoftStealRadius = CarrySystem.softStealRadius,
    }
    if pbFrame then
        config.progressBarPos = {
            XScale = pbFrame.Position.X.Scale,
            XOffset = pbFrame.Position.X.Offset,
            YScale = pbFrame.Position.Y.Scale,
            YOffset = pbFrame.Position.Y.Offset
        }
    end
    if MobilePanel and MobilePanel:FindFirstChild("FloatingPanel") then
        local container = MobilePanel:FindFirstChild("FloatingPanel")
        config.mobilePanelPos = {
            XScale = container.Position.X.Scale,
            XOffset = container.Position.X.Offset,
            YScale = container.Position.Y.Scale,
            YOffset = container.Position.Y.Offset
        }
    end
    return config
end

function saveAllSettings()
    if _isResetting or _isLoading then return true end
    local ok, configOrError = pcall(buildConfigTable)
    if not ok then
        warn("[russo] Could not build settings:", configOrError)
        return false
    end
    local encoded, jsonOrError = pcall(function() return HS:JSONEncode(configOrError) end)
    if not encoded then
        warn("[russo] Could not encode settings:", jsonOrError)
        return false
    end
    local json = jsonOrError
    local fileStillExists = false
    pcall(function()
        fileStillExists = isfile and isfile(CONFIG_FILE) == true
    end)
    if json == _lastSavedJSON and fileStillExists then return true end
    if type(writefile) ~= "function" then
        warn("[russo] This executor does not provide writefile; settings cannot be saved.")
        return false
    end
    local success, err = pcall(function() writefile(CONFIG_FILE, json) end)
    if success then
        _lastSavedJSON = json
    else
        warn("[russo] Failed to write settings:", err)
    end
    return success
end

function loadAllSettings()
    if not isfile or not isfile(CONFIG_FILE) then return false end
    local success, data = pcall(function() return HS:JSONDecode(readfile(CONFIG_FILE)) end)
    if not success or not data then return false end
    _isLoading = true
    NS = data.normalSpeed or NS
    CS = data.carrySpeed or CS
    LAGGER_SPEED = data.laggerSpeed1 or LAGGER_SPEED
    LAGGER_CARRY_SPEED = data.laggerSpeed2 or LAGGER_CARRY_SPEED
    CONFIG.STEAL_RANGE = data.stealRadius or CONFIG.STEAL_RANGE
    if radInput then radInput.Text = tostring(CONFIG.STEAL_RANGE) end
    if data.lockUI ~= nil then uiLocked = data.lockUI == true end
    editModeEnabled = data.editMode == true
    if data.antiRagdollMode then
        antiRagdollMode = data.antiRagdollMode
    else
        antiRagdollMode = data.antiRagdoll and "v2" or "off"
    end
    antiDieEnabled = data.antiDieEnabled == true
    antiFlingEnabled = data.antiFlingEnabled == true
    autoBatEnabled = data.autoBatEnabled == true
    if autoBatEnabled then
        task.defer(function()
            if enableAutoBat then pcall(enableAutoBat) end
            if autoBatSetVisual then autoBatSetVisual(true) end
        end)
    elseif autoBatSetVisual then
        autoBatSetVisual(false)
    end
    if antiFlingEnabled and AntiFlingShieldModule and AntiFlingShieldModule.start then
        task.defer(function() pcall(function() AntiFlingShieldModule.start() end) end)
    end
    CONFIG.AUTO_STEAL_ENABLED = data.autoSteal == true
    batCounterEnabled = data.batCounter or false
    antiLagEnabled = data.antiLag or false
    laggerToggled = data.laggerToggled or false
    speedMode = data.carryMode or false
    laggerCarryToggled = data.laggerCarryToggled or false

    uiScaleValue = data.uiScale or 78
    if mainUIScale then mainUIScale.Scale = uiScaleValue / 100 end
    progressBarScale = data.progressBarScale or 1
    if pbScale then pbScale.Scale = progressBarScale end
    espEnabled = data.espEnabled or false
    if espEnabled then toggleESP(true) else toggleESP(false) end
    currentColorTheme = "Gris"
    selectedColor = COLOR_THEMES["Gris"]

    _G.AceAntiBypassLoadFromConfig(data)
    autoBatV2Enabled = _G.AceAntiBypassAimbotOn == true
    if autoBatV2Enabled then
        task.defer(function()
            enableBatV2()
            if autoBatV2SetVisual then autoBatV2SetVisual(true) end
        end)
    else
        if autoBatV2SetVisual then autoBatV2SetVisual(false) end
    end

    katanaSkinEnabled = data.katanaSkinEnabled or false
    CleanHubShowE01Warning = data.cleanHubShowE01Warning == true
    if CleanHubShowE01WarningSetVisual then CleanHubShowE01WarningSetVisual(CleanHubShowE01Warning) end
    if katanaSkinEnabled then
        task.defer(function() pcall(function() KatanaSkin.ApplySkin("KATANA") end) end)
    end
    tpBatVersion = 1
    _G.__tpBatV2Distance = 8

    infJumpEnabled      = data.holdJumpEnabled == true
    infJumpMode         = (data.holdJumpMode == "MANUAL") and "MANUAL" or "HOLD"
    -- Re-apply the loaded state to the Hold Jump controller as well as its saved flag.
    if _G.setInfJumpInternal then
        pcall(_G.setInfJumpInternal, infJumpEnabled)
    end
    antiKickEnabled     = data.safeMode == true


    local tpBatStateLoaded = data.tpBatEnabled or false
    if tpBatStateLoaded then
        task.defer(function()
            startBatDesyncTp()
            if batDesyncTpSetVisual then batDesyncTpSetVisual(true) end
        end)
    else
        if batDesyncTpSetVisual then batDesyncTpSetVisual(false) end
    end
        neonWeatherEnabled = data.neonWeather or false
    if neonWeatherEnabled then
        task.defer(function() toggleNeonWeather(true) end)
    else
        toggleNeonWeather(false)
                end
    if data.animPack and ANIM_PACKS[data.animPack] then
        startAnimPack(data.animPack)
    else
        currentAnimPack = "Off"
        stopAnimPack()
    end
    local function lk(e, d)
        if not d then return end
        if d.kb and Enum.KeyCode[d.kb] then e.kb = Enum.KeyCode[d.kb] end
        if d.gp and Enum.KeyCode[d.gp] then e.gp = Enum.KeyCode[d.gp] end
    end
    lk(KB.DropBrainrot, data.dropBrainrotKey)
    lk(KB.AutoBat, data.autoBatKey)
    lk(KB.TPFloor, data.tpFloorKey)
    lk(KB.CarryToggle, data.carryToggleKey)
    lk(KB.LaggerMode, data.laggerModeKey)
    lk(KB.BatV2, data.batV2Key)
    lk(KB.InstaReset, data.instaResetKey)
    if data.mobileButtonPositions then savedButtonPositions = data.mobileButtonPositions end
    if data.mobilePanelPos then savedMobilePanelPos = data.mobilePanelPos end
    if data.tpBatFloatingPos then tpBatFloatingPos = data.tpBatFloatingPos end
    if data.batV2FloatingPos then batV2FloatingPos = data.batV2FloatingPos end
    if data.instaResetFloatingPos then instaResetFloatingPos = data.instaResetFloatingPos end
    if data.progressBarPos then savedProgressBarPos = data.progressBarPos end
    dropMode = data.dropMode or 1
    stretchEnabled = data.stretchEnabled or false
    fovValue = data.fovValue or 70
    fovEnabled = data.fovEnabled or false
    if fovSliderSet then fovSliderSet(fovValue) end
    if fovEnabled then enableCustomFov() end
    if setFovVisual then setFovVisual(fovEnabled) end
    stretchFOV = data.stretchFOV or 120
    BAT_AIMBOT_SPEED = data.batAimbotSpeed or BAT_AIMBOT_SPEED
    floatingButtonScale = data.floatingButtonScale or 1
    if data.outfitIndex then
        if data.outfitIndex == 6 then
            currentOutfitIndex = 2
        else
            currentOutfitIndex = 1
        end
        task.defer(function()
            pcall(function() applyOutfitByIndex(currentOutfitIndex) end)
            if outfitSelectorLabel then
                outfitSelectorLabel.Text = OUTFITS[currentOutfitIndex].label
            end
        end)
    end

    if data.backgroundIndex then
        backgroundIndex = _clamp(data.backgroundIndex, 1, #BACKGROUND_IMAGES)
    end
    if data.backgroundImageTransparency ~= nil then
        backgroundImageTransparency = _clamp(data.backgroundImageTransparency, 0, 1)
    end

    useCarrySystem = data.useCarrySystem or false
    autoStealVariant = _clamp(tonumber(data.autoStealVariant) or 1, 1, 3)
    if autoStealVariantLabel then
        autoStealVariantLabel.Text = autoStealVariantName(autoStealVariant)
    end
    CarrySystem.normalSpeed = data.carrySysNormal or NS
    CarrySystem.carrySpeed = data.carrySysCarry or CS
    CarrySystem.laggerSpeed = data.carrySysLagger or LAGGER_SPEED
    CarrySystem.laggerCarrySpeed = data.carrySysLaggerCarry or LAGGER_CARRY_SPEED
    CarrySystem.softStealSpeed = CarrySystem.carrySpeed
    CarrySystem.softStealRadius = data.carrySysSoftStealRadius or 10
    if useCarrySystem then
        CarrySystem:start()
        CarrySystem.speedToggled = speedMode
        if laggerCarryToggled then
            CarrySystem:setLaggerMode(2)
        elseif laggerToggled then
            CarrySystem:setLaggerMode(1)
        else
            CarrySystem:setLaggerMode(0)
        end
    else
        CarrySystem:stop()
    end

    autoBatEnabled = false
    if dropModeBtnRef then dropModeBtnRef.Text = dropMode == 1 and "Fling" or "Jump Drop" end
    refreshSpeedModeLabel()

    _isLoading = false
    _lastSavedJSON = nil
    pcall(saveAllSettings)
    return true
end

function forceResetUI()
    if normalBox then normalBox.Text = tostring(NS) end
    if carryBox then carryBox.Text = tostring(CS) end
    if radInput then radInput.Text = tostring(CONFIG.STEAL_RANGE) end
    if laggerBox then laggerBox.Text = tostring(LAGGER_SPEED) end
    if lagger2Box then lagger2Box.Text = tostring(LAGGER_CARRY_SPEED) end
    if batSpeedBox then batSpeedBox.Text = tostring(BAT_AIMBOT_SPEED) end
    if uiScaleBox then uiScaleBox.Text = tostring(uiScaleValue) end
    if dropModeBtnRef then dropModeBtnRef.Text = dropMode == 1 and "Fling" or "Jump Drop" end
    if carrySysNormalBox then carrySysNormalBox.Text = tostring(CarrySystem.normalSpeed) end
    if carrySysCarryBox then carrySysCarryBox.Text = tostring(CarrySystem.carrySpeed) end
    if carrySysLaggerBox then carrySysLaggerBox.Text = tostring(CarrySystem.laggerSpeed) end
    if carrySysLaggerCarryBox then carrySysLaggerCarryBox.Text = tostring(CarrySystem.laggerCarrySpeed) end
    if carrySysSoftStealSpeedBox then carrySysSoftStealSpeedBox.Text = tostring(CarrySystem.carrySpeed) end
    if carrySysSoftStealRadiusBox then carrySysSoftStealRadiusBox.Text = tostring(CarrySystem.softStealRadius) end
    if carrySystemToggleSetter then carrySystemToggleSetter(useCarrySystem) end
    local function safeSet(fn, val) if fn then fn(val) end end
    safeSet(autoBatSetVisual, false)
    safeSet(setBatCounterVisual, false)
    safeSet(setAntiLagVisual, false)
    safeSet(setLockUIVisual, false)
    safeSet(setEditModeVisual, false)
    safeSet(setInstaGrab, false)
    safeSet(batDesyncTpSetVisual, false)
    safeSet(setESPVIsual, false)
    safeSet(setNeonWeatherVisual, false)
    safeSet(autoBatV2SetVisual, false)
    safeSet(setAntiDieVisual, false)
    safeSet(setSafeModeVisual, false)
    safeSet(infJumpSetVisual, false)
    safeSet(infJumpModeSetVisual, nil)
        if _G.stretchToggleSetter then _G.stretchToggleSetter(false) end
    safeSet(mobSetAutoBat, false)
    safeSet(mobSetDropBR, false)
    safeSet(mobSetTpDown, false)
    safeSet(mobSetCarry, false)
    safeSet(mobSetLagger1, false)
    safeSet(mobSetLagger2, false)
    autoStealVariant = 1
    if autoStealVariantLabel then autoStealVariantLabel.Text = autoStealVariantName(1) end
    tpBatVersion = 1
    _G.__tpBatV2Distance = 8
    refreshSpeedModeLabel()
    updateProgressBarVisibility()
    disableAntiLag()
    toggleNeonWeather(false)
        disableBatV2()
    if antiDieEnabled then
        AntiDieModule.stop()
        antiDieEnabled = false
    end
    if antiFlingEnabled then
        AntiFlingShieldModule.stop()
        antiFlingEnabled = false
    end
    for _, ref in ipairs(keyButtonRefs) do
        local entry = ref.entry
        local label = (entry.gp and entry.gp.Name) or (entry.kb and entry.kb.Name) or "None"
        ref.btn.Text = label
    end
    currentColorTheme = "Gris"
    selectedColor = COLOR_THEMES["Gris"]
    updateAllUIThemeColors(selectedColor)
    backgroundIndex = 1
    if backgroundImage then
        applyBackground(1)
    end
    if contentPages and contentPages["Visual"] then
        local vPage = contentPages["Visual"]
        for _, child in ipairs(vPage:GetChildren()) do
            if child:IsA("Frame") and child:FindFirstChild("BackgroundPreview") then
                local previewImage = child.BackgroundPreview:FindFirstChild("PreviewImage")
                local previewPlaceholder = child.BackgroundPreview:FindFirstChild("PreviewPlaceholder")
                if previewImage then previewImage.Image = "" end
                if previewPlaceholder then previewPlaceholder.Visible = true end
                break
            end
        end
    end
    if miniBtn then miniBtn.TextColor3 = Color3.fromRGB(210, 210, 220) end
    local pGui = LP:FindFirstChild("PlayerGui")
    if pGui then
        local bb = pGui:FindFirstChild("RagCountdownBillboard")
        if bb then
            local lbl = bb:FindFirstChildOfClass("TextLabel")
            if lbl then lbl.TextColor3 = Color3.fromRGB(255, 255, 255) end
        end
    end
    if MobilePanel then
        local container = MobilePanel:FindFirstChild("FloatingPanel")
        if container then
            local btnContainer = container:FindFirstChild("ButtonsContainer")
            if btnContainer then
                for _, btn in ipairs(btnContainer:GetChildren()) do
                    if btn:IsA("TextButton") then
                        local label = btn:FindFirstChildOfClass("TextLabel")
                        if label then
                            local isActive = btn.BackgroundColor3 == selectedColor
                            if not isActive then label.TextColor3 = selectedColor end
                        end
                    end
                end
            end
        end
    end
    saveAllSettings()
end

function resetFloatingPositions()
    if MobilePanel and MobilePanel:FindFirstChild("FloatingPanel") then
        local container = MobilePanel:FindFirstChild("FloatingPanel")
        container.Position = UDim2.new(0, 10, 0, 0)
        savedButtonPositions = {}
        if container:FindFirstChild("ButtonsContainer") then
            for _, btn in ipairs(container.ButtonsContainer:GetChildren()) do
                if btn:IsA("TextButton") and btn.Name then
                    local defX, defY = getDefaultButtonPosition(btn.Name)
                    btn.Position = UDim2.new(0, defX, 0, defY)
                end
            end
        end
    end
    if tpBatFloatingButton and tpBatFloatingButton:FindFirstChild("Frame") then
        local btnFrame = tpBatFloatingButton:FindFirstChild("Frame")
        btnFrame.Position = UDim2.new(0.5, 20, 0, 10)
        tpBatFloatingPos = nil
    end
    if batV2FloatingButton and batV2FloatingButton:FindFirstChild("Frame") then
        local btnFrame = batV2FloatingButton:FindFirstChild("Frame")
        btnFrame.Position = UDim2.new(0.5, -50, 0, 10)
        batV2FloatingPos = nil
    end
    if instaResetFloatingButton and instaResetFloatingButton:FindFirstChild("Frame") then
        instaResetFloatingButton.Frame.Position = UDim2.new(0.5, 90, 0, 10)
        instaResetFloatingPos = nil
    end
    if pbFrame then
        pbFrame.Position = UDim2.new(0.5, -160, 1, -60)
        savedProgressBarPos = nil
    end
    savedMobilePanelPos = nil
    tpBatFloatingPos = nil
    batV2FloatingPos = nil
end

function resetToFactoryDefaults()
    _isResetting = true
    local ok, err = pcall(function()
        stopAllBackgroundTasks()
        stopAutoSteal()
        stopBatCounter()
                if AntiRagdollV1.isRunning() then AntiRagdollV1.stop() end
        if AntiRagdollV2.Enabled then stopAntiRagdollV2() end
        if antiDieEnabled then AntiDieModule.stop() end
        if antiFlingEnabled then AntiFlingShieldModule.stop() end
        stopUnwalk()
        disableAutoBat()
        if batDesyncTpEnabled then stopBatDesyncTp() end
        disableBatV2()
            if espEnabled then toggleESP(false) end
        if stretchEnabled then disableStretch() end
        if antiLagEnabled then disableAntiLag() end
        if dropActive then stopDropBrainrot() end
        toggleNeonWeather(false)
                    if antiDieEnabled then
            AntiDieModule.stop()
            antiDieEnabled = false
        end
        if antiFlingEnabled then
            AntiFlingShieldModule.stop()
            antiFlingEnabled = false
        end
                        NS = 60
        CS = 29
        LAGGER_SPEED = 15
        LAGGER_CARRY_SPEED = 24.5
        CONFIG.STEAL_RANGE = 61
        speedMode = false
        laggerToggled = false
        laggerCarryToggled = false
        antiRagdollMode = "off"
        antiDieEnabled = false
        antiFlingEnabled = false
                batCounterEnabled = false
        autoBatEnabled = false
                antiLagEnabled = false
        uiLocked = true
        editModeEnabled = false
        CONFIG.AUTO_STEAL_ENABLED = false
        autoStealVariant = 1
        BAT_AIMBOT_SPEED = 58
        dropMode = 1
        stretchEnabled = false
        stretchFOV = 120
        fovValue = 70
        disableCustomFov()
        if fovSliderSet then fovSliderSet(70) end
        if setFovVisual then setFovVisual(false) end
        uiScaleValue = 78
        if mainUIScale then mainUIScale.Scale = 1 end
        progressBarScale = 1
        if pbScale then pbScale.Scale = progressBarScale end
        espEnabled = false
                        autoBatV2Enabled = false
        katanaSkinEnabled = false
        pcall(function() KatanaSkin.ApplySkin("NONE") end)
                            floatingButtonScale = 1
        if batDesyncTpEnabled then stopBatDesyncTp() end
        tpBatVersion = 1
        _G.__tpBatV2Distance = 8
        currentAnimPack = "Off"
        stopAnimPack()
        currentOutfitIndex = 1
        currentColorTheme = "Gris"
        selectedColor = COLOR_THEMES["Gris"]
        backgroundIndex = 1
        backgroundImageTransparency = 0.35
        applyBackground(1)
        for key, val in pairs(DEFAULT_KB) do
            if KB[key] then
                KB[key].kb = val.kb
                KB[key].gp = val.gp
            end
        end
        useCarrySystem = false
        CarrySystem:stop()
        CarrySystem.normalSpeed = NS
        CarrySystem.carrySpeed = CS
        CarrySystem.laggerSpeed = LAGGER_SPEED
        CarrySystem.laggerCarrySpeed = LAGGER_CARRY_SPEED
        CarrySystem.softStealSpeed = CarrySystem.carrySpeed
        CarrySystem.softStealRadius = 10
        CarrySystem.speedToggled = false
        CarrySystem.laggerMode = 0
        CarrySystem.softStealEnabled = false
        infJumpEnabled      = false
        infJumpMode         = "HOLD"
        antiKickEnabled     = false
        if isfile and isfile(CONFIG_FILE) then
            pcall(delfile, CONFIG_FILE)
        end
        resetFloatingPositions()
        forceResetUI()
        updateProgressBarVisibility()
        refreshSpeedModeLabel()
        _lastSavedJSON = nil
        saveAllSettings()
    end)
    _isResetting = false
    if not ok then warn("[resetToFactoryDefaults]", err) end
    return ok
end

function updateProgressBarVisibility()
    if pbFrame and pbFrame.Parent then
        pbFrame.Visible = CONFIG ~= nil and CONFIG.AUTO_STEAL_ENABLED == true
    end
end

function applyShimmerToText(obj, speed)
    speed = speed or 0.8
    local color = getThemeColor()
    local grad = Instance.new("UIGradient", obj)
    grad.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, color),
        ColorSequenceKeypoint.new(0.3, Color3.fromRGB(200,200,200)),
        ColorSequenceKeypoint.new(0.5, Color3.fromRGB(255,255,255)),
        ColorSequenceKeypoint.new(0.7, Color3.fromRGB(200,200,200)),
        ColorSequenceKeypoint.new(1, color),
    })
    grad.Rotation = 45
    grad.Offset = Vector2.new(0,0)
    task.spawn(function()
        local t = 0
        while grad and grad.Parent do
            t = t + 0.02
            grad.Offset = Vector2.new(math.sin(t * speed) * 0.4, 0)
            task.wait(0.04)
        end
    end)
    return grad
end

function getDefaultButtonPosition(btnName)
    local BTN_W, BTN_H = 60, 60
    local GAP = 8
    local orderMap = {
        DropBR = 0, AutoBat = 1, TpDown = 2, Carry = 3, Lagger1 = 4, Lagger2 = 5
    }
    local order = orderMap[btnName] or 0
    local row = _floor(order / 2)
    local col = order % 2
    return col * (BTN_W + GAP), row * (BTN_H + GAP + 10)
end

do

    if _G.InstaResetLoaded then
    else
        _G.InstaResetLoaded = true

        local resetCooldown          = false
        local resetThread            = nil
        local currentResetChar       = nil
        local resetSuccessful        = false
        local stopResetSequence      = false
        local cameraLocked           = false
        local lockedCameraCFrame     = nil
        local _lastInstaResetRequest = 0


        local function instaResetFast()
            if resetCooldown then return end
            resetCooldown     = true
            resetSuccessful   = false
            stopResetSequence = false
            cameraLocked      = false

            local character = LP.Character
            if not character then resetCooldown = false return end
            local humanoid = character:FindFirstChildOfClass("Humanoid")
            if not humanoid then resetCooldown = false return end

            local cam = workspace.CurrentCamera
            if cam then
                lockedCameraCFrame = cam.CFrame
                cameraLocked       = true
                cam.CFrame         = lockedCameraCFrame
            end

            currentResetChar = character
            local isRespawning = false

            resetThread = task.spawn(function()
                local attempts          = 0
                local maxAttempts       = 40
                local originalHipHeight = humanoid.HipHeight

                while character and character.Parent and humanoid and humanoid.Health > 0
                      and not isRespawning and not stopResetSequence do
                    if LP.Character ~= character then
                        isRespawning = true
                        break
                    end
                    pcall(function()
                        humanoid.HipHeight  = 1e30
                        humanoid.AutoRotate = true
                        local rootPart = character:FindFirstChild("HumanoidRootPart")
                        if rootPart then rootPart.CanCollide = false end
                        for _, part in ipairs(character:GetChildren()) do
                            if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
                                part.CanCollide = false
                            end
                        end
                    end)

                    if not character or not character.Parent or not humanoid
                       or humanoid.Health <= 0 or LP.Character ~= character then
                        resetSuccessful = true
                        break
                    end

                    attempts = attempts + 1
                    if attempts >= maxAttempts then break end
                    task.wait(0.05)
                end

                if not resetSuccessful then
                    if character and character.Parent and humanoid
                       and humanoid.Health > 0 and not isRespawning then
                        pcall(function() humanoid.Health = 0 end)
                        task.wait(0.1)
                        if not character.Parent or humanoid.Health <= 0 then
                            resetSuccessful = true
                        end
                    end
                end

                if not resetSuccessful and character and character.Parent and humanoid then
                    pcall(function()
                        humanoid.HipHeight = originalHipHeight
                        local rootPart = character:FindFirstChild("HumanoidRootPart")
                        if rootPart then rootPart.CanCollide = true end
                        for _, part in ipairs(character:GetChildren()) do
                            if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
                                part.CanCollide = true
                            end
                        end
                    end)
                end

                cameraLocked      = false
                resetCooldown     = false
                resetThread       = nil
                currentResetChar  = nil
                stopResetSequence = false
            end)
        end

        local function instaReset()
            local now = os.clock()
            if now - _lastInstaResetRequest < 0.75 then return end
            _lastInstaResetRequest = now
            instaResetFast()
        end


        _G.InstaResetCharConn = LP.CharacterAdded:Connect(function()
            stopResetSequence = true
            if resetThread then pcall(task.cancel, resetThread); resetThread = nil end
            resetCooldown    = false
            currentResetChar = nil
            cameraLocked     = false
        end)

        task.spawn(function()
            while not _G.BloodHoundsUnloaded do
                task.wait(0.016)
                local cam = workspace.CurrentCamera
                if cameraLocked and lockedCameraCFrame and cam then
                    cam.CFrame = lockedCameraCFrame
                end
            end
        end)


        _G.InstaReset = { Trigger = instaReset }
    end
end

function buildGui()
    local SILVER = Color3.fromRGB(180, 180, 190)
    local SILVER_DARK = Color3.fromRGB(170, 170, 180)
    local SILVER_LIGHT = Color3.fromRGB(220, 220, 230)
    local BG = Color3.fromRGB(0,0,0)
    local BG2 = Color3.fromRGB(10,10,10)
    local ROW_BG = Color3.fromRGB(10,10,10)
    local ROW_BORDER = Color3.fromRGB(50,50,50)
    local WHITE = Color3.fromRGB(255,255,255)
    local GRAY = Color3.fromRGB(60,60,60)
    local INP = Color3.fromRGB(15,15,15)
    local OFF = Color3.fromRGB(25,25,30)
    local TAB_ACTIVE = SILVER
    local TAB_INACT = SILVER_DARK
    local SECT_LBL = SILVER
    local HOV = Color3.fromRGB(25,25,25)
    local DOT_ON = SILVER
    local ON_COLOR = SILVER
    local STROKE_COLOR = Color3.fromRGB(50,50,50)
    local GUI_W, GUI_H = 300, 545

    local old = game:GetService("CoreGui"):FindFirstChild("russo")
    if old then old:Destroy() end
    local pg = LP:FindFirstChild("PlayerGui")
    if pg then local o = pg:FindFirstChild("russo"); if o then o:Destroy() end end

    gui = Instance.new("ScreenGui")
    gui.Name = "russo"
    gui.ResetOnSpawn = false
    gui.DisplayOrder = 10
    gui.IgnoreGuiInset = true
    pcall(function() if syn and syn.protect_gui then syn.protect_gui(gui) end end)
    local guiOk = pcall(function() gui.Parent = game:GetService("CoreGui") end)
    if not guiOk then gui.Parent = LP:WaitForChild("PlayerGui") end

    main = Instance.new("Frame", gui)
    main.Size = UDim2.new(0, GUI_W, 0, GUI_H)
    main.AnchorPoint = Vector2.new(1, 1)
    main.Position = UDim2.new(1, -18, 1, -18)
    main.BackgroundColor3 = Color3.fromRGB(5, 5, 8)
    main.BackgroundTransparency = 0
    main.BorderSizePixel = 0
    main.ClipsDescendants = true
    Instance.new("UICorner", main).CornerRadius = UDim.new(0, 18)

    backgroundImage = Instance.new("ImageLabel", main)
    backgroundImage.Name = "BackgroundImage"
    backgroundImage.Size = UDim2.new(1, 0, 1, 0)
    backgroundImage.Position = UDim2.new(0, 0, 0, 0)
    backgroundImage.BackgroundTransparency = 1
    backgroundImage.BorderSizePixel = 0
    backgroundImage.Image = ""
    backgroundImage.ScaleType = Enum.ScaleType.Stretch
    backgroundImage.ImageTransparency = 1
    backgroundImage.ZIndex = 1
    Instance.new("UICorner", backgroundImage).CornerRadius = UDim.new(0, 18)

    mainUIScale = Instance.new("UIScale", main)
    mainUIScale.Scale = uiScaleValue / 100

    local titleFrame = Instance.new("Frame", main)
    titleFrame.Name = "TitleFrame"
    titleFrame.Size = UDim2.new(1, -24, 0, 48)
    titleFrame.Position = UDim2.new(0, 12, 0, 6)
    titleFrame.BackgroundTransparency = 1
    titleFrame.ZIndex = 20

    local titleImage = Instance.new("ImageLabel", titleFrame)
    titleImage.Name = "TitleImage"
    titleImage.AnchorPoint = Vector2.new(0.5, 0.5)
    titleImage.Size = UDim2.new(1.20, 0, 1.20, 0)
    titleImage.Position = UDim2.new(0.5, 0, 0.5, 0)
    titleImage.BackgroundTransparency = 1
    titleImage.Image = ""
    titleImage.ScaleType = Enum.ScaleType.Crop
    titleImage.Visible = false
    titleImage.ZIndex = 21

    local russoTitle = Instance.new("TextLabel", titleFrame)
    russoTitle.Name = "RussoTitle"
    russoTitle.Size = UDim2.new(1, 0, 1, 0)
    russoTitle.BackgroundTransparency = 1
    russoTitle.Text = "russo"
    russoTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
    russoTitle.Font = Enum.Font.GothamBlack
    russoTitle.TextSize = 21
    russoTitle.ZIndex = 22

    local closeBtn = Instance.new("TextButton", main)
    closeBtn.Size = UDim2.new(0, 32, 0, 32)
    closeBtn.Position = UDim2.new(1, -42, 0, 8)
    closeBtn.BackgroundColor3 = Color3.fromRGB(30,30,35)
    closeBtn.BackgroundTransparency = 0.6
    closeBtn.BorderSizePixel = 0
    closeBtn.Text = "−"
    closeBtn.TextColor3 = WHITE
    closeBtn.Font = Enum.Font.GothamBold
    closeBtn.TextSize = 26
    closeBtn.AutoButtonColor = false
    closeBtn.ZIndex = 200
    Instance.new("UICorner", closeBtn).CornerRadius = UDim.new(0, 8)

    closeBtn.MouseEnter:Connect(function()
        TS:Create(closeBtn, TweenInfo.new(0.12), {TextColor3 = WHITE, BackgroundColor3 = getThemeColor()}):Play()
    end)
    closeBtn.MouseLeave:Connect(function()
        TS:Create(closeBtn, TweenInfo.new(0.12), {TextColor3 = WHITE, BackgroundColor3 = Color3.fromRGB(30,30,35)}):Play()
    end)

    miniBtn = Instance.new("TextButton", gui)
    miniBtn.Size = UDim2.new(0, 160, 0, 42)
    miniBtn.Position = UDim2.new(0, 16, 0, 58)
    miniBtn.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    miniBtn.BackgroundTransparency = 0
    miniBtn.BorderSizePixel = 0
    miniBtn.ClipsDescendants = true
    miniBtn.Text = ""
    miniBtn.TextColor3 = Color3.fromRGB(210, 210, 220)
    miniBtn.Font = Enum.Font.SciFi
    miniBtn.TextSize = 16
    miniBtn.ZIndex = 20
    miniBtn.Visible = false
    Instance.new("UICorner", miniBtn).CornerRadius = UDim.new(1, 0)

    local miniTitleImage = Instance.new("ImageLabel", miniBtn)
    miniTitleImage.Name = "MiniTitleImage"
    miniTitleImage.AnchorPoint = Vector2.new(0.5, 0.5)
    miniTitleImage.Size = UDim2.new(0.88, 0, 0.88, 0)
    miniTitleImage.Position = UDim2.new(0.5, 0, 0.5, 0)
    miniTitleImage.BackgroundTransparency = 1
    miniTitleImage.Image = ""
    miniTitleImage.ScaleType = Enum.ScaleType.Crop
    miniTitleImage.Visible = false
    miniTitleImage.ZIndex = 21
    miniBtn.Text = "russo"

    local slideTween = nil
    local mainOriginalPos = main.Position

    showGui = function()
        if slideTween then slideTween:Cancel() end
        if not main then return end
        main.Visible = true
        miniBtn.Visible = false
        main.Position = UDim2.new(0, -GUI_W - 20, 0, 2)
        slideTween = TS:Create(main, TweenInfo.new(0.4, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Position = mainOriginalPos})
        slideTween:Play()
        slideTween.Completed:Connect(function() slideTween = nil end)
    end

    hideGui = function()
        if slideTween then slideTween:Cancel() end
        if not main or not main.Visible then return end
        local targetPos = UDim2.new(0, -GUI_W - 20, 0, 2)
        slideTween = TS:Create(main, TweenInfo.new(0.4, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Position = targetPos})
        slideTween:Play()
        slideTween.Completed:Connect(function()
            main.Visible = false
            miniBtn.Visible = true
            slideTween = nil
        end)
    end

    closeBtn.MouseButton1Click:Connect(hideGui)
    miniBtn.MouseButton1Click:Connect(showGui)

    local tabBar = Instance.new("Frame", main)
    tabBar.Size = UDim2.new(1, -32, 0, 34)
    local tabsDivider = Instance.new("Frame", main)
    tabsDivider.Name = "TabsDivider"
    tabsDivider.Size = UDim2.new(1, -32, 0, 1)
    tabsDivider.Position = UDim2.new(0, 16, 0, 58)
    tabsDivider.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    tabsDivider.BackgroundTransparency = 0.25
    tabsDivider.BorderSizePixel = 0
    tabsDivider.ZIndex = 11

    tabBar.Position = UDim2.new(0, 16, 0, 64)
    tabBar.BackgroundTransparency = 1
    tabBar.ZIndex = 10

    local tabLayout = Instance.new("UIListLayout", tabBar)
    tabLayout.FillDirection = Enum.FillDirection.Horizontal
    tabLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
    tabLayout.VerticalAlignment = Enum.VerticalAlignment.Center
    tabLayout.Padding = UDim.new(0, 4)

    local tabContent = Instance.new("Frame", main)
    tabContent.Size = UDim2.new(1, -16, 1, -68)
    tabContent.Position = UDim2.new(0, 8, 0, 60)
    tabContent.BackgroundTransparency = 1
    tabContent.ClipsDescendants = true
    tabContent.ZIndex = 5

    local tabs = {"All"}
    tabButtons = {}
    local contentPages = {}

    for i, name in ipairs(tabs) do
        local btn = Instance.new("TextButton", tabBar)
        btn.Size = UDim2.new(0.155, 0, 1, -6)
        btn.BackgroundColor3 = Color3.fromRGB(18,18,22)
        btn.BackgroundTransparency = 0.35
        btn.BorderSizePixel = 0
        btn.Text = name
        btn.TextColor3 = TAB_INACT
        btn.Font = Enum.Font.GothamBold
        btn.TextSize = 12
        btn.AutoButtonColor = false
        btn.ZIndex = 11
        Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 8)
        local stroke = Instance.new("UIStroke", btn)
        stroke.Color = ROW_BORDER
        stroke.Thickness = 1

        local page = Instance.new("ScrollingFrame", tabContent)
        page.Size = UDim2.new(1, 0, 1, 0)
        page.Position = UDim2.new(0, 0, 0, 0)
        page.BackgroundColor3 = Color3.fromRGB(5,5,5)
        page.BackgroundTransparency = 0.6
        page.BorderSizePixel = 0
        page.ClipsDescendants = true
        page.ScrollBarThickness = 2
        page.ScrollBarImageColor3 = Color3.fromRGB(30,30,35)
        page.ScrollBarImageTransparency = 0.3
        page.CanvasSize = UDim2.new(0, 0, 0, 0)
        page.AutomaticCanvasSize = Enum.AutomaticSize.Y
        page.ScrollingDirection = Enum.ScrollingDirection.Y
        page.ZIndex = 6
        Instance.new("UICorner", page).CornerRadius = UDim.new(0, 16)
        page.Visible = (i == 1)

        local layout = Instance.new("UIListLayout", page)
        layout.SortOrder = Enum.SortOrder.LayoutOrder
        layout.Padding = UDim.new(0, 6)
        layout.HorizontalAlignment = Enum.HorizontalAlignment.Center

        local padding = Instance.new("UIPadding", page)
        padding.PaddingLeft = UDim.new(0, 8)
        padding.PaddingRight = UDim.new(0, 8)
        padding.PaddingTop = UDim.new(0, 6)
        padding.PaddingBottom = UDim.new(0, 20)

        contentPages[name] = page

        btn.MouseButton1Click:Connect(function()
            for _, pg in pairs(contentPages) do pg.Visible = false end
            page.Visible = true
            for _, b in ipairs(tabButtons) do
                b.TextColor3 = TAB_INACT
                b.BackgroundColor3 = Color3.fromRGB(18,18,22)
            end
            btn.TextColor3 = Color3.fromRGB(255, 255, 255)
            btn.BackgroundColor3 = Color3.fromRGB(30,30,35)
        end)

        table.insert(tabButtons, btn)
    end

    -- One continuous scroll page, with the requested categories in order.
    tabBar.Visible = false
    tabsDivider.Visible = false
    for _, category in ipairs({"Speed", "Combat", "Visual", "Utils", "Settings", "Keybinds", "Custom"}) do
        contentPages[category] = contentPages.All
    end

    if tabButtons[1] then
        tabButtons[1].TextColor3 = Color3.fromRGB(255, 255, 255)
        tabButtons[1].BackgroundColor3 = Color3.fromRGB(30,30,35)
    end

    local pageCounters = {}

    local function getNextOrder(page)
        if not pageCounters[page] then pageCounters[page] = 0 end
        pageCounters[page] = pageCounters[page] + 1
        return pageCounters[page]
    end

    local function mkSect(page, txt)
        local f = Instance.new("Frame", page)
        f.Size = UDim2.new(1, 0, 0, 26)
        f.BackgroundTransparency = 1
        f.BorderSizePixel = 0
        f.LayoutOrder = getNextOrder(page)
        f.ZIndex = 7
        local l = Instance.new("TextLabel", f)
        l.Size = UDim2.new(1, -16, 1, 0)
        l.Position = UDim2.new(0, 8, 0, 0)
        l.BackgroundTransparency = 1
        l.Text = txt:upper()
        l.TextColor3 = Color3.fromRGB(230, 230, 235)
        l.Font = Enum.Font.SciFi
        l.TextSize = 15
        l.TextXAlignment = Enum.TextXAlignment.Left
        l.TextStrokeColor3 = Color3.fromRGB(60,60,60)
        l.TextStrokeTransparency = 1
        l.ZIndex = 8
        local line = Instance.new("Frame", f)
        line.Size = UDim2.new(1, -24, 0, 1.5)
        line.Position = UDim2.new(0, 12, 1, -4)
        line.BackgroundColor3 = Color3.fromRGB(200, 200, 210)
        line.BackgroundTransparency = 0.6
        line.BorderSizePixel = 0
        line.ZIndex = 8
        return f
    end

    local function mkRow(page, h)
        local f = Instance.new("Frame", page)
        f.Size = UDim2.new(1, -4, 0, h or 38)
        f.BackgroundColor3 = ROW_BG
        f.BackgroundTransparency = 0.7
        f.BorderSizePixel = 0
        f.LayoutOrder = getNextOrder(page)
        f.ZIndex = 7
        Instance.new("UICorner", f).CornerRadius = UDim.new(0, 10)
        local rowStroke = Instance.new("UIStroke", f)
        rowStroke.Color = ROW_BORDER
        rowStroke.Thickness = 1
        rowStroke.Transparency = 0.5
        f.MouseEnter:Connect(function()
            TS:Create(f, TweenInfo.new(0.1), {BackgroundColor3 = Color3.fromRGB(28,28,28)}):Play()
        end)
        f.MouseLeave:Connect(function()
            TS:Create(f, TweenInfo.new(0.1), {BackgroundColor3 = ROW_BG}):Play()
        end)
        return f
    end

    local function mkLabel(row, txt)
        local l = Instance.new("TextLabel", row)
        l.Size = UDim2.new(0.55, 0, 1, 0)
        l.Position = UDim2.new(0, 10, 0, 0)
        l.BackgroundTransparency = 1
        l.Text = txt
        l.TextColor3 = Color3.fromRGB(235, 235, 240)
        l.Font = Enum.Font.GothamBold
        l.TextSize = 11
        l.TextXAlignment = Enum.TextXAlignment.Left
        l.TextTruncate = Enum.TextTruncate.AtEnd
        l.TextStrokeColor3 = Color3.fromRGB(0,0,0)
        l.TextStrokeTransparency = 1
        l.ZIndex = 8
        return l
    end

    local function mkPill(row, offset)
        local pill = Instance.new("Frame", row)
        pill.Name = "Track"
        pill.Size = UDim2.new(0, 34, 0, 18)
        pill.AnchorPoint = Vector2.new(0.5, 0.5)
        pill.Position = UDim2.new(1, -(offset or 48), 0.5, 0)
        pill.BackgroundColor3 = Color3.fromRGB(255,255,255)
        pill.BackgroundTransparency = 0.2
        pill.BorderSizePixel = 0
        pill.ZIndex = 8
        Instance.new("UICorner", pill).CornerRadius = UDim.new(0, 9)
        local stroke = Instance.new("UIStroke", pill)
        stroke.Color = ROW_BORDER
        stroke.Thickness = 1
        stroke.Transparency = 0.45
        stroke.Name = "PillStroke"

        local dot = Instance.new("Frame", pill)
        dot.Name = "Knob"
        dot.Size = UDim2.new(0, 13, 0, 13)
        dot.AnchorPoint = Vector2.new(0, 0)
        dot.Position = UDim2.new(0, 3, 0.5, -6)
        dot.BackgroundColor3 = Color3.fromRGB(18,18,22)
        dot.BorderSizePixel = 0
        dot.ZIndex = 9
        Instance.new("UICorner", dot).CornerRadius = UDim.new(1, 0)

        local shine = Instance.new("Frame", dot)
        shine.Name = "Shine"
        shine.Size = UDim2.new(1, -4, 0, 4)
        shine.Position = UDim2.new(0, 2, 0, 2)
        shine.BackgroundColor3 = WHITE
        shine.BackgroundTransparency = 0.72
        shine.BorderSizePixel = 0
        shine.ZIndex = 10
        Instance.new("UICorner", shine).CornerRadius = UDim.new(0, 4)

        return pill, dot
    end

    local function animPill(pill, dot, on)
        local stroke = pill:FindFirstChildOfClass("UIStroke")
        local c = getThemeColor()
        local info = TweenInfo.new(0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
        TS:Create(dot, info, {
            Position = on and UDim2.new(1, -16, 0.5, -6) or UDim2.new(0, 3, 0.5, -6),
            BackgroundColor3 = on and Color3.fromRGB(255,255,255) or Color3.fromRGB(18,18,22),
        }):Play()
        TS:Create(pill, info, {
            BackgroundColor3 = on and c or Color3.fromRGB(255,255,255),
            BackgroundTransparency = on and 0 or 0.2,
        }):Play()
        if stroke then
            TS:Create(stroke, info, {
                Color = on and c or ROW_BORDER,
                Transparency = on and 0.2 or 0.45,
                Thickness = on and 1.5 or 1,
            }):Play()
        end
    end

    local function mkSlider(row, minV, maxV, default, cb)
        local W = 118
        local track = Instance.new("Frame", row)
        track.Size = UDim2.new(0, W, 0, 4)
        track.Position = UDim2.new(1, -(W + 44), 0.5, -2)
        track.BackgroundColor3 = INP
        track.BackgroundTransparency = 0.3
        track.BorderSizePixel = 0
        track.ZIndex = 8
        Instance.new("UICorner", track).CornerRadius = UDim.new(1, 0)

        local fill = Instance.new("Frame", track)
        fill.Size = UDim2.new(0, 0, 1, 0)
        fill.BackgroundColor3 = getThemeColor()
        fill.BorderSizePixel = 0
        fill.ZIndex = 9
        Instance.new("UICorner", fill).CornerRadius = UDim.new(1, 0)

        local knob = Instance.new("Frame", track)
        knob.Size = UDim2.new(0, 13, 0, 13)
        knob.AnchorPoint = Vector2.new(0.5, 0.5)
        knob.Position = UDim2.new(0, 0, 0.5, 0)
        knob.BackgroundColor3 = WHITE
        knob.BorderSizePixel = 0
        knob.ZIndex = 11
        Instance.new("UICorner", knob).CornerRadius = UDim.new(1, 0)
        local kStroke = Instance.new("UIStroke", knob)
        kStroke.Color = getThemeColor()
        kStroke.Thickness = 2

        local valLabel = Instance.new("TextLabel", row)
        valLabel.Size = UDim2.new(0, 34, 0, 20)
        valLabel.Position = UDim2.new(1, -38, 0.5, -10)
        valLabel.BackgroundTransparency = 1
        valLabel.Text = tostring(default)
        valLabel.TextColor3 = WHITE
        valLabel.Font = Enum.Font.GothamBold
        valLabel.TextSize = 11
        valLabel.TextXAlignment = Enum.TextXAlignment.Right
        valLabel.ZIndex = 9

        local hit = Instance.new("TextButton", row)
        hit.Size = UDim2.new(0, W + 16, 0, 26)
        hit.Position = UDim2.new(1, -(W + 52), 0.5, -13)
        hit.BackgroundTransparency = 1
        hit.Text = ""
        hit.AutoButtonColor = false
        hit.ZIndex = 12

        local current = default

        local function render(a)
            a = _clamp(a, 0, 1)
            fill.Size = UDim2.new(a, 0, 1, 0)
            knob.Position = UDim2.new(a, 0, 0.5, 0)
            fill.BackgroundColor3 = getThemeColor()
            kStroke.Color = getThemeColor()
        end

        local function applyFromX(px)
            local left = track.AbsolutePosition.X
            local width = track.AbsoluteSize.X
            if width <= 0 then width = W end
            if left <= 0 then return end
            local a = (px - left) / width
            a = _clamp(a, 0, 1)
            local v = _floor(minV + (maxV - minV) * a + 0.5)
            current = v
            valLabel.Text = tostring(v)
            render(a)
            if cb then pcall(cb, v) end
        end

        local dragging = false

        hit.InputBegan:Connect(function(i)
            if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
                dragging = true
                _isDraggingButton = true
                applyFromX(i.Position.X)
            end
        end)

        hit.InputEnded:Connect(function(i)
            if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
                dragging = false
                _isDraggingButton = false
                if type(saveAllSettings) == "function" then pcall(saveAllSettings) end
            end
        end)

        UIS.InputChanged:Connect(function(i)
            if not dragging then return end
            if i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch then
                applyFromX(i.Position.X)
            end
        end)

        UIS.InputEnded:Connect(function(i)
            if not dragging then return end
            if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
                dragging = false
                _isDraggingButton = false
            end
        end)

        local function setValue(v)
            v = _clamp(tonumber(v) or minV, minV, maxV)
            current = v
            valLabel.Text = tostring(v)
            render((v - minV) / (maxV - minV))
        end

        setValue(default)
        return setValue
    end

    local function mkToggle(page, txt, cb)
        local row = mkRow(page, 38)
        mkLabel(row, txt)
        local pill, dot = mkPill(row, 48)
        local on = false
        local function sv(s) on = s; animPill(pill, dot, s) end
        local clk = Instance.new("TextButton", pill)
        clk.Size = UDim2.new(1,0,1,0)
        clk.BackgroundTransparency = 1
        clk.Text = ""
        clk.AutoButtonColor = false
        clk.ZIndex = 10
        clk.MouseButton1Click:Connect(function()
            if editModeEnabled and not uiLocked then
                pcall(cb, not on)
            else
                on = not on
                sv(on)
                local ok, err = pcall(cb, on)
                if not ok then warn("[russo] Toggle callback failed (" .. tostring(txt) .. "):", err) end
                if type(saveAllSettings) == "function" then pcall(saveAllSettings) end
            end
        end)
        return sv
    end

    local function mkBox(parent, default, w, xOff, cb)
        local tb = Instance.new("TextBox", parent)
        local bw = w or 50
        local xo = math.max(xOff or 56, bw + 12)
        tb.Size = UDim2.new(0, bw, 0, 24)
        tb.Position = UDim2.new(1, -xo, 0.5, -12)
        tb.BackgroundColor3 = INP
        tb.BackgroundTransparency = 0.7
        tb.BorderSizePixel = 0
        tb.Text = tostring(default)
        tb.TextColor3 = WHITE
        tb.Font = Enum.Font.GothamBold
        tb.TextSize = 11
        tb.ClearTextOnFocus = false
        tb.ZIndex = 8
        Instance.new("UICorner", tb).CornerRadius = UDim.new(0, 6)
        local bs = Instance.new("UIStroke", tb)
        bs.Color = ROW_BORDER
        bs.Thickness = 1.2
        bs.Transparency = 0.25
        tb.Focused:Connect(function() TS:Create(bs, TweenInfo.new(0.12), {Color = getThemeColor(), Transparency = 0}):Play() end)
        tb.FocusLost:Connect(function()
            TS:Create(bs, TweenInfo.new(0.12), {Color = ROW_BORDER, Transparency = 0.25}):Play()
            if cb then
                local n = tonumber(tb.Text)
                if n then
                    local ok, err = pcall(cb, n)
                    if not ok then warn("[russo] Numeric setting callback failed:", err) end
                    if type(saveAllSettings) == "function" then pcall(saveAllSettings) end
                else
                    tb.Text = tostring(default)
                end
            end
        end)
        return tb
    end

    local function mkKeyButton(parent, kbEntry)
        local btn = Instance.new("TextButton", parent)
        btn.Size = UDim2.new(0, 80, 0, 24)
        btn.Position = UDim2.new(1, -88, 0.5, -12)
        btn.BackgroundColor3 = INP
        btn.BackgroundTransparency = 0.5
        btn.BorderSizePixel = 0
        local function getLabel() return (kbEntry.gp and kbEntry.gp.Name) or (kbEntry.kb and kbEntry.kb.Name) or "None" end
        btn.Text = getLabel()
        btn.TextColor3 = WHITE
        btn.Font = Enum.Font.GothamBold
        btn.TextSize = 9
        btn.ZIndex = 8
        btn.AutoButtonColor = false
        Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
        local bs = Instance.new("UIStroke", btn)
        bs.Color = ROW_BORDER
        bs.Thickness = 1
        local li = false; local lc; local pv = btn.Text; local listenStart = 0
        btn.Activated:Connect(function()
            if li then li = false; _anyKeyListening = false; if lc then lc:Disconnect(); lc = nil end; btn.Text = pv; btn.TextColor3 = WHITE; return end
            pv = btn.Text; li = true; _anyKeyListening = true; listenStart = _tick(); btn.Text = "..."; btn.TextColor3 = WHITE
            lc = UIS.InputBegan:Connect(function(inp)
                if not li then return end
                if inp.KeyCode == Enum.KeyCode.Escape then li = false; _anyKeyListening = false; if lc then lc:Disconnect(); lc = nil end; btn.Text = pv; btn.TextColor3 = WHITE; return end
                local isGp = isGamepadInput(inp)
                if isGp and _tick()-listenStart < 0.15 then return end
                if not isBindableInput(inp) then return end
                btn.Text = inp.KeyCode.Name; pv = inp.KeyCode.Name; btn.TextColor3 = WHITE
                li = false; _anyKeyListening = false; if lc then lc:Disconnect(); lc = nil end
                if isGp then kbEntry.gp = inp.KeyCode; kbEntry.kb = nil else kbEntry.kb = inp.KeyCode; kbEntry.gp = nil end
                if type(saveAllSettings) == "function" then pcall(saveAllSettings) end
            end)
        end)
        table.insert(keyButtonRefs, {btn = btn, entry = kbEntry})
        return btn
    end

    local function mkNeoSelector(parent, defaultText, options, cb)
        local container = Instance.new("Frame", parent)
        container.Size = UDim2.new(0, 175, 0, 30)
        container.Position = UDim2.new(1, -183, 0.5, -15)
        container.BackgroundColor3 = Color3.fromRGB(18,18,22)
        container.BackgroundTransparency = 0.35
        container.BorderSizePixel = 0
        container.ZIndex = 8
        Instance.new("UICorner", container).CornerRadius = UDim.new(1, 0)

        local contStroke = Instance.new("UIStroke", container)
        contStroke.Color = ROW_BORDER
        contStroke.Thickness = 1
        contStroke.Transparency = 0.35

        local glow = Instance.new("Frame", container)
        glow.Size = UDim2.new(0, 3, 0.55, 0)
        glow.Position = UDim2.new(0, 6, 0.225, 0)
        glow.BackgroundColor3 = Color3.fromRGB(220, 220, 230)
        glow.BorderSizePixel = 0
        glow.ZIndex = 9
        Instance.new("UICorner", glow).CornerRadius = UDim.new(1, 0)

        local label = Instance.new("TextLabel", container)
        label.Size = UDim2.new(1, -80, 1, 0)
        label.Position = UDim2.new(0, 14, 0, 0)
        label.BackgroundTransparency = 1
        label.Text = tostring(defaultText)
        label.TextColor3 = Color3.fromRGB(255, 255, 255)
        label.Font = Enum.Font.GothamBold
        label.TextSize = 12
        label.TextXAlignment = Enum.TextXAlignment.Left
        label.TextTruncate = Enum.TextTruncate.AtEnd
        label.ZIndex = 9

        local dotsRow = Instance.new("Frame", container)
        dotsRow.Size = UDim2.new(0, 70, 0, 6)
        dotsRow.Position = UDim2.new(1, -76, 0.5, -3)
        dotsRow.BackgroundTransparency = 1
        dotsRow.ZIndex = 9

        local dotsLayout = Instance.new("UIListLayout", dotsRow)
        dotsLayout.FillDirection = Enum.FillDirection.Horizontal
        dotsLayout.HorizontalAlignment = Enum.HorizontalAlignment.Right
        dotsLayout.VerticalAlignment = Enum.VerticalAlignment.Center
        dotsLayout.Padding = UDim.new(0, 3)

        local MAX_DOTS = 8
        local dots = {}
        local dotCount = math.min(#options, MAX_DOTS)
        for i = 1, dotCount do
            local d = Instance.new("Frame", dotsRow)
            d.Size = UDim2.new(0, 5, 0, 5)
            d.BackgroundColor3 = Color3.fromRGB(150, 150, 160)
            d.BorderSizePixel = 0
            d.ZIndex = 10
            Instance.new("UICorner", d).CornerRadius = UDim.new(1, 0)
            dots[i] = d
        end

        local currentIdx = 1
        for i, opt in ipairs(options) do
            if tostring(opt) == tostring(defaultText) then currentIdx = i; break end
        end

        local leftZone = Instance.new("TextButton", container)
        leftZone.Size = UDim2.new(0.5, 0, 1, 0)
        leftZone.Position = UDim2.new(0, 0, 0, 0)
        leftZone.BackgroundTransparency = 1
        leftZone.Text = ""
        leftZone.AutoButtonColor = false
        leftZone.ZIndex = 11

        local rightZone = Instance.new("TextButton", container)
        rightZone.Size = UDim2.new(0.5, 0, 1, 0)
        rightZone.Position = UDim2.new(0.5, 0, 0, 0)
        rightZone.BackgroundTransparency = 1
        rightZone.Text = ""
        rightZone.AutoButtonColor = false
        rightZone.ZIndex = 11

        local function refreshDotsSilent()
            for i, d in ipairs(dots) do
                local active = (i == currentIdx)
                d.BackgroundColor3 = active and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(150, 150, 160)
                d.Size = active and UDim2.new(0, 10, 0, 5) or UDim2.new(0, 5, 0, 5)
            end
        end

        local function refreshDotsAnimated()
            for i, d in ipairs(dots) do
                local active = (i == currentIdx)
                TS:Create(d, TweenInfo.new(0.22, Enum.EasingStyle.Quad), {
                    BackgroundColor3 = active and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(150, 150, 160),
                    Size = active and UDim2.new(0, 10, 0, 5) or UDim2.new(0, 5, 0, 5),
                }):Play()
            end
        end

        local function setTextAnimated(newText)
            local tweenOut = TS:Create(label, TweenInfo.new(0.10), {
                TextTransparency = 1,
                Position = UDim2.new(0, 22, 0, 0),
            })
            tweenOut:Play()
            tweenOut.Completed:Connect(function()
                label.Text = newText
                label.Position = UDim2.new(0, 4, 0, 0)
                TS:Create(label, TweenInfo.new(0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
                    TextTransparency = 0,
                    Position = UDim2.new(0, 14, 0, 0),
                }):Play()
            end)
        end

        local function step(dir)
            currentIdx = currentIdx + dir
            if currentIdx < 1 then currentIdx = #options end
            if currentIdx > #options then currentIdx = 1 end
            setTextAnimated(tostring(options[currentIdx]))
            refreshDotsAnimated()
            if cb then
                local ok, err = pcall(cb, options[currentIdx], currentIdx)
                if not ok then warn("[russo] Selector callback failed:", err) end
            end
            if type(saveAllSettings) == "function" then pcall(saveAllSettings) end
        end

        leftZone.MouseButton1Click:Connect(function() step(-1) end)
        rightZone.MouseButton1Click:Connect(function() step(1) end)

        container.MouseEnter:Connect(function()
            TS:Create(contStroke, TweenInfo.new(0.12), {Color = Color3.fromRGB(220, 220, 230), Transparency = 0}):Play()
            TS:Create(glow, TweenInfo.new(0.12), {Size = UDim2.new(0, 4, 0.7, 0)}):Play()
        end)
        container.MouseLeave:Connect(function()
            TS:Create(contStroke, TweenInfo.new(0.12), {Color = ROW_BORDER, Transparency = 0.35}):Play()
            TS:Create(glow, TweenInfo.new(0.12), {Size = UDim2.new(0, 3, 0.55, 0)}):Play()
        end)

        refreshDotsSilent()

        local proxy = {}
        setmetatable(proxy, {
            __index = function(_, k) return label[k] end,
            __newindex = function(_, k, v)
                if k == "Text" then
                    local newText = tostring(v)
                    for i, opt in ipairs(options) do
                        if tostring(opt) == newText then
                            currentIdx = i
                            refreshDotsSilent()
                            break
                        end
                    end
                    label.Text = newText
                else
                    label[k] = v
                end
            end,
        })
        return proxy
    end

    local function addKeybindRow(page, labelText, kbEntry)
        local row = mkRow(page, 36)
        mkLabel(row, labelText)
        mkKeyButton(row, kbEntry)
    end

    local speedPage = contentPages["Speed"]

    mkSect(speedPage, "Base Speeds")
    do local row = mkRow(speedPage, 38); mkLabel(row, "Normal Speed");    normalBox  = mkBox(row, NS, 50, 56, function(v) if v > 0 and v <= 500 then NS = v; CarrySystem:setNormalSpeed(v) end end) end
    do local row = mkRow(speedPage, 38); mkLabel(row, "Carry Speed");     carryBox   = mkBox(row, CS, 50, 56, function(v) if v > 0 and v <= 500 then CS = v; CarrySystem:setCarrySpeed(v) end end) end
    do local row = mkRow(speedPage, 38); mkLabel(row, "Lagger Normal");   laggerBox  = mkBox(row, LAGGER_SPEED, 50, 56, function(v) if v > 0 and v <= 500 then LAGGER_SPEED = v end end) end
    do local row = mkRow(speedPage, 38); mkLabel(row, "Lagger Carry");    lagger2Box = mkBox(row, LAGGER_CARRY_SPEED, 50, 56, function(v) if v > 0 and v <= 500 then LAGGER_CARRY_SPEED = v end end) end

    do
        local row = mkRow(speedPage, 38)
        mkLabel(row, "Current Mode")
        local initModeText = "Normal"
        if laggerCarryToggled then initModeText = "Lagger Carry"
        elseif laggerToggled then initModeText = "Lagger"
        elseif speedMode then initModeText = "Carry" end
        modeValLbl = mkNeoSelector(row, initModeText, {"Normal", "Carry", "Lagger", "Lagger Carry"}, function(selected)
            speedMode = false
            laggerToggled = false
            laggerCarryToggled = false
            if selected == "Carry" then
                speedMode = true
            elseif selected == "Lagger" then
                laggerToggled = true
            elseif selected == "Lagger Carry" then
                laggerCarryToggled = true
            end
            resetMovementState()
            if mobSetCarry then mobSetCarry(speedMode) end
            if mobSetLagger1 then mobSetLagger1(laggerToggled) end
            if mobSetLagger2 then mobSetLagger2(laggerCarryToggled) end
            if useCarrySystem then
                if laggerCarryToggled then
                    CarrySystem:setLaggerMode(2)
                elseif laggerToggled then
                    CarrySystem:setLaggerMode(1)
                else
                    CarrySystem:setLaggerMode(0)
                end
                CarrySystem.speedToggled = speedMode
            end
        end)
    end

    mkSect(speedPage, "Auto Carry System")
    carrySystemToggleSetter = mkToggle(speedPage, "Auto Carry", function(on)
        useCarrySystem = on
        if on then
            CarrySystem:start()
            CarrySystem.autoCarryEnabled = true
            CarrySystem.speedToggled = false
            if laggerCarryToggled then
                CarrySystem:setLaggerMode(2)
            elseif laggerToggled then
                CarrySystem:setLaggerMode(1)
            else
                CarrySystem:setLaggerMode(0)
            end
            CarrySystem:setSoftStealEnabled(false)
        else
            CarrySystem.autoCarryEnabled = false
            CarrySystem:stop()
            CarrySystem:setSoftStealEnabled(false)
        end
        saveAllSettings()
    end)

    do local row = mkRow(speedPage, 38); mkLabel(row, "Normal Speed");       carrySysNormalBox          = mkBox(row, CarrySystem.normalSpeed, 50, 56, function(v) if v > 0 and v <= 500 then CarrySystem:setNormalSpeed(v); saveAllSettings() end end) end
    do local row = mkRow(speedPage, 38); mkLabel(row, "Carry Speed");        carrySysCarryBox           = mkBox(row, CarrySystem.carrySpeed, 50, 56, function(v) if v > 0 and v <= 500 then CarrySystem:setCarrySpeed(v); saveAllSettings() end end) end
    do local row = mkRow(speedPage, 38); mkLabel(row, "Lagger Speed");       carrySysLaggerBox          = mkBox(row, CarrySystem.laggerSpeed, 50, 56, function(v) if v > 0 and v <= 500 then CarrySystem:setLaggerSpeed(v); saveAllSettings() end end) end
    do local row = mkRow(speedPage, 38); mkLabel(row, "Lagger Carry Spd");   carrySysLaggerCarryBox     = mkBox(row, CarrySystem.laggerCarrySpeed, 50, 56, function(v) if v > 0 and v <= 500 then CarrySystem:setLaggerCarrySpeed(v); saveAllSettings() end end) end
    do local row = mkRow(speedPage, 38); mkLabel(row, "Soft Steal Radius");  carrySysSoftStealRadiusBox = mkBox(row, CarrySystem.softStealRadius, 50, 56, function(v) if v > 0 then CarrySystem:setSoftStealRadius(v); saveAllSettings() end end) end

    mkSect(speedPage, "Teleport")
    do
        local row = mkRow(speedPage, 38)
        mkLabel(row, "TP Down")
        local clk = Instance.new("TextButton", row)
        clk.Size = UDim2.new(0.58, 0, 1, 0)
        clk.BackgroundTransparency = 1
        clk.Text = ""
        clk.AutoButtonColor = false
        clk.ZIndex = 8
        clk.MouseButton1Click:Connect(function() doTpDown() end)
        local actLbl = Instance.new("TextLabel", row)
        actLbl.Size = UDim2.new(0, 70, 1, 0)
        actLbl.Position = UDim2.new(1, -78, 0, 0)
        actLbl.BackgroundTransparency = 1
        actLbl.Text = "ACTIVATE"
        actLbl.TextColor3 = Color3.fromRGB(230, 230, 235)
        actLbl.Font = Enum.Font.GothamBold
        actLbl.TextSize = 9
        actLbl.TextXAlignment = Enum.TextXAlignment.Right
        actLbl.ZIndex = 8
    end
    infJumpSetVisual = mkToggle(speedPage, "Hold Jump", function(on)
        -- Keep the persisted flag synchronized with the controller state.
        infJumpEnabled = (on == true)
        if _G.setInfJumpInternal then
            _G.setInfJumpInternal(infJumpEnabled)
        end
        local ok, saved = pcall(saveAllSettings)
        if not ok or saved ~= true then
            warn("[russo] Failed to save Hold Jump setting:", saved)
        end
    end)
    do
        local row = mkRow(speedPage, 38)
        mkLabel(row, "Jump Mode")
        infJumpModeSetVisual = mkNeoSelector(row, infJumpMode, {"HOLD", "MANUAL"}, function(selected)
            if selected ~= "MANUAL" then selected = "HOLD" end
            infJumpMode = selected
            if infJumpMode == "MANUAL" then
                _G.AmbitiousStopNormalInfJumpHoldState()
            end
            saveAllSettings()
        end)
    end

    local combatPage = contentPages["Combat"]

    mkSect(combatPage, "Aimbots")
    autoBatSetVisual = mkToggle(combatPage, "Auto Bat", function(on)
        if on then enableAutoBat() else disableAutoBat() end
        if mobSetAutoBat then mobSetAutoBat(on) end
    end)
    do local row = mkRow(combatPage, 38); mkLabel(row, "Bat Aimbot Speed"); batSpeedBox = mkBox(row, BAT_AIMBOT_SPEED, 50, 56, function(v) if v > 0 and v <= 200 then BAT_AIMBOT_SPEED = v end end) end

-- TP BAT control removed from the visible panel.
    autoBatV2SetVisual = mkToggle(combatPage, "Bat Bypass", function(on)
        if on then enableBatV2() else disableBatV2() end
    end)
    if autoBatV2SetVisual then autoBatV2SetVisual(autoBatV2Enabled) end
    _G.AceAimbotSetVisual = autoBatV2SetVisual

    mkSect(combatPage, "Counters")
    setBatCounterVisual = mkToggle(combatPage, "Bat Counter", function(on)
        batCounterEnabled = on
        if on then startBatCounter() else stopBatCounter() end
    end)

    setSafeModeVisual = mkToggle(combatPage, "Safe Mode", function(on)
        antiKickEnabled = on
        if on and _G.AmbitiousSafeModeForceStop then
            _G.AmbitiousSafeModeForceStop("SAFE MODE")
        end
        saveAllSettings()
    end)

    mkSect(combatPage, "Survival")
    do
        local row = mkRow(combatPage, 38)
        mkLabel(row, "Anti Ragdoll")

        local initialLabel = "Off"
        if antiRagdollMode == "v1" then initialLabel = "V1"
        elseif antiRagdollMode == "v2" then initialLabel = "V2" end

        local antiRagdollSelector = mkNeoSelector(row, initialLabel, {"Off", "V1", "V2"}, function(selected)
            local mode = selected:lower()
            setAntiRagdollMode(mode)
        end)

        _G.updateAntiRagdollUI = function(mode)
            local label = "Off"
            if mode == "v1" then label = "V1"
            elseif mode == "v2" then label = "V2" end
            if antiRagdollSelector then antiRagdollSelector.Text = label end
        end

        _G.updateAntiRagdollUI(antiRagdollMode)
    end

    setAntiDieVisual = mkToggle(combatPage, "Anti Die", function(on)
        antiDieEnabled = on
        if on then AntiDieModule.start() else AntiDieModule.stop() end
        saveAllSettings()
    end)
    if setAntiDieVisual then setAntiDieVisual(antiDieEnabled) end

    mkSect(combatPage, "Drop")
    dropBrainrotSetVisual = mkToggle(combatPage, "Drop Brainrot", function(on)
        if on then
            executeDropWithToggle(function(v)
                dropBrainrotSetVisual(v)
                if mobSetDropBR then mobSetDropBR(v) end
            end)
        end
    end)
    setDropVisual = dropBrainrotSetVisual

    do
        local row = mkRow(combatPage, 38)
        mkLabel(row, "Drop Mode")
        dropModeBtnRef = mkNeoSelector(row, dropMode == 1 and "Fling" or "Jump Drop", {"Fling", "Jump Drop"}, function(sel)
            if dropActive then stopDropBrainrot() end
            dropMode = (sel == "Fling") and 1 or 2
        end)
    end

    local visualPage = contentPages["Visual"]

    mkSect(visualPage, "Interface")
    setEditModeVisual = mkToggle(visualPage, "Edit Button", function(on)
        toggleEditMode(on)
        if on and uiLocked then
            editModeEnabled = false
            setEditModeVisual(false)
        end
    end)
    if setEditModeVisual then setEditModeVisual(editModeEnabled) end

    setLockUIVisual = mkToggle(visualPage, "Lock UI", function(on)
        toggleLockUI(on)
        if on and editModeEnabled then
            editModeEnabled = false
            if setEditModeVisual then setEditModeVisual(false) end
        end
    end)

    setESPVIsual = mkToggle(visualPage, "Player ESP", function(on) toggleESP(on) end)

    mkSect(visualPage, "Camera")
    setFovVisual = mkToggle(visualPage, "FOV", function(on)
        if on then enableCustomFov() else disableCustomFov() end
    end)
    if setFovVisual then setFovVisual(fovEnabled) end

    do
        local row = mkRow(visualPage, 38)
        mkLabel(row, "FOV Value")
        fovSliderSet = mkSlider(row, 20, 120, fovValue, function(v)
            fovValue = v
            if not fovEnabled then
                enableCustomFov()
                if setFovVisual then setFovVisual(true) end
            end
            local cam = workspace.CurrentCamera
            if cam then pcall(function() cam.FieldOfView = fovValue end) end
            if type(saveAllSettings) == "function" then pcall(saveAllSettings) end
        end)
    end

    do
        local row = mkRow(visualPage, 38)
        mkLabel(row, "Stretch Rez")
        local stretchPill, stretchDot = mkPill(row, 48)
        local stretchOn = false
        local function setStretch(s)
            stretchOn = s
            animPill(stretchPill, stretchDot, s)
            if s then enableStretch() else disableStretch() end
            stretchEnabled = s
            if type(saveAllSettings) == "function" then pcall(saveAllSettings) end
        end
        local stretchClk = Instance.new("TextButton", stretchPill)
        stretchClk.Size = UDim2.new(1,0,1,0)
        stretchClk.BackgroundTransparency = 1
        stretchClk.Text = ""
        stretchClk.AutoButtonColor = false
        stretchClk.ZIndex = 10
        stretchClk.MouseButton1Click:Connect(function() setStretch(not stretchOn) end)
        _G.stretchToggleSetter = setStretch
    end

    mkSect(visualPage, "Performance")
    setAntiLagVisual = mkToggle(visualPage, "Anti Lag", function(on)
        if on then enableAntiLag() else disableAntiLag() end
    end)
    CleanHubShowE01WarningSetVisual = mkToggle(visualPage, "Show E01 Warning", function(on)
        CleanHubShowE01Warning = on
        saveAllSettings()
    end)

    local utilsPage = contentPages["Utils"]
    mkSect(utilsPage, "Utils")

    local katanaSkinSetVisual = mkToggle(utilsPage, "Bat Skin", function(on)
        katanaSkinEnabled = on
        if on then
            pcall(function() KatanaSkin.ApplySkin("KATANA") end)
        else
            pcall(function() KatanaSkin.ApplySkin("NONE") end)
        end
        saveAllSettings()
    end)
    if katanaSkinSetVisual then katanaSkinSetVisual(katanaSkinEnabled) end

    do
        local row = mkRow(utilsPage, 38)
        mkLabel(row, "Anim Pack")
        local options = {}
        for _, entry in ipairs(ANIM_PACK_ORDER) do
            table.insert(options, entry[2])
        end
        animSelectorLabel = mkNeoSelector(row, currentAnimPack, options, function(selected)
            if selected == "Off" then
                stopAnimPack()
            else
                startAnimPack(selected)
            end
        end)
    end

    do
        local row = mkRow(utilsPage, 38)
        mkLabel(row, "Outfit")
        local options = {}
        for _, o in ipairs(OUTFITS) do table.insert(options, o.label) end
        outfitSelectorLabel = mkNeoSelector(row, OUTFITS[currentOutfitIndex].label, options, function(selected)
            for i, o in ipairs(OUTFITS) do
                if o.label == selected then
                    currentOutfitIndex = i
                    pcall(function() applyOutfitByIndex(currentOutfitIndex) end)
                    saveAllSettings()
                    break
                end
            end
        end)
    end

    do
        local row = mkRow(utilsPage, 56)
        mkLabel(row, "Background")

        local previewFrame = Instance.new("Frame", row)
        previewFrame.Name = "BackgroundPreview"
        previewFrame.Size = UDim2.new(0, 56, 0, 44)
        previewFrame.Position = UDim2.new(1, -250, 0.5, -22)
        previewFrame.BackgroundColor3 = Color3.fromRGB(10, 10, 12)
        previewFrame.BackgroundTransparency = 0.15
        previewFrame.BorderSizePixel = 0
        previewFrame.ClipsDescendants = true
        previewFrame.ZIndex = 8
        Instance.new("UICorner", previewFrame).CornerRadius = UDim.new(0, 8)

        local previewStroke = Instance.new("UIStroke", previewFrame)
        previewStroke.Color = ROW_BORDER
        previewStroke.Thickness = 1.2
        previewStroke.Transparency = 0.35

        local previewImage = Instance.new("ImageLabel", previewFrame)
        previewImage.Name = "PreviewImage"
        previewImage.Size = UDim2.new(1, 0, 1, 0)
        previewImage.Position = UDim2.new(0, 0, 0, 0)
        previewImage.BackgroundTransparency = 1
        previewImage.Image = ""
        previewImage.ScaleType = Enum.ScaleType.Crop
        previewImage.ZIndex = 9
        Instance.new("UICorner", previewImage).CornerRadius = UDim.new(0, 8)

        local previewPlaceholder = Instance.new("TextLabel", previewFrame)
        previewPlaceholder.Name = "PreviewPlaceholder"
        previewPlaceholder.Size = UDim2.new(1, 0, 1, 0)
        previewPlaceholder.BackgroundTransparency = 1
        previewPlaceholder.Text = "OFF"
        previewPlaceholder.TextColor3 = Color3.fromRGB(160, 160, 170)
        previewPlaceholder.Font = Enum.Font.GothamBlack
        previewPlaceholder.TextSize = 12
        previewPlaceholder.ZIndex = 10

        local function updateBackgroundPreview(idx)
            local cfg = BACKGROUND_IMAGES[idx]
            if not cfg or not cfg.id or cfg.id == "" then
                previewImage.Image = ""
                previewPlaceholder.Visible = true
                previewStroke.Color = ROW_BORDER
            else
                previewImage.Image = cfg.id
                previewPlaceholder.Visible = false
                previewStroke.Color = getThemeColor()
            end
        end

        local function animatePreview()
            previewFrame.Size = UDim2.new(0, 56, 0, 44)
            TS:Create(previewFrame, TweenInfo.new(0.15, Enum.EasingStyle.Back), {
                Size = UDim2.new(0, 60, 0, 47),
            }):Play()
            task.delay(0.15, function()
                TS:Create(previewFrame, TweenInfo.new(0.15), {
                    Size = UDim2.new(0, 56, 0, 44),
                }):Play()
            end)
        end

        previewFrame.MouseEnter:Connect(function()
            TS:Create(previewStroke, TweenInfo.new(0.12), {
                Color = getThemeColor(),
                Transparency = 0.05,
            }):Play()
        end)
        previewFrame.MouseLeave:Connect(function()
            local cfg = BACKGROUND_IMAGES[backgroundIndex]
            local isOff = (not cfg) or (not cfg.id) or (cfg.id == "")
            TS:Create(previewStroke, TweenInfo.new(0.12), {
                Color = isOff and ROW_BORDER or getThemeColor(),
                Transparency = isOff and 0.35 or 0.2,
            }):Play()
        end)

        local options = {}
        for _, bg in ipairs(BACKGROUND_IMAGES) do
            table.insert(options, bg.label)
        end

        backgroundSelectorLabel = mkNeoSelector(row, BACKGROUND_IMAGES[backgroundIndex].label, options, function(selected)
            for i, bg in ipairs(BACKGROUND_IMAGES) do
                if bg.label == selected then
                    applyBackground(i)
                    updateBackgroundPreview(i)
                    animatePreview()
                    saveAllSettings()
                    break
                end
            end
        end)

        if backgroundSelectorLabel then
            local selContainer = backgroundSelectorLabel.Parent
            if selContainer and selContainer:IsA("Frame") then
                selContainer.Size = UDim2.new(0, 175, 0, 30)
                selContainer.Position = UDim2.new(1, -183, 0.5, -15)
            end
        end

        updateBackgroundPreview(backgroundIndex)
    end

    local configPage = contentPages["Settings"]

    mkSect(configPage, "Auto Steal")
    setInstaGrab = mkToggle(configPage, "Auto Steal", function(on)
        CONFIG.AUTO_STEAL_ENABLED = on
        if on then pcall(startAutoSteal) else stopAutoSteal() end
        updateProgressBarVisibility()
    end)

    do
        local row = mkRow(configPage, 38)
        mkLabel(row, "Steal Radius")
        radInput = mkBox(row, CONFIG.STEAL_RANGE, 50, 56, function(v)
            if v and v >= 5 and v <= 300 then
                CONFIG.STEAL_RANGE = _floor(v+0.5)
                Steal.StealRadius = CONFIG.STEAL_RANGE
                radInput.Text = tostring(CONFIG.STEAL_RANGE)
                saveAllSettings()
            end
        end)
    end

    do
        local row = mkRow(configPage, 38)
        mkLabel(row, "Steal Version")
        autoStealVariantLabel = mkNeoSelector(row, autoStealVariantName(autoStealVariant), AUTO_STEAL_VARIANT_NAMES, function(selected)
            autoStealVariant = autoStealVariantFromName(selected)
            saveAllSettings()
        end)
    end

    mkSect(configPage, "UI Settings")
    do
        local row = mkRow(configPage, 38)
        mkLabel(row, "UI Scale")
        uiScaleBox = mkBox(row, uiScaleValue, 50, 56, function(v)
            local n = _clamp(_floor(v+0.5), 50, 150)
            uiScaleValue = n
            if mainUIScale then mainUIScale.Scale = n/100 end
            saveAllSettings()
        end)
    end

    do
        local row = mkRow(configPage, 38)
        mkLabel(row, "Float Scale")
        local box = mkBox(row, _floor(floatingButtonScale * 100), 50, 56, function(v)
            local val = _clamp(v, 50, 200)
            floatingButtonScale = val / 100
            applyFloatingButtonScale()
            saveAllSettings()
        end)
    end

    do
        local row = mkRow(configPage, 38)
        mkLabel(row, "Steal Bar Scale")
        local box = mkBox(row, _floor(progressBarScale * 100), 50, 56, function(v)
            local val = _clamp(v, 50, 200)
            progressBarScale = val / 100
            if pbScale then pbScale.Scale = progressBarScale end
            saveAllSettings()
        end)
    end

    mkSect(configPage, "UI Position")

    do
        local row = mkRow(configPage, 44)
        row.Size = UDim2.new(1, 0, 0, 44)
        local resetPosBtn = Instance.new("TextButton", row)
        resetPosBtn.Size = UDim2.new(1, -20, 0.82, 0)
        resetPosBtn.Position = UDim2.new(0, 10, 0.09, 0)
        resetPosBtn.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        resetPosBtn.BorderSizePixel = 0
        resetPosBtn.Text = ""
        resetPosBtn.AutoButtonColor = false
        resetPosBtn.ZIndex = 8
        Instance.new("UICorner", resetPosBtn).CornerRadius = UDim.new(1, 0)

        local resetGrad = Instance.new("UIGradient", resetPosBtn)
        resetGrad.Rotation = 90
        resetGrad.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0.00, Color3.fromRGB(80, 90, 110)),
            ColorSequenceKeypoint.new(0.50, Color3.fromRGB(45, 55, 75)),
            ColorSequenceKeypoint.new(1.00, Color3.fromRGB(18, 22, 32)),
        })

        local resetStroke = Instance.new("UIStroke", resetPosBtn)
        resetStroke.Color = Color3.fromRGB(150, 175, 210)
        resetStroke.Thickness = 1.5
        resetStroke.Transparency = 0.35

        local resetLbl = Instance.new("TextLabel", resetPosBtn)
        resetLbl.Size = UDim2.new(1, 0, 1, 0)
        resetLbl.BackgroundTransparency = 1
        resetLbl.Text = "RESET POSITIONS"
        resetLbl.TextColor3 = Color3.fromRGB(255, 255, 255)
        resetLbl.Font = Enum.Font.GothamBlack
        resetLbl.TextSize = 13
        resetLbl.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
        resetLbl.TextStrokeTransparency = 0.55
        resetLbl.ZIndex = 9

        resetPosBtn.MouseEnter:Connect(function()
            TS:Create(resetPosBtn, TweenInfo.new(0.14, Enum.EasingStyle.Quad), {
                Size = UDim2.new(1, -12, 0.92, 0),
                Position = UDim2.new(0, 6, 0.04, 0),
            }):Play()
            TS:Create(resetStroke, TweenInfo.new(0.14), {Transparency = 0.05}):Play()
        end)
        resetPosBtn.MouseLeave:Connect(function()
            TS:Create(resetPosBtn, TweenInfo.new(0.14, Enum.EasingStyle.Quad), {
                Size = UDim2.new(1, -20, 0.82, 0),
                Position = UDim2.new(0, 10, 0.09, 0),
            }):Play()
            TS:Create(resetStroke, TweenInfo.new(0.14), {Transparency = 0.35}):Play()
        end)

        local resetDebounce = false
        resetPosBtn.MouseButton1Click:Connect(function()
            if resetDebounce then return end
            resetDebounce = true
            resetFloatingPositions()
            resetLbl.Text = "RESET ✓"
            task.delay(1.2, function()
                if resetLbl and resetLbl.Parent then
                    resetLbl.Text = "RESET POSITIONS"
                    resetDebounce = false
                end
            end)
        end)
    end


    local keyPage = contentPages["Keybinds"]
    mkSect(keyPage, "Keybinds")
    addKeybindRow(keyPage, "Carry Mode", KB.CarryToggle)
    addKeybindRow(keyPage, "Lagger Mode", KB.LaggerMode)
    addKeybindRow(keyPage, "Auto Bat", KB.AutoBat)
    addKeybindRow(keyPage, "Bat Bypass Toggle", KB.BatV2)
    addKeybindRow(keyPage, "Insta Reset", KB.InstaReset)
    addKeybindRow(keyPage, "TP Down", KB.TPFloor)
    addKeybindRow(keyPage, "Drop Brainrot", KB.DropBrainrot)
    addKeybindRow(keyPage, "Hide GUI", KB.GuiHide)

    local spacer = Instance.new("Frame", keyPage)
    spacer.Size = UDim2.new(1, 0, 0, 16)
    spacer.BackgroundTransparency = 1
    spacer.LayoutOrder = getNextOrder(keyPage)
    spacer.ZIndex = 7

    -- Compact Auto Steal progress bar (styled like the supplied AUTO GRAB reference)
    pbFrame = Instance.new("Frame", gui)
    pbFrame.Name = "AutoStealProgress"
    pbFrame.Size = UDim2.new(0, 300, 0, 50)
    pbFrame.Position = UDim2.new(0.5, -150, 1, -66)
    pbFrame.BackgroundColor3 = Color3.fromRGB(5, 12, 9)
    pbFrame.BackgroundTransparency = 0
    pbFrame.BorderSizePixel = 0
    pbFrame.Active = true
    pbFrame.ClipsDescendants = true
    pbFrame.Visible = CONFIG.AUTO_STEAL_ENABLED == true
    pbFrame.ZIndex = 10

    local pbCorner = Instance.new("UICorner", pbFrame)
    pbCorner.CornerRadius = UDim.new(0, 8)

    local pbStroke = Instance.new("UIStroke", pbFrame)
    pbStroke.Name = "AutoStealBorder"
    pbStroke.Color = Color3.fromRGB(25, 75, 48)
    pbStroke.Thickness = 1
    pbStroke.Transparency = 0.15

    backgroundImagePB = Instance.new("ImageLabel", pbFrame)
    backgroundImagePB.Name = "BackgroundImagePB"
    backgroundImagePB.Size = UDim2.new(1, 0, 1, 0)
    backgroundImagePB.Position = UDim2.new(0, 0, 0, 0)
    backgroundImagePB.BackgroundTransparency = 1
    backgroundImagePB.BorderSizePixel = 0
    backgroundImagePB.Image = ""
    backgroundImagePB.ScaleType = Enum.ScaleType.Crop
    backgroundImagePB.ImageTransparency = 1
    backgroundImagePB.ZIndex = 1
    Instance.new("UICorner", backgroundImagePB).CornerRadius = UDim.new(0, 8)

    pbScale = Instance.new("UIScale", pbFrame)
    pbScale.Scale = progressBarScale

    if savedProgressBarPos then
        pbFrame.Position = UDim2.new(
            savedProgressBarPos.XScale or 0.5,
            savedProgressBarPos.XOffset or -150,
            savedProgressBarPos.YScale or 1,
            savedProgressBarPos.YOffset or -66
        )
    end

    local grabLabel = Instance.new("TextLabel", pbFrame)
    grabLabel.Name = "DiscordLabel"
    grabLabel.Size = UDim2.new(1, -66, 0, 16)
    grabLabel.Position = UDim2.new(0, 12, 0, 5)
    grabLabel.BackgroundTransparency = 1
    grabLabel.Text = "AUTO GRAB"
    grabLabel.TextColor3 = Color3.fromRGB(235, 240, 238)
    grabLabel.Font = Enum.Font.GothamBold
    grabLabel.TextSize = 11
    grabLabel.TextXAlignment = Enum.TextXAlignment.Left
    grabLabel.TextYAlignment = Enum.TextYAlignment.Center
    grabLabel.ZIndex = 13

    local percentLabel = Instance.new("TextLabel", pbFrame)
    percentLabel.Name = "AutoStealPercent"
    percentLabel.Size = UDim2.new(0, 48, 0, 16)
    percentLabel.Position = UDim2.new(1, -58, 0, 5)
    percentLabel.BackgroundTransparency = 1
    percentLabel.Text = "0%"
    percentLabel.TextColor3 = Color3.fromRGB(235, 240, 238)
    percentLabel.Font = Enum.Font.GothamBold
    percentLabel.TextSize = 11
    percentLabel.TextXAlignment = Enum.TextXAlignment.Right
    percentLabel.ZIndex = 13

    local progressRow = Instance.new("Frame", pbFrame)
    progressRow.Name = "ProgressRow"
    progressRow.Size = UDim2.new(1, -24, 0, 12)
    progressRow.Position = UDim2.new(0, 12, 0, 28)
    progressRow.BackgroundTransparency = 1
    progressRow.ZIndex = 11

    local fillRegion = Instance.new("Frame", progressRow)
    fillRegion.Name = "FillRegion"
    fillRegion.Size = UDim2.new(1, 0, 1, 0)
    fillRegion.BackgroundColor3 = Color3.fromRGB(18, 29, 23)
    fillRegion.BackgroundTransparency = 0
    fillRegion.BorderSizePixel = 0
    fillRegion.ClipsDescendants = true
    fillRegion.ZIndex = 12
    Instance.new("UICorner", fillRegion).CornerRadius = UDim.new(1, 0)

    local fillStroke = Instance.new("UIStroke", fillRegion)
    fillStroke.Name = "FillRegionStroke"
    fillStroke.Color = Color3.fromRGB(35, 75, 53)
    fillStroke.Thickness = 0.7
    fillStroke.Transparency = 0.15

    progressFill = Instance.new("Frame", fillRegion)
    progressFill.Name = "ProgressFill"
    progressFill.Size = UDim2.new(0, 0, 1, 0)
    progressFill.Position = UDim2.new(0, 0, 0, 0)
    progressFill.BackgroundColor3 = Color3.fromRGB(235, 240, 238)
    progressFill.BorderSizePixel = 0
    progressFill.ZIndex = 13
    Instance.new("UICorner", progressFill).CornerRadius = UDim.new(1, 0)

    progressPct = percentLabel

    -- The steal bar remains draggable independently of the main panel lock.
    do
        local draggingBar = false
        local dragStart, barStart
        local function stopBarDrag()
            if not draggingBar then return end
            draggingBar = false
            savedProgressBarPos = {
                XScale = pbFrame.Position.X.Scale,
                XOffset = pbFrame.Position.X.Offset,
                YScale = pbFrame.Position.Y.Scale,
                YOffset = pbFrame.Position.Y.Offset,
            }
            pcall(saveAllSettings)
        end
        pbFrame.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                draggingBar = true
                dragStart = input.Position
                barStart = pbFrame.Position
            end
        end)
        UIS.InputChanged:Connect(function(input)
            if not draggingBar then return end
            if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
                local delta = input.Position - dragStart
                pbFrame.Position = UDim2.new(
                    barStart.X.Scale, barStart.X.Offset + delta.X,
                    barStart.Y.Scale, barStart.Y.Offset + delta.Y
                )
            end
        end)
        UIS.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                stopBarDrag()
            end
        end)
    end

    -- Give the steal bar its own ScreenGui so hiding the legacy menu cannot hide it.
    local stealBarGui = Instance.new("ScreenGui")
    stealBarGui.Name = "russoStealBar"
    stealBarGui.ResetOnSpawn = false
    stealBarGui.IgnoreGuiInset = true
    stealBarGui.DisplayOrder = 100003
    stealBarGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    pcall(function()
        if syn and syn.protect_gui then syn.protect_gui(stealBarGui) end
    end)
    local stealBarParented = pcall(function()
        stealBarGui.Parent = game:GetService("CoreGui")
    end)
    if not stealBarParented then
        stealBarGui.Parent = LP:WaitForChild("PlayerGui")
    end
    pbFrame.Parent = stealBarGui
    updateProgressBarVisibility()

    task.spawn(function()
        local lastFrame = _tick()
        local fpsSamples = {}
        local fpsAvg = 60
        RunService.RenderStepped:Connect(function()
            local now = _tick()
            local dt = now - lastFrame
            lastFrame = now
            if dt > 0 then
                table.insert(fpsSamples, 1 / dt)
                if #fpsSamples > 30 then table.remove(fpsSamples, 1) end
                local sum = 0
                for _, v in ipairs(fpsSamples) do sum = sum + v end
                fpsAvg = sum / #fpsSamples
            end
        end)
        while not _G.BloodHoundsUnloaded do
            local ping = 0
            pcall(function() ping = LP:GetNetworkPing() * 1000 end)
            if fpsNeon then
                fpsNeon.Text = string.format("%dFPS · %dms", _floor(fpsAvg + 0.5), _floor(ping + 0.5))
            end
            task.wait(0.75)
        end
    end)

    drag(main)
end

function createMobilePanel()
    local panel = Instance.new("ScreenGui")
    panel.Name = "FictionHubMobilePanel"
    panel.ResetOnSpawn = false
    panel.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    pcall(function() if syn and syn.protect_gui then syn.protect_gui(panel) end end)
    local okPanel = pcall(function() panel.Parent = game:GetService("CoreGui") end)
    if not okPanel then panel.Parent = LP:WaitForChild("PlayerGui") end

    local BTN_W, BTN_H = 60, 60
    local GAP = 8
    local COLUMNS = 2
    local ROWS = 3
    local PANEL_W = BTN_W * COLUMNS + GAP * (COLUMNS - 1)
    local PANEL_H = BTN_H * ROWS + (GAP + 10) * (ROWS - 1)

    local container = Instance.new("Frame", panel)
    container.Name = "FloatingPanel"
    container.Size = UDim2.new(0, PANEL_W, 0, PANEL_H)
    container.Position = UDim2.new(0, 10, 0, 0)
    container.BackgroundTransparency = 1
    container.BorderSizePixel = 0
    container.Active = true
    container.Selectable = true
    container.ClipsDescendants = false

    local containerScale = Instance.new("UIScale", container)
    containerScale.Scale = floatingButtonScale
    table.insert(_floatingUIScales, containerScale)

    local btnContainer = Instance.new("Frame", container)
    btnContainer.Name = "ButtonsContainer"
    btnContainer.Size = UDim2.new(1, 0, 1, 0)
    btnContainer.BackgroundTransparency = 1
    btnContainer.ClipsDescendants = false

    local WHITE = Color3.fromRGB(255, 255, 255)
    local INACTIVE_BG = Color3.fromRGB(8,8,10)

    local buttons = {}
    local buttonNames = {"DropBR", "AutoBat", "TpDown", "Carry", "Lagger1", "Lagger2"}
    local buttonTexts = {"DROP\nBR", "BAT\nAIMBOT", "TP\nDOWN", "CARRY\nSPD", "LAGGER\nNORMAL", "LAGGER\nCARRY"}

    local function createButton(name, text, order, isToggle, callback)
        local btn = Instance.new("TextButton", btnContainer)
        btn.Name = name
        btn.Size = UDim2.new(0, BTN_W, 0, BTN_H)
        btn.BackgroundColor3 = INACTIVE_BG
        btn.BorderSizePixel = 0
        btn.Text = ""
        btn.AutoButtonColor = false
        btn.ZIndex = 10

        local savedPos = savedButtonPositions[name]
        if savedPos then
            btn.Position = UDim2.new(0, savedPos.X or 0, 0, savedPos.Y or 0)
        else
            local defX, defY = getDefaultButtonPosition(name)
            btn.Position = UDim2.new(0, defX, 0, defY)
        end

        btn.BackgroundColor3 = INACTIVE_BG
        Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 10)

        local bgGrad = Instance.new("UIGradient", btn)
        bgGrad.Name = "BtnGrad"
        bgGrad.Rotation = 90
        bgGrad.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0.00, Color3.fromRGB(70, 70, 78)),
            ColorSequenceKeypoint.new(0.35, Color3.fromRGB(28, 28, 34)),
            ColorSequenceKeypoint.new(0.70, Color3.fromRGB(10, 10, 14)),
            ColorSequenceKeypoint.new(1.00, Color3.fromRGB(0, 0, 0)),
        })

        local stroke = Instance.new("UIStroke", btn)
        stroke.Color = Color3.fromRGB(70,70,78)
        stroke.Thickness = 1
        stroke.Transparency = 0.45
        stroke.Name = "NormalStroke"

        local label = Instance.new("TextLabel", btn)
        label.Name = "TextLabel"
        label.Size = UDim2.new(1, 0, 1, 0)
        label.BackgroundTransparency = 1
        label.Text = text
        label.TextColor3 = Color3.fromRGB(255, 255, 255)
        label.Font = Enum.Font.GothamBlack
        label.TextSize = 10
        label.TextWrapped = true
        label.ZIndex = 11

        local active = false
        local function setActive(state)
            active = state
            btn:SetAttribute("MobActive", state and true or false)
            paintFloatingBtn(btn, state)
        end
        setActive(false)

        local dragging = false
        local hasMoved = false
        local dragStart = nil
        local startPos = nil
        local movedDistance = 0

        local function onInputBegan(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                dragging = true
                hasMoved = false
                movedDistance = 0
                dragStart = input.Position
                startPos = btn.Position
                _isDraggingButton = true
            end
        end

        local function onInputChanged(input)
            if not dragging then return end
            if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
                local delta = input.Position - dragStart
                movedDistance = delta.Magnitude
                if editModeEnabled and not uiLocked then
                    hasMoved = true
                    btn.Position = UDim2.new(0, startPos.X.Offset + delta.X, 0, startPos.Y.Offset + delta.Y)
                end
            end
        end

        local function onInputEnded(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                if dragging then
                    if movedDistance < 3 then
                        if isToggle then
                            if editModeEnabled and not uiLocked then
                                if callback then callback(function() end) end
                            else
                                if callback then callback(setActive) end
                            end
                        else
                            if callback then callback(setActive, active) end
                        end
                    elseif editModeEnabled and not uiLocked and hasMoved then
                        savedButtonPositions[name] = {
                            X = btn.Position.X.Offset,
                            Y = btn.Position.Y.Offset
                        }
                        pcall(saveAllSettings)
                    end
                    dragging = false
                    hasMoved = false
                    dragStart = nil
                    startPos = nil
                    movedDistance = 0
                    _isDraggingButton = false
                end
            end
        end

        btn.InputBegan:Connect(onInputBegan)
        btn.InputChanged:Connect(onInputChanged)
        btn.InputEnded:Connect(onInputEnded)

        buttons[name] = {btn = btn, setActive = setActive, label = label}
        return setActive
    end

    for i, name in ipairs(buttonNames) do
        local text = buttonTexts[i]
        local callback
        if name == "DropBR" then
            callback = function(setActive)
                if autoBatEnabled then return end
                setActive(true)
                executeDropWithToggle(function(v)
                    if dropBrainrotSetVisual then dropBrainrotSetVisual(v) end
                end)
                task.delay(0.3, function() setActive(false) end)
            end
        elseif name == "AutoBat" then
            callback = function(setActive)
                if not autoBatEnabled then enableAutoBat() else disableAutoBat() end
                setActive(autoBatEnabled)
            end
        elseif name == "TpDown" then
            callback = function(setActive)
                doTpDown()
                setActive(true)
                task.delay(0.2, function() setActive(false) end)
            end
        elseif name == "Carry" then
            callback = function(setActive)
                if not speedMode then
                    speedMode = true; laggerToggled = false; laggerCarryToggled = false; setActive(true)
                    if buttons.Lagger1 and buttons.Lagger1.setActive then buttons.Lagger1.setActive(false) end
                    if buttons.Lagger2 and buttons.Lagger2.setActive then buttons.Lagger2.setActive(false) end
                else
                    speedMode = false; setActive(false)
                end
                refreshSpeedModeLabel()
            end
        elseif name == "Lagger1" then
            callback = function(setActive)
                if speedMode then speedMode = false; if mobSetCarry then mobSetCarry(false) end end
                if not laggerToggled then
                    laggerToggled = true; laggerCarryToggled = false; setActive(true)
                    if buttons.Lagger2 and buttons.Lagger2.setActive then buttons.Lagger2.setActive(false) end
                else
                    laggerToggled = false; setActive(false)
                end
                refreshSpeedModeLabel()
            end
        elseif name == "Lagger2" then
            callback = function(setActive)
                if speedMode then speedMode = false; if mobSetCarry then mobSetCarry(false) end end
                if not laggerCarryToggled then
                    laggerCarryToggled = true; laggerToggled = false; setActive(true)
                    if buttons.Lagger1 and buttons.Lagger1.setActive then buttons.Lagger1.setActive(false) end
                else
                    laggerToggled = false; setActive(false)
                end
                refreshSpeedModeLabel()
            end
        end
        mobSetAutoBat = buttons.AutoBat and buttons.AutoBat.setActive
        mobSetDropBR = buttons.DropBR and buttons.DropBR.setActive
        mobSetTpDown = buttons.TpDown and buttons.TpDown.setActive
        mobSetCarry = buttons.Carry and buttons.Carry.setActive
        mobSetLagger1 = buttons.Lagger1 and buttons.Lagger1.setActive
        mobSetLagger2 = buttons.Lagger2 and buttons.Lagger2.setActive

        local setActive = createButton(name, text, i-1, true, callback)
        if name == "AutoBat" then mobSetAutoBat = setActive end
        if name == "DropBR" then mobSetDropBR = setActive end
        if name == "TpDown" then mobSetTpDown = setActive end
        if name == "Carry" then mobSetCarry = setActive end
        if name == "Lagger1" then mobSetLagger1 = setActive end
        if name == "Lagger2" then mobSetLagger2 = setActive end
    end

    if buttons.AutoBat and buttons.AutoBat.setActive then buttons.AutoBat.setActive(autoBatEnabled) end
    if buttons.Carry and buttons.Carry.setActive then buttons.Carry.setActive(speedMode) end
    if buttons.Lagger1 and buttons.Lagger1.setActive then buttons.Lagger1.setActive(laggerToggled) end
    if buttons.Lagger2 and buttons.Lagger2.setActive then buttons.Lagger2.setActive(laggerCarryToggled) end

    if savedMobilePanelPos then
        container.Position = UDim2.new(
            savedMobilePanelPos.XScale or 0,
            savedMobilePanelPos.XOffset or 10,
            savedMobilePanelPos.YScale or 0,
            savedMobilePanelPos.YOffset or 0
        )
    end

    local draggingPanel = false
    local dragStartPos = nil
    local dragStartMousePos = nil
    local function startDragPanel(input)
        if uiLocked or _isDraggingButton or editModeEnabled then return end
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            draggingPanel = true
            dragStartPos = container.Position
            dragStartMousePos = input.Position
        end
    end
    local function onDragPanel(input)
        if not draggingPanel or uiLocked then return end
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            if dragStartPos and dragStartMousePos then
                local delta = input.Position - dragStartMousePos
                local newX = dragStartPos.X.Offset + delta.X
                local newY = dragStartPos.Y.Offset + delta.Y
                container.Position = UDim2.new(dragStartPos.X.Scale, newX, dragStartPos.Y.Scale, newY)
            end
        end
    end
    local function endDragPanel()
        if draggingPanel then
            draggingPanel = false
            savedMobilePanelPos = {
                XScale = container.Position.X.Scale,
                XOffset = container.Position.X.Offset,
                YScale = container.Position.Y.Scale,
                YOffset = container.Position.Y.Offset
            }
            pcall(saveAllSettings)
        end
        dragStartPos = nil
        dragStartMousePos = nil
    end
    container.InputBegan:Connect(startDragPanel)
    container.InputEnded:Connect(endDragPanel)
    UIS.InputChanged:Connect(onDragPanel)

    UIS.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            endDragPanel()
        end
    end)

    return panel
end

-- Remove floating mobile buttons left behind by earlier executions.
pcall(function()
    local names = {TpBatButton = true, BatV2Button = true, InstaResetButton = true}
    for _, root in ipairs({game:GetService("CoreGui"), LP:FindFirstChildOfClass("PlayerGui")}) do
        if root then
            for _, child in ipairs(root:GetChildren()) do
                if names[child.Name] then pcall(function() child:Destroy() end) end
            end
        end
    end
end)
function createTpBatFloatingButton()
    return nil
end

function createBatV2FloatingButton()
    return nil
end

function createInstaResetFloatingButton()
    return nil
end

function updateUIFromLoaded()
    task.wait()
    if normalBox then normalBox.Text = tostring(NS) end
    if carryBox then carryBox.Text = tostring(CS) end
    if radInput then radInput.Text = tostring(CONFIG.STEAL_RANGE) end
    if laggerBox then laggerBox.Text = tostring(LAGGER_SPEED) end
    if lagger2Box then lagger2Box.Text = tostring(LAGGER_CARRY_SPEED) end
    if batSpeedBox then batSpeedBox.Text = tostring(BAT_AIMBOT_SPEED) end
    if uiScaleBox then uiScaleBox.Text = tostring(uiScaleValue) end
    if dropModeBtnRef then dropModeBtnRef.Text = dropMode == 1 and "Fling" or "Jump Drop" end
    if carrySysNormalBox then carrySysNormalBox.Text = tostring(CarrySystem.normalSpeed) end
    if carrySysCarryBox then carrySysCarryBox.Text = tostring(CarrySystem.carrySpeed) end
    if carrySysLaggerBox then carrySysLaggerBox.Text = tostring(CarrySystem.laggerSpeed) end
    if carrySysLaggerCarryBox then carrySysLaggerCarryBox.Text = tostring(CarrySystem.laggerCarrySpeed) end
    if carrySysSoftStealSpeedBox then carrySysSoftStealSpeedBox.Text = tostring(CarrySystem.carrySpeed) end
    if carrySysSoftStealRadiusBox then carrySysSoftStealRadiusBox.Text = tostring(CarrySystem.softStealRadius) end
    if carrySystemToggleSetter then carrySystemToggleSetter(useCarrySystem) end
    if autoStealVariantLabel then autoStealVariantLabel.Text = autoStealVariantName(autoStealVariant or 1) end
    refreshSpeedModeLabel()

    if infJumpSetVisual then infJumpSetVisual(infJumpEnabled) end
    if infJumpModeSetVisual then infJumpModeSetVisual.Text = infJumpMode end
    if setSafeModeVisual then setSafeModeVisual(antiKickEnabled) end

    for _, ref in ipairs(keyButtonRefs) do
        local entry = ref.entry
        local label = (entry.gp and entry.gp.Name) or (entry.kb and entry.kb.Name) or "None"
        ref.btn.Text = label
    end

    if savedProgressBarPos and pbFrame then
        pbFrame.Position = UDim2.new(
            savedProgressBarPos.XScale or 0.5,
            savedProgressBarPos.XOffset or -150,
            savedProgressBarPos.YScale or 1,
            savedProgressBarPos.YOffset or -66
        )
    end

    applyFloatingButtonScale()

    if uiLocked and setLockUIVisual then setLockUIVisual(true) end
    if editModeEnabled and setEditModeVisual then setEditModeVisual(true) end

    if _G.updateAntiRagdollUI then _G.updateAntiRagdollUI(antiRagdollMode) end
    if antiRagdollMode == "v1" then
        AntiRagdollV1.start()
    elseif antiRagdollMode == "v2" then
        startAntiRagdollV2()
    end

    if antiDieEnabled then
        if setAntiDieVisual then setAntiDieVisual(true) end
        AntiDieModule.start()
    else
        if setAntiDieVisual then setAntiDieVisual(false) end
    end

    if CONFIG.AUTO_STEAL_ENABLED and setInstaGrab then setInstaGrab(true); pcall(startAutoSteal) end


    if batCounterEnabled and setBatCounterVisual then
        setBatCounterVisual(true)
        startBatCounter()
    end
    if antiLagEnabled then
        if setAntiLagVisual then setAntiLagVisual(true) end
        enableAntiLag()
    else
        if setAntiLagVisual then setAntiLagVisual(false) end
        disableAntiLag()
    end
    if espEnabled then
        toggleESP(true)
        if setESPVIsual then setESPVIsual(true) end
    else
        toggleESP(false)
        if setESPVIsual then setESPVIsual(false) end
    end

    if batDesyncTpEnabled then
        if batDesyncTpSetVisual then batDesyncTpSetVisual(true) end
        if not _G.AlvaroTP.on then startBatDesyncTp() end
        updateTpBatButtonWithAntiDie(true)
    else
        if batDesyncTpSetVisual then batDesyncTpSetVisual(false) end
        updateTpBatButtonWithAntiDie(false)
    end

    if autoBatV2Enabled then
        if autoBatV2SetVisual then autoBatV2SetVisual(true) end
        if batV2FloatingButton then
            local btnFrame = batV2FloatingButton:FindFirstChild("Frame")
            if btnFrame then paintFloatingBtn(btnFrame, true) end
        end
    else
        if autoBatV2SetVisual then autoBatV2SetVisual(false) end
        if batV2FloatingButton then
            local btnFrame = batV2FloatingButton:FindFirstChild("Frame")
            if btnFrame then paintFloatingBtn(btnFrame, false) end
        end
    end

    if neonWeatherEnabled then
        applyNeonWeather()
        if setNeonWeatherVisual then setNeonWeatherVisual(true) end
    else
        restoreLightingState()
        if setNeonWeatherVisual then setNeonWeatherVisual(false) end
    end

    if stretchEnabled then
        enableStretch()
        if _G.stretchToggleSetter then _G.stretchToggleSetter(true) end
    else
        if _G.stretchToggleSetter then _G.stretchToggleSetter(false) end
    end

    if setFovVisual then setFovVisual(fovEnabled) end
    if fovSliderSet then fovSliderSet(fovValue) end

    if mobSetAutoBat then mobSetAutoBat(autoBatEnabled) end
    if mobSetCarry then mobSetCarry(speedMode) end
    if mobSetLagger1 then mobSetLagger1(laggerToggled) end
    if mobSetLagger2 then mobSetLagger2(laggerCarryToggled) end

    updateProgressBarVisibility()
    startEnemySpeed()

    toggleLockUI(uiLocked)

    pcall(function() applyOutfitByIndex(currentOutfitIndex) end)
    if outfitSelectorLabel then
        outfitSelectorLabel.Text = OUTFITS[currentOutfitIndex].label
    end

    if backgroundImage then
        applyBackground(backgroundIndex)
    end
    if contentPages and contentPages["Visual"] then
        local vPage = contentPages["Visual"]
        for _, child in ipairs(vPage:GetChildren()) do
            if child:IsA("Frame") and child:FindFirstChild("BackgroundPreview") then
                local prevFrame = child.BackgroundPreview
                local previewImage = prevFrame:FindFirstChild("PreviewImage")
                local previewPlaceholder = prevFrame:FindFirstChild("PreviewPlaceholder")
                local cfg = BACKGROUND_IMAGES[backgroundIndex]
                if previewImage then
                    if cfg and cfg.id and cfg.id ~= "" then
                        previewImage.Image = cfg.id
                        if previewPlaceholder then previewPlaceholder.Visible = false end
                    else
                        previewImage.Image = ""
                        if previewPlaceholder then previewPlaceholder.Visible = true end
                    end
                end
                break
            end
        end
    end

    if MobilePanel then
        local container = MobilePanel:FindFirstChild("FloatingPanel")
        if container then
            if savedMobilePanelPos then
                container.Position = UDim2.new(
                    savedMobilePanelPos.XScale or 0,
                    savedMobilePanelPos.XOffset or 10,
                    savedMobilePanelPos.YScale or 0,
                    savedMobilePanelPos.YOffset or 0
                )
            end
            local btnCont = container:FindFirstChild("ButtonsContainer")
            if btnCont then
                for _, btn in ipairs(btnCont:GetChildren()) do
                    if btn:IsA("TextButton") then
                        local sp = savedButtonPositions and savedButtonPositions[btn.Name]
                        if sp then
                            btn.Position = UDim2.new(0, sp.X or 0, 0, sp.Y or 0)
                        end
                    end
                end
            end
        end
    end

    if tpBatFloatingButton and tpBatFloatingPos then
        local bf = tpBatFloatingButton:FindFirstChild("Frame")
        if bf then
            bf.Position = UDim2.new(
                tpBatFloatingPos.XScale or 0.5,
                tpBatFloatingPos.XOffset or 20,
                tpBatFloatingPos.YScale or 0,
                tpBatFloatingPos.YOffset or 10
            )
        end
    end

    if batV2FloatingButton and batV2FloatingPos then
        local bf = batV2FloatingButton:FindFirstChild("Frame")
        if bf then
            bf.Position = UDim2.new(
                batV2FloatingPos.XScale or 0.5,
                batV2FloatingPos.XOffset or -50,
                batV2FloatingPos.YScale or 0,
                batV2FloatingPos.YOffset or 10
            )
        end
    end

    if instaResetFloatingButton and instaResetFloatingPos then
        local bf = instaResetFloatingButton:FindFirstChild("Frame")
        if bf then
            bf.Position = UDim2.new(
                instaResetFloatingPos.XScale or 0.5,
                instaResetFloatingPos.XOffset or 90,
                instaResetFloatingPos.YScale or 0,
                instaResetFloatingPos.YOffset or 10
            )
        end
    end
end

CleanHubShowE01Warning = false
CleanHubE01StartedAt = nil
CleanHubE01Gui = nil

local function CleanHubIsCarrying()
    local CleanHubCharacter = LP.Character
    if not CleanHubCharacter then return false end
    for _, CleanHubChild in ipairs(CleanHubCharacter:GetChildren()) do
        local CleanHubName = CleanHubChild.Name:lower()
        if CleanHubName:find("brain", 1, true) or CleanHubName:find("animal", 1, true) or CleanHubName:find("carry", 1, true) or CleanHubName:find("steal", 1, true) then return true end
    end
    local CleanHubHumanoid = CleanHubCharacter:FindFirstChildOfClass("Humanoid")
    return CleanHubHumanoid and CleanHubHumanoid.WalkSpeed > 0 and CleanHubHumanoid.WalkSpeed <= 25 and CleanHubHumanoid.WalkSpeed ~= 16
end

CleanHubE01Conn = RunService.Heartbeat:Connect(function()
    if _G.BloodHoundsUnloaded then return end
    if not CleanHubShowE01Warning or not CleanHubIsCarrying() then
        if CleanHubE01Gui then CleanHubE01Gui.Enabled = false end
        CleanHubE01StartedAt = nil
        return
    end
    CleanHubE01StartedAt = CleanHubE01StartedAt or tick()
    local CleanHubProgress = math.clamp((tick() - CleanHubE01StartedAt) / 2.6, 0, 1)
    if not CleanHubE01Gui then
        CleanHubE01Gui = Instance.new("ScreenGui")
        CleanHubE01Gui.Name = "CleanHubE01Warning"
        CleanHubE01Gui.ResetOnSpawn = false
        CleanHubE01Gui.IgnoreGuiInset = true
        CleanHubE01Gui.Parent = LP:WaitForChild("PlayerGui")
        local CleanHubBar = Instance.new("Frame", CleanHubE01Gui)
        CleanHubBar.Name = "Bar"
        CleanHubBar.AnchorPoint = Vector2.new(.5, 0)
        CleanHubBar.Position = UDim2.new(.5, 0, 0, 18)
        CleanHubBar.Size = UDim2.new(0, 300, 0, 36)
        CleanHubBar.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
        CleanHubBar.BackgroundTransparency = .34
        CleanHubBar.BorderSizePixel = 0
        CleanHubBar.ClipsDescendants = true
        Instance.new("UICorner", CleanHubBar).CornerRadius = UDim.new(1, 0)
        local CleanHubTrack = Instance.new("Frame", CleanHubBar)
        CleanHubTrack.Name = "Track"
        CleanHubTrack.Position = UDim2.new(0, 3, 0, 3)
        CleanHubTrack.Size = UDim2.new(1, -6, 1, -6)
        CleanHubTrack.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
        CleanHubTrack.BackgroundTransparency = .48
        CleanHubTrack.BorderSizePixel = 0
        CleanHubTrack.ClipsDescendants = true
        Instance.new("UICorner", CleanHubTrack).CornerRadius = UDim.new(1, 0)
        local CleanHubFill = Instance.new("Frame", CleanHubTrack)
        CleanHubFill.Name = "Fill"
        CleanHubFill.Size = UDim2.new(0, 0, 1, 0)
        CleanHubFill.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        CleanHubFill.BackgroundTransparency = .48
        CleanHubFill.BorderSizePixel = 0
        Instance.new("UICorner", CleanHubFill).CornerRadius = UDim.new(1, 0)
        local CleanHubLabel = Instance.new("TextLabel", CleanHubBar)
        CleanHubLabel.Name = "Label"
        CleanHubLabel.Size = UDim2.new(1, -16, 1, 0)
        CleanHubLabel.Position = UDim2.new(0, 8, 0, 0)
        CleanHubLabel.BackgroundTransparency = 1
        CleanHubLabel.Font = Enum.Font.GothamBlack
        CleanHubLabel.TextSize = 14
        CleanHubLabel.TextStrokeTransparency = .18
        CleanHubLabel.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
        CleanHubLabel.ZIndex = 2
    end
    CleanHubE01Gui.Enabled = true
    local CleanHubFinished = CleanHubProgress >= 1
    CleanHubE01Gui.Bar.Label.Text = CleanHubFinished and "COMPLETED" or "IN PROGRESS"
    CleanHubE01Gui.Bar.Label.TextColor3 = CleanHubFinished and Color3.fromRGB(85, 255, 125) or Color3.fromRGB(255, 75, 85)
    CleanHubE01Gui.Bar.Track.Fill.Size = UDim2.new(CleanHubProgress, 0, 1, 0)
end)

buildGui()
if gui then pcall(function() gui.Enabled = false end) end

-- Floating mobile panels/buttons disabled: do not create them at startup.
MobilePanel = nil
tpBatFloatingButton = nil
batV2FloatingButton = nil
instaResetFloatingButton = nil

-- Remove remnants from a previous execution, if present.
pcall(function()
    local names = {"FictionHubMobilePanel", "TpBatButton", "BatV2Button", "InstaResetButton"}
    for _, name in ipairs(names) do
        local core = game:GetService("CoreGui")
        local obj = core:FindFirstChild(name)
        if obj then obj:Destroy() end
        local pg = LP and LP:FindFirstChildOfClass("PlayerGui")
        obj = pg and pg:FindFirstChild(name)
        if obj then obj:Destroy() end
    end
end)

-- ═══════════════════════════════════════════════════════════════════════════
-- LAST POSITION MARKER
-- ═══════════════════════════════════════════════════════════════════════════
local LastPosMarker = (function()
    local LPM_Players    = game:GetService("Players")
    local LPM_RunService = game:GetService("RunService")
    local LPM_Workspace  = game:GetService("Workspace")
    local LPM_LP         = LPM_Players.LocalPlayer

    local COLOR_MARKER           = Color3.fromRGB(180, 180, 190)
    local COLOR_MARKER_GLOW      = Color3.fromRGB(170, 170, 180)
    local COLOR_OUTLINE          = Color3.fromRGB(230, 230, 235)
    local COLOR_TEXT             = Color3.fromRGB(230, 230, 235)
    local COLOR_TEXT_STROKE      = Color3.fromRGB(30, 30, 35)
    local COLOR_HANDLE_ADORNMENT = Color3.fromRGB(200, 200, 210)

    local LPM_MIN_X, LPM_MAX_X = -536.2, -422
    local LPM_MIN_Y, LPM_MAX_Y = -10, 75
    local LPM_MIN_Z, LPM_MAX_Z = -71.8, 192.9

    local MARKER_GROUND_Y = -7
    local LOOKBACK        = 0.2
    local INTERVAL        = 1 / 60

    local state = {
        marker          = nil,
        markerLocked    = false,
        markerTarget    = nil,
        samples         = {},
        conn            = nil,
        acc             = 0,
    }

    local function finiteVector(v)
        return v ~= nil
            and v.X == v.X and v.Y == v.Y and v.Z == v.Z
            and math.abs(v.X) < 1e7
            and math.abs(v.Y) < 1e7
            and math.abs(v.Z) < 1e7
    end

    local function isInsideMap(pos)
        if not pos then return false end
        return pos.X >= LPM_MIN_X and pos.X <= LPM_MAX_X
           and pos.Y >= LPM_MIN_Y and pos.Y <= LPM_MAX_Y
           and pos.Z >= LPM_MIN_Z and pos.Z <= LPM_MAX_Z
    end

    local function getSnapshotBeforeExit(sample, now)
        if not sample or not sample.history or #sample.history == 0 then return nil end
        local cutoff  = now - LOOKBACK
        local picked  = nil
        for _, entry in ipairs(sample.history) do
            if entry.time <= cutoff then picked = entry else break end
        end
        return picked or sample.history[1]
    end

    local function clearLastTargetMarker()
        state.markerLocked = false
        local m = state.marker
        if not m then return end
        pcall(function()
            m.Transparency = 1
            local sph = m:FindFirstChild("AlwaysOnTopSphere")
            if sph then sph.Visible = false end
            local hl = m:FindFirstChild("LastPositionHighlight")
            if hl then hl.Enabled = false end
            local bb = m:FindFirstChild("LastPositionLabel")
            if bb then bb.Enabled = false end
        end)
    end

    local function updateLastTargetMarker(targetCFrame)
        if state.markerLocked then return end
        if not targetCFrame or not isInsideMap(targetCFrame.Position) then return end

        local groundCFrame = CFrame.new(
            targetCFrame.Position.X,
            MARKER_GROUND_Y,
            targetCFrame.Position.Z
        ) * targetCFrame.Rotation

        local marker = state.marker
        if not marker or not marker.Parent then
            marker = Instance.new("Part")
            marker.Name            = "CleanHubLastPosition"
            marker.Shape           = Enum.PartType.Ball
            marker.Size            = Vector3.new(3, 3, 3)
            marker.Color           = COLOR_MARKER
            marker.Material        = Enum.Material.Neon
            marker.Anchored        = true
            marker.CanCollide      = false
            marker.CanTouch        = false
            marker.CanQuery        = false
            marker.CastShadow      = false
            marker.Parent          = LPM_Workspace

            local sphere = Instance.new("SphereHandleAdornment")
            sphere.Name        = "AlwaysOnTopSphere"
            sphere.Adornee     = marker
            sphere.Radius      = 1.55
            sphere.Color3      = COLOR_HANDLE_ADORNMENT
            sphere.Transparency= 0.05
            sphere.AlwaysOnTop = true
            sphere.Visible     = true
            sphere.ZIndex      = 10
            sphere.Parent      = marker

            local hl = Instance.new("Highlight")
            hl.Name                = "LastPositionHighlight"
            hl.Adornee             = marker
            hl.FillColor           = COLOR_MARKER
            hl.FillTransparency    = 0.15
            hl.OutlineColor        = COLOR_OUTLINE
            hl.OutlineTransparency = 0
            hl.DepthMode           = Enum.HighlightDepthMode.AlwaysOnTop
            hl.Enabled             = true
            hl.Parent              = marker

            local bb = Instance.new("BillboardGui")
            bb.Name        = "LastPositionLabel"
            bb.Size        = UDim2.new(0, 110, 0, 20)
            bb.StudsOffset = Vector3.new(0, 2.1, 0)
            bb.AlwaysOnTop = true
            bb.Adornee     = marker
            bb.Parent      = marker

            local label = Instance.new("TextLabel")
            label.Size                   = UDim2.fromScale(1, 1)
            label.BackgroundTransparency = 1
            label.Text                   = "ultima posicion"
            label.TextColor3             = COLOR_TEXT
            label.TextStrokeColor3       = COLOR_TEXT_STROKE
            label.TextStrokeTransparency = 0.25
            label.TextSize               = 11
            label.Font                   = Enum.Font.GothamMedium
            label.Parent                 = bb

            state.marker = marker
        end

        marker.Transparency = 0
        local sph = marker:FindFirstChild("AlwaysOnTopSphere"); if sph then sph.Visible = true end
        local hl  = marker:FindFirstChild("LastPositionHighlight"); if hl  then hl.Enabled = true end
        local bb  = marker:FindFirstChild("LastPositionLabel");     if bb  then bb.Enabled = true end
        marker.CFrame = groundCFrame
        state.markerLocked = true
    end

    local function targetAlive(player, myRoot)
        if not player or player.Parent ~= LPM_Players or not player.Character then return false end
        local r = player.Character:FindFirstChild("HumanoidRootPart")
        local h = player.Character:FindFirstChildOfClass("Humanoid")
        if not r or not h or h.Health <= 0 or not finiteVector(r.Position) then return false end
        return isInsideMap(r.Position)
    end

    local function monitorLastTarget()
        local myChar = LPM_LP.Character
        local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
        if not myRoot then return end

        local tracked = state.markerTarget
        if tracked and tracked.Parent == LPM_Players then
            local tChar = tracked.Character
            local tRoot = tChar and tChar:FindFirstChild("HumanoidRootPart")
            local tHum  = tChar and tChar:FindFirstChildOfClass("Humanoid")
            local sample = state.samples[tracked] or {}
            state.samples[tracked] = sample

            if tRoot and tHum and tHum.Health > 0 and finiteVector(tRoot.Position) then
                if isInsideMap(tRoot.Position) then
                    local now = tick()
                    if state.markerLocked then sample.history = {} end
                    sample.history = sample.history or {}
                    table.insert(sample.history, {
                        time         = now,
                        safePosition = tRoot.Position,
                        safeCFrame   = tRoot.CFrame,
                    })
                    while sample.history[1] and now - sample.history[1].time > (LOOKBACK + 0.25) do
                        table.remove(sample.history, 1)
                    end
                    sample.safePosition = tRoot.Position
                    sample.safeCFrame   = tRoot.CFrame
                    clearLastTargetMarker()
                else
                    local snap = getSnapshotBeforeExit(sample, tick())
                    updateLastTargetMarker((snap and snap.safeCFrame) or sample.safeCFrame)
                end
            end

            if (tHum and tHum.Health <= 0)
               or (not tRoot and not sample.safeCFrame) then
                state.markerTarget = nil
                clearLastTargetMarker()
            else
                return
            end
        end

        local closest, closestDist = nil, math.huge
        for _, p in ipairs(LPM_Players:GetPlayers()) do
            if p ~= LPM_LP and targetAlive(p, myRoot) then
                local r = p.Character:FindFirstChild("HumanoidRootPart")
                local d = (r.Position - myRoot.Position).Magnitude
                if d < closestDist then closest, closestDist = p, d end
            end
        end

        if closest then
            state.markerTarget = closest
            local r = closest.Character:FindFirstChild("HumanoidRootPart")
            state.samples[closest] = {
                safePosition = r.Position,
                safeCFrame   = r.CFrame,
                history      = { { time = tick(), safePosition = r.Position, safeCFrame = r.CFrame } },
            }
        end
    end

    local api = {}

    function api.GetMarkerCFrame()
        local m = state.marker
        if state.markerLocked and m and m.Parent and m.Transparency < 1 then
            local tracked = state.markerTarget
            local tRoot = tracked and tracked.Character and tracked.Character:FindFirstChild("HumanoidRootPart")
            if tRoot and isInsideMap(tRoot.Position) then
                clearLastTargetMarker()
                return nil
            end
            return m.CFrame
        end
        return nil
    end

    function api.ForceMarker(player, fallbackCFrame)
        if player and player.Parent == LPM_Players then state.markerTarget = player end
        local sample = player and state.samples[player]
        local snap   = getSnapshotBeforeExit(sample, tick())
        local remembered = (snap and snap.safeCFrame) or (sample and sample.safeCFrame)
        if not remembered and fallbackCFrame and isInsideMap(fallbackCFrame.Position) then
            remembered = fallbackCFrame
        end
        if remembered then updateLastTargetMarker(remembered) end
        return api.GetMarkerCFrame()
    end

    function api.SetTarget(player)
        if player and player ~= LPM_LP and player.Parent == LPM_Players then
            state.markerTarget = player
        end
    end

    function api.Clear()
        state.markerTarget = nil
        clearLastTargetMarker()
    end

    function api.Destroy()
        api.Stop()
        if state.marker and state.marker.Parent then
            pcall(function() state.marker:Destroy() end)
        end
        state.marker = nil
    end

    function api.Start()
        if state.conn then return state.conn end
        state.conn = LPM_RunService.Heartbeat:Connect(function(dt)
            state.acc = state.acc + (dt or 0)
            if state.acc < INTERVAL then return end
            state.acc = state.acc % INTERVAL
            monitorLastTarget()
        end)
        return state.conn
    end

    function api.Stop()
        if state.conn then
            state.conn:Disconnect()
            state.conn = nil
        end
    end

    api.IsInsideMap = isInsideMap
    return api
end)()

_G.CleanHubLastPosMarker = LastPosMarker
LastPosMarker.Start()

-- Fully unload Clean Hub: stop active features and remove its UI while preserving saved settings.
function unloadCleanHub()
    if _G.BloodHoundsUnloading or _G.BloodHoundsUnloaded then return end
    _G.BloodHoundsUnloading = true

    -- Snapshot the exact saved config before cleanup. Some feature stop routines save
    -- their disabled runtime state; restore the original file after cleanup completes.
    local _explodeConfigExisted = false
    local _explodeConfigSnapshot = nil
    pcall(function()
        if isfile and isfile(CONFIG_FILE) then
            _explodeConfigExisted = true
            if readfile then _explodeConfigSnapshot = readfile(CONFIG_FILE) end
        end
    end)

    -- Stop feature systems first; each call is isolated so one failure cannot block the rest.
    local cleanupCalls = {
        -- Explode UI must preserve CONFIG_FILE and all saved preferences.
        -- Do not call resetToFactoryDefaults here: it deletes and rewrites saved settings.
        function() if stopAllBackgroundTasks then stopAllBackgroundTasks() end end,
        function() if stopAutoSteal then stopAutoSteal() end end,
        function() if stopBatCounter then stopBatCounter() end end,
        function() if disableAutoBat then disableAutoBat() end end,
        function() if stopBatDesyncTp then stopBatDesyncTp() end end,
        function() if disableBatV2 then disableBatV2() end end,
        function() if stopAnimPack then stopAnimPack() end end,
        function() if CarrySystem and CarrySystem.stop then CarrySystem:stop() end end,
        function() if toggleNeonWeather then toggleNeonWeather(false) end end,
        function() if disableCustomFov then disableCustomFov() end end,
        function() if stopEnemySpeed then stopEnemySpeed() end end,
        function() if stopESPLoop then stopESPLoop() end end,
        function() if stopUnwalk then stopUnwalk() end end,
        function() if stopAlvaroTP then stopAlvaroTP() end end,
        function() if AntiRagdollV1 and AntiRagdollV1.isRunning() then AntiRagdollV1.stop() end end,
        function() if AntiRagdollV2 and AntiRagdollV2.Enabled then stopAntiRagdollV2() end end,
        function() if AntiDieModule and AntiDieModule.stop then AntiDieModule.stop() end end,
        function() if AntiFlingShieldModule and AntiFlingShieldModule.stop then AntiFlingShieldModule.stop() end end,
        function() if BB and BB.Stop then BB.Stop() end end,
        function() if LastPosMarker then LastPosMarker.Destroy() end end,
        function() if _G.CursedBatKatana and _G.CursedBatKatana.ApplySkin then _G.CursedBatKatana.ApplySkin("NONE") end end,
    }
    for _, cleanup in ipairs(cleanupCalls) do pcall(cleanup) end

    -- Restore the exact saved settings file (or its original absence) after all stop routines.
    pcall(function()
        if _explodeConfigExisted then
            if _explodeConfigSnapshot ~= nil and writefile then
                writefile(CONFIG_FILE, _explodeConfigSnapshot)
            end
        elseif isfile and isfile(CONFIG_FILE) and delfile then
            delfile(CONFIG_FILE)
        end
    end)

    -- Turn off state-driven loops before destroying the interface.
    _G.BloodHoundsUnloaded = true
    _G.BloodHoundsRunning = false
    _G.InstaResetLoaded = nil
    _G.InstaReset = nil
    _G.BloodHoundsUnloading = nil
    katanaSkinEnabled = false
    infJumpEnabled = false
    CleanHubShowE01Warning = false
    if _G.AmbitiousNormalInfJump then
        _G.AmbitiousNormalInfJump.holdPressed = false
        _G.AmbitiousNormalInfJump.holdActive = false
        _G.AmbitiousNormalInfJump.mobileActive = false
        _G.AmbitiousNormalInfJump.controllerActive = false
    end

    -- Disconnect persistent input/respawn/render listeners owned by this script.
    for _, conn in pairs({
        mainKeybindConnection, mainCharacterAddedConn, alvaroShiftInputConn,
        alvaroCharAddedConn, _G.InstaResetCharConn, CleanHubE01Conn,
        infJumpConn, tryardHeartbeatConn, enemySpeedConn, espConn,
        movementLoop, steppedConn, stretchConn, stretchFovConn,
        customFovConn, batDesyncTpConn, _dropConn,
    }) do
        if conn then pcall(function() conn:Disconnect() end) end
    end
    mainKeybindConnection, mainCharacterAddedConn = nil, nil
    alvaroShiftInputConn, alvaroCharAddedConn = nil, nil
    _G.InstaResetCharConn, CleanHubE01Conn = nil, nil

    -- Remove only GUI/visual instances created by this hub.
    local guiNames = {
        russo = true,
        FictionHubMobilePanel = true,
        CleanHubE01Warning = true,
    }
    for _, parent in ipairs({game:GetService("CoreGui"), LP and LP:FindFirstChildOfClass("PlayerGui")}) do
        if parent then
            for _, child in ipairs(parent:GetChildren()) do
                if guiNames[child.Name] then pcall(function() child:Destroy() end) end
            end
        end
    end
    if LP and LP.Character then
        for _, obj in ipairs(LP.Character:GetDescendants()) do
            if obj.Name == "FictionHubSpeedIndicator" or obj.Name == "RagCountdownBillboard" then
                pcall(function() obj:Destroy() end)
            end
        end
    end
    for _, player in ipairs(Players:GetPlayers()) do
        local char = player.Character
        if char then
            for _, obj in ipairs(char:GetDescendants()) do
                if obj.Name == "EnemySpeedGui" or obj.Name == "FictionHubESP" then
                    pcall(function() obj:Destroy() end)
                end
            end
        end
    end
end

if loadAllSettings() then
    updateUIFromLoaded()
end

if useCarrySystem then
    CarrySystem:start()
    CarrySystem.speedToggled = speedMode
    if laggerCarryToggled then
        CarrySystem:setLaggerMode(2)
    elseif laggerToggled then
        CarrySystem:setLaggerMode(1)
    else
        CarrySystem:setLaggerMode(0)
    end
    CarrySystem:setSoftStealEnabled(true)
end

if LP.Character then
    task.wait(0.1)
    if waitForCharReady(LP.Character, 5) then
        setupMovementAndIndicators(LP.Character)
        if currentAnimPack ~= "Off" then
            startAnimPack(currentAnimPack)
        end
        pcall(function() applyOutfitByIndex(currentOutfitIndex) end)
        if katanaSkinEnabled then
            task.wait(0.6)
            pcall(function() KatanaSkin.ApplySkin("KATANA") end)
        end
    end
end

local _respawnQueue = 0
mainCharacterAddedConn = LP.CharacterAdded:Connect(function(char)
    if _G.BloodHoundsUnloaded then return end
    _respawnQueue = _respawnQueue + 1
    local myId = _respawnQueue

    if stealConnection then stealConnection:Disconnect(); stealConnection = nil end
    isStealing = false
    stopBatCounter()
    if not _tpBatUnwalkForced then stopUnwalk() end
    stopDropBrainrot()
    if autoBatEnabled then disableAutoBat() end
    if batDesyncTpEnabled then stopBatDesyncTp() end
    if autoBatV2Enabled then disableBatV2() end

    local deadline = _tick() + 5
    while (not char.Parent) or (not char:FindFirstChild("HumanoidRootPart"))
          or (not char:FindFirstChildOfClass("Humanoid")) do
        if _tick() > deadline then return end
        if myId ~= _respawnQueue then return end
        task.wait(0.05)
    end

    _hookedVelParts = {}
    local _hrpRespawn = _setupVelChecked(char)
    _hookVelHRP(_hrpRespawn)

    setupMovementAndIndicators(char)

    if antiRagdollMode == "v1" then AntiRagdollV1.start()
    elseif antiRagdollMode == "v2" then startAntiRagdollV2() end

    if AntiDieModule.enabled then task.defer(function() activateOnCharacter(char) end) end
    if CONFIG.AUTO_STEAL_ENABLED then pcall(startAutoSteal) end
    if batDesyncTpEnabled then task.defer(startBatDesyncTp) end
    if autoBatV2Enabled then task.defer(enableBatV2) end


    if batCounterEnabled then startBatCounter() end

    if currentAnimPack ~= "Off" then
        task.wait(0.3)
        startAnimPack(currentAnimPack)
    end

    updateProgressBarVisibility()
    refreshSpeedModeLabel()

    pcall(function() applyOutfitByIndex(currentOutfitIndex) end)
    if outfitSelectorLabel then
        outfitSelectorLabel.Text = OUTFITS[currentOutfitIndex].label
    end

    if katanaSkinEnabled then
        task.wait(1)
        pcall(function() KatanaSkin.ApplySkin("KATANA") end)
    end
end)

local lastLaggerToggle = 0
local LAGGER_COOLDOWN = 0.3

mainKeybindConnection = UIS.InputBegan:Connect(function(input, gpe)
    if _G.BloodHoundsUnloaded then return end
    if _anyKeyListening then return end
    if input.UserInputType == Enum.UserInputType.Keyboard then
        if gpe or UIS:GetFocusedTextBox() then return end
    elseif not isGamepadInput(input) then
        return
    end
    if not isBindableInput(input) then return end

    local kc = input.KeyCode
    if not kc then return end

    if kbMatch(KB.LaggerMode, kc) then
        if _tick() - lastLaggerToggle >= LAGGER_COOLDOWN then
            lastLaggerToggle = _tick()
            toggleLaggerCycle()
        end
        return
    end
    if kbMatch(KB.CarryToggle, kc) then toggleCarryMode(); return end
    if kbMatch(KB.DropBrainrot, kc) then
        if not dropActive then
            if dropBrainrotSetVisual then dropBrainrotSetVisual(true) end
            executeDropWithToggle(dropBrainrotSetVisual)
        end
        return
    end
    if kbMatch(KB.TPFloor, kc) then doTpDown(); return end
    if kbMatch(KB.InstaReset, kc) then
        if _G.InstaReset and _G.InstaReset.Trigger then
            _G.InstaReset.Trigger()
        end
        return
    end
    if kbMatch(KB.AutoBat, kc) then
        if not autoBatEnabled then
            enableAutoBat()
            if autoBatSetVisual then autoBatSetVisual(true) end
            if mobSetAutoBat then mobSetAutoBat(true) end
        else
            disableAutoBat()
            if autoBatSetVisual then autoBatSetVisual(false) end
            if mobSetAutoBat then mobSetAutoBat(false) end
        end
        return
    end
    if kbMatch(KB.BatV2, kc) then
        toggleBatV2()
        if _G.russoSetToggle then _G.russoSetToggle("combat.batBypass", autoBatV2Enabled) end
        if autoBatV2SetVisual then autoBatV2SetVisual(autoBatV2Enabled) end
        if batV2FloatingButton then
            local btnFrame = batV2FloatingButton:FindFirstChild("Frame")
            if btnFrame then paintFloatingBtn(btnFrame, autoBatV2Enabled) end
        end
        return
    end
    if kbMatch(KB.GuiHide, kc) then
        if _G.russoPanel then _G.russoPanel.Visible = not _G.russoPanel.Visible end
        return
    end
end)
-- ============================================================
-- RUSSO UI: compact frontend wired to the original Clean Hub logic
-- Intro omitted. The original menu is retained disabled as a backend
-- compatibility layer; only this russo panel is visible.
-- ============================================================

do
    if gui then pcall(function() gui.Enabled = false end) end

    local Rus = {}
    local RS = {
        bg = Color3.fromRGB(7, 7, 8),
        row = Color3.fromRGB(25, 25, 27),
        field = Color3.fromRGB(48, 48, 50),
        text = Color3.fromRGB(245, 245, 245),
        muted = Color3.fromRGB(145, 145, 148),
        white = Color3.fromRGB(255, 255, 255),
    }

    local function inst(className, props, parent)
        local obj = Instance.new(className)
        for k, v in pairs(props or {}) do obj[k] = v end
        obj.Parent = parent
        return obj
    end
    local function corner(obj, px)
        inst("UICorner", {CornerRadius = UDim.new(0, px or 7)}, obj)
    end
    local function setCorner(obj, px)
        local c = obj:FindFirstChildOfClass("UICorner")
        if c then c.CornerRadius = UDim.new(0, px or 7) else corner(obj, px) end
    end

    local pg = LP:WaitForChild("PlayerGui")
    -- Keep the original ScreenGui alive because AutoStealProgress is parented to it.
    -- Hide only its old menu; the compact steal bar remains independently visible.
    if gui and gui.Parent then
        gui.Name = "russoStealLayer"
        gui.DisplayOrder = 100002
        if main then main.Visible = false end
    end
    local oldRusso = game:GetService("CoreGui"):FindFirstChild("russo") or pg:FindFirstChild("russo")
    if oldRusso then oldRusso:Destroy() end

    local russoGui = inst("ScreenGui", {
        Name = "russo", ResetOnSpawn = false,
        ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
        DisplayOrder = 100001,
    }, pg)
    pcall(function()
        russoGui.Parent = game:GetService("CoreGui")
    end)

    -- The steal progress panel stays under its original ScreenGui.

    local panel = inst("Frame", {
        Name = "Main", Size = UDim2.fromOffset(255, 545),
        AnchorPoint = Vector2.new(1, 1), Position = UDim2.new(1, -18, 1, -58),
        BackgroundColor3 = RS.bg, BorderSizePixel = 0,
    }, russoGui)
    panel.ClipsDescendants = true
    Rus.Panel = panel
    corner(panel, 15)
    inst("UIStroke", {Color = Color3.fromRGB(78, 78, 82), Transparency = 0.12, Thickness = 1.2}, panel)
    local scale = inst("UIScale", {Scale = 1}, panel)
    -- Route the source hub's Background selector to the visible russo panel.
    local russoBackground = inst("ImageLabel", {
        Name = "PanelBackground", Size = UDim2.fromScale(1, 1), Position = UDim2.fromScale(0, 0),
        BackgroundTransparency = 1, Image = "", ImageTransparency = backgroundImageTransparency,
        ScaleType = Enum.ScaleType.Crop, ZIndex = 0,
    }, panel)
    corner(russoBackground, 15)
    backgroundImage = russoBackground

    local title = inst("TextLabel", {
        Name = "Title", Size = UDim2.new(1, -12, 0, 40), Position = UDim2.fromOffset(6, 3),
        BackgroundTransparency = 1, Text = "russo", TextColor3 = RS.text,
        Font = Enum.Font.GothamBlack, TextSize = 22, Active = true,
    }, panel)

    local scroll = inst("ScrollingFrame", {
        Name = "Options", Size = UDim2.new(1, -12, 1, -48), Position = UDim2.fromOffset(6, 44),
        BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 3,
        ScrollBarImageColor3 = RS.muted, CanvasSize = UDim2.new(),
        AutomaticCanvasSize = Enum.AutomaticSize.Y,
    }, panel)
    inst("UIPadding", {PaddingBottom = UDim.new(0, 6)}, scroll)
    inst("UIListLayout", {Padding = UDim.new(0, 4), SortOrder = Enum.SortOrder.LayoutOrder}, scroll)

    local uiControls = {}
    local keyCaptureId = nil
    local function save()
        if type(saveAllSettings) == "function" then pcall(saveAllSettings) end
    end
    local function section(label)
        inst("TextLabel", {
            Size = UDim2.new(1, -2, 0, 18), BackgroundTransparency = 1,
            Text = string.upper(label), TextColor3 = RS.muted,
            Font = Enum.Font.GothamBold, TextSize = 14,
            TextXAlignment = Enum.TextXAlignment.Left,
        }, scroll)
    end
    local function rowBase(label)
        local row = inst("Frame", {
            Size = UDim2.new(1, -2, 0, 34), BackgroundColor3 = RS.row, BorderSizePixel = 0,
        }, scroll)
        corner(row, 8)
        inst("TextLabel", {
            Size = UDim2.new(1, -98, 1, 0), Position = UDim2.fromOffset(8, 0),
            BackgroundTransparency = 1, Text = label, TextColor3 = RS.text,
            Font = Enum.Font.GothamBold, TextSize = 12,
            TextXAlignment = Enum.TextXAlignment.Left,
        }, row)
        return row
    end
    local function setToggleVisual(id, on)
        local c = uiControls[id]
        if not c then return end
        c.value = on == true
        c.input.Text = c.value and "ON" or "OFF"
        c.input.TextColor3 = c.value and Color3.fromRGB(20,20,20) or RS.muted
        c.input.BackgroundColor3 = c.value and RS.white or RS.field
    end
    local function addToggle(label, id, initial, callback)
        local row = rowBase(label)
        local b = inst("TextButton", {
            Size = UDim2.fromOffset(49, 22), Position = UDim2.new(1, -56, 0.5, -11),
            BackgroundColor3 = initial and RS.white or RS.field, BorderSizePixel = 0,
            Text = initial and "ON" or "OFF", TextColor3 = initial and Color3.fromRGB(20,20,20) or RS.muted,
            Font = Enum.Font.GothamBold, TextSize = 12, AutoButtonColor = false,
        }, row)
        corner(b, 7)
        uiControls[id] = {row = row, input = b, value = initial == true}
        b.MouseButton1Click:Connect(function()
            local value = not uiControls[id].value
            setToggleVisual(id, value)
            local ok, err = pcall(callback, value)
            if not ok then warn("[russo] " .. id .. ": " .. tostring(err)) end
            save()
        end)
    end
    -- E01 Warning: progress warning while a brainrot/animal is being carried.
    local function e01IsCarrying()
        local character = LP.Character
        if not character then return false end
        for _, child in ipairs(character:GetChildren()) do
            local name = child.Name:lower()
            if name:find("brain", 1, true) or name:find("animal", 1, true)
                or name:find("carry", 1, true) or name:find("steal", 1, true) then
                return true
            end
        end
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        local speed = humanoid and humanoid.WalkSpeed
        return speed ~= nil and speed > 0 and speed <= 25 and speed ~= 16
    end

    local function createE01WarningGui()
        local playerGui = LP:FindFirstChildOfClass("PlayerGui") or LP:WaitForChild("PlayerGui")
        local old = playerGui:FindFirstChild("russoE01Warning")
        if old then old:Destroy() end
        local screen = Instance.new("ScreenGui")
        screen.Name = "russoE01Warning"
        screen.ResetOnSpawn = false
        screen.IgnoreGuiInset = true
        screen.DisplayOrder = 100004
        screen.Parent = playerGui

        local bar = Instance.new("Frame")
        bar.Name = "Bar"
        bar.AnchorPoint = Vector2.new(0.5, 0)
        bar.Position = UDim2.fromOffset(0, 0)
        bar.Size = UDim2.fromOffset(360, 48)
        bar.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
        bar.BackgroundTransparency = 0.34
        bar.BorderSizePixel = 0
        bar.ClipsDescendants = true
        bar.Parent = screen
        Instance.new("UICorner", bar).CornerRadius = UDim.new(1, 0)

        local track = Instance.new("Frame")
        track.Name = "Track"
        track.Position = UDim2.fromOffset(3, 3)
        track.Size = UDim2.new(1, -6, 1, -6)
        track.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
        track.BackgroundTransparency = 0.48
        track.BorderSizePixel = 0
        track.ClipsDescendants = true
        track.Parent = bar
        Instance.new("UICorner", track).CornerRadius = UDim.new(1, 0)

        local fill = Instance.new("Frame")
        fill.Name = "Fill"
        fill.Size = UDim2.new(0, 0, 1, 0)
        fill.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        fill.BackgroundTransparency = 0.25
        fill.BorderSizePixel = 0
        fill.Parent = track
        Instance.new("UICorner", fill).CornerRadius = UDim.new(1, 0)

        local label = Instance.new("TextLabel")
        label.Name = "Label"
        label.Size = UDim2.new(1, -16, 1, 0)
        label.Position = UDim2.fromOffset(8, 0)
        label.BackgroundTransparency = 1
        label.Font = Enum.Font.GothamBlack
        label.TextSize = 19
        label.TextStrokeTransparency = 0.18
        label.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
        label.TextColor3 = Color3.fromRGB(255, 75, 85)
        label.Text = "E01 WARNING"
        label.ZIndex = 2
        label.Parent = bar
        return screen
    end

    RunService.Heartbeat:Connect(function()
        if not e01WarningEnabled or not e01IsCarrying() then
            if e01WarningGui then e01WarningGui.Enabled = false end
            e01WarningStartedAt = nil
            return
        end
        e01WarningStartedAt = e01WarningStartedAt or tick()
        local progress = math.clamp((tick() - e01WarningStartedAt) / 2.6, 0, 1)
        if not e01WarningGui or not e01WarningGui.Parent then
            e01WarningGui = createE01WarningGui()
        end
        e01WarningGui.Enabled = true
        local warningBar = e01WarningGui:FindFirstChild("Bar")
        if warningBar and pbFrame and pbFrame.Parent then
            warningBar.Position = UDim2.fromOffset(
                pbFrame.AbsolutePosition.X + (pbFrame.AbsoluteSize.X / 2),
                pbFrame.AbsolutePosition.Y + pbFrame.AbsoluteSize.Y + 75
            )
        end
        local finished = progress >= 1
        e01WarningGui.Bar.Label.Text = finished and "SAFE" or "E01 WARNING"
        e01WarningGui.Bar.Label.TextColor3 = finished and Color3.fromRGB(85, 255, 125) or Color3.fromRGB(255, 75, 85)
        e01WarningGui.Bar.Track.Fill.Size = UDim2.new(progress, 0, 1, 0)
    end)
    local function addNumber(label, id, initial, callback, width)
        local row = rowBase(label)
        local box = inst("TextBox", {
            Size = UDim2.fromOffset(width or 64, 22), Position = UDim2.new(1, -(width or 64)-7, 0.5, -11),
            BackgroundColor3 = RS.field, BorderSizePixel = 0, Text = tostring(initial),
            TextColor3 = RS.text, Font = Enum.Font.GothamBold, TextSize = 13,
            ClearTextOnFocus = false,
        }, row)
        corner(box, 7)
        uiControls[id] = {row = row, input = box}
        box.FocusLost:Connect(function()
            local value = tonumber(box.Text)
            if value == nil then box.Text = tostring(initial); return end
            local ok, err = pcall(callback, value)
            if not ok then warn("[russo] " .. id .. ": " .. tostring(err)) end
            save()
        end)
    end
    local function addChoice(label, id, options, initial, callback)
        local row = rowBase(label)
        local b = inst("TextButton", {
            Size = UDim2.fromOffset(94, 22), Position = UDim2.new(1, -101, 0.5, -11),
            BackgroundColor3 = RS.field, BorderSizePixel = 0, Text = tostring(initial),
            TextColor3 = RS.text, Font = Enum.Font.GothamBold, TextSize = 12,
            AutoButtonColor = false,
        }, row)
        corner(b, 7)
        local index = table.find(options, initial) or 1
        uiControls[id] = {row = row, input = b}
        b.MouseButton1Click:Connect(function()
            index = index % #options + 1
            local value = options[index]
            b.Text = tostring(value)
            local ok, err = pcall(callback, value)
            if not ok then warn("[russo] " .. id .. ": " .. tostring(err)) end
            save()
        end)
    end
    local function addKey(label, id, kbKey, initial)
        local row = rowBase(label)
        local entry = KB[kbKey]
        local function currentName()
            return entry and entry.kb and entry.kb.Name or "None"
        end
        local keyBtn = inst("TextButton", {
            Size = UDim2.fromOffset(45, 22), Position = UDim2.new(1, -69, 0.5, -11),
            BackgroundColor3 = RS.field, BorderSizePixel = 0, Text = currentName(),
            TextColor3 = RS.text, Font = Enum.Font.GothamBold, TextSize = 12,
            AutoButtonColor = false,
        }, row)
        corner(keyBtn, 6)
        local clearBtn = inst("TextButton", {
            Size = UDim2.fromOffset(17, 22), Position = UDim2.new(1, -20, 0.5, -11),
            BackgroundColor3 = RS.field, BorderSizePixel = 0, Text = "X",
            TextColor3 = RS.muted, Font = Enum.Font.GothamBold, TextSize = 12,
            AutoButtonColor = false,
        }, row)
        corner(clearBtn, 6)
        uiControls[id] = {row = row, input = keyBtn, clear = clearBtn}
        keyBtn.MouseButton1Click:Connect(function()
            keyCaptureId = id
            _anyKeyListening = true
            keyBtn.Text = "..."
        end)
        clearBtn.MouseButton1Click:Connect(function()
            if entry then entry.kb = nil; entry.gp = nil end
            keyBtn.Text = "None"
            if keyCaptureId == id then keyCaptureId = nil; _anyKeyListening = false end
            save()
        end)
    end
    local function setKeyFromName(id, keyName)
        local map = {
            ["speed.laggerModeKey"] = "LaggerMode", ["speed.toggleKey"] = "CarryToggle",
            ["combat.autoBatKey"] = "AutoBat", ["combat.batBypassToggleKey"] = "BatV2",
            ["keybinds.tpDown"] = "TPFloor", ["keybinds.dropBrainrot"] = "DropBrainrot",
            ["keybinds.hideGui"] = "GuiHide",
        }
        local target = map[id]
        if not target then return end
        local keyCode = keyName ~= "None" and Enum.KeyCode[keyName] or nil
        KB[target].kb = keyCode
        KB[target].gp = nil
        local c = uiControls[id]
        if c then c.input.Text = keyName end
        save()
    end

    -- SPEED: these edit the actual variables used by the movement controller.
    section("Speed")
    addNumber("Normal Speed", "speed.normal", NS, function(v) NS = _clamp(v, 1, 500); CarrySystem:setNormalSpeed(NS) end)
    addNumber("Carry Speed", "speed.carry", CS, function(v) CS = _clamp(v, 1, 500); CarrySystem:setCarrySpeed(CS) end)
    addNumber("Lagger Normal", "speed.lagger", LAGGER_SPEED, function(v) LAGGER_SPEED = _clamp(v, 0.1, 500); CarrySystem:setLaggerSpeed(LAGGER_SPEED) end)
    addNumber("Lagger Carry", "speed.laggerCarry", LAGGER_CARRY_SPEED, function(v) LAGGER_CARRY_SPEED = _clamp(v, 0.1, 500); CarrySystem:setLaggerCarrySpeed(LAGGER_CARRY_SPEED) end)
    addKey("Lagger Mode", "speed.laggerModeKey", "LaggerMode")
    addKey("Speed Toggle", "speed.toggleKey", "CarryToggle")

    -- COMBAT: controls call the original feature functions, not copied UI toggles.
    section("Combat")
    addKey("Auto Bat", "combat.autoBatKey", "AutoBat")
    addToggle("Bat Bypass", "combat.batBypass", autoBatV2Enabled, function(on)
        if on then enableBatV2() else disableBatV2() end
    end)
    addKey("Bat Bypass Toggle", "combat.batBypassToggleKey", "BatV2")
    addToggle("Bat Counter", "combat.batCounter", batCounterEnabled, function(on)
        batCounterEnabled = on
        if on then startBatCounter() else stopBatCounter() end
    end)

    -- MECHANICS
    section("Mechanics")
    addToggle("Infinite Jump", "mechanics.infinityJump", infJumpEnabled, function(on)
        infJumpEnabled = on
        if _G.setInfJumpInternal then _G.setInfJumpInternal(on) end
    end)
    addToggle("Safe Mode", "mechanics.safeMode", antiKickEnabled, function(on)
        antiKickEnabled = on
        if on and _G.AmbitiousSafeModeForceStop then _G.AmbitiousSafeModeForceStop("SAFE MODE") end
    end)
    addChoice("Anti Ragdoll", "mechanics.antiRagdoll", {"Off", "V1", "V2"},
        antiRagdollMode == "v1" and "V1" or (antiRagdollMode == "v2" and "V2" or "Off"),
        function(v) setAntiRagdollMode(string.lower(v)) end)
    addToggle("Anti Die", "mechanics.antiDie", antiDieEnabled, function(on)
        antiDieEnabled = on
        if on then AntiDieModule.start() else AntiDieModule.stop() end
    end)
    addToggle("Auto Steal", "mechanics.autoSteal", CONFIG.AUTO_STEAL_ENABLED, function(on)
        if on then
            startAutoSteal()
        else
            stopAutoSteal()
        end
        updateProgressBarVisibility()
    end)
    addNumber("Steal Radius", "mechanics.stealRadius", CONFIG.STEAL_RANGE, function(v)
        CONFIG.STEAL_RANGE = _clamp(_floor(v + 0.5), 5, 300)
        Steal.StealRadius = CONFIG.STEAL_RANGE
    end)
    addChoice("Steal Version", "mechanics.stealVersion", AUTO_STEAL_VARIANT_NAMES,
        autoStealVariantName(autoStealVariant), function(v)
            autoStealVariant = autoStealVariantFromName(v)
        end)

    -- VISUAL
    section("Visual")
    addToggle("Player ESP", "visual.playerEsp", espEnabled, function(on) toggleESP(on) end)
    addToggle("FOV", "visual.fov", fovEnabled, function(on)
        if on then enableCustomFov() else disableCustomFov() end
    end)
    addNumber("FOV Value", "visual.fovValue", fovValue, function(v)
        fovValue = _clamp(v, 20, 120)
        if fovEnabled then
            local cam = Workspace.CurrentCamera
            if cam then cam.FieldOfView = fovValue end
        end
    end)
    addToggle("Stretch Rez", "visual.stretchRez", stretchEnabled, function(on)
        if on then enableStretch() else disableStretch() end
        stretchEnabled = on
    end)
    addToggle("Anti Lag", "visual.antiLag", antiLagEnabled, function(on)
        if on then enableAntiLag() else disableAntiLag() end
    end)

    -- UTILS
    section("Utils")
    addToggle("Bat Skin", "utils.batSkin", katanaSkinEnabled, function(on)
        katanaSkinEnabled = on
        if on then KatanaSkin.ApplySkin("KATANA") else KatanaSkin.ApplySkin("NONE") end
    end)
    local animOptions = {"Off"}
    for _, entry in ipairs(ANIM_PACK_ORDER) do
        if entry[2] ~= "Off" then table.insert(animOptions, entry[2]) end
    end
    addChoice("Anim Pack", "utils.animPack", animOptions, currentAnimPack, function(v)
        if v == "Off" then stopAnimPack() else startAnimPack(v) end
    end)
    local outfitOptions = {}
    for _, outfit in ipairs(OUTFITS) do table.insert(outfitOptions, outfit.label) end
    local currentOutfitLabel = OUTFITS[currentOutfitIndex] and OUTFITS[currentOutfitIndex].label or outfitOptions[1]
    addChoice("Outfit", "utils.outfit", outfitOptions, currentOutfitLabel, function(v)
        for i, outfit in ipairs(OUTFITS) do
            if outfit.label == v then currentOutfitIndex = i; applyOutfitByIndex(i); break end
        end
    end)
    local bgOptions = {}
    for _, bg in ipairs(BACKGROUND_IMAGES) do table.insert(bgOptions, bg.label) end
    addChoice("Background", "utils.background", bgOptions, BACKGROUND_IMAGES[backgroundIndex].label, function(v)
        for i, bg in ipairs(BACKGROUND_IMAGES) do if bg.label == v then applyBackground(i); break end end
    end)

    -- SETTINGS: scale values update the real UI/float/progress scale variables.
    section("Settings")
    addNumber("UI Scale", "settings.uiScale", 100, function(v)
        uiScaleValue = _clamp(_floor(v + 0.5), 50, 150)
        scale.Scale = uiScaleValue / 100
        if mainUIScale then mainUIScale.Scale = uiScaleValue / 100 end
    end)
    addNumber("Float Scale", "settings.floatScale", _floor(floatingButtonScale * 100), function(v)
        floatingButtonScale = _clamp(v, 50, 200) / 100
        applyFloatingButtonScale()
    end)
    addNumber("Steal Bar Scale", "settings.stealBarScale", _floor(progressBarScale * 100), function(v)
        progressBarScale = _clamp(v, 50, 200) / 100
        if pbScale then pbScale.Scale = progressBarScale end
    end)

    -- ADDITIONAL KEYBINDS
    section("Additional Keybinds")
    addKey("TP Down", "keybinds.tpDown", "TPFloor")
    addChoice("Drop Mode", "keybinds.dropMode", {"Fling", "Jump Drop"}, dropMode == 1 and "Fling" or "Jump Drop", function(v)
        dropMode = v == "Fling" and 1 or 2
    end)
    addKey("Drop Brainrot", "keybinds.dropBrainrot", "DropBrainrot")
    addKey("Hide GUI", "keybinds.hideGui", "GuiHide")

    -- Input capture for all russo keybind fields; X clears the binding.
    UIS.InputBegan:Connect(function(input, processed)
        if not keyCaptureId then return end
        if input.UserInputType ~= Enum.UserInputType.Keyboard then return end
        local name = input.KeyCode == Enum.KeyCode.Backspace and "None" or input.KeyCode.Name
        local id = keyCaptureId
        keyCaptureId = nil
        _anyKeyListening = false
        setKeyFromName(id, name)
    end)

    -- The original Clean Hub hotkey handler now controls this panel's visibility.
    _G.russoSetToggle = setToggleVisual
    _G.russoPanel = panel

    -- Make panel draggable from the title, with mouse and touch support.
    do
        local dragging, dragStart, startPos, activeInput = false, nil, nil, nil
        title.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                dragging = true; dragStart = input.Position; startPos = panel.Position
                input.Changed:Connect(function()
                    if input.UserInputState == Enum.UserInputState.End then dragging = false end
                end)
            end
        end)
        title.InputChanged:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then activeInput = input end
        end)
        UIS.InputChanged:Connect(function(input)
            if dragging and (input == activeInput or input.UserInputType == Enum.UserInputType.MouseMovement) then
                local delta = input.Position - dragStart
                panel.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
            end
        end)
    end

    -- Publish the actual runtime state accessors for later integrations.
    _G.russo = {
        Panel = panel,
        Controls = uiControls,
        SetToggle = setToggleVisual,
        Save = saveAllSettings,
    }
end
