
# 脚本示例

<!-- 更多使用脚本的例子参考[本站博客](/blog) -->

## 断点与Hook

[使用教程](../quick-start/tutorial.html)里已有详细说明

```lua
local target = udbg.target
target:add_bp('kernel32!CreateFileW', function()
    local path = target:read_wstring(reg[1])
    log('[CreateFileW]', path)
end)
```

```lua
target:add_bp('kernel32!CreateFileW', function()
    local path = target:read_wstring(reg[1])
    log('[CreateFileW]', path)
    if path:find 'test.txt' then
        return true
    end
end)
```

```lua
target:add_bp('kernel32!CreateFileW', function()
    local path = target:read_wstring(reg[1])
    log('[CreateFileW]', path)

    target:add_bp(reg._sp, {
        callback = function() log('  return', reg.rax) end,
        temp = true, type = 'table'
    })
end)
```

## 自动化调试

以upx脱壳为例，以下脚本会自动定位到OEP (*upx 3.96, windows 32位exe测试通过*)

保存到upx.lua，执行`udbg -e upx.lua <加壳的exe路径>`查看效果
```lua
local yield = coroutine.yield

-- 忽略初始断点
__config.ignore_initbp = true
-- 进程入口断点
__config.bp_process_entry = true

-- 等待入口断点
uevent.on 'targetBreakpoint'; yield 'run'
log('[upx]', '[entry]', get_symbol(reg._pc))
assert(disasm(reg._pc).string:find '^pusha', 'not a upx program')

-- pusha 单步 下ESP硬件访问断点
uevent.on 'targetStep'; yield 'step'

udbg.target:add_bp(reg._sp, {temp = true, type = 'access'})

-- 等待ESP断点
uevent.on 'targetBreakpoint'; yield 'run'
log('[upx]', '...ESP')

local iter = enum_disasm(reg._pc)
for i = 1, 10, 1 do
    local insn = iter()
    if insn.mnemonic == 'jmp' then
        uevent.on 'targetBreakpoint' yield('goto', insn.address)
        uevent.on 'targetStep' yield 'step'
        return ui.pause('OEP')
    end
end
```

其中`uevent.on '<event>'`表示等待某个事件，`yield(...)`表示对事件处理的返回值，目标因断点或者单步断下时，通过yield告知调试器下一步的动作，可以是
- 'run' 继续运行
- 'step' 单步运行
- 'stepout' 单步跳过
- 'goto', address 运行到某个地址
- 'go_ret' 运行到返回

## 指令追踪

在调试目标断下时，可以通过以下脚本来启动一个单步跟踪过程

```lua
-- 单步跟踪1000步并输出每一步的汇编语句
local count = 1000
local disasm = disasm
ui.continue('step', function()
    count = count - 1
    local pc = reg._pc
    log(hex(pc), disasm(pc).string)
    -- 返回true表示停止单步跟踪
    return count == 0
end)
```

## Unicorn模拟执行

## GUI相关

## 远程调试相关

<!-- ## uspy相关 -->