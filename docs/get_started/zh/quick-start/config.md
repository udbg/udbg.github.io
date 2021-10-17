
# 配置文件

udbg的配置是通过lua的表来存储的，分客户端配置和调试器配置，参考[客户端环境与调试器环境](../lua/required-intro.html#客户端环境与调试器环境)

## 客户端配置

udbg会将自身目录(或其上层目录)的`udbg-config.lua`文件作为配置脚本

默认配置如下
```lua
---数据存放目录，默认是 `<UDBGDIR>/data`
data_dir = false

---指定其他插件/配置路径
--- plugins = {
---     {path = [[D:\Plugin1]]},
---     {path = [[D:\Plugin2]]},
--- }
plugins = {}

---指定启动编辑器的命令，默认是notepad
---如果安装了vscode，建议改为
--- edit_cmd = 'code %1'
edit_cmd = false

---远程地址映射, 配置了此项可在命令行中用键名代替对应的地址端口；
---比如配置了以下映射:
--- remote_map = {
---     ['local'] = '127.0.0.1:2333',
---     local1 = '127.0.0.1:2334',
---     local2 = '127.0.0.1:2335',
---     vmware = '192.168.1.239:2333',
--- }
---即可使用`udbg -r vmware C:\Windows\notepad.exe`进行远程调试
remote_map = {}

---是否预编译调试器端require的lua
precompile = true

---传递给调试器的配置
udbg_config = {
    ---PDB符号缓存目录
    -- symbol_cache = [[D:\Symbols]],
    ---是否忽略初始断点
    ignore_initbp = false,
    ---是否忽略所有异常
    ignore_all_exception = false,
    ---线程创建时暂停
    pause_thread_create = false,
    ---线程退出时暂停
    pause_thread_exit = false,
    ---模块加载时暂停
    pause_module_load = false,
    ---模块卸载时暂停
    pause_module_unload = false,
    ---进程创建时暂停
    pause_process_create = false,
    ---进程退出时暂停
    pause_process_exit = false,
    ---进程入口断点
    bp_process_entry = false,
    ---模块入口断点
    bp_module_entry = false,
}
```

### 界面主题

界面主题可以通过修改主窗口的[qss](https://doc.qt.io/qt-5/stylesheet.html)来实现，参考`script/udbg/client/ui.lua`中的用法

默认是
```css
QAbstractScrollArea {
    background-color: rgb(255, 248, 240);
    font: 8pt "Lucida Console";
}

QAbstractScrollArea:focus {
    border: 1px solid #0078d7;
}
```

可以在配置文件(udbg-config.lua)中加入以下代码来修改
```lua
uevent.on.clientUiInited = function()
    g_client.qt.gui(function()
        -- 自定义的qss主题
        g_client.ui.main:setStyleSheet [[
            QAbstractScrollArea {
                background-color: white;
            }
        ]]
    end)
end
```

## 调试器配置

调试器端的配置存储在`__config`这个全局表里，可以通过客户端配置的`udbg_config`来静态配置，也可以在`udbg/__init.lua`脚本里来做一些动态配置，参考[插件编写](../lua/write-plugin.html)