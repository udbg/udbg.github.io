
# 基本功能

## GUI功能简介

1. CPU窗口：反汇编 & 寄存器
  - 反汇编窗口中，双击跳转指令可跳到对应的地址，`F2`启用/禁用/删除断点；`Ctrl+G`可以输入地址表达式并跳转到对应地址
  - `Backspace` 上个地址 `Ctrl+Enter` 下个地址
  - 寄存器窗口中，双击寄存器数值可以修改对应的寄存器 ![](/static/image/2021-10-18-12-05-54.png)
2. 模块视图：显示已加载的模块，右侧显示该模块加载的符号 ![](/static/image/2021-10-20-23-32-36.png)
3. 断点视图
   ![](/static/image/2021-10-18-12-39-29.png)
   线程视图 ![](/static/image/2021-10-18-12-38-29.png)
   句柄视图 ![](/static/image/2021-10-18-12-40-08.png)
   内存布局视图 ![](/static/image/2021-10-18-12-40-45.png)
4. 内存视图：
  - `Ctrl+1` `Ctrl+2` `Ctrl+3` `Ctrl+4` 分别切换BYTE WORD DWORD QWORD显示
  `Ctrl+F`切换float显示
  `Ctrl+D`切换double显示
  `Ctrl+G`可以输入地址表达式并跳转到对应地址
  - 双击可以编辑内存 ![](/static/image/2021-10-18-12-04-18.png)
5. 日志&命令：日志视图展示调试日志、命令输出
  - 快捷键`Atl+.`定位到命令输入框，可以执行一些命令，调试命令参考`script/udbg/command`目录下的lua文件
  - 命令输入框内`Ctrl+R`键可以弹出历史记录并补全
  - 日志窗口内选中一个地址，右键菜单可以快速跳到 CPU/内存/内存页 ![](/static/image/2021-10-18-12-03-30.png)

## 远程调试

远程调试需要将`udbg-server.exe` `lua54.dll` `plugin/` 放到远程机器上，然后启动`udbg-server.exe`开始监听；server启动后会显示所监听的IP和端口，默认是*0.0.0.0:2333*，可以通过命令行修改

![](/static/image/2021-10-19-14-27-40.png)

客户端执行`udbg -r <IP:端口>`连接到`udbg-server`，连接成功后，会显示调试窗口，后续的命令/脚本都是在远程机器上运行的，包括attach窗口显示的进程也都是远程机器上的进程，**所有自定义lua脚本也都是运行在调试器端，这意味只需在远程机器上放几个核心文件，然后便可以在客户端机器上操纵一切！**

*`script/udbg/`目录下大部分lua文件都是在远程进程中运行的，只有`script/udbg/client/`目录下的是只能在客户端运行的*

# 扩展功能

## Windows内核

以管理员身份运行`udbg -A krnl -o 4`可以查看Windows内核

![](/static/image/udbg-open-krnl.gif)

*依赖驱动driver.dll，需要自己加签名*

## 进程信息查看

udbg支持非侵入式调试，通过`udbg -o <PID/ProcessName>`可打开正在运行的进程，但并不进行调试

在此模式下可以查看进程内存、反汇编、模块符号、线程、句柄等信息，可使用内存搜索系列的命令搜索内存

## VEH调试器

通过`udbg -A uspy`来指定使用VEH调试引擎

VEH调试引擎除了可以支持标准调试引擎的断点功能外，还可以在目标进程内执行lua脚本，调用通过lua的libffi调用目标进程内的函数，对目标进程任意函数进行Hook

TODO: uspy

# 插件功能

本文介绍udbg自带实现的插件功能

插件大部分功能都由lua实现，或者lua配合第三方功能模块实现

## 窗口列表

该插件其实是个命令[list-window](https://github.com/udbg/udbg/blob/master/udbg/command/list-window.lua)，由**lua+ffi**实现，也有相应的菜单入口

![](/static/image/2021-10-18-20-23-41.png)

## PE View

解析内存中PE镜像，目前只展示了一些基本数据，源码参考`script/udbg/client/__init.lua`中的`.pe-view`命令的实现

![](/static/image/2021-10-18-22-55-53.png) ![](/static/image/2021-10-18-22-56-16.png)

![](/static/image/2021-10-18-22-56-37.png)

![](/static/image/2021-10-18-22-57-08.png)

## Dump PE

将内存中的PE镜像转储到文件，便于后续的静态分析；该功能集成到了PE View界面中，支持扫描导入表，编辑导入表

![](/static/image/2021-10-18-22-57-40.png)

## PDB文件下载

从微软官网下载所需的pdb符号文件，该功能依赖**powershell的Invoke-WebRequest**命令

源码参考`script/udbg/client/__init.lua`中的`.download-pdb`命令的实现
- 可以在模块列表中右键选择`Download PDB`来下载某一个模块的pdb文件
- 执行`.download-pdb *`命令下载所有模块的pdb

>! 需要在[配置脚本](./config.html)中配置符号缓存路径

## Scan Patch

扫描模块内存修改（包括可执行段和导出表）

![](/static/image/scan-patch.gif)

## 内存搜索(yara)

目前是通过[yara](https://github.com/udbg/udbg/blob/master/udbg/command/yara.lua)命令来提供搜索功能，后续会实现一个GUI来配置搜索参数

    [cmd] >> yara -h
    yara                             -- Search Memory use Yara
        <file> (string)                 Yara file | Pattern string
        --start (optional number)       Start Address
        --stop  (optional number)       Stop Address
        --max  (optional number)        Max count
        -m, --module (optional string)  Search in this module
        -b, --binary                    Search binary pattern
        -a, --ascii                     Search string pattern
        -u, --utf16le
        -r, --rule                      file is a rule

    example:
        yara -a abcdefghi
        yara -b 'FF 25 ?? ?? ?? ??' -m ntdll

## KeyStone汇编

[asm命令](https://github.com/udbg/udbg/blob/master/udbg/command/asm.lua)封装了keystone汇编的功能

![](/static/image/2021-10-21-14-11-08.png)

    [cmd] >> asm -h
    asm                          --     assembly a instruction
        <insn>    (string)              the instruction
        <address> (optional address)    the address

        --arch    (optional string)
        --hex                       from hex string
        -w, --write                 assembly and write to address