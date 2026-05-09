---
name: doubao-image-gen
description: 用户消息以"db"开头时，通过 Edge CDP 控制浏览器进入豆包网页版进行 AI 图像生成并返回图片。
tags:
  - search
  - doubao
  - image-generation
  - web-browser
  - edge-cdp
triggers:
  - db 开头的问题
related_skills:
  - ds-expert
  - remote-desktop-setup
---

# 豆包图像生成（Doubao Image Generation）

当用户消息以 `db` 开头时，自动使用豆包网页版生成图片。

## 触发条件

用户消息开头为 `db`（不区分大小写），去掉前缀后作为生图提示词。例如：
- `db 一只可爱的猫` → 用豆包生成猫的图片
- `db DNF魔道学者表情包` → 生成 DNF 相关图片

## 用户偏好（2026-05-08 用户纠正记录）

- **保持技能通用**：不要擅自在技能中嵌入特定角色/话题的提示词模板（如 DNF、魔道学者）。生图提示词直接用用户的描述，可适当润色但不要增加特定领域模板内容。
- **自动加构图要求**：每次生图在提示词末尾追加「画面上方留出约30像素的空间，主体放在中下部」，确保主体不被裁切。此条为固定规则。
- **`db` 前缀触发**：按 `ds`/`db` 前缀模式，去掉前缀后内容作为提示词。
- **3:4 为默认比例**：必须先点"比例"按钮设为 3:4，再输入提示词。

## 前置条件

- **Edge 浏览器持久运行**：CDP 端口 9222（`edge-browser.service`），profile 在 `/root/.edge-profile`
- **Hermes 已配置**：`browser.cdp_url: ws://127.0.0.1:9222`
- **豆包已登录**：用户通过 NoVNC 登录一次后 Cookie 持久化
- **图片处理工具**：Pillow（用于裁剪水印）

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
    time.sleep(5)

# 3. 创建新标签页并直接导航到豆包（不要创建空白页再 browser_navigate）
result = browser_cdp(method="Target.createTarget", params={"url": "https://www.doubao.com/chat/"})
db_tab_id = result["targetId"]
```

> ⚠️ 每次操作都用新标签页，任务完成后关闭。
>
> **不要使用 browser_navigate / browser_click / browser_type / browser_press / browser_snapshot / browser_console 等 Hermes 浏览器工具**——它们操作的是默认标签页，不是 CDP 创建的新标签页。
>
> 后续所有操作必须使用 `browser_cdp(method="Runtime.evaluate", ..., target_id=db_tab_id)`。

### 关闭标签页
```python
browser_cdp(method="Target.closeTarget", params={"targetId": db_tab_id})
```

## 工作流程（全 CDP 方式）

所有操作使用 `browser_cdp(method="Runtime.evaluate", ..., target_id=db_tab_id)` 执行 JavaScript。

**注意：不要使用 browser_navigate / browser_click / browser_type / browser_press / browser_snapshot / browser_console 等 Hermes 浏览器工具——它们操作的是默认标签页，不是 CDP 创建的新标签页。**

### 步骤1：等待页面加载

```python
terminal(command="sleep 5")
```

检测是否已登录：检查页面是否包含用户名。
```python
check = browser_cdp(method="Runtime.evaluate", params={
    "expression": "document.body.innerText.includes('登录') ? 'need_login' : 'logged_in'",
    "returnByValue": True
}, target_id=db_tab_id)
```

如果显示登录页面，通知用户通过 VNC 登录。

### 步骤2：进入图像生成模式

在页面底部工具栏找到"图像生成"按钮并点击：

```python
browser_cdp(method="Runtime.evaluate", params={
    "expression": """(() => {
        const all = document.querySelectorAll('*');
        for(const el of all) {
            if(el.textContent && el.textContent.trim() === '图像生成') {
                // 找可点击的父元素
                let clickable = el;
                while(clickable && clickable.tagName !== 'BUTTON' && (!clickable.onclick && window.getComputedStyle(clickable).cursor !== 'pointer' && clickable.tagName !== 'A')) {
                    clickable = clickable.parentElement;
                }
                if(clickable.tagName === 'BODY') clickable = el;
                clickable.click();
                return 'clicked 图像生成';
            }
        }
        return 'not found';
    })()""",
    "returnByValue": True
}, target_id=db_tab_id)
terminal(command="sleep 3")  # 等待界面切换
```

### 步骤3：设置比例

在图像生成工具栏中找到"比例"按钮并点击：

```python
browser_cdp(method="Runtime.evaluate", params={
    "expression": """(() => {
        const all = document.querySelectorAll('*');
        for(const el of all) {
            if(el.textContent && el.textContent.trim() === '比例') {
                let clickable = el;
                while(clickable && clickable.tagName !== 'BUTTON' && (!clickable.onclick && window.getComputedStyle(clickable).cursor !== 'pointer' && clickable.tagName !== 'A')) {
                    clickable = clickable.parentElement;
                }
                if(clickable.tagName === 'BODY') clickable = el;
                clickable.click();
                return 'clicked 比例';
            }
        }
        return 'not found';
    })()""",
    "returnByValue": True
}, target_id=db_tab_id)
terminal(command="sleep 2")

# 在弹出选项中选择 3:4
browser_cdp(method="Runtime.evaluate", params={
    "expression": """(() => {
        const all = document.querySelectorAll('*');
        for(const el of all) {
            if(el.textContent && el.textContent.trim() === '3:4') {
                let clickable = el;
                while(clickable && window.getComputedStyle(clickable).cursor !== 'pointer') {
                    clickable = clickable.parentElement;
                }
                clickable.click();
                return 'selected 3:4';
            }
        }
        return 'not found';
    })()""",
    "returnByValue": True
}, target_id=db_tab_id)
terminal(command="sleep 2")
```

| 比例 | 适用场景 |
|------|---------|
| 1:1 方形 | 头像、表情包 |
| **3:4 竖版** | **默认推荐，适合手机壁纸、小红书封面、表情包** |
| 16:9 横版 | PC 壁纸、横版海报 |

### 步骤4：输入提示词

找到图像生成输入框（placeholder 为"描述你想要的图片"），用 native value setter 输入：

```python
prompt = "用户提示词" + "，画面上方留出约30像素的空间，主体放在中下部"
browser_cdp(method="Runtime.evaluate", params={
    "expression": f"""(() => {{
        const inputs = document.querySelectorAll('textarea, input, [contenteditable]');
        for(const el of inputs) {{
            const ph = el.placeholder || el.getAttribute('aria-label') || '';
            if(ph.includes('描述') || ph.includes('图片') || ph.includes('输入')) {{
                if(el.tagName === 'TEXTAREA' || el.tagName === 'INPUT') {{
                    const nativeSetter = Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype, 'value').set ||
                                         Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
                    if(nativeSetter) {{
                        nativeSetter.call(el, {json.dumps(prompt)});
                        el.dispatchEvent(new Event('input', {{ bubbles: true }}));
                    }}
                }} else {{
                    el.innerText = {json.dumps(prompt)};
                }}
                return 'typed ' + {len(prompt)} + ' chars';
            }}
        }}
        return 'no matching input';
    }})()""",
    "returnByValue": True
}, target_id=db_tab_id)
```

#### 提示词构图要求（必加）
在提示词末尾追加：
```
画面上方留出约30像素的空间，主体放在中下部
```

### 步骤5：点击发送按钮

**不要按 Enter**（实测按 Enter 会导致页面重置回到首页），必须找发送按钮点击：

```python
browser_cdp(method="Runtime.evaluate", params={
    "expression": """(() => {
        // 找发送按钮：通常是一个箭头/发送图标，在输入框附近
        const all = document.querySelectorAll('button, [role="button"], svg, [class*="send"], [class*="submit"]');
        for(const el of all) {
            const text = el.textContent.trim();
            const cls = el.className || '';
            const aria = el.getAttribute('aria-label') || '';
            if(text === '发送' || text === '生成' || text.includes('发送') || text.includes('生成') ||
               cls.includes('send') || cls.includes('submit') || cls.includes('生成') ||
               aria.includes('发送') || aria.includes('生成')) {
                el.click();
                return 'clicked';
            }
        }
        // 找不到文本，尝试找最后一个箭头图标
        const svgs = document.querySelectorAll('svg');
        for(const svg of svgs) {
            if(svg.getAttribute('name') === 'Send' || svg.getAttribute('data-icon') === 'send') {
                svg.closest('button, [role="button"], div')?.click();
                return 'clicked svg';
            }
        }
        return 'button not found, will try alternatives';
    })()""",
    "returnByValue": True
}, target_id=db_tab_id)
```

如果以上找不到，尝试用 vision 截图找到发送按钮坐标后点击（仅在兜底时使用）。

### 步骤6：等待生成

生成状态会显示"正在为您生成..."。豆包生成时间：第一张约 **15-20 秒**，全部 4 张约 **25 秒**。

轮询检测策略（每 5 秒检查一次，最多 12 次）：

```python
for i in range(12):
    terminal(command="sleep 5")
    check = browser_cdp(method="Runtime.evaluate", params={
        "expression": "document.querySelectorAll('img[src*=\"rc_gen_image\"]').length",
        "returnByValue": True
    }, target_id=db_tab_id)
    img_count = check["result"]["value"]
    if img_count >= 4:
        break
```

### 步骤7：获取图片 URL

```python
result = browser_cdp(method="Runtime.evaluate", params={
    "expression": """JSON.stringify(Array.from(document.querySelectorAll('img'))
        .filter(img => img.src && img.src.includes('rc_gen_image'))
        .map(img => img.src))""",
    "returnByValue": True
}, target_id=db_tab_id)
urls = json.loads(result["result"]["value"])
```

返回 4 个 byteimg URL，格式如：
```
https://p5-flow-imagex-sign.byteimg.com/.../rc_gen_image/<hash>.jpeg~tplv-<size>_watermark_1_5_b.png?<params>
```

### 步骤8：下载图片

豆包防盗链不严格，加 Referer 头即可：

```python
import requests
headers = {
    'Referer': 'https://www.doubao.com/',
    'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36'
}
for i, url in enumerate(urls):
    r = requests.get(url, headers=headers, timeout=10)
    with open(f'doubao_{i+1}.jpg', 'wb') as f:
        f.write(r.content)
```

每张约 115-190KB（视比例不同：3:4 约 115-124KB / 288×384；方形约 170-190KB / 384×384）。

### 步骤9：裁剪水印并发送

```python
from PIL import Image
for i in range(1, 5):
    img = Image.open(f'doubao_{i}.jpg')
    cropped = img.crop((0, 23, img.width, img.height))  # 裁剪顶部23像素水印
    cropped.save(f'clean_{i}.png')
    send_message(message=f"MEDIA:/tmp/clean_{i}.png")
```

4 张图依次发送给用户。

### 步骤10：待用户确认收到后关闭标签页

**⚠️ 关键规则**：先把图片发给用户，等用户确认收到（或给出回复）后，再关闭标签页。不要先关标签页再发图。

```python
# 等用户确认后再关闭
# browser_cdp(method="Target.closeTarget", params={"targetId": db_tab_id})
```

## 水印裁剪说明

豆包图片左上角有水印，统一裁剪顶部 23 像素。不同比例的裁剪结果：

| 比例 | 原始尺寸 | 裁剪后尺寸 |
|------|---------|-----------|
| 1:1 方形 | 384 × 384 | 384 × 361 |
| **3:4 竖版（默认）** | **288 × 384** | **288 × 361** |
| 16:9 横版 | 512 × 288 | 512 × 265 |

裁剪代码通用，自动适配任意尺寸：
```python
cropped = img.crop((0, 23, img.width, img.height))
```

## 常见陷阱

1. **按 Enter 重置页面**：豆包图像生成模式下按 Enter 会导致输入框清空并回到首页。**必须点击发送按钮**。
2. **比例在生图前设置**：比例按钮需要在输入提示词之前点击并选好，生图过程中无法再改。
3. **提示词构图要求自动加入**：每次生图都要在提示词末尾加上「画面上方留出约30像素的空间，主体放在中下部」，确保主体不被裁切。
4. **登录失效**：Cookie 约 7 天过期，需要用户通过 VNC（http://43.159.168.34/desktop/）重新登录一次。
5. **浏览器挂掉**：CDP 9222 连不上时执行 `systemctl restart edge-browser.service`。
6. **页面改版**：豆包经常更新 UI，所有 ref ID 每次加载可能不同。用 placeholder 文本（描述你想要的图片、发消息...）和按钮文本（图像生成、比例）作为识别依据。
7. **先发图再关闭标签页**：必须先把图片发送给用户，确认收到后，再 Target.closeTarget。不要先关 tab 再发图。

## 参考文件

- `references/test-session-2026-05-08.md` — 实测流程记录，包含 URL 格式、尺寸数据、COLG 参考帖子等细节
- `references/doubao-selectors.md` — 豆包 CDP 操作速查表（已验证的选择器 + 完整操作模板）

## 优先级说明

本技能属于前缀触发（`db`），优先级高于通用搜索技能 `web-access`。
用户消息以 `db` 开头时直接走本技能生图，不走联网搜索流程。
