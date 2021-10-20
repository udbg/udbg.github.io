
# 使用教程

udbg使用了Lua编写了大量逻辑代码，提供了丰富的Lua接口来定制化分析，就像Frida提供了一系列JS接口，而且udbg提供了基本的GUI以及扩展GUI的接口；要想高效地使用udbg，离不开编写Lua脚本，所以会优先介绍udbg中Lua脚本的使用

## 脚本执行

假定帅气的你已经下载好了udbg，并将udbg的目录加到了PATH环境变量，命令行中执行`udbg --version`应该能看到版本信息，那我们就可以开始了

新建一个工作目录，比如叫`udbg-lua`，我们后续要给udbg执行的lua脚本都放在这个目录里

    mkdir udbg-lua
    cd udbg-lua

从一个简单的例子开始：获取调试目标中所加载的模块。新建一个文件，比如 start.lua，写入以下脚本
```lua
uevent.on('targetInitBp', function()
    for m in udbg.target:enum_module() do
        log('[module]', hex(m.base), m.name)
    end
end)
```
执行`udbg -e start.lua notepad`，在日志窗口里应该可以看到打印的日志；其中`uevent.on('targetInitBp', ...)`表示在初始断点这个事件触发时执行后面的回调函数，回调函数里枚举了调试目标已经加载了的模块；`udbg.target`代表正在调试的目标对象，没有调试目标时，udbg.target是nil

关掉调试器，再来看个断点的例子：在CreateFileW函数上下断点并且打印第一个参数
```lua
uevent.on('targetInitBp', function()
    local target = udbg.target
    target:add_bp('kernel32!CreateFileW', function()
        local path = target:read_wstring(reg[1])
        log('[CreateFileW]', path)
    end)
    return 'run'
end)
```
执行`udbg -e start.lua notepad`，应该会直接弹出notepad的窗口，然后从‘文件’菜单中选择‘打开’，便会触发CreateFileW函数的执行，不出意外的话日志窗口也会有我们打印的log

其中reg变量表示断点触发时的寄存器环境，可通过`reg.寄存器名`的形式来获取寄存器的值，`reg[1]`封装了获取函数参数的逻辑，x64下相当于`reg.rcx`；read_wstring方法读取第一个参数对应的utf16字符串(读取之后转换成了utf8)

注意这次在targetInitBp事件的回调函数中返回了'run'，表示初始断点断下后继续运行，不暂停给用户处理

前面两个例子都是通过启动参数来执行脚本，实际分析过程中我们需要实时执行lua来更新逻辑，实时执行可以通过`.exec 脚本路径`命令完成，也可以加`-w`参数，在脚本保存时自动执行，还可以通过配置插件目录，指定自己常用的自动执行脚本

接着上面的例子，在启动之后我们想对CreateFileW的断点逻辑进行更改，可以新建一个文件，比如 auto.lua，然后在udbg的右下角命令框里输入`.exec -w auto.lua`将会执行一次auto.lua，然监控auto.lua文件的写入事件自动执行；要更改的断点逻辑是当CreateFileW的路径中包含test.txt时断下，将以下代码复制到auto.lua并保存
```lua
local target = udbg.target
target:add_bp('kernel32!CreateFileW', function()
    local path = target:read_wstring(reg[1])
    log('[CreateFileW]', path)
    if path:find 'test.txt' then
        return true
    end
end)
```
在断点回调函数中`return true`表示这次断点将中断给用户处理

最后再看一个复杂点的例子，打印CreateFileW的参数及其返回值；将以下代码复制到auto.lua并保存
```lua
local target = udbg.target
target:add_bp('kernel32!CreateFileW', function()
    local path = target:read_wstring(reg[1])
    log('[CreateFileW]', path)

    target:add_bp(reg._sp, {
        callback = function() log('  return', reg.rax) end,
        temp = true, type = 'table'
    })
end)
```
‘文件’->‘打开’，触发CreateFileW函数的执行，在日志中应该可以看到CreateFileW的路径参数及其返回值。

更多Lua的用法参考[脚本示例](../lua/script-example.html)中的内容，API参考 https://github.com/udbg/luadoc/blob/master/_udbg.lua

## 启动调试目标

脚本执行常用于定制化分析，一般的分析过程有很多种方式来启动调试目标

1. 双击 udbg.exe 直接启动，调试器内通过File->Attach菜单选择要附加的调试目标
2. 命令行带参数启动；常用命令行：
  - `udbg` 启动udbg，并显示主窗口，相当于在资源管理器中双击udbg.exe进行启动
  - `udbg -W` 无窗口启动udbg，在命令行中调试
  - `udbg -e test.lua` 启动udbg并执行 test.lua 脚本
  - `udbg notepad.exe` (默认的调试引擎) 创建并调试 notepad.exe
  - `udbg notepad.exe -- test.txt test2.txt` (默认的调试引擎) 创建并调试 notepad.exe，传递**命令行参数** `test.txt test2.txt`
  - `udbg -a notepad.exe` (默认的调试引擎) 附加到 notepad.exe
  - `udbg -A uspy notepad.exe` 通过uspy调试引擎 创建并调试 notepad.exe
  - `udbg -A uspy -a notepad.exe` 通过uspy调试引擎 附加到 notepad.exe
  - `udbg -o explorer.exe` 打开explorer.exe进程，不调试
  - `udbg -r localhost:2333` 连接到udbg-server，并启动，后面可跟上面所有的参数，参考 [远程调试](#远程调试)
3. 调试器内执行`dbg`命令启动调试目标，参数和udbg的命令行参数类型，详见`dbg -h`

udbg完整的命令行语法如下

    udbg 0.1.0
    metaworm

    USAGE:
        udbg.exe [FLAGS] [OPTIONS] [--] [ARGS]

    FLAGS:
        -a, --attach       Attach target
        -h, --help         Prints help information
        -W, --no-window    Dont show the main window
        -o, --open         Open the target, not attach
        -p, --pid          Target as pid
            --version      Prints version information
        -V, --verbose      Show the verbose info
        -w, --watch        Watch the executed lua script

    OPTIONS:
        -A, --adaptor <adaptor>          Specify the adaptor [default: ]
        -r, --remote <address:port>      Connect to udbg server, with cui [env: UDBG_SERVER=]
            --cwd <directory>            Set CWD for target
        -e, --execute <lua path>         Execute lua script
        -c, --config <udbg config>...    Set the __config

    ARGS:
        <target-path>    Create debug target
        <args>...        Shell Arguments to target

## CUI && GUI

udbg也提供了基本的UI界面，以下是UI使用相关的简单介绍

如果不加任何参数启动udbg，默认会显示udbg窗口，工作于GUI模式下；`udbg -W`启动udbg会工作于CUI模式下，其实窗口也创建了，只不过没有显示；
可通过`Alt+;`快捷键来切换GUI模式和CUI模式；

- GUI模式下可以方便直观地查看调试目标的各种数据，可以交互式调试/执行命令，一般也是工作在GUI模式下；
- CUI模式用于纯命令行交互，或者执行脚本时方便查看输出(比如vscode中编辑保存脚本，即可在下方地终端窗口中查看输出)

>! 注意：CUI模式下在进行命令输入时，会阻塞`log`系列函数地输出

### 快捷键

CUI & GUI 通用
- `F7` 单步步进
- `F8` 单步步过
- `F9` 继续运行(未处理异常)
- `Shift+F9` 继续运行(已处理异常)
- `Ctrl+F9` (单步)运行到返回

GUI：
- `F12` 中断目标

CUI:
- `Ctrl+D` 中断目标

### CUI、GUI切换

- `Alt+;` 在CUI、GUI之间切换(隐藏主窗口)
- `Alt+/` 从GUI切换到CUI界面(不隐藏主窗口)

## 断点命令

设置断点最简单的方法是在反汇编视图中，选择对应汇编语句，按`F2`键

一般的条件断点、日志断点可以通过bp命令来完成，复杂的断点逻辑处理需要编写lua脚本实现
- 常规断点 `bp CreateFileW` 在 CreateFileW 函数处下断点
- 日志断点 `bp CreateFileW wstr(reg[1])` 在 CreateFileW 函数处下断点，并显示第一个参数
  - 其中`wstr(reg[1])`是一个lua表达式，表示需要记录的内容，`reg[1]`表示当前架构下标准调用约定的第一个参数(x64下相当于`reg.rcx`)，可跟多个表达式比如 `bp CreateFileW wstr(reg[1]) reg.rdx`
  - 如果想过滤需要记录的日志，可加`-f`参数+lua表达式，比如 `bp CreateFileW wstr(reg[1]) -f v1:find'log.txt'`可以过滤带有log.txt的路径，其中v1表示CreateFileW后面的表达式列表里的第一个表达式的值
- 条件断点 `bp CreateFileW wstr(reg[1]) -c v1:find'log.txt'` 当CreateFileW的第一个参数路径包含 log.txt 时断下
- 硬件断点 `bp CreateFileW -t e1` 临时(一次性)断点 `bp CreateFileW --temp`

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