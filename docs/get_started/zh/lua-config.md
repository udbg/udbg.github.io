
udbg里使用的lua基于lua5.4版本 [lua5.4手册](http://www.lua.org/manual/5.4/manual.html)

除了`LUA_PATH`指定的的模块搜索路径，udbg还会从程序目录下的`script/`和`user-lua/`两个目录搜索lua模块

## udbg初始化脚本

udbg创建完lua虚拟机并完成基本的配置设置之后，会执行`user-lua/user-init.lua`文件，用户可以在这个文件里针对不同的调试目标做一些配置、实现自己常用的功能函数

调试环境相关的变量
* `__llua_os`            操作系统: windows android linux
* `__llua_arch`          CPU架构: x86_64 x86 arm aarch64

调试目标相关的变量
* `__target_path`        调试目标的程序路径
* `__target_pid`         调试目标的进程ID
* `__target_cmdline`     调试目标的命令行
* `__target_name`        调试目标的名称，默认是调试目标的文件名；这个变量比较特殊，udbg会以这个名字在`data/`目录下创建一个和调试目标相关的私有数据目录，用于记录调试过程中一些数据，比如命令行历史、flags等；用户可以在初始化脚本里根据详细的目标信息修改成不同的名字，以作区分

## __config 表

顾名思义，`__config`表里保存着调试器所使用的配置，可以在初始化脚本中针对不同的调试目标定制配置

目前有如下几个配置项：
* `ignore_initbp: bool` 是否忽略初始断点
* `ignore_exception: 'all' | table`
  - 这个值为 'all' 时，会忽略所有的异常
  - 为table时，如果table里异常code对应的value为true，则忽略，否则中断给用户
* `symbol_cache: str` Windows符号的缓存目录 (调试目标所在系统)

## __flags 表

`__flags`和`__config`的作用是一致的，区别在于`__flags`可以让用户通过**flags命令**动态改变，并存储到目标相关的数据里，下次调试同一个目标时，直接利用上次设置的配置

## init脚本配置实例

`user-lua/user-init.lua`

```lua
if __target_path:find('notepad.exe$') then
    log('--------- notepad.exe --------')
    __config.ignore_initbp = true
end
```

## 调试目标的数据目录

udbg第一次启动时，会在自己所在的目录创建data目录，data目录下又会各自以调试目标的名称`__target_name`创建一个目录，用来存放调试数据

这个目录的文件结构如下：
* `autorun/` 位于这个目录里的lua脚本发生改变时，udbg会自动执行
* `history.txt` 命令行历史记录
* `flags.lua` 如果使用 flags 命令修改了 `__flags` 变量，则会将 `__flags`表保存在这个脚本里，下次执行时，会自动恢复到 `__flags` 变量里

## 在init脚本中实现常用功能

### 在DLL入口处下断点

```lua
local module_load = on_module_load
function on_module_load(base, path)
    if path:find 'target.dll$' then
        add_bp(get_module(base).entry)
    end
    return module_load(base, path)
end
```

### 实现忽略指定异常

```lua
local k = require 'win.const'

-- 忽略指定异常(配置实现)
__config.ignore_exception = {
    [k.STATUS_CPP_EH_EXCEPTION] = true,
}

-- 忽略指定异常(代码实现)
local on_exception_o = on_exception
function on_exception(tid, code)
    if code == k.STATUS_CPP_EH_EXCEPTION then
        return false      -- 返回 false(nil) 忽略异常(即不中断)
    end
    return on_exception_o(tid, code)
end

-- 忽略所有异常
function on_exception(tid, code) return false end
```