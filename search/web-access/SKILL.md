---
name: web-access
description: 所有联网操作必须通过此 skill 处理，包括搜索、网页抓取、登录后操作等。三层通道调度：搜索发现→内容抓取→CDP浏览器。
tags:
  - search
  - web
  - browser
  - cdp
  - fetch
  - mandatory
triggers:
  - 搜索信息
  - 查看网页
  - 联网操作
  - 查询资料
  - 任何需要联网的任务
---

# Web Access — 联网搜索三层通道

## ⚠️ 强制加载规则

**任何联网搜索、查资料、看网页的任务，必须先加载本技能，不得跳过。**
Hermes 没有 `web_search` API 工具，所有联网操作必须经过本技能的三层通道。
前缀触发（`ds`/`db`/`colg`）走对应技能，除此之外**一律走 web-access**。

## 三层通道详解

```
Tier 1 🟢  Tavily API     → tavily-search 命令（~1s，不开浏览器）
Tier 2 🟡  内容抓取       → curl r.jina.ai/URL 或 curl URL（轻量）
Tier 3 🔴  CDP 浏览器     → Edge CDP 9222（登录态/JS渲染/反爬）
```

### 各层能力对比

| 层级 | 方法 | 速度 | 开浏览器？ | 适合场景 |
|------|------|------|-----------|---------|
| Tier 1 🆕 | `tavily-search <query>` | **快（~1s）** | ❌ **不开浏览器** | 搜索关键词、发现来源、自带AI摘要 |
| Tier 2 | `curl r.jina.ai/URL` | 快 | ❌ 不开浏览器 | 已知URL抓正文 |
| Tier 2 | `curl URL` | 快 | ❌ 不开浏览器 | 拿原始HTML/meta |
| Tier 3 | CDP `browser_*` 工具集 | 慢 | ✅ 全面操作 | 登录态、JS渲染、反爬网站 |

### 关键理解

**Tavily 是最快最省的方式**：命令行调用，1 秒返回结构化搜索结果 + AI 摘要。不需要开浏览器，也没有 API 限额限制（当前 key 免费 1000 次/月，日常够用）。

只有 Tavily 搜不到相关内容时，才降级到 Tier 2（Jina.ai 抓正文）和 Tier 3（CDP 浏览器）。不要跳过 Tavily 直接开浏览器搜 Google。

### 详细决策树

| 场景 | 走哪层 | 具体方法 |
|------|--------|---------|
| 需要搜索某主题，无已知URL | **Tier 1** | `tavily-search <关键词>`（最快，~1s） |
| 有已知URL，抓取正文 | Tier 2 | `r.jina.ai/URL` 转 Markdown（20 RPM免费） |
| 需要页面原始HTML/meta | Tier 2 | `curl URL` |
| Tavily 搜不到相关内容 | 降级 | Tier 1 → Tier 2/3 |
| 需要登录后内容 | Tier 3 | Edge CDP 直接访问（已登录） |
| JS动态渲染页面 | Tier 3 | Edge CDP 导航+截图 |
| 小红书/微博等反爬平台 | Tier 3 | 直接 CDP，跳过静态层 |
| 用户指定 `ds/DS` 开头 | 走 ds-expert | DeepSeek 专家模式 |
| 用户指定 `db` 开头 | 走 doubao-image-gen | 豆包图像生成 |
| 用户指定 `colg` 开头 | 走 colg-hotlist | COLG热榜 |

### 为什么不跳过步骤

- **Tier 1（Tavily）比开浏览器快 10 倍**，1 秒返回，不用等页面加载
- Tier 2（Jina.ai）适合在已知 URL 时抓取详细内容，免费但限 20 RPM
- 只有 Tier 1/2 都拿不到才走 Tier 3（登录态、JS渲染、反爬）
- **Tavily 自带 AI 摘要**，很多时候直接就能给用户答案，不需要额外步骤

**像人一样思考，兼顾高效与适应性。** 不依赖固化的步骤规划，而是带着目标进入，边看边判断，遇到阻碍就解决。

### 四个步骤

1. **拿到请求** — 明确用户要什么，定义成功标准
2. **选择起点** — 根据任务性质选最可能直达的方式作为第一步
3. **过程校验** — 每一步的结果对照成功标准，方向错了立即调整，不在同一方式上反复重试
4. **完成判断** — 对照成功标准确认完成后停止，不过度操作

## 三层通道选择

| 场景 | 工具 | 说明 |
|------|------|------|
| 搜索关键词，发现信息来源 | **Tavily API** 🆕 | `tavily-search <query>`，1秒返回，不开浏览器 |
| URL 已知，需要从页面定向提取信息 | **Jina.ai / curl** | `r.jina.ai/URL` 转 Markdown 省 token |
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
- **Tavily API**：https://tavily.com （免费 1000 次/月，已安装 `tavily-search` CLI）
- **Jina.ai**：`r.jina.ai/URL` 免费转换网页为 Markdown

## 用户偏好：行动优先（严格执行）

**以下三条是用户反复强调的偏好，违反会被批评：**

1. **先做，再解释**：搜索任务直接给出答案，不要在回答中详细说明搜索过程或用哪个技能。结论+一行理由就够。❌「我用Tavily搜到了x个结果...」 → ✅ `> 答案内容`
2. **分析压缩**：涉及多方案对比时，先给最优结论+一句话理由，具体细节作为可选补充。
3. **用户不满意「沉默」**：长时间操作（如 CDP 浏览器等待）时先给阶段性回应（"查到了，正在整理"），不要闷头操作到完才回复。

## 已知陷阱

1. **加载本技能后再用 Tavily**：`tavily-search` 是终端命令（不是 Hermes 原生工具），调用前必须先 `skill_view('web-access')` 获取完整三层通道知识。直接用 terminal 调 Tavily 没问题，但跳过了技能你就忘了还有 Jina.ai 和 CDP 兜底。**必先 `skill_view('web-access')` 再执行。**
2. **Reddit 拦 Jina.ai**：Jina 抓 Reddit 返回 403，需降级到 CDP 浏览器直接访问。
3. **Jina 20 RPM 限额**：频繁抓取可能触发，间隔至少 3 秒。
4. **Tier 1 就够时不要进 Tier 2**：Google 摘要内容足够回答时直接回，不需要额外抓取确认。
5. **CDP 浏览器不是默认选项**：只有 Tier 1/2 都失败或需要登录态时再用。
6. **前缀触发比通用搜索优先**：用户说 `ds`/`db`/`colg` 时走专门技能，不走 web-access。

## 参考文件

- `references/site-compatibility.md` — 各网站对 Jina.ai/curl/CDP 的兼容性备忘
