---
name: colg-hotlist
description: 用户消息以"colg"开头时，查看 COLG 论坛社区热榜，提取热帖列表并截图发送。
triggers:
  - colg 开头的问题
  - COLG热榜
  - 论坛热榜
---

# COLG论坛热榜

## 前置条件

1. Edge 浏览器已开启远程调试（`--remote-debugging-port=9222`）
2. `~/.agent-browser/config.json` 配置了 `{"cdp": "9222", "defaultTimeout": 60000}`
3. 已登录 COLG（Cookie 有效）

## 快速验证

```bash
agent-browser --cdp 9222 open "https://bbs.colg.cn/"
```

## 工作流程

> ⚠️ **关键**：必须使用 Hermes 内置浏览器工具（`browser_navigate` / `browser_snapshot` / `browser_console` / `browser_vision`），不要用 `terminal` + `agent-browser` CLI。截图必须用 `browser_vision` 并以 `MEDIA:` 格式发送给用户，否则用户看不到图片。

### 步骤1：打开论坛主页

使用 Hermes `browser_navigate`（自动携带 Cookie）：
```
browser_navigate "https://bbs.colg.cn/"
```

### 步骤2：验证登录状态

在 browser_console 中执行：
```js
document.cookie.match(/dz_username=([^;]+)/)?.[1]
```
如果显示用户名，说明已登录。

### 步骤3：提取首页热榜（主要方法，实测稳定）

热榜数据在首页 DOM 里直接可用，带有 `click_from=hotchart` 参数的链接就是热帖。无需任何弹窗交互：

```js
// 使用 Hermes browser_console 执行
(function() {
  var links = document.querySelectorAll('a[href*="click_from=hotchart"]');
  var results = [];
  for(var i=0; i<Math.min(links.length, 20); i++) {
    var href = links[i].href;
    var tid_match = href.match(/tid=(\d+)/);
    var tid = tid_match ? tid_match[1] : 'unknown';
    var title = links[i].innerText.trim();
    if(title) results.push((results.length+1) + '. [' + tid + '] ' + title);
  }
  return results.length > 0 ? results.join('\n') : '热榜为空，请确保已登录且在首页';
})()
```

**注意**：首页 DOM 里直接有热榜数据，**不需要点击"VP模拟器 HOT"弹窗**。那是侧边栏工具，`click_from=hotchart` 链接才是真正的热榜帖子。

### 步骤4备用：论坛热帖排序页

> ⚠️ 经验性：此页面经常为空（返回"热榜为空"），实际以首页 `click_from=hotchart` 为准。

如果首页热榜为空，访问论坛版块热帖排序页：

```
https://bbs.colg.cn/forum.php?mod=forumdisplay&fid=171&filter=heat&orderby=heats
```

提取方式：
```js
// Hermes browser_console 执行
(function() {
  var threads = document.querySelectorAll('tbody th');
  var results = [];
  for(var i=0; i<Math.min(threads.length, 20); i++) {
    var el = threads[i];
    var link = el.querySelector('a.xst') || el.querySelector('a[id*="thread"]');
    if(link) {
      results.push((i+1) + '. ' + link.innerText.trim());
    }
  }
  return results.join('\n');
})()
```

### 步骤5：截图

使用 `browser_vision` 截取页面，并直接发送 `MEDIA:截图路径` 给用户：

```
browser_vision question="截取 COLG 首页热榜区域的截图"
```

在回复中附带截图：
```
MEDIA:/tmp/colg_hotlist.png
```

> ⚠️ 不要用 `agent-browser screenshot` + 文字路径发送，用户看不到图片，必须用 `browser_vision` + `MEDIA:` 格式。

## Cookie 配置说明

COLG 关键 Cookie：
- `6KaR_be18_auth` — 登录凭证
- `6KaR_be18_saltkey` — Session Key
- `6KaR_be18_sid` — Session ID
- `6KaR_be18_dz_uid` — 用户ID
- `6KaR_be18_dz_ip` — 必须是本机IP（如 120.239.77.205）

Cookie 存储在 `cookies.json`，通过 `agent-browser cookies set` 命令注入。

## 依赖

- `agent-browser`（已配置 `--cdp 9222`）
- Edge 浏览器（端口 9222）
- 有效 COLG Cookie

## 优先级说明

本技能属于前缀触发（`colg`），优先级高于通用搜索技能 `web-access`。
用户消息以 `colg` 开头时直接走本技能，不走通用联网搜索流程。
