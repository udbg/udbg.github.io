
# 调试命令

udbg的调试命令都是由lua脚本实现，自带的命令可以参考`script/udbg/command/`目录下的lua脚本

## 常用命令

**以`.`开头的命令是客户端命令，运行在客户端环境中**

大部分命令都可通过加`-h`参数来查看使用方法

- 运行Lua脚本`.exec`
- 启动调试 `dbg`
- 汇编/反汇编 `asm` `dis`
- 进程/线程列表 `list-process` `lt`
- 窗口列表 `list-window`
- 内存查看 `d` `dw` `dd`
- 指针解析 `dp`
- 堆栈 `k`
- 断点管理 `bp` `bl` `bc`

## 自定义命令

有两种添加自定义命令的方式：
1. 通过lua脚本添加 `requier 'udbg.cmd'.register(name, func[, option])`
2. 直接在插件目录的`udbg/command`目录下编写命令名对应的lua脚本，参考`script/udbg/command`里的命令编写方式

TODO