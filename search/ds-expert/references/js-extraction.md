# DeepSeek 答案提取 JS 代码

从 DeepSeek 页面提取答案正文（排除思考过程、侧边栏）。

## 提取原理

DeepSeek 将回复渲染在多个 `.ds-markdown` 元素中。思考过程在独立的折叠区域里，答案正文从"根据..."等开头关键字处开始。

## 完整提取代码

```javascript
// 获取所有 markdown 内容块
let all = document.querySelectorAll('[class*="ds-markdown"]');
let parts = [];
let capture = false;

for (let el of all) {
  let t = el.innerText;
  // 答案通常以"根据"、"XX的官方文档"等开头，用首句特征触发捕获
  if (t.includes('根据') || t.match(/^[^。]{3,}的[^。]{2,}(是|为|指|分为)/)) {
    capture = true;
  }
  if (capture) {
    let html = el.innerHTML;
    // 清理 SVG 图标
    html = html.replace(/<svg[^>]*>[\s\S]*?<\/svg>/g, '');
    // 表格：保留分隔线结构
    html = html.replace(/<table[^>]*>/g, '\\n').replace(/<\/table>/g, '');
    html = html.replace(/<tr[^>]*>/g, '').replace(/<\/tr>/g, '\\n');
    html = html.replace(/<th[^>]*>/g, '| ').replace(/<\/th>/g, ' ');
    html = html.replace(/<td[^>]*>/g, '| ').replace(/<\/td>/g, ' ');
    // 加粗/斜体
    html = html.replace(/<strong[^>]*>/g, '**').replace(/<\/strong>/g, '**');
    html = html.replace(/<b[^>]*>/g, '**').replace(/<\/b>/g, '**');
    html = html.replace(/<em[^>]*>/g, '*').replace(/<\/em>/g, '*');
    // 段落/换行
    html = html.replace(/<p[^>]*>/g, '\\n\\n').replace(/<\/p>/g, '');
    html = html.replace(/<br\s*\/?>/g, '\\n');
    // 标题
    html = html.replace(/<h2[^>]*>/g, '\\n\\n## ').replace(/<\/h2>/g, '');
    html = html.replace(/<h3[^>]*>/g, '\\n\\n### ').replace(/<\/h3>/g, '');
    // 列表
    html = html.replace(/<ul[^>]*>/g, '\\n').replace(/<\/ul>/g, '');
    html = html.replace(/<li[^>]*>/g, '- ').replace(/<\/li>/g, '\\n');
    // 清理无用标签
    html = html.replace(/<span[^>]*>/g, '').replace(/<\/span>/g, '');
    html = html.replace(/<div[^>]*>/g, '').replace(/<\/div>/g, '');
    html = html.replace(/<a[^>]*>(.*?)<\/a>/g, '$1');
    html = html.replace(/<code[^>]*>/g, '`').replace(/<\/code>/g, '`');
    html = html.replace(/<pre[^>]*>/g, '\\n```\\n').replace(/<\/pre>/g, '\\n```\\n');
    // 清理 DeepSeek 引用标记
    html = html.replace(/-\d+/g, '');
    // 合并多余空行
    html = html.replace(/\\n{4,}/g, '\\n\\n').trim();
    if (html) parts.push(html);
  }
}

// 合并所有部分，去重
let result = parts.join('\\n\\n');
let lines = result.split('\\n');
let unique = [];
let seen = new Set();
for (let line of lines) {
  let trimmed = line.trim();
  if (trimmed && !seen.has(trimmed)) {
    seen.add(trimmed);
    unique.push(line);
  } else if (!trimmed) {
    unique.push(line);
  }
}
result = unique.join('\\n').replace(/\\n{4,}/g, '\\n\\n').trim();
result;
```

## 使用方式（CDP 模式）

在 CDP 标签页中使用 `browser_cdp` + `target_id` 执行：

```python
extraction_js = "<上面的 JS 代码，缩成一行的版本>"
result = browser_cdp(method="Runtime.evaluate", params={
    "expression": extraction_js,
    "returnByValue": True
}, target_id=ds_tab_id)
answer_text = result["result"]["value"]
```

### 快速检查用（轮询时判断是否回复完成）

```python
check = browser_cdp(method="Runtime.evaluate", params={
    "expression": "document.querySelectorAll('[class*=\"ds-markdown\"]').length",
    "returnByValue": True
}, target_id=ds_tab_id)
```

然后将返回的 Markdown 原文直接回复给用户，不做任何修改或概括。
