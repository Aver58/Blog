---
layout:     post
title:      "Lua运行时更新代码"
subtitle:   "开发神器"
date:       2019-07-10 16:00:00
author:     "Zero"
header-img: "img/post-bg-2015.jpg"
tags:
    - Coder's Life
---

      最近沉迷lua脚本热更，想说这个可以提高多少菜鸡的调试效率，找了网上好多文章，但是都不行，尝试了很久，并且自己测试和学习，写了一遍，勉强能热更了（还是有个问题，要按2次才能输出正确热更的值，一直找不到，有人找到麻烦告诉我一下308164213）。下面记录一下找热更的过程。

一、用来卸载表格的加载
--最简单粗暴的热更新就是将package.loaded[modelname]的值置为nil，强制重新加载：
function reload_module_obsolete(module_name)
    package.loaded[module_name] = nil
    require(module_name)
end
--这样做虽然能完成热更，但问题是已经引用了该模块的地方不会得到更新， 因此我们需要将引用该模块的地方的值也做对应的更新。
function ReloadUtil.Reload_Module(module_name)
    local old_module = _G[module_name]

    package.loaded[module_name] = nil
    require (module_name)

    local new_module = _G[module_name]
    for k, v in pairs(new_module) do
        old_module[k] = v
    end

    package.loaded[module_name] = old_module
end

二、我认为逻辑最清晰的，但是可能是我不会用吧！
源链接：https://github.com/tickbh/td_rlua/wiki/Lua-%E7%83%AD%E6%9B%B4%E6%96%B0%E5%B0%8F%E7%BB%93

--region 利用_ENV环境，在加载的时候把数据加载到_ENV下，然后再通过对比的方式修改_G底下的值，从而实现热更新，函数 -- 失败

function ReloadUtil.hotfix(chunk, check_name)
    check_name = check_name or 'hotfix'
    local env = {}
    setmetatable(env, { __index = _G })
    local f, err = load(chunk, check_name,  't', env)
    assert(f,err)
    local ok, err = pcall(f)
    assert(ok,err)

    local protection = {
        setmetatable = true,
        pairs = true,
        ipairs = true,
        next = true,
        require = true,
        _ENV = true,
    }
    --防止重复的table替换，造成死循环
    local visited_sig = {}

    function ReloadUtil.update_table(newTable, oldTable, name, deep)
        --对某些关键函数不进行比对
        if protection[newTable] or protection[oldTable] then return end
        --如果原值与当前值内存一致，值一样不进行对比
        if newTable == oldTable then return end
        local signature = tostring(oldTable)..tostring(newTable)
        if visited_sig[signature] then return end
        visited_sig[signature] = true
        --遍历对比值，如进行遍历env类似的步骤
        for name, newValue in pairs(newTable) do
            local old_value = oldTable[name]
            if type(newValue) == type(old_value) then
                if type(newValue) == 'function' then
                    ReloadUtil.update_func(newValue, old_value, name, deep..'  '..name..'  ')
                    oldTable[name] = newValue
                elseif type(newValue) == 'table' then
                    ReloadUtil.update_table(newValue, old_value, name, deep..'  '..name..'  ')
                end
            else
                oldTable[name] = newValue
            end
        end
        --遍历table的元表，进行对比
        local old_meta = debug.getmetatable(oldTable)
        local new_meta = debug.getmetatable(newTable)
        if type(old_meta) == 'table' and type(new_meta) == 'table' then
            ReloadUtil.update_table(new_meta, old_meta, name..'s Meta', deep..'  '..name..'s Meta'..'  ' )
        end
    end

    function ReloadUtil.update_func(newFunc, oldFunc, name, deep)
        --取得原值所有的upvalue，保存起来
        local old_upvalue_map = {}
        for i = 1, math.huge do
            local name, value = debug.getupvalue(oldFunc, i)
            if not name then break end
            old_upvalue_map[name] = value
        end
        --遍历所有新的upvalue，根据名字和原值对比，如果原值不存在则进行跳过，如果为其它值则进行遍历env类似的步骤
        for i = 1, math.huge do
            local name, value = debug.getupvalue(newFunc, i)
            if not name then break end
            local old_value = old_upvalue_map[name]
            if old_value then
                if type(old_value) ~= type(value) then
                    debug.setupvalue(newFunc, i, old_value)
                elseif type(old_value) == 'function' then
                    ReloadUtil.update_func(value, old_value, name, deep..'  '..name..'  ')
                elseif type(old_value) == 'table' then
                    ReloadUtil.update_table(value, old_value, name, deep..'  '..name..'  ')
                    debug.setupvalue(newFunc, i, old_value)
                else
                    debug.setupvalue(newFunc, i, old_value)
                end
            end
        end
    end

    --原理
    --利用_ENV环境，在加载的时候把数据加载到_ENV下，然后再通过对比的方式修改_G底下的值，从而实现热更新，函数
    for name,value in pairs(env) do
        local g_value = _G[name]
        if type(g_value) ~= type(value) then
            _G[name] = value
        elseif type(value) == 'function' then
            ReloadUtil.update_func(value, g_value, name, 'G'..'  ')
            _G[name] = value
        elseif type(value) == 'table' then
            ReloadUtil.update_table(value, g_value, name, 'G'..'  ')
        end
    end
    return 0
end

function ReloadUtil.hotfix_file(debugName)
    local newCode
    local fp = io.open(debugName)
    if fp then
        newCode = fp:read('*all')
        io.close(fp)
    end
    if not newCode then
        return -1
    end
    return ReloadUtil.hotfix(newCode, debugName)
end

--endregion

三、有点复杂，递归了debug.getregistry()，然后去替换旧的值
源链接：http://asqbtcupid.github.io/luahotupdate1-require/

有介绍原理，讲得还蛮细的，学习了蛮多，但是这个递归真的是复杂，我注释掉了一些递归，能满足基本的需求。

local ReloadUtil = {}

local tableInsert = table.insert
local tableRemove = table.remove
local tableConcat = table.concat
local ioPopen = io.popen
local ioInput = io.input
local ioRead = io.read
local stringMatch = string.match
local stringFind = string.find
local stringSub = string.sub
local stringGsub = string.gsub
local packageLoaded = package.loaded
local type = type
local getfenv = getfenv
local setfenv = setfenv
local loadstring = loadstring
local mathHuge = math.huge
local debugGetupvalue = debug.getupvalue
local debugSetupvalue = debug.setupvalue
local debugGetmetatable = debug.getmetatable
local debugSetfenv = debug.setfenv

function ReloadUtil.FailNotify(...)
    printAError(...)
end

function ReloadUtil.DebugNofity(...)
    print(...)
end

function ReloadUtil.ErrorHandle(e)
    ReloadUtil.FailNotify("HotUpdate Error\n"..tostring(e))
    ReloadUtil.ErrorHappen = true
end

function ReloadUtil.InitProtection()
    ReloadUtil.Protection = {}
    local Protection = ReloadUtil.Protection
    Protection[setmetatable] = true
    Protection[pairs] = true
    Protection[ipairs] = true
    Protection[next] = true
    Protection[require] = true
    Protection[ReloadUtil] = true
    Protection[ReloadUtil.Meta] = true
    Protection[math] = true
    Protection[string] = true
    Protection[table] = true
end

local function Normalize(path)
    path = path:gsub("/","\\")

    local pathLen = #path
    if path:sub(pathLen, pathLen) == "\\" then
        path = path:sub(1, pathLen - 1)
    end

    local parts = { }
    for w in path:gmatch("[^\\]+") do
        if     w == ".." and #parts ~=0 then tableRemove(parts)
        elseif w ~= "."  then tableInsert(parts, w)
        end
    end
    return tableConcat(parts, "\\")
end

-- 根据给的路径，找到路径下所有文件，HU.FileMap[FileName] = {SysPath = line, LuaPath = luapath}
function ReloadUtil.InitFileMap(RootPath)
    local systemPathList = {}
    local HotUpdateDic = ReloadUtil.HotUpdateDic
    local OldCode = ReloadUtil.OldCode
    RootPath = Normalize(RootPath)
    --获取一个File对象其下的所有文件和目录的绝对路径: 的所有文件(/S/B)，不包括文件夹（/A:A），ioPopen返回文件句柄file handle
    --todo 这里有的问题，多次启动后会报bad file decorator ，检查是否是打开文件没有关闭导致
    local file = ioPopen("dir /S/B /A:A \""..RootPath.."\"")

    for SysPath in ioInput(file):lines() do
        local FileName = stringMatch(SysPath,".*\\(.*)%.lua")
        --todo meta 优化一下regex
        local metaFile = stringMatch(SysPath,".*\\(.*)%.lua.meta")
        if FileName ~= nil and metaFile == nil then
            --todo !!! luaPath在保存的时候是按文件夹路径保存的比如：game.modules.XXX.lua，所以可能要自己搞对这个路径

            local startIndex = stringFind(SysPath,"game")
            local luaPath = stringSub(SysPath, startIndex, #SysPath -4)
            luaPath = stringGsub(luaPath, "\\", ".")
            HotUpdateDic[luaPath] = SysPath
            tableInsert(systemPathList,SysPath)
            -- 初始化旧代码
            ioInput(SysPath)
            OldCode[SysPath] = ioRead("*all")
            ioInput():close()
        end
    end

    file:close()

    return systemPathList
end

function ReloadUtil.InitFakeTable()
    local meta = {}
    ReloadUtil.Meta = meta
    local function FakeT() return setmetatable({}, meta) end
    local function EmptyFunc() end
    local function pairs() return EmptyFunc end
    local function setmetatable(t, metaT)
        ReloadUtil.MetaMap[t] = metaT
        return t
    end
    local function getmetatable(t, metaT)
        return setmetatable({}, t)
    end

    local function require(LuaPath)
        if not ReloadUtil.RequireMap[LuaPath] then
            local FakeTable = FakeT()
            ReloadUtil.RequireMap[LuaPath] = FakeTable
        end
        return ReloadUtil.RequireMap[LuaPath]
    end

    function meta.__index(table, key)
        if key == "setmetatable" then
            return setmetatable
        elseif key == "pairs" or key == "ipairs" then
            return pairs
        elseif key == "next" then
            return EmptyFunc
        elseif key == "require" then
            return require
        else
            local FakeTable = FakeT()
            rawset(table, key, FakeTable)
            return FakeTable
        end
    end
    function meta.__newindex(table, key, value) rawset(table, key, value) end
    function meta.__call() return FakeT(), FakeT(), FakeT() end
    function meta.__add() return meta.__call() end
    function meta.__sub() return meta.__call() end
    function meta.__mul() return meta.__call() end
    function meta.__div() return meta.__call() end
    function meta.__mod() return meta.__call() end
    function meta.__pow() return meta.__call() end
    function meta.__unm() return meta.__call() end
    function meta.__concat() return meta.__call() end
    function meta.__eq() return meta.__call() end
    function meta.__lt() return meta.__call() end
    function meta.__le() return meta.__call() end
    function meta.__len() return meta.__call() end
    return FakeT
end

function ReloadUtil.IsNewCode(SysPath)
    ioInput(SysPath)
    local newCode = ioRead("*all")

    local oldCode = ReloadUtil.OldCode[SysPath]
    if oldCode == newCode then
        ioInput():close()
        return false
    end

    ReloadUtil.DebugNofity(SysPath)
    return true, newCode
end

function ReloadUtil.GetNewObject(newCode, LuaPath, SysPath)
    --loadstring 一段lua代码以后，会经过语法解析返回一个函数，执行返回的函数时，字符串中的代码就被执行了。
    local NewFunction = loadstring(newCode)
    if not NewFunction then
        ReloadUtil.FailNotify(SysPath.." has syntax error.")
        collectgarbage("collect")
    else
        -- 把加载的字符串放置在空环境了，防止报错
        setfenv(NewFunction, ReloadUtil.FakeENV)
        local NewObject
        ReloadUtil.ErrorHappen = false
        --类似其它语言里的 try-catch, xpcall 类似 pcall xpcall接受两个参数：调用函数、错误处理函数
        -- todo 父类没拿到
        xpcall(function () NewObject = NewFunction() end, ReloadUtil.ErrorHandle)

        if not ReloadUtil.ErrorHappen then
            ReloadUtil.OldCode[SysPath] = newCode
            return NewObject
        else
            collectgarbage("collect")
        end
    end
end

function ReloadUtil.ResetENV(object, name, From, Deepth)
    local visited = {}
    local function f(object, name)
        if not object or visited[object] then return end
        visited[object] = true
        if type(object) == "function" then
            ReloadUtil.DebugNofity(Deepth.."HU.ResetENV", name, "  from:"..From)
            xpcall(function () setfenv(object, ReloadUtil.ENV) end, ReloadUtil.FailNotify)
        elseif type(object) == "table" then
            ReloadUtil.DebugNofity(Deepth.."HU.ResetENV", name, "  from:"..From)
            for k, v in pairs(object) do
                f(k, tostring(k).."__key", " HU.ResetENV ", Deepth.."    " )
                f(v, tostring(k), " HU.ResetENV ", Deepth.."    ")
            end
        end
    end
    f(object, name)
end

-- 遍历_G这张全局表，替换HU.ChangedFuncList 有改动列表 的函数
function ReloadUtil.Travel_G()
    local visited = {}
    local ChangedFuncList = ReloadUtil.ChangedFuncList
    visited[ReloadUtil] = true

    local function f(table)
        if (type(table) ~= "function" and type(table) ~= "table") or visited[table] or ReloadUtil.Protection[table] then return end

        visited[table] = true

        if type(table) == "function" then
            for i = 1, mathHuge do
                local name, value = debugGetupvalue(table, i)
                if not name then break end

                if type(value) == "function" then
                    for _, funcs in ipairs(ChangedFuncList) do
                        if value == funcs.OldObject then
                            debugSetupvalue(table, i, funcs.NewObject)
                        end
                    end
                end
                -- todo
                --f(value)
            end
        elseif type(table) == "table" then
            -- 不要漏掉元表和upvalue的表，元表的获取用debug.getmetatable，
            -- todo 这样对于有metatable这个key的元表，也能正确获取。
            --f(debugGetmetatable(table))

            local changeIndexList = {}
            for key, value in pairs(table) do
                -- todo 还有注意table的key也可以是函数。
                --f(key)
                f(value)

                if type(value) == "function" then
                    for _, funcs in ipairs(ChangedFuncList) do
                        if value == funcs.OldObject then
                            table[key] = funcs.NewObject
                        end
                    end
                end

                -- 找出改动的index
                if type(key) == "function" then
                    for index, funcs in ipairs(ChangedFuncList) do
                        if key == funcs.OldObject then
                            changeIndexList[#changeIndexList + 1] = index
                        end
                    end
                end
            end

            -- 修改改动的值
            for _, index in ipairs(changeIndexList) do
                local funcs = ChangedFuncList[index]
                table[funcs.NewObject] = table[funcs.OldObject]
                table[funcs.OldObject] = nil
            end
        end
    end

    --遍历_G这张全局表，_G在registryTable里
    --f(_G)
    --如果有宿主语言，那么还要遍历一下注册表，用debug.getregistry()获得。
    local registryTable = debug.getregistry()
    f(registryTable)
end

function ReloadUtil.UpdateTable(OldTable, NewTable, Name, From, Deepth)
    if ReloadUtil.Protection[OldTable] or ReloadUtil.Protection[NewTable] then return end

    if OldTable == NewTable then return end

    local signature = tostring(OldTable)..tostring(NewTable)

    if ReloadUtil.VisitedSig[signature] then return end

    ReloadUtil.VisitedSig[signature] = true
    ReloadUtil.DebugNofity(Deepth.."HU.UpdateTable "..Name.."  from:"..From)

    for ElementName, newValue in pairs(NewTable) do
        local OldElement = OldTable[ElementName]
        if type(newValue) == type(OldElement) then
            if type(newValue) == "function" then
                ReloadUtil.UpdateOneFunction(OldElement, newValue, ElementName, OldTable, "HU.UpdateTable", Deepth.."    ")
            elseif type(newValue) == "table" then
                ReloadUtil.UpdateTable(OldElement, newValue, ElementName, "HU.UpdateTable", Deepth.."    ")
            end
        elseif OldElement == nil and type(newValue) == "function" then
            -- 新增的函数，添加到旧环境里
            if pcall(setfenv, newValue, ReloadUtil.ENV) then
                OldTable[ElementName] = newValue
            end
        end
    end

    -- todo 更新metatable
    --local OldMeta = debug.getmetatable(OldTable)
    --local NewMeta = ReloadUtil.MetaMap[NewTable]
    --if type(OldMeta) == "table" and type(NewMeta) == "table" then
    --    ReloadUtil.UpdateTable(OldMeta, NewMeta, Name.."'s Meta", "HU.UpdateTable", Deepth.."    ")
    --end
end

-- Upvalue 是指那些函数外被引用到的local变量
function ReloadUtil.UpdateUpvalue(OldFunction, NewFunction, Name, From, Deepth)
    ReloadUtil.DebugNofity(Deepth.."HU.UpdateUpvalue", Name, "  from:"..From)
    local OldUpvalueMap = {}
    local OldExistName = {}
    -- 记录旧的upvalue表
    for i = 1, mathHuge do
        local name, value = debugGetupvalue(OldFunction, i)
        if not name then break end
        OldUpvalueMap[name] = value
        OldExistName[name] = true
    end

    -- 新的upvalue表进行替换
    for i = 1, mathHuge do
        local name, value = debugGetupvalue(NewFunction, i)
        if not name then break end
        if OldExistName[name] then
            local OldValue = OldUpvalueMap[name]
            if type(OldValue) ~= type(value) then
                -- 新的upvalue类型不一致时，用旧的upvalue
                debugSetupvalue(NewFunction, i, OldValue)
            elseif type(OldValue) == "function" then
                -- 替换单个函数
                ReloadUtil.UpdateOneFunction(OldValue, value, name, nil, "HU.UpdateUpvalue", Deepth.."    ")
            elseif type(OldValue) == "table" then
                -- 对table里面的函数继续递归替换
                ReloadUtil.UpdateTable(OldValue, value, name, "HU.UpdateUpvalue", Deepth.."    ")
                debugSetupvalue(NewFunction, i, OldValue)
            else
                -- 其他类型数据有改变，也要用旧的
                debugSetupvalue(NewFunction, i, OldValue)
            end
        else
            -- 对新添加的upvalue设置正确的环境表
            ReloadUtil.ResetENV(value, name, "HU.UpdateUpvalue", Deepth.."    ")
        end
    end
end

function ReloadUtil.UpdateOneFunction(OldObject, NewObject, FuncName, OldTable, From, Deepth)
    if ReloadUtil.Protection[OldObject] or ReloadUtil.Protection[NewObject] then return end

    if OldObject == NewObject then return end

    local signature = tostring(OldObject)..tostring(NewObject)

    if ReloadUtil.VisitedSig[signature] then return end
    ReloadUtil.DebugNofity(Deepth.."HU.UpdateOneFunction "..FuncName.."  from:"..From)
    ReloadUtil.VisitedSig[signature] = true
    --最后注意把热更新的函数的环境表再改回旧函数的环境表即可，方法是setfenv(newfunction, getfenv(oldfunction))。
    if pcall(debugSetfenv, NewObject, getfenv(OldObject)) then
        ReloadUtil.UpdateUpvalue(OldObject, NewObject, FuncName, "HU.UpdateOneFunction", Deepth.."    ")
        ReloadUtil.ChangedFuncList[#ReloadUtil.ChangedFuncList + 1] = {OldObject = OldObject,NewObject = NewObject,FuncName = FuncName,OldTable = OldTable}
    end
end

function ReloadUtil.ReplaceOld(OldObject, NewObject, LuaPath, From)
    if type(OldObject) == type(NewObject) then
        if type(NewObject) == "table" then
            ReloadUtil.UpdateTable(OldObject, NewObject, LuaPath, From, "")
        elseif type(NewObject) == "function" then
            ReloadUtil.UpdateOneFunction(OldObject, NewObject, LuaPath, nil, From, "")
        end
    end
end

function ReloadUtil.HotUpdateCode(LuaPath, SysPath)
    local OldObject = packageLoaded[LuaPath]
    if not OldObject then
        -- 没加载的就不热更？？
        return
    end

    local isNew,newCode = ReloadUtil.IsNewCode(SysPath)
    if not isNew then
        return
    end

    local newObject = ReloadUtil.GetNewObject(newCode,SysPath,LuaPath)
    -- 更新旧代码
    ReloadUtil.ReplaceOld(OldObject, newObject, LuaPath, "Main")

    --原理
    --利用_ENV环境，在加载的时候把数据加载到_ENV下，然后再通过对比的方式修改_G底下的值，从而实现热更新，函数
    --setmetatable(ReloadUtil.FakeENV, nil)
    --todo ？？
    --ReloadUtil.UpdateTable(ReloadUtil.ENV, ReloadUtil.FakeENV, " ENV ", "Main", "")

    -- 替换完，上一次的代码就是旧代码
    ReloadUtil.OldCode[SysPath] = newCode
end


-- 外部调用（先Init需要的路径，然后Update是热更时候调用的）

---@param RootPath 需要被更新的文件夹路径
---@param UpdateListFile 需要被更新的文件列表,不传为整个文件夹，会卡
function ReloadUtil.Init(RootPath, ENV)
    ReloadUtil.HotUpdateDic = {}
    ReloadUtil.FileMap = {}
    ReloadUtil.OldCode = {}
    ReloadUtil.ChangedFuncList = {}
    ReloadUtil.VisitedSig = {}
    ReloadUtil.FakeENV = ReloadUtil.InitFakeTable()()
    --当我们加载lua模块的时候，这时候这个模块信息并不像初始化全局代码一样，就算提前设置了package.loaded["AA"] = nil,
    --也不会出现在env中同时也不会调用_G的__newindex函数，也就是说env["AA"]为空，故这种写法无法进行热更新，所以通常模块的写法改成如下
    --定义模块AA
    local AA = {}
    --相当于package.seeall
    setmetatable(AA, {__index = _G})
    --环境隔离
    local _ENV = AA
    ReloadUtil.NewENV = _ENV
    ReloadUtil.ENV = ENV or _G
    ReloadUtil.InitProtection()
    ReloadUtil.ALL = false
    return ReloadUtil.InitFileMap(RootPath)
end

function ReloadUtil.Update()
    ReloadUtil.VisitedSig = {}
    ReloadUtil.ChangedFuncList = {}
    for LuaPath, SysPath in pairs(ReloadUtil.HotUpdateDic) do
        ReloadUtil.HotUpdateCode(LuaPath, SysPath)
    end

    if #ReloadUtil.ChangedFuncList > 0 then
        ReloadUtil.Travel_G()
    end
    collectgarbage("collect")
end



    我跟老大炫耀的时候，老大说，那你懂其中的原理吗，一下问懵我了，老大说，你要学习到其他的原理才能进步啊，不然就只是个会用工具的人。好有道理，搞得我羞愧难当，赶紧好好学习其中原理。零零碎碎学习了好久，感觉智商不大够。

  upvalue
Upvalue 是指那些函数外被引用到的local变量,比如：
local a = 1
function foo()

    print(a)

end

 那么a就是这个foo的upvalue。

getupvalue (f, up)

此函数返回函数 f 的第 up 个上值的名字和值。 如果该函数没有那个上值，返回 nil 。 
以 '(' （开括号）打头的变量名表示没有名字的变量 （去除了调试信息的代码块）。

setupvalue (f, up, value):

这个函数将 value 设为函数 f 的第 up 个上值。 如果函数没有那个上值，返回 nil 否则，返回该上值的名字。



_G和debug.getregistry这2张表
     学习的时候，我一直以为只有把_G这张全局表的旧值替换掉就好了，然后真正实施的时候，还是会有种种问题，实在是很糟糕，看这段代码的时候一直不是很理解，看了debug.getregistry的定义：

debug.getregistry（）：返回注册表表，这是一个预定义出来的表， 可以用来保存任何 C 代码想保存的 Lua 值。

还是半桶水，但是我有注意到_G这种表其实在debug.getregistry返回的这张注册表里有，所以最后就递归这张表其替换里面的旧值就好了。

getfenv(object):
返回对象的环境变量。

setfenv（function,_ENV）
设置一段代码的运行环境

最后总结一下流程：

1. 初始化需要热更的文件路径，用一张哈希表存下来：

1. 对了InitFileMap这个函数有个要注意的，luaPath在保存的时候是按文件夹路径保存的比如：game.modules.XXX.lua，所以可能要自己搞对这个路径；

2. io.popen("dir /S/B /A:A \""..RootPath.."\"")

获取一个File对象其下的所有文件和目录的绝对路径: 的所有文件(/S/B)，不包括文件夹（/A:A），io.popen返回文件句柄file handle
2.  然后遍历这些路径，io.read("*all")读取所有的代码，判断是否是新代码，新的话记录到ChangedFuncList

列表里面

3. 新代码的话，就把加载的字符串放置在空环境了，防止报错setfenv(NewFunction, ReloadUtil.FakeENV)

4. 代替旧代码，拿着ChangedFuncList列表到注册表(debug.getregistry())里面去找旧值，递归注册表的旧值替换成新的值