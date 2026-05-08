---
name: chemistry-game-debug
description: 化学游戏 chemistry-game.html 调试与修复手册
---

# 化学游戏 chemistry-game.html 调试与修复手册

## 部署信息
- 部署地址: http://43.159.168.34/chemistry-game.html
- 源文件: /var/www/html/chemistry-game.html
- 每次修改后用 `?t=时间戳` 强制刷新浏览器缓存

## 四个游戏模式
| 模式 | 变量前缀 | 关键函数 | 题库 |
|------|----------|----------|------|
| 化合物配对 | `matchState` | `startMatchGame()` | `COMPOUNDS` (49条) |
| 方程式配平 | `catState` | `startEquationGame()` | `EQUATIONS` (从外部JS加载) |
| 物质推断 | `puzzState` | `startPuzzleGame()` | `PUZZLES` (63条) |
| 符号速记 | `symbolState` | `startSymbolGame()` | `SYMBOLS` |

## 已知修复记录
1. `matchState.matched` 需为数组`[]`而非数字`0`
2. `catState.correctStreak` 有空格非法属性名 → `correctStreak`
3. `symbolState` 需 `gameActive` 标志防止退出后 toast 弹出
4. `puzzState.timerId` 未在 `clearAllTimers()` 中清除导致退出后超时 toast
5. `PUZZLES` 的 `desc` 含中文双引号导致 JS 语法错误，需 `\"` 转义
6. `showFeedback` 有两个函数定义（行1387方程式用/行1586符号速记用），第二个模板答对时也显示括号，方程式调用只传2个参数导致 `name=undefined` 显示"（undefined）"
7. **Bug 修复：跨模式 toast 残留** — `goHome()`/`quitGame()`/`startQuiz()` 未清除 toast 的 `.show` class
   - 修复：新增 `hideToast()` 函数（移除 `.toast` 的 `show` class），在三处入口/出口调用
   - 根因：`showToast()` 在 DOM 元素上设 `textContent` + 加 `show` class
   - 相关函数位置：`showToast()` 行1006, `hideToast()` 行977, `goHome()` 行985, `quitGame()` 行993, `startQuiz()` 行1074
8. **Bug 修复：符号速记反馈措辞错误** — 答对答错显示"配平正确！/配平错误！"
   - 修复：行1593 改为"回答正确！/回答错误！"
   - 根因：`showFeedback` 函数（4参数版，行1588）硬编码了方程式模式的文案

## Toast 系统解析
- `showToast(msg, type)` (行1006): 设 `textContent` + `className = 'toast ${type} show'`
- CSS 显示: `.toast.show { transform: translateX(-50%) translateY(0); }` (行459)
- 自动隐藏: `setTimeout(() => t.classList.remove('show'), 2000)` 内建在 showToast 中
- `hideToast()`: 直接 `t.classList.remove('show')`，视觉隐藏但 textContent 保留（下次 showToast 覆盖）
- **调试注意**: `browser_snapshot` 会抓到 toast 的文本即使 `.show` 已移除；判断隐藏不要看文本要看 className

## 预览动画参数（化合物配对）
- 每张卡翻开间隔: 300ms（`stepDelay = 300`）
- 全部翻完后停留: 2秒
- 翻回后恢复交互等待: 100ms
- 动画方式: 所有牌同时翻开 → 等停留时间 → 所有牌同时翻回

## 调试技巧
1. `browser_navigate` 后加 `?t=1` 强制刷新避免缓存
2. `browser_vision` 看到的HTML可能因缓存与JS执行结果不同步
3. 新代码已部署但浏览器仍显示旧内容 — 多次刷新或加时间戳
4. 直接在浏览器 console 调用函数测试: `startPuzzleGame()`, `submitEquation()`, `showResult()`
5. 检查函数是否定义: `typeof startQuiz` → 应返回 `"function"` 而非 `"undefined"`

## 关键函数位置
- `showFeedback()`: 行1387(方程式), 行1586(符号速记) — 第二个覆盖第一个
- `hideTimer()`: 清除 `window.timerId`, `timerInterval`, `puzzState.timerId`, `symbolState.timerId`
- `showToast(msg, type)`: 行1006 — 显示 toast（设 textContent + show class，2秒自动隐藏）
- `hideToast()`: 行977 — 移除 toast 的 show class（视觉隐藏）
- `handlePuzzleTimeout()`: 行1233, 超时调用 `scoreWrong++` 后 `hideTimer()` 再 `nextPuzzleOrEnd()`
- `submitSymbol(timeout)`: 行1470, `timeout=true` 时绕过 `answerLocked`
- `showResult()`: 行1614, 设置 `#resultWrong` 和 `#resultComment`
