---
name: debug-js-syntax-errors
description: Debug JavaScript syntax errors in single-file HTML apps with inline <script> tags, especially Chinese nested quotes
category: debugging
---
# Debug JavaScript Syntax Errors in Single-File HTML

## When
- Browser page shows nothing / functions undefined, but no obvious JS errors in editor
- A web app with inline `<script>` tags suddenly stops working
- Suspect the issue is inside a large inline JS block (~1000+ lines)

## How

### Step 1: Isolate the JS from HTML and check syntax
```bash
python3 -c "
with open('file.html','r') as f:
    c=f.read()
s=c.find('<script>')+len('<script>')
e=c.find('</script>')
with open('/tmp/extracted.js','w') as f:
    f.write(c[s:e])
"
node --check /tmp/extracted.js
```
If syntax errors exist, `node` will point to line/column.

### Step 2: Common culprits in inline HTML scripts
1. **Chinese/nested double quotes**: Chinese text like `俗称"熟石灰"` inside a JS double-quoted string — the `"` inside breaks the string. Fix: escape as `\"` or use single quotes for the inner Chinese terms.
2. **Unclosed strings**: Missing closing `"` on a string that spans multiple lines
3. **Trailing comma after last array item**: `{a:1,}` fails in some parsers

### Step 3: Find and fix all problematic nested quotes in string values
```python
import re

with open('file.html','r',encoding='utf-8') as f:
    content = f.read()

script_start = content.find('<script>') + len('<script>')
script_end = content.find('</script>')
js = content[script_start:script_end]
lines = js.split('\n')

def is_chinese(c):
    return bool(re.match(r'[\u4e00-\u9fff\uFF01-\uFF0C\u3001-\u3003\uFF08-\uFF0F]', c))

def fix_line(line):
    if 'desc:' not in line:
        return line
    desc_key_start = line.find('desc:"')
    if desc_key_start == -1:
        return line
    content_start = desc_key_start + len('desc:"')
    end_pattern = '", correct:'
    end_pos = line.find(end_pattern, content_start)
    if end_pos == -1:
        return line
    desc_content = line[content_start:end_pos]
    result = []
    i = 0
    while i < len(desc_content):
        c = desc_content[i]
        if c == '"':
            prev_is_cn = (i > 0) and is_chinese(desc_content[i-1])
            next_is_cn = (i+1 < len(desc_content)) and is_chinese(desc_content[i+1])
            if prev_is_cn or next_is_cn:
                result.append('\\"')
            else:
                result.append(c)
        else:
            result.append(c)
        i += 1
    fixed_desc_content = ''.join(result)
    return line[:content_start] + fixed_desc_content + line[end_pos:]

fixed_lines = []
for line in lines:
    stripped = line.strip()
    if not stripped or stripped.startswith('//'):
        fixed_lines.append(line)
        continue
    fixed_lines.append(fix_line(line))

fixed_js = '\n'.join(fixed_lines)
with open('/tmp/fixed.js','w') as f:
    f.write(fixed_js)
```

### Step 4: Verify fix
```bash
node --check /tmp/fixed.js
```

### Step 5: Write back to HTML
```python
with open('file.html','r',encoding='utf-8') as f:
    content = f.read()
script_start = content.find('<script>') + len('<script>')
script_end = content.find('</script>')
new_content = content[:script_start] + fixed_js + content[script_end:]
with open('file.html','w',encoding='utf-8') as f:
    f.write(new_content)
```

## Key Insight
When Chinese text with embedded double-quotes (e.g., `俗称\"熟石灰\"`) appears inside a JS double-quoted string value, the inner `\"` terminates the string prematurely, causing a syntax error that breaks the entire script. The browser console may not show clear errors for this case. Always use `node --check` on extracted JS as the first debugging step.

---

## Critical: `</script>` Inside Inline Script Blocks

**Symptom**: Browser console shows `"Unexpected token '<' at line undefined"` or `"Unexpected token '<'"`. `typeof someFunction` returns `"undefined"` even though the function exists in the HTML source. `new Function(scriptText)` also fails.

**Root Cause**: The HTML parser is a state machine with no JS string context. It matches the literal string `</script>` to close the current `<script>` block — even when that string appears inside a JS string literal or regex.

**Diagnosis**:
```bash
grep -n "</script>" target.html
```
Any occurrence INSIDE a `<script>...</script>` block breaks JS parsing.

**Fix**: Escape the closing tag wherever it appears in JS content:
- In string literals: `<\/script>` or `'<' + '/script>'`  
- In regex: `/<\/script>/`
- Or split the string: `"</" + "script>"`

**Prevention**: When embedding any data containing HTML/JS-sensitive characters in inline scripts, always escape `</script>`.

**⚠️ WARNING — Global string replacement breaks HTML parser**: Blind global replace of `</script>` → `<\/script>` is dangerous. The HTML parser is a **raw text matcher** — it has no concept of JavaScript context. If you do `sed 's|</script|<\/script|g'` and any of your external `<script src="...">` tags happen to be inside an unclosed inline `<script>` block (or if the replacement accidentally removes a needed `</script>`), the HTML parser will never see a script close tag, and EVERYTHING after the broken point will be rendered as text in the page. The `<\/script>` form only helps inside JavaScript **string literals** (where JS engine sees it), not in HTML attribute values or as a "markup escaping" technique.

**Safe replacement workflow**:
```bash
# Step 1: Map every <script and </script> position BEFORE making changes
python3 -c "
import re
with open('target.html','r') as f:
    content = f.read()
for m in re.finditer(r'<script|</script>', content):
    line_num = content[:m.start()].count('\n') + 1
    print(f'Position {m.start()}, Line {line_num}: {repr(content[m.start():m.start()+60])}')
"

# Step 2: Make surgical replacements only inside string/regex literals
# Use Python with proper range awareness, not blind sed

# Step 3: Verify structure AFTER replacement — check that every <script> has a matching </script> in the right order
python3 -c "
import re
with open('target.html','r') as f:
    content = f.read()
# Must have exactly one </script> closing the inline script before any <script src=> appear
script_tags = [(m.start(), m.group()) for m in re.finditer(r'<script|</script>', content)]
print(f'Found {len(script_tags)} script tags:')
for pos, tag in script_tags:
    print(f'  {repr(tag)} at pos {pos}')
"
```

**Why `new Function()` also fails**: It uses the JS engine to parse the text, but if the HTML source has already been corrupted by premature tag closure (or if you pass HTML-wrapped content), it can fail at the HTML level too.

**Real-world example from this codebase**:
- Extracted data blocks into external `.js` files, placed `<script src="chem-data.js">` tags at line 829
- Ran `sed -i 's|</script|<\/script|g'` to escape `</script>` in strings
- This transformed the external script tag's own `</script>` (which HTML needs to see) into `<\/script>` (which HTML ignores)
- Result: inline `<script>` at line 808 never closed → all code at lines 834–1589 rendered as visible text in the page
- Fix: manually restore the correct `</script>` / `<script>` structure with Python after sed
