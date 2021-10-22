
# Tutorial

udbg write a large number of logic code in Lua, and provides rich API to customize, like frida provides a series of JavaScript API, but udbg provides GUI.

To use udbg effectively, cannot without Lua script, so let's give priority to introduce it.

## Run script

Assume you've downloaded udbg, and add it to `PATH` environment variable, you can look the version by command `udbg --version`, then we can start

Make a new directory, such as `udbg-lua`, store the following script to be execute in udbg

    mkdir udbg-lua
    cd udbg-lua

Let's start with a simple example: get the modules loaded in target. Create a file, such as start.lua, write
```lua
uevent.on('targetInitBp', function()
    for m in udbg.target:enum_module() do
        log('[module]', hex(m.base), m.name)
    end
end)
```

Execute `udbg -e start.lua notepad`, the log will show in the log view in udbg window. `uevent.on('targetInitBp', ...)` represent that execute the callback after the event `targetInitBp`, in the callback we enum the module and print it. `udbg.target` represent the target being debugged, it is nil if there is no target.

Close udbg, let's do another example about breakpoint: breakpoint in CreateFileW, and print the first argument
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

Execute `udbg -e start.lua notepad`, the notepad window should popup directly, then you click the menu 'File' -> 'Open', will trigger the execution of CreateFileW, then you can look the log

`reg` variable represent the register values on breakpoint/exception triggered, and you can get the value by `reg.<register name>`, `reg[1]` wrapper the logic to get function argument, its equal to `reg.rcx` in x64; method `read_wstring` read the utf16 string with the pointer passed to it, and then convert to utf8 as the return value

Notice it return 'run' in the callback of targetInitBp this time, it represent that udbg will continue run after the initial breakpoint, dont pause to request handle from user

Both of the previous two examples are execute script by shell argument, in the real analysis, we frequently need to execute lua in real-time, you can exeucte `.exec <script-path>` in udbg command, and you can pass the `-w` argument, let udbg automatically execute the script after write

Continue from the above example, if you want to change the callback of breakpoint after udbg startup, you can create a file, such as 'auto.lua', then execute `.exec -w auto.lua` in udbg command, and then copy the following code to auto.lua and store
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
In this example, udbg will pause to request handle from user, Yes, `return true` in breakpoint callback represent to handle with user

The last, let's do a complex example: print the arguments of CreateFileW and its returned value. Copy the following code to auto.lua and store
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
click the menu 'File' -> 'Open' to trigger the execution to CreateFileW, then you should be able to see the log

More examples refer to [Script example](../lua/script-example.html), API refer to https://github.com/udbg/luadoc/blob/master/_udbg.lua

## Start target

Scripts are often used for customized analysis, there is some simple methods to start a normal debugging task

1. Double click the udbg.exe to start directly, and attach to target by File->Attach menu
2. Pass arguments to udbg in command line:
  - `udbg` Start udbg, and show main window
  - `udbg -W` Start udbg without main window, debug in command line
  - `udbg -e test.lua` Start udbg and execute test.lua
  - `udbg notepad.exe` Start udbg and debug notepad.exe with default adaptor
  - `udbg notepad.exe -- test.txt test2.txt` Start udbg and debug notepad.exe, and pass the shell arguments `test.txt test2.txt`
  - `udbg -a notepad.exe` Start udbg and attach to notepad.exe
  - `udbg -A uspy notepad.exe` By adaptor `uspy`, create and debug notepad.exe
  - `udbg -A uspy -a notepad.exe` 通过uspy调试引擎 附加到 notepad.exe
  - `udbg -o explorer.exe` Open the process explorer.exe, dont debug
  - `udbg -r localhost:2333` Connect to udbg-server, and start udbg, you can pass any args listed in in above to start target in remote machine, refer to [Remote debugging](#Remote-debugging)
3. Execute `dbg` in udbg command, arguments like the udbg command line, show detail by `dbg -h`

The full command line arguments of udbg

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

udbg provides basic UI as well, the folloing text is about UI

If you start udbg without argument, it will show the main window defaultly, and worked in GUI mode; execute `udbg -W` will start udbg and work in CUI mode, window was created actually, but not show

You can toggle the tow mode by shortcut `Alt+;`

- In GUI mode, you can view various debugging data conveniently, and you can execute debugging command, it usually works in GUI mode
- The CUI mode, used for command line interaction, or view the output of script execution, for example, edit lua in vscode, and you can view the log in below terminal window

The `log` function will output to terminal in CUI mode, and output to log view in the main window in GUI mode.

>! Notice: CUI mode will block the output of `log` series function

### Shortcuts

CUI & GUI
- `F7` Step in
- `F8` Step out
- `F9` Conntinue (exception not handle)
- `Shift+F9` Continue (exception handled)
- `Ctrl+F9` Run to return by step

GUI：
- `F12` Break the target

CUI:
- `Ctrl+D` Break the target

### Switch between CUI and GUI

- `Alt+;` Switch one mode between CUI and GUI, if you switch to CUI, the main window will be hided
- `Alt+/` Switch to CUI window, without hiding main window

## Breakpoint

The easiest way to add breakpoint is that press `F2` in the disasm view.

The conditional breakpoint and log breakpoint you can add by `bp` command, to handle the more complex breakpoint you need to write lua script.
- `bp CreateFileW` add breakpoint on `CreateFileW`
- `bp CreateFileW wstr(reg[1])` add breakpoint on `CreateFileW`, and log the first argument
- Conditional breakpoint `bp CreateFileW wstr(reg[1]) -c v1:find'log.txt'`
- Hardware breakpoint `bp CreateFileW -t e1` Temporary breakpoint `bp CreateFileW --temp`

## Remote debugging