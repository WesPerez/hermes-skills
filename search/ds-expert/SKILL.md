---
name: ds-expert
description: 用户消息以"ds/DS"开头时，通过 Edge CDP 进入 DeepSeek 专家模式搜索，提取答案并保留原始格式返回。
tags:
  - search
  - deepseek
  - expert-mode
  - web-browser
  - cdp
triggers:
  - ds 开头的问题
  - DS 开头的问题
---

# DeepSeek 专家模式搜索

当用户消息以 `ds` 开头时，**必须**立即执行本技能，使用 DeepSeek 官网专家模式搜索答案。**即使问题看似与 ds 本身相关（如元问题），也必须去 DeepSeek 查询，不得自行回答。**

## 触发条件

用户消息开头为 `ds` 或 `DS`（不区分大小写），去掉前缀后作为搜索问题。

## 前置条件

- **Edge 浏览器持久运行**：CDP 端口 **9222**（`edge-browser.service`），profile 在 `/root/.edge-profile`
- **Hermes 已配置**：`browser.cdp_url: ws://127.0.0.1:9222`
- **DeepSeek 已登录**：用户通过 NoVNC 登录一次后，Cookie 持久化

## 标签页管理（每次操作前执行）

每次执行前先检查标签页数量，避免浏览器标签页堆积：

```python
tabs = browser_cdp(method="Target.getTargets")
targets = tabs.get("targetInfos", [])
tab_count = len(targets)

# 如果标签页 > 10，全部关闭并重启浏览器
if tab_count > 10:
    for t in targets:
        browser_cdp(method="Target.closeTarget", params={"targetId": t["targetId"]})
    terminal(command="systemctl restart edge-browser.service")
    time.sleep(5)
```

### 创建新标签页

每次创建新标签页，**直接在 Target.createTarget 中指定 URL**，不要事后调用 browser_navigate：

```python
result = browser_cdp(method="Target.createTarget", params={"url": "https://chat.deepseek.com/"})
ds_tab_id = result["targetId"]
# 等待页面加载
terminal(command="sleep 5")
```

> ⚠️ **不要**创建 about:blank 后再 browser_navigate，因为 browser_navigate 不受 target_id 控制，会错误地导航到旧标签页。
>
> 后续所有操作必须使用 `browser_cdp(method="Runtime.evaluate", ..., target_id=ds_tab_id)`，**不要使用** browser_click / browser_type / browser_press / browser_navigate / browser_snapshot。

任务完成后关闭标签页：
```python
browser_cdp(method="Target.closeTarget", params={"targetId": ds_tab_id})
```

## 工作流程（全 CDP 方式）

所有操作使用 `browser_cdp(method="Runtime.evaluate", ..., target_id=ds_tab_id)` 执行 JavaScript。

### 步骤1：提取问题

去掉 `ds` 前缀，获取纯净问题文本。

### 步骤2：等待页面加载并切专家模式

创建标签页后，轮询检测 textarea 就绪（每 0.5s 检查，最多 6s）：

```python
for i in range(12):
    r = browser_cdp(method="Runtime.evaluate", params={"expression": "!!document.querySelector('textarea')", "returnByValue": True}, target_id=ds_tab_id)
    if r["result"]["result"]["value"]: break
    terminal(command="sleep 0.5")
```

**跳过「开启新对话」**：新标签页直接就是空白页，无需点击。

切换到专家模式：

```python
browser_cdp(method="Runtime.evaluate", params={
    "expression": "(()=>{const all=document.querySelectorAll('*');for(const el of all){if(el.textContent==='专家模式'){el.click();return 1}}return 0})()",
    "returnByValue": True
}, target_id=ds_tab_id)
```

### 步骤3：输入问题（nativeSetter + dispatchEvent）

React 受控组件，必须用 native value setter + dispatchEvent('input')：

```python
q = "你的问题文本"
browser_cdp(method="Runtime.evaluate", params={
    "expression": f"(()=>{{const ta=document.querySelector('textarea');const ns=Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype,'value').set;ns.call(ta,`{q}`);ta.dispatchEvent(new Event('input',{{bubbles:true}}));return 1}})()",
    "returnByValue": True
}, target_id=ds_tab_id)
```

### 步骤4：发送消息并轮询等待回复

发送后立即开始轮询检测 `.ds-markdown` 元素（每 2s 检查一次）：

```python
# 用 KeyboardEvent 模拟回车发送
browser_cdp(method="Runtime.evaluate", params={
    "expression": "(()=>{const ta=document.querySelector('textarea');ta.focus();ta.dispatchEvent(new KeyboardEvent('keydown',{key:'Enter',code:'Enter',bubbles:true,cancelable:true}));ta.dispatchEvent(new KeyboardEvent('keyup',{key:'Enter',code:'Enter',bubbles:true,cancelable:true}));return 1})()",
    "returnByValue": True
}, target_id=ds_tab_id)

# 轮询检测回复，每 2s 检查
for i in range(15):
    r = browser_cdp(method="Runtime.evaluate", params={"expression": "document.querySelectorAll('[class*=\"ds-markdown\"]').length", "returnByValue": True}, target_id=ds_tab_id)
    if r["result"]["result"]["value"] >= 3:
        terminal(command="sleep 3")
        break
    terminal(command="sleep 2")
```

### 步骤5：提取答案

**首选简单方法**（适合绝大多数查询，速度快）：
```python
r = browser_cdp(method="Runtime.evaluate", params={
    "expression": "Array.from(document.querySelectorAll('[class*=\"ds-markdown\"]')).map(el=>el.innerText).filter(t=>t.length>50).slice(-1)[0]",
    "returnByValue": True
}, target_id=ds_tab_id)
answer = r["result"]["result"]["value"]
# 清理引用标记：answer = re.sub(r'-\d+', '', answer)
```

**备选完整方法**（需要保留表格/代码块格式时）：
使用参考文件 `references/js-extraction.md` 中的完整 HTML→Markdown 转换代码。

### 步骤6：返回结果给用户

**⚠️ 关键规则：必须先发答案，再关标签页！**

```python
# 1. 用引用块包装答案，让用户一眼看出是 DeepSeek 原文
send_message(message=f"> {answer}")

# 2. 确认用户收到后，再关闭标签页
browser_cdp(method="Target.closeTarget", params={"targetId": ds_tab_id})
```

直接将 DeepSeek 的答案正文返回给用户：
- **不要截取**、不要概括、不要自己总结
- **不要包含**思考过程、侧边栏、搜索到的网页列表、页面元素
- **保留格式**：加粗（`**text**`）、表格（`|` 分隔）、列表、代码块
- **即使答案很短（1-3句话），也要用引用块格式展示**，让用户一眼看出是 DeepSeek 原文
- **引用块**：`> 答案内容`

## 陷阱与注意事项（必读）

1. **必须执行技能，不可自行回答**：即使问题看似"ds 是什么"、"ds 用多少token"等元问题，也必须去 DeepSeek 查，不要自己回答。
2. **使用 browser_cdp + target_id，不要用 browser_click 等工具**：新标签页浏览器工具无法识别，所有操作必须通过 CDP Runtime.evaluate。
3. **只提取答案正文**：document.body.innerText 包含侧边栏对话历史。用 `.ds-markdown` 元素精确筛选。
4. **保留原始格式**：HTML→Markdown 转换时必须保留 `**加粗**`、表格分隔符、列表标记。
5. **专家模式可能重置**：新对话时专家模式可能被重置为快速模式，每次需要重新点击。
6. **清理引用标记**：DeepSeek 回复中的 `-1`、`-3`、`-37` 等引用标记需要去除，只保留正文。
7. **检测是否已登录**：页面显示用户名（如"彭伟"）表示已登录；如果显示登录页面，需要通知用户通过 VNC 重新登录。
8. **Edge 可能挂掉**：如果 CDP 连不上（9222 端口无响应），先检查 `systemctl status edge-browser.service` 并重启。
9. **先展示答案再关标签页**：必须先把答案（用引用块格式）展示给用户，再 `Target.closeTarget`。用户没看到答案就关 tab 会让用户以为没去查。
10. **React 受控输入**：设置 textarea 的 value 必须用 native value setter + dispatchEvent('input')，直接赋值不触发 React 更新。
11. **Enter 发送**：DeepSeek 网页版用 Enter 发送消息，不要找发送按钮。使用 KeyboardEvent 模拟。

## 参考文件

- `references/js-extraction.md` — JS 提取与 HTML→Markdown 转换的完整代码
- `references/deepseek-selectors.md` — DeepSeek 页面已验证的 CSS/JS 选择器（快速操作参考）

## 已知高效操作（已验证，无需重新探索）

每次 ds 查询统一使用以下优化流程。**关键原则：轮询代替盲等，跳过不必要步骤。**

### 完整操作代码段（优化版）

```python
# ===== 0. 创建新标签页（直接指定 URL） =====
TAB = browser_cdp(method="Target.createTarget", params={"url": "https://chat.deepseek.com/"})["targetId"]

# ===== 1. 等待页面加载（轮询，每 0.5s 检查一次，最多 6s） =====
for i in range(12):
    r = browser_cdp(method="Runtime.evaluate", params={"expression": "!!document.querySelector('textarea')", "returnByValue": True}, target_id=TAB)
    if r["result"]["result"]["value"]: break
    terminal(command="sleep 0.5")

# ===== 2. 切专家模式（新标签页就是空白页，无需先点「开启新对话」） =====
browser_cdp(method="Runtime.evaluate", params={
    "expression": "(()=>{const all=document.querySelectorAll('*');for(const el of all){if(el.textContent==='专家模式'){el.click();return 1}}return 0})()",
    "returnByValue": True
}, target_id=TAB)

# ===== 3. 输入问题（nativeSetter + dispatchEvent） =====
q = "你的问题"
browser_cdp(method="Runtime.evaluate", params={
    "expression": f"(()=>{{const ta=document.querySelector('textarea');const ns=Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype,'value').set;ns.call(ta,`{q}`);ta.dispatchEvent(new Event('input',{{bubbles:true}}));return 1}})()",
    "returnByValue": True
}, target_id=TAB)

# ===== 4. 发送（KeyboardEvent Enter） =====
browser_cdp(method="Runtime.evaluate", params={
    "expression": "(()=>{const ta=document.querySelector('textarea');ta.focus();ta.dispatchEvent(new KeyboardEvent('keydown',{key:'Enter',code:'Enter',bubbles:true,cancelable:true}));ta.dispatchEvent(new KeyboardEvent('keyup',{key:'Enter',code:'Enter',bubbles:true,cancelable:true}));return 1})()",
    "returnByValue": True
}, target_id=TAB)

# ===== 5. 等待回复（轮询 ds-markdown，每 2s 检查，最多 30s） =====
for i in range(15):
    r = browser_cdp(method="Runtime.evaluate", params={"expression": "document.querySelectorAll('[class*=\"ds-markdown\"]').length", "returnByValue": True}, target_id=TAB)
    if r["result"]["result"]["value"] >= 3:
        terminal(command="sleep 3")  # 再等 3s 收尾
        break
    terminal(command="sleep 2")

# ===== 6. 提取答案（取最后一个长文本块） =====
r = browser_cdp(method="Runtime.evaluate", params={
    "expression": "Array.from(document.querySelectorAll('[class*=\"ds-markdown\"]')).map(el=>el.innerText).filter(t=>t.length>50).slice(-1)[0]",
    "returnByValue": True
}, target_id=TAB)
answer = r["result"]["result"]["value"]

# ===== 7. **先发答案给用户，再关标签页** =====
send_message(message=f"> {answer}")
browser_cdp(method="Target.closeTarget", params={"targetId": TAB})
```

### 优化前后对比

| 阶段 | 旧方案（盲等） | 新方案（轮询） |
|------|--------------|--------------|
| 等待页面加载 | sleep 5 | 轮询 0.5s×最多12次 → ~2-3s |
| 开启新对话 | sleep 2 | **跳过**（新标签页已是空白页） |
| 切专家模式 | sleep 1 | **无需等待** |
| 等待回复 | sleep 12+10=22s | 轮询 2s×最多15次 → ~8-15s |
| **总耗时** | **~30s** | **~10-18s** |

### 关键技术点

| 操作 | 方法 | 说明 |
|------|------|------|
| 新建标签页 | `Target.createTarget` 直接传 URL | 不要 about:blank + browser_navigate |
| 点击元素 | `Runtime.evaluate` + JS click() | React 页面，普通 selector 可能不生效 |
| 输入文字 | nativeSetter + dispatchEvent('input') | React 受控组件，直接设 value 不触发更新 |
| 发送消息 | dispatchEvent KeyboardEvent Enter | 比点按钮可靠 |
| 读页面 | `Runtime.evaluate` + target_id | browser_snapshot 不认 CDP 创建的 tab |
| 页面加载检测 | 轮询 `document.querySelector('textarea')` | 替代 blind sleep，提前感知就绪 |
| 回复检测 | 轮询 `[class*="ds-markdown"]` 数量 | 一旦 >=3 说明回复生成中，补 3s 收尾 |

## 优先级说明

本技能属于前缀触发（`ds`/`DS`），优先级高于通用搜索技能 `web-access`。
用户消息以 `ds` 开头时**直接走本技能**，不走 `web-access` 的三层通道调度。

## 依赖

- `browser` 工具集的 `browser_cdp` 方法
- `edge-browser.service`（systemctl start edge-browser.service）
- DeepSeek 登录状态有效
