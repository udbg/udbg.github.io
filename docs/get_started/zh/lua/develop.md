
# 开发环境

推荐开发环境:
- 编辑器: vscode https://code.visualstudio.com/
- 代码补全(LSP)插件: `sumneko.lua` https://github.com/sumneko/lua-language-server
- Lua调试插件: `stuartwang.luapanda` https://github.com/Tencent/LuaPanda/

## 开发配置

安装好`sumneko.lua`插件后，编辑vscode配置，添加udbg相关的lua库目录，即可使用API自动补全、查看文档、函数跳转等功能

```json
{
    // ...
    "Lua.workspace.library": [
        // udbg的lua实现目录
        "<UDBG-DIR>/script",
        // Native接口注解文档 https://github.com/udbg/luadoc
        "<UDBG-DIR>/luadoc",
    ],
    // ...
}
```

## 调试配置

安装好`stuartwang.luapanda`插件后，编辑调试配置(launch.json)，添加如下配置

```json
{
    "version": "0.2.0",
    "configurations": [
      {
          "type": "lua",
          "request": "launch",
          "tag": "normal",
          "name": "LuaPanda",
          "description": "通用模式,通常调试项目请选择此模式 | launchVer:3.2.0",
          "cwd": "${workspaceFolder}",
          "luaFileExtension": "",
          "connectionPort": 8818,
          "stopOnEntry": false,
          "useCHook": true,
          "autoPathMode": false
      },
    // ...
    ]
```

调试时在vscode调试面板中选择 LuaPanda，开始调试按钮，vscode等待连接，然后udbg命令框中执行`require 'udbg.luadebug'.init_luapanda("127.0.0.1",8818)`即可连接到vscode里的LuaPanda调试器进行调试，注意 8818这个端口号 要和配置中 connectionPort 指定的一致