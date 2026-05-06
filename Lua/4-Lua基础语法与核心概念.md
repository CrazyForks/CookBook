# Lua 基础语法与核心概念

## 1. Lua 概述

```
Lua 是什么？

┌─────────────────────────────────────────────────────────────┐
│                      Lua 语言                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  定义: 轻量级、可嵌入的脚本语言                            │
│                                                             │
│  核心特点:                                                  │
│  ├── 轻量级: 解释器仅 200KB                               │
│  ├── 高性能: 接近 C 语言性能                               │
│  ├── 可嵌入: 完美嵌入 C/C++ 程序                          │
│  ├── 动态类型: 变量无需声明类型                            │
│  └── 垃圾回收: 自动内存管理                                │
│                                                             │
│  应用场景:                                                  │
│  ├── 游戏开发: World of Warcraft, Angry Birds              │
│  ├── Web 服务: OpenResty (Nginx + Lua)                    │
│  ├── 数据库: Redis Lua 脚本                               │
│  ├── 嵌入式: 路由器、IoT 设备                             │
│  └── 配置管理: Nginx, Redis 配置                          │
│                                                             │
│  版本:                                                      │
│  ├── Lua 5.1: 2006年，稳定版本                             │
│  ├── Lua 5.3: 2015年，整数类型                             │
│  ├── Lua 5.4: 2020年，最新版本                             │
│  └── LuaJIT: JIT 编译版本，性能更高                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 基础语法

### 2.1 数据类型

```lua
-- Lua 数据类型
-- nil: 空值
local a = nil

-- boolean: 布尔值
local b = true
local c = false

-- number: 数字（默认双精度浮点）
local d = 42
local e = 3.14
local f = 1.5e10

-- string: 字符串
local g = "Hello"
local h = 'World'
local i = [[
  多行
  字符串
]]

-- table: 表（Lua 唯一的数据结构）
local j = {1, 2, 3}           -- 数组
local k = {name="张三", age=25}  -- 字典
local l = {["key"]="value"}   -- 字典

-- function: 函数
local m = function(x) return x * 2 end

-- userdata: 用户数据（C 语言数据）
-- thread: 协程
```

### 2.2 变量

```lua
-- 全局变量（默认）
global_var = "I'm global"

-- 局部变量（推荐）
local local_var = "I'm local"

-- 多变量赋值
local a, b, c = 1, 2, 3

-- 交换变量
a, b = b, a

-- 变量作用域
do
    local x = 10
    print(x)  -- 10
end
-- print(x)  -- 错误: x 未定义
```

### 2.3 运算符

```lua
-- 算术运算符
local a = 10 + 5    -- 15
local b = 10 - 5    -- 5
local c = 10 * 5    -- 50
local d = 10 / 5    -- 2.0
local e = 10 % 3    -- 1
local f = 10 ^ 2    -- 100

-- 比较运算符
local g = (10 == 10)  -- true
local h = (10 ~= 5)   -- true（不等于）
local i = (10 > 5)    -- true
local j = (10 < 5)    -- false
local k = (10 >= 10)  -- true
local l = (10 <= 5)   -- false

-- 逻辑运算符
local m = true and false  -- false
local n = true or false   -- true
local o = not true        -- false

-- 字符串连接
local p = "Hello" .. " " .. "World"  -- "Hello World"

-- 字符串长度
local q = #("Hello")  -- 5
```

### 2.4 控制结构

```lua
-- if 语句
if condition then
    -- code
elseif condition2 then
    -- code
else
    -- code
end

-- 示例
local score = 85
if score >= 90 then
    print("优秀")
elseif score >= 80 then
    print("良好")
elseif score >= 60 then
    print("及格")
else
    print("不及格")
end

-- while 循环
local i = 1
while i <= 10 do
    print(i)
    i = i + 1
end

-- repeat...until 循环（至少执行一次）
local j = 1
repeat
    print(j)
    j = j + 1
until j > 10

-- for 循环（数值型）
for i = 1, 10 do
    print(i)
end

for i = 10, 1, -1 do  -- 从10到1，步长-1
    print(i)
end

-- for 循环（泛型）
local fruits = {"apple", "banana", "orange"}
for index, value in ipairs(fruits) do
    print(index, value)
end

-- break 和 goto（Lua 5.2+）
for i = 1, 10 do
    if i == 5 then
        break
    end
    print(i)
end
```

## 3. 函数

```lua
-- 基本函数
local function add(a, b)
    return a + b
end

-- 匿名函数
local multiply = function(a, b)
    return a * b
end

-- 多返回值
local function divide(a, b)
    if b == 0 then
        return nil, "division by zero"
    end
    return a / b, a % b
end

local result, remainder = divide(10, 3)

-- 可变参数
local function sum(...)
    local args = {...}
    local total = 0
    for _, v in ipairs(args) do
        total = total + v
    end
    return total
end

print(sum(1, 2, 3, 4, 5))  -- 15

-- 函数作为参数
local function apply(func, value)
    return func(value)
end

print(apply(function(x) return x * 2 end, 5))  -- 10

-- 闭包
local function counter()
    local count = 0
    return function()
        count = count + 1
        return count
    end
end

local c = counter()
print(c())  -- 1
print(c())  -- 2
print(c())  -- 3
```

## 4. 表（Table）

```lua
-- 数组
local fruits = {"apple", "banana", "orange"}
print(fruits[1])      -- "apple"（Lua 索引从 1 开始）
print(#fruits)        -- 3（长度）

-- 字典
local person = {
    name = "张三",
    age = 25,
    city = "北京"
}
print(person.name)    -- "张三"
print(person["age"])  -- 25

-- 嵌套表
local users = {
    {name="张三", age=25},
    {name="李四", age=30}
}
print(users[1].name)  -- "张三"

-- 表操作
table.insert(fruits, "grape")      -- 添加元素
table.remove(fruits, 1)            -- 删除元素
table.sort(fruits)                 -- 排序

-- 遍历数组
for i, v in ipairs(fruits) do
    print(i, v)
end

-- 遍历字典
for k, v in pairs(person) do
    print(k, v)
end

-- 元表（Metatable）
local mt = {
    __add = function(a, b)
        return {value = a.value + b.value}
    end,
    __tostring = function(a)
        return "Value: " .. a.value
    end
}

local obj1 = {value = 10}
local obj2 = {value = 20}
setmetatable(obj1, mt)
setmetatable(obj2, mt)

local obj3 = obj1 + obj2  -- 调用 __add
print(obj3.value)         -- 30
print(tostring(obj1))     -- "Value: 10"
```

## 5. 模块

```lua
-- mymodule.lua
local M = {}

function M.greet(name)
    return "Hello, " .. name .. "!"
end

function M.add(a, b)
    return a + b
end

return M

-- 使用模块
local mymodule = require("mymodule")
print(mymodule.greet("World"))  -- "Hello, World!"
print(mymodule.add(1, 2))      -- 3
```

## 6. 协程

```lua
-- 创建协程
local co = coroutine.create(function()
    print("Step 1")
    coroutine.yield()
    print("Step 2")
    coroutine.yield()
    print("Step 3")
end)

print(coroutine.status(co))  -- "suspended"

coroutine.resume(co)  -- "Step 1"
print(coroutine.status(co))  -- "suspended"

coroutine.resume(co)  -- "Step 2"
coroutine.resume(co)  -- "Step 3"
print(coroutine.status(co))  -- "dead"

-- 生产者-消费者模式
local function producer()
    return coroutine.create(function()
        while true do
            local x = io.read()
            coroutine.yield(x)
        end
    end)
end

local function consumer(co)
    while true do
        local status, value = coroutine.resume(co)
        if value then
            print("Received: " .. value)
        end
    end
end
```

## 7. 错误处理

```lua
-- error 函数
local function validate(age)
    if type(age) ~= "number" then
        error("年龄必须是数字")
    end
    if age < 0 or age > 150 then
        error("年龄必须在 0-150 之间")
    end
    return true
end

-- pcall（受保护调用）
local status, err = pcall(function()
    error("Something went wrong")
end)
print(status)  -- false
print(err)     -- "Something went wrong"

-- xpcall（带错误处理函数）
local function errorHandler(err)
    return "Error: " .. err
end

local status, result = xpcall(function()
    error("test error")
end, errorHandler)
print(result)  -- "Error: test error"
```

## 8. 字符串处理

```lua
-- 字符串操作
local s = "Hello, World!"

print(string.len(s))           -- 13
print(string.upper(s))         -- "HELLO, WORLD!"
print(string.lower(s))         -- "hello, world!"
print(string.sub(s, 1, 5))     -- "Hello"
print(string.find(s, "World")) -- 8 12
print(string.gsub(s, "World", "Lua"))  -- "Hello, Lua!" 1
print(string.rep("ab", 3))     -- "ababab"
print(string.reverse(s))       -- "!dlroW ,olleH"

-- 格式化
print(string.format("Name: %s, Age: %d", "张三", 25))
print(string.format("%.2f", 3.14159))  -- "3.14"

-- 模式匹配
local year, month, day = string.match("2024-01-15", "(%d+)-(%d+)-(%d+)")
print(year, month, day)  -- 2024 01 15

-- 字符串分割
local function split(str, delimiter)
    local result = {}
    for match in (str .. delimiter):gmatch("(.-)" .. delimiter) do
        table.insert(result, match)
    end
    return result
end

local parts = split("a,b,c,d", ",")
for i, v in ipairs(parts) do
    print(i, v)
end
```
