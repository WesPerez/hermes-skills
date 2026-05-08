---
name: colg-reply
description: 用户消息以"colg"开头时，COLG论坛自动回复帖子。使用 Hermes 浏览器工具自动化回复。
triggers:
  - colg 开头的问题
  - colg回复
  - 回复帖子
---

# COLG论坛自动回复

使用 Hermes 内置浏览器工具自动回复 COLG 论坛帖子。

## 使用方法

发送以下指令触发：
- `colg回复 你好，支持一下！`

## 前置条件

- Edge 浏览器开启 9222 调试端口
- 已登录 COLG（Cookie 有效）

## 工作流程

### 步骤1：确认当前页面

```
browser_snapshot → 确认已在帖子页面
```

### 步骤2：截图确认（必须）

```
browser_vision → 截图当前页面，确认要回复的帖子
```

发送截图给用户确认是否在正确页面。**只有用户说"确定"才继续。**

### 步骤3：找到回复框

用 `browser_snapshot` 找到回复输入框的 ref ID。

典型位置：
- 页面底部回复表单
- 使用 `browser_click` 点击激活回复框

### 步骤4：输入回复内容

```
browser_type → 输入用户指定的回复内容
```

### 步骤5：截图预览（提交前必须）

```
browser_vision → 截图预览回复内容
```

发送截图给用户确认。**只有用户说"确定"才提交。**

### 步骤6：提交回复

找到并点击提交按钮：
```
browser_click → 提交按钮 ref
```

### 步骤7：确认提交结果

```
browser_vision → 确认回复已发出
```

## 注意事项

- 步骤2和步骤5必须截图确认
- 只有用户明确说"确定"才执行下一步
- 找不到位置时先截图询问用户
- 登录状态由 Cookie 维持，如遇登录问题参考 `colg-hotlist` 重新设置 Cookie

## 依赖

- Hermes `browser` 工具集
- Edge 浏览器开启 9222 调试端口
- COLG 有效登录 Cookie
