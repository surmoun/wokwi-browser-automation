# Wokwi Monaco 编辑器输入拦截详解

Wokwi 的代码编辑器基于 Monaco，对"非受信任输入"有多层防护。自动化脚本写入时头部会被静默丢弃、内容会被重排，这是本项目最大的坑。

## 拦截机制

Monaco 通过 input 事件门控（trusted 检查）+ 内部文档模型（model）与 DOM 分离实现：
1. **trusted 事件检查**：合成事件 `isTrusted=false` 被忽略，页面内 `execCommand` 无用户手势也被 Chrome 拒绝。
2. **大文本截断**：`bu.type` 的底层实现（真实按键模拟）在长文本时只可靠投递尾部 ~160-500 字符，头部丢失且不报错。
3. **自动格式化**：`insertText` 分块插入会被 Monaco 的自动缩进/格式化逻辑重排，产生乱缩进或内容合并。

## 已排除方案（不要再试）

| 方案 | 结果 | 原因 |
|---|---|---|
| `bu.type` replace 大文本（>1KB） | 只保留尾部 | 事件投递截断 |
| 分块 `execCommand('insertText')` | 乱缩进/合并 | Monaco 自动格式化 |
| 合成 `ClipboardEvent` | 忽略 | isTrusted=false |
| 页内 `execCommand('copy')` + paste | Chrome 拒绝 | 无用户手势 |
| Library Manager 安装外部库 | 永远 Loading | 库索引域名超时 |

## 唯一可行路径

**"清空 → 写入"组合**：
1. 全选删除（`ctrl+a` + `Backspace`）把文档清成空；
2. 在空文档上 `bu.type(replace=True)` 纯键入。空文档时无需替换旧内容，Monaco 只会追加，规避截断叠加问题；
3. 写入后等 ~4s，截图确认。

实测：在空文档上写入 ~2KB 自包含代码，完整落盘 32 行（截图验证）。

## 验证陷阱

- `textarea.value` 被 Monaco 门控：读出的 0/21/163/500 是"窗口内有效输入"计数，不代表文档真实内容。**截图才是权威**。
- 文件 tab 之间内容可能错位（写入落进激活 tab）。写前必须截图确认激活文件。

## 会话同步

- bu 自动化会话与用户 App 内浏览器是**两个独立浏览器会话**，本地状态不同步：一边看到的内容不代表另一边。
- 用户接管（interaction.request_action, type="browserControl"）后，bu 需 `bu.resync()` 再重新观察。
