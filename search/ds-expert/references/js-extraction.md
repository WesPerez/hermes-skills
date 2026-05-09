# DeepSeek 答案提取方案

从 DeepSeek 页面提取答案正文（排除思考过程、侧边栏）。

## 提取原理

DeepSeek 将回复渲染在多个 `.ds-markdown` 元素中。前面几个元素通常是思考过程/搜索过程，最后几个是实际答案。

## 方案一：简单提取（推荐，适用于大多数查询）

```python
# 取最后一个长文本块（>=50字符），自动跳过思考过程
r = browser_cdp(method="Runtime.evaluate", params={
    "expression": "Array.from(document.querySelectorAll('[class*=\"ds-markdown\"]')).map(el=>el.innerText).filter(t=>t.length>50).slice(-1)[0]",
    "returnByValue": True
}, target_id=TAB)
answer = r["result"]["result"]["value"]
```

**优点**：快，简单，不用处理 HTML
**缺点**：纯文本，丢失加粗/表格格式
**适合**：绝大多数文本查询

## 方案二：全文拼接（适合需要看到完整对话）

```python
r = browser_cdp(method="Runtime.evaluate", params={
    "expression": "Array.from(document.querySelectorAll('[class*=\"ds-markdown\"]')).map(el=>el.innerText).join('\n---\n')",
    "returnByValue": True
}, target_id=TAB)
```

## 方案三：HTML 转 Markdown（适合含表格的答案）

当答案包含复杂表格时，用完整 HTML→Markdown 转换保留格式：

```python
extraction_js = "(() => { let all = document.querySelectorAll('[class*=\"ds-markdown\"]'); let parts = []; let capture = false; for (let el of all) { let t = el.innerText; if (t.includes('\u6839\u636e') || t.match(/^[^\u3002]{3,}\u7684[^\u3002]{2,}(\u662f|\u4e3a|\u6307|\u5206\u4e3a)/)) { capture = true; } if (capture) { let html = el.innerHTML; html = html.replace(/<svg[^>]*>[\\s\\S]*?<\\/svg>/g, ''); html = html.replace(/<table[^>]*>/g, '\\n').replace(/<\\/table>/g, ''); html = html.replace(/<tr[^>]*>/g, '').replace(/<\\/tr>/g, '\\n'); html = html.replace(/<th[^>]*>/g, '| ').replace(/<\\/th>/g, ' '); html = html.replace(/<td[^>]*>/g, '| ').replace(/<\\/td>/g, ' '); html = html.replace(/<(?:strong|b)[^>]*>/g, '**').replace(/<\\/(?:strong|b)>/g, '**'); html = html.replace(/<(?:em|i)[^>]*>/g, '*').replace(/<\\/(?:em|i)>/g, '*'); html = html.replace(/<p[^>]*>/g, '\\n\\n').replace(/<\\/p>/g, ''); html = html.replace(/<br\\s*\\/?>/g, '\\n'); html = html.replace(/<h[23][^>]*>/g, '\\n\\n## ').replace(/<\\/h[23]>/g, ''); html = html.replace(/<ul[^>]*>/g, '\\n').replace(/<\\/ul>/g, ''); html = html.replace(/<li[^>]*>/g, '- ').replace(/<\\/li>/g, '\\n'); html = html.replace(/<span[^>]*>/g, '').replace(/<\\/span>/g, ''); html = html.replace(/<div[^>]*>/g, '').replace(/<\\/div>/g, ''); html = html.replace(/<a[^>]*>(.*?)<\\/a>/g, '$1'); html = html.replace(/<code[^>]*>/g, '`').replace(/<\\/code>/g, '`'); html = html.replace(/<pre[^>]*>/g, '\\n```\\n').replace(/<\\/pre>/g, '\\n```\\n'); html = html.replace(/-\\d+/g, ''); html = html.replace(/\\n{4,}/g, '\\n\\n').trim(); if (html) parts.push(html); } } let result = parts.join('\\n\\n'); let lines = result.split('\\n'); let unique = []; let seen = new Set(); for (let line of lines) { let t = line.trim(); if (t && !seen.has(t)) { seen.add(t); unique.push(line); } else if (!t) { unique.push(line); } } return unique.join('\\n').replace(/\\n{4,}/g, '\\n\\n').trim(); })()"

result = browser_cdp(method="Runtime.evaluate", params={
    "expression": extraction_js,
    "returnByValue": True
}, target_id=TAB)
answer = result["result"]["value"]
```

**适合**：答案包含复杂表格、代码块等需要保留格式的场景

## 方案选择指南

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| 一般问答、文本描述 | 方案一 | 最快，不丢失重点 |
| 需要完整对话内容 | 方案二 | 拿到全部信息，自行过滤 |
| 含表格/代码块 | 方案三 | 保留格式 |
| 不确定 | 方案一优先 | 如果格式损失严重再降级方案三 |

## 回复完成检测（轮询用）

```python
r = browser_cdp(method="Runtime.evaluate", params={
    "expression": "document.querySelectorAll('[class*=\"ds-markdown\"]').length",
    "returnByValue": True
}, target_id=TAB)
count = r["result"]["result"]["value"]
# count >= 3 表示回复生成中
# count >= 6 表示回复基本完成
```
