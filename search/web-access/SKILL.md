---
name: web-access
description: 所有联网操作必须通过此技能处理，包括搜索、网页抓取、登录后操作等。三层通道调度：搜索发现→内容抓取→CDP浏览器。参考 eze-is/web-access v2.5.0（6.1k stars）设计哲学。
tags:
  - search
  - web
  - browser
  - cdp
  - fetch
triggers:
  - 搜索信息
  - 查看网页
  - 联网操作
  - 查询资料
---

# Web Access — 联网搜索三层通道

## 优先级规则（重要！）

所有联网操作按以下优先级执行，由轻到重：

```
Tier 1  搜索发现  →  浏览器打开 Google/百度搜索关键词
Tier 2  内容抓取  →  Jina.ai (r.jina.ai/URL) 或 curl 抓取已知URL
Tier 3  CDP浏览器  →  Edge CDP 9222 直接操作交互式页面
```

### 详细决策树

| 场景 | 走哪层 | 具体方法 |
|------|--------|---------|
| 需要搜索某主题，无已知URL | Tier 1 → Tier 3 | 浏览器开 Google 搜索，看结果摘要 |
| 有已知URL，抓取正文 | Tier 2 | `r.jina.ai/URL` 转 Markdown（20 RPM免费，省token） |
| 需要页面原始HTML/meta | Tier 2 | `curl URL` |
| 需要登录后内容 | Tier 3 | Edge CDP 直接访问（已登录） |
| JS动态渲染页面 | Tier 3 | Edge CDP 导航+截图 |
| 小红书/微博等反爬平台 | Tier 3 | 直接 CDP，跳过静态层 |
| 用户指定 `ds/DS` 开头 | 走 ds-expert | DeepSeek 专家模式 |
| 用户指定 `db` 开头 | 走 doubao-image-gen | 豆包图像生成 |
| 用户指定 `colg` 开头 | 走 colg-hotlist | COLG热榜 |

### 为什么不跳过步骤

- Tier 1 和 Tier 2 比 CDP 浏览器 **快得多、省token**，不需要每次都开浏览器
- 只有 Tier 1/2 拿不到内容才走 Tier 3（登录态、JS渲染、反爬）
- **Jina.ai** 不需要 API Key，直接 `r.jina.ai/https://example.com` 即可，限 20 RPM

**像人一样思考，兼顾高效与适应性。** 不依赖固化的步骤规划，而是带着目标进入，边看边判断，遇到阻碍就解决。

### 四个步骤

1. **拿到请求** — 明确用户要什么，定义成功标准
2. **选择起点** — 根据任务性质选最可能直达的方式作为第一步
3. **过程校验** — 每一步的结果对照成功标准，方向错了立即调整，不在同一方式上反复重试
4. **完成判断** — 对照成功标准确认完成后停止，不过度操作

## 三层通道选择

| 场景 | 工具 | 说明 |
|------|------|------|
| 搜索摘要或关键词结果，发现信息来源 | **WebSearch** | 搜索引擎 API 快速定位 |
| URL 已知，需要从页面定向提取信息 | **WebFetch** | 拉取网页内容，转 Markdown 省 token |
| 需要原始 HTML 源码（meta、JSON-LD 等） | **curl** | 直接获取原始 HTML |
| 非公开内容，反爬严格的平台（小红书、公众号等） | **CDP 浏览器** | 跳过静态层，直接浏览器渲染 |
| 需要登录态、交互操作、自由导航探索 | **CDP 浏览器** | 像人一样操作页面 |

Hermes 自带浏览器工具（`browser_navigate` / `browser_snapshot` / `browser_console`）通过 Edge CDP 9222 工作。

## 浏览器操作技巧

- **程序化方式**（构造 URL 直接导航、eval 操作 DOM）：速度快但可能触发反爬
- **GUI 交互**（点击按钮、填输入框、滚动）：速度慢但确定性最高，不会被限制
- 根据对目标平台的了解灵活选择，GUI 交互也可为程序化方式提供探测依据
- 站点内交互产生的链接天然携带平台所需的完整上下文，比手动构造的 URL 更可靠

## 图片防盗链处理

通用方案 — 加 Referer 头：
```python
headers = {
    'Referer': 'https://www.doubao.com/',  # 替换为实际网站
    'User-Agent': 'Mozilla/5.0 ...'
}
requests.get(url, headers=headers)
```

## 一手信息优先

| 信息类型 | 一手来源 |
|---------|---------|
| 政策/法规 | 发布机构官网 |
| 企业公告 | 公司官方新闻页 |
| 学术声明 | 原始论文/机构官网 |
| 工具能力/用法 | 官方文档、源码 |

找不到官网时，权威媒体的原创报道（非转载）可作为次级依据，但需向用户说明来源不确定性。

## 媒体资源提取

- 判断内容在图片里时，用 JS 从 DOM 直接拿图片 URL，比全页截图精准
- 滚动到底部触发懒加载，使图片完成加载
- 视频内容：操控 `<video>` 元素获取时长、seek 到任意时间点采帧

## 参考

- `eze-is/web-access` GitHub 仓库（6.1k stars）：https://github.com/eze-is/web-access
- 设计详解：https://www.eze.is/blog/web-access-skill
