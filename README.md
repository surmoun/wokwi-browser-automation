# wokwi-browser-automation

通过**浏览器自动化**操作 [Wokwi](https://wokwi.com) 在线硬件仿真平台的 Agent Skill：创建/编辑项目、写入代码与电路图、启动仿真、验证外设显示、获取分享链接。核心价值：**规避 Wokwi Monaco 编辑器对自动化输入的拦截**，提供可靠的"清空-写入-截图验证"流程，并支持可选的 wokwi-cli / Wokwi MCP Server 工具链。

> 本文件面向"人"（安装指引）；`wokwi-browser-automation/SKILL.md` 面向"Agent"（执行入口）。安装时整个 `wokwi-browser-automation/` 目录一起复制即可。

## 功能

- 自动导航到 Wokwi 项目页 / 新建项目
- 逐个写入 `sketch.ino` / `diagram.json` / `libraries.txt`（Monaco 拦截规避：清空→写入→截图验证）
- 启动/重启仿真，验证运行态（计时走动、串口日志、外设画面）
- 获取并交付 SHARE 分享链接
- 可选工具链：本机 `wokwi-cli`（本地验证/断言/截图）与内置 MCP Server（`wokwi-cli mcp`）
- 自带 SSD1306 自包含驱动与走线美化模板（ESP32-C3 + OLED 项目开箱即用）

## 目录结构

```
wokwi-browser-automation/
├── SKILL.md                          # 执行入口（Agent 触发后加载）
└── references/
    ├── write-workflow.md             # 可复制的 computer_use_tool 代码（导航/清空/写入/启动/分享）
    ├── monaco-bypass.md              # Monaco 输入拦截机制 + 已排除方案表
    ├── self-contained-ssd1306.md     # SSD1306 自包含驱动完整代码 + 走线美化指南
    └── toolchain-cli-mcp.md          # wokwi-cli 命令表 + Wokwi MCP Server 配置
```

## 安装

### 方式 A：豆包 / 豆包办公（推荐）

1. 克隆本仓库，把 `wokwi-browser-automation/` 整个目录复制到你的 **Skills 目录**：
   - 豆包：`<工作区>\.doubao\agent_mode\workspace\.user_skills\`
   - 或 Cursor/其他 agent：复制到对应 `skills` / `.agents/skills` 目录
2. 重启/新开会话，Agent 自动发现该 Skill

### 方式 B：Anthropic Claude Code / Cursor

```bash
mkdir -p ~/.claude/skills
cp -r wokwi-browser-automation ~/.claude/skills/
```

### 前置依赖（必装）

- Agent 环境具备 **browser-use-automation** 能力（`computer_use_tool` plane=`bu` + `aha_browser_use`）；无此能力时 Skill 会退回 Wokwi 网页手动粘贴路径

### 可选依赖

- `wokwi-cli`（Windows 安装：`iwr https://wokwi.com/ci/install.ps1 -useb | iex`）+ `WOKWI_CLI_TOKEN`（Wokwi CI Dashboard 生成，`wok_` 开头 44 字符）——启用本地验证 / MCP 工具链

## 使用示例（Agent 触发语句）

- "帮我在浏览器里跑通一个 Wokwi 项目"
- "把这个代码写到 Wokwi 上并启动仿真"（附代码/链接）
- "操作 Wokwi 编辑器，跑一下 OLED 眨眼"
- "用 wokwi-cli 断言串口输出 Hello"（可选工具链）
- "用 Wokwi MCP 监控串口日志"（可选工具链）

## 关键经验（为什么需要这个 Skill）

Wokwi 的 Monaco 编辑器会拦截所有自动化输入：大文本只保留尾部、分块插入被重排、合成剪贴板事件被忽略。唯一可靠路径是**"全选清空 → 空文档 replace 写入 → 截图验证"**；且 `textarea.value` 被门控不可信，**截图才是权威**。详见 `references/monaco-bypass.md`。

## License

MIT
