---
name: html-game-timer-debug
description: Debug double-counting bugs in single-file HTML games where timer callbacks fire after answer is already submitted, inflating wrong-answer counts.
---

# HTML Game Timer Bug Debugging

## Context
Single-file HTML games with multiple game modes often have bugs where wrong answers are double-counted. The root cause is usually **timer state not being cleared** when the user answers before the timer fires.

## Key Pattern
```js
// Game structure with timer:
startTimer(seconds, () => handleTimeout())  // sets timerId
// User clicks answer → handleClick()
// handleClick calls hideTimer() but may miss clearing the specific timerId
// → handleTimeout() still fires later → scoreWrong++ again!
```

## Debugging Workflow

### Step 1: Confirm the score logic is correct
Use browser console to directly inspect and manipulate game state:
```js
scoreRight  // current right count
scoreWrong  // current wrong count
answerLocked  // should be true after answer

// Force an answer:
submitEquation()  // or appropriate submit function
scoreRight  // verify incremented
scoreWrong  // verify NOT double-incremented
```

### Step 2: Find all timer-related variables
```js
// grep for timer declarations:
timerId, timerInterval, window.timerId, puzzState.timerId, symbolState.timerId, catState.timerId
```

### Step 3: Check hideTimer() clears ALL timer state
```js
function hideTimer() {
  if (timerInterval) { clearInterval(timerInterval); timerInterval = null; }
  if (window.timerId) { clearTimeout(window.timerId); window.timerId = null; }
  if (puzzState.timerId) { clearTimeout(puzzState.timerId); puzzState.timerId = null; }
  if (symbolState.timerId) { clearTimeout(symbolState.timerId); symbolState.timerId = null; }
  // Add any other timerId fields here
  document.getElementById('timerWrap').style.display = 'none';
}
```

### Step 4: Verify each game mode's timeout callback
Each `handleXxxTimeout()` function should:
1. Check `if (answerLocked) return;` at the very top
2. Set `answerLocked = true;` before doing anything else
3. Call `hideTimer()` to clear the timer

### Step 5: Check submit functions also clear their own timerId
```js
function submitSymbol(timeout = false) {
  if (answerLocked && !timeout) return;  // timeout=true bypasses lock
  // ...
  clearTimeout(symbolState.timerId);  // explicit clear in submit path too
}
```

## Verification Test
1. Play a game, deliberately timeout on 3 questions
2. Also answer 2 correctly
3. Verify final result: 答对=2, 答错=3, 总计=5
4. If 答错 shows 5+ (e.g., 5 instead of 3), timer is firing twice per timeout

## Prevention
When adding a new timer to any game mode, ALWAYS:
1. Declare the timerId in the state object (e.g., `symbolState.timerId = null`)
2. Clear it in `hideTimer()`
3. Check `if (answerLocked) return;` at top of timeout callback
4. Set `answerLocked = true;` in both submit and timeout paths

## Relevant File
`/var/www/html/chemistry-game.html` — has 4 game modes (match, equation, puzzle, symbol), all using shared `hideTimer()` + per-state `timerId` pattern.
