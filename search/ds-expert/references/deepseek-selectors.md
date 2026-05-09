# DeepSeek 页面选择器速查表

> 所有已验证，下次直接用，无需重新探测

## 元素定位

| 元素 | 定位方法 | 已验证 |
|------|---------|-------|
| textarea | `document.querySelector('textarea')` | ✅ |
| 开启新对话 | NodeIterator 搜索含"开启新对话"文本节点 → 向上找 cursor:pointer 的父元素 | ✅ |
| 专家模式 | 遍历 `document.querySelectorAll('*')`，找 `el.textContent === '专家模式'` | ✅ |
| ds-markdown 块 | `document.querySelectorAll('[class*="ds-markdown"]')` | ✅ |

## 完整操作模板

```python
# === 准备阶段 ===
tab = browser_cdp(method="Target.createTarget", params={"url": "https://chat.deepseek.com/"})["targetId"]
terminal(command="sleep 5")

# === 1. 开启新对话 ===
browser_cdp(method="Runtime.evaluate", params={
    "expression": "(()=>{const it=document.createNodeIterator(document.body,4);let n;while(n=it.nextNode()){if(n.nodeValue&&n.nodeValue.includes('开启新对话')){let el=n.parentElement;while(el&&getComputedStyle(el).cursor!=='pointer')el=el.parentElement;if(el){el.click();return 1}}}})()",
    "returnByValue": True
}, target_id=tab)
terminal(command="sleep 2")

# === 2. 切专家模式 ===
browser_cdp(method="Runtime.evaluate", params={
    "expression": "(()=>{const all=document.querySelectorAll('*');for(const el of all){if(el.textContent==='专家模式'){el.click();return 1}}return 0})()",
    "returnByValue": True
}, target_id=tab)
terminal(command="sleep 1")

# === 3. 输入问题 ===
question = "你的问题"
browser_cdp(method="Runtime.evaluate", params={
    "expression": f"(()=>{{const ta=document.querySelector('textarea');const ns=Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype,'value').set;ns.call(ta,'{question}');ta.dispatchEvent(new Event('input',{{bubbles:true}}));return ta.value.length}})()",
    "returnByValue": True
}, target_id=tab)

# === 4. 发送 ===
browser_cdp(method="Runtime.evaluate", params={
    "expression": "(()=>{const ta=document.querySelector('textarea');ta.focus();ta.dispatchEvent(new KeyboardEvent('keydown',{key:'Enter',code:'Enter',bubbles:true,cancelable:true}));ta.dispatchEvent(new KeyboardEvent('keyup',{key:'Enter',code:'Enter',bubbles:true,cancelable:true}));return 1})()",
    "returnByValue": True
}, target_id=tab)

# === 5. 等回复 ===
for i in range(6):
    terminal(command="sleep 5")
    r = browser_cdp(method="Runtime.evaluate", params={"expression": "document.querySelectorAll('[class*=\"ds-markdown\"]').length", "returnByValue": True}, target_id=tab)
    if r["result"]["result"]["value"] >= 3: break
terminal(command="sleep 10")

# === 6. 提取 ===
r = browser_cdp(method="Runtime.evaluate", params={"expression": "Array.from(document.querySelectorAll('[class*=\"ds-markdown\"]')).map(el=>el.innerText).join('\\n\\n')", "returnByValue": True}, target_id=tab)
answer = r["result"]["result"]["value"]

# === 7. 关闭 ===
browser_cdp(method="Target.closeTarget", params={"targetId": tab})
```

## 时间参考

| 阶段 | 耗时 |
|------|------|
| 页面加载 | ~5s |
| 开启新对话 | ~2s |
| 切专家模式 | ~1s |
| 输入+发送 | ~1s |
| 等待生成 | 简单 ~10s, 复杂 ~25s |
| **合计** | **~20-35s** |

## 注意事项

- 不要用 browser_navigate/browser_click/browser_snapshot，全用 browser_cdp + target_id
- textarea 赋值必须用 nativeSetter + dispatchEvent('input')
- Enter 发送用 KeyboardEvent，不要点按钮
- 任务完成后必须关闭标签页
