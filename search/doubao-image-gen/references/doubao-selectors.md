# 豆包 CDP 操作速查表

> 所有选择器和操作已验证，直接复用

## 元素定位

| 元素 | 定位方法 | 说明 |
|------|---------|------|
| 登录检测 | `document.body.innerText.includes('登录')` | true=未登录 |
| 图像生成按钮 | 遍历所有元素，找 `el.textContent.trim() === '图像生成'`，然后向上找可点击父元素 | 豆包底部工具栏 |
| 比例按钮 | 同上，找 `el.textContent.trim() === '比例'` | 需先点出比例面板 |
| 3:4 选项 | 同上，找 `el.textContent.trim() === '3:4'` | 比例面板弹出后 |
| 输入框 | 遍历 textarea/input/contenteditable，找 placeholder 含"描述"或"图片"或"输入" | 图像生成专属输入框 |
| 发送按钮 | 遍历按钮/可点击元素，找文本含"发送"或"生成"，或 class 含"send"/"submit"/"生成" | **不要用 Enter** |
| 生成图片 | `img[src*="rc_gen_image"]` | 生成完成后出现4张 |

## 完整操作模板

```python
# === 准备 ===
tab = browser_cdp(method="Target.createTarget", params={"url": "https://www.doubao.com/chat/"})["targetId"]

# === 等待页面加载（轮询，最多 6s） ===
for i in range(12):
    r = browser_cdp(method="Runtime.evaluate", params={"expression": "!!document.body && document.body.innerText.length > 100", "returnByValue": True}, target_id=tab)
    if r["result"]["result"]["value"]: break
    terminal(command="sleep 0.5")

# === 检测登录 ===
logged = browser_cdp(method="Runtime.evaluate", params={
    "expression": "!document.body.innerText.includes('登录')",
    "returnByValue": True
}, target_id=tab)
# 未登录则通知用户

# === 点击"图像生成" ===
browser_cdp(method="Runtime.evaluate", params={
    "expression": "(()=>{const all=document.querySelectorAll('*');for(const el of all){if(el.textContent&&el.textContent.trim()==='图像生成'){let c=el;while(c&&c.tagName!=='BUTTON'&&(!c.onclick&&getComputedStyle(c).cursor!=='pointer'))c=c.parentElement;if(c.tagName==='BODY')c=el;c.click();return 1}}return 0})()",
    "returnByValue": True
}, target_id=tab)
# 等待图像生成界面（轮询检测输入框，最多 4s）
for i in range(8):
    r = browser_cdp(method="Runtime.evaluate", params={"expression": "Array.from(document.querySelectorAll('textarea,input,[contenteditable]')).some(el=>(el.placeholder||'').includes('描述')||(el.placeholder||'').includes('图片'))", "returnByValue": True}, target_id=tab)
    if r["result"]["result"]["value"]: break
    terminal(command="sleep 0.5")

# === 设置比例 3:4 ===
browser_cdp(method="Runtime.evaluate", params={
    "expression": "(()=>{const all=document.querySelectorAll('*');for(const el of all){if(el.textContent&&el.textContent.trim()==='比例'){let c=el;while(c&&c.tagName!=='BUTTON'&&(!c.onclick&&getComputedStyle(c).cursor!=='pointer'))c=c.parentElement;if(c.tagName==='BODY')c=el;c.click();return 1}}return 0})()",
    "returnByValue": True
}, target_id=tab)
# 等待比例面板弹出（轮询，最多 4s）
for i in range(8):
    r = browser_cdp(method="Runtime.evaluate", params={"expression": "Array.from(document.querySelectorAll('*')).some(el=>el.textContent&&el.textContent.trim()==='3:4'&&getComputedStyle(el).cursor==='pointer'&&el.offsetParent!==null)", "returnByValue": True}, target_id=tab)
    if r["result"]["result"]["value"]: break
    terminal(command="sleep 0.5")

# 选 3:4（无需等待）

# === 输入提示词 ===
prompt = "用户描述，画面上方留出约30像素的空间，主体放在中下部"
browser_cdp(method="Runtime.evaluate", params={
    "expression": f"(()=>{{const inputs=document.querySelectorAll('textarea,input,[contenteditable]');for(const el of inputs){{const ph=el.placeholder||el.getAttribute('aria-label')||'';if(ph.includes('描述')||ph.includes('图片')){{const ns=Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype,'value').set;ns.call(el,'{prompt}');el.dispatchEvent(new Event('input',{{bubbles:true}}));return 1}}}}return 0}})()",
    "returnByValue": True
}, target_id=tab)

# === 点击发送（勿用Enter） ===
# 用 vision 截图找发送按钮坐标，或遍历文本
browser_cdp(method="Runtime.evaluate", params={
    "expression": "(()=>{const all=document.querySelectorAll('button,[role=\"button\"]');for(const el of all){const t=el.textContent.trim();if(t==='发送'||t==='生成'||t.includes('发送')){el.click();return 1}}return 0})()",
    "returnByValue": True
}, target_id=tab)

# === 轮询等待 ===
for i in range(12):
    terminal(command="sleep 5")
    r = browser_cdp(method="Runtime.evaluate", params={
        "expression": "document.querySelectorAll('img[src*=\"rc_gen_image\"]').length",
        "returnByValue": True
    }, target_id=tab)
    if r["result"]["result"]["value"] >= 4:
        break

# === 获取图片 URL ===
r = browser_cdp(method="Runtime.evaluate", params={
    "expression": "JSON.stringify(Array.from(document.querySelectorAll('img')).filter(i=>i.src&&i.src.includes('rc_gen_image')).map(i=>i.src))",
    "returnByValue": True
}, target_id=tab)
urls = json.loads(r["result"]["result"]["value"])

# === 下载并发送 ===
# ... (下载、裁剪、发送)

# ⚠️ 先发图再关标签页
```

## 时间参考（优化后）

| 阶段 | 旧方案 | 新方案 |
|------|-------|-------|
| 页面加载 | 5s 盲等 | ~2-3s 轮询 |
| 切换图像生成 | 3s 盲等 | ~1-2s 轮询 |
| 设置比例 | 4s 盲等 | ~1-2s 轮询 |
| 输入+发送 | ~2s | ~1s |
| 等待生成 | ~25-30s | ~20-25s（不变，实际生成时间） |
| **合计** | **~35-40s** | **~25-32s** |

## 已知问题

- **不要按 Enter**：豆包图像生成模式按 Enter 会重置回首页
- **发送按钮难找**：可能是个箭头图标而非文字按钮，兜底方案用 browser_vision 截图定位点击
- **连接器复用**：注意 `Target.createTarget` 创建新标签页后，不要用 `openerFrameId` 的旧标签页
- **先发图再关标签页**：用户确认收到后再 closeTarget
