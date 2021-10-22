
# Introduce

Dynamic binary analysis tools based on Lua

* Support more analysis scene
  - Windows
    * Starndard API, VEH Debugger and view kernel space
    * [ ] WinDbg Engine: analyze dmp file, kernel debugging
  - Non-invasive attach
  - [ ] Cross-platform：support linux、android
* Providing rich script API, easy to extend and customize
* Separate the debugger from UI, support remote debugging: only put a few core files into remote machine, and you can control everything

## Debug Adaptor

`Debug Adaptor` is the abstraction of debugging method, udbg provides various functions and debugging for different kind of target by different adaptors. *Target in udbg, commonly is process, and can be `.dmp file` or `kernel space` with other adaptor*
- Adaptor only implement the core functions such as memory read/write, enum module/memory, then user can use the various functions provided by udbg and plugins, such as disassembly, memory search, PE View/Dump, etc.
- You can implement your adaptor by the [Rust API](https://github.com/udbg/udbg-base)

Overall, there are three main functions in udbg
- **Target (process) view** like the concept 'Non-invasive attach' in windbg, this mode mainly contains functions: memory read/write, enum module/thread/memory page/handle, suspend/resume target, stack backtrace
- **Target (process) debugging** this mode contains functions in **Target (process) view**, and you can debug it, using breakpoint
- **Dynamic hook and function call** inject to target process, dynamic hook and call any function use lua in target process, like frida

udbg implements follow adaptors (on Windows)
- `default` Debugger implemented by Windows API
- `uspy` Debugger implemented by the VEH mechanism on Windows. In addition to debug, you can hook any functions in target process by Lua, or call any functions use lua ffi dynamically through it
- `krnl` Adaptor implemented by Windows driver, you can view the memory space of process or kernel through it, but it have not debugging function

## Download

Download the lastest zip from release page, and unzip to a new directory

* https://github.com/udbg/udbg/releases
* https://gitee.com/udbg/udbg/releases

>! Notice: DONT inclucde Non-ANSI character in the path

## Current state


## Contact

- Email: metaworm@outlook.com
- Chat：[Discord](https://discord.gg/emqz592zfT) [Telegram](https://t.me/joinchat/Uiww_6QfRKA1NGY1) [FEISHU](https://applink.feishu.cn/client/chat/chatter/add_by_link?link_token=31as7df3-4ddb-4160-aab0-0ab00c45cd24)