repeat task.wait() until game:IsLoaded()
-- ============================================================
-- Faded.vs [Freemium] - REDUCED FEATURES
-- ============================================================

local Players         = game:GetService("Players")
local RunService      = game:GetService("RunService")
local UIS             = game:GetService("UserInputService")
local TweenService    = game:GetService("TweenService")
local HttpService     = game:GetService("HttpService")
local ContentProvider = game:GetService("ContentProvider")
local Lighting        = game:GetService("Lighting")
local LP              = Players.LocalPlayer

local _isfile   = isfile   or (syn and syn.isfile)   or function() return false end
local _readfile = readfile  or (syn and syn.readfile)  or function() return nil  end
local _writefile= writefile or (syn and syn.writefile) or function() end
local getconnections = getconnections or get_signal_cons or getconnects or (syn and syn.get_signal_cons)

-- ============================================================
-- STATE
-- ============================================================
local State = {
    infJumpEnabled=false,
    guiVisible=true, uiLocked=false,
    isStealing=false, stealStartTime=nil, lastStealTick=0,
    autoLeftEnabled=false, autoRightEnabled=false,
    autoLeftPhase=1, autoRightPhase=1,
    lastMoveDir=Vector3.new(0,0,0),
    autoStealEnabled=false,
}

local Keys = {
    autoLeft   = Enum.KeyCode.Z,
    autoRight  = Enum.KeyCode.C,
    tpDown     = Enum.KeyCode.V,
    guiHide    = Enum.KeyCode.LeftControl,
}

local Steal = {
    AutoStealEnabled=false, StealRadius=60, StealDuration=1.4,
    Data={}, plotCache={}, plotCacheTime={}, cachedPrompts={}, promptCacheTime=0,
}

local CONFIG_FILE = "faded_config.json"

local POS={
    L1=Vector3.new(-476.48,-6.28,92.73), L2=Vector3.new(-483.12,-4.95,94.80),
    R1=Vector3.new(-476.16,-6.52,25.62), R2=Vector3.new(-483.04,-5.09,23.14),
}

local Conns={autoSteal=nil,autoLeft=nil,autoRight=nil,progress=nil}

local PLOT_CACHE_DURATION=2
local PROMPT_CACHE_REFRESH=0.15
local STEAL_COOLDOWN=0.1

local isTouchEnabled = UIS.TouchEnabled

local h,hrp
local toggleRefs={}
local mbGroup
local setAL, setAR

-- ============================================================
-- HELPER FUNCTIONS
-- ============================================================
local function getKeyName(kc)
    local n = kc.Name
    if n == "Unknown" then return "—" end
    if n == "LeftControl" then return "CTRL" end
    if n == "RightControl" then return "RCTL" end
    if n == "LeftShift" then return "SHFT" end
    if n == "Space" then return "SPC" end
    return n:sub(1, 4):upper()
end

-- ============================================================
-- CONFIG SAVE/LOAD
-- ============================================================
local function saveConfig()
    local cfg = {
        stealRadius = Steal.StealRadius,
        stealDuration = Steal.StealDuration,
        autoLeftKey = Keys.autoLeft.Name,
        autoRightKey = Keys.autoRight.Name,
        guiHideKey = Keys.guiHide.Name,
        tpDownKey = Keys.tpDown.Name,
    }
    local ok, enc = pcall(function() return HttpService:JSONEncode(cfg) end)
    if ok and enc then 
        pcall(function() _writefile(CONFIG_FILE, enc) end) 
    end
end

local function loadConfig()
    local has = false
    pcall(function() has = _isfile(CONFIG_FILE) end)
    if not has then return end
    local raw
    pcall(function() raw = _readfile(CONFIG_FILE) end)
    if not raw then return end
    local cfg
    local ok = pcall(function() cfg = HttpService:JSONDecode(raw) end)
    if not ok or not cfg then return end
    
    if cfg.stealRadius then Steal.StealRadius = cfg.stealRadius end
    if cfg.stealDuration then Steal.StealDuration = cfg.stealDuration end
    
    local function tryKey(field, kt)
        if cfg[field] and Enum.KeyCode[cfg[field]] then
            Keys[kt] = Enum.KeyCode[cfg[field]]
        end
    end
    tryKey("autoLeftKey", "autoLeft")
    tryKey("autoRightKey", "autoRight")
    tryKey("guiHideKey", "guiHide")
    tryKey("tpDownKey", "tpDown")
end

-- ============================================================
-- CLEANUP OLD GUIs
-- ============================================================
for _,name in pairs({"opiumhubV5_3", "Fadedvs"}) do
    pcall(function() local o=game:GetService("CoreGui"):FindFirstChild(name); if o then o:Destroy() end end)
    pcall(function() local o=LP:WaitForChild("PlayerGui"):FindFirstChild(name); if o then o:Destroy() end end)
end

-- ============================================================
-- ROOT GUI
-- ============================================================
local gui = Instance.new("ScreenGui")
gui.Name = "Fadedvs"
gui.ResetOnSpawn = false
gui.DisplayOrder = 999
gui.IgnoreGuiInset = true
pcall(function() if syn and syn.protect_gui then syn.protect_gui(gui) end end)
pcall(function() gui.Parent = game:GetService("CoreGui") end)
if not gui.Parent then pcall(function() gui.Parent = LP.PlayerGui end) end

local uiScaleObj=Instance.new("UIScale",gui)
uiScaleObj.Scale=1.0

-- ============================================================
-- COLORS
-- ============================================================
local C = {
    bg          = Color3.fromRGB(5, 5, 5),
    panel       = Color3.fromRGB(26, 26, 26),
    card        = Color3.fromRGB(32, 32, 32),
    cardHov     = Color3.fromRGB(38, 38, 38),
    border      = Color3.fromRGB(48, 48, 48),
    borderDim   = Color3.fromRGB(36, 36, 36),
    text        = Color3.fromRGB(230, 230, 230),
    textSub     = Color3.fromRGB(130, 130, 130),
    textDim     = Color3.fromRGB(75, 75, 75),
    accent      = Color3.fromRGB(255, 255, 255),
    pillOff     = Color3.fromRGB(38, 38, 38),
    pillOn      = Color3.fromRGB(50, 50, 50),
    dotOff      = Color3.fromRGB(65, 65, 65),
    dotOn       = Color3.fromRGB(255, 255, 255),
    header      = Color3.fromRGB(8, 8, 8),
}

-- ============================================================
-- HELPERS
-- ============================================================
local function mkCorner(p,r) local c=Instance.new("UICorner",p); c.CornerRadius=UDim.new(0,r or 8); return c end
local function mkStroke(p,col,th)
    local s=Instance.new("UIStroke",p); s.Color=col or Color3.fromRGB(50,50,50); s.Thickness=th or 1
    s.ApplyStrokeMode=Enum.ApplyStrokeMode.Border; return s
end

local function makeDraggable(frame, handle, force)
    local src=handle or frame
    local dragging,dragInput,dragStart,startPos=false,nil,nil,nil
    src.InputBegan:Connect(function(inp)
        if State.uiLocked and not force then return end
        if inp.UserInputType==Enum.UserInputType.MouseButton1 or inp.UserInputType==Enum.UserInputType.Touch then
            dragging=true; dragStart=inp.Position; startPos=frame.Position
            inp.Changed:Connect(function() if inp.UserInputState==Enum.UserInputState.End then dragging=false end end)
        end
    end)
    src.InputChanged:Connect(function(inp)
        if inp.UserInputType==Enum.UserInputType.MouseMovement or inp.UserInputType==Enum.UserInputType.Touch then dragInput=inp end
    end)
    UIS.InputChanged:Connect(function(inp)
        if inp==dragInput and dragging and (force or not State.uiLocked) then
            local dx=inp.Position.X-dragStart.X; local dy=inp.Position.Y-dragStart.Y
            frame.Position=UDim2.new(startPos.X.Scale,startPos.X.Offset+dx,startPos.Y.Scale,startPos.Y.Offset+dy)
        end
    end)
end

-- ============================================================
-- MAIN WINDOW
-- ============================================================
local WIN_W = 330
local WIN_H = 580

local mainOuter = Instance.new("Frame", gui)
mainOuter.Name = "MainOuter"
mainOuter.Size = UDim2.new(0, WIN_W, 0, WIN_H)
mainOuter.Position = UDim2.new(0, 8, 0, 8)
mainOuter.BackgroundColor3 = C.bg
mainOuter.BorderSizePixel = 0
mainOuter.ClipsDescendants = true
mkCorner(mainOuter, 14)
mkStroke(mainOuter, C.border, 1)
makeDraggable(mainOuter)

-- Background Decal (107977050874654) with rounded corners
local bgDecal = Instance.new("ImageLabel", mainOuter)
bgDecal.Size = UDim2.new(1, 0, 1, 0)
bgDecal.Position = UDim2.new(0, 0, 0, 0)
bgDecal.BackgroundTransparency = 1
bgDecal.Image = "rbxassetid://107977050874654"
bgDecal.ZIndex = 1
bgDecal.ScaleType = Enum.ScaleType.Crop
mkCorner(bgDecal, 14)

-- ============================================================
-- HEADER
-- ============================================================
local HEADER_H = 96
local headerFrame = Instance.new("Frame", mainOuter)
headerFrame.Size = UDim2.new(1, 0, 0, HEADER_H)
headerFrame.Position = UDim2.new(0, 0, 0, 0)
headerFrame.BackgroundColor3 = C.header
headerFrame.BackgroundTransparency = 0.8
headerFrame.BorderSizePixel = 0
headerFrame.ZIndex = 3
mkCorner(headerFrame, 14)
headerFrame.ClipsDescendants = true

local accentStripe = Instance.new("Frame", headerFrame)
accentStripe.Size = UDim2.new(0, 3, 0, 52)
accentStripe.Position = UDim2.new(0, 0, 0.5, -26)
accentStripe.BackgroundColor3 = Color3.fromRGB(200, 200, 200)
accentStripe.BorderSizePixel = 0
accentStripe.ZIndex = 4
mkCorner(accentStripe, 2)

local usernameLbl = Instance.new("TextLabel", headerFrame)
usernameLbl.Size = UDim2.new(1, -140, 0, 22)
usernameLbl.Position = UDim2.new(0, 20, 0, 20)
usernameLbl.BackgroundTransparency = 1
usernameLbl.Text = LP.DisplayName
usernameLbl.TextColor3 = C.text
usernameLbl.Font = Enum.Font.GothamBlack
usernameLbl.TextSize = 16
usernameLbl.TextXAlignment = Enum.TextXAlignment.Left
usernameLbl.ZIndex = 4

local handleLbl = Instance.new("TextLabel", headerFrame)
handleLbl.Size = UDim2.new(1, -140, 0, 14)
handleLbl.Position = UDim2.new(0, 20, 0, 44)
handleLbl.BackgroundTransparency = 1
handleLbl.Text = "@" .. LP.Name
handleLbl.TextColor3 = C.textSub
handleLbl.Font = Enum.Font.Gotham
handleLbl.TextSize = 11
handleLbl.TextXAlignment = Enum.TextXAlignment.Left
handleLbl.ZIndex = 4

local badgeBg = Instance.new("Frame", headerFrame)
badgeBg.Size = UDim2.new(0, 130, 0, 18)
badgeBg.Position = UDim2.new(0, 20, 0, 64)
badgeBg.BackgroundColor3 = Color3.fromRGB(22, 22, 22)
badgeBg.BackgroundTransparency = 0.3
badgeBg.BorderSizePixel = 0
badgeBg.ZIndex = 4
mkCorner(badgeBg, 4)
mkStroke(badgeBg, Color3.fromRGB(60, 60, 60), 1)

local badgeLbl = Instance.new("TextLabel", badgeBg)
badgeLbl.Size = UDim2.new(1, 0, 1, 0)
badgeLbl.BackgroundTransparency = 1
badgeLbl.Text = "FREEMIUM"
badgeLbl.TextColor3 = Color3.fromRGB(160, 160, 160)
badgeLbl.Font = Enum.Font.GothamBold
badgeLbl.TextSize = 9
badgeLbl.ZIndex = 5

local minimizeBtn = Instance.new("TextButton", headerFrame)
minimizeBtn.Size = UDim2.new(0, 28, 0, 28)
minimizeBtn.Position = UDim2.new(1, -42, 0, 12)
minimizeBtn.BackgroundColor3 = C.card
minimizeBtn.BackgroundTransparency = 0.3
minimizeBtn.BorderSizePixel = 0
minimizeBtn.Text = "—"
minimizeBtn.TextColor3 = C.textSub
minimizeBtn.Font = Enum.Font.GothamBold
minimizeBtn.TextSize = 14
minimizeBtn.ZIndex = 5
mkCorner(minimizeBtn, 6)
mkStroke(minimizeBtn, C.border, 1)
minimizeBtn.MouseButton1Click:Connect(function()
    mainOuter.Visible = false
end)

local headerDiv = Instance.new("Frame", mainOuter)
headerDiv.Size = UDim2.new(1, 0, 0, 1)
headerDiv.Position = UDim2.new(0, 0, 0, HEADER_H)
headerDiv.BackgroundColor3 = C.border
headerDiv.BackgroundTransparency = 0.5
headerDiv.BorderSizePixel = 0
headerDiv.ZIndex = 3

-- ============================================================
-- SCROLL AREA
-- ============================================================
local SCROLL_Y = HEADER_H + 1
local scrollFrame = Instance.new("ScrollingFrame", mainOuter)
scrollFrame.Size = UDim2.new(1, 0, 1, -SCROLL_Y)
scrollFrame.Position = UDim2.new(0, 0, 0, SCROLL_Y)
scrollFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
scrollFrame.BackgroundTransparency = 0.5
scrollFrame.BorderSizePixel = 0
scrollFrame.ClipsDescendants = true
scrollFrame.ScrollBarThickness = 2
scrollFrame.ScrollBarImageColor3 = C.border
scrollFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y
scrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
scrollFrame.ZIndex = 2
mkCorner(scrollFrame, 14)

local scrollLL = Instance.new("UIListLayout", scrollFrame)
scrollLL.SortOrder = Enum.SortOrder.LayoutOrder
scrollLL.Padding = UDim.new(0, 2)

local scrollPad = Instance.new("UIPadding", scrollFrame)
scrollPad.PaddingLeft = UDim.new(0, 10)
scrollPad.PaddingRight = UDim.new(0, 10)
scrollPad.PaddingTop = UDim.new(0, 8)
scrollPad.PaddingBottom = UDim.new(0, 12)

local loCount = 0
local function LO() loCount = loCount + 1; return loCount end

-- ============================================================
-- SECTION HEADER
-- ============================================================
local function makeSectionHeader(label)
    local gap = Instance.new("Frame", scrollFrame)
    gap.Size = UDim2.new(1, 0, 0, 4)
    gap.BackgroundTransparency = 1
    gap.BorderSizePixel = 0
    gap.LayoutOrder = LO()

    local row = Instance.new("Frame", scrollFrame)
    row.Size = UDim2.new(1, 0, 0, 26)
    row.BackgroundTransparency = 1
    row.BorderSizePixel = 0
    row.LayoutOrder = LO()
    row.ZIndex = 3

    local lbl = Instance.new("TextLabel", row)
    lbl.Size = UDim2.new(1, 0, 1, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = label:upper()
    lbl.TextColor3 = Color3.fromRGB(160, 160, 160)
    lbl.Font = Enum.Font.GothamBold
    lbl.TextSize = 10
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.ZIndex = 4
end

local function makeDivider()
    local f = Instance.new("Frame", scrollFrame)
    f.Size = UDim2.new(1, 0, 0, 1)
    f.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    f.BackgroundTransparency = 0.5
    f.BorderSizePixel = 0
    f.LayoutOrder = LO()
end


-- ============================================================
-- TOGGLE ROW WITH KEYBIND SUPPORT
-- ============================================================
local function makeToggleRow(label, defaultOn, onToggle, keyRef)
    local row = Instance.new("Frame", scrollFrame)
    row.Size = UDim2.new(1, 0, 0, 52)
    row.BackgroundColor3 = C.bg
    row.BackgroundTransparency = 0.6
    row.BorderSizePixel = 0
    row.LayoutOrder = LO()
    row.ZIndex = 3

    local div = Instance.new("Frame", row)
    div.Size = UDim2.new(1, -28, 0, 1)
    div.Position = UDim2.new(0, 14, 1, -1)
    div.BackgroundColor3 = C.borderDim
    div.BackgroundTransparency = 0.5
    div.BorderSizePixel = 0
    div.ZIndex = 4

    local lblX = 14
    
    if keyRef then
        local chip = Instance.new("TextButton", row)
        chip.Size = UDim2.new(0, 32, 0, 24)
        chip.Position = UDim2.new(0, 14, 0.5, -12)
        chip.BackgroundColor3 = C.bg
        chip.BackgroundTransparency = 0.5
        chip.BorderSizePixel = 0
        chip.Text = getKeyName(Keys[keyRef] or Enum.KeyCode.Unknown)
        chip.TextColor3 = C.textSub
        chip.Font = Enum.Font.GothamBold
        chip.TextSize = 9
        chip.ZIndex = 5
        mkCorner(chip, 6)
        mkStroke(chip, C.borderDim, 1)
        
        local listening = false
        local lkb, lgp = nil, nil
        
        local function stopListening(key)
            listening = false
            if lkb then lkb:Disconnect(); lkb = nil end
            if lgp then lgp:Disconnect(); lgp = nil end
            chip.TextColor3 = C.textSub
            if key then
                Keys[keyRef] = key
                chip.Text = getKeyName(key)
                task.spawn(saveConfig)
            else
                chip.Text = getKeyName(Keys[keyRef] or Enum.KeyCode.Unknown)
            end
        end
        
        chip.MouseButton1Click:Connect(function()
            if listening then stopListening(nil); return end
            listening = true
            chip.Text = "···"
            chip.TextColor3 = C.text
            
            lkb = UIS.InputBegan:Connect(function(inp)
                if not listening then return end
                if inp.UserInputType ~= Enum.UserInputType.Keyboard then return end
                if inp.KeyCode == Enum.KeyCode.Escape then stopListening(nil); return end
                stopListening(inp.KeyCode)
            end)
            
            lgp = UIS.InputBegan:Connect(function(inp)
                if not listening then return end
                local isGp = inp.UserInputType == Enum.UserInputType.Gamepad1 or 
                            inp.UserInputType == Enum.UserInputType.Gamepad2
                if not isGp then return end
                if inp.KeyCode == Enum.KeyCode.Unknown then return end
                stopListening(inp.KeyCode)
            end)
        end)
        
        lblX = 54
    end

    local lbl = Instance.new("TextLabel", row)
    lbl.Size = UDim2.new(1, -(lblX + 66), 1, 0)
    lbl.Position = UDim2.new(0, lblX, 0, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = label
    lbl.TextColor3 = C.text
    lbl.Font = Enum.Font.GothamBold
    lbl.TextSize = 13
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.ZIndex = 4

    local pill = Instance.new("Frame", row)
    pill.Size = UDim2.new(0, 44, 0, 24)
    pill.Position = UDim2.new(1, -58, 0.5, -12)
    pill.BackgroundColor3 = defaultOn and C.pillOn or C.pillOff
    pill.BackgroundTransparency = 0.3
    pill.BorderSizePixel = 0
    pill.ZIndex = 5
    mkCorner(pill, 12)
    mkStroke(pill, C.border, 1)

    local dot = Instance.new("Frame", pill)
    dot.Size = UDim2.new(0, 16, 0, 16)
    dot.Position = defaultOn and UDim2.new(1, -19, 0.5, -8) or UDim2.new(0, 3, 0.5, -8)
    dot.BackgroundColor3 = defaultOn and C.dotOn or C.dotOff
    dot.BorderSizePixel = 0
    dot.ZIndex = 6
    mkCorner(dot, 8)

    local isOn = defaultOn or false
    local function setV(on)
        isOn = on
        TweenService:Create(pill, TweenInfo.new(0.18, Enum.EasingStyle.Quad), {BackgroundColor3 = on and C.pillOn or C.pillOff}):Play()
        TweenService:Create(dot, TweenInfo.new(0.18, Enum.EasingStyle.Back), {
            Position = on and UDim2.new(1, -19, 0.5, -8) or UDim2.new(0, 3, 0.5, -8),
            BackgroundColor3 = on and C.dotOn or C.dotOff
        }):Play()
    end
    local function toggle() isOn = not isOn; setV(isOn); if onToggle then pcall(onToggle, isOn) end end

    local clk = Instance.new("TextButton", row)
    clk.Size = UDim2.new(1, -66, 1, 0)
    clk.Position = UDim2.new(0, lblX, 0, 0)
    clk.BackgroundTransparency = 1
    clk.Text = ""
    clk.ZIndex = 5
    clk.BorderSizePixel = 0
    clk.MouseButton1Click:Connect(toggle)

    local pClk = Instance.new("TextButton", pill)
    pClk.Size = UDim2.new(1, 0, 1, 0)
    pClk.BackgroundTransparency = 1
    pClk.Text = ""
    pClk.ZIndex = 7
    pClk.BorderSizePixel = 0
    pClk.MouseButton1Click:Connect(toggle)

    return setV
end

-- ============================================================
-- INPUT ROW
-- ============================================================
local function makeInputRow(label, default, onChange)
    local row = Instance.new("Frame", scrollFrame)
    row.Size = UDim2.new(1, 0, 0, 52)
    row.BackgroundColor3 = C.bg
    row.BackgroundTransparency = 0.6
    row.BorderSizePixel = 0
    row.LayoutOrder = LO()
    row.ZIndex = 3

    local div = Instance.new("Frame", row)
    div.Size = UDim2.new(1, -28, 0, 1)
    div.Position = UDim2.new(0, 14, 1, -1)
    div.BackgroundColor3 = C.borderDim
    div.BackgroundTransparency = 0.5
    div.BorderSizePixel = 0
    div.ZIndex = 4

    local lbl = Instance.new("TextLabel", row)
    lbl.Size = UDim2.new(0.6, 0, 1, 0)
    lbl.Position = UDim2.new(0, 14, 0, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = label
    lbl.TextColor3 = C.text
    lbl.Font = Enum.Font.GothamBold
    lbl.TextSize = 13
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.ZIndex = 4

    local boxWrap = Instance.new("Frame", row)
    boxWrap.Size = UDim2.new(0, 70, 0, 30)
    boxWrap.Position = UDim2.new(1, -84, 0.5, -15)
    boxWrap.BackgroundColor3 = C.card
    boxWrap.BackgroundTransparency = 0.3
    boxWrap.BorderSizePixel = 0
    mkCorner(boxWrap, 6)
    local bs = mkStroke(boxWrap, C.border, 1)

    local box = Instance.new("TextBox", boxWrap)
    box.Size = UDim2.new(1, -8, 1, 0)
    box.Position = UDim2.new(0, 4, 0, 0)
    box.BackgroundTransparency = 1
    box.Text = tostring(default)
    box.TextColor3 = C.text
    box.Font = Enum.Font.GothamBold
    box.TextSize = 13
    box.ClearTextOnFocus = false
    box.ZIndex = 5
    box.TextXAlignment = Enum.TextXAlignment.Center
    box.Focused:Connect(function()
        TweenService:Create(bs,TweenInfo.new(0.15),{Color=Color3.fromRGB(120,120,120)}):Play()
    end)
    box.FocusLost:Connect(function()
        TweenService:Create(bs,TweenInfo.new(0.15),{Color=C.border}):Play()
        if onChange then
            local n=tonumber(box.Text)
            if n then onChange(n) else box.Text=tostring(default) end
        end
    end)
    
    return box, row
end

-- ============================================================
-- TP DOWN
-- ============================================================
local function doTpDown()
    pcall(function()
        local c=LP.Character; if not c then return end
        local root=c:FindFirstChild("HumanoidRootPart"); if not root then return end
        local rp=RaycastParams.new(); rp.FilterDescendantsInstances={c}; rp.FilterType=Enum.RaycastFilterType.Exclude
        local res=workspace:Raycast(root.Position,Vector3.new(0,-1000,0),rp)
        if res then
            root.CFrame=CFrame.new(res.Position+Vector3.new(0,root.Size.Y/2+0.5,0))
            root.AssemblyLinearVelocity=Vector3.zero
        end
    end)
end

-- ============================================================
-- INFINITE JUMP
-- ============================================================
UIS.JumpRequest:Connect(function()
    if not State.infJumpEnabled then return end
    local char = LP.Character
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    if root then
        root.Velocity = Vector3.new(root.Velocity.X, 55, root.Velocity.Z)
    end
end)

RunService.Heartbeat:Connect(function()
    if not State.infJumpEnabled then return end
    local char = LP.Character
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    if root and root.Velocity.Y < -120 then
        root.Velocity = Vector3.new(root.Velocity.X, -120, root.Velocity.Z)
    end
end)

-- ============================================================
-- AUTO LEFT / RIGHT
-- ============================================================
local function faceSouth()
    pcall(function()
        local c=LP.Character; if not c then return end
        local root=c:FindFirstChild("HumanoidRootPart")
        if root then root.CFrame=CFrame.new(root.Position)*CFrame.Angles(0,0,0) end
    end)
end

local function faceNorth()
    pcall(function()
        local c=LP.Character; if not c then return end
        local root=c:FindFirstChild("HumanoidRootPart")
        if root then root.CFrame=CFrame.new(root.Position)*CFrame.Angles(0,math.rad(180),0) end
    end)
end

local function startAutoLeft()
    if Conns.autoLeft then Conns.autoLeft:Disconnect() end
    State.autoLeftPhase=1
    Conns.autoLeft=RunService.Heartbeat:Connect(function()
        if not State.autoLeftEnabled then return end
        local c=LP.Character; if not c then return end
        local root=c:FindFirstChild("HumanoidRootPart")
        local hum2=c:FindFirstChildOfClass("Humanoid")
        if not root or not hum2 then return end
        if State.autoLeftPhase==1 then
            local tgt=Vector3.new(POS.L1.X,root.Position.Y,POS.L1.Z)
            if (tgt-root.Position).Magnitude<1 then
                State.autoLeftPhase=2
                local d=(POS.L2-root.Position)
                local mv=Vector3.new(d.X,0,d.Z).Unit
                hum2:Move(mv,false)
                root.AssemblyLinearVelocity=Vector3.new(mv.X*50,root.AssemblyLinearVelocity.Y,mv.Z*50)
                return
            end
            local d=(POS.L1-root.Position)
            local mv=Vector3.new(d.X,0,d.Z).Unit
            hum2:Move(mv,false)
            root.AssemblyLinearVelocity=Vector3.new(mv.X*50,root.AssemblyLinearVelocity.Y,mv.Z*50)
        elseif State.autoLeftPhase==2 then
            local tgt=Vector3.new(POS.L2.X,root.Position.Y,POS.L2.Z)
            if (tgt-root.Position).Magnitude<1 then
                hum2:Move(Vector3.zero,false)
                root.AssemblyLinearVelocity=Vector3.zero
                State.autoLeftEnabled=false
                if Conns.autoLeft then Conns.autoLeft:Disconnect(); Conns.autoLeft=nil end
                State.autoLeftPhase=1
                faceSouth()
                return
            end
            local d=(POS.L2-root.Position)
            local mv=Vector3.new(d.X,0,d.Z).Unit
            hum2:Move(mv,false)
            root.AssemblyLinearVelocity=Vector3.new(mv.X*50,root.AssemblyLinearVelocity.Y,mv.Z*50)
        end
    end)
end

local function stopAutoLeft()
    if Conns.autoLeft then Conns.autoLeft:Disconnect(); Conns.autoLeft=nil end
    State.autoLeftPhase=1
    local c=LP.Character
    if c then
        local hum2=c:FindFirstChildOfClass("Humanoid")
        if hum2 then hum2:Move(Vector3.zero,false) end
    end
end

local function startAutoRight()
    if Conns.autoRight then Conns.autoRight:Disconnect() end
    State.autoRightPhase=1
    Conns.autoRight=RunService.Heartbeat:Connect(function()
        if not State.autoRightEnabled then return end
        local c=LP.Character; if not c then return end
        local root=c:FindFirstChild("HumanoidRootPart")
        local hum2=c:FindFirstChildOfClass("Humanoid")
        if not root or not hum2 then return end
        if State.autoRightPhase==1 then
            local tgt=Vector3.new(POS.R1.X,root.Position.Y,POS.R1.Z)
            if (tgt-root.Position).Magnitude<1 then
                State.autoRightPhase=2
                local d=(POS.R2-root.Position)
                local mv=Vector3.new(d.X,0,d.Z).Unit
                hum2:Move(mv,false)
                root.AssemblyLinearVelocity=Vector3.new(mv.X*50,root.AssemblyLinearVelocity.Y,mv.Z*50)
                return
            end
            local d=(POS.R1-root.Position)
            local mv=Vector3.new(d.X,0,d.Z).Unit
            hum2:Move(mv,false)
            root.AssemblyLinearVelocity=Vector3.new(mv.X*50,root.AssemblyLinearVelocity.Y,mv.Z*50)
        elseif State.autoRightPhase==2 then
            local tgt=Vector3.new(POS.R2.X,root.Position.Y,POS.R2.Z)
            if (tgt-root.Position).Magnitude<1 then
                hum2:Move(Vector3.zero,false)
                root.AssemblyLinearVelocity=Vector3.zero
                State.autoRightEnabled=false
                if Conns.autoRight then Conns.autoRight:Disconnect(); Conns.autoRight=nil end
                State.autoRightPhase=1
                faceNorth()
                return
            end
            local d=(POS.R2-root.Position)
            local mv=Vector3.new(d.X,0,d.Z).Unit
            hum2:Move(mv,false)
            root.AssemblyLinearVelocity=Vector3.new(mv.X*50,root.AssemblyLinearVelocity.Y,mv.Z*50)
        end
    end)
end

local function stopAutoRight()
    if Conns.autoRight then Conns.autoRight:Disconnect(); Conns.autoRight=nil end
    State.autoRightPhase=1
    local c=LP.Character
    if c then
        local hum2=c:FindFirstChildOfClass("Humanoid")
        if hum2 then hum2:Move(Vector3.zero,false) end
    end
end

-- ============================================================
-- AUTO STEAL SYSTEM
-- ============================================================
local function isMyPlotByName(pn)
    local ct=tick()
    if Steal.plotCache[pn] and (ct-(Steal.plotCacheTime[pn] or 0))<PLOT_CACHE_DURATION then
        return Steal.plotCache[pn]
    end
    local plots=workspace:FindFirstChild("Plots")
    if not plots then Steal.plotCache[pn]=false; Steal.plotCacheTime[pn]=ct; return false end
    local plot=plots:FindFirstChild(pn)
    if not plot then Steal.plotCache[pn]=false; Steal.plotCacheTime[pn]=ct; return false end
    local sign=plot:FindFirstChild("PlotSign")
    if sign then
        local yb=sign:FindFirstChild("YourBase")
        if yb and yb:IsA("BillboardGui") then
            local r=yb.Enabled==true
            Steal.plotCache[pn]=r
            Steal.plotCacheTime[pn]=ct
            return r
        end
    end
    Steal.plotCache[pn]=false
    Steal.plotCacheTime[pn]=ct
    return false
end

local function findNearestPrompt()
    local c=LP.Character; if not c then return nil end
    local root=c:FindFirstChild("HumanoidRootPart"); if not root then return nil end
    local ct=tick()
    if ct-Steal.promptCacheTime<PROMPT_CACHE_REFRESH and #Steal.cachedPrompts>0 then
        local np,nd=nil,math.huge
        for _,data in ipairs(Steal.cachedPrompts) do
            if data.spawn then
                local dist=(data.spawn.Position-root.Position).Magnitude
                if dist<=Steal.StealRadius and dist<nd then np=data.prompt; nd=dist end
            end
        end
        if np then return np end
    end
    Steal.cachedPrompts={}
    Steal.promptCacheTime=ct
    local plots=workspace:FindFirstChild("Plots")
    if not plots then return nil end
    local np,nd=nil,math.huge
    for _,plot in ipairs(plots:GetChildren()) do
        if isMyPlotByName(plot.Name) then continue end
        local pods=plot:FindFirstChild("AnimalPodiums")
        if not pods then continue end
        for _,pod in ipairs(pods:GetChildren()) do
            pcall(function()
                local base=pod:FindFirstChild("Base")
                local sp=base and base:FindFirstChild("Spawn")
                if sp then
                    local att=sp:FindFirstChild("PromptAttachment")
                    if att then
                        for _,child in ipairs(att:GetChildren()) do
                            if child:IsA("ProximityPrompt") then
                                local dist=(sp.Position-root.Position).Magnitude
                                table.insert(Steal.cachedPrompts,{prompt=child,spawn=sp})
                                if dist<=Steal.StealRadius and dist<nd then np=child; nd=dist end
                                break
                            end
                        end
                    end
                end
            end)
        end
    end
    return np
end

local function executeSteal(prompt)
    local ct=tick()
    if ct-State.lastStealTick<STEAL_COOLDOWN then return end
    if State.isStealing then return end
    if not Steal.Data[prompt] then
        Steal.Data[prompt]={hold={},trigger={},ready=true}
        pcall(function()
            if getconnections then
                for _,c in ipairs(getconnections(prompt.PromptButtonHoldBegan)) do
                    if c.Function then table.insert(Steal.Data[prompt].hold,c.Function) end
                end
                for _,c in ipairs(getconnections(prompt.Triggered)) do
                    if c.Function then table.insert(Steal.Data[prompt].trigger,c.Function) end
                end
            else
                Steal.Data[prompt].useFallback=true
            end
        end)
    end
    local data=Steal.Data[prompt]
    if not data.ready then return end
    data.ready=false
    State.isStealing=true
    State.stealStartTime=ct
    State.lastStealTick=ct
    task.spawn(function()
        local ok=false
        pcall(function()
            if not data.useFallback then
                for _,fn in ipairs(data.hold) do task.spawn(fn) end
                task.wait(Steal.StealDuration)
                for _,fn in ipairs(data.trigger) do task.spawn(fn) end
                ok=true
            end
        end)
        if not ok and fireproximityprompt then
            pcall(function() fireproximityprompt(prompt); ok=true end)
        end
        if not ok then
            pcall(function()
                prompt:InputHoldBegin()
                task.wait(Steal.StealDuration)
                prompt:InputHoldEnd()
            end)
        end
        task.wait(Steal.StealDuration*0.3)
        task.wait(0.05)
        data.ready=true
        State.isStealing=false
    end)
end

local function startAutoSteal()
    if Conns.autoSteal then return end
    Conns.autoSteal=RunService.Heartbeat:Connect(function()
        if not Steal.AutoStealEnabled or State.isStealing then return end
        local p=findNearestPrompt()
        if p then executeSteal(p) end
    end)
end

local function stopAutoSteal()
    if Conns.autoSteal then Conns.autoSteal:Disconnect(); Conns.autoSteal=nil end
    State.isStealing=false
    State.lastStealTick=0
    Steal.plotCache={}
    Steal.plotCacheTime={}
    Steal.cachedPrompts={}
end


-- ============================================================
-- BUILD MENU SECTIONS
-- ============================================================

-- === AUTO SECTION ===
makeSectionHeader("AUTO")

toggleRefs.autoSteal = makeToggleRow("Auto Steal", false, function(on)
    Steal.AutoStealEnabled = on
    State.autoStealEnabled = on
    if on then
        if not pcall(startAutoSteal) then
            Steal.AutoStealEnabled=false
            if toggleRefs.autoSteal then toggleRefs.autoSteal(false) end
        end
    else
        stopAutoSteal()
    end
end)

toggleRefs.infJump = makeToggleRow("Infinite Jump", false, function(on)
    State.infJumpEnabled = on
end)

makeInputRow("Steal Radius", Steal.StealRadius, function(n)
    if n >= 5 and n <= 300 then
        Steal.StealRadius = math.floor(n)
        Steal.cachedPrompts={}
        Steal.promptCacheTime=0
        task.spawn(saveConfig)
    end
end)

makeInputRow("Steal Duration", Steal.StealDuration, function(n)
    if n >= 0.05 and n <= 2 then 
        Steal.StealDuration = n
        task.spawn(saveConfig)
    end
end)

makeDivider()

-- === COMBAT SECTION ===
makeSectionHeader("COMBAT")

toggleRefs.autoLeft = makeToggleRow("Auto Left", false, function(on)
    State.autoLeftEnabled = on
    if on then startAutoLeft() else stopAutoLeft() end
end, "autoLeft")

toggleRefs.autoRight = makeToggleRow("Auto Right", false, function(on)
    State.autoRightEnabled = on
    if on then startAutoRight() else stopAutoRight() end
end, "autoRight")

makeDivider()

-- === VISUALS SECTION ===
makeSectionHeader("VISUALS")

makeToggleRow("TP Down", false, function(on)
    if on then doTpDown() end
end, "tpDown")

makeDivider()

-- === SETTINGS SECTION ===
makeSectionHeader("SETTINGS")

makeInputRow("UI Scale", 1.0, function(n)
    if n >= 0.5 and n <= 2.0 then
        if uiScaleObj then uiScaleObj.Scale = n end
    end
end)

makeToggleRow("Lock UI", false, function(on)
    State.uiLocked = on
end)

toggleRefs.hideButtons = makeToggleRow("Hide Buttons", false, function(on)
    if mbGroup then mbGroup.Visible = not on end
end)


-- ============================================================
-- REOPEN BUTTON
-- ============================================================
local reopenBtn = Instance.new("TextButton", gui)
reopenBtn.Name = "FadedReopenBtn"
reopenBtn.Size = UDim2.new(0, 80, 0, 32)
reopenBtn.Position = UDim2.new(0, 10, 0, 10)
reopenBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
reopenBtn.BorderSizePixel = 0
reopenBtn.Text = "FADED"
reopenBtn.TextColor3 = Color3.fromRGB(160, 160, 160)
reopenBtn.Font = Enum.Font.GothamBlack
reopenBtn.TextSize = 14
reopenBtn.ZIndex = 200
reopenBtn.Visible = true
mkCorner(reopenBtn, 8)
mkStroke(reopenBtn, Color3.fromRGB(60, 60, 60), 1.5)
reopenBtn.MouseButton1Click:Connect(function()
    mainOuter.Visible = not mainOuter.Visible
end)

do
    local rbDragging, rbDragStart, rbStartPos = false, nil, nil
    reopenBtn.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1 or inp.UserInputType == Enum.UserInputType.Touch then
            rbDragging = true
            rbDragStart = inp.Position
            rbStartPos = reopenBtn.Position
            inp.Changed:Connect(function()
                if inp.UserInputState == Enum.UserInputState.End then rbDragging = false end
            end)
        end
    end)
    UIS.InputChanged:Connect(function(inp)
        if not rbDragging then return end
        if inp.UserInputType == Enum.UserInputType.MouseMovement or inp.UserInputType == Enum.UserInputType.Touch then
            local d = inp.Position - rbDragStart
            reopenBtn.Position = UDim2.new(
                rbStartPos.X.Scale, rbStartPos.X.Offset + d.X,
                rbStartPos.Y.Scale, rbStartPos.Y.Offset + d.Y
            )
        end
    end)
    UIS.InputEnded:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1 or inp.UserInputType == Enum.UserInputType.Touch then
            rbDragging = false
        end
    end)
end


-- ============================================================
-- MOBILE BUTTONS
-- ============================================================
local QS, QG, QR = 58, 6, 10
local Q_OFF = Color3.fromRGB(10, 10, 10)
local Q_ON = Color3.fromRGB(85, 85, 85)
local Q_BORDER = Color3.fromRGB(35, 35, 35)
local Q_BORDER_ON = Color3.fromRGB(160, 160, 160)
local Q_TEXT = Color3.fromRGB(255, 255, 255)

local QW = QS * 2 + QG
local QH = QS * 2 + QG

mbGroup = Instance.new("Frame", gui)
mbGroup.Name = "MobileButtons"
mbGroup.Size = UDim2.new(0, QW + 20, 0, QH + 20)
mbGroup.Position = UDim2.new(1, -QW - 34, 0.5, -QH/2 - 10)
mbGroup.BackgroundTransparency = 1
mbGroup.BorderSizePixel = 0
mbGroup.Active = true
mbGroup.ZIndex = 100

makeDraggable(mbGroup)

local function makeMobileBtn(label, col, row, isToggle, onAction)
    local relX = 10 + col*(QS+QG)
    local relY = 10 + row*(QS+QG)
    
    local frame = Instance.new("Frame", mbGroup)
    frame.Size = UDim2.new(0, QS, 0, QS)
    frame.Position = UDim2.new(0, relX, 0, relY)
    frame.BackgroundColor3 = Q_OFF
    frame.BorderSizePixel = 0
    frame.Active = true
    frame.ZIndex = 102
    mkCorner(frame, QR)
    
    local stroke = Instance.new("UIStroke", frame)
    stroke.Color = Q_BORDER
    stroke.Thickness = 1.5
    
    local btn = Instance.new("TextButton", frame)
    btn.Size = UDim2.new(1, 0, 1, 0)
    btn.BackgroundTransparency = 1
    btn.Text = label
    btn.TextColor3 = Q_TEXT
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 11
    btn.TextWrapped = true
    btn.LineHeight = 1.25
    btn.BorderSizePixel = 0
    btn.AutoButtonColor = false
    btn.ZIndex = 103
    
    local isOn = false
    btn.MouseButton1Click:Connect(function()
        if isToggle then
            isOn = not isOn
            TweenService:Create(frame, TweenInfo.new(0.15), {BackgroundColor3 = isOn and Q_ON or Q_OFF}):Play()
            TweenService:Create(stroke, TweenInfo.new(0.15), {Color = isOn and Q_BORDER_ON or Q_BORDER}):Play()
            if onAction then onAction(isOn) end
        else
            TweenService:Create(frame, TweenInfo.new(0.08), {BackgroundColor3 = Q_ON}):Play()
            TweenService:Create(stroke, TweenInfo.new(0.08), {Color = Q_BORDER_ON}):Play()
            task.delay(0.25, function()
                TweenService:Create(frame, TweenInfo.new(0.15), {BackgroundColor3 = Q_OFF}):Play()
                TweenService:Create(stroke, TweenInfo.new(0.15), {Color = Q_BORDER}):Play()
            end)
            if onAction then onAction() end
        end
    end)
    
    local function setter(s)
        isOn = s
        TweenService:Create(frame, TweenInfo.new(0.15), {BackgroundColor3 = s and Q_ON or Q_OFF}):Play()
        TweenService:Create(stroke, TweenInfo.new(0.15), {Color = s and Q_BORDER_ON or Q_BORDER}):Play()
    end
    
    return frame, setter
end

-- Row 0: AUTO LEFT | AUTO RIGHT
local _, setALfn = makeMobileBtn("AUTO\nLEFT", 0, 0, true, function(on)
    State.autoLeftEnabled = on
    if toggleRefs.autoLeft then toggleRefs.autoLeft(on) end
    if on then
        startAutoLeft()
    else
        stopAutoLeft()
    end
end)
setAL = setALfn

local _, setARfn = makeMobileBtn("AUTO\nRIGHT", 1, 0, true, function(on)
    State.autoRightEnabled = on
    if toggleRefs.autoRight then toggleRefs.autoRight(on) end
    if on then
        startAutoRight()
    else
        stopAutoRight()
    end
end)
setAR = setARfn

-- Row 1: TP DOWN | INF JUMP
makeMobileBtn("TP\nDOWN", 0, 1, false, function()
    doTpDown()
end)

makeMobileBtn("INF\nJUMP", 1, 1, true, function(on)
    State.infJumpEnabled = on
    if toggleRefs.infJump then toggleRefs.infJump(on) end
end)


-- ============================================================
-- CHARACTER SETUP
-- ============================================================
local function setupChar(char)
    task.wait(0.1)
    h=char:WaitForChild("Humanoid",5)
    hrp=char:WaitForChild("HumanoidRootPart",5)
    if not h or not hrp then return end
    
    local head=char:FindFirstChild("Head")
    if head then
        local oldBB=head:FindFirstChild("FadedBB")
        if oldBB then oldBB:Destroy() end
        
        local bb=Instance.new("BillboardGui",head)
        bb.Name="FadedBB"
        bb.Size=UDim2.new(0,180,0,76)
        bb.StudsOffset=Vector3.new(0,3,0)
        bb.AlwaysOnTop=true

        local speedBillLbl=Instance.new("TextLabel",bb)
        speedBillLbl.Name="SpeedBillLbl"
        speedBillLbl.Size=UDim2.new(1,0,0,24)
        speedBillLbl.Position=UDim2.new(0,0,0,26)
        speedBillLbl.BackgroundTransparency=1
        speedBillLbl.Text="0.0"
        speedBillLbl.TextColor3=Color3.fromRGB(210,210,210)
        speedBillLbl.Font=Enum.Font.GothamBlack
        speedBillLbl.TextScaled=true
        speedBillLbl.TextStrokeTransparency=0.1
        speedBillLbl.TextStrokeColor3=Color3.new(0,0,0)

        local lbl2=Instance.new("TextLabel",bb)
        lbl2.Size=UDim2.new(1,0,0,24)
        lbl2.Position=UDim2.new(0,0,0,52)
        lbl2.BackgroundTransparency=1
        lbl2.Text="/faded"
        lbl2.TextColor3=Color3.fromRGB(255,255,255)
        lbl2.Font=Enum.Font.GothamBold
        lbl2.TextScaled=true
        lbl2.TextStrokeTransparency=0.1
        lbl2.TextStrokeColor3=Color3.new(0,0,0)
    end
end

LP.CharacterAdded:Connect(setupChar)
if LP.Character then task.spawn(function() setupChar(LP.Character) end) end

-- ============================================================
-- RUNTIME LOOPS
-- ============================================================
local MOVE_KEYS={
    [Enum.KeyCode.W]=true,[Enum.KeyCode.A]=true,[Enum.KeyCode.S]=true,[Enum.KeyCode.D]=true,
    [Enum.KeyCode.Up]=true,[Enum.KeyCode.Left]=true,[Enum.KeyCode.Down]=true,[Enum.KeyCode.Right]=true
}

RunService.RenderStepped:Connect(function()
    if not (h and hrp) then return end
    
    if not State.autoLeftEnabled and not State.autoRightEnabled then
        local md=h.MoveDirection
        
        if md.Magnitude>0 then
            State.lastMoveDir=md
            hrp.Velocity=Vector3.new(md.X*50,hrp.Velocity.Y,md.Z*50)
        elseif State.lastMoveDir.Magnitude>0 then
            local anyHeld=false
            for key in pairs(MOVE_KEYS) do
                if UIS:IsKeyDown(key) then anyHeld=true; break end
            end
            if anyHeld then
                hrp.Velocity=Vector3.new(State.lastMoveDir.X*50,hrp.Velocity.Y,State.lastMoveDir.Z*50)
            end
        end
    end
    
    pcall(function()
        local head2=LP.Character and LP.Character:FindFirstChild("Head")
        if head2 then
            local bb2=head2:FindFirstChild("FadedBB")
            local sl=bb2 and bb2:FindFirstChild("SpeedBillLbl")
            if sl then
                sl.Text=string.format("%.1f",Vector3.new(hrp.Velocity.X,0,hrp.Velocity.Z).Magnitude)
            end
        end
    end)
end)

-- ============================================================
-- INPUT HANDLERS
-- ============================================================
UIS.InputBegan:Connect(function(inp, gp)
    if gp then return end
    local t = inp.UserInputType
    local isKb = t==Enum.UserInputType.Keyboard
    local isGP = t==Enum.UserInputType.Gamepad1 or t==Enum.UserInputType.Gamepad2
    if not isKb and not isGP then return end
    local kc = inp.KeyCode
    if kc == Enum.KeyCode.Unknown then return end
    
    if kc == Keys.autoLeft then
        State.autoLeftEnabled = not State.autoLeftEnabled
        if toggleRefs.autoLeft then toggleRefs.autoLeft(State.autoLeftEnabled) end
        if setAL then setAL(State.autoLeftEnabled) end
        if State.autoLeftEnabled then startAutoLeft() else stopAutoLeft() end
    elseif kc == Keys.autoRight then
        State.autoRightEnabled = not State.autoRightEnabled
        if toggleRefs.autoRight then toggleRefs.autoRight(State.autoRightEnabled) end
        if setAR then setAR(State.autoRightEnabled) end
        if State.autoRightEnabled then startAutoRight() else stopAutoRight() end
    elseif kc == Keys.tpDown then
        doTpDown()
    elseif kc == Keys.guiHide then
        mbGroup.Visible = not mbGroup.Visible
    end
end)

-- ============================================================
-- INIT
-- ============================================================
loadConfig()

mainOuter.Size = UDim2.new(0, 0, 0, 0)
mainOuter.Visible = true
TweenService:Create(mainOuter, TweenInfo.new(0.45, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
    Size = UDim2.new(0, WIN_W, 0, WIN_H)
}):Play()

task.spawn(function()
    while task.wait(10) do
        pcall(saveConfig)
    end
end)

print("[FADED.vs] ✅ Freemium version loaded!")
print("✓ TP Down: Keybind (V)")
print("✓ Infinite Jump: Working")
print("✓ Auto Left/Right: Working")
print("✓ Auto Steal: Working")
print("✓ Floating Buttons: Active")
print("✓ Config Saving: Active")
