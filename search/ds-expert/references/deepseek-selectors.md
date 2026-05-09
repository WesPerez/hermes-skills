# DeepSeek 页面操作速查表（优化版）

> 所有操作均使用 `browser_cdp(method="Runtime.evaluate", ..., target_id=TAB)` 执行

## 优化原则

1. **轮询代替盲等**：永远不写 `sleep(5)`，用轮询检测 DOM 状态
2. **新标签页就是空白页**：跳过「开启新对话」点击
3. **并发检测**：等待回复时用轮询，检测到内容就立即提取

## 极速操作模板

```python
TAB = browser_cdp(method="Target.createTarget", params={"url": "https://chat.deepseek.com/"})["targetId"]

# === 等待页面加载（轮询，最多等 6s） ===
for i in range(12):
    r = browser_cdp(method="Runtime.evaluate", params={"expression": "!!document.querySelector('textarea')", "returnByValue": True}, target_id=TAB)
    if r["result"]["result"]["value"]: break
    terminal(command="sleep 0.5")

# === 切专家模式（新标签页直接是空白页，不用点「开启新对话」） ===
browser_cdp(method="Runtime.evaluate", params={"expression": "(()=>{const all=document.querySelectorAll('*');for(const el of all){if(el.textContent==='专家模式'){el.click();return 1}}return 0})()", "returnByValue": True}, target_id=TAB)

# === 输入问题 ===
q = "你的问题"
browser_cdp(method="Runtime.evaluate", params={
    "expression": f"(()=>{{const ta=document.querySelector('textarea');const ns=Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype,'value').set;ns.call(ta,`{q}`);ta.dispatchEvent(new Event('input',{{bubbles:true}}));return 1}})()",
    "returnByValue": True
}, target_id=TAB)

# === 发送 ===
browser_cdp(method="Runtime.evaluate", params={
    "expression": "(()=>{const ta=document.querySelector('textarea');ta.focus();ta.dispatchEvent(new KeyboardEvent('keydown',{key:'Enter',code:'Enter',bubbles:true,cancelable:true}));ta.dispatchEvent(new KeyboardEvent('keyup',{key:'Enter',code:'Enter',bubbles:true,cancelable:true}));return 1})()",
    "returnByValue": True
}, target_id=TAB)

# === 等待回复（轮询 ds-markdown 元素，每 2s 检一次） ===
for i in range(15):  # 最多等 30s
    r = browser_cdp(method="Runtime.evaluate", params={"expression": "document.querySelectorAll('[class*=\"ds-markdown\"]').length", "returnByValue": True}, target_id=TAB)
    count = r["result"]["result"]["value"]
    if count >= 3:
        terminal(command="sleep 3")  # 再多等 3s 让内容写完
        break
    terminal(command="sleep 2")

# === 提取答案 ===
r = browser_cdp(method="Runtime.evaluate", params={
    "expression": "Array.from(document.querySelectorAll('[class*=\"ds-markdown\"]')).map(el=>el.innerText).filter(t=>t.length>50).slice(-1)[0]",
    "returnByValue": True
}, target_id=TAB)
answer = r["result"]["result"]["value"]

# === 关闭标签页 ===
browser_cdp(method="Target.closeTarget", params={"targetId": TAB})
```

## 优化后预期耗时

| 阶段 | 旧方案 | 新方案 |
|------|-------|-------|
| 等待页面加载 | 5s 盲等 | ~2-3s 轮询 |
| 开启新对话 | 2s 盲等 | **跳过** |
| 切专家模式 | 1s 盲等 | 立即（~0.1s） |
| 输入+发送 | ~0.2s | ~0.2s |
| 等待回复 | 12s 盲等 | ~8-15s 轮询 |
| **总时间** | **~20s** | **~10-18s** |

## 关键技巧

- 新标签页打开 DeepSeek 时已经自动处于新对话状态，无需点「开启新对话」
- `terminal(command="sleep 0.5")` 比 `terminal(command="sleep 5")` 精细很多，轮询总时间反而更短
- 回复检测用 `.ds-markdown` 长度 >= 3 作为信号，然后仅再等 3s 收尾
