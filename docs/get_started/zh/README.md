
# UDBG介绍

基于Lua的二进制调试/分析工具

* 支持更多的调试/分析场景
  - Windows
    * 标准API调试器、VEH调试器、[内核信息查看](quick-start/basic.html#Windows内核)
    * [ ] WinDbg 调试引擎：可用于分析dmp、内核调试
  - 非侵入式调试、进程信息查看
  - [ ] 跨平台：支持linux、android
* 提供丰富的脚本接口，方便编写扩展、定制化分析
* 界面/核心分离，可进行[远程分析/调试](quick-start/basic.html#远程调试)：只需在远程机器上放几个核心文件，然后便可以在客户端机器上操纵一切
* 功能抽象、分类分层，跨平台：既保留各平台的共性，降低学习成本，也允许差异化定制，实现平台特定的功能

整体上有三大功能
- **目标(进程)查看**，类似于WinDbg中非侵入式调试的概念，可以实现CE中的一部分功能，此模式下主要包含 `内存读写` `枚举模块/线程/内存页/句柄` `挂起/唤醒目标` `内存搜索` `线程堆栈回溯` 等功能
- **目标(进程)调试**，此模式下包含了**目标(进程)查看**的功能，还有`断点管理: 添加/删除断点，启用/禁用断点` `调试事件循环` [断点Lua脚本Hook](./quick-start/basic.md#断点与Hook) 等功能
- **动态Hook/调用**，注入进程，动态Hook，动态调用目标中的函数，类似于Frida
  - Lua动态Hook目标内任意函数 InlineHook、IATHook
  - Lua调用目标内任意函数

*udbg的调试**目标**概念，最常见的是指进程，借助其他调试引擎的扩展后也可能是`Dmp文件` `内核空间`*

## 下载安装

从release页面下载最新的压缩包，并解压到新的目录

* https://github.com/udbg/udbg/releases
* https://gitee.com/udbg/udbg/releases

>! 注意路径中不要包含非ANSI字符

## 目前状态

界面：目前的GUI只是处于刚好够用的状态，和x64dbg和CE等流行的分析工具相比还差很多；udbg的首要目标是**提供更多的分析功能以及脚本控制能力**，GUI的完善将作为次要目标

功能：
- 基于Windows API实现的调试功能自测正常，无明显问题
- VEH实现的调试功能在某些极限情况下会出现异常
- 基于Windbg的dbgeng引擎实现的调试功能基本可用，还在内部完善中
- 内核信息查看，可以查看内核中的反汇编、内存、模块符号、线程等信息
- Linux/Android支持，待适配中

## 联系

- 邮箱：metaworm@outlook.com
- Chat：[Discord](https://discord.gg/emqz592zfT) [Telegram](https://t.me/joinchat/Uiww_6QfRKA1NGY1) [FEISHU](https://applink.feishu.cn/client/chat/chatter/add_by_link?link_token=31as7df3-4ddb-4160-aab0-0ab00c45cd24)

<!-- 
## 致谢

TODO -->