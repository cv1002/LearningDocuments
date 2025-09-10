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
    local args = {...}
end

function func(...)
    select('#', ...)
end
```

## Lua 的 metatable
```lua
setmetatable(table, metatable)
```
