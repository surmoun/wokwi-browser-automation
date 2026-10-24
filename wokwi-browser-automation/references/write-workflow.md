# Wokwi 写入工作流（可复制代码）

以下代码片段可直接粘贴进 `computer_use_tool(plane="bu")` 的 code 参数运行。
完整流程：导航 → 点击 tab → 清空 → 写入 → 截图验证 → 启动 → 分享。

## 1. 导航到项目

```python
import aha_browser_use as bu
import time

bu.navigate("https://wokwi.com/projects/478729910076314625")  # 已知项目 ID
bu.wait_for_load(timeout=15)
print(bu.snapshot(within="viewport", view="interactive", max_chars=700))
```

## 2. 清空当前激活文件（对每个文件先做这步）

```python
# 聚焦 textarea（Monaco 的隐藏输入框）并移入可视区
bu.js("""(() => {
  const ta = document.querySelector('textarea');
  if (!ta) return 'no-ta';
  ta.focus();
  ta.style.position='fixed'; ta.style.left='350px'; ta.style.top='300px';
  ta.style.width='200px'; ta.style.height='50px'; ta.style.opacity='1'; ta.style.zIndex='99999';
  return 'ok';
})()""")
time.sleep(0.3)
bu.hotkey("ctrl", "a")
time.sleep(0.5)
bu.press_key("Backspace")
time.sleep(1.5)
```

清空后**必须截图确认**（编辑器应只剩行号 1）：

```python
f = bu.screenshot(with_cursor=False)
print(f.path)  # Read 该路径查看
```

## 3. 确认激活的是目标文件

点击 tab 后截图，按首行特征判断：
- sketch.ino → `#include <...>`
- diagram.json → `{`
- libraries.txt → `#`

```python
import re
snap = bu.snapshot(within="viewport", view="interactive", max_chars=600)
m = re.search(r'(n\d+) button "sketch\.ino"', str(snap))
if m:
    bu.click(m.group(1)); time.sleep(1.2)
```

## 4. 写入内容（replace 到已清空的文档）

```python
bu.type(bu.css("textarea", match="first"), <完整内容>, submit=False, replace=True)
time.sleep(4.0)  # 等待 Monaco 落盘
f = bu.screenshot(with_cursor=False)
print(f.path)  # 截图验证：内容应完整显示
```

注意：内容超过 ~1KB 时，**必须先清空再 replace**（在非空文档上 replace 会截断）。写入后以截图为准。

## 5. 启动仿真

```python
snap = bu.snapshot(within="viewport", view="interactive", max_chars=700)
m = re.search(r'(n\d+) button "Start the simulation"', str(snap))
if m:
    bu.click(m.group(1)); time.sleep(8.0)  # 编译需要时间
snap2 = bu.snapshot(within="viewport", view="interactive", max_chars=1200)
print(str(snap2)[:1200])
```

运行成功标志：右上计时走动、串口日志 `entry 0x403ce000`、外设可见。

## 6. SAVE + SHARE

```python
snap = bu.snapshot(within="viewport", view="interactive", max_chars=600)
m = re.search(r'(n\d+) button "SAVE"', str(snap))
if m:
    bu.click(m.group(1)); time.sleep(2.0)
snap2 = bu.snapshot(within="viewport", view="interactive", max_chars=600)
m2 = re.search(r'(n\d+) button "SHARE"', str(snap2))
if m2:
    bu.click(m2.group(1)); time.sleep(2.0)
# 弹窗 "Share Project" 里 textbox 的值就是分享链接
txt = str(bu.js("(() => { const e = document.querySelector('dialog input[type=text], [role=dialog] input[type=text]'); return e ? e.value : ''; })()"))
print("LINK:", txt)
```

## 7. 通用提示

- 每个文件写入前后各截一张图，形成"前后对照"证据链。
- 若发现文件内容错位（sketch 混入 JSON 等），回到步骤 2/3 对该文件单独清空重写。
- bu 会话状态可能与用户浏览器不一致，交付链接后提醒用户以自己浏览器为准。
