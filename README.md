--=============================================================
-- Mammoth 上飞 + V技能 + 虚空M1 + 自动跳服(新加坡/2分钟)
-- 依赖: Blox Fruits (Third Sea)
-- 用法: 直接执行, 脚本会自行 装备猛犸 -> 上飞300 -> V -> 循环M1
--=============================================================

local Players           = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService        = game:GetService("RunService")
local HttpService       = game:GetService("HttpService")
local TeleportService   = game:GetService("TeleportService")
local VirtualInputManager = game:GetService("VirtualInputManager")

local LocalPlayer = Players.LocalPlayer
if not LocalPlayer then
    Players:GetPropertyChangedSignal("LocalPlayer"):Wait()
    LocalPlayer = Players.LocalPlayer
end

--=============================================================
-- 配置
--=============================================================
local CFG = {
    FruitName        = "Mammoth-Mammoth", -- 果实工具名
    FlySpeed         = 300,               -- 上升速度 (studs/秒)
    SkillKey         = "V",               -- 技能按键
    M1Interval       = 0.1,               -- M1 循环间隔
    ServerRegion     = "Singapore",       -- 默认服务器区域
    MinPlayers       = 1,
    MaxPlayers       = 12,
    MinBounty        = 0,
    MaxBounty        = 999999999,
    HopInterval      = 120,               -- 跳服间隔 (秒) = 2分钟
    ServerListTimeout= 20,

    AutoV3           = true,              -- 自动释放 V3 (种族技能第三阶)
    AutoV4           = true,              -- 自动释放 V4 (种族觉醒)
    V3Interval       = 2,                 -- V3 检测间隔 (秒)
    V4Interval       = 1,                 -- V4 检测间隔 (秒)

    Noclip           = true,              -- 穿墙

    -- 自动选海贼
    AutoTeam         = true,              -- 自动加入海贼阵营
    Team             = "Pirates",         -- "Pirates" 或 "Marines"

    -- 换服自动执行
    AutoExec         = true,              -- 跳服后自动重新执行
    AutoExecFile     = "mammoth_fly.lua", -- src 留空时, 从工作目录读这个文件当源码
}

--=============================================================
-- 换服自动执行: 源码
--=============================================================
-- 【填这里】把本脚本源码整段粘到 [[ ]] 之间, 跳服时会被写入工作目录
-- 并交给执行器的跳服执行函数(queue_on_teleport), 新服加载后自动重跑。
-- 留空则自动从工作目录读取本脚本文件(CFG.AutoExecFile)当源码。
local src = [[

]]
local SRC_TERMINATOR = "--===SRC-END==="


local VOID_VECTOR = Vector3.new(9e30, 9e30, 9e30) -- 虚空M1参数

--=============================================================
-- 工具函数
--=============================================================
local function Log(fmt, ...)
    local ok, msg = pcall(string.format, fmt, ...)
    print("[Mammoth] " .. (ok and msg or tostring(fmt)))
end

local function GetChar()
    local char = LocalPlayer.Character
    if char and char.Parent then return char end
    return nil
end

local function GetHRP()
    local char = GetChar()
    return char and char:FindFirstChild("HumanoidRootPart")
end

local function GetHum()
    local char = GetChar()
    return char and char:FindFirstChildOfClass("Humanoid")
end

-- 安全 WaitForChild (带超时, 不报错)
local function SafeWait(parent, name, timeout)
    if not parent then return nil end
    local child = parent:FindFirstChild(name)
    if child then return child end
    local ok, result = pcall(function()
        return parent:WaitForChild(name, timeout or 5)
    end)
    return ok and result or nil
end

-- Pcall 包装的远程调用 (带超时保护, 防止 InvokeServer 挂死)
local function InvokeRemote(remote, timeout, ...)
    if not remote then return false end
    local args = table.pack(...)
    local done, result = false, nil
    task.spawn(function()
        local ok, res = pcall(function()
            return remote:InvokeServer(table.unpack(args, 1, args.n))
        end)
        done, result = true, ok and res or nil
    end)
    local deadline = os.clock() + (timeout or 5)
    while not done and os.clock() < deadline do
        task.wait(0.05)
    end
    return done, result
end

--=============================================================
-- 装备猛犸果实
--=============================================================
local function FindFruit()
    local name = CFG.FruitName
    local char = GetChar()
    if char then
        local tool = char:FindFirstChild(name)
        if tool and tool:IsA("Tool") then return tool end
    end
    local backpack = LocalPlayer:FindFirstChild("Backpack")
    if backpack then
        local tool = backpack:FindFirstChild(name)
        if tool and tool:IsA("Tool") then return tool end
    end
    -- 兜底: 按 ToolTip 找恶魔果实
    for _, container in ipairs({ char, backpack }) do
        if container then
            for _, child in ipairs(container:GetChildren()) do
                if child:IsA("Tool") and child.ToolTip == "Blox Fruit" then
                    return child
                end
            end
        end
    end
    return nil
end

local function EquipMammoth()
    local hum = GetHum()
    if not hum then return false end

    -- 已经装备
    local char = GetChar()
    local equipped = char and char:FindFirstChildOfClass("Tool")
    if equipped and equipped.Name == CFG.FruitName then
        return true
    end

    -- 方式1: Humanoid:EquipTool
    local fruit = FindFruit()
    if fruit and fruit.Parent ~= char then
        pcall(function() hum:EquipTool(fruit) end)
        task.wait(0.3)
    end

    char = GetChar()
    equipped = char and char:FindFirstChildOfClass("Tool")
    if equipped and equipped.Name == CFG.FruitName then
        return true
    end

    -- 方式2: 按快捷键 (数字键对应工具栏槽位)
    local hotbar = LocalPlayer:FindFirstChild("PlayerGui")
    hotbar = hotbar and hotbar:FindFirstChild("Hotbar")
    hotbar = hotbar and hotbar:FindFirstChild("Hotbar")
    if hotbar then
        for _, slot in ipairs(hotbar:GetChildren()) do
            local key = slot:FindFirstChild("KeyLabel")
            local toolName = slot:GetAttribute("ItemName") or (slot:FindFirstChild("ToolName") and slot.ToolName.Text)
            if (toolName and tostring(toolName):find("Mammoth", 1, true)) and key then
                local code = Enum.KeyCode[key.Text]
                if code then
                    pcall(function()
                        VirtualInputManager:SendKeyEvent(true, code, false, game)
                        task.wait(0.1)
                        VirtualInputManager:SendKeyEvent(false, code, false, game)
                    end)
                    task.wait(0.3)
                end
            end
        end
    end

    char = GetChar()
    equipped = char and char:FindFirstChildOfClass("Tool")
    return equipped ~= nil and equipped.Name == CFG.FruitName
end

-- 等待 Mammoth 工具完全加载 (含 M1 远程)
local function WaitForMammoth(timeout)
    local deadline = os.clock() + (timeout or 15)
    while os.clock() < deadline do
        local char = GetChar()
        local mammoth = char and char:FindFirstChild(CFG.FruitName)
        if mammoth and mammoth:FindFirstChild("LeftClickRemote") then
            return mammoth
        end
        if not mammoth then
            pcall(EquipMammoth)
        end
        task.wait(0.2)
    end
    return nil
end

--=============================================================
-- 上飞 300 studs/s
--=============================================================
local FlyActive = false
local FlyConn = nil
local FlyObjects = {}

local function MakeFlyObjects(hrp)
    local objects = { Parts = {} }

    local att = Instance.new("Attachment")
    att.Name = "MammothFlyAtt"
    att.Parent = hrp
    objects.Attachment = att

    local bv = Instance.new("BodyVelocity")
    bv.Name = "MammothFlyVelocity"
    bv.MaxForce = Vector3.new(9e9, 9e9, 9e9)
    bv.P = 1250
    bv.Velocity = Vector3.new(0, CFG.FlySpeed, 0)
    bv.Parent = hrp
    objects.Velocity = bv

    local bg = Instance.new("BodyGyro")
    bg.Name = "MammothFlyGyro"
    bg.MaxTorque = Vector3.new(9e9, 9e9, 9e9)
    bg.P = 3000
    bg.D = 500
    bg.CFrame = hrp.CFrame
    bg.Parent = hrp
    objects.Gyro = bg

    return objects
end

local function SetPartsCollide(char, collide)
    for _, part in ipairs(char:GetDescendants()) do
        if part:IsA("BasePart") then
            pcall(function() part.CanCollide = collide end)
        end
    end
end

local function StopFly()
    FlyActive = false

    if FlyConn then
        pcall(function() FlyConn:Disconnect() end)
        FlyConn = nil
    end

    if FlyObjects.Velocity then pcall(function() FlyObjects.Velocity:Destroy() end) end
    if FlyObjects.Gyro then pcall(function() FlyObjects.Gyro:Destroy() end) end
    if FlyObjects.Attachment then pcall(function() FlyObjects.Attachment:Destroy() end) end
    FlyObjects = {}

    local hum = GetHum()
    local hrp = GetHRP()
    if hum then
        pcall(function()
            hum.PlatformStand = false
            hum:ChangeState(Enum.HumanoidStateType.GettingUp)
        end)
    end
    if hrp then
        pcall(function()
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
        end)
    end
    local char = GetChar()
    if char then SetPartsCollide(char, true) end
end

local function StartFly()
    StopFly()

    local hrp = GetHRP()
    local hum = GetHum()
    if not hrp or not hum then
        Log("StartFly 失败: 角色未加载")
        return false
    end

    -- 取得网络所有权, 否则速度会被服务器回滚
    pcall(function() hrp:SetNetworkOwner(LocalPlayer) end)

    hum.PlatformStand = true
    pcall(function() hum:ChangeState(Enum.HumanoidStateType.Physics) end)
    SetPartsCollide(GetChar(), false)

    FlyObjects = MakeFlyObjects(hrp)
    FlyActive = true

    FlyConn = RunService.Heartbeat:Connect(function()
        if not FlyActive then return end

        local root = GetHRP()
        local health = GetHum()
        if not root or not health or health.Health <= 0 then
            StopFly()
            return
        end

        -- 持续压速度: 每秒 CFG.FlySpeed studs 垂直上升
        local bv = FlyObjects.Velocity
        if bv and bv.Parent then
            bv.Velocity = Vector3.new(0, CFG.FlySpeed, 0)
        end
        local bg = FlyObjects.Gyro
        if bg and bg.Parent then
            bg.CFrame = CFrame.new(root.Position)
        end
    end)

    Log("上升已开启: %d studs/秒", CFG.FlySpeed)
    return true
end

--=============================================================
-- 穿墙 (Noclip)
--=============================================================
-- 持续把角色所有 BasePart 的 CanCollide 关掉, 并用弱键表记住改过的部件
-- 以便关闭时精确还原 (不会把本来就不碰撞的部件误设成碰撞)
local NoclipActive = false
local NoclipConn = nil
local NoclipChanged = setmetatable({}, { __mode = "k" })
local NoclipIndex = 0
local NoclipLastScan = 0

local function ApplyNoclipPass()
    local char = GetChar()
    if not char then return end

    local parts = char:GetDescendants()
    local count = #parts
    if count == 0 then return end

    -- 每次只处理一个切片, 大角色模型下避免单帧卡顿
    local perPass = 40
    if count <= perPass then
        NoclipIndex = 0
    else
        NoclipIndex = NoclipIndex % count
    end

    local processed = 0
    local i = NoclipIndex
    while processed < perPass and processed < count do
        i = (i % count) + 1
        local part = parts[i]
        if part and part:IsA("BasePart") and part.CanCollide then
            NoclipChanged[part] = true
            part.CanCollide = false
        end
        processed = processed + 1
    end
    NoclipIndex = i
end

local function StopNoclip()
    NoclipActive = false

    if NoclipConn then
        pcall(function() NoclipConn:Disconnect() end)
        NoclipConn = nil
    end

    -- 还原被我们改过的部件
    for part in pairs(NoclipChanged) do
        if typeof(part) == "Instance" and part.Parent then
            pcall(function() part.CanCollide = true end)
        end
    end
    NoclipChanged = setmetatable({}, { __mode = "k" })
    NoclipIndex = 0
end

local function StartNoclip()
    if NoclipActive then return end
    NoclipActive = true

    NoclipConn = RunService.Stepped:Connect(function()
        if not NoclipActive then return end

        local char = GetChar()
        if not char then return end

        ApplyNoclipPass()

        -- 保险: 万一穿到地图底下, 拉回上方, 免得掉进虚空
        local root = GetHRP()
        if root and root.Position.Y < -500 then
            pcall(function()
                root.CFrame = CFrame.new(root.Position.X, 200, root.Position.Z)
            end)
            Log("检测到掉出地图, 已拉回上方")
        end
    end)

    Log("穿墙已开启")
end

--=============================================================
-- 释放一次 V 技能
--=============================================================
-- 技能板路径: PlayerGui.Main.Skills.<工具名>.<按键>
local function GetSkillBoard(toolName)
    local pg = LocalPlayer:FindFirstChild("PlayerGui")
    local main = pg and pg:FindFirstChild("Main")
    local skills = main and main:FindFirstChild("Skills")
    return skills and skills:FindFirstChild(toolName)
end

-- 方式1: 直接触发技能板的 Mobile 按钮连接 (无需真实输入)
local function FireSkillByConnections(skillKey)
    if not getconnections then return false end

    local char = GetChar()
    local tool = char and char:FindFirstChildOfClass("Tool")
    if not tool then return false end

    local board = GetSkillBoard(tool.Name)
    local button = board and board:FindFirstChild(skillKey)
    local mobile = button and button:FindFirstChild("Mobile")
    if not mobile then return false end

    local conns = getconnections(mobile.MouseButton1Down)
    if not conns or #conns == 0 then return false end

    for _, conn in ipairs(conns) do
        pcall(function() conn:Fire() end)
    end
    task.wait(0.15)
    if getconnections(mobile.MouseButton1Up) then
        for _, conn in ipairs(getconnections(mobile.MouseButton1Up)) do
            pcall(function() conn:Fire() end)
        end
    end
    return true
end

-- 方式2: 模拟键盘按键
local function FireSkillByKey(skillKey)
    local code = Enum.KeyCode[skillKey:upper()]
    if not code then return false end
    local ok = pcall(function()
        VirtualInputManager:SendKeyEvent(true, code, false, game)
        task.wait(0.3)
        VirtualInputManager:SendKeyEvent(false, code, false, game)
    end)
    task.wait(0.2)
    return ok
end

local function CastV()
    -- 确保猛犸在身上, 技能板才存在
    if not EquipMammoth() then
        Log("V技能: 装备猛犸失败, 仍尝试释放")
    end
    WaitForMammoth(8)

    local char = GetChar()
    local tool = char and char:FindFirstChildOfClass("Tool")
    local board = tool and GetSkillBoard(tool.Name)

    local ok = false
    if board then
        pcall(function() ok = FireSkillByConnections(CFG.SkillKey) end)
    end

    -- 兜底1: 模拟按键
    if not ok then
        pcall(function() ok = FireSkillByKey(CFG.SkillKey) end)
    end

    -- 兜底2: 键盘方式触发后, 技能板可能才刷出来, 再试一次连接
    if not ok then
        task.wait(0.3)
        pcall(function() ok = FireSkillByConnections(CFG.SkillKey) end)
    end

    Log(ok and "V技能已释放" or "V技能释放失败(技能板未就绪)")
    return ok
end

--=============================================================
-- 虚空 M1 循环
--=============================================================
local M1Running = false
local M1Thread = nil

local function FireM1()
    local char = GetChar()
    local mammoth = char and char:FindFirstChild(CFG.FruitName)
    local remote = mammoth and mammoth:FindFirstChild("LeftClickRemote")
    if not remote then return false end
    return pcall(function()
        remote:FireServer(VOID_VECTOR)
    end)
end

local function StartM1Loop()
    if M1Running then return end
    M1Running = true

    M1Thread = task.spawn(function()
        Log("虚空M1循环已开启")
        while M1Running do
            task.wait(CFG.M1Interval)

            local char = GetChar()
            if not char then
                task.wait(0.5)
                continue
            end

            -- 果实掉了就重新装备
            local mammoth = char:FindFirstChild(CFG.FruitName)
            if not (mammoth and mammoth:FindFirstChild("LeftClickRemote")) then
                pcall(EquipMammoth)
                task.wait(0.3)
                continue
            end

            FireM1()
        end
    end)
end

local function StopM1Loop()
    M1Running = false
    M1Thread = nil
end

--=============================================================
-- 自动选海贼
--=============================================================
local function JoinTeam()
    local target = (CFG.Team == "Marines") and "Marines" or "Pirates"

    -- 已经在目标阵营就不用动
    local ok, myTeam = pcall(function() return LocalPlayer.Team end)
    if ok and myTeam and myTeam.Name == target then
        return true
    end

    -- 方式1: CommF_ 直接设置阵营
    local remotes = ReplicatedStorage:FindFirstChild("Remotes")
    local commF = remotes and remotes:FindFirstChild("CommF_")
    if commF then
        local done = InvokeRemote(commF, 5, "SetTeam", target)
        if done then
            task.wait(0.5)
            local ok2, team2 = pcall(function() return LocalPlayer.Team end)
            if ok2 and team2 and team2.Name == target then
                return true
            end
        end
    end

    -- 方式2: 点选阵营界面 (还没选阵营时会出现)
    local pg = LocalPlayer:FindFirstChild("PlayerGui")
    for _, guiName in ipairs({ "Main (minimal)", "Main" }) do
        local gui = pg and pg:FindFirstChild(guiName)
        local choose = gui and gui:FindFirstChild("ChooseTeam")
        local container = choose and choose:FindFirstChild("Container")
        local side = container and container:FindFirstChild(target)
        local frame = side and side:FindFirstChild("Frame")
        local button = frame and frame:FindFirstChild("TextButton")
        if button then
            local fired = false
            if getconnections then
                local conns = getconnections(button.MouseButton1Down)
                if conns and #conns > 0 then
                    for _, conn in ipairs(conns) do
                        pcall(function() conn:Fire() end)
                    end
                    fired = true
                end
            end
            if not fired and firesignal then
                pcall(function() firesignal(button.MouseButton1Down) end)
                fired = true
            end
            if fired then
                task.wait(0.8)
                return true
            end
        end
    end

    return false
end

--=============================================================
-- 换服自动执行
--=============================================================
local QUEUE_GUARD_KEY = "MAMMOTH_FLY_AUTOEXEC_JOB"
local AutoExecArmed = false
local AutoExecWarned = false
local AutoExecQueuedFor = nil

local function GetQueueOnTeleport()
    local env = getgenv and getgenv() or _G
    local fn = env and rawget(env, "queue_on_teleport")
    if type(fn) ~= "function" then
        fn = env and rawget(env, "queueonteleport")
    end
    if type(fn) ~= "function" and syn then fn = syn.queue_on_teleport end
    if type(fn) ~= "function" and fluxus then fn = fluxus.queue_on_teleport end
    return type(fn) == "function" and fn or nil
end

-- 取得要注入的源码: 优先用顶部 src, 留空则读本地文件
local function GetHopSource()
    if src and src:find("%S") then
        return src
    end

    local fname = CFG.AutoExecFile
    if isfile and readfile then
        local ok, data = pcall(function()
            if isfile(fname) then return readfile(fname) end
            return nil
        end)
        if ok and type(data) == "string" and data:find("%S") then
            return data
        end
    end
    return nil
end

-- 防重复: 同一个 JobId 只自动执行一次
local function BuildPayload(code)
    return table.concat({
        "-- Mammoth auto-exec payload",
        "task.spawn(function()",
        "    repeat task.wait(1) until game:IsLoaded()",
        "    task.wait(2)",
        string.format("    if getgenv().%s == game.JobId then return end", QUEUE_GUARD_KEY),
        string.format("    getgenv().%s = game.JobId", QUEUE_GUARD_KEY),
        "    local src = [==[",
        code,
        "]==]",
        "    local ok, err = pcall(function() loadstring(src)() end)",
        '    if not ok then warn("[Mammoth auto-exec] " .. tostring(err)) end',
        "end)",
        SRC_TERMINATOR,
    }, "\n")
end

local function SetupAutoExec()
    if not CFG.AutoExec then return false end
    if AutoExecQueuedFor == game.JobId then return true end

    local queueFn = GetQueueOnTeleport()
    if not queueFn then
        if not AutoExecWarned then
            AutoExecWarned = true
            Log("换服自动执行: 执行器不支持 queue_on_teleport, 已跳过")
        end
        return false
    end

    local code = GetHopSource()
    if not code then
        if not AutoExecWarned then
            AutoExecWarned = true
            Log("换服自动执行: src 为空且读不到 %s, 已跳过", tostring(CFG.AutoExecFile))
        end
        return false
    end

    -- 写入工作目录 (执行器本地磁盘, 不是游戏的 workspace 服务)
    if writefile then
        pcall(function()
            if makefolder and isfolder and not isfolder("mammoth") then
                makefolder("mammoth")
            end
            writefile("mammoth/src.lua", code)
        end)
    end

    local ok = pcall(queueFn, BuildPayload(code))
    if ok then
        AutoExecArmed = true
        AutoExecQueuedFor = game.JobId
        Log("换服自动执行已排队 (%d 字节源码)", #code)
    else
        Log("换服自动执行排队失败")
    end
    return ok
end

--=============================================================
-- 自动 V3 / V4 (种族技能)
--=============================================================
-- V3: CommE:FireServer("ActivateAbility")
local function TurnOnV3()
    local char = GetChar()
    if not char then return false end

    -- 已经变身就别重复发
    local transformed = char:FindFirstChild("RaceTransformed")
    if transformed and transformed.Value then return false end

    local remotes = ReplicatedStorage:FindFirstChild("Remotes")
    local commE = remotes and remotes:FindFirstChild("CommE")
    if not commE then return false end

    return pcall(function() commE:FireServer("ActivateAbility") end)
end

-- V4: Awakening.RemoteFunction:InvokeServer(true)
local function TurnOnV4()
    local char = GetChar()
    if not char then return false end

    local energy      = char:FindFirstChild("RaceEnergy")
    local transformed = char:FindFirstChild("RaceTransformed")
    -- 能量不足 / 已在觉醒状态 都跳过
    if not energy or tonumber(energy.Value) == nil or energy.Value < 1 then return false end
    if not transformed or transformed.Value then return false end

    local awakening = char:FindFirstChild("Awakening")
    if not awakening then
        local backpack = LocalPlayer:FindFirstChild("Backpack")
        awakening = backpack and backpack:FindFirstChild("Awakening")
    end
    if not awakening then return false end

    local remote = awakening:FindFirstChild("RemoteFunction")
    if not remote then return false end

    local ok = pcall(function() remote:InvokeServer(true) end)
    if not ok then
        -- 有些版本要传 table
        ok = pcall(function() remote:InvokeServer({ true }) end)
    end
    return ok
end

local AbilityLoopStarted = false

local function StartAbilityLoop()
    if AbilityLoopStarted then return end
    AbilityLoopStarted = true

    task.spawn(function()
        Log("自动V3已开启: 每 %s 秒检测", tostring(CFG.V3Interval))
        while true do
            task.wait(CFG.V3Interval)
            if CFG.AutoV3 then
                pcall(TurnOnV3)
            end
        end
    end)

    task.spawn(function()
        Log("自动V4已开启: 每 %s 秒检测", tostring(CFG.V4Interval))
        while true do
            task.wait(CFG.V4Interval)
            if CFG.AutoV4 then
                pcall(TurnOnV4)
            end
        end
    end)
end

--=============================================================
-- 跳服逻辑 (取自 RJR付费.lua, 默认新加坡)
--=============================================================
local ServerBrowser = SafeWait(ReplicatedStorage, "__ServerBrowser", 20)
local ServerHopCache = { List = nil, Fetching = false }
local Hopping = false

local function FetchServerList(timeout)
    local browser = ReplicatedStorage:FindFirstChild("__ServerBrowser")
    if not browser then return nil end
    local servers = {}
    local pending = 0
    for page = 1, 100 do
        pending = pending + 1
        task.delay(page * 0.02, function()
            local ok, result = pcall(function()
                return browser:InvokeServer(page)
            end)
            if not ok then
                local tries = 0
                repeat
                    task.wait(0.5)
                    tries = tries + 1
                    ok, result = pcall(function()
                        return browser:InvokeServer(page)
                    end)
                until ok or tries >= 2
            end
            if ok and type(result) == "table" then
                for job, info in pairs(result) do
                    if type(info) == "table" then
                        servers[job] = {
                            Region = tostring(info.Region or "Unknown"),
                            Count  = tonumber(info.Count) or 0,
                            Bounty = tonumber(info.Bounty) or 0,
                        }
                    end
                end
            end
            pending = pending - 1
        end)
    end
    local deadline = os.clock() + (timeout or 20)
    while pending > 0 and os.clock() < deadline do
        task.wait(0.1)
    end
    return servers
end

local function GetServerList()
    if ServerHopCache.List then
        return ServerHopCache.List
    end
    if ServerHopCache.Fetching then
        local deadline = os.clock() + 25
        while ServerHopCache.Fetching and os.clock() < deadline do
            task.wait(0.1)
        end
        return ServerHopCache.List
    end
    ServerHopCache.Fetching = true
    local ok, servers = pcall(FetchServerList, CFG.ServerListTimeout)
    ServerHopCache.Fetching = false
    if ok and type(servers) == "table" and next(servers) ~= nil then
        ServerHopCache.List = servers
    end
    return ServerHopCache.List
end

local function MatchServerInfo(info)
    if not info then return false end
    local regionFilter = tostring(CFG.ServerRegion or ""):lower()
    if regionFilter ~= "" then
        local region = tostring(info.Region or ""):lower()
        if not region:find(regionFilter, 1, true) then return false end
    end
    local count = tonumber(info.Count) or 0
    if count < CFG.MinPlayers or count > CFG.MaxPlayers then return false end
    local bounty = tonumber(info.Bounty) or 0
    if bounty < CFG.MinBounty or bounty > CFG.MaxBounty then return false end
    return true
end

local function PickServer(servers)
    if type(servers) ~= "table" then return nil end
    local currentJob = game.JobId
    local matches = {}
    for job, info in pairs(servers) do
        if job ~= currentJob and MatchServerInfo(info) then
            table.insert(matches, {
                Job    = job,
                Region = info.Region,
                Count  = info.Count,
                Bounty = info.Bounty,
            })
        end
    end
    if #matches == 0 then return nil end
    return matches[math.random(1, #matches)]
end

-- 跳转到指定 jobId (主方式: __ServerBrowser, 兜底: TeleportService)
local function TeleportToJob(jobId)
    local browser = ReplicatedStorage:FindFirstChild("__ServerBrowser")
    if browser then
        local done, result = InvokeRemote(browser, 8, "teleport", jobId)
        if done then return true, "browser" end
    end
    if TeleportService and game.PlaceId and game.PlaceId > 0 then
        local ok, err = pcall(function()
            TeleportService:TeleportToPlaceInstance(game.PlaceId, jobId, LocalPlayer)
        end)
        if ok then return true, "teleportservice" end
        return false, tostring(err)
    end
    return false, "no method"
end

local function DoServerHop()
    if Hopping then return false end
    Hopping = true

    Log("开始跳服: 区域=%s", tostring(CFG.ServerRegion))

    local servers = GetServerList()
    local total = 0
    if type(servers) == "table" then
        for _ in pairs(servers) do total = total + 1 end
    end

    local target = PickServer(servers)

    if not target then
        Log("未找到匹配的服务器 (共 %d 个)", total)
        Hopping = false
        -- 列表为空时清缓存, 下次重新拉取
        if total == 0 then
            ServerHopCache.List = nil
        end
        return false
    end

    Log("匹配到服务器: 区域=%s 人数=%s 赏金=%s", tostring(target.Region), tostring(target.Count), tostring(target.Bounty))

    -- 跳服前先收手, 避免在新服残留状态
    StopM1Loop()
    StopNoclip()
    StopFly()

    local ok, method = TeleportToJob(target.Job)
    if not ok then
        Log("跳服失败: %s", tostring(method))
        Hopping = false
        return false
    end

    Log("已发送跳服请求 (%s)", tostring(method))
    task.wait(0.5)

    -- 补排一次, 保证新服里也备着 payload
    pcall(SetupAutoExec)

    Hopping = false
    return true
end

--=============================================================
-- 主体流程: 装备 -> 上飞 -> V -> M1 循环 -> 2分钟跳服
--=============================================================
local Initialized = false
local HopLoopStarted = false

local function RunOnce()
    Log("等待角色加载...")
    local char = GetChar()
    if not char then
        local ok = pcall(function()
            char = LocalPlayer.CharacterAdded:Wait()
        end)
        if not ok or not char then return end
    end
    task.wait(1)

    -- 0) 自动加入海贼阵营
    if CFG.AutoTeam then
        Log("正在加入阵营: %s ...", tostring(CFG.Team))
        local joined = false
        for _ = 1, 10 do
            if JoinTeam() then joined = true break end
            task.wait(1)
        end
        Log(joined and ("阵营已确认: " .. tostring(CFG.Team)) or "阵营设置失败, 继续执行")
    end

    -- 1) 装备猛犸
    Log("正在装备 %s ...", CFG.FruitName)
    local mammoth = nil
    for _ = 1, 15 do
        pcall(EquipMammoth)
        mammoth = WaitForMammoth(2)
        if mammoth then break end
    end
    if mammoth then
        Log("已装备 %s", CFG.FruitName)
    else
        Log("警告: 未检测到 %s, 仍继续执行", CFG.FruitName)
    end

    -- 2) 以 300 studs/s 往上飞
    StartFly()

    -- 3) 穿墙
    if CFG.Noclip then
        StartNoclip()
    end

    -- 4) 执行一次 V 技能
    task.wait(0.6)
    CastV()

    -- 5) 循环虚空 M1
    StartM1Loop()

    -- 6) 自动 V3 / V4 (全局只保留一套循环, 换服后自动跟随新角色)
    StartAbilityLoop()

    -- 7) 每 2 分钟跳服一次 (默认新加坡), 全局只保留一个计时器
    if not HopLoopStarted then
        HopLoopStarted = true
        task.spawn(function()
            while true do
                task.wait(CFG.HopInterval)
                DoServerHop()
            end
        end)
        Log("自动跳服已开启: 每 %d 秒一次 (%s)", CFG.HopInterval, CFG.ServerRegion)
    end
end

local function Init()
    if Initialized then return end
    Initialized = true

    -- 同一服务器内只跑一次 (自动执行可能把脚本重灌进来)
    local env = getgenv and getgenv() or _G
    if env then
        if env.__MAMMOTH_FLY_RUNNING == game.JobId then
            Log("本服务器已初始化过, 跳过重复执行")
            return
        end
        env.__MAMMOTH_FLY_RUNNING = game.JobId
    end

    -- 加载即排队, 这样手动换服/游戏内换服也能自动重跑
    SetupAutoExec()

    RunOnce()
end

-- 角色重生 (含跳服换服) 后重新走一遍流程
LocalPlayer.CharacterAdded:Connect(function()
    if not Initialized then return end
    StopNoclip()
    StopFly()
    StopM1Loop()
    task.wait(2)
    task.spawn(RunOnce)
end)

Init()
