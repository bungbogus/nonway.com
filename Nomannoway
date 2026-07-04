-- ====================================================================
-- Configuration
-- ====================================================================
local token = "" -- Set via Login UI
local channelID = "1519388781117964490"
local REFRESH_RATE = 3 

-- Services
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local CoreGui = game:GetService("CoreGui")
local TweenService = game:GetService("TweenService")

-- Executor Compatibility Check
local request = syn and syn.request or http and http.request or request
if not request then
    Players.LocalPlayer:Kick("No HTTP request function available!")
    return
end

local lastMessageId = ""
local isInitialLoad = true

-- Settings State
local clientSettings = {
    AutoJoin = false,
    Notifications = true,
    HideExpired = false,
    TargetKeywords = "" 
}

-- Forward Declarations
local fetchMessages 
local updateEmptyState

-- Clean setup
local existingGui = (CoreGui:FindFirstChild("DiscordChat") or Players.LocalPlayer.PlayerGui:FindFirstChild("DiscordChat"))
if existingGui then existingGui:Destroy() end

-- ====================================================================
-- HELPERS & ROBUST PARSING ENGINE
-- ====================================================================
local function decimalToColor3(decimal)
    if not decimal then return Color3.fromRGB(88, 101, 242) end
    local r = math.floor(decimal / 65536) % 256
    local g = math.floor(decimal / 256) % 256
    local b = decimal % 256
    return Color3.fromRGB(r, g, b)
end

local function formatTimestamp(timestamp)
    if not timestamp or type(timestamp) ~= "string" then return "Now" end
    local success, dt = pcall(function() return DateTime.fromISOString(timestamp) end)
    
    if not success or not dt then
        local year, month, day, hour, min, sec = string.match(timestamp, "(%d%d%d%d)%-(%d%d)%-(%d%d)[Tt](%d%d)%:(%d%d)%:(%d%d)")
        if year then
            success, dt = pcall(function()
                return DateTime.fromUnixTimestamp(os.time({
                    year = tonumber(year), month = tonumber(month), day = tonumber(day),
                    hour = tonumber(hour), min = tonumber(min), sec = tonumber(sec)
                }))
            end)
        end
    end
    
    if not success or not dt then return "1m" end
    
    local diff = math.abs(os.time() - dt.UnixTimestamp)
    if diff < 60 then return "Now"
    elseif diff < 3600 then return math.floor(diff / 60) .. "m"
    elseif diff < 86400 then return math.floor(diff / 3600) .. "h"
    else return math.floor(diff / 86400) .. "d" end
end

local function parseMarkdown(text)
    if type(text) ~= "string" then return "" end
    text = string.gsub(text, "```[a-zA-Z]*", "")
    text = string.gsub(text, "```", "")
    text = string.gsub(text, "``", "")
    text = string.gsub(text, "`", "")
    text = string.gsub(text, "<a?:([%w_]+):%d+>", ":%1:") 
    text = string.gsub(text, "<@&?(%d+)>", "@%1")        
    text = string.gsub(text, "<#(%d+)>", "#%1")          
    text = string.gsub(text, "%*%*%*(.-)%*%*%*", "<b><i>%1</i></b>")
    text = string.gsub(text, "%*% *", "**")
    text = string.gsub(text, "%*%*(.-)%*%*", "<b>%1</b>")
    text = string.gsub(text, "%*(.-)%*", "<i>%1</i>")
    text = string.gsub(text, "__(.-)__", "<u>%1</u>")
    text = string.gsub(text, "~~(.-)~~", "<s>%1</s>")
    return text
end

local function cleanAndExtractIds(text)
    if type(text) ~= "string" then return nil, nil end
    local clean = string.gsub(text, "%s+", "")
    
    local placeId = string.match(clean, "[Pp]lace[Ii]d[=:]%s*(%d+)") or
                    string.match(clean, "TeleportToPlaceInstance%((%d+)") or
                    string.match(clean, "Teleport%((%d+)") or
                    string.match(clean, "games/(%d+)")
                    
    local jobId = string.match(clean, "[Gg]ame[Ii]nstance[Ii]d[=:]%s*[\"']?([%w%-]+)[\"']?") or
                  string.match(clean, "[Jj]ob[Ii]d[=:]%s*[\"']?([%w%-]+)[\"']?")
                  
    if not placeId then placeId = string.match(clean, "(%d%d%d%d%d%d%d%d%d+)") end
    if not jobId then jobId = string.match(clean, "([%w]+%-[%w]+%-[%w]+%-[%w]+%-[%w]+)") end
    
    return placeId, jobId
end

local function extractIdsFromMessage(msg)
    local foundPlaceId, foundJobId
    local p, j = cleanAndExtractIds(msg.content)
    if p then foundPlaceId, foundJobId = p, j end
    
    if msg.embeds then
        for _, embed in ipairs(msg.embeds) do
            p, j = cleanAndExtractIds(embed.url)
            if p then foundPlaceId, foundJobId = p, (j or foundJobId) end
            p, j = cleanAndExtractIds(embed.description)
            if p then foundPlaceId, foundJobId = p, (j or foundJobId) end
            if embed.fields then
                for _, field in ipairs(embed.fields) do
                    p, j = cleanAndExtractIds(field.value)
                    if p then foundPlaceId, foundJobId = p, (j or foundJobId) end
                end
            end
        end
    end
    return foundPlaceId, foundJobId
end

local function passesKeywordFilter(msg)
    if not clientSettings.TargetKeywords or clientSettings.TargetKeywords:match("^%s*$") then
        return true 
    end

    local searchString = string.lower((msg.content or "") .. " ")
    if msg.embeds then
        for _, emb in ipairs(msg.embeds) do
            searchString = searchString .. string.lower(emb.title or "") .. " "
            searchString = searchString .. string.lower(emb.description or "") .. " "
            if emb.fields then
                for _, fld in ipairs(emb.fields) do
                    searchString = searchString .. string.lower(fld.name or "") .. " "
                    searchString = searchString .. string.lower(fld.value or "") .. " "
                end
            end
        end
    end

    for keyword in string.gmatch(string.lower(clientSettings.TargetKeywords), "[^,]+") do
        keyword = keyword:match("^%s*(.-)%s*$") 
        if keyword ~= "" and string.find(searchString, keyword, 1, true) then
            return true 
        end
    end

    return false
end

-- ====================================================================
-- INTERFACE STRUCTURE
-- ====================================================================
local gui = Instance.new("ScreenGui")
gui.Name = "DiscordChat"
gui.ResetOnSpawn = false
local success = pcall(function() gui.Parent = CoreGui end)
if not success then gui.Parent = Players.LocalPlayer:WaitForChild("PlayerGui") end

-- TOGGLE BUTTON (Hidden until Login)
local toggleBtn = Instance.new("ImageButton")
toggleBtn.Size = UDim2.new(0, 34, 0, 34) 
toggleBtn.Position = UDim2.new(0, 12, 0, 15)
toggleBtn.BackgroundColor3 = Color3.fromRGB(43, 45, 49)
toggleBtn.Image = "rbxthumb://type=Asset&id=8248378219&w=150&h=150" 
toggleBtn.ScaleType = Enum.ScaleType.Fit
toggleBtn.Visible = false
toggleBtn.Parent = gui
Instance.new("UICorner", toggleBtn).CornerRadius = UDim.new(0, 6)
Instance.new("UIStroke", toggleBtn).Color = Color3.fromRGB(65, 67, 74)

-- LOGIN UI
local loginFrame = Instance.new("Frame")
loginFrame.Size = UDim2.new(0, 260, 0, 140)
loginFrame.Position = UDim2.new(0.5, -130, 0.5, -70)
loginFrame.BackgroundColor3 = Color3.fromRGB(49, 51, 56)
loginFrame.Parent = gui
Instance.new("UICorner", loginFrame).CornerRadius = UDim.new(0, 8)
Instance.new("UIStroke", loginFrame).Color = Color3.fromRGB(30, 31, 34)

local loginTitle = Instance.new("TextLabel")
loginTitle.Size = UDim2.new(1, 0, 0, 30)
loginTitle.Position = UDim2.new(0, 0, 0, 10)
loginTitle.BackgroundTransparency = 1
loginTitle.Text = "Discord Login"
loginTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
loginTitle.Font = Enum.Font.GothamBold
loginTitle.TextSize = 16
loginTitle.Parent = loginFrame

local tokenInput = Instance.new("TextBox")
tokenInput.Size = UDim2.new(1, -40, 0, 35)
tokenInput.Position = UDim2.new(0, 20, 0, 45)
tokenInput.BackgroundColor3 = Color3.fromRGB(30, 31, 34)
tokenInput.TextColor3 = Color3.fromRGB(218, 222, 225)
tokenInput.PlaceholderText = "Paste Token Here..."
tokenInput.PlaceholderColor3 = Color3.fromRGB(100, 105, 110)
tokenInput.Font = Enum.Font.Gotham
tokenInput.TextSize = 12
tokenInput.ClearTextOnFocus = false
tokenInput.Parent = loginFrame
Instance.new("UICorner", tokenInput).CornerRadius = UDim.new(0, 4)
Instance.new("UIStroke", tokenInput).Color = Color3.fromRGB(65, 67, 74)

local loginBtn = Instance.new("TextButton")
loginBtn.Size = UDim2.new(1, -40, 0, 35)
loginBtn.Position = UDim2.new(0, 20, 0, 90)
loginBtn.BackgroundColor3 = Color3.fromRGB(88, 101, 242)
loginBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
loginBtn.Text = "Connect"
loginBtn.Font = Enum.Font.GothamBold
loginBtn.TextSize = 14
loginBtn.Parent = loginFrame
Instance.new("UICorner", loginBtn).CornerRadius = UDim.new(0, 4)

-- MAIN CONTAINER
local container = Instance.new("Frame")
container.Size = UDim2.new(0, 225, 0.68, 0) 
container.Position = UDim2.new(0, 12, 0.15, 0) 
container.AnchorPoint = Vector2.new(0, 0)
container.BackgroundColor3 = Color3.fromRGB(49, 51, 56)
container.ClipsDescendants = true
container.Visible = false
container.Parent = gui
Instance.new("UICorner", container).CornerRadius = UDim.new(0, 6)
Instance.new("UIStroke", container).Color = Color3.fromRGB(30, 31, 34)

local function setPanelVisibility(visible) container.Visible = visible end

local header = Instance.new("Frame")
header.Size = UDim2.new(1, 0, 0, 32)
header.BackgroundColor3 = Color3.fromRGB(30, 31, 34)
header.BorderSizePixel = 0
header.Parent = container
Instance.new("UICorner", header).CornerRadius = UDim.new(0, 6)

local headerFix = Instance.new("Frame")
headerFix.Size = UDim2.new(1, 0, 0, 6)
headerFix.Position = UDim2.new(0, 0, 1, -6)
headerFix.BackgroundColor3 = Color3.fromRGB(30, 31, 34)
headerFix.BorderSizePixel = 0
headerFix.Parent = header

local titleLabel = Instance.new("TextLabel")
titleLabel.Size = UDim2.new(0.4, 0, 1, 0)
titleLabel.Position = UDim2.new(0, 10, 0, 0)
titleLabel.BackgroundTransparency = 1
titleLabel.Text = "Discord Feed"
titleLabel.TextColor3 = Color3.fromRGB(242, 243, 245)
titleLabel.TextSize = 12
titleLabel.Font = Enum.Font.GothamBold
titleLabel.Parent = header
titleLabel.TextXAlignment = Enum.TextXAlignment.Left

-- Header Buttons (Updated for Symbol fixes)
local refreshBtn = Instance.new("TextButton")
refreshBtn.Size = UDim2.new(0, 28, 0, 28)
refreshBtn.Position = UDim2.new(1, -88, 0.5, -14)
refreshBtn.BackgroundTransparency = 1
refreshBtn.Text = "↻"
refreshBtn.TextColor3 = Color3.fromRGB(148, 155, 164)
refreshBtn.TextSize = 20
refreshBtn.Font = Enum.Font.SourceSansBold 
refreshBtn.Parent = header

local settingsBtn = Instance.new("TextButton")
settingsBtn.Size = UDim2.new(0, 28, 0, 28)
settingsBtn.Position = UDim2.new(1, -60, 0.5, -14)
settingsBtn.BackgroundTransparency = 1
settingsBtn.Text = "⚙"
settingsBtn.TextColor3 = Color3.fromRGB(148, 155, 164)
settingsBtn.TextSize = 20
settingsBtn.Font = Enum.Font.SourceSansBold 
settingsBtn.Parent = header

local closeBtn = Instance.new("TextButton")
closeBtn.Size = UDim2.new(0, 28, 0, 28)
closeBtn.Position = UDim2.new(1, -32, 0.5, -14)
closeBtn.BackgroundTransparency = 1
closeBtn.Text = "X"
closeBtn.TextColor3 = Color3.fromRGB(148, 155, 164)
closeBtn.TextSize = 14
closeBtn.Font = Enum.Font.GothamBold
closeBtn.Parent = header
closeBtn.MouseButton1Click:Connect(function() setPanelVisibility(false) end)

local messageContainer = Instance.new("ScrollingFrame")
messageContainer.Size = UDim2.new(1, 0, 1, -32)
messageContainer.Position = UDim2.new(0, 0, 0, 32)
messageContainer.BackgroundColor3 = Color3.fromRGB(49, 51, 56)
messageContainer.BorderSizePixel = 0
messageContainer.ScrollBarThickness = 2
messageContainer.ScrollBarImageColor3 = Color3.fromRGB(43, 45, 49)
messageContainer.AutomaticCanvasSize = Enum.AutomaticSize.Y
messageContainer.Parent = container

local padding = Instance.new("UIPadding")
padding.PaddingLeft = UDim.new(0, 4)
padding.PaddingRight = UDim.new(0, 4)
padding.PaddingTop = UDim.new(0, 4)
padding.PaddingBottom = UDim.new(0, 4)
padding.Parent = messageContainer

local listLayout = Instance.new("UIListLayout")
listLayout.SortOrder = Enum.SortOrder.LayoutOrder
listLayout.Padding = UDim.new(0, 4)
listLayout.Parent = messageContainer

-- Empty State Label
local noPetsLabel = Instance.new("TextLabel")
noPetsLabel.Name = "NoPetsLabel"
noPetsLabel.Size = UDim2.new(1, 0, 0, 30)
noPetsLabel.BackgroundTransparency = 1
noPetsLabel.Text = "⏳ Wait for Pets to Spawn!"
noPetsLabel.TextColor3 = Color3.fromRGB(148, 155, 164)
noPetsLabel.TextSize = 11
noPetsLabel.Font = Enum.Font.GothamMedium
noPetsLabel.Visible = false
noPetsLabel.LayoutOrder = 0 
noPetsLabel.Parent = messageContainer

updateEmptyState = function()
    local hasVisibleMessage = false
    for _, child in ipairs(messageContainer:GetChildren()) do
        if child:IsA("Frame") and child.Visible then
            hasVisibleMessage = true
            break
        end
    end
    noPetsLabel.Visible = not hasVisibleMessage
end

-- ====================================================================
-- SETTINGS PANEL UI 
-- ====================================================================
local settingsPanel = Instance.new("ScrollingFrame")
settingsPanel.Size = UDim2.new(1, 0, 1, -32)
settingsPanel.Position = UDim2.new(0, 0, 0, 32)
settingsPanel.BackgroundColor3 = Color3.fromRGB(43, 45, 49)
settingsPanel.BorderSizePixel = 0
settingsPanel.ScrollBarThickness = 2
settingsPanel.AutomaticCanvasSize = Enum.AutomaticSize.Y
settingsPanel.Visible = false
settingsPanel.Parent = container

local settingsPad = Instance.new("UIPadding")
settingsPad.PaddingLeft = UDim.new(0, 10)
settingsPad.PaddingRight = UDim.new(0, 10)
settingsPad.PaddingTop = UDim.new(0, 12)
settingsPad.PaddingBottom = UDim.new(0, 12)
settingsPad.Parent = settingsPanel

local settingsLayout = Instance.new("UIListLayout")
settingsLayout.SortOrder = Enum.SortOrder.LayoutOrder
settingsLayout.Padding = UDim.new(0, 12)
settingsLayout.Parent = settingsPanel

local function createToggleSetting(labelText, settingKey, layoutOrder)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 24)
    row.BackgroundTransparency = 1
    row.LayoutOrder = layoutOrder
    row.Parent = settingsPanel

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(0.7, 0, 1, 0)
    label.BackgroundTransparency = 1
    label.Text = labelText
    label.TextColor3 = Color3.fromRGB(242, 243, 245)
    label.TextSize = 11
    label.Font = Enum.Font.GothamMedium
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = row

    local toggleBox = Instance.new("TextButton")
    toggleBox.Size = UDim2.new(0, 40, 0, 20)
    toggleBox.Position = UDim2.new(1, -40, 0, 2)
    toggleBox.Text = ""
    toggleBox.Parent = row
    Instance.new("UICorner", toggleBox).CornerRadius = UDim.new(1, 0)

    local indicator = Instance.new("Frame")
    indicator.Size = UDim2.new(0, 16, 0, 16)
    indicator.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    indicator.Parent = toggleBox
    Instance.new("UICorner", indicator).CornerRadius = UDim.new(1, 0)

    local function updateVisuals()
        if clientSettings[settingKey] then
            toggleBox.BackgroundColor3 = Color3.fromRGB(36, 128, 70) 
            TweenService:Create(indicator, TweenInfo.new(0.2), {Position = UDim2.new(1, -18, 0.5, -8)}):Play()
        else
            toggleBox.BackgroundColor3 = Color3.fromRGB(65, 67, 74) 
            TweenService:Create(indicator, TweenInfo.new(0.2), {Position = UDim2.new(0, 2, 0.5, -8)}):Play()
        end
    end
    
    updateVisuals()
    toggleBox.MouseButton1Click:Connect(function()
        clientSettings[settingKey] = not clientSettings[settingKey]
        updateVisuals()
        
        if settingKey == "HideExpired" and fetchMessages then
            task.spawn(function() fetchMessages(true) end)
        end
    end)
end

createToggleSetting("Auto-Join on Alert", "AutoJoin", 1)

local filterContainer = Instance.new("Frame")
filterContainer.Size = UDim2.new(1, 0, 0, 50)
filterContainer.BackgroundTransparency = 1
filterContainer.LayoutOrder = 2
filterContainer.Parent = settingsPanel

local filterLabel = Instance.new("TextLabel")
filterLabel.Size = UDim2.new(1, 0, 0, 16)
filterLabel.BackgroundTransparency = 1
filterLabel.Text = "Keyword Filter (Comma separated):"
filterLabel.TextColor3 = Color3.fromRGB(148, 155, 164)
filterLabel.TextSize = 10
filterLabel.Font = Enum.Font.GothamMedium
filterLabel.TextXAlignment = Enum.TextXAlignment.Left
filterLabel.Parent = filterContainer

local filterBox = Instance.new("TextBox")
filterBox.Size = UDim2.new(1, 0, 0, 28)
filterBox.Position = UDim2.new(0, 0, 0, 20)
filterBox.BackgroundColor3 = Color3.fromRGB(30, 31, 34)
filterBox.TextColor3 = Color3.fromRGB(218, 222, 225)
filterBox.PlaceholderText = "e.g. Huge, Raccoon"
filterBox.PlaceholderColor3 = Color3.fromRGB(100, 105, 110)
filterBox.Text = ""
filterBox.TextSize = 11
filterBox.Font = Enum.Font.Gotham
filterBox.TextXAlignment = Enum.TextXAlignment.Left
filterBox.ClearTextOnFocus = false
filterBox.Parent = filterContainer
Instance.new("UICorner", filterBox).CornerRadius = UDim.new(0, 4)
local fbPadding = Instance.new("UIPadding", filterBox)
fbPadding.PaddingLeft = UDim.new(0, 8)
fbPadding.PaddingRight = UDim.new(0, 8)
Instance.new("UIStroke", filterBox).Color = Color3.fromRGB(65, 67, 74)

filterBox.FocusLost:Connect(function() clientSettings.TargetKeywords = filterBox.Text end)

createToggleSetting("Show Notifications", "Notifications", 3)
createToggleSetting("Hide Expired Pets", "HideExpired", 4)

settingsBtn.MouseButton1Click:Connect(function()
    settingsPanel.Visible = not settingsPanel.Visible
    messageContainer.Visible = not settingsPanel.Visible
    settingsBtn.TextColor3 = settingsPanel.Visible and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(148, 155, 164)
end)

-- ====================================================================
-- ACHIEVEMENT ROBLOX NOTIFICATION ENGINE
-- ====================================================================
local notifSound = Instance.new("Sound")
notifSound.SoundId = "rbxassetid://138079535" 
notifSound.Volume = 0.5
notifSound.Parent = game:GetService("SoundService")

local function showAchievementNotification(msg)
    if not clientSettings.Notifications then return end
    if container.Visible then return end

    local petName = "New Notification"
    local petImageUrl = nil
    
    if msg.embeds and msg.embeds[1] then
        petName = msg.embeds[1].title or msg.author.username
        petImageUrl = (msg.embeds[1].thumbnail and msg.embeds[1].thumbnail.url) or 
                      (msg.embeds[1].image and msg.embeds[1].image.url)
    end

    local notifFrame = Instance.new("Frame")
    notifFrame.Size = UDim2.new(0, 280, 0, 55)
    notifFrame.Position = UDim2.new(0.5, -140, 0, -70)
    notifFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
    notifFrame.BorderSizePixel = 0
    notifFrame.ClipsDescendants = true
    notifFrame.Parent = gui

    Instance.new("UICorner", notifFrame).CornerRadius = UDim.new(0, 8)
    local stroke = Instance.new("UIStroke", notifFrame)
    stroke.Color = Color3.fromRGB(255, 200, 0)
    stroke.Thickness = 1.5

    local petImg = Instance.new("ImageLabel")
    petImg.Size = UDim2.new(0, 38, 0, 38)
    petImg.Position = UDim2.new(0, 10, 0.5, -19)
    petImg.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
    petImg.Parent = notifFrame
    Instance.new("UICorner", petImg).CornerRadius = UDim.new(0, 6)

    if petImageUrl then
        task.spawn(function()
            pcall(function()
                local hash = 0
                for i = 1, #petImageUrl do hash = ((hash * 33) + string.byte(petImageUrl, i)) % 2^32 end
                local file = string.format("pet_img_%x.png", hash)
                if not isfile(file) then
                    local res = request({Url = petImageUrl, Method = "GET"})
                    if res and res.StatusCode == 200 then writefile(file, res.Body) end
                end
                if isfile(file) then petImg.Image = getcustomasset(file) end
            end)
        end)
    end

    local nameLabel = Instance.new("TextLabel")
    nameLabel.Size = UDim2.new(1, -65, 0, 16)
    nameLabel.Position = UDim2.new(0, 56, 0, 10)
    nameLabel.BackgroundTransparency = 1
    nameLabel.Font = Enum.Font.GothamBold
    nameLabel.TextSize = 11
    nameLabel.TextColor3 = Color3.fromRGB(255, 215, 0)
    nameLabel.TextXAlignment = Enum.TextXAlignment.Left
    nameLabel.Text = petName 
    nameLabel.Parent = notifFrame

    local textLabel = Instance.new("TextLabel")
    textLabel.Size = UDim2.new(1, -65, 0, 22)
    textLabel.Position = UDim2.new(0, 56, 0, 24)
    textLabel.BackgroundTransparency = 1
    textLabel.Font = Enum.Font.GothamMedium
    textLabel.TextSize = 10
    textLabel.TextColor3 = Color3.fromRGB(240, 240, 240)
    textLabel.TextXAlignment = Enum.TextXAlignment.Left
    textLabel.Text = "Check your feed!"
    textLabel.Parent = notifFrame

    pcall(function() notifSound:Play() end)
    local tweenIn = TweenService:Create(notifFrame, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
        Position = UDim2.new(0.5, -140, 0, 20)
    })
    tweenIn:Play()

    task.wait(4.5)
    
    local tweenOut = TweenService:Create(notifFrame, TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
        Position = UDim2.new(0.5, -140, 0, -70)
    })
    tweenOut:Play()
    tweenOut.Completed:Connect(function() notifFrame:Destroy() end)
end

-- ====================================================================
-- LIVE TIME ENGINE & RENDERER
-- ====================================================================
local function startLiveRender(label, rawText, msgTimestampISO, getJoinBtn, getParentFrame)
    if type(rawText) ~= "string" or rawText == "" then return end

    local targetUnix = nil
    local isDiscordTs = false
    
    local tsMatch = string.match(rawText, "<t:(%d+)")
    if tsMatch then
        targetUnix = tonumber(tsMatch)
        isDiscordTs = true
    else
        local lowerText = string.lower(rawText)
        local minPart, secPart = string.match(lowerText, "(%d+)%:%s*(%d%d)")
        local m = string.match(lowerText, "(%d+)%s*m") or string.match(lowerText, "(%d+)%s*min")
        local s = string.match(lowerText, "(%d+)%s*s") or string.match(lowerText, "(%d+)%s*sec")
        
        local msgCreated = 0
        pcall(function() msgCreated = DateTime.fromISOString(msgTimestampISO).UnixTimestamp end)
        if msgCreated == 0 then msgCreated = os.time() end

        if minPart and secPart then
            targetUnix = msgCreated + (tonumber(minPart) * 60) + tonumber(secPart)
        elseif m or s then
            targetUnix = msgCreated + (tonumber(m or 0) * 60) + tonumber(s or 0)
        end
    end

    if not targetUnix then
        local formattedText = string.gsub(rawText, "<t:(%d+):?([%a]?)>", function(ts, flag)
            local ut = tonumber(ts)
            if not ut then return "Time" end
            local s, dt = pcall(function() return DateTime.fromUnixTimestamp(ut) end)
            if not s or not dt then return "Time" end
            if flag == "t" or flag == "T" then return dt:FormatLocalTime("LT", "en-us") end
            return dt:FormatLocalTime("lll", "en-us")
        end)
        label.Text = parseMarkdown(formattedText)
        return
    end

    task.spawn(function()
        while label and label.Parent do
            local currentUnix = os.time()
            local remaining = targetUnix - currentUnix
            local button = getJoinBtn()
            local frame = getParentFrame()
            
            if remaining <= 0 then
                if clientSettings.HideExpired and frame then
                    frame.Visible = false
                    updateEmptyState() 
                    
                    task.defer(function()
                        if messageContainer then
                            messageContainer.CanvasPosition = Vector2.new(0, messageContainer.AbsoluteCanvasSize.Y)
                        end
                    end)
                    break
                end
                
                label.Text = parseMarkdown(string.gsub(rawText, "<t:%d+:?[%a]?>", '❌ <font color="#F23F42"><b>EXPIRED</b></font>'))
                if button then
                    button.Text = "❌ Expired"
                    button.BackgroundColor3 = Color3.fromRGB(43, 45, 49)
                    button.TextColor3 = Color3.fromRGB(148, 155, 164)
                end
                break
            else
                local minutes = math.floor(remaining / 60)
                local seconds = remaining % 60
                local liveTimeText = string.format('⏳ <font color="#F2B90B"><b>%02dm %02ds left</b></font>', minutes, seconds)

                local workText = rawText
                if isDiscordTs then
                    workText = string.gsub(workText, "<t:%d+:?[%a]?>", liveTimeText)
                else
                    workText = string.gsub(workText, "%d+:%d%d", liveTimeText)
                    workText = string.gsub(workText, "%d+%s*m%s*%d+%s*s", liveTimeText)
                    if not string.find(workText, "left") then
                        workText = workText .. " " .. liveTimeText
                    end
                end
                label.Text = parseMarkdown(workText)
            end
            task.wait(1)
        end
    end)
end

local function loadExternalImage(url, parent)
    if not url or not getcustomasset or not writefile or not isfile then return end
    local hash = 5381
    for i = 1, #url do hash = ((hash * 33) + string.byte(url, i)) % 2^32 end
    local fileName = string.format("discord_asset_%x.png", hash)

    local img = Instance.new("ImageLabel")
    img.Size = UDim2.new(1, 0, 0, 65)
    img.BackgroundTransparency = 1
    img.ScaleType = Enum.ScaleType.Fit
    img.LayoutOrder = 99
    img.Parent = parent
    
    task.spawn(function()
        pcall(function()
            if not isfile(fileName) then
                local res = request({Url = url, Method = "GET"})
                if res and res.StatusCode == 200 then writefile(fileName, res.Body) end
            end
            if isfile(fileName) then img.Image = getcustomasset(fileName) end
        end)
    end)
end

local function createMessageUI(msg)
    local foundPlaceId, foundJobId = extractIdsFromMessage(msg)
    local joinBtn = nil 
    local frame = Instance.new("Frame")
    
    local function getJoinButtonRef() return joinBtn end
    local function getParentFrameRef() return frame end
    
    frame.Size = UDim2.new(1, 0, 0, 0)
    frame.BackgroundColor3 = Color3.fromRGB(43, 45, 49)
    frame.AutomaticSize = Enum.AutomaticSize.Y
    frame.Parent = messageContainer
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 4)
    Instance.new("UIStroke", frame).Color = Color3.fromRGB(56, 58, 62)
    
    local frameLayout = Instance.new("UIListLayout")
    frameLayout.SortOrder = Enum.SortOrder.LayoutOrder
    frameLayout.Padding = UDim.new(0, 2)
    frameLayout.Parent = frame
    
    local pad = Instance.new("UIPadding")
    pad.PaddingLeft = UDim.new(0, 6)
    pad.PaddingRight = UDim.new(0, 6)
    pad.PaddingTop = UDim.new(0, 4)
    pad.PaddingBottom = UDim.new(0, 4)
    pad.Parent = frame

    local layoutTracker = 1

    local authorRow = Instance.new("Frame")
    authorRow.Size = UDim2.new(1, 0, 0, 14)
    authorRow.BackgroundTransparency = 1
    authorRow.LayoutOrder = layoutTracker
    authorRow.Parent = frame
    layoutTracker = layoutTracker + 1

    local authorLbl = Instance.new("TextLabel")
    authorLbl.Size = UDim2.new(1, 0, 1, 0)
    authorLbl.BackgroundTransparency = 1
    authorLbl.RichText = true
    
    local nameString = msg.author and msg.author.username or "System"
    local timeString = formatTimestamp(msg.timestamp)
    authorLbl.Text = string.format('<b>%s</b>   <font color="#949FA4" size="9">%s</font>', nameString, timeString)
    authorLbl.TextColor3 = Color3.fromRGB(242, 243, 245)
    authorLbl.TextSize = 10 
    authorLbl.TextXAlignment = Enum.TextXAlignment.Left
    authorLbl.Font = Enum.Font.Gotham
    authorLbl.Parent = authorRow

    if msg.content and msg.content ~= "" then
        local loweredContent = string.lower(msg.content)
        if not string.match(loweredContent, "teleportservice") and not string.match(loweredContent, "project%-reverse") then
            local contentLbl = Instance.new("TextLabel")
            contentLbl.Size = UDim2.new(1, 0, 0, 0)
            contentLbl.BackgroundTransparency = 1
            contentLbl.RichText = true
            contentLbl.TextColor3 = Color3.fromRGB(218, 222, 225)
            contentLbl.TextSize = 10 
            contentLbl.TextWrapped = true
            contentLbl.TextXAlignment = Enum.TextXAlignment.Left
            contentLbl.Font = Enum.Font.GothamMedium
            contentLbl.AutomaticSize = Enum.AutomaticSize.Y
            contentLbl.LayoutOrder = layoutTracker
            contentLbl.Parent = frame
            layoutTracker = layoutTracker + 1
            
            startLiveRender(contentLbl, msg.content, msg.timestamp, getJoinButtonRef, getParentFrameRef)
        end
    end

    if msg.embeds and #msg.embeds > 0 then
        for _, embed in ipairs(msg.embeds) do
            local embedWrapper = Instance.new("Frame")
            embedWrapper.Size = UDim2.new(1, 0, 0, 0)
            embedWrapper.BackgroundColor3 = Color3.fromRGB(49, 51, 56)
            embedWrapper.AutomaticSize = Enum.AutomaticSize.Y
            embedWrapper.LayoutOrder = layoutTracker
            embedWrapper.Parent = frame
            Instance.new("UICorner", embedWrapper).CornerRadius = UDim.new(0, 3)
            layoutTracker = layoutTracker + 1

            local colorBar = Instance.new("Frame")
            colorBar.Size = UDim2.new(0, 2, 1, 0)
            colorBar.BackgroundColor3 = decimalToColor3(embed.color)
            colorBar.BorderSizePixel = 0
            colorBar.Parent = embedWrapper
            Instance.new("UICorner", colorBar).CornerRadius = UDim.new(0, 3)

            local embedInner = Instance.new("Frame")
            embedInner.Size = UDim2.new(1, -10, 0, 0)
            embedInner.Position = UDim2.new(0, 8, 0, 0)
            embedInner.BackgroundTransparency = 1
            embedInner.AutomaticSize = Enum.AutomaticSize.Y
            embedInner.Parent = embedWrapper

            local embedPadding = Instance.new("UIPadding")
            embedPadding.PaddingLeft = UDim.new(0, 4)
            embedPadding.PaddingRight = UDim.new(0, 4)
            embedPadding.PaddingTop = UDim.new(0, 3)
            embedPadding.PaddingBottom = UDim.new(0, 3)
            embedPadding.Parent = embedInner

            local embedLayout = Instance.new("UIListLayout")
            embedLayout.SortOrder = Enum.SortOrder.LayoutOrder
            embedLayout.Padding = UDim.new(0, 1)
            embedLayout.Parent = embedInner
            
            local eTracker = 1

            if embed.title then
                local eTitle = Instance.new("TextLabel")
                eTitle.Size = UDim2.new(1, 0, 0, 0)
                eTitle.BackgroundTransparency = 1
                eTitle.Text = parseMarkdown(embed.title)
                eTitle.RichText = true
                eTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
                eTitle.TextSize = 10
                eTitle.TextWrapped = true
                eTitle.TextXAlignment = Enum.TextXAlignment.Left
                eTitle.Font = Enum.Font.GothamBold
                eTitle.AutomaticSize = Enum.AutomaticSize.Y
                eTitle.LayoutOrder = eTracker
                eTracker = eTracker + 1
                eTitle.Parent = embedInner
            end
            
            if embed.description then
                local loweredDesc = string.lower(embed.description)
                if not string.match(loweredDesc, "teleportservice") and not string.match(loweredDesc, "project%-reverse") then
                    local eDesc = Instance.new("TextLabel")
                    eDesc.Size = UDim2.new(1, 0, 0, 0)
                    eDesc.BackgroundTransparency = 1
                    eDesc.RichText = true
                    eDesc.TextColor3 = Color3.fromRGB(218, 222, 225)
                    eDesc.TextSize = 10
                    eDesc.TextWrapped = true
                    eDesc.TextXAlignment = Enum.TextXAlignment.Left
                    eDesc.Font = Enum.Font.GothamMedium
                    eDesc.AutomaticSize = Enum.AutomaticSize.Y
                    eDesc.LayoutOrder = eTracker
                    eTracker = eTracker + 1
                    eDesc.Parent = embedInner
                    
                    startLiveRender(eDesc, embed.description, msg.timestamp, getJoinButtonRef, getParentFrameRef)
                end
            end

            if embed.fields then
                for _, field in ipairs(embed.fields) do
                    local lowerName = string.lower(field.name or "")
                    local lowerValue = string.lower(field.value or "")
                    
                    local isTimeField = string.match(lowerValue, "<t:") or string.match(lowerValue, "%d+%s*m") or string.match(lowerValue, "%d+:%d%d") or string.match(lowerName, "despawn") or string.match(lowerName, "expire") or string.match(lowerValue, "left")
                    
                    if not isTimeField then
                        if string.match(lowerValue, "teleport") or string.match(lowerValue, "loadstring") or string.match(lowerValue, "http") or string.match(lowerName, "script") or string.match(lowerName, "join") then
                            continue
                        end
                    end
                    
                    local fName = Instance.new("TextLabel")
                    fName.Size = UDim2.new(1, 0, 0, 0)
                    fName.BackgroundTransparency = 1
                    fName.Text = parseMarkdown(field.name or "")
                    fName.RichText = true
                    fName.TextColor3 = Color3.fromRGB(148, 155, 164)
                    fName.TextSize = 9 
                    fName.TextWrapped = true
                    fName.TextXAlignment = Enum.TextXAlignment.Left
                    fName.Font = Enum.Font.GothamBold
                    fName.AutomaticSize = Enum.AutomaticSize.Y
                    fName.LayoutOrder = eTracker
                    fName.Parent = embedInner
                    eTracker = eTracker + 1

                    local fValue = Instance.new("TextLabel")
                    fValue.Size = UDim2.new(1, 0, 0, 0)
                    fValue.BackgroundTransparency = 1
                    fValue.RichText = true
                    fValue.TextColor3 = Color3.fromRGB(230, 233, 235)
                    fValue.TextSize = 10
                    fValue.TextWrapped = true
                    fValue.TextXAlignment = Enum.TextXAlignment.Left
                    fValue.Font = Enum.Font.GothamMedium
                    fValue.AutomaticSize = Enum.AutomaticSize.Y
                    fValue.LayoutOrder = eTracker
                    fValue.Parent = embedInner
                    eTracker = eTracker + 1

                    startLiveRender(fValue, field.value or "", msg.timestamp, getJoinButtonRef, getParentFrameRef)
                end
            end

            if embed.thumbnail and embed.thumbnail.url then
                loadExternalImage(embed.thumbnail.url, embedInner)
            end
            if embed.image and embed.image.url then
                loadExternalImage(embed.image.url, embedInner)
            end
        end
    end

    if foundPlaceId then
        joinBtn = Instance.new("TextButton")
        joinBtn.Size = UDim2.new(1, 0, 0, 24) 
        joinBtn.Font = Enum.Font.GothamBold
        joinBtn.TextSize = 10
        joinBtn.LayoutOrder = 999 
        joinBtn.Parent = frame
        Instance.new("UICorner", joinBtn).CornerRadius = UDim.new(0, 3)
        
        local initialTarget = nil
        local tsCheck = string.match(msg.content, "<t:(%d+)")
        if tsCheck then
            initialTarget = tonumber(tsCheck)
        else
            if msg.embeds then
                for _, emb in ipairs(msg.embeds) do
                    local embedTs = string.match(emb.description or "", "<t:(%d+)")
                    if embedTs then initialTarget = tonumber(embedTs) break end
                    if emb.fields then
                        for _, fld in ipairs(emb.fields) do
                            local fieldTs = string.match(fld.value or "", "<t:(%d+)")
                            if fieldTs then initialTarget = tonumber(fieldTs) break end
                        end
                    end
                end
            end
        end
        
        if initialTarget and (initialTarget - os.time() <= 0) then
            joinBtn.Text = "❌ Expired"
            joinBtn.BackgroundColor3 = Color3.fromRGB(43, 45, 49)
            joinBtn.TextColor3 = Color3.fromRGB(148, 155, 164)
            if clientSettings.HideExpired then 
                frame.Visible = false 
                updateEmptyState() 
                task.defer(function()
                    if messageContainer then
                        messageContainer.CanvasPosition = Vector2.new(0, messageContainer.AbsoluteCanvasSize.Y)
                    end
                end)
            end
        else
            joinBtn.Text = "▶ Join Server"
            joinBtn.BackgroundColor3 = Color3.fromRGB(36, 128, 70)
            joinBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
        end
        
        local processing = false
        joinBtn.MouseButton1Click:Connect(function()
            if processing or joinBtn.Text == "❌ Expired" then return end
            processing = true
            joinBtn.Text = "⏳ Joining..."
            joinBtn.BackgroundColor3 = Color3.fromRGB(65, 67, 74)
            
            task.spawn(function()
                local targetPlace = tonumber(foundPlaceId)
                local success, err = pcall(function()
                    if foundJobId and string.len(foundJobId) > 5 then
                        TeleportService:TeleportToPlaceInstance(targetPlace, foundJobId, Players.LocalPlayer)
                    else
                        TeleportService:Teleport(targetPlace, Players.LocalPlayer)
                    end
                end)
                
                if not success then
                    warn("[Direct Connection Failure]: " .. tostring(err))
                    joinBtn.Text = "❌ Failed"
                    joinBtn.BackgroundColor3 = Color3.fromRGB(218, 55, 60)
                    task.wait(2)
                end
                
                processing = false
                if joinBtn.Text ~= "❌ Expired" then
                    joinBtn.Text = "▶ Join Server"
                    joinBtn.BackgroundColor3 = Color3.fromRGB(36, 128, 70)
                end
            end)
        end)
    end
end

toggleBtn.MouseButton1Click:Connect(function()
    setPanelVisibility(not container.Visible)
end)

-- ====================================================================
-- FETCH LOOP NETWORK ENGINE & AUTO-JOIN LOGIC
-- ====================================================================
fetchMessages = function(forceRefresh)
    local success, response = pcall(function()
        return request({
            Url = "https://discord.com/api/v9/channels/" .. channelID .. "/messages?limit=25",
            Method = "GET",
            Headers = {
                ["Authorization"] = token,
                ["Content-Type"] = "application/json",
            }
        })
    end)
    
    if success and response and response.StatusCode == 200 then
        local decodeSuccess, messages = pcall(function() return HttpService:JSONDecode(response.Body) end)
        if decodeSuccess and type(messages) == "table" and messages[1] then
            if forceRefresh or messages[1].id ~= lastMessageId then
                local isNewerMessage = (lastMessageId ~= "") and not forceRefresh
                lastMessageId = messages[1].id
                
                -- Clear old frames but NOT the NoPetsLabel
                for _, child in ipairs(messageContainer:GetChildren()) do
                    if child:IsA("Frame") then child:Destroy() end
                end
                
                for i = #messages, 1, -1 do
                    createMessageUI(messages[i])
                end
                
                updateEmptyState()
                
                task.defer(function()
                    messageContainer.CanvasPosition = Vector2.new(0, messageContainer.AbsoluteCanvasSize.Y)
                end)

                if isNewerMessage and not isInitialLoad then
                    showAchievementNotification(messages[1])
                    
                    if clientSettings.AutoJoin then
                        local pId, jId = extractIdsFromMessage(messages[1])
                        if pId and passesKeywordFilter(messages[1]) then
                            task.spawn(function()
                                pcall(function()
                                    if jId and string.len(jId) > 5 then
                                        TeleportService:TeleportToPlaceInstance(tonumber(pId), jId, Players.LocalPlayer)
                                    else
                                        TeleportService:Teleport(tonumber(pId), Players.LocalPlayer)
                                    end
                                end)
                            end)
                        end
                    end
                end
                isInitialLoad = false
            end
        end
    end
end

refreshBtn.MouseButton1Click:Connect(function()
    refreshBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    
    task.spawn(function()
        fetchMessages(true)
    end)
    
    task.wait(0.2)
    refreshBtn.TextColor3 = Color3.fromRGB(148, 155, 164)
end)

-- ====================================================================
-- LOGIN HANDLER 
-- ====================================================================
loginBtn.MouseButton1Click:Connect(function()
    local inputToken = string.gsub(tokenInput.Text, "^%s*(.-)%s*$", "%1") -- trim whitespace
    if inputToken ~= "" then
        token = inputToken
        loginFrame:Destroy() -- Removes login menu
        toggleBtn.Visible = true -- Shows standard pet menu toggle
        
        -- Start fetching loop only after login
        task.spawn(function()
            fetchMessages()
            while true do
                task.wait(REFRESH_RATE)
                fetchMessages()
            end
        end)
    else
        tokenInput.PlaceholderText = "Token required!"
        tokenInput.PlaceholderColor3 = Color3.fromRGB(218, 55, 60)
    end
end)
