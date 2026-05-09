# 网站抓取兼容性备忘（2026-05-08）

## Jina.ai (r.jina.ai/URL)

免费 20 RPM，不需要 API Key，直接 `r.jina.ai/https://example.com`。

### ✅ 可用
- 普通博客/文档站
- 新闻网站
- 官方文档/API 文档

### ❌ 被拦（403）
- **Reddit** — 返回 403，Jina 爬虫被拦截

### 处理方式
- Jina 被拦 → 降级到 Tier 3（CDP 浏览器）
- 或用 curl 加 User-Agent 头尝试

## CDP 浏览器（Edge 9222）
兜底方案，能访问任何登录态/反爬网站。比 Jina/curl 慢但一定能拿到内容。
