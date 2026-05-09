---
name: ds-expert
description: 用户消息以"ds/DS"开头时，通过 Edge CDP 进入 DeepSeek 专家模式搜索，提取答案并保留原始格式返回。
tags:
  - search
  - deepseek
  - expert-mode
  - web-browser
triggers:
  - ds 开头的问题
  - DS 开头的问题
---

# DeepSeek 专家模式搜索

当用户消息以 `ds` 开头时，自动使用 DeepSeek 官网的专家模式搜索答案，提取回复原文并**保持原始格式**返回。

## 触发条件

用户消息开头为 `ds` 或 `DS`（不区分大小写），去掉前缀后作为搜索问题。

## 前置条件

- **Edge 浏览器持久运行**：CDP 端口 **9222**（`edge-browser.service`），profile 在 `/root/.edge-profile`
- **Hermes 已配置**：`browser.cdp_url: ws://127.0.0.1:9222`
- **DeepSeek 已登录**：用户通过 NoVNC 登录一次后，Cookie 持久化

## 标签页管理（每次操作前执行）

每次执行前先检查标签页数量，避免浏览器标签页堆积：

```python
# 1. 获取当前所有标签页
tabs = browser_cdp(method="Target.getTargets")
targets = tabs.get("targetInfos", [])
tab_count = len(targets)

# 2. 如果标签页 > 10，全部关闭并重启浏览器
if tab_count > 10:
    for t in targets:
        browser_cdp(method="Target.closeTarget", target_id=t["targetId"])
    terminal(command="systemctl restart edge-browser.service")
    time.sleep(5)  # 等待浏览器重启

# 3. 创建一个新标签页用于本次操作
new_tab = browser_cdp(method="Target.createTarget", params={"url": "about:blank"})
new_target_id = new_tab["targetId"]
```

> ⚠️ 每次重新导航前都要检查 tab 数量。**不要复用旧的对话页面**，创建新标签页避免干扰历史对话。任务完成后关闭自己创建的标签页。

## 工作流程

### 步骤1：提取问题
去掉 `ds` 前缀，获取纯净问题文本。

### 步骤2：导航到 DeepSeek
```python
browser_navigate(url="https://chat.deepseek.com/")
```
先开启一个新对话（点击"开启新对话"按钮）。

### 步骤3：切换到专家模式
在 radiogroup 中找到"专家模式"并选中：
```python
browser_click(ref="e12")  # 专家模式 radio button
```

### 步骤4：输入问题并发送
```python
browser_type(ref="e17", text="<问题>")  # 输入框
browser_press(key="Enter")
```

### 步骤5：轮询等待回复完成
多次检查页面，直到 DeepSeek 生成完整回复。检测标志：
- 最初显示"正在思考"
- 搜索完成后出现"深度思考"或已思考完成的提示
- 回复正文开始出现

规律轮询（约每3-5秒检查一次），直到检测到包含答案关键字的文本出现。

### 步骤6：提取答案（关键！）
使用 JS 从 `.ds-markdown` 元素中提取答案**正文**（排除思考过程、侧边栏等页面元素）：

**提取策略**：找到包含答案关键字的第一个 `.ds-markdown` 元素作为起点，收集后续所有同级元素，将 HTML 转换为 Markdown（保留加粗、表格、列表格式）。参考 `references/js-extraction.md`。

### 步骤7：返回结果
直接将 DeepSeek 的答案正文返回给用户：
- **不要截取**、不要概括、不要自己总结——用户明确要求"一字不漏"
- **不要包含**思考过程、侧边栏、搜索到的网页列表、页面元素
- **保留格式**：加粗（`**text**`）、表格（`|` 分隔）、列表、代码块

## 陷阱与注意事项（必读）

1. **只提取答案正文，不要页面元素**：browser_console 获取 `document.body.innerText` 会包含侧边栏对话历史。要用 JS 精确筛选 `.ds-markdown` 元素。
2. **保留原始格式**：用户要求"和网页答案的格式一致，加粗、表格等等"。HTML→Markdown 转换时必须保留 `**加粗**`、表格分隔符、列表标记。
3. **不要开多标签页**：每次 ds 查询只用一个标签页。不要重复打开新标签页。如果已有对话页面，直接复用。
4. **专家模式可能重置**：新对话时专家模式可能被重置为快速模式，每次需要重新点击。
5. **清理引用标记**：DeepSeek 回复中的 `-1`、`-3`、`-37` 等引用标记需要去除，只保留正文。
6. **检测是否已登录**：页面显示用户名（如"彭伟"）表示已登录；如果显示登录页面，需要通知用户通过 VNC 重新登录。
7. **Edge 可能挂掉**：如果 CDP 连不上（9222 端口无响应），先检查 `systemctl status edge-browser.service` 并重启。

## 参考文件

- `references/js-extraction.md` — JS 提取与 HTML→Markdown 转换的完整代码

## 优先级说明

本技能属于前缀触发（`ds`/`DS`），优先级高于通用搜索技能 `web-access`。
用户消息以 `ds` 开头时直接走本技能，不走 `web-access` 的三层通道调度。

## 依赖

- `browser` 工具集（CDP 模式）
- `edge-browser.service`（systemctl start edge-browser.service）
- DeepSeek 登录状态有效
