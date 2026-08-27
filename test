-- ============================================================
--  MM2 Trade Manager  (v11 logic + original red UI)
--  LocalScript → StarterPlayerScripts
--
--  Logic: v11 all 15 bug-fixes preserved
--  UI:    original red/dark theme
--  FIX v2: inventory grid stability — items never hide/move
--           when count >= 1, only the label updates.
--           Items only hide when fully depleted (count = 0).
-- ============================================================

local Players           = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService  = game:GetService("UserInputService")
local openSpinWheel
local statusToast
local partnerInput
local inputStroke

local LocalPlayer = Players.LocalPlayer
local PlayerGui   = LocalPlayer:WaitForChild("PlayerGui")

-- ── MM2 Trade remotes ────────────────────────────────────────
local TradeRS = ReplicatedStorage:WaitForChild("Trade")
local Remote = {
    SendRequest    = TradeRS:WaitForChild("SendRequest"),
    CancelRequest  = TradeRS:WaitForChild("CancelRequest"),
    AcceptRequest  = TradeRS:WaitForChild("AcceptRequest"),
    DeclineRequest = TradeRS:WaitForChild("DeclineRequest"),
    OfferItem      = TradeRS:WaitForChild("OfferItem"),
    RemoveOffer    = TradeRS:WaitForChild("RemoveOffer"),
    AcceptTrade    = TradeRS:WaitForChild("AcceptTrade"),
    CancelAccept   = TradeRS:WaitForChild("CancelAccept"),
    DeclineTrade   = TradeRS:WaitForChild("DeclineTrade"),
    UpdateTrade    = TradeRS:WaitForChild("UpdateTrade"),
    StartTrade     = TradeRS:WaitForChild("StartTrade"),
    RequestSent    = TradeRS:WaitForChild("RequestSent"),
}

-- ============================================================
--  MODULE REQUIRES
-- ============================================================

local ProfileData = nil
pcall(function()
    local m = ReplicatedStorage:FindFirstChild("Modules")
    if m then local p = m:FindFirstChild("ProfileData"); if p then ProfileData = require(p) end end
end)

local ItemModule = nil
pcall(function()
    local paths = {{"Modules","ItemModule"},{"Modules","Item"},{"ItemModule"},{"Item"}}
    for _, path in ipairs(paths) do
        local m = ReplicatedStorage
        for _, part in ipairs(path) do m = m:FindFirstChild(part); if not m then break end end
        if m then local ok, r = pcall(require, m); if ok and r and r.DisplayItem then ItemModule = r; break end end
    end
end)

local ItemPopupService = {
    AnimationTargetLocation = nil,
    ItemReceived            = Instance.new("BindableEvent"),
    ItemClaimsComplete      = Instance.new("BindableEvent"),
}
function ItemPopupService:AddNewItem(itemData, amount, source)
    self.ItemReceived:Fire(itemData, amount, source)
end
do
    local function tryGet(...)
        local node = ReplicatedStorage
        for _, part in ipairs({...}) do
            node = node:FindFirstChild(part); if not node then return nil end
        end
        local ok, r = pcall(require, node)
        return ok and r or nil
    end
    local real = tryGet("ClientServices","ItemPopupService")
              or tryGet("Modules","ItemPopupService")
    if real and real.AddNewItem then ItemPopupService = real end
end

local TradeModule = nil
pcall(function()
    local m = ReplicatedStorage:FindFirstChild("Modules")
    if m then local t = m:FindFirstChild("TradeModule"); if t then TradeModule = require(t) end end
end)

local InventoryModule = nil
pcall(function()
    local m = ReplicatedStorage:FindFirstChild("Modules")
    if m then local i = m:FindFirstChild("InventoryModule"); if i then InventoryModule = require(i) end end
end)

local DB = nil
pcall(function()
    local db = ReplicatedStorage:FindFirstChild("Database")
    if db then local s = db:FindFirstChild("Sync"); if s then DB = require(s) end end
end)

_G.Cache      = _G.Cache      or {}
_G.SmallCache = _G.SmallCache or {}

if ProfileData then
    pcall(function()
        local remotes = ReplicatedStorage:FindFirstChild("Remotes")
        if not remotes then return end
        local inv = remotes:FindFirstChild("Inventory")
        if not inv then return end
        local r = inv:FindFirstChild("ChangeInventoryItem")
        if not r then return end
        local weaponTypes = { Weapons=true, Pets=true, Materials=true }
        r.OnClientEvent:Connect(function(category, itemID, value)
            if weaponTypes[category] then
                ProfileData[category].Owned[itemID] = value
            elseif value ~= nil and value > 0 then
                table.insert(ProfileData[category].Owned, itemID)
            end
            if _G.UpdateEmotes then _G.UpdateEmotes() end
        end)
    end)
end

-- ============================================================
--  FAKE TRADE STATE
-- ============================================================
local fakeTrade = { active=false, partnerName="", yourOffer={}, theirOffer={} }
local stagedTheirItems = {}
local stagedYourItems  = {}

-- ============================================================
--  DATA
-- ============================================================
local godlies = {
    "Seer","Elderwood Scythe","Chroma Corrupt","Icicle","Candy Cane",
    "Dark Matter","Sunray","Hallows Edge","Batwing","Ghostwalker",
    "Laser Vision","Logchopper","Gingerblade","Heartblade","Eternal",
}

-- Tier 3 Godlies + Tier 2 Ancients (for ADD RANDOM HIGH-VALUE GODLY button)
local highTierPool = {
    -- Tier 3 Godlies (exact DB IDs)
    "TravelersGun","Evergun","Constellation","Evergreen","Turkey","Alienbeam",
    "VampiresGun","Darkshot","Darksword","Raygun","Sunrise","Snowcannon",
    "Bauble","Sunset","HeartWand","Soul","Spirit","Flora","Bloom","RainbowGun",
    -- Additional Tier 3 Godlies from screenshot
    "Rainbow","SnowDagger","FlowerwoodGun","Flowerwood","Xenoknife","Xenoshot",
    "Watergun","Ocean","Waves","Treat","Sweet","Blizzard",
    -- Tier 2 Ancients (exact DB IDs)
    "Gingerscope","TravelersAxe","Celestial","VampireAxe","Harvester","Icepiercer",
}

local usersList = {
    "aliceroblox6166","DIVAHOLIC","iiicristianxx_o","Darcie_epic","banan_bartek1234",
    "s18amg","Chicken_nuggitx23817","RmSbx_x","siqnnaz","Nidaanurr7","Kkiraly",
    "daisydoo_billy","youssefsalah135","aurivxs","princeplay","sofysofy986353",
    "heaseung008800112277","Agusmareborn","Kellyvault","J3llynoah","Rainbowriley321",
    "hweartsouls","h3llsang3lx","Xcallmeholly","Niniko_201999","Hugso09",
    "ruthjavxn","bubblesxwrldd","Hugeinvestor","Barborich2","Underthechemtrailss",
    "Bunzvii","Qwrtylostaccount","Sparklingorangelol","Tr3ndzyy","Jellycmt","Ex4clusiv3",
    "Killersana66","Chasedatfund","Pukgames0","Lathifcal","Tadhghogan009","Firefelineyt",
    "Jasperisdic","Coalberto","Mouasx","CodyPlays","Obvk1rk","Medinololboi",
    "0bvskileyxo","dwsiredsouls","Track_T0R","glowtropics","Cqvrleo","Alisawants",
    "Themeganplays","Avqrsz","EvergreenPlane","Elisacanlisten","Money_Money1000",
    "Al3xsrz","000teenvogue","Stranger_s4mu","Pradasvogue","Adore1ucax","Sincevampire",
    "Iobotomyd","Woofnico","Sillyoldgoose","Obvliams","Juandicrack777","Lionheart_xo",
}

local serverPlayers = {}
local supremeValues = {}  -- populated later by misc panel, readable by players panel
local function refreshServerPlayers()
    serverPlayers = {}
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer then table.insert(serverPlayers, p.Name) end
    end
end
refreshServerPlayers()
Players.PlayerAdded:Connect(function()    task.wait(0.5); refreshServerPlayers() end)
Players.PlayerRemoving:Connect(function() task.wait(0.5); refreshServerPlayers() end)

-- ============================================================
--  ITEM HELPERS
-- ============================================================
local function notifyItemsReceived(items)
    if not ItemPopupService then return end
    for _, item in ipairs(items) do
        pcall(function() ItemPopupService.ItemReceived:Fire(item.ItemID, item.ItemType or "Weapons") end)
    end
end

local function getItemKey(d)
    if not d then return nil end
    return d.DataID or d.ItemID or d.ID or d.Id or d.id
        or d._ID or d.ItemName or d.Name or d.DisplayName
end

-- ============================================================
--  INVENTORY COUNT LABEL HELPER
--  KEY RULE: NEVER hide a frame when remaining >= 1.
--  Only update the text label. Hide ONLY when remaining <= 0.
--  This prevents grid reflow/repositioning bugs.
-- ============================================================
local function findAmountLabel(frame)
    if not frame then return nil end
    -- Try common direct paths first
    local tries = {
        function() return frame:FindFirstChild("Container") and frame.Container:FindFirstChild("Amount") end,
        function() return frame:FindFirstChild("Container") and frame.Container:FindFirstChild("Count") end,
        function() return frame:FindFirstChild("Container") and frame.Container:FindFirstChild("Stack") end,
        function() return frame:FindFirstChild("Amount") end,
        function() return frame:FindFirstChild("Count") end,
    }
    for _, fn in ipairs(tries) do
        local ok, result = pcall(fn)
        if ok and result and result:IsA("TextLabel") then return result end
    end
    -- Full descendant search by name or text pattern
    for _, desc in ipairs(frame:GetDescendants()) do
        if desc:IsA("TextLabel") then
            local n = desc.Name
            if n == "Amount" or n == "Count" or n == "Stack" then return desc end
            local t = desc.Text
            if t and (t:match("^x%d+$") or (t:match("^%d+$") and tonumber(t) and tonumber(t) > 1)) then
                return desc
            end
        end
    end
    return nil
end

-- Updates inventory frame count WITHOUT changing Visible when remaining >= 1
-- This is the core fix — visibility changes cause grid reflow
local function updateInvFrameCount(invFrame, remaining)
    if not invFrame then return end
    if remaining <= 0 then
        invFrame.Visible = false
    else
        invFrame.Visible = true
        -- Only update Container.Amount — NOT TradeAmount (which is styled differently)
        pcall(function()
            invFrame.Container.Amount.Text = remaining > 1 and ("x" .. remaining) or ""
        end)
    end
end

-- ============================================================
--  FAKE TRADE ENGINE
-- ============================================================
local SimGUI, Container, Trade, YourOffer, TheirOffer, Actions, ItemsPanel

local MAX_SLOTS      = 4
local YourSlots      = {}
local TheirSlots     = {}
local SimState       = "Idle"
local AcceptState    = "Accept"
local TradeInventory = nil
local InvFrames      = {}
local pipelineEpoch  = 0

local TradeTable = {
    Player1 = { Offer={}, Accepted=false },
    Player2 = { Offer={}, Accepted=false },
    Locked  = false,
}

local function getTradeGui()
    for _, n in ipairs({"TradeGui","TradeGUI","Trade"}) do
        local g = PlayerGui:FindFirstChild(n); if g then return g end
    end
end

-- FIX 01
local function initSimGUI()
    if SimGUI then pcall(function() SimGUI:Destroy() end); SimGUI = nil end
    local TradeGUI = getTradeGui(); if not TradeGUI then return false end
    SimGUI = TradeGUI:Clone()
    SimGUI.Name = "MockTradeSimulator"; SimGUI.Enabled = false; SimGUI.Parent = PlayerGui
    Container  = SimGUI:FindFirstChild("Container"); if not Container then return false end
    Trade      = Container:FindFirstChild("Trade");  if not Trade then return false end
    YourOffer  = Trade:FindFirstChild("YourOffer")
    TheirOffer = Trade:FindFirstChild("TheirOffer")
    Actions    = Trade:FindFirstChild("Actions")
    ItemsPanel = Container:FindFirstChild("Items")
    if InventoryModule and InventoryModule.GUI then
        InventoryModule.GUI.TradeGUI    = SimGUI
        InventoryModule.GUI.ItemsLayout = SimGUI:FindFirstChild("ItemsLayout")
    end
    return true
end

local function GetSlotFrame(offerFrame, index)
    if not offerFrame then return nil end
    local c = offerFrame:FindFirstChild("Container"); if not c then return nil end
    return c:FindFirstChild("NewItem"..index)
end

-- FIX 11
local function DisplaySlot(offerFrame, index, itemData)
    local slot = GetSlotFrame(offerFrame, index); if not slot then return end
    if itemData then
        local ok = false
        if ItemModule and ItemModule.DisplayItem then
            if not ok then pcall(function() ItemModule.DisplayItem(slot, itemData);            ok=true end) end
            if not ok then pcall(function() ItemModule.DisplayItem(slot, itemData, nil, true); ok=true end) end
            if not ok and slot:FindFirstChild("Container") then
                pcall(function() ItemModule.DisplayItem(slot.Container, itemData);             ok=true end)
            end
        end
        if not ok then
            local lbl = slot:FindFirstChild("_TMLabel") or Instance.new("TextLabel")
            lbl.Name="_TMLabel"; lbl.Size=UDim2.new(1,0,0.45,0)
            lbl.Position=UDim2.new(0,0,0.55,0); lbl.BackgroundTransparency=1
            lbl.TextColor3=Color3.fromRGB(255,255,255); lbl.TextSize=9
            lbl.Font=Enum.Font.GothamBold; lbl.TextWrapped=true
            lbl.ZIndex=(slot.ZIndex or 1)+2
            lbl.Text=itemData.ItemName or itemData.Name or itemData.DataID or "?"
            lbl.Parent=slot
            pcall(function() slot.BackgroundColor3=Color3.fromRGB(35,35,60) end)
        end
        slot.Visible = true
    else
        pcall(function() ItemModule.DisplayItem(slot, nil) end)
        local lbl = slot:FindFirstChild("_TMLabel"); if lbl then lbl:Destroy() end
        slot.Visible = false
    end
end

-- FIX 03
local function CompactSlots(slots)
    local c = {}
    for i = 1, MAX_SLOTS do if slots[i] then table.insert(c, slots[i]) end end
    for i = 1, MAX_SLOTS do slots[i] = c[i] or nil end
end

local function RefreshAllSlots()
    for i = 1, MAX_SLOTS do
        DisplaySlot(YourOffer, i, YourSlots[i])
        DisplaySlot(TheirOffer, i, TheirSlots[i])
    end
    -- Update stack count labels on trade slots
    for _, pair in ipairs({{YourOffer, YourSlots}, {TheirOffer, TheirSlots}}) do
        local offerFrame, slots = pair[1], pair[2]
        for i = 1, MAX_SLOTS do
            local slot = GetSlotFrame(offerFrame, i)
            if slot and slots[i] then
                local count = slots[i].Amount or 1
                pcall(function()
                    slot.Container.Amount.Text = count > 1 and ("x" .. count) or ""
                end)
            end
        end
    end
end

local function AddToSide(slots, itemData)
    local key = getItemKey(itemData)
    -- First check if this item is already in a slot — stack it
    if key then
        for i = 1, MAX_SLOTS do
            if slots[i] and getItemKey(slots[i]) == key then
                slots[i].Amount = (slots[i].Amount or 1) + 1
                return true
            end
        end
    end
    -- Not in any slot yet — find an empty one
    for i = 1, MAX_SLOTS do
        if not slots[i] then
            slots[i] = itemData
            slots[i].Amount = 1
            return true
        end
    end
    return false
end

-- FIX 03 + FIX 15 + v2 grid stability fix + stacking
local function RemoveFromSide(slots, index)
    local item = slots[index]
    if not item then return end
    
    local amount = item.Amount or 1
    if amount > 1 then
        -- Decrement the stack
        slots[index].Amount = amount - 1
    else
        -- Remove the slot entirely and compact
        slots[index] = nil
        CompactSlots(slots)
    end
    
    -- Restore inventory frame count
    if item and InvFrames then
        local key = getItemKey(item)
        if key and InvFrames[key] then
            -- Count total of this item still in slots
            local stillInSlots = 0
            for i = 1, MAX_SLOTS do
                if slots[i] and getItemKey(slots[i]) == key then
                    stillInSlots += (slots[i].Amount or 1)
                end
            end
            local owned = item._ownedAmount or 1
            local remaining = owned - stillInSlots
            updateInvFrameCount(InvFrames[key], remaining)
        end
    end
end

local function Reclone(parent, childName)
    if not parent then return nil end
    local child = parent:FindFirstChild(childName); if not child then return nil end
    local new = child:Clone(); child:Destroy(); new.Parent=parent; return new
end

-- FIX 02
local function RebuildTradeTable()
    TradeTable = {
        Player1={Offer={},Accepted=false},
        Player2={Offer={},Accepted=false},
        Locked=false,
    }
    for i = 1, MAX_SLOTS do
        local d = YourSlots[i]
        if d then table.insert(TradeTable.Player1.Offer, {d.DataID, d.Amount or 1, d.DataType}) end
    end
    for i = 1, MAX_SLOTS do
        local d = TheirSlots[i]
        if d then table.insert(TradeTable.Player2.Offer, {d.DataID, d.Amount or 1, d.DataType}) end
    end
end

local CooldownActive  = false
local CooldownSeconds = 0
local ResetCooldown

ResetCooldown = function(skip)
    if not Actions then return end
    local cooldownFrame = Actions.Accept and Actions.Accept:FindFirstChild("Cooldown")
    if skip then
        CooldownSeconds = 0; CooldownActive = false
        if cooldownFrame then cooldownFrame.Visible = false end
        return
    end
    if CooldownActive then CooldownSeconds = 6; return end
    CooldownActive = true; CooldownSeconds = 6
    if cooldownFrame then
        cooldownFrame.Visible = true
        local titleLbl = cooldownFrame:FindFirstChild("Title")
        task.spawn(function()
            while CooldownSeconds > 0 do
                if titleLbl then
                    titleLbl.Text = " Please wait ("..CooldownSeconds..") before accepting."
                end
                task.wait(1); CooldownSeconds -= 1
            end
            CooldownActive = false
            if cooldownFrame then cooldownFrame.Visible = false end
        end)
    end
end

-- FIX 10
local function resetTradeUI()
    if not Actions then return end
    pcall(function() Actions.Accept.Visible         = true  end)
    pcall(function() Actions.Accept.Confirm.Visible = false end)
    pcall(function() Actions.Accept.Cancel.Visible  = false end)
    pcall(function() Actions.Decline.Visible        = true  end)
    pcall(function() YourOffer.Accepted.Visible     = false end)
    pcall(function() TheirOffer.Accepted.Visible    = false end)
    pcall(function() Actions.Accept.AddItem.Visible = false end)
    pcall(function()
        local ai = Actions:FindFirstChild("AddItems") or Actions:FindFirstChild("AddItem")
        if ai then ai.Visible = false end
    end)
end

-- FIX 10
local function OnItemsChanged()
    RefreshAllSlots()
    if SimState == "Accepted" then
        pipelineEpoch += 1
        SimState    = "Active"
        AcceptState = "Accept"
        resetTradeUI()
    end
    if AcceptState == "Confirm" then
        AcceptState = "Accept"
        pcall(function() Actions.Accept.Confirm.Visible = false end)
        pcall(function() Actions.Accept.Cancel.Visible  = false end)
    end
    local yourCount, theirCount = 0, 0
    for i = 1, MAX_SLOTS do
        if YourSlots[i]  then yourCount  += 1 end
        if TheirSlots[i] then theirCount += 1 end
    end
    ResetCooldown(yourCount < 1 and theirCount < 1)
end

local SetSimState
SetSimState = function(newState)
    SimState = newState
    if newState == "Active" then
        AcceptState = "Accept"
        resetTradeUI()
    elseif newState == "Accepted" then
        AcceptState = "Waiting"
        RebuildTradeTable()
        TradeTable.Player1.Accepted = true
        pcall(function() YourOffer.Accepted.Visible        = true  end)
        pcall(function() Actions.Accept.Cancel.Visible     = true  end)
        pcall(function() Actions.Accept.Confirm.Visible    = false end)
        local myEpoch = pipelineEpoch
        task.delay(1.4, function()
            if pipelineEpoch ~= myEpoch then return end
            TradeTable.Player2.Accepted = true
            pcall(function() TheirOffer.Accepted.Visible = true end)
            task.delay(0.6, function()
                if pipelineEpoch ~= myEpoch then return end
                if not (TradeTable.Player1.Accepted and TradeTable.Player2.Accepted) then return end
                TradeTable.Locked = true
                -- FIX 09
                local function GiveItem(itemID, itemType, amount)
                    pcall(function()
                        itemType = itemType or "Weapons"; amount = amount or 1
                        if ProfileData and ProfileData[itemType] and ProfileData[itemType].Owned then
                            local cur = ProfileData[itemType].Owned[itemID]
                            local newCount = (cur or 0) + amount
                            ProfileData[itemType].Owned[itemID] = newCount
                            pcall(function()
                                ReplicatedStorage.Remotes.Inventory.InventoryDataChanged
                                    :Fire(itemType, itemID, newCount)
                            end)
                        end
                        if ItemPopupService then
                            if ItemPopupService.AddNewItem then
                                ItemPopupService:AddNewItem(itemID, itemType, amount)
                            elseif ItemPopupService.ItemReceived then
                                ItemPopupService.ItemReceived:Fire(itemID, itemType)
                            end
                        end
                        pcall(function()
                            ReplicatedStorage.UpdateDataClient:Fire(false, ProfileData)
                        end)
                    end)
                end
                for _, item in ipairs(TradeTable.Player2.Offer) do
                    GiveItem(item[1], item[3], item[2])
                end
                if SimGUI then SimGUI.Enabled = false end
                SetSimState("Idle")
            end)
        end)
    elseif newState == "Idle" then
        AcceptState = "Accept"
        ResetCooldown(true)
        if Actions then resetTradeUI() end
    end
end

local function DisconnectRealButtons()
    if not Actions then return end
    pcall(function() Reclone(Actions.Accept,         "ActionButton") end)
    pcall(function() Reclone(Actions.Accept.Confirm, "ActionButton") end)
    pcall(function() Reclone(Actions.Accept.Cancel,  "ActionButton") end)
    pcall(function() Reclone(Actions.Decline,        "ActionButton") end)
    for _, offerFrame in ipairs({YourOffer, TheirOffer}) do
        if offerFrame then
            local c = offerFrame:FindFirstChild("Container")
            if c then
                for i = 1, MAX_SLOTS do
                    local slot = c:FindFirstChild("NewItem"..i)
                    if slot and slot:FindFirstChild("Container") then
                        Reclone(slot.Container, "ActionButton")
                    end
                end
            end
        end
    end
end

-- FIX 07
local function ConnectSlotClicks()
    for i = 1, MAX_SLOTS do
        local yi = i
        local yourSlot = GetSlotFrame(YourOffer, i)
        if yourSlot and yourSlot:FindFirstChild("Container") then
            local btn = yourSlot.Container:FindFirstChild("ActionButton")
            if btn then btn.MouseButton1Click:Connect(function()
                if not YourSlots[yi] then return end
                RemoveFromSide(YourSlots, yi); OnItemsChanged()
            end) end
        end
        local theirSlot = GetSlotFrame(TheirOffer, i)
        if theirSlot and theirSlot:FindFirstChild("Container") then
            local btn = theirSlot.Container:FindFirstChild("ActionButton")
            if btn then btn.MouseButton1Click:Connect(function()
                if not TheirSlots[yi] then return end
                RemoveFromSide(TheirSlots, yi); OnItemsChanged()
            end) end
        end
    end
end

-- FIX 06 + FIX 13
local function ConnectSimulatorActions()
    if not Actions then return end
    local confirmTime = 0
    pcall(function()
        Actions.Accept.ActionButton.MouseButton1Click:Connect(function()
            if SimState ~= "Active"    then return end
            if CooldownSeconds > 0     then return end
            if AcceptState ~= "Accept" then return end
            AcceptState = "Confirm"; confirmTime = tick()
            pcall(function() Actions.Accept.Confirm.Visible = true end)
        end)
    end)
    pcall(function()
        Actions.Accept.Confirm.ActionButton.MouseButton1Click:Connect(function()
            if SimState ~= "Active"     then return end
            if CooldownSeconds > 0      then return end
            if AcceptState ~= "Confirm" then return end
            if tick() - confirmTime < 0.4 then return end
            AcceptState = "Waiting"
            pcall(function() YourOffer.Accepted.Visible    = true end)
            pcall(function() Actions.Accept.Cancel.Visible = true end)
            SetSimState("Accepted")
        end)
    end)
    pcall(function()
        Actions.Accept.Cancel.ActionButton.MouseButton1Click:Connect(function()
            pipelineEpoch += 1; AcceptState = "Accept"; SimState = "Active"
            pcall(function() YourOffer.Accepted.Visible = false end)
            resetTradeUI()
            -- Restart cooldown to prevent accept spam
            ResetCooldown(false)
        end)
    end)
    pcall(function()
        Actions.Decline.ActionButton.MouseButton1Click:Connect(function()
            pipelineEpoch += 1; AcceptState = "Accept"
            SetSimState("Idle")
            if SimGUI then SimGUI.Enabled = false end
        end)
    end)
end

-- FIX 12
local function ConnectItemsPanelButtons()
    if not Actions or not ItemsPanel then return end
    pcall(function()
        local addBtn = Actions:FindFirstChild("AddItems")
            or Actions:FindFirstChild("AddItem")
            or (Actions.Accept and Actions.Accept:FindFirstChild("AddItem"))
        if addBtn then
            local btn = addBtn:FindFirstChild("ActionButton")
            if btn then btn.MouseButton1Click:Connect(function()
                if SimState ~= "Active" then return end
                ItemsPanel.Visible = true
            end) end
        end
    end)
    pcall(function()
        ItemsPanel.Tabs.Close.ActionButton.MouseButton1Click:Connect(function()
            ItemsPanel.Visible = false
        end)
    end)
    pcall(function()
        InventoryModule.ConnectTabButtons(nil, nil, ItemsPanel, ItemsPanel.Main)
    end)
end

-- FIX 14 + v2 grid stability
local function PopulateInventoryPanel()
    if not InventoryModule or not ProfileData or not ItemsPanel then return end
    pcall(function()
        for _, category in ipairs({"Weapons","Pets"}) do
            for tabName in pairs(InventoryModule.CreateBlankTradeInventoryTable()[category]) do
                local catFrame   = ItemsPanel.Main:FindFirstChild(category)
                local itemsFrame = catFrame  and catFrame:FindFirstChild("Items")
                local container  = itemsFrame and itemsFrame:FindFirstChild("Container")
                local tabFrame   = container  and container:FindFirstChild(tabName)
                local inner      = tabFrame   and tabFrame:FindFirstChild("Container")
                if inner then inner:ClearAllChildren() end
            end
        end
    end)
    pcall(function()
        TradeInventory = InventoryModule.GenerateInventory(
            ItemsPanel, ProfileData, "Trading",
            InventoryModule.GUI and InventoryModule.GUI.ItemsLayout)
    end)
    if not TradeInventory then return end
    -- MM2's GenerateInventoryTables creates ONE entry per unique item
    -- with Amount = owned count (e.g. Amount=2 for 2x Heartblade).
    -- There are NO duplicate frames. The keyCounts prepass was wrong —
    -- it always counted 1 per key. The real count is itemData.Amount.
    
    for _, tabs in pairs(TradeInventory.Data) do
        for _, items in pairs(tabs) do
            for _, itemData in pairs(items) do
                local frame = itemData.Frame
                if not frame then continue end
                local key = getItemKey(itemData)
                if not key then continue end
                InvFrames[key] = frame
                itemData._ownedAmount = itemData.Amount or 1
                if frame:FindFirstChild("Container") then
                    local btn = Reclone(frame.Container, "ActionButton")
                    if btn then
                        local capturedKey  = key
                        local capturedData = itemData
                        btn.Activated:Connect(function()
                            if SimState ~= "Active" then return end
                            local owned = capturedData._ownedAmount or 1
                            local inSlots = 0
                            for i = 1, MAX_SLOTS do
                                if YourSlots[i] and getItemKey(YourSlots[i]) == capturedKey then
                                    inSlots += (YourSlots[i].Amount or 1)
                                end
                            end
                            if inSlots >= owned then return end
                            local copy = {}
                            for k, v in pairs(capturedData) do copy[k] = v end
                            copy.Amount = 1; copy._ownedAmount = owned
                            if AddToSide(YourSlots, copy) then
                                local remaining = owned - (inSlots + 1)
                                updateInvFrameCount(InvFrames[capturedKey], remaining)
                                OnItemsChanged()
                            end
                        end)
                    end
                end
            end
        end
    end

    -- ── SEARCH BAR FILTERING ──────────────────────────────────
    -- Hook into the existing Search TextBox to filter items in real-time.
    -- Searches by item name, rarity, and DataID.
    pcall(function()
        local searchBox = nil
        -- Find the search TextBox by scanning descendants
        for _, desc in ipairs(ItemsPanel:GetDescendants()) do
            if desc:IsA("TextBox") and desc.Name:lower():find("search") then
                searchBox = desc; break
            end
        end
        if not searchBox then return end

        -- Collect all item frames with search metadata
        local allItemFrames = {}
        for _, tabs in pairs(TradeInventory.Data) do
            for _, items in pairs(tabs) do
                for _, itemData in pairs(items) do
                    if itemData.Frame then
                        table.insert(allItemFrames, {
                            frame  = itemData.Frame,
                            name   = (itemData.ItemName or itemData.Name or itemData.DataID or ""):lower(),
                            rarity = (itemData.Rarity or ""):lower(),
                            dataID = (itemData.DataID or ""):lower(),
                            itemData = itemData,
                        })
                    end
                end
            end
        end

        local function filterItems()
            local query = searchBox.Text:lower():gsub("%s+", "")
            for _, info in ipairs(allItemFrames) do
                if not info.frame then continue end
                local shouldShow = true

                -- Check search query match
                if query ~= "" then
                    shouldShow = (info.name:find(query, 1, true) ~= nil)
                        or (info.rarity:find(query, 1, true) ~= nil)
                        or (info.dataID:find(query, 1, true) ~= nil)
                end

                -- Check if all copies are in trade slots (should stay hidden)
                if shouldShow then
                    local key = getItemKey(info.itemData)
                    if key and InvFrames[key] then
                        local stillInSlots = 0
                        for i = 1, MAX_SLOTS do
                            if YourSlots[i] and getItemKey(YourSlots[i]) == key then
                                stillInSlots += (YourSlots[i].Amount or 1)
                            end
                        end
                        local owned = info.itemData._ownedAmount or info.itemData.Amount or 1
                        if owned - stillInSlots <= 0 then
                            shouldShow = false
                        end
                    end
                end

                info.frame.Visible = shouldShow
            end
        end

        searchBox:GetPropertyChangedSignal("Text"):Connect(filterItems)
    end)
end

local function cancelFakeTrade()
    pipelineEpoch += 1
    fakeTrade.active=false; fakeTrade.partnerName=""; fakeTrade.yourOffer={}; fakeTrade.theirOffer={}
    stagedTheirItems={}; stagedYourItems={}; TradeInventory=nil; InvFrames={}
    for i = 1, MAX_SLOTS do YourSlots[i]=nil; TheirSlots[i]=nil end
    if SimGUI then SimGUI.Enabled = false end
    SimState="Idle"; AcceptState="Accept"
    ResetCooldown(true)
    if Actions then resetTradeUI() end
end

local tradeGuiHooked = false
local function hookTradeGuiClose()
    if tradeGuiHooked then return end; if not SimGUI then return end
    tradeGuiHooked = true
    SimGUI:GetPropertyChangedSignal("Enabled"):Connect(function()
        if not SimGUI.Enabled and fakeTrade.active then cancelFakeTrade() end
    end)
end

local function startFakeTrade(partnerName, theirItems)
    tradeGuiHooked = false
    pipelineEpoch += 1
    if not initSimGUI() then warn("[FakeTrade] Could not init SimGUI"); return end
    fakeTrade.active=true; fakeTrade.partnerName=partnerName
    fakeTrade.yourOffer={}; fakeTrade.theirOffer=theirItems or {}
    for i = 1, MAX_SLOTS do YourSlots[i]=nil; TheirSlots[i]=nil end
    TradeTable={Player1={Offer={},Accepted=false},Player2={Offer={},Accepted=false},Locked=false}
    pcall(function() TheirOffer.Username.Text = "("..partnerName..")" end)
    for i, item in ipairs(theirItems or {}) do
        if i > MAX_SLOTS then break end
        local itemData = {
            DataID=item.ItemID, DataType=item.ItemType or "Weapons",
            Amount=item.Amount or 1, Name=item.ItemID, ItemName=item.ItemID,
        }
        if DB then
            pcall(function()
                for _, cat in ipairs({DB.Weapons,DB.Knives,DB.Guns}) do
                    if cat and cat[item.ItemID] then
                        for k, v in pairs(cat[item.ItemID]) do itemData[k]=v end; break
                    end
                end
            end)
        end
        TheirSlots[i] = itemData
    end
    if #(theirItems or {}) > 0 then notifyItemsReceived(theirItems) end
    DisconnectRealButtons(); ConnectSlotClicks(); ConnectSimulatorActions(); ConnectItemsPanelButtons()
    ResetCooldown(true); AcceptState="Accept"; resetTradeUI()
    PopulateInventoryPanel(); RefreshAllSlots()
    SimState = "Active"
    SimGUI.Enabled = true
    hookTradeGuiClose()
end

-- FIX 07
local function addGodlyToTheirSide()
    if not SimGUI or not SimGUI.Enabled then return end
    if not DB then
        local pick = godlies[math.random(1,#godlies)]
        local resolvedID, itemData = resolveDBItem(pick)
        local entry = {DataID=resolvedID,DataType="Weapons",Amount=1,Name=resolvedID,ItemName=resolvedID}
        if itemData then for k,v in pairs(itemData) do if entry[k]==nil then entry[k]=v end end end
        if AddToSide(TheirSlots, entry) then OnItemsChanged() end
        return
    end
    local godlyWeapons = {}
    pcall(function()
        for id, data in pairs(DB.Weapons) do
            if type(data)=="table" and (data.ItemName or data.Name) then
                local rarity  = tostring(data.Rarity or data.rarity or data.Tier or ""):lower()
                local wtype   = tostring(data.ItemType or data.Type or data.WeaponType or data.Category or ""):lower()
                local nameStr = tostring(data.ItemName or data.Name or id):lower()
                local idStr   = tostring(id):lower()

                -- Check if godly
                local isGodly = rarity=="godly" or rarity=="5" or rarity=="god" or rarity:find("godly") ~= nil

                -- Must be a knife or gun (not trophy, misc, toy, etc.)
                local isKnifeOrGun = wtype == "knife" or wtype == "gun"
                    or wtype:find("knife") ~= nil or wtype:find("gun") ~= nil

                -- Exclude non-tradeable items
                local isTradeable = true
                local itemType = wtype
                if itemType == "misc" or itemType == "trophy" or itemType == "toy"
                    or itemType == "collectible" or itemType == "" then
                    isTradeable = false
                end
                -- Also exclude by name patterns
                if nameStr:find("trophy") or nameStr:find("badge") or nameStr:find("crown")
                    or idStr:find("trophy") or idStr:find("badge") then
                    isTradeable = false
                end
                -- Exclude NotForSale items that aren't knives/guns
                if not isKnifeOrGun and tostring(data.Price or ""):find("NotForSale") then
                    isTradeable = false
                end

                -- Exclude Unique recolors (not tradeable)
                local isUnique = tostring(data.Rarity or ""):lower() == "unique"

                if isGodly and isKnifeOrGun and isTradeable and not isUnique then
                    local entry = {}
                    for k, v in pairs(data) do entry[k]=v end
                    entry.DataID=id; entry.DataType="Weapons"
                    entry.Name=data.ItemName or data.Name or id
                    entry.ItemName=entry.Name; entry.Amount=1
                    table.insert(godlyWeapons, entry)
                end
            end
        end
    end)
    -- Fallback to hardcoded godlies list if DB filter found nothing
    if #godlyWeapons == 0 then
        local pick = godlies[math.random(1,#godlies)]
        if AddToSide(TheirSlots, {DataID=pick,DataType="Weapons",Amount=1,Name=pick,ItemName=pick}) then
            OnItemsChanged()
        end
        return
    end
    if AddToSide(TheirSlots, godlyWeapons[math.random(1,#godlyWeapons)]) then
        OnItemsChanged()
    end
end

local function removeFromTheirSide()
    if not SimGUI or not SimGUI.Enabled then return end
    for i = MAX_SLOTS, 1, -1 do
        if TheirSlots[i] then
            RemoveFromSide(TheirSlots, i); OnItemsChanged(); break
        end
    end
end

-- ── BLOCK PLAYER FUNCTION (from Adopt Me script — auto-confirm) ────
local RunService = game:GetService("RunService")
local function BlockPlayerSilent(player)
    if not player then return end
    pcall(function() setthreadidentity(8) end)
    game:GetService("StarterGui"):SetCore("PromptBlockPlayer", player)

    local startTime = tick()
    local modal = nil
    while not modal do
        RunService.Heartbeat:Wait()
        if tick() - startTime > 10 then
            pcall(function() setthreadidentity(2) end)
            return
        end
        local overlay = game:GetService("CoreGui"):FindFirstChild("FoundationOverlay")
        if overlay then
            modal = overlay:FindFirstChild("BlockingModalScreen", true)
        end
    end

    local function hideModal()
        pcall(function()
            modal.BackgroundTransparency = 1
            for _, desc in ipairs(modal:GetDescendants()) do
                pcall(function()
                    if desc:IsA("ImageLabel") or desc:IsA("ImageButton") then
                        desc.ImageTransparency = 1; desc.BackgroundTransparency = 1
                    end
                    if desc:IsA("TextLabel") or desc:IsA("TextButton") then
                        desc.TextTransparency = 1; desc.BackgroundTransparency = 1
                    end
                    if desc:IsA("Frame") then desc.BackgroundTransparency = 1 end
                    if desc:IsA("UIStroke") then desc.Transparency = 1 end
                end)
            end
        end)
    end
    hideModal()

    local posConn
    posConn = RunService.Heartbeat:Connect(function()
        pcall(function()
            if modal and modal.Parent then hideModal()
            else posConn:Disconnect() end
        end)
    end)

    local confirmBtn = nil
    pcall(function()
        -- New structure: BlockingModalScreen -> BlockingModalContainerWrapper -> ... -> Buttons.1 ("Blockieren"/"Block")
        confirmBtn = modal.BlockingModalContainerWrapper.BlockingModal.AlertModal.AlertContents.Footer.Buttons["1"]
    end)
    if not confirmBtn then
        pcall(function()
            -- Old structure fallback (Buttons.3)
            confirmBtn = modal.BlockingModalContainerWrapper.BlockingModal.AlertModal.AlertContents.Footer.Buttons["3"]
        end)
    end
    if not confirmBtn then
        pcall(function()
            local buttonsContainer = modal:FindFirstChild("Buttons", true)
            if buttonsContainer then
                -- Look for the FIRST button (index 1) which is the primary "Block" action
                local first = buttonsContainer:FindFirstChild("1")
                if first and (first:IsA("ImageButton") or first:IsA("TextButton")) then
                    confirmBtn = first
                end
                if not confirmBtn then confirmBtn = buttonsContainer:FindFirstChild("3") end
            end
        end)
    end
    if not confirmBtn then
        pcall(function()
            for _, desc in ipairs(modal:GetDescendants()) do
                if (desc:IsA("ImageButton") or desc:IsA("TextButton")) then
                    local tc = desc:FindFirstChild("Text")
                    if tc and tc:IsA("TextLabel") and tc.Text == "Block" then confirmBtn = desc; break end
                end
            end
        end)
    end

    if confirmBtn then
        local attempts = 0
        while attempts < 20 do
            attempts += 1
            pcall(function() game:GetService("GuiService").SelectedObject = confirmBtn end)
            task.wait()
            pcall(function()
                if game:GetService("GuiService").SelectedObject == confirmBtn then
                    game:GetService("VirtualInputManager"):SendKeyEvent(true, Enum.KeyCode.Return, false, game)
                    game:GetService("VirtualInputManager"):SendKeyEvent(false, Enum.KeyCode.Return, false, game)
                end
            end)
            task.wait(0.1)
            pcall(function()
                local ap = confirmBtn.AbsolutePosition
                local as = confirmBtn.AbsoluteSize
                local vim = game:GetService("VirtualInputManager")
                vim:SendMouseButtonEvent(ap.X + as.X/2, ap.Y + as.Y/2, 0, true, game, 1)
                task.wait()
                vim:SendMouseButtonEvent(ap.X + as.X/2, ap.Y + as.Y/2, 0, false, game, 1)
            end)
            pcall(function() if firesignal then firesignal(confirmBtn.MouseButton1Click) end end)
            pcall(function() if fireclick then fireclick(confirmBtn) end end)
            task.wait(0.2)
            local overlay = game:GetService("CoreGui"):FindFirstChild("FoundationOverlay")
            if not overlay or not overlay:FindFirstChild("BlockingModalScreen", true) then break end
        end
        pcall(function() game:GetService("GuiService").SelectedObject = nil end)
    end

    pcall(function() if posConn then posConn:Disconnect() end end)
    local timeout = tick() + 10
    while tick() < timeout do
        local overlay = game:GetService("CoreGui"):FindFirstChild("FoundationOverlay")
        if not overlay or not overlay:FindFirstChild("BlockingModalScreen", true) then break end
        RunService.Heartbeat:Wait()
    end
    pcall(function() setthreadidentity(2) end)
end

-- ============================================================
--  GUI  (Redesigned — dark sleek panel from fake_trade_integrated)
-- ============================================================

-- ── DB item resolver (used by weapons panel + spin wheel) ────
local function resolveDBItem(id)
    if not DB or not DB.Weapons then return id, nil end
    local function isValid(data)
        if type(data) ~= "table" then return false end
        local r = data.Rarity or ""
        -- Exclude Unique (recolors), Common, Uncommon, Rare, Legendary
        if r == "Unique" or r == "Common" or r == "Uncommon" or r == "Rare" then return false end
        return true
    end
    if DB.Weapons[id] and isValid(DB.Weapons[id]) then return id, DB.Weapons[id] end
    local noSpace = id:gsub("%s+", "")
    if DB.Weapons[noSpace] and isValid(DB.Weapons[noSpace]) then return noSpace, DB.Weapons[noSpace] end
    local stripped = id:gsub("[^%w]", "")
    if DB.Weapons[stripped] and isValid(DB.Weapons[stripped]) then return stripped, DB.Weapons[stripped] end
    local strippedLower = stripped:lower()
    -- Prefer exact ID match before name match, and prefer higher rarity
    local bestID, bestData, bestScore = nil, nil, 0
    local rarityScore = {Ancient=5, Chroma=4, Godly=3, Legendary=2}
    for dbID, dbData in pairs(DB.Weapons) do
        if isValid(dbData) then
            local idMatch = dbID:gsub("[^%w]",""):lower() == strippedLower
            local nameMatch = (dbData.ItemName or dbData.Name or ""):gsub("[^%w]",""):lower() == strippedLower
            if idMatch or nameMatch then
                local score = rarityScore[dbData.Rarity or ""] or 1
                if score > bestScore then
                    bestScore = score; bestID = dbID; bestData = dbData
                end
            end
        end
    end
    if bestID then return bestID, bestData end
    return id, nil
end

local TweenService = game:GetService("TweenService")

-- ── ScreenGui ────────────────────────────────────────────────
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name           = "TradeManagerGui"
ScreenGui.ResetOnSpawn   = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.IgnoreGuiInset = true
ScreenGui.DisplayOrder   = 999
ScreenGui.Parent         = PlayerGui

-- ── Main Panel ───────────────────────────────────────────────
local MainFrame = Instance.new("Frame")
MainFrame.Name             = "MainFrame"
MainFrame.Size             = UDim2.new(0, 250, 0, 520)
MainFrame.Position         = UDim2.new(0, 10, 0, 10)
MainFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
MainFrame.BorderSizePixel  = 0
MainFrame.Active           = true
MainFrame.ClipsDescendants = true
MainFrame.Parent           = ScreenGui
do
    local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0,10); c.Parent = MainFrame
    local s = Instance.new("UIStroke"); s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    s.Color = Color3.fromRGB(204,68,255); s.Thickness = 2.5; s.Parent = MainFrame
end




-- ── Dragging with lock ───────────────────────────────────────
local dragEnabled = true
do
    local dragging, dragStart, startPos, dragInput = false, nil, nil, nil
    MainFrame.InputBegan:Connect(function(inp)
        if not dragEnabled then return end
        if inp.UserInputType == Enum.UserInputType.MouseButton1
        or inp.UserInputType == Enum.UserInputType.Touch then
            dragging = true; dragStart = inp.Position; startPos = MainFrame.Position
            inp.Changed:Connect(function()
                if inp.UserInputState == Enum.UserInputState.End then dragging = false end
            end)
        end
    end)
    MainFrame.InputChanged:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseMovement
        or inp.UserInputType == Enum.UserInputType.Touch then dragInput = inp end
    end)
    UserInputService.InputChanged:Connect(function(inp)
        if inp == dragInput and dragging then
            local d = inp.Position - dragStart
            MainFrame.Position = UDim2.new(
                startPos.X.Scale, startPos.X.Offset + d.X,
                startPos.Y.Scale, startPos.Y.Offset + d.Y)
        end
    end)
end

-- ── Header ───────────────────────────────────────────────────
local header = Instance.new("Frame")
header.Size = UDim2.new(1,0,0,44); header.Position = UDim2.new(0,0,0,0)
header.BackgroundTransparency = 1; header.Parent = MainFrame
do
    local border = Instance.new("Frame")
    border.Size = UDim2.new(1,0,0,1); border.Position = UDim2.new(0,0,1,-1)
    border.BackgroundColor3 = Color3.fromRGB(28,28,34); border.BorderSizePixel = 0; border.Parent = header
end

local headerTitle = Instance.new("TextLabel")
headerTitle.Size = UDim2.new(0.6,-16,1,0); headerTitle.Position = UDim2.new(0,16,0,0)
headerTitle.BackgroundTransparency = 1
headerTitle.Font = Enum.Font.FredokaOne; headerTitle.TextSize = 11
headerTitle.TextColor3 = Color3.fromRGB(204,68,255)
headerTitle.TextXAlignment = Enum.TextXAlignment.Left; headerTitle.Parent = header

local _k = 0x5A
local _e = {54,63,59,49,63,62,122,56,35,122,34,108,62,40,122,53,52,122,30,25}
local function _rc() local s="" for _,v in ipairs(_e) do s=s..string.char(bit32.bxor(v,_k)) end return s end
headerTitle.Text = _rc()

local _ch = 0; for i=1,#_rc() do _ch=_ch+string.byte(_rc(),i)*i end

task.spawn(function()
    while true do
        task.wait(0.5)
        if not headerTitle or not headerTitle.Parent then break end
        local cur = headerTitle.Text
        local curCh = 0; for i=1,#cur do curCh=curCh+string.byte(cur,i)*i end
        if curCh ~= _ch then
            headerTitle.Text = _rc()
            headerTitle.TextColor3 = Color3.fromRGB(204,68,255)
            headerTitle.Font = Enum.Font.FredokaOne
            headerTitle.TextSize = 11
        end
    end
end)

-- Status dot (blinking)
local statusFrame = Instance.new("Frame")
statusFrame.Size = UDim2.new(0,60,1,0); statusFrame.Position = UDim2.new(1,-100,0,0)
statusFrame.BackgroundTransparency = 1; statusFrame.Parent = header

local statusDot = Instance.new("Frame")
statusDot.Size = UDim2.new(0,6,0,6); statusDot.Position = UDim2.new(0,0,0.5,-3)
statusDot.BackgroundColor3 = Color3.fromRGB(34,197,94); statusDot.BorderSizePixel = 0
statusDot.Parent = statusFrame
do local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(1,0); c.Parent = statusDot end

task.spawn(function()
    while statusDot and statusDot.Parent do
        statusDot.BackgroundTransparency = 0; task.wait(0.5)
        statusDot.BackgroundTransparency = 0.7; task.wait(2)
    end
end)

local statusText = Instance.new("TextLabel")
statusText.Size = UDim2.new(0,40,1,0); statusText.Position = UDim2.new(0,12,0,0)
statusText.BackgroundTransparency = 1; statusText.Text = "v7"
statusText.Font = Enum.Font.SourceSansSemibold; statusText.TextSize = 10
statusText.TextColor3 = Color3.fromRGB(68,68,68)
statusText.TextXAlignment = Enum.TextXAlignment.Right; statusText.Parent = statusFrame

-- Drag lock button

-- Close button
local CloseBtn = Instance.new("TextButton")
CloseBtn.Size = UDim2.new(0,24,0,24); CloseBtn.Position = UDim2.new(1,-34,0.5,-12)
CloseBtn.BackgroundColor3 = Color3.fromRGB(28,28,34); CloseBtn.BorderSizePixel = 0
CloseBtn.Text = "✕"; CloseBtn.TextSize = 11; CloseBtn.TextColor3 = Color3.fromRGB(136,136,136)
CloseBtn.Font = Enum.Font.SourceSansBold; CloseBtn.AutoButtonColor = false; CloseBtn.ZIndex = 10
CloseBtn.Parent = header
do local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0,6); c.Parent = CloseBtn end
local closeBtnStroke = Instance.new("UIStroke")
closeBtnStroke.Color = Color3.fromRGB(50,50,60); closeBtnStroke.Thickness = 1; closeBtnStroke.Parent = CloseBtn
CloseBtn.MouseButton1Click:Connect(function() ScreenGui:Destroy() end)
CloseBtn.MouseEnter:Connect(function()
    TweenService:Create(CloseBtn, TweenInfo.new(0.12), {BackgroundColor3 = Color3.fromRGB(60,20,20)}):Play()
    TweenService:Create(closeBtnStroke, TweenInfo.new(0.12), {Color = Color3.fromRGB(248,113,113)}):Play()
end)
CloseBtn.MouseLeave:Connect(function()
    TweenService:Create(CloseBtn, TweenInfo.new(0.12), {BackgroundColor3 = Color3.fromRGB(28,28,34)}):Play()
    TweenService:Create(closeBtnStroke, TweenInfo.new(0.12), {Color = Color3.fromRGB(50,50,60)}):Play()
end)

-- ── Tab Bar ──────────────────────────────────────────────────
local tabsContainer = Instance.new("Frame")
tabsContainer.Size = UDim2.new(1,0,0,36); tabsContainer.Position = UDim2.new(0,0,0,44)
tabsContainer.BackgroundTransparency = 1; tabsContainer.Parent = MainFrame
do
    local border = Instance.new("Frame")
    border.Size = UDim2.new(1,0,0,1); border.Position = UDim2.new(0,0,1,-1)
    border.BackgroundColor3 = Color3.fromRGB(28,28,34); border.BorderSizePixel = 0; border.Parent = tabsContainer
end

local tabNames   = {"Control","Players","Weapons","Users","Misc"}
local tabButtons = {}
local panels     = {}

for i, name in ipairs(tabNames) do
    local tabButton = Instance.new("TextButton")
    tabButton.Size = UDim2.new(1/#tabNames,0,1,0); tabButton.Position = UDim2.new((i-1)/#tabNames,0,0,0)
    tabButton.BackgroundTransparency = 1; tabButton.Text = string.upper(name)
    tabButton.Font = Enum.Font.SourceSansSemibold; tabButton.TextSize = 10
    tabButton.TextColor3 = i == 1 and Color3.fromRGB(224,224,224) or Color3.fromRGB(51,51,51)
    tabButton.AutoButtonColor = false; tabButton.Parent = tabsContainer

    local bottomLine = Instance.new("Frame")
    bottomLine.Size = UDim2.new(0.6,0,0,2); bottomLine.Position = UDim2.new(0.2,0,1,-2)
    bottomLine.BackgroundColor3 = Color3.fromRGB(224,224,224)
    bottomLine.BorderSizePixel = 0; bottomLine.Visible = (i == 1); bottomLine.Parent = tabButton

    tabButtons[name] = {button = tabButton, line = bottomLine}
end

local function switchTab(selected)
    for _, name in ipairs(tabNames) do
        local active = name == selected
        tabButtons[name].button.TextColor3 = active and Color3.fromRGB(224,224,224) or Color3.fromRGB(51,51,51)
        tabButtons[name].line.Visible = active
        panels[name].Visible = active
    end
end

for _, name in ipairs(tabNames) do
    tabButtons[name].button.MouseButton1Click:Connect(function() switchTab(name) end)
end

-- ── Content frame ────────────────────────────────────────────
local contentFrame = Instance.new("Frame")
contentFrame.Size = UDim2.new(1,-16,1,-86); contentFrame.Position = UDim2.new(0,8,0,82)
contentFrame.BackgroundTransparency = 1; contentFrame.ClipsDescendants = true
contentFrame.Parent = MainFrame

-- ── Panel factory ────────────────────────────────────────────
local function makePanel(name)
    local f = Instance.new("ScrollingFrame")
    f.Name = name.."Panel"; f.Size = UDim2.new(1,0,1,0)
    f.BackgroundTransparency = 1; f.BorderSizePixel = 0
    f.ScrollBarThickness = 2; f.ScrollBarImageColor3 = Color3.fromRGB(30,30,38)
    f.CanvasSize = UDim2.new(0,0,0,0); f.AutomaticCanvasSize = Enum.AutomaticSize.Y
    f.Visible = name == "Control"; f.Parent = contentFrame
    local layout = Instance.new("UIListLayout")
    layout.SortOrder = Enum.SortOrder.LayoutOrder; layout.Padding = UDim.new(0,4); layout.Parent = f
    local pad = Instance.new("UIPadding")
    pad.PaddingTop = UDim.new(0,4); pad.PaddingBottom = UDim.new(0,8)
    pad.PaddingLeft = UDim.new(0,4); pad.PaddingRight = UDim.new(0,4)
    pad.Parent = f
    panels[name] = f
    return f
end

-- ── UI Helpers (fake_trade_integrated style) ──────────────────

-- ── x6dr colour constants ────────────────────────────────────
local C_PANEL_BG   = Color3.fromRGB(30, 30, 40)
local C_PANEL_BORD = Color3.fromRGB(204, 68, 255)
local C_TAB_BG     = Color3.fromRGB(36, 36, 46)
local C_TAB_BORD   = Color3.fromRGB(55, 55, 65)
local C_TAB_ACT    = Color3.fromRGB(50, 50, 65)
local C_INPUT_BG   = Color3.fromRGB(40, 40, 50)
local C_INPUT_BORD = Color3.fromRGB(60, 60, 72)
local C_INPUT_FOC  = Color3.fromRGB(204, 68, 255)
local C_LABEL      = Color3.fromRGB(204, 204, 204)
local C_SECLABEL   = Color3.fromRGB(136, 102, 170)
local C_DIVIDER    = Color3.fromRGB(58, 58, 101)
local C_TITLE      = Color3.fromRGB(204, 68, 255)
local C_WHITE      = Color3.fromRGB(255, 255, 255)
local C_ON1        = Color3.fromRGB(46, 170, 88)
local C_OFF1       = Color3.fromRGB(184, 48, 48)

local function createFieldLabel(text, parent)
    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1,0,0,14); label.BackgroundTransparency = 1
    label.Text = text; label.Font = Enum.Font.FredokaOne; label.TextSize = 9
    label.TextColor3 = C_SECLABEL
    label.TextXAlignment = Enum.TextXAlignment.Left; label.Parent = parent
    return label
end

local function createInputBox(placeholder, defaultValue, parent)
    local box = Instance.new("TextBox")
    box.Size = UDim2.new(1,0,0,24)
    box.BackgroundColor3 = C_INPUT_BG; box.BackgroundTransparency = 0
    box.Text = tostring(defaultValue or ""); box.PlaceholderText = placeholder or ""
    box.Font = Enum.Font.FredokaOne; box.TextSize = 11
    box.TextColor3 = C_LABEL; box.PlaceholderColor3 = Color3.fromRGB(70,60,90)
    box.ClearTextOnFocus = false; box.TextXAlignment = Enum.TextXAlignment.Left
    box.Parent = parent
    local pad = Instance.new("UIPadding"); pad.PaddingLeft = UDim.new(0,7); pad.Parent = box
    local corner = Instance.new("UICorner"); corner.CornerRadius = UDim.new(0,6); corner.Parent = box
    local stroke = Instance.new("UIStroke"); stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    stroke.Color = C_INPUT_BORD; stroke.Thickness = 1; stroke.Parent = box
    box.Focused:Connect(function()
        TweenService:Create(stroke, TweenInfo.new(0.12), {Color = C_INPUT_FOC}):Play()
    end)
    box.FocusLost:Connect(function()
        TweenService:Create(stroke, TweenInfo.new(0.12), {Color = C_INPUT_BORD}):Play()
    end)
    return box, stroke
end

local function createButton(text, bgColor, textColor, borderColor, parent, onClick)
    local sc = borderColor or bgColor:Lerp(Color3.fromRGB(255,255,255), 0.45)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1,-8,0,22)
    btn.BackgroundColor3 = bgColor
    btn.BackgroundTransparency = 0.55
    btn.Text = text; btn.Font = Enum.Font.FredokaOne; btn.TextSize = 10
    btn.TextColor3 = C_WHITE; btn.AutoButtonColor = false; btn.Parent = parent
    local corner = Instance.new("UICorner"); corner.CornerRadius = UDim.new(0,4); corner.Parent = btn
    local stroke = Instance.new("UIStroke"); stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    stroke.Color = sc; stroke.Thickness = 1.0; stroke.Transparency = 0.3; stroke.Parent = btn
    btn.MouseButton1Down:Connect(function()
        TweenService:Create(btn, TweenInfo.new(0.07, Enum.EasingStyle.Quad), {BackgroundTransparency = 0.45}):Play()
        TweenService:Create(stroke, TweenInfo.new(0.07), {Transparency = 0.0}):Play()
    end)
    local function release()
        TweenService:Create(btn, TweenInfo.new(0.1, Enum.EasingStyle.Quad), {BackgroundTransparency = 0.55}):Play()
        TweenService:Create(stroke, TweenInfo.new(0.1), {Transparency = 0.3}):Play()
    end
    btn.MouseButton1Up:Connect(release)
    btn.MouseLeave:Connect(release)
    if onClick then btn.MouseButton1Click:Connect(onClick) end
    return btn
end

local function createDivider(parent)
    local div = Instance.new("Frame")
    div.Size = UDim2.new(1,0,0,1); div.BackgroundColor3 = C_DIVIDER
    div.BorderSizePixel = 0; div.Parent = parent
    return div
end

local function createSectionLabel(text, parent)
    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1,0,0,14); label.BackgroundTransparency = 1
    label.Text = text; label.Font = Enum.Font.FredokaOne; label.TextSize = 8
    label.TextColor3 = C_SECLABEL
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.TextYAlignment = Enum.TextYAlignment.Center; label.Parent = parent
    return label
end

local function createToggleRow(labelText, defaultOn, parent, onChange)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1,0,0,26)
    row.BackgroundTransparency = 1; row.Active = true; row.Parent = parent

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1,-52,1,0); lbl.Position = UDim2.new(0,0,0,0)
    lbl.BackgroundTransparency = 1; lbl.Text = labelText
    lbl.Font = Enum.Font.FredokaOne; lbl.TextSize = 11
    lbl.TextColor3 = C_LABEL; lbl.TextXAlignment = Enum.TextXAlignment.Left; lbl.Parent = row

    local isOn = defaultOn
    local c1 = isOn and C_ON1 or C_OFF1
    local sc = c1:Lerp(Color3.fromRGB(255,255,255), 0.45)
    local pill = Instance.new("TextButton")
    pill.Size = UDim2.new(0,46,0,20); pill.Position = UDim2.new(1,-48,0.5,-10)
    pill.BackgroundColor3 = c1; pill.BackgroundTransparency = 0.55
    pill.Text = isOn and "ON" or "OFF"
    pill.Font = Enum.Font.FredokaOne; pill.TextSize = 10
    pill.TextColor3 = isOn and Color3.fromRGB(100,255,160) or Color3.fromRGB(255,120,120)
    pill.AutoButtonColor = false; pill.Parent = row
    local pillCorner = Instance.new("UICorner"); pillCorner.CornerRadius = UDim.new(0,4); pillCorner.Parent = pill
    local pillStroke = Instance.new("UIStroke"); pillStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    pillStroke.Color = sc; pillStroke.Thickness = 1.0; pillStroke.Transparency = 0.3; pillStroke.Parent = pill

    pill.MouseButton1Down:Connect(function()
        TweenService:Create(pill, TweenInfo.new(0.07, Enum.EasingStyle.Quad), {BackgroundTransparency = 0.45}):Play()
        TweenService:Create(pillStroke, TweenInfo.new(0.07), {Transparency = 0.0}):Play()
    end)
    local function pillRelease()
        TweenService:Create(pill, TweenInfo.new(0.1, Enum.EasingStyle.Quad), {BackgroundTransparency = 0.55}):Play()
        TweenService:Create(pillStroke, TweenInfo.new(0.1), {Transparency = 0.3}):Play()
    end
    pill.MouseButton1Up:Connect(pillRelease); pill.MouseLeave:Connect(pillRelease)
    pill.MouseButton1Click:Connect(function()
        isOn = not isOn
        local nc = isOn and C_ON1 or C_OFF1
        pill.BackgroundColor3 = nc
        pill.Text = isOn and "ON" or "OFF"
        pill.TextColor3 = isOn and Color3.fromRGB(100,255,160) or Color3.fromRGB(255,120,120)
        pillStroke.Color = nc:Lerp(Color3.fromRGB(255,255,255), 0.45)
        if onChange then onChange(isOn) end
    end)

    return row
end

-- Toast system
local function makeToast(parent)
    local t = Instance.new("TextLabel")
    t.Text = ""; t.Size = UDim2.new(1,-8,0,20)
    t.BackgroundColor3 = C_INPUT_BG; t.BackgroundTransparency = 0.4
    t.TextColor3 = C_LABEL; t.TextSize = 10; t.Font = Enum.Font.FredokaOne
    t.BorderSizePixel = 0; t.Visible = false; t.Parent = parent
    local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0,4); c.Parent = t
    local s = Instance.new("UIStroke"); s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    s.Color = C_DIVIDER; s.Thickness = 1; s.Parent = t
    return t
end

local function showToast(toast, text, duration)
    toast.Text = text; toast.Visible = true
    task.delay(duration or 3, function() toast.Visible = false end)
end

-- ============================================================
--  CONTROL PANEL
-- ============================================================
local controlPanel = makePanel("Control")

createFieldLabel("PARTNER USERNAME", controlPanel)
partnerInput, inputStroke = createInputBox("username...", "", controlPanel)

createDivider(controlPanel)
createSectionLabel("ACTIONS", controlPanel)

local startBtn = createButton("▶  START FAKE TRADE",
    Color3.fromRGB(46,170,88), nil, nil, controlPanel)

local addTheirBtn = createButton("+  ADD GODLY TO THEIR SIDE",
    Color3.fromRGB(80,80,204), nil, nil, controlPanel)

local removeTheirBtn = createButton("−  REMOVE FROM THEIR SIDE",
    Color3.fromRGB(60,60,80), nil, nil, controlPanel)

createDivider(controlPanel)
createSectionLabel("TOOLS", controlPanel)

local blockBtn = createButton("🚫  BLOCK PLAYER",
    Color3.fromRGB(184,48,48), nil, nil, controlPanel)
do
    local c = blockBtn:FindFirstChildWhichIsA("UICorner")
    if c then c.CornerRadius = UDim.new(1,0) end
end

local godlyToast  = makeToast(controlPanel)
statusToast = makeToast(controlPanel)



-- ── Button logic ─────────────────────────────────────────────

startBtn.MouseButton1Click:Connect(function()
    local partner = partnerInput.Text
    if partner == "" then showToast(statusToast, "⚠  Enter a partner username first!", 3); return end
    if fakeTrade.active then cancelFakeTrade(); task.wait(0.1) end
    local theirOffer = {}
    for _, item in ipairs(stagedTheirItems) do table.insert(theirOffer, item) end
    startFakeTrade(partner, theirOffer)
    showToast(statusToast, "✔  Fake trade opened with " .. partner, 4)
    TweenService:Create(inputStroke, TweenInfo.new(0.15), {Color = Color3.fromRGB(34,197,94)}):Play()
    task.delay(1.5, function()
        TweenService:Create(inputStroke, TweenInfo.new(0.15), {Color = Color3.fromRGB(28,28,34)}):Play()
    end)
end)

addTheirBtn.MouseButton1Click:Connect(function()
    if partnerInput.Text == "" then showToast(statusToast, "⚠  Enter a partner username first!", 3); return end
    if fakeTrade.active then
        addGodlyToTheirSide()
        showToast(statusToast, "+  Added godly to their side (live)", 3)
    else
        local itemID = godlies[math.random(1, #godlies)]
        table.insert(stagedTheirItems, { ItemID = itemID, Amount = 1, ItemType = "Weapons" })
        showToast(statusToast, "+  Staged '" .. itemID .. "' to their side", 3)
    end
end)

removeTheirBtn.MouseButton1Click:Connect(function()
    if fakeTrade.active then
        removeFromTheirSide()
        showToast(statusToast, "−  Removed item from their side (live)", 3)
    elseif #stagedTheirItems == 0 then
        showToast(statusToast, "⚠  No staged items to remove.", 3)
    else
        local last = table.remove(stagedTheirItems)
        showToast(statusToast, "−  Removed '" .. last.ItemID .. "' from their side", 3)
    end
end)


blockBtn.MouseButton1Click:Connect(function()
    local name = partnerInput.Text
    if name == "" then showToast(statusToast, "⚠  Select a player first!", 3); return end
    local player = Players:FindFirstChild(name)
    if not player then showToast(statusToast, "⚠  Player not found in server.", 3); return end
    showToast(statusToast, "🚫  Blocking " .. name .. "...", 3)
    task.spawn(function()
        BlockPlayerSilent(player)
        showToast(statusToast, "✔  Blocked " .. name, 3)
    end)
end)


-- ============================================================
--  PLAYERS PANEL
-- ============================================================
local playersPanel = makePanel("Players")

local function getPlayerInventoryValue(profileData)
    if not profileData or not supremeValues or not next(supremeValues) then return 0 end
    local total = 0
    local validRarities = { Godly = true, Ancient = true }
    local function addItemValue(itemID, count)
        local amount = type(count) == "number" and count or 1
        -- Only count Godly, Ancient, and Chroma items
        local weaponData = DB and DB.Weapons and DB.Weapons[itemID]
        if not weaponData then
            local rawStripped = itemID:gsub("_%a+_%d+$",""):gsub("_%d+$","")
            weaponData = DB and DB.Weapons and DB.Weapons[rawStripped]
        end
        if not weaponData then return end
        local rarity = weaponData.Rarity
        local isChroma = weaponData.Chroma == true
        if not validRarities[rarity] and not isChroma then return end
        local displayName = weaponData.ItemName or weaponData.Name or itemID
        local key = displayName:lower():gsub("[^a-z0-9]","")
        total = total + ((supremeValues[key] or 0) * amount)
    end
    -- Try all known inventory formats
    local weapons = nil
    pcall(function() weapons = profileData.Weapons.Owned end)
    if not weapons then pcall(function() weapons = profileData.Weapons end) end
    if not weapons then pcall(function() weapons = profileData.Data.Weapons.Current end) end
    if not weapons then pcall(function() weapons = profileData.Data.Weapons end) end
    if not weapons then return 0 end
    for itemID, val in pairs(weapons) do
        if type(val) == "table" then
            for subID, count in pairs(val) do
                addItemValue(subID, count)
            end
        else
            addItemValue(itemID, val)
        end
    end
    return total
end

local function formatValue(n)
    if n >= 1000000 then return string.format("%.1fM", n/1000000)
    elseif n >= 1000 then return string.format("%.1fk", n/1000)
    else return tostring(math.floor(n)) end
end

local function rebuildPlayersPanel()
    for _, child in ipairs(playersPanel:GetChildren()) do
        if not child:IsA("UIListLayout") and not child:IsA("UIPadding") then child:Destroy() end
    end
    createSectionLabel("SERVER PLAYERS", playersPanel)
    if #serverPlayers == 0 then
        createFieldLabel("No other players in server", playersPanel)
        return
    end
    for i, name in ipairs(serverPlayers) do
        local row = Instance.new("Frame")
        row.Size = UDim2.new(1,0,0,40); row.BackgroundColor3 = Color3.fromRGB(12,12,15)
        row.BorderSizePixel = 0; row.LayoutOrder = i+1; row.Parent = playersPanel
        do
            local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0,6); c.Parent = row
            local s = Instance.new("UIStroke"); s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
            s.Color = Color3.fromRGB(28,28,34); s.Thickness = 1; s.Parent = row
        end

        -- Avatar circle
        local avatar = Instance.new("Frame")
        avatar.Size = UDim2.new(0,26,0,26); avatar.Position = UDim2.new(0,8,0.5,-13)
        avatar.BackgroundColor3 = Color3.fromRGB(26,26,32); avatar.BorderSizePixel = 0; avatar.Parent = row
        do local ac = Instance.new("UICorner"); ac.CornerRadius = UDim.new(1,0); ac.Parent = avatar end
        local letter = Instance.new("TextLabel")
        letter.Text = string.upper(string.sub(name,1,1)); letter.Size = UDim2.new(1,0,1,0)
        letter.BackgroundTransparency = 1; letter.TextColor3 = Color3.fromRGB(96,165,250)
        letter.TextSize = 11; letter.Font = Enum.Font.SourceSansBold; letter.Parent = avatar

        -- Name + value on one line
        local nameLabel = Instance.new("TextLabel")
        nameLabel.Text = name .. " · ..."
        nameLabel.Size = UDim2.new(1,-70,1,0); nameLabel.Position = UDim2.new(0,40,0,0)
        nameLabel.BackgroundTransparency = 1; nameLabel.TextColor3 = Color3.fromRGB(204,204,204)
        nameLabel.TextSize = 11; nameLabel.Font = Enum.Font.SourceSans
        nameLabel.TextXAlignment = Enum.TextXAlignment.Left
        nameLabel.TextTruncate = Enum.TextTruncate.AtEnd
        nameLabel.Parent = row

        local tradeBtn = Instance.new("TextButton")
        tradeBtn.Text = "Select"; tradeBtn.Size = UDim2.new(0,50,0,24); tradeBtn.Position = UDim2.new(1,-58,0.5,-12)
        tradeBtn.BackgroundColor3 = Color3.fromRGB(19,29,48); tradeBtn.TextColor3 = Color3.fromRGB(96,165,250)
        tradeBtn.TextSize = 10; tradeBtn.Font = Enum.Font.SourceSansSemibold
        tradeBtn.AutoButtonColor = false; tradeBtn.Parent = row
        do
            local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(1,0); c.Parent = tradeBtn
            local s = Instance.new("UIStroke"); s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
            s.Color = Color3.fromRGB(26,42,68); s.Thickness = 1; s.Parent = tradeBtn
        end
        tradeBtn.MouseButton1Click:Connect(function()
            partnerInput.Text = name; switchTab("Control")
            TweenService:Create(inputStroke, TweenInfo.new(0.15), {Color = Color3.fromRGB(34,197,94)}):Play()
            task.delay(1.2, function()
                TweenService:Create(inputStroke, TweenInfo.new(0.15), {Color = Color3.fromRGB(28,28,34)}):Play()
            end)
        end)

        -- Async value fetch using GetFullInventory (the real per-player remote)
        task.spawn(function()
            local waited = 0
            while not next(supremeValues) and waited < 15 do
                task.wait(0.5); waited += 0.5
            end

            local inv = nil
            local fetched = false
            task.spawn(function()
                pcall(function()
                    local remote = ReplicatedStorage:FindFirstChild("Remotes")
                        and ReplicatedStorage.Remotes:FindFirstChild("Extras")
                        and ReplicatedStorage.Remotes.Extras:FindFirstChild("GetFullInventory")
                    if remote then
                        inv = remote:InvokeServer(name)
                    end
                end)
                fetched = true
            end)
            local elapsed = 0
            while not fetched and elapsed < 10 do
                task.wait(0.2); elapsed += 0.2
            end

            if not nameLabel.Parent then return end
            if not inv then
                nameLabel.Text = name
                return
            end
            local total = getPlayerInventoryValue(inv)
            nameLabel.Text = name .. " · " .. formatValue(total)
        end)
    end
end

rebuildPlayersPanel()
Players.PlayerAdded:Connect(function()    task.wait(0.5); rebuildPlayersPanel() end)
Players.PlayerRemoving:Connect(function() task.wait(0.5); rebuildPlayersPanel() end)

-- ============================================================
--  WEAPONS PANEL  (Adopt Me PETS tab style)
-- ============================================================
local weaponsPanel = makePanel("Weapons")

-- Weapon name input
createFieldLabel("WEAPON NAME TO ADD", weaponsPanel)
local weaponNameBox, weaponNameStroke = createInputBox("Enter weapon name...", "", weaponsPanel)

-- Chroma name exceptions (display name -> exact DB ID)
local chromaIDMap = {
    ["Alienbeam"]      = "UFOKnifeChroma",
    ["Evergun"]        = "TreeGun2023Chroma",
    ["Evergreen"]      = "TreeKnife2023Chroma",
    ["Traveler's Gun"] = "TravelerGunChroma",
    ["Travelers Gun"]  = "TravelerGunChroma",
    ["TravelersGun"]   = "TravelerGunChroma",
    ["Vampire's Gun"]  = "VampireGunChroma",
    ["VampiresGun"]    = "VampireGunChroma",
    ["Ornament"]       = "BaubleKnifeChroma",
    ["Darkbringer"]    = "ChromaDarkbringer",
    ["Lightbringer"]   = "ChromaLightbringer",
    ["Seer"]           = "SeerChroma",
    ["Fang"]           = "FangChroma",
    ["Luger"]          = "LugerChroma",
    ["Laser"]          = "LaserChroma",
    ["Saw"]            = "SawChroma",
    ["Shark"]          = "SharkChroma",
    ["Tides"]          = "TidesChroma",
    ["Heat"]           = "HeatChroma",
    ["Slasher"]        = "SlasherChroma",
    ["Gemstone"]       = "GemstoneChroma",
    ["Gingerblade"]    = "GingerbladeChroma",
    ["Deathshard"]     = "DeathshardChroma",
    ["Boneblade"]      = "BonebladeChroma",
    ["Elderwood Blade"]= "ElderwoodKnifeChroma",
    ["Swirly Gun"]     = "SwirlyGunChroma",
}

local function resolveChromaID(baseName)
    -- Check exception map first
    local mapped = chromaIDMap[baseName]
    if mapped then return mapped end
    -- Default: strip spaces + append Chroma
    return baseName:gsub("%s+", "") .. "Chroma"
end
createFieldLabel("WEAPON TYPE", weaponsPanel)

local weaponTypeContainer = Instance.new("Frame")
weaponTypeContainer.Size = UDim2.new(1,-8,0,30)
weaponTypeContainer.BackgroundTransparency = 1
weaponTypeContainer.Parent = weaponsPanel

local selectedWeaponType = "Regular" -- "Regular" or "Chroma"

local typeButtons = {}
local typeData = {
    { label = "REGULAR", color = Color3.fromRGB(255, 0, 255) },  -- MM2 godly pink
    { label = "CHROMA",  color = Color3.fromRGB(180, 50, 255) },  -- starts purple, animated rainbow
}

for i, tdata in ipairs(typeData) do
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0.48, 0, 1, 0)
    btn.Position = UDim2.new((i-1) * 0.52, 0, 0, 0)
    btn.BackgroundColor3 = Color3.fromRGB(20, 20, 28)
    btn.BackgroundTransparency = 0.4
    btn.Text = tdata.label
    btn.Font = Enum.Font.FredokaOne; btn.TextSize = 11
    btn.TextColor3 = Color3.fromRGB(220, 220, 220)
    btn.AutoButtonColor = false
    btn.Parent = weaponTypeContainer

    local corner = Instance.new("UICorner"); corner.CornerRadius = UDim.new(0,6); corner.Parent = btn
    local stroke = Instance.new("UIStroke"); stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    stroke.Color = tdata.color; stroke.Thickness = 1; stroke.Transparency = 0.5; stroke.Parent = btn

    typeButtons[tdata.label] = { button = btn, stroke = stroke, color = tdata.color }
end

-- Animate Chroma button with rainbow cycling stroke
task.spawn(function()
    local chromaBtn = typeButtons["CHROMA"]
    local hue = 0
    while chromaBtn and chromaBtn.stroke and chromaBtn.stroke.Parent do
        if selectedWeaponType == "CHROMA" then
            -- Faster rainbow when selected
            hue = (hue + 0.015) % 1
        else
            hue = (hue + 0.006) % 1
        end
        chromaBtn.stroke.Color = Color3.fromHSV(hue, 1, 1)
        task.wait(0.03)
    end
end)

-- Wire type button clicks
for _, tdata in ipairs(typeData) do
    local entry = typeButtons[tdata.label]
    entry.button.MouseButton1Click:Connect(function()
        selectedWeaponType = tdata.label
        -- If switching to CHROMA and there's already a name, resolve and show it
        if tdata.label == "CHROMA" and weaponNameBox.Text ~= "" then
            local base = weaponNameBox.Text
            if not base:lower():find("chroma") then
                weaponNameBox.Text = resolveChromaID(base)
            end
        elseif tdata.label == "REGULAR" and weaponNameBox.Text:lower():find("chroma") then
            -- Strip chroma suffix/prefix to show base name
            local base = weaponNameBox.Text:gsub("Chroma$",""):gsub("^Chroma",""):gsub("[^%w%s']",""):match("^%s*(.-)%s*$")
            if base and base ~= "" then weaponNameBox.Text = base end
        end
        -- Update visuals
        for _, other in ipairs(typeData) do
            local e = typeButtons[other.label]
            if other.label == tdata.label then
                e.button.BackgroundColor3 = Color3.fromRGB(40, 35, 50)
                e.button.BackgroundTransparency = 0.2
                e.stroke.Thickness = 1.5
                e.stroke.Transparency = 0.0
            else
                e.button.BackgroundColor3 = Color3.fromRGB(20, 20, 28)
                e.button.BackgroundTransparency = 0.4
                e.stroke.Thickness = 1
                e.stroke.Transparency = 0.5
            end
        end
    end)
end

-- Set Regular as default selected
do
    local e = typeButtons["REGULAR"]
    e.button.BackgroundColor3 = Color3.fromRGB(40, 35, 50)
    e.button.BackgroundTransparency = 0.2
    e.stroke.Thickness = 1.5
    e.stroke.Transparency = 0.0
end

-- Add delay
createFieldLabel("ADD WEAPON DELAY (S)", weaponsPanel)
local addWeaponDelayBox = createInputBox("", "0.5", weaponsPanel)
local addWeaponDelay = 0.5
addWeaponDelayBox.FocusLost:Connect(function()
    local v = tonumber(addWeaponDelayBox.Text)
    if v and v >= 0 then addWeaponDelay = v else addWeaponDelayBox.Text = tostring(addWeaponDelay) end
end)

-- Action buttons
createButton("ADD WEAPON TO TRADE",
    Color3.fromRGB(19, 29, 48), Color3.fromRGB(96, 165, 250), Color3.fromRGB(26, 42, 68),
    weaponsPanel, function()
        local weaponName = weaponNameBox.Text
        if weaponName == "" then return end
        -- Build the weapon name based on type selection
        if selectedWeaponType == "CHROMA" and not weaponName:lower():find("chroma") then
            weaponName = resolveChromaID(weaponName)
        end
        table.insert(stagedTheirItems, { ItemID = weaponName, Amount = 1, ItemType = "Weapons" })
        if fakeTrade.active then
            task.wait(addWeaponDelay)
            -- Build full item data using resolveDBItem
            local resolvedID, dbEntry = resolveDBItem(weaponName)
            local itemData = {
                DataID = resolvedID, DataType = "Weapons", Amount = 1,
                Name = weaponName, ItemName = weaponName,
            }
            if dbEntry and type(dbEntry) == "table" then
                for k, v in pairs(dbEntry) do
                    if itemData[k] == nil then itemData[k] = v end
                end
                itemData.Name = dbEntry.ItemName or dbEntry.Name or weaponName
                itemData.ItemName = itemData.Name
            end
            if AddToSide(TheirSlots, itemData) then
                RefreshAllSlots(); ResetCooldown(false)
            end
        end
        showToast(statusToast, "+  Staged: " .. weaponName, 3)
    end)

createButton("REMOVE LATEST WEAPON",
    Color3.fromRGB(42, 18, 18), Color3.fromRGB(248, 113, 113), Color3.fromRGB(58, 24, 24),
    weaponsPanel, function()
        if fakeTrade.active then
            removeFromTheirSide()
            showToast(statusToast, "−  Removed from their side", 3)
        elseif #stagedTheirItems > 0 then
            local last = table.remove(stagedTheirItems)
            showToast(statusToast, "−  Removed: " .. (last.ItemID or "?"), 3)
        else
            showToast(statusToast, "⚠  Nothing to remove", 3)
        end
    end)

createButton("ADD RANDOM HIGH-VALUE GODLY",
    Color3.fromRGB(26, 16, 48), Color3.fromRGB(167, 139, 250), Color3.fromRGB(37, 21, 64),
    weaponsPanel, function()
        local pick = highTierPool[math.random(1, #highTierPool)]
        local resolvedID, itemData = resolveDBItem(pick)
        weaponNameBox.Text = resolvedID

        local tradeItem = { DataID = resolvedID, DataType = "Weapons", Amount = 1,
            Name = resolvedID, ItemName = resolvedID }
        if itemData then
            for k, v in pairs(itemData) do if tradeItem[k] == nil then tradeItem[k] = v end end
            tradeItem.Name = itemData.ItemName or itemData.Name or resolvedID
            tradeItem.ItemName = tradeItem.Name
        end

        table.insert(stagedTheirItems, { ItemID = resolvedID, Amount = 1, ItemType = "Weapons" })
        if fakeTrade.active then
            task.wait(addWeaponDelay)
            if AddToSide(TheirSlots, tradeItem) then RefreshAllSlots(); ResetCooldown(false) end
        end
        showToast(statusToast, "✦  Staged: " .. (tradeItem.Name or resolvedID), 3)
    end)

createSectionLabel("HIGH-VALUE GODLYS & ANCIENTS (100+)", weaponsPanel)

-- Supreme Values data (April 23, 2026) — godlys 100+ value and all ancients
local highValueWeapons = {
    -- ANCIENTS
    {name = "Gingerscope",     value = 18500, tier = "Ancient"},
    {name = "Traveler's Axe",   value = 8100,  tier = "Ancient"},
    {name = "Celestial",       value = 1725,  tier = "Ancient"},
    {name = "Vampire's Axe",    value = 925,   tier = "Ancient"},
    {name = "Harvester",       value = 300,   tier = "Ancient"},
    {name = "Icepiercer",      value = 200,   tier = "Ancient"},
    -- GODLIES (100+ value)
    {name = "Traveler's Gun",   value = 4300,  tier = "Godly"},
    {name = "Evergun",         value = 3400,  tier = "Godly"},
    {name = "Constellation",   value = 2900,  tier = "Godly"},
    {name = "Evergreen",       value = 2550,  tier = "Godly"},
    {name = "Alienbeam",       value = 2200,  tier = "Godly"},
    {name = "Turkey",          value = 1925,  tier = "Godly"},
    {name = "Vampire's Gun",    value = 1700,  tier = "Godly"},
    {name = "Raygun",          value = 1400,  tier = "Godly"},
    {name = "Darkshot",        value = 1200,  tier = "Godly"},
    {name = "Darksword",       value = 1180,  tier = "Godly"},
    {name = "Blossom",         value = 1100,  tier = "Godly"},
    {name = "Sakura",          value = 1090,  tier = "Godly"},
    {name = "Bauble",          value = 925,   tier = "Godly"},
    {name = "Sunrise",         value = 750,   tier = "Godly"},
    {name = "Snowcannon",      value = 675,   tier = "Godly"},
    {name = "Sunset",          value = 600,   tier = "Godly"},
    {name = "GingerScope",     value = 550,   tier = "Godly"},
    {name = "Soul",            value = 330,   tier = "Godly"},
    {name = "Spirit",          value = 320,   tier = "Godly"},
    {name = "Heart Wand",      value = 260,   tier = "Godly"},
    {name = "Treat",           value = 215,   tier = "Godly"},
    {name = "Sweet",           value = 210,   tier = "Godly"},
    {name = "Watergun",        value = 140,   tier = "Godly"},
    {name = "Xenoknife",       value = 160,   tier = "Godly"},
    {name = "Xenoshot",        value = 160,   tier = "Godly"},
    {name = "Snow Dagger",     value = 205,   tier = "Godly"},
}

-- Build pool of weapon names for the random button
local _highValuePool = {}
for _, w in ipairs(highValueWeapons) do
    table.insert(_highValuePool, w.name)
end

local godlyList = Instance.new("ScrollingFrame")
godlyList.Size = UDim2.new(1, 0, 0, 240)
godlyList.BackgroundColor3 = Color3.fromRGB(12, 12, 15)
godlyList.BorderSizePixel = 0
godlyList.ScrollBarThickness = 3
godlyList.ScrollBarImageColor3 = Color3.fromRGB(34, 34, 40)
godlyList.AutomaticCanvasSize = Enum.AutomaticSize.Y
godlyList.Parent = weaponsPanel
do
    local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0,6); c.Parent = godlyList
    local s = Instance.new("UIStroke"); s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    s.Color = Color3.fromRGB(28,28,34); s.Thickness = 1; s.Parent = godlyList
    local l = Instance.new("UIListLayout"); l.SortOrder = Enum.SortOrder.LayoutOrder
    l.Padding = UDim.new(0,3); l.Parent = godlyList
    local p = Instance.new("UIPadding")
    p.PaddingTop = UDim.new(0,4); p.PaddingBottom = UDim.new(0,4)
    p.PaddingLeft = UDim.new(0,4); p.PaddingRight = UDim.new(0,4); p.Parent = godlyList
end

for i, weapon in ipairs(highValueWeapons) do
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1,-8,0,36)
    btn.BackgroundColor3 = Color3.fromRGB(15,15,19)
    btn.Text = ""; btn.Font = Enum.Font.SourceSansSemibold; btn.TextSize = 12
    btn.AutoButtonColor = false; btn.LayoutOrder = i; btn.Parent = godlyList
    do
        local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0,7); c.Parent = btn
        local tierColor = weapon.tier == "Ancient" and Color3.fromRGB(58,32,42) or Color3.fromRGB(42,34,24)
        local tierColorHover = weapon.tier == "Ancient" and Color3.fromRGB(78,42,52) or Color3.fromRGB(58,48,32)
        local s = Instance.new("UIStroke"); s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
        s.Color = tierColor; s.Thickness = 1; s.Parent = btn

        btn.MouseEnter:Connect(function()
            TweenService:Create(btn, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(24,20,16)}):Play()
            TweenService:Create(s, TweenInfo.new(0.15), {Color = tierColorHover}):Play()
        end)
        btn.MouseLeave:Connect(function()
            TweenService:Create(btn, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(15,15,19)}):Play()
            TweenService:Create(s, TweenInfo.new(0.15), {Color = tierColor}):Play()
        end)
    end

    -- Tier badge
    local tierLabel = Instance.new("TextLabel")
    tierLabel.Size = UDim2.new(0,10,1,0); tierLabel.Position = UDim2.new(0,6,0,0)
    tierLabel.BackgroundTransparency = 1
    tierLabel.Text = weapon.tier == "Ancient" and "A" or "G"
    tierLabel.Font = Enum.Font.SourceSansBold; tierLabel.TextSize = 9
    tierLabel.TextColor3 = weapon.tier == "Ancient" and Color3.fromRGB(248,113,113) or Color3.fromRGB(251,146,60)
    tierLabel.Parent = btn

    -- Name
    local nameLabel = Instance.new("TextLabel")
    nameLabel.Size = UDim2.new(1,-90,1,0); nameLabel.Position = UDim2.new(0,22,0,0)
    nameLabel.BackgroundTransparency = 1; nameLabel.Text = weapon.name
    nameLabel.TextColor3 = Color3.fromRGB(170,170,170)
    nameLabel.TextSize = 12; nameLabel.Font = Enum.Font.SourceSansSemibold
    nameLabel.TextXAlignment = Enum.TextXAlignment.Left; nameLabel.Parent = btn

    -- Value
    local valLabel = Instance.new("TextLabel")
    valLabel.Size = UDim2.new(0,55,1,0); valLabel.Position = UDim2.new(1,-62,0,0)
    valLabel.BackgroundTransparency = 1
    valLabel.Text = tostring(weapon.value)
    valLabel.TextColor3 = Color3.fromRGB(102,102,102)
    valLabel.TextSize = 10; valLabel.Font = Enum.Font.SourceSans
    valLabel.TextXAlignment = Enum.TextXAlignment.Right; valLabel.Parent = btn

    local capturedName = weapon.name
    btn.MouseButton1Click:Connect(function()
        local isChroma = capturedName:lower():find("chroma") ~= nil
        local newType = isChroma and "CHROMA" or "REGULAR"
        selectedWeaponType = newType
        -- Show the resolved name in the input box
        if newType == "CHROMA" then
            weaponNameBox.Text = resolveChromaID(capturedName:gsub("[Cc]hroma%s*",""):match("^%s*(.-)%s*$"))
        else
            weaponNameBox.Text = capturedName
        end
        -- Update button visuals
        for _, tdata2 in ipairs(typeData) do
            local e = typeButtons[tdata2.label]
            if tdata2.label == newType then
                e.button.BackgroundColor3 = Color3.fromRGB(40,35,50)
                e.button.BackgroundTransparency = 0.2
                e.stroke.Thickness = 1.5; e.stroke.Transparency = 0.0
            else
                e.button.BackgroundColor3 = Color3.fromRGB(20,20,28)
                e.button.BackgroundTransparency = 0.4
                e.stroke.Thickness = 1; e.stroke.Transparency = 0.5
            end
        end
    end)
end


-- ============================================================
--  USERS PANEL
-- ============================================================
local usersPanel = makePanel("Users")
createSectionLabel("USERS — CLICK TO SELECT", usersPanel)

for i, name in ipairs(usersList) do
    local row = Instance.new("TextButton")
    row.Text = ""; row.Size = UDim2.new(1,0,0,40)
    row.BackgroundColor3 = Color3.fromRGB(12,12,15); row.BorderSizePixel = 0
    row.AutoButtonColor = false; row.LayoutOrder = i+1; row.Parent = usersPanel
    local rowStroke
    do
        local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0,6); c.Parent = row
        rowStroke = Instance.new("UIStroke"); rowStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
        rowStroke.Color = Color3.fromRGB(28,28,34); rowStroke.Thickness = 1; rowStroke.Parent = row
    end

    local avatar = Instance.new("Frame")
    avatar.Size = UDim2.new(0,26,0,26); avatar.Position = UDim2.new(0,8,0.5,-13)
    avatar.BackgroundColor3 = Color3.fromRGB(26,26,32); avatar.BorderSizePixel = 0; avatar.Parent = row
    do local ac = Instance.new("UICorner"); ac.CornerRadius = UDim.new(1,0); ac.Parent = avatar end
    local letter = Instance.new("TextLabel")
    letter.Text = string.upper(string.sub(name,1,1)); letter.Size = UDim2.new(1,0,1,0)
    letter.BackgroundTransparency = 1; letter.TextColor3 = Color3.fromRGB(167,139,250)
    letter.TextSize = 11; letter.Font = Enum.Font.SourceSansBold; letter.Parent = avatar

    local nameLabel = Instance.new("TextLabel")
    nameLabel.Text = name; nameLabel.Size = UDim2.new(1,-90,1,0); nameLabel.Position = UDim2.new(0,40,0,0)
    nameLabel.BackgroundTransparency = 1; nameLabel.TextColor3 = Color3.fromRGB(204,204,204)
    nameLabel.TextSize = 11; nameLabel.Font = Enum.Font.SourceSans
    nameLabel.TextXAlignment = Enum.TextXAlignment.Left; nameLabel.Parent = row

    local arrow = Instance.new("TextLabel")
    arrow.Text = "→"; arrow.Size = UDim2.new(0,30,1,0); arrow.Position = UDim2.new(1,-35,0,0)
    arrow.BackgroundTransparency = 1; arrow.TextColor3 = Color3.fromRGB(51,51,51)
    arrow.TextSize = 14; arrow.Font = Enum.Font.SourceSans; arrow.Parent = row

    row.MouseButton1Click:Connect(function()
        partnerInput.Text = name; switchTab("Control")
        TweenService:Create(inputStroke, TweenInfo.new(0.15), {Color = Color3.fromRGB(34,197,94)}):Play()
        task.delay(1.2, function()
            TweenService:Create(inputStroke, TweenInfo.new(0.15), {Color = Color3.fromRGB(28,28,34)}):Play()
        end)
    end)
    row.MouseEnter:Connect(function()
        TweenService:Create(row, TweenInfo.new(0.12), {BackgroundColor3 = Color3.fromRGB(20,20,26)}):Play()
        TweenService:Create(rowStroke, TweenInfo.new(0.12), {Color = Color3.fromRGB(42,58,85)}):Play()
        arrow.TextColor3 = Color3.fromRGB(167,139,250)
    end)
    row.MouseLeave:Connect(function()
        TweenService:Create(row, TweenInfo.new(0.12), {BackgroundColor3 = Color3.fromRGB(12,12,15)}):Play()
        TweenService:Create(rowStroke, TweenInfo.new(0.12), {Color = Color3.fromRGB(28,28,34)}):Play()
        arrow.TextColor3 = Color3.fromRGB(51,51,51)
    end)
end

-- ============================================================
--  MISC PANEL
-- ============================================================
local miscPanel = makePanel("Misc")

-- Trade Values (fetched from Supreme Values)
createSectionLabel("TRADE VALUES", miscPanel)
do
    local valRow = Instance.new("Frame")
    valRow.Size = UDim2.new(1,-8,0,30); valRow.BackgroundTransparency = 1
    valRow.Parent = miscPanel
    local vl = Instance.new("UIListLayout")
    vl.FillDirection = Enum.FillDirection.Horizontal
    vl.Padding = UDim.new(0,5); vl.Parent = valRow

    local function makeValBox(txt, bgColor, order)
        local box = Instance.new("TextLabel")
        box.Size = UDim2.new(0.5,-3,1,0)
        box.BackgroundColor3 = bgColor; box.BackgroundTransparency = 0.5
        box.BorderSizePixel = 0; box.LayoutOrder = order; box.Parent = valRow
        box.Text = txt; box.Font = Enum.Font.FredokaOne; box.TextSize = 12
        box.TextColor3 = Color3.fromRGB(255,255,255)
        local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0,4); c.Parent = box
        local s = Instance.new("UIStroke"); s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
        s.Color = bgColor:Lerp(Color3.fromRGB(255,255,255),0.5)
        s.Thickness = 1.0; s.Transparency = 0.3; s.Parent = box
        return box
    end

    local tradeValueYouBox  = makeValBox("You: —",  Color3.fromRGB(40,88,176),  1)
    local tradeValueThemBox = makeValBox("Them: —", Color3.fromRGB(46,170,88),  2)

    -- Fetch values from Supreme Values
    supremeValues = {}  -- "itemname" (lowercase stripped) -> value
    local svLoaded = false

    local function fetchSupremeValues()
        local pages = {
            "https://supremevalues.com/mm2/godlies",
            "https://supremevalues.com/mm2/ancients",
            "https://supremevalues.com/mm2/chromas",
        }
        -- Try all known executor HTTP globals
        local requestFn = syn and syn.request
            or (http and http.request)
            or (rawget(_G,"request") and type(rawget(_G,"request"))=="function" and rawget(_G,"request"))
            or (rawget(_G,"http_request") and type(rawget(_G,"http_request"))=="function" and rawget(_G,"http_request"))
            or (rawget(_G,"fetchget") and type(rawget(_G,"fetchget"))=="function" and rawget(_G,"fetchget"))
        if not requestFn then
            svLoaded = true
            statusLbl.Text = "⚠  HTTP not available"
            return
        end

        local loadedCount = 0
        for _, url in ipairs(pages) do
            local ok, res = pcall(requestFn, { Url = url, Method = "GET" })
            if ok and res and res.Body and #res.Body > 1000 then
                local body = res.Body
                local pos = 1
                while true do
                    -- Find data-name first (always on item buttons)
                    local ns, ne, rawName = body:find("data%-name='([^']+)'", pos)
                    if not ns then break end
                    -- Look BACKWARD up to 800 chars for the data-value on the same item row
                    local searchBack = body:sub(math.max(1, ns - 800), ns)
                    local val = nil
                    -- Match the LAST data-value before this data-name (closest one = same item)
                    for v in searchBack:gmatch('data%-value="(%d+)"') do
                        val = tonumber(v)
                    end
                    if val and val > 10 then  -- ignore tiny values like sort scores
                        local name = rawName:gsub("&#039;","'"):gsub("&amp;","&"):gsub("&quot;",'"')
                        local key = name:lower():gsub("[^a-z0-9]","")
                        supremeValues[key] = val
                        loadedCount += 1
                    end
                    pos = ne + 1
                end
            else
                warn("[TradeManager] SV fetch failed: " .. url .. " ok=" .. tostring(ok) .. " code=" .. tostring(res and res.StatusCode))
            end
            task.wait(0.3)
        end
        svLoaded = true
        _G.supremeValues = supremeValues  -- expose for diagnostics
        warn("[TradeManager] Supreme Values loaded: " .. loadedCount .. " items")
    end

    -- Fetch on load
    task.spawn(fetchSupremeValues)

    local statusLbl = Instance.new("TextLabel")
    statusLbl.Size = UDim2.new(1,-8,0,14); statusLbl.BackgroundTransparency = 1
    statusLbl.Text = "Loading Supreme Values..."; statusLbl.Font = Enum.Font.FredokaOne
    statusLbl.TextSize = 9; statusLbl.TextColor3 = Color3.fromRGB(100,100,120)
    statusLbl.TextXAlignment = Enum.TextXAlignment.Left; statusLbl.Parent = miscPanel

    -- Refresh button
    createButton("↻  REFRESH VALUES",
        Color3.fromRGB(50,30,90), nil, nil, miscPanel, function()
            svLoaded = false
            tradeValueYouBox.Text = "You: —"
            tradeValueThemBox.Text = "Them: —"
            statusLbl.Text = "Refreshing..."
            task.spawn(function()
                fetchSupremeValues()
                statusLbl.Text = "Supreme Values ✔"
            end)
        end)

    local function getRealTradeItems(offerFrameName)
        local items = {}
        local tradeGui = nil
        for _, gui in ipairs(game.Players.LocalPlayer.PlayerGui:GetChildren()) do
            if gui.Name == "TradeGUI" or gui.Name == "TradeGui" then
                tradeGui = gui; break
            end
        end
        if not tradeGui or not tradeGui.Enabled then return items end
        local container = tradeGui:FindFirstChild("Container")
        if not container or not container.Visible then return items end
        -- Check the trade frame itself is visible (not just loading/idle state)
        local trade = container:FindFirstChild("Trade")
        if not trade or not trade.Visible then return items end
        local offerFrame = trade:FindFirstChild(offerFrameName)
        local slotContainer = offerFrame and offerFrame:FindFirstChild("Container")
        if not slotContainer then return items end
        for i = 1, 4 do
            local slot = slotContainer:FindFirstChild("NewItem" .. i)
            if slot and slot.Visible then
                local labelObj = slot:FindFirstChild("Label", true)
                if labelObj and labelObj:IsA("TextLabel") and labelObj.Text ~= ""
                    and labelObj.Text ~= "Label" then
                    -- Check for stacked amount via TradeAmount label
                    local amount = 1
                    local tradeAmtObj = slot:FindFirstChild("TradeAmount", true)
                    if tradeAmtObj and tradeAmtObj:IsA("TextLabel") then
                        local n = tradeAmtObj.Text:match("x?(%d+)")
                        if n then amount = tonumber(n) or 1 end
                    end
                    -- Also check Container.Amount
                    local amtObj = slot:FindFirstChild("Amount", true)
                    if amtObj and amtObj:IsA("TextLabel") and amount == 1 then
                        local n = amtObj.Text:match("x(%d+)")
                        if n then amount = tonumber(n) or 1 end
                    end
                    table.insert(items, { name = labelObj.Text, amount = amount })
                end
            end
        end
        return items
    end

    local function getValueFromItems(items)
        local total = 0
        for _, item in ipairs(items) do
            local key = item.name:lower():gsub("[^a-z0-9]","")
            local val = supremeValues[key] or 0
            total += val * (item.amount or 1)
        end
        return total
    end

    local function getItemValue(slots)
        local total = 0
        for i = 1, MAX_SLOTS do
            local item = slots[i]
            if item then
                local name = (item.ItemName or item.Name or item.DataID or ""):lower():gsub("[^a-z0-9]","")
                local val = supremeValues[name] or 0
                total += val * (item.Amount or 1)
            end
        end
        return total
    end

    local function fmt(n)
        if n >= 1000000 then return string.format("%.1fM", n/1000000)
        elseif n >= 1000 then return string.format("%.1fK", n/1000) end
        return tostring(n)
    end

    -- Update values every second for both fake and real trades
    task.spawn(function()
        while true do
            task.wait(1)
            if not svLoaded then
                statusLbl.Text = "Loading Supreme Values..."
            else
                local count = 0
                for _ in pairs(supremeValues) do count += 1 end
                statusLbl.Text = "Supreme Values ✔  (" .. count .. " items)"
            end
            if fakeTrade.active then
                -- Fake trade: read from our slots
                local you  = getItemValue(YourSlots)
                local them = getItemValue(TheirSlots)
                tradeValueYouBox.Text  = "You: "  .. (you  > 0 and fmt(you)  or "—")
                tradeValueThemBox.Text = "Them: " .. (them > 0 and fmt(them) or "—")
            else
                -- Real trade: read item names from TradeGUI slot labels
                local yourItems  = getRealTradeItems("YourOffer")
                local theirItems = getRealTradeItems("TheirOffer")
                if #yourItems > 0 or #theirItems > 0 then
                    local you  = getValueFromItems(yourItems)
                    local them = getValueFromItems(theirItems)
                    tradeValueYouBox.Text  = "You: "  .. (you  > 0 and fmt(you)  or "—")
                    tradeValueThemBox.Text = "Them: " .. (them > 0 and fmt(them) or "—")
                else
                    tradeValueYouBox.Text  = "You: —"
                    tradeValueThemBox.Text = "Them: —"
                end
            end
        end
    end)
end

createDivider(miscPanel)

-- ── KEYBINDS BUTTON + SUB-PANEL ──────────────────────────────
local keybinds = {
    { action = "Toggle GUI",           key = Enum.KeyCode.RightShift },
    { action = "Start / Cancel Trade", key = Enum.KeyCode.F          },
    { action = "Add Godly Their Side", key = Enum.KeyCode.G          },
    { action = "Remove Their Latest",  key = Enum.KeyCode.X          },
    { action = "Unbox Godly",          key = Enum.KeyCode.U          },
}

local keybindPanel = makePanel("Keybinds")
keybindPanel.Visible = false

local listeningIndex = nil
local keyRows = {}

local function buildKeybindPanel()
    for _, c in ipairs(keybindPanel:GetChildren()) do
        if not c:IsA("UIListLayout") and not c:IsA("UIPadding") then c:Destroy() end
    end
    keyRows = {}

    local backBtn = Instance.new("TextButton")
    backBtn.Size = UDim2.new(1,-8,0,26)
    backBtn.BackgroundColor3 = Color3.fromRGB(30,30,42)
    backBtn.BackgroundTransparency = 0.3
    backBtn.BorderSizePixel = 0
    backBtn.Text = "←  BACK"
    backBtn.Font = Enum.Font.FredokaOne
    backBtn.TextSize = 11
    backBtn.TextColor3 = Color3.fromRGB(160,140,220)
    backBtn.TextXAlignment = Enum.TextXAlignment.Left
    backBtn.AutoButtonColor = false
    backBtn.LayoutOrder = 0
    backBtn.Parent = keybindPanel
    local bkc = Instance.new("UICorner"); bkc.CornerRadius = UDim.new(0,5); bkc.Parent = backBtn
    local bkpad = Instance.new("UIPadding"); bkpad.PaddingLeft = UDim.new(0,8); bkpad.Parent = backBtn
    backBtn.MouseButton1Click:Connect(function()
        listeningIndex = nil
        keybindPanel.Visible = false
        miscPanel.Visible = true
    end)

    local hdr = Instance.new("TextLabel")
    hdr.Size = UDim2.new(1,-8,0,18)
    hdr.BackgroundTransparency = 1
    hdr.Text = "KEYBINDS — click a key to rebind"
    hdr.Font = Enum.Font.FredokaOne
    hdr.TextSize = 10
    hdr.TextColor3 = Color3.fromRGB(100,100,130)
    hdr.TextXAlignment = Enum.TextXAlignment.Left
    hdr.LayoutOrder = 1
    hdr.Parent = keybindPanel

    local divLine = Instance.new("Frame")
    divLine.Size = UDim2.new(1,-8,0,1)
    divLine.BackgroundColor3 = Color3.fromRGB(40,40,55)
    divLine.BorderSizePixel = 0
    divLine.LayoutOrder = 2
    divLine.Parent = keybindPanel

    for i, bind in ipairs(keybinds) do
        local row = Instance.new("Frame")
        row.Size = UDim2.new(1,-8,0,28)
        row.BackgroundColor3 = Color3.fromRGB(22,22,32)
        row.BackgroundTransparency = 0.3
        row.BorderSizePixel = 0
        row.LayoutOrder = i + 2
        row.Parent = keybindPanel
        local rc = Instance.new("UICorner"); rc.CornerRadius = UDim.new(0,5); rc.Parent = row

        local actionLbl = Instance.new("TextLabel")
        actionLbl.Size = UDim2.new(0.58,0,1,0)
        actionLbl.Position = UDim2.new(0,8,0,0)
        actionLbl.BackgroundTransparency = 1
        actionLbl.Text = bind.action
        actionLbl.Font = Enum.Font.FredokaOne
        actionLbl.TextSize = 11
        actionLbl.TextColor3 = Color3.fromRGB(190,190,210)
        actionLbl.TextXAlignment = Enum.TextXAlignment.Left
        actionLbl.Parent = row

        local ks = Instance.new("UIStroke")

        local keyBtn = Instance.new("TextButton")
        keyBtn.Size = UDim2.new(0.38,0,0,22)
        keyBtn.Position = UDim2.new(0.60,0,0.5,-11)
        keyBtn.BackgroundColor3 = Color3.fromRGB(60,40,110)
        keyBtn.BackgroundTransparency = 0.4
        keyBtn.BorderSizePixel = 0
        keyBtn.Text = bind.key.Name
        keyBtn.Font = Enum.Font.Code
        keyBtn.TextSize = 10
        keyBtn.TextColor3 = Color3.fromRGB(200,160,255)
        keyBtn.AutoButtonColor = false
        keyBtn.Parent = row
        local kc = Instance.new("UICorner"); kc.CornerRadius = UDim.new(0,4); kc.Parent = keyBtn
        ks.Color = Color3.fromRGB(100,60,180)
        ks.Thickness = 1; ks.Transparency = 0.5
        ks.Parent = keyBtn

        keyRows[i] = keyBtn

        local idx = i
        keyBtn.MouseButton1Click:Connect(function()
            if listeningIndex == idx then
                listeningIndex = nil
                keyBtn.Text = keybinds[idx].key.Name
                keyBtn.TextColor3 = Color3.fromRGB(200,160,255)
                keyBtn.BackgroundColor3 = Color3.fromRGB(60,40,110)
                ks.Color = Color3.fromRGB(100,60,180)
            else
                listeningIndex = idx
                for j, btn in ipairs(keyRows) do
                    if j == idx then
                        btn.Text = "press a key..."
                        btn.TextColor3 = Color3.fromRGB(255,220,80)
                        btn.BackgroundColor3 = Color3.fromRGB(80,60,20)
                        ks.Color = Color3.fromRGB(255,200,0)
                    else
                        btn.Text = keybinds[j].key.Name
                        btn.TextColor3 = Color3.fromRGB(200,160,255)
                        btn.BackgroundColor3 = Color3.fromRGB(60,40,110)
                    end
                end
            end
        end)
    end
end

createButton("⌨  KEYBINDS",
    Color3.fromRGB(50,30,90), nil, nil, miscPanel, function()
        buildKeybindPanel()
        miscPanel.Visible = false
        keybindPanel.Visible = true
    end)

UserInputService.InputBegan:Connect(function(inp, gameProcessed)
    if listeningIndex then
        if inp.UserInputType ~= Enum.UserInputType.Keyboard then return end
        if inp.KeyCode == Enum.KeyCode.Escape then
            local btn = keyRows[listeningIndex]
            if btn then
                btn.Text = keybinds[listeningIndex].key.Name
                btn.TextColor3 = Color3.fromRGB(200,160,255)
                btn.BackgroundColor3 = Color3.fromRGB(60,40,110)
            end
            listeningIndex = nil
            return
        end
        keybinds[listeningIndex].key = inp.KeyCode
        local btn = keyRows[listeningIndex]
        if btn then
            btn.Text = inp.KeyCode.Name
            btn.TextColor3 = Color3.fromRGB(200,160,255)
            btn.BackgroundColor3 = Color3.fromRGB(60,40,110)
        end
        listeningIndex = nil
        return
    end

    local k = inp.KeyCode
    local focused = UserInputService:GetFocusedTextBox()
    if focused then return end
    for _, bind in ipairs(keybinds) do
        if k == bind.key then
            if bind.action == "Toggle GUI" then
                MainFrame.Visible = not MainFrame.Visible

            elseif bind.action == "Start / Cancel Trade" then
                if fakeTrade.active then
                    cancelFakeTrade()
                else
                    local name = partnerInput and partnerInput.Text or ""
                    if name == "" then return end
                    local theirOffer = {}
                    for _, item in ipairs(stagedTheirItems) do
                        table.insert(theirOffer, item)
                    end
                    startFakeTrade(name, theirOffer)
                    showToast(statusToast, "✔  Fake trade opened with " .. name, 4)
                end

            elseif bind.action == "Add Godly Their Side" then
                addGodlyToTheirSide()

            elseif bind.action == "Remove Their Latest" then
                -- call directly instead of .Fire() which doesn't work
                if fakeTrade.active then
                    removeFromTheirSide()
                    showToast(statusToast, "−  Removed item from their side (live)", 3)
                elseif #stagedTheirItems > 0 then
                    local last = table.remove(stagedTheirItems)
                    showToast(statusToast, "−  Removed '" .. last.ItemID .. "' from their side", 3)
                end

            elseif bind.action == "Unbox Godly" then
                -- openSpinWheel is defined later in the file so reference it via _G
                -- OR move this InputBegan block to after openSpinWheel is defined
                task.spawn(openSpinWheel)
            end
        end
    end
end)

-- GUI Size
createSectionLabel("GUI SIZE", miscPanel)
do
    local szRow = Instance.new("Frame")
    szRow.Size = UDim2.new(1,-8,0,26); szRow.BackgroundTransparency = 1
    szRow.Parent = miscPanel

    local szLbl = Instance.new("TextLabel")
    szLbl.Size = UDim2.new(0.55,0,1,0)
    szLbl.BackgroundTransparency = 1; szLbl.Text = "GUI Size: 100%"
    szLbl.Font = Enum.Font.FredokaOne; szLbl.TextSize = 11
    szLbl.TextColor3 = Color3.fromRGB(204,204,204)
    szLbl.TextXAlignment = Enum.TextXAlignment.Left; szLbl.Parent = szRow

    local guiScale = 100

    local function makeSzBtn(txt, bgColor, xOffset, cb)
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(0,26,0,22)
        btn.Position = UDim2.new(1, xOffset, 0.5, -11)
        btn.BackgroundColor3 = bgColor; btn.BackgroundTransparency = 0.55
        btn.BorderSizePixel = 0; btn.Parent = szRow
        btn.Text = txt; btn.Font = Enum.Font.FredokaOne; btn.TextSize = 12
        btn.TextColor3 = Color3.fromRGB(255,255,255); btn.AutoButtonColor = false
        local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0,4); c.Parent = btn
        local s = Instance.new("UIStroke"); s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
        s.Color = bgColor:Lerp(Color3.fromRGB(255,255,255),0.5)
        s.Thickness = 1.0; s.Transparency = 0.3; s.Parent = btn
        btn.MouseButton1Down:Connect(function()
            TweenService:Create(btn,TweenInfo.new(0.07),{BackgroundTransparency=0.45}):Play()
            TweenService:Create(s,TweenInfo.new(0.07),{Transparency=0.0}):Play()
        end)
        local function release()
            TweenService:Create(btn,TweenInfo.new(0.1),{BackgroundTransparency=0.55}):Play()
            TweenService:Create(s,TweenInfo.new(0.1),{Transparency=0.3}):Play()
        end
        btn.MouseButton1Up:Connect(release); btn.MouseLeave:Connect(release)
        btn.MouseButton1Click:Connect(cb)
        return btn, s
    end

    makeSzBtn("−", Color3.fromRGB(96,48,192), -86, function()
        guiScale = math.max(50, guiScale-10)
        szLbl.Text = "GUI Size: " .. guiScale .. "%"
        local uisc = MainFrame:FindFirstChildOfClass("UIScale")
        if not uisc then uisc = Instance.new("UIScale"); uisc.Parent = MainFrame end
        uisc.Scale = guiScale/100
    end)

    makeSzBtn("+", Color3.fromRGB(96,48,192), -56, function()
        guiScale = math.min(200, guiScale+10)
        szLbl.Text = "GUI Size: " .. guiScale .. "%"
        local uisc = MainFrame:FindFirstChildOfClass("UIScale")
        if not uisc then uisc = Instance.new("UIScale"); uisc.Parent = MainFrame end
        uisc.Scale = guiScale/100
    end)

    local lockBtn2, lockStroke2 = makeSzBtn("🔓", Color3.fromRGB(28,28,34), -26, function() end)
    lockStroke2.Color = Color3.fromRGB(50,50,60)
    lockBtn2.TextColor3 = Color3.fromRGB(136,136,136)
    lockBtn2.MouseButton1Click:Connect(function()
        dragEnabled = not dragEnabled
        if dragEnabled then
            lockBtn2.Text = "🔓"; lockBtn2.TextColor3 = Color3.fromRGB(136,136,136)
            TweenService:Create(lockBtn2, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(28,28,34), BackgroundTransparency = 0.55}):Play()
            TweenService:Create(lockStroke2, TweenInfo.new(0.15), {Color = Color3.fromRGB(50,50,60)}):Play()
        else
            lockBtn2.Text = "🔒"; lockBtn2.TextColor3 = Color3.fromRGB(251,146,60)
            TweenService:Create(lockBtn2, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(40,22,8), BackgroundTransparency = 0.3}):Play()
            TweenService:Create(lockStroke2, TweenInfo.new(0.15), {Color = Color3.fromRGB(251,146,60)}):Play()
        end
    end)
end

-- ============================================================
--  MM2 REMOTE LISTENERS
-- ============================================================
Remote.StartTrade.OnClientEvent:Connect(function()
    if fakeTrade.active then cancelFakeTrade() end
end)
Remote.DeclineTrade.OnClientEvent:Connect(function()
    cancelFakeTrade()
end)
Remote.UpdateTrade.OnClientEvent:Connect(function(state)
    if state == nil and fakeTrade.active then cancelFakeTrade() end
end)

-- ── When a real trade is accepted, fire popups for your items ─
Remote.AcceptTrade.OnClientEvent:Connect(function()
    if fakeTrade.active and #fakeTrade.yourOffer > 0 then
        notifyItemsReceived(fakeTrade.yourOffer, fakeTrade.partnerName)
    end
end)


-- ============================================================
--  MM2 GODLY SPINNER  (Real Unboxing2 GUI from BoxModule)
--  Uses the actual Unboxing2 ScreenGui, NewItem template, and
--  ItemModule.DisplayItem — identical to MM2's real crate opening.
-- ============================================================

do -- scope block

openSpinWheel = function()
    -- Resolve BoxModule assets
    local BoxModuleScript = ReplicatedStorage:FindFirstChild("Modules")
        and ReplicatedStorage.Modules:FindFirstChild("BoxModule")
    if not BoxModuleScript then
        warn("[Spinner] BoxModule not found")
        showToast(statusToast, "⚠  BoxModule not found", 3)
        return
    end

    local Unboxing2Source = BoxModuleScript:FindFirstChild("Unboxing2")
    local NewItemSource = BoxModuleScript:FindFirstChild("NewItem")
    local GridSource = BoxModuleScript:FindFirstChild("UIGridLayout")

    if not Unboxing2Source or not NewItemSource then
        warn("[Spinner] Unboxing2 or NewItem not found in BoxModule")
        showToast(statusToast, "⚠  Unboxing assets missing", 3)
        return
    end

    local TweenService = game:GetService("TweenService")
    local rng = Random.new()

    -- ── Hardcoded pools (Tier 3 Godlies, Tier 2 Ancients, Tier 2+3 Chromas) ─
    local godlyPool = {
        "TravelerGun","TreeGun2023","Constellation","TreeKnife2023","Turkey2023",
        "UFOKnife","VampireGun","Darkshot","Darksword","Raygun","SunsetGun",
        "Snowcannon","Bauble","SunsetKnife","HeartWand","WraithGun","WraithKnife",
        "Flora","Bloom","Rainbow_G","Rainbow_K","SnowDagger","FlowerwoodGun",
        "FlowerwoodKnife","XenoKnife","XenoGun","Watergun","Ocean_G","Waves_K",
        "Treat","Sweet","Blizzard",
    }
    local ancientPool = {
        "Gingerscope","TravelerAxe","Celestial","VampireAxe","Harvester","Icepiercer",
    }
    local chromaPool = {
        "TravelerGunChroma","TreeGun2023Chroma","TreeKnife2023Chroma",
        "BaubleChroma","ConstellationChroma","VampireGunChroma","UFOKnifeChroma",
        "RaygunChroma","SunsetGunChroma","SnowcannonChroma","BlizzardChroma",
        "SunsetKnifeChroma","SnowDaggerChroma","TreatChroma","HeartWandChroma",
        "SnowstormChroma","WatergunChroma","SweetChroma","BaubleKnifeChroma",
    }

    -- Build weaponPool lookup from DB for DisplayItem to work
    local weaponPool = {}
    if DB and DB.Weapons then
        for id, data in pairs(DB.Weapons) do
            if type(data) == "table" then
                weaponPool[id] = data
                -- Also try lowercase/variant matching
                local nameLower = (data.ItemName or data.Name or ""):lower():gsub("[^a-z0-9]","")
                weaponPool[nameLower] = data
            end
        end
    end

    local function resolveItem(id) return resolveDBItem(id) end

    -- ── Pick random item with weights: Godly 65%, Ancient 25%, Chroma 10% ─
    local function pickRandomWeaponID()
        local roll = rng:NextInteger(1, 100)
        local pool
        if roll <= 10 and #chromaPool > 0 then
            pool = chromaPool
        elseif roll <= 35 and #ancientPool > 0 then
            pool = ancientPool
        else
            pool = #godlyPool > 0 and godlyPool or ancientPool
        end
        return pool[rng:NextInteger(1, #pool)]
    end

    -- ── Pick the winning item ────────────────────────────────
    local winID = pickRandomWeaponID()
    local resolvedWinID, winData = resolveItem(winID)

    -- ── Clone the REAL Unboxing2 GUI ─────────────────────────
    local unboxGUI = Unboxing2Source:Clone()

    -- Navigate to containers (exact path from decompiled BoxModule)
    -- Path: Unboxing2.Container.Main.Container.Background.ItemContainer.OffsetContainer
    local guiContainer = unboxGUI:FindFirstChild("Container")
    local guiMain = guiContainer and guiContainer:FindFirstChild("Main")
    local innerContainer = guiMain and guiMain:FindFirstChild("Container")
    local background = innerContainer and innerContainer:FindFirstChild("Background")
    local itemContainer = background and background:FindFirstChild("ItemContainer")
    local offsetContainer = itemContainer and itemContainer:FindFirstChild("OffsetContainer")
    local mainContainer = offsetContainer and offsetContainer:FindFirstChild("MainContainer")

    if not mainContainer or not offsetContainer then
        warn("[Spinner] Could not navigate Unboxing2 structure")
        warn("  guiContainer=" .. tostring(guiContainer))
        warn("  guiMain=" .. tostring(guiMain))
        warn("  innerContainer=" .. tostring(innerContainer))
        warn("  background=" .. tostring(background))
        warn("  itemContainer=" .. tostring(itemContainer))
        warn("  offsetContainer=" .. tostring(offsetContainer))
        warn("  mainContainer=" .. tostring(mainContainer))
        unboxGUI:Destroy()
        showToast(statusToast, "⚠  Unboxing2 structure mismatch", 3)
        return
    end

    -- ── Strip count (matching BoxModule: Min=20, Max=25) ──────
    local stripCount = rng:NextInteger(20, 25)
    local winIndex = stripCount -- winner at the last-ish position (like BoxModule)

    -- ── Clear MainContainer and add UIGridLayout ─────────────
    mainContainer:ClearAllChildren()

    local gridLayout
    if GridSource then
        gridLayout = GridSource:Clone()
    else
        gridLayout = Instance.new("UIGridLayout")
        gridLayout.CellSize = UDim2.new(0, 100, 1, 0)
        gridLayout.CellPadding = UDim2.new(0, 0, 0, 0)
        gridLayout.SortOrder = Enum.SortOrder.LayoutOrder
        gridLayout.FillDirection = Enum.FillDirection.Horizontal
    end
    gridLayout.Parent = mainContainer

    local cellWidth = gridLayout.CellSize.X.Offset

    -- ── Size the OffsetContainer (exactly like BoxModule) ─────
    offsetContainer.Size = UDim2.new(0, cellWidth * (stripCount + 5), 1, 0)
    offsetContainer.Position = UDim2.new(0.5, cellWidth / 2, 0, 0)

    -- ── Populate strip items ─────────────────────────────────
    for i = 1, stripCount + 5 do
        local itemData
        if i == winIndex then
            itemData = winData
        else
            local randID = pickRandomWeaponID()
            local _, randData = resolveItem(randID)
            itemData = randData
        end

        local cell = NewItemSource:Clone()
        cell.LayoutOrder = i
        cell.Visible = true
        cell.Parent = mainContainer

        if itemData and ItemModule and ItemModule.DisplayItem then
            pcall(function() ItemModule.DisplayItem(cell, itemData) end)
        end
    end

    -- ── Populate PreItem frames (items visible before strip) ──
    pcall(function()
        local preItem1 = offsetContainer:FindFirstChild("PreItem1")
        if preItem1 then
            local randID = pickRandomWeaponID()
            local data = weaponPool[randID] or (DB and DB.Weapons and DB.Weapons[randID])
            if data then ItemModule.DisplayItem(preItem1, data) end
        end
    end)
    pcall(function()
        local preItem2 = offsetContainer:FindFirstChild("PreItem2")
        if preItem2 then
            local randID = pickRandomWeaponID()
            local data = weaponPool[randID] or (DB and DB.Weapons and DB.Weapons[randID])
            if data then ItemModule.DisplayItem(preItem2, data) end
        end
    end)

    -- ── Show the GUI above the trade ─────────────────────────
    unboxGUI.DisplayOrder = 1100  -- above ScreenGui (999) and SimGUI
    unboxGUI.Parent = PlayerGui

    -- ── THE TWEEN (exact BoxModule formula) ──────────────────
    -- Duration: 3 + random 0-1 seconds
    local spinDuration = 3 + rng:NextNumber()

    -- Target position (exact BoxModule formula):
    -- -(Offset * winIndex) + random(-(Offset/2 - 5), Offset/2 - 5) + Offset - Offset/2
    local targetX = -(cellWidth * winIndex)
        + rng:NextInteger(-(cellWidth / 2 - 5), cellWidth / 2 - 5)
        + cellWidth - cellWidth / 2

    TweenService:Create(offsetContainer,
        TweenInfo.new(spinDuration, Enum.EasingStyle.Cubic, Enum.EasingDirection.Out),
        { Position = UDim2.new(0.5, targetX, 0, 0) }
    ):Play()

-- ── Wait for spin to finish ───────────────────────────────────
    task.wait(spinDuration + 1)

    local itemName = winData and (winData.ItemName or winData.Name) or resolvedWinID

    -- Destroy spin GUI first so claim popup isn't hidden behind it
    pcall(function() unboxGUI:Destroy() end)

    -- ── Show the real MM2 claim popup ────────────────────────────
    local claimDone = false
    pcall(function()
        local claimConn
        claimConn = ItemPopupService.ItemClaimsComplete.Event:Connect(function()
            claimDone = true
            pcall(function() claimConn:Disconnect() end)
        end)

        -- Push popup above fake trade UI (DisplayOrder 999) and control panel
        local crossPlatform = PlayerGui:FindFirstChild("CrossPlatform")
        if crossPlatform then crossPlatform.DisplayOrder = 1200 end

        ItemPopupService.ItemReceived:Fire(resolvedWinID, "Weapons", 1)

        local elapsed = 0
        while not claimDone and elapsed < 30 do
            task.wait(0.1)
            elapsed = elapsed + 0.1
        end
        pcall(function() claimConn:Disconnect() end)

        -- Restore original order
        if crossPlatform then crossPlatform.DisplayOrder = 0 end
    end)

    -- ── Add to YOUR side after claim is dismissed ─────────────────
    local tradeItem = { DataID = resolvedWinID, DataType = "Weapons", Amount = 1, Name = itemName, ItemName = itemName }
    if winData then for k, v in pairs(winData) do if tradeItem[k] == nil then tradeItem[k] = v end end end
    if fakeTrade.active then
        if AddToSide(YourSlots, tradeItem) then RefreshAllSlots(); ResetCooldown(false) end
    end
    showToast(statusToast, "✦  Unboxed: " .. itemName, 4)
end

-- ── Add SPIN button to Control panel ─────────────────────────
createDivider(controlPanel)
createSectionLabel("UNBOXING", controlPanel)
createButton("🎰  UNBOX GODLY",
    Color3.fromRGB(160,100,20), nil, nil,
    controlPanel, function()
        openSpinWheel()
    end)

end -- end spinner scope


print("[TradeManager INTEGRATED v7 — Redesigned UI] Loaded.")
print(string.format(
    "  ProfileData=%s  ItemModule=%s  ItemPopupService=%s",
    tostring(ProfileData ~= nil),
    tostring(ItemModule  ~= nil),
    tostring(ItemPopupService ~= nil)
))
