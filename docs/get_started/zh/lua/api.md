
# 常用接口

详细的接口定义可在 https://github.com/udbg/luadoc 查看

## UI/日志输出

* `log(...)` `log.warn(...)` `log.info(...)` `log.error(...)` 输出日志
* `log.color(color, ...)` 带颜色的输出 比如 `log.color('green', 'abc\n')`

log系列函数可以输出日志，GUI模式时输出到日志窗口，CUI模式时输出到控制台/终端，远程调试时会将输出重定向到客户端

>! CUI模式下在进行命令输入时，会阻塞`log`系列函数地输出

## GUI接口

0.2之前udbg使用了自己封装的一套RPC-UI接口，源码位于`script/udbg/ui.lua`

0.2版本接入了[lqt](TODO)，可以在lua中很方便的调用qt的功能

如果需要编写GUI，建议使用lqt的接口，0.2之前的RPC-UI接口不会再更新，仅作内部使用

lqt是对Qt的封装，API大部分需要参考Qt的文档，只是要注意参数传递转换的部分，相关资料
* 官方文档 https://github.com/lqt5/lqt/blob/qt5/doc/USAGE.md
* udbg中的实例
  - https://github.com/udbg/udbg/blob/master/udbg/client/ui.lua
  - https://github.com/udbg/udbg/blob/master/udbg/client/__init.lua

>! 
  * 使用lqt的代码必须用`require 'udbg.client.qt'.gui()`函数封装保证在主线程中运行
  * 远程调试时，lqt只在客户端环境中可以使用

TODO 补充一些用法说明

## FFI接口

udbg自带了两套ffi接口
1. 自己封装的libffi接口 `require 'libffi'` https://github.com/udbg/luadoc/blob/master/libffi.lua
2. 从0.2版本开始集成了[兼容luajit的luaffi](https://github.com/facebookarchive/luaffifb) `require 'ffi'`

udbg自己封装的libffi相比luaffi可以无类型(不用声明参数)调用C函数，但访问结构体比较麻烦；

建议用luaffi，文档较多且相对完善 http://luajit.org/ext_ffi.html 如果是临时测试一些接口可以用libffi，比较方便

以下是自带的libffi库用法介绍

### 无类型调用

```lua
local MessageBoxA = libffi.C.GetProcAddress(libffi.C.LoadLibraryA('user32'), 'MessageBoxA')
local msgbox = libffi.fn(MessageBoxA)
msgbox(0, 'ABC', 'DEF', 0)
```

`libffi.fn`函数直接传入一个整数（C函数地址），返回一个无类型的函数对象，无类型的函数对象会根据传入的lua参数类型自动为每个C参数分配类型，映射规则如下
- `string|userdata` => pointer
- `integer|boolean` => size_t
- `nil|none` => NULL
- `number` => double

无类型的调用在于写起来非常简单，能够覆盖大部分应用场景，但有些时候还是需要知道明确的函数参数才能成功调用一些函数，比如涉及到浮点数的函数、非标准的调用约定等情况

### 声明函数类型

声明函数类型需要指定函数的返回值类型和参数类型 `libffi.fn(returnType, {argsType...})`，支持的类型如下

- `void`
- `char`  `int8`
- `byte`  `uchar`
- `short`  `int16`
- `ushort`  `uint16`
- `int`  `int32`
- `uint`  `uint32`
- `int64`  `long long`
- `uint64`
- `float`
- `double`
- `pointer`

```lua
local pow = api.GetProcAddress(api.LoadLibraryA'msvcrt', 'pow')
-- 通过fn声明函数类型
pow = libffi.fn('double', {'double', 'double'})(pow)
log('powf(2, 2)', pow(2, 2))

local powf = api.GetProcAddress(api.LoadLibraryA'msvcrt', 'powf')
powf = libffi.fn('float', {'float', 'float'})(powf)
log('powf(2, 3)', powf(2, 3))
```

指定x86调用约定： TODO

### 生成回调函数

示例如下
```lua
local WNDENUMPROC = libffi.fn('int', {'pointer', 'pointer'})
api.EnumChildWindows(0, WNDENUMPROC(function(hwnd, param)
    log('HWND:', hwnd)
end), 0)
```

## 调试器接口

详细用法参考注解文档 https://github.com/udbg/luadoc/blob/master/_udbg.lua

下面的函数在0.2之前是全局函数，0.2开始成了UDbgTarget对象的方法；为了兼容0.1 udbg中对全局变量表设置了元方法，可以以全局函数的方式调用UDbgTarget的方法，这些函数使用时建议加`udbg.target:` 比如 `udbg.target:enum_module()` `udbg.target:get_address(0x40000)`

断点管理
* add_bp() 返回断点对象 UDbgBreakpoint

模块/符号
* enum_module()
* get_module()
* get_symbol(address) 获取指定地址处的符号
* parse_address(address) 解析地址(符号表达式) 将符号地址解析为地址数值

查询/枚举内存页
* `virtual_query(address) -> MemoryInfo` 查询指定地址处的内存页属性。返回值为[MemoryInfo](#MemoryInfo) 或 `nil`
* `enum_memory()` 枚举内存页 详见
  ```lua
  for m in enum_memory() do
      log(hex(m.base), hex(m.size), m.protect)
  end
  ```

读写内存
* `read_string(address, [max_bytes])` 读取以`\0`结尾的C字符串
* `read_wstring(address, [max_bytes])` 读取宽字符串(Windows Only)
* `read_bytes(address, size)` 读取一段内存数据
* `write_bytes(address, bytes)` 写入内存数据
* `write_string(address, str)` 写入以0结尾的C字符串
* `write_wstring(address, wstr)` 写入宽字符串(0结尾)
* `read_{type}(address)` 读取一个数字 `write_{type}(address, number)` 写入一个数字 `{type}`可以是：
  * `u8` 字节
  * `u16` WORD
  * `u32` DWORD
  * `u64` QWORD
  * `ptr` 指针类型

线程信息
* enum_thread() 枚举线程
```lua
for tid in enum_thread() do
    local t = open_thread(tid)
    if t then
        log(t.name)     -- 线程名称，Linux下有效
        log(t.teb)      -- 线程的TEB地址，Windows下有效
        log(t.entry)    -- 线程创建时的入口，Windows下有效
    end
end
```
* thread_list() 获取详细的线程列表
```lua
for tid, t in pairs(thread_list()) do
    log(t.name)     -- 线程名称，Linux下有效
    log(t.teb)      -- 线程的TEB地址，Windows下有效
    log(t.entry)    -- 线程创建时的入口，Windows下有效
end
```

全局变量`reg`: 断点或异常触发时的寄存器值
* x64下可用寄存器：`rax` `rbx` `rcx` `rdx` `rsp` `rbp` `rip` `rsi` `rdi` `r8` .. `r15` `rflags` 浮点数(float)：`mm0` .. `mm7` 浮点数(double)：`xmm0` .. `xmm15`
* x86下可用寄存器：`eax` `ebx` `ecx` `edx` `esp` `ebp` `eip` `esi` `edi` `eflags` 浮点数(float)：`mm0` .. `mm7` 浮点数(double)：`xmm0` .. `xmm15`
* 可用整数索引参数值，x64下 `reg[1] = reg.rcx` `reg[2] = reg.rdx` `reg[3] = reg.r8` `reg[4] = reg.r9`，其他整数 `reg[i] = qword(reg.rsp + i * 8)`；x86下 `reg[i] = dword(reg.esp + i * 4)`

## 事件机制

通过事件模块`require 'udbg.event'`，各个插件可以订阅自己感兴趣的事件，并作出自己的处理，或者阻止事件继续往下传递

具体的API可以查看`script/udbg/event.lua`文件中的文档注解

事件的处理机制示例：
```lua
local event = require 'udbg.event'

-- 订阅模块加载事件，示例：在入口处下断点
event.on('targetModuleLoad', function(m)
    add_bp(m.entry_point)
end)

-- 订阅调试异常事件，示例：忽略指定的异常
event.on('targetException', function(tid, code, first)
    if code == 0x1000000 then
        -- 忽略异常码为 0x1000000 的异常
        return 'run'
        -- 如果返回一个真值，后面的事件回调将不会执行
    end
end)

-- 事件处理器可以是一个协程，对于协程处理器，每resume一次都会取消，协程每次yield之前都要订阅事件
event.on('uiInited', coroutine.create(function()
    -- 订阅 targetInitBp 事件
    uevent.on 'targetInitBp'
    coroutine.yield()

    -- targetInitBp 事件触发
    log 'targetInitBp'

    -- 订阅 targetEnded 事件
    uevent.on 'targetEnded'
    coroutine.yield()

    log 'targetEnded'
end))

-- 自定义事件
event.on('customEvent', function(arg1, arg2)
    log('sub1', arg1, arg2)
    if arg1 == 'hijack' then
        -- 如果返回一个真值，后面的事件回调将不会执行
        return 3
    end
end)

event.on('customEvent', function(arg1, arg2)
    log('sub2', arg1 + arg2)
    -- 取消订阅事件
    event.cancel()
end)

-- result的结果为3，对应第一个事件订阅者的返回值
local result = event.fire('customEvent', 'hijack', 2)
-- result的结果为nil，对应最后一个事件订阅者的返回值
local result = event.fire('customEvent', 1, 2)
```

## 调试事件

### 客户端事件

客户端事件只在客户端环境中触发，参考[客户端环境与调试器环境](./required-intro.md#客户端环境与调试器环境)

* `clientUiInited` 此时客户端UI已经初始化完成，RPC连接已建立

### 调试器事件

* `uiInited`：客户端环境中 `clientUiInited` 事件处理完成后，会通过RPC触发调试器环境中的 `uiInited` 事件；一般用于在调试器环境中获得一个时机，来初始化一些东西

以下事件都是和调试目标/调试循环相关的事件
* `targetTargetSuccess` 目标创建/附加成功
* `targetInitBp` 初始断点(或者叫系统断点)
* `targetStep` 单步异常
* `targetBreakpoint` 断点异常
* `targetTargetEnded` 目标运行结束/Detach
* `targetException` (tid: integer, code: integer, first: bool)
* `targetModuleLoad` (m: UDbgModule)
* `targetModuleUnload` (m: UDbgModule)
* `targetProcessCreate` (pid: integer)
* `targetProcessExit` (code: integer)
* `targetThreadCreate` 线程创建
* `targetThreadExit` 线程退出