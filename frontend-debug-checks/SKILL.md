---
name: frontend-debug-checks
description: Debug JS syntax errors in inline HTML script blocks, CSS specificity issues, and common silent failures
category: software-development
---

# Frontend Debug: JS Syntax Errors in Inline HTML

## Context
When editing inline JS inside HTML files, certain bugs are hard to catch by reading code because:
1. No build step = no compiler errors shown
2. One JS error = entire `<script>` block silently fails
3. All onclick handlers and function definitions break silently

## Bugs that silently break everything

### 1. Invalid property names with spaces
```javascript
// BROKEN - space in key name
let catState = { correct streak: 0 };

// FIXED
let catState = { correctStreak: 0 };
```
**Symptom**: Game mode buttons (`onclick="startQuiz('match')"`) do nothing. No errors in browser console visible to user.

### 2. Wrong type in state object
```javascript
// BROKEN - initialized as number, used as array
let matchState = { matched: 0 };
// ... later:
matchState.matched.push(c1.pairId); // TypeError!

// FIXED
let matchState = { matched: [] };
```

### 3. CSS specificity overrides
```css
/* BROKEN - #choicesGrid has display:block which overrides .choices-grid */
#choicesGrid { display: block; } /* specificity wins over class rule */
.choices-grid { display: grid; grid-template-columns: 1fr 1fr; }

/* FIXED - remove the ID rule, let class rule work */
#choicesGrid { /* no display rule */ }
```

## Debugging workflow
1. If clicking buttons does nothing → likely JS syntax error in `<script>`
2. Look for: spaces in object keys, wrong types, unclosed brackets
3. Check browser console: open DevTools → Console → look for red errors
4. Use `grep -n "broken pattern" file.html` to find suspicious lines

## Quick grep checks for common bugs
```bash
grep -n " [a-z]+ [a-z]+:" file.html   # spaces in object keys
grep -n "matched: 0\|flipped: 0" file.html  # number used as array
grep -n "#choicesGrid" file.html      # CSS specificity overrides
```

### 4. Unclosed array/object bracket (bracket depth mismatch)
**Symptom**: `startQuiz` is undefined. Clicking buttons does nothing. Browser console shows "Unexpected token 'const'" or similar syntax error.

```javascript
// BROKEN — COMPOUNDS array never closed, PUZZLES is parsed as inside COMPOUNDS
const COMPOUNDS = [
  {f:"H₂O", n:"水"},
  // ... hundreds of entries ...
  {f:"SO₂Cl₂", n:"硫酰氯"},
  // 推断题数据：描述 → 正确答案 + 干扰项  ← PUZZLES starts here but is inside unclosed COMPOUNDS
  const PUZZLES = [

// FIXED — close COMPOUNDS before PUZZLES
const COMPOUNDS = [
  {f:"H₂O", n:"水"},
  // ...
  {f:"SO₂Cl₂", n:"硫酰氯"},
];
const PUZZLES = [
```

**Diagnosis steps**:
1. Extract inline script to external `.js`: `grep -n '<script>' file.html` → byte position, then `dd if=file.html bs=1 skip=$((pos+8)) count=... of=file.js`
2. Run `node --check file.js` — directly pinpoints the syntax error line
3. Count bracket depth with a parser that respects string literals (brackets inside `'`, `"`, or backtick` strings don't count):
   - Write a Python script using `execute_code` to find the line where bracket depth first goes negative
4. Browser `textContent.length` vs byte length: if HTML reads all chars but `typeof startQuiz === 'undefined'` → JS parse failed, not HTML truncation

**Key insight**: A simple char-by-char bracket count is wrong — you must ignore brackets inside string literals. Example: `opts:["[","]"]` contains brackets but they're inside a string and don't affect nesting depth.
