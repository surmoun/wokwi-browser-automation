# Wokwi CLI 与 MCP Server（可选工具链）

用户已配置 **wokwi-cli** 及其内置的**实验性 MCP Server**。浏览器自动化（SKILL.md 主流程）是默认交付路径；本机已装 CLI 时，本地验证/测试优先走 CLI/MCP。

## 安装（Windows 本机）

```powershell
iwr https://wokwi.com/ci/install.ps1 -useb | iex
```

或从 Wokwi GitHub Releases 下载二进制，重命名为 `wokwi-cli.exe` 放入 PATH。
（ESP-IDF 工程另可 `pip install idf-wokwi`。）

## Token 认证

- 在 Wokwi CI Dashboard 生成 token：`wok_` 开头、共 44 字符。
- 设环境变量（PowerShell）：

```powershell
$env:WOKWI_CLI_TOKEN = "wok_..."
# 持久化：setx WOKWI_CLI_TOKEN "wok_..."
```

- token 是敏感信息：不外泄、不写入项目文件。

## 常用命令

| 命令 | 作用 |
|---|---|
| `wokwi-cli init` | 在项目根目录生成 `wokwi.toml` + `diagram.json` |
| `wokwi-cli <项目目录>` | 运行仿真并显示串口输出，30s 后自动退出 |
| `wokwi-cli lint` | 校验 diagram.json（未知器件/非法引脚连接等） |
| `wokwi-cli <dir> --timeout 5000 --expect-text "Hello"` | 断言串口出现指定文本（测试用） |
| `wokwi-cli <dir> --fail-text "Error"` | 串口出现指定文本即失败 |
| `wokwi-cli <dir> --scenario <file>.yaml` | 运行自动化场景（PlatformIO + Wokwi CLI） |
| `wokwi-cli <dir> --screenshot-part <part_id> --screenshot-time 4500 --screenshot-file shot.png` | 指定仿真时刻截取指定器件画面 |
| `wokwi-cli <dir> --serial-log-file out.log --timeout 10000` | 串口输出存文件 |
| `wokwi-cli <dir> --interactive` | 把 stdin 接到仿真串口 |
| `wokwi-cli <dir> -g 3333 --timeout 0` | 开启 GDB 调试端口（3333），仿真暂停等待断点 |

常用参数：`--elf`、`--diagram-file`、`--timeout`（仿真毫秒）、`--quiet`。

## MCP Server（实验性，CLI 内置）

启动方式：`wokwi-cli mcp`（stdio 传输）。

Agent 的 MCP 配置（在客户端配置中添加）：

```json
{
  "servers": {
    "Wokwi": {
      "type": "stdio",
      "command": "wokwi-cli",
      "args": ["mcp"],
      "env": {
        "WOKWI_CLI_TOKEN": "${input:wokwi-cli-token}"
      }
    }
  }
}
```

能力：
- 仿真复杂嵌入式系统
- 运行嵌入式项目的自动化测试
- 监控串口控制台输出与日志
- 实时与虚拟硬件交互

认证：需要有效的 `WOKWI_CLI_TOKEN`（见上）。

## 场景选择

| 目标 | 首选路径 |
|---|---|
| 交付"可运行/可分享链接"、用户要看仿真画面 | 浏览器自动化（网页端 SHARE） |
| 本地自动化验证：断言输出、失败判定、定时截图 | CLI（`--expect-text` / `--fail-text` / `--screenshot-*`） |
| Agent 直接调用仿真工具（监控串口、跑测试、操作虚拟硬件） | MCP（`wokwi-cli mcp`） |
| 交互式调试（GDB 断点） | CLI（`-g <port> --timeout 0`） |

## 注意事项

- MCP 为实验性特性，接口可能变化；调用失败时回退到 CLI 或浏览器路径。
- `wokwi-cli <dir>` 默认 30s 退出；长任务设 `--timeout 0`。
- 命令行需要真实固件（Arduino/PlatformIO/ESP-IDF 编译产物或 wokwi.toml 配置）；纯网页端项目可先用浏览器自动化。
