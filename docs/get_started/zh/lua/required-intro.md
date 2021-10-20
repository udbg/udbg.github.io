
## Lua版本

udbg使用的是lua5.4版本，在原生lua的基础上，添加了多线程的支持、扩展了C库的函数，并且自带了一些第三方库
- [penlight](https://stevedonovan.github.io/Penlight/api/index.html) lua标准库扩展增强
- [glue](https://luapower.com/glue) lua标准库扩展增强
- [serpent](https://github.com/pkulchenko/serpent) lua对象序列化库 pretty
- [cjson](https://github.com/mpx/lua-cjson) [cmsgpack](https://github.com/antirez/lua-cmsgpack) [luaffi](https://github.com/facebookarchive/luaffifb) [luasocket](https://github.com/diegonehab/luasocket) [lfs](https://github.com/keplerproject/luafilesystem) [lqt](https://github.com/lqt5/lqt) [lsqlite3](http://lua.sqlite.org/)

## 客户端环境与调试器环境

udbg原生支持远程调试，从一开始的设计就是客户端UI与调试器分离，通过RPC通信交互数据

本地调试时，虽然客户端UI与调试器在同一进程中，但其实内部也是通过RPC来交互数据的

0.2版本之前，lua只用于调试器环境中，客户端UI由C++(Qt)实现，lua无法直接访问客户端中的任何数据；
从0.2版本开始，引入了lqt库，可以通过lua定制Qt的UI逻辑，客户端也会用lua来编写大量逻辑；

注意这里说的环境并不是lua中~~`_ENV`~~的概念，而是指进程(接口)环境
* 本地调试时，客户端与调试器在同一进程中，使用的也是同一个lua虚拟机，所以客户端的API和调试器的API都可使用；
* 远程调试时，客户端与调试器在不同进程中甚至不同机器中，使用的是各自进程内的lua虚拟机，所以在执行自己写的lua代码时需要注意其所处的环境，是否可使用对应的API

**如果不考虑远程调试的情况，就不用关心这些区别**

## 运行Lua的方式

1. `.exec`命令：执行`.exec <脚本路径>`命令
2. autorun保存自动运行：在script目录或者插件目录下的`autorun`目录里保存lua文件，udbg监控到写入时会自动执行
3. 作为命令运行：可以在udbg的`script/udbg/command`目录下添加自己的命令实现，然后在命令框中输入相应的命令名称来运行，参考[自定义命令](../quick-start/command.html)

*以上三种方式都是在调试器环境中运行lua脚本，在远程调试的情况下lua会运行在远程机器上*

## 启动流程/脚本加载流程

udbg客户端启动后，首先执行 `script/__client__.lua`：
- 执行用户配置文件 `udbg-config.lua`
- 启动RPC会话 `client:start_session(remote)`：remote参数如果指定了服务端IP和端口，则尝试与调试器服务端建立连接进行远程调试；否则，在客户端进程内启动一个新的线程运行调试器服务端，并建立进程内的RPC会话
  - 调试器服务端与客户端建立连接后，初始化调试器相关的API，然后通过RPC加载`script/__udbg__.lua`
  - `__udbg__.lua`会执行`udbg.core`初始化调试器核心逻辑，执行`udbg.ui`初始化和客户端UI相关的API
  - TODO 最终调用`ui.session:request('ui_info', device)`，告知客户端自己的设备ID，并获得在客户端的数据目录路径；
  - 完成上述步骤后，客户端的 `start_session()` 函数才会返回
- 根据配置文件(udbg-config.lua)里的`plugins`列表确定要加载的插件目录
  - 加载每个插件目录下的客户端初始化脚本 `udbg/client/__init.lua`
  - 加载每个插件目录下的调试器初始化脚本 `udbg/__init.lua`
  - 监听每个插件目录下的 `autorun/` 目录，此目录下的lua文件改动时会被自动执行(在调试器环境中)

`__client__.lua`执行完成之后，创建主窗口，然后执行`__client__.lua`里的`service.onUiInited()`函数:
- 执行 `udbg.client.qt` `udbg.client.ui` 两个模块，初始化客户端UI逻辑
- 触发 `clientUiInited` 事件
- 触发调试器端 `uiInited` 事件

最后，开始UI事件循环
