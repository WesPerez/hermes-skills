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

### 步骤2：等待页面加载并点击"开启新对话"

```python
# 检查页面是否已加载
check = browser_cdp(method="Runtime.evaluate", params={
    "expression": "document.querySelector('textarea') !== null",
    "returnByValue": True
}, target_id=ds_tab_id)

# 点击"开启新对话"按钮（用文本搜索可点击父元素）
browser_cdp(method="Runtime.evaluate", params={
    "expression": """(() => {
        const it = document.createNodeIterator(document.body, 4);
        let n;
        while(n = it.nextNode()) {
            if(n.nodeValue && n.nodeValue.includes('开启新对话')) {
                let el = n.parentElement;
                while(el && window.getComputedStyle(el).cursor !== 'pointer') el = el.parentElement;
                if(el) { el.click(); return 'clicked'; }
            }
        }
        return 'not found';
    })()""",
    "returnByValue": True
}, target_id=ds_tab_id)
```

### 步骤3：切换到专家模式

```python
browser_cdp(method="Runtime.evaluate", params={
    "expression": """(() => {
        const all = document.querySelectorAll('*');
        for(const el of all) {
            if(el.textContent === '专家模式') { el.click(); return 'clicked'; }
        }
        return 'not found';
    })()""",
    "returnByValue": True
}, target_id=ds_tab_id)
```

### 步骤4：输入问题

用 native value setter 设置 textarea 值（React 受控组件需要触发 input 事件）：

```python
question = "你的问题文本"
browser_cdp(method="Runtime.evaluate", params={
    "expression": f"""(() => {{
        const ta = document.querySelector('textarea');
        if(!ta) return 'no textarea';
        const nativeSetter = Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype, 'value').set;
        nativeSetter.call(ta, {json.dumps(question)});
        ta.dispatchEvent(new Event('input', {{ bubbles: true }}));
        return 'OK';
    }})()""",
    "returnByValue": True
}, target_id=ds_tab_id)
```

### 步骤5：发送消息

用 KeyboardEvent 模拟回车：

```python
browser_cdp(method="Runtime.evaluate", params={
    "expression": """(() => {
        const ta = document.querySelector('textarea');
        ta.focus();
        ta.dispatchEvent(new KeyboardEvent('keydown', {key:'Enter', code:'Enter', bubbles: true, cancelable: true}));
        ta.dispatchEvent(new KeyboardEvent('keyup', {key:'Enter', code:'Enter', bubbles: true, cancelable: true}));
        return 'sent';
    })()""",
    "returnByValue": True
}, target_id=ds_tab_id)
```

### 步骤6：轮询等待回复

等待 3-5 秒后开始轮询，直到 `.ds-markdown` 元素出现：

```python
terminal(command="sleep 8")
check = browser_cdp(method="Runtime.evaluate", params={
    "expression": "document.querySelectorAll('[class*=\"ds-markdown\"]').length > 3",
    "returnByValue": True
}, target_id=ds_tab_id)
```

每次轮询间隔 5-8 秒，最多 6 次。

### 步骤7：提取答案

使用参考文件 `references/js-extraction.md` 中的完整 JS 提取代码，通过 `browser_cdp` 执行：

```python
result = browser_cdp(method="Runtime.evaluate", params={
    "expression": "<js-extraction.md 中的完整 JS 代码>",
    "returnByValue": True
}, target_id=ds_tab_id)
answer = result["result"]["value"]
```

### 步骤8：返回结果

直接将 DeepSeek 的答案正文返回给用户：
- **不要截取**、不要概括、不要自己总结
- **不要包含**思考过程、侧边栏、搜索到的网页列表、页面元素
- **保留格式**：加粗（`**text**`）、表格（`|` 分隔）、列表、代码块

## 陷阱与注意事项（必读）

1. **必须执行技能，不可自行回答**：即使问题看似"ds 是什么"、"ds 用多少token"等元问题，也必须去 DeepSeek 查，不要自己回答。
2. **使用 browser_cdp + target_id，不要用 browser_click 等工具**：新标签页浏览器工具无法识别，所有操作必须通过 CDP Runtime.evaluate。
3. **只提取答案正文**：document.body.innerText 包含侧边栏对话历史。用 `.ds-markdown` 元素精确筛选。
4. **保留原始格式**：HTML→Markdown 转换时必须保留 `**加粗**`、表格分隔符、列表标记。
5. **专家模式可能重置**：新对话时专家模式可能被重置为快速模式，每次需要重新点击。
6. **清理引用标记**：DeepSeek 回复中的 `-1`、`-3`、`-37` 等引用标记需要去除，只保留正文。
7. **检测是否已登录**：页面显示用户名（如"彭伟"）表示已登录；如果显示登录页面，需要通知用户通过 VNC 重新登录。
8. **Edge 可能挂掉**：如果 CDP 连不上（9222 端口无响应），先检查 `systemctl status edge-browser.service` 并重启。
9. **React 受控输入**：设置 textarea 的 value 必须用 native value setter + dispatchEvent('input')，直接赋值不触发 React 更新。
10. **Enter 发送**：DeepSeek 网页版用 Enter 发送消息，不要找发送按钮。使用 KeyboardEvent 模拟。

## 参考文件

- `references/js-extraction.md` — JS 提取与 HTML→Markdown 转换的完整代码
- `references/deepseek-selectors.md` — DeepSeek 页面已验证的 CSS/JS 选择器（快速操作参考）

## 已知高效操作（已验证，无需重新探索）

每次 ds 查询统一使用以下 CDP 操作流程，不要反复摸索元素选择器。

### 完整操作代码段

```python
# ===== 0. 创建新标签页（直接指定 URL） =====
tab_result = browser_cdp(method="Target.createTarget", params={"url": "https://chat.deepseek.com/"})
TAB = tab_result["targetId"]

# 等待加载
terminal(command="sleep 5")

# ===== 1. 开启新对话（避免复用旧会话状态） =====
browser_cdp(method="Runtime.evaluate", params={
    "expression": """(() => { const it = document.createNodeIterator(document.body, 4); let n; while(n = it.nextNode()) { if(n.nodeValue.includes('开启新对话')) { const el = n.parentElement; while(el && window.getComputedStyle(el).cursor !== 'pointer') el = el.parentElement; if(el) { el.click(); return 'ok'; } } } return 'not found'; })()""",
    "returnByValue": True
}, target_id=TAB)

terminal(command="sleep 2")

# ===== 2. 切换到专家模式 =====
browser_cdp(method="Runtime.evaluate", params={
    "expression": """(() => { const all = document.querySelectorAll('*'); for(const el of all) { if(el.textContent === '专家模式') { el.click(); return 'ok'; } } return 'not found'; })()""",
    "returnByValue": True
}, target_id=TAB)

terminal(command="sleep 1")

# ===== 3. 输入问题（必须用 nativeSetter + dispatchEvent） =====
QUESTION = "你的问题"
browser_cdp(method="Runtime.evaluate", params={
    "expression": f"""(function() {{
        const ta = document.querySelector('textarea');
        if(!ta) return 'no textarea';
        const ns = Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype, 'value').set;
        ns.call(ta, `{QUESTION}`);
        ta.dispatchEvent(new Event('input', {{ bubbles: true }}));
        return 'ok ' + ta.value.length;
    }})()""",
    "returnByValue": True
}, target_id=TAB)

# ===== 4. 按 Enter 发送 =====
browser_cdp(method="Runtime.evaluate", params={
    "expression": """const ta = document.querySelector('textarea'); ta.focus(); ta.dispatchEvent(new KeyboardEvent('keydown', {key:'Enter', code:'Enter', bubbles: true, cancelable: true})); ta.dispatchEvent(new KeyboardEvent('keyup', {key:'Enter', code:'Enter', bubbles: true, cancelable: true})); 'sent'""",
    "returnByValue": True
}, target_id=TAB)

# ===== 5. 等待回复完成（轮询检测） =====
# 检测标志：textarea 内容被清空 或 出现 ds-markdown 元素
for i in range(6):  # 最多等 30s
    terminal(command="sleep 5")
    done = browser_cdp(method="Runtime.evaluate", params={
        "expression": "document.querySelectorAll('[class*=\"ds-markdown\"]').length",
        "returnByValue": True
    }, target_id=TAB)
    if done["result"]["result"]["value"] >= 3:  # 出现3+个markdown块表示回复生成中
        break

# 再等一会让完整回复生成
terminal(command="sleep 10")

# ===== 6. 提取答案 =====
answer = browser_cdp(method="Runtime.evaluate", params={
    "expression": """(() => { const items = document.querySelectorAll('[class*="ds-markdown"]'); return Array.from(items).map(el => el.innerText).join('\\n\\n').substring(0, 10000); })()""",
    "returnByValue": True
}, target_id=TAB)

# ===== 7. 关闭标签页 =====
browser_cdp(method="Target.closeTarget", params={"targetId": TAB})
```

### 关键技术点

| 操作 | 方法 | 说明 |
|------|------|------|
| 新建标签页 | `Target.createTarget` 直接传 URL | 不要 about:blank + browser_navigate |
| 点击元素 | `Runtime.evaluate` + JS click() | React 页面，普通 selector 可能不生效 |
| 输入文字 | nativeSetter + dispatchEvent('input') | React 受控组件，直接设 value 不触发更新 |
| 发送消息 | dispatchEvent KeyboardEvent Enter | 比点按钮可靠 |
| 读页面 | `Runtime.evaluate` + target_id | browser_snapshot 不认 CDP 创建的 tab |

### 提速提示

- Page load: 5s 足够（CDP 直连比 browser_navigate 快）
- 新对话: 2s
- 专家模式: 1s
- 生成等待: 简单问题 ~10s，复杂问题 ~20-25s（含深度思考）
- **总时间通常在 20-40s 内完成一次完整查询**

## 优先级说明

本技能属于前缀触发（`ds`/`DS`），优先级高于通用搜索技能 `web-access`。
用户消息以 `ds` 开头时**直接走本技能**，不走 `web-access` 的三层通道调度。

## 依赖

- `browser` 工具集的 `browser_cdp` 方法
- `edge-browser.service`（systemctl start edge-browser.service）
- DeepSeek 登录状态有效
