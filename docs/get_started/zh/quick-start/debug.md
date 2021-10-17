
## 断点

设置断点最简单的方法是在反汇编视图中，选择对应汇编语句，按`F2`键

但如果想设置更复杂的断点需要使用`bp`命令，列几个常用的断点设置示例
- 常规断点 `bp kernel32!CreateFileW` 在 CreateFileW 函数处下断点
- 日志断点 `bp kernel32!CreateFileW wstr(reg[1])` 在 CreateFileW 函数处下断点，并显示第一个参数
  - 其中`wstr(reg[1])`是一个lua表达式，表示需要记录的内容，`reg[1]`表示当前架构下标准调用约定的第一个参数(x64下相当于`reg.rcx`)，可跟多个表达式比如 `bp kernel32!CreateFileW wstr(reg[1]) reg.rdx`
  - 如果想过滤需要记录的日志，可加`-f`参数+lua表达式，比如 `bp kernel32!CreateFileW wstr(reg[1]) -f v1:find'log.txt'`可以过滤带有log.txt的路径，其中v1表示CreateFileW后面的表达式列表里的第一个表达式的值
- 条件断点 `bp kernel32!CreateFileW wstr(reg[1]) -c v1:find'log.txt'` 当CreateFileW的第一个参数路径包含 log.txt 时断下
- 硬件断点 `bp kernel32!CreateFileW -t e1` 临时(一次性)断点 `bp CreateFileW --temp`

bp命令的完整帮助 `bp -h`

    bp                                        设置断点
        <address>       (string)              断点地址
        <vars...>       (optional string)     变量列表

        -n, --name        (optional string)   断点名称
        -c, --cond        (optional string)   中断条件
        -l, --log         (optional string)   日志表达式
        -f, --filter      (optional string)   日志条件
        -s, --statistics  (optional string)   统计表达式
        -t, --type        (optional bp_type)  断点类型(e|w|a)(1|2|4|8)
        --tid             (optional number)   命中线程
        --temp                                临时断点
        --symbol                              转换为符号
        --hex                                 十六进制显示
        --caller                              显示调用者
        -m, --module                          模块加载断点

如果需要了解bp命令的详细实现方式，参考`script/udbg/command/bp.lua`

### 自定义断点回调

通过脚本定制更复杂的断点处理逻辑

```lua
add_bp('kernel32!CreateFileW', function()
    local path = read_wstring(reg.rcx)
    log('[CreateFileW]', path)
    if path:find 'xxx' then
        -- 返回true表示这个断点会中断给用户处理
        return true
    end
    -- 在调用 CreateFileW 的返回点处设置断点
    add_bp(reg.rsp, function()
        log('[CreateFileW]', 'return', reg.rax)
    end, {temp = true, type = 'table'})
end)
```

## 堆栈回溯

除了堆栈视图，还可以使用`dp -r rsp`来查看堆栈；udbg目前未实现基于符号的堆栈回溯，`dp -r`本质上是暴力搜索堆栈上的返回地址

## 模块符号

udbg自带的调试引擎支持加载pdb符号，可在配置文件中通过`__config.symbol_cache = 'D:/Symbols`来指定pdb符号所在的目录，目录结构和windbg的符号缓存目录一致，可以在windbg里下载系统所需要的符号

udbg首先会根据dll/exe里的pdb路径去加载符号，如果没有的话再寻找dll所在目录下的同名pdb，如果也没有就会去符号缓存目录中加载

如果想给一个模块手动指定pdb路径，可以使用命令 `load-symbol xx.dll D:\xx.pdb`

也可以在配置文件中写脚本来自动加载
```lua
function uevent.on.module_load(m)
    if m.base == PA 'xx.dll' then
        m:load_symbol [[D:\xx.pdb]]
    end
end
```
<!-- 
## 内存搜索

参考yara命令
- 搜索特征码（binary_search） `yara -b 'FF 15 ?? ?? ?? ??'`
- 搜索字符串 UTF16 `yara -u '你好'` UTF8 `yara -a hello`
- 在某个模块内搜索 `yara -b 'FF 15 ?? ?? ?? ??' -m ntdll` -->

## 内存读写

- 直接在内存视图中查看
  - `Ctrl+1` `Ctrl+2` `Ctrl+3` `Ctrl+4` 分别切换BYTE WORD DWORD QWORD显示
  - `Ctrl+F`切换float显示
  - `Ctrl+D`切换double显示
  - `Ctrl+G`可以输入地址表达式并跳转到对应地址
- `mem <address>` 命令在内存视图中 跳转到对应的地址表达式 `mem ntdll.dll` `mem [[0x403000]+0x10]`
- `e` 命令编辑内存
- Lua函数 `{read|write}_{u8|u16|u32|u64|ptr|float|double}`

## 异常处理

### 相关配置

- `__config.ignore_all_exception = false` 是否忽略所有异常

### 处理异常事件

可以通过脚本配置更复杂的异常处理

```lua
function uevent.on.exception(tid, code, first)
    if code == STATUS_CPP_EH_EXCEPTION then
        return 'run'
    end
    if reg.rip == 0 then
        return 'run'
    end
end
```

## 线程列表

## 单步跟踪

在目标断下时，可以通过如下脚本来启动一个单步跟踪过程

```lua
-- 单步跟踪1000步并输出每一步的汇编语句
local count = 1000
local disasm = disasm
ui.continue('step', function()
    count = count - 1
    local pc = reg._pc
    log(hex(pc), disasm(pc).string)
    return count == 0
end)
```

<!-- ## 汇编、反汇编 -->