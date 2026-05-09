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

# 3. 创建新标签页
new_tab = browser_cdp(method="Target.createTarget", params={"url": "about:blank"})
```

> ⚠️ 每次操作都用新标签页，任务完成后关闭。不要复用旧标签页。

## 工作流程

### 步骤1：导航到豆包

```python
browser_navigate(url="https://www.doubao.com/chat/")
```

检测是否已登录（右上角显示用户名表示已登录）。如果显示登录页面，通知用户通过 VNC 登录。

### 步骤2：进入图像生成模式

在页面底部工具栏找到\"图像生成\"按钮并点击。点击后会出现图像生成专属界面，包含专属输入框（placeholder=\"描述你想要的图片\"）。

### 步骤3：设置比例

在图像生成工具栏中找到\"比例\"按钮并点击，从弹出选项中选择 3:4（竖版）。这一步必须在输入提示词之前完成，否则比例设置不生效。

| 比例 | 适用场景 |
|------|---------|
| 1:1 方形 | 头像、表情包 |
| **3:4 竖版** | **默认推荐，适合手机壁纸、小红书封面、表情包** |
| 16:9 横版 | PC 壁纸、横版海报 |

### 步骤4：输入提示词

```python
browser_type(ref="<输入框ref>", text="<提示词>")
```

#### 提示词构图要求（必加）
在提示词末尾加入以下构图要求，避免主体被裁切：
```
画面上方留出约30像素的空间，主体放在中下部
```

#### 多风格提示词指南
| 风格 | 关键词 |
|------|--------|
| 写实/摄影 | 摄影级、写实风格、电影感、8K 细节 |
| 二次元/动漫 | 二次元、动漫风、赛博朋克、厚涂 |
| 插画/手绘 | 插画风格、水彩风、手绘感 |
| 国风 | 水墨画、国风、工笔画 |
| Q版/可爱 | Q版、可爱风格、两头身 |
| 极简 | 极简风格、扁平化、干净线条 |

### 步骤5：点击发送按钮

输入文字后，在输入框右侧找发送按钮（通常是一个箭头图标），用 `browser_click` 点击。

> ⚠️ **不要按 Enter**：实测按 Enter 会导致页面重置回到首页，必须点击发送按钮。

### 步骤6：等待生成

生成状态消息会显示类似\"正在为您生成...~\"。豆包生成时间：第一张约 **15-20 秒**，全部 4 张约 **25 秒**。

轮询检测策略（每 5 秒检查一次，最多 12 次）：
```python
browser_console(expression="document.querySelectorAll('img[src*=\"rc_gen_image\"]').length")
```
当出现 4 张图片时表示生成完成。

### 步骤7：获取图片 URL

```python
urls = browser_console(expression="""JSON.stringify(Array.from(document.querySelectorAll('img'))
  .filter(img => img.src.includes('rc_gen_image'))
  .map(img => img.src))""")
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
6. **页面改版**：豆包经常更新 UI，所有 ref ID 每次加载可能不同。用 placeholder 文本（"描述你想要的图片"、"发消息..."）和按钮文本（"图像生成"、"比例"）作为识别依据。

## 参考文件

- `references/test-session-2026-05-08.md` — 实测流程记录，包含 URL 格式、尺寸数据、COLG 参考帖子等细节

## 优先级说明

本技能属于前缀触发（`db`），优先级高于通用搜索技能 `web-access`。
用户消息以 `db` 开头时直接走本技能生图，不走联网搜索流程。
