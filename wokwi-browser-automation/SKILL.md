---
name: wokwi-browser-automation
description: 通过浏览器自动操作 Wokwi（wokwi.com）在线硬件仿真平台：导航到项目页、写入 sketch.ino / diagram.json / libraries.txt、启动与重启仿真、验证外设显示效果、获取分享链接。当用户要求"在浏览器里跑通 Wokwi 项目"、"操作 Wokwi 编辑器"、"把代码写入 Wokwi 并启动仿真"，或给出 wokwi.com/projects 项目链接要求自动操作时使用。核心价值：规避 Wokwi Monaco 编辑器对自动化输入的拦截，提供可靠的"清空-写入-截图验证"流程，以及绕开外部库安装超时的自包含代码策略。若本机已配置 wokwi-cli / Wokwi MCP Server（可选工具链），本地验证与测试优先走 CLI/MCP。
---

# Wokwi 浏览器自动化

在 Wokwi 在线仿真平台上自动创建/编辑/运行项目，交付可分享的仿真链接。

## 总流程

1. 导航到项目页（已知 ID：`https://wokwi.com/projects/<id>`；新建：`https://wokwi.com/projects/new/<board>`，如 `esp32-c3`）
2. 逐个写入文件（**写入是本 Skill 的核心难点**，见下）
3. 启动仿真，验证编译通过并处于运行态
4. SAVE 保存 → SHARE 获取分享链接并交付

## 环境

- 用 `computer_use_tool(plane="bu")` + `import aha_browser_use as bu` 操作页面；动手前先 Read `browser-use-automation` Skill 与其 observation-api。
- 页面是 Wokwi 时，最终权威证据永远是**截图**（见"验证原则"）。

## 可选工具链：Wokwi CLI / MCP（本机已配置）

本机已安装 wokwi-cli 并配置 `WOKWI_CLI_TOKEN`（其内置实验性 MCP Server：`wokwi-cli mcp`）。使用原则：

- 交付"可运行/分享链接"或需要用户看仿真画面 → 浏览器自动化（本 Skill 主流程）
- 本地自动化验证（断言串口输出、定时截图、跑 scenario）→ CLI
- Agent 直接调用仿真工具（监控串口、跑测试、操作虚拟硬件）→ MCP
- 交互式 GDB 调试 → CLI `-g <port> --timeout 0`

完整命令表、MCP 配置 JSON、token 说明见 `references/toolchain-cli-mcp.md`。

## 写入文件（核心难点）

### Wokwi 的 Monaco 编辑器会拦截一切自动化输入

| 尝试 | 结果 |
|---|---|
| `bu.type` 一次性写入大文本（>1KB） | 只保留尾部 ~160-500 字符，头部丢失 |
| 分块 `execCommand('insertText')` | 被 Monaco 自动重排成乱缩进 |
| 合成 `ClipboardEvent`（isTrusted=false） | 直接忽略 |
| 页内 `execCommand('copy')` 再粘贴 | 无用户手势被 Chrome 拒绝 |

### 唯一可靠写入路径（对每个文件执行）

1. 点击目标文件 tab（sketch.ino / diagram.json / libraries.txt）
2. **截图确认激活的就是该文件**——每个文件首行特征不同：
   - sketch.ino → `#include <...>`
   - diagram.json → `{`
   - libraries.txt → `#` 注释
3. 用 `bu.js` 聚焦 textarea 并把 textarea 移到可见区域（便于 Monaco 接收真实输入）
4. `bu.hotkey("ctrl","a")` 全选 → `bu.press_key("Backspace")` 清空
5. **截图确认文档已清空**（编辑器只剩行号 1）
6. `bu.type(css("textarea", match="first"), <内容>, submit=False, replace=True)` 写入
7. **截图验证写入结果**（不要信 textarea.value）

### 文件身份混淆

- `bu.type` 会落进**当前激活的标签页**；若之前停留在别的 tab，写入会进错文件。
- 症状：sketch.ino 尾部混入 diagram JSON、diagram.json 变成 sketch 内容、文件内容互相拼接。
- 修复：对该文件单独执行"清空→重写→截图验证"，不要盲目重试。

### 库依赖策略

- Wokwi 库索引域名可能连接超时 → Library Manager 永远 "Loading..."，装不了外部库。
- **代码必须自包含**：只 `#include` 内置库（如 `Wire.h`），不要用 Adafruit GFX/SSD1306 等外部库。
- `libraries.txt` 残留的外部库条目不影响自包含代码编译，但保险起见可清空，避免编译时触发库下载。
- OLED 项目可直接用 `references/self-contained-ssd1306.md` 里的自包含驱动。

## 启动仿真与验证

1. 点击 `Start the simulation` 按钮。
2. 编译失败会弹 `Build failed!` 对话框（用截图 OCR 读错误信息，按错误修代码）。
3. **运行成功标志**：
   - 右上角仿真计时器持续走动（如 `05:39.878`）
   - 右下串口日志出现 `rst:0x1 (POWERON) → boot:0xc → load:... → entry 0x403ce000`
   - 右侧仿真面板的外设可见（如 OLED 屏渲染出图案）
4. 每次改代码后仿真可能自动重启，用截图确认新固件生效。

## 分享链接

1. 点击 `SAVE` 保存项目。
2. 点击 `SHARE` → 弹窗 "Share Project" 里的链接即分享链接，格式 `https://wokwi.com/projects/<id>`。
3. 直接交付该链接；用户浏览器中该 URL 可直接打开并运行仿真。

## 验证原则（血泪教训）

- **截图是唯一权威**。`textarea.value` 被 Monaco 门控，读出的 0/21/163/500 等数值不代表真实内容；编辑器截图显示完整代码才是真。
- bu 自动化会话与用户 App 内浏览器是**两个独立会话，状态不同步**；最终验收以用户可见页面为准。
- 修改文件后必须截图确认，不要相信"操作成功"的返回码。

## Resources

- `references/write-workflow.md`：完整可复制的 computer_use_tool 代码（导航/清空/写入/验证/启动/分享全流程）
- `references/monaco-bypass.md`：Monaco 拦截机制详解与已排除方案表
- `references/self-contained-ssd1306.md`：SSD1306 自包含驱动完整代码（OLED 项目免装外部库）
- `references/toolchain-cli-mcp.md`：Wokwi CLI 命令表 + MCP Server 配置（本机可选工具链）
