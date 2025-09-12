# Lua

## Lua 的 if
```lua
if condition then
end
```

## Lua 的 for
```lua
for i = 1, #arr do
end

for _, v in pairs(t) do
end

for _, v in ipairs(t) do
end
```

## Lua 的 while
```lua
while condition do
end
```

## Lua 的 repeat
```lua
repeat
until condition
```

## Lua 的 function
```lua
function func(args)
end

function func(...)
    local args = {...} -- 变长参数用 table 接收
end

function func(...)
    select('#', ...) -- 获取变长参数的长度
end
```

## Lua 的 metatable
```lua
setmetatable(table, metatable) -- 设置元表
getmetatable(table) -- 获取元表
```

## Lua 错误处理
```lua
if pcall(func, ...) then
    -- 处理成功
else
    -- 处理错误
end

-- 函数
function func ()
   n = n / nil
end

-- 错误处理
function errorhandler(err)
   print("ERROR occured: ", err)
   print("Here is the traceback: ")
   print(debug.traceback())
end

status = xpcall(func, errorHandler)
```

