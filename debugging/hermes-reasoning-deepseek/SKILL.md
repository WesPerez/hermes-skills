---
name: hermes-reasoning-deepseek
description: Diagnose and fix DeepSeek thinking/reasoning configuration in Hermes Agent — when reasoning_effort doesn't actually take effect because the code gates it behind OpenRouter
read_when:
  - Hermes responses don't show thinking/reasoning content with DeepSeek
  - User wants to enable DeepSeek thinking mode in Hermes
  - Debugging why agent.reasoning_effort config doesn't work
  - Understanding Hermes extra_body reasoning dispatch logic
---

# Hermes Reasoning with DeepSeek

## The Problem

In Hermes Agent, `agent.reasoning_effort` in `config.yaml` can be set but may not actually take effect for direct DeepSeek API calls (not routed through OpenRouter).

## Root Cause

The method `_supports_reasoning_extra_body()` in `run_agent.py` gates reasoning extra_body to OpenRouter-only (plus some special endpoints):

```python
def _supports_reasoning_extra_body(self) -> bool:
    if "nousresearch" in self._base_url_lower:
        return True
    if "ai-gateway.vercel.sh" in self._base_url_lower:
        return True
    # ... GitHub models ...
    if "openrouter" not in self._base_url_lower:
        return False  # ❌ Direct DeepSeek API hits this
    # ... remaining checks for model prefixes only on OpenRouter ...
```

When `base_url` is `https://api.deepseek.com` (direct, no OpenRouter), this returns `False`, so `extra_body["reasoning"]` is NEVER sent to DeepSeek.

## Check Current State

```bash
# Check if reasoning_effort is configured
grep -A2 "^agent:" ~/.hermes/config.yaml | grep reasoning_effort

# Check current provider and base_url
grep -A5 "^model:" ~/.hermes/config.yaml
```

## DeepSeek API Thinking Parameters

From DeepSeek official docs (https://api-docs.deepseek.com/):

**Without thinking (default)**: No `thinking` param → reasoner_content may be empty
```json
{
  "model": "deepseek-v4-pro",
  "messages": [...]
}
```

**With thinking enabled**:
```json
{
  "model": "deepseek-v4-pro",
  "messages": [...],
  "thinking": {"type": "enabled"},
  "reasoning_effort": "high"
}
```

DeepSeek accepts `thinking` field, NOT `reasoning` (which is OpenRouter's format).

DeepSeek returns thinking content in `reasoning_content` field (not `reasoning`).

## DeepSeek Model IDs (as of May 2026)

| Model | Status |
|-------|--------|
| `deepseek-v4-pro` | ✅ Current Pro |
| `deepseek-v4-flash` | ✅ Current Flash |
| `deepseek-chat` | ⚠️ Deprecating 2026/07/24 — **backend maps to `deepseek-v4-flash`** |
| `deepseek-reasoner` | ⚠️ Deprecating 2026/07/24 |

## ⚠️ Critical: deepseek-chat = deepseek-v4-flash on Backend

**DeepSeek's API silently maps `deepseek-chat` → `deepseek-v4-flash`.** When a request uses `model: "deepseek-chat"`, the response returns `model: "deepseek-v4-flash"`. Any code path that sends `deepseek-chat` is actually using the fast (flash) model.

Verified empirically 2026-05-07:
```bash
curl -s https://api.deepseek.com/v1/chat/completions \
  -H "Authorization: Bearer $DEEPSEEK_API_KEY" \
  -d '{"model":"deepseek-chat","messages":[...]}' | jq .model
# Returns: "deepseek-v4-flash"  ← NOT "deepseek-chat"!
```

### The Silent Downgrade Bug in Hermes < v0.12.0

In Hermes earlier versions, `hermes_cli/model_normalize.py` had `_normalize_for_deepseek()` only recognizing `deepseek-chat` and `deepseek-reasoner`. The normalization chain was:

1. Config: `model.default: deepseek-v4-pro`
2. `normalize_model_for_provider("deepseek-v4-pro", "deepseek")` → not in canonical set → returns `"deepseek-chat"`
3. Agent sends `model: "deepseek-chat"` to API
4. DeepSeek backend maps to `deepseek-v4-flash` 🚨

**Result: User pays for v4-pro, gets v4-flash. Silent — no errors.**

### Fix in v0.12.0+

Hermes v0.12.0 added `deepseek-v4-pro` and `deepseek-v4-flash` to canonical models plus a regex `_DEEPSEEK_V_SERIES_RE` for future v5/v6 and dated variants like `deepseek-v4-flash-20260423`.

### Verify

```bash
hermes --version  # must be >= 0.12.0
python3 -c "from hermes_cli.model_normalize import normalize_model_for_provider; print(normalize_model_for_provider('deepseek-v4-pro', 'deepseek'))"
# Should print: deepseek-v4-pro (NOT deepseek-chat)

grep 'Auxiliary auto-detect' ~/.hermes/logs/agent.log | tail -3
# Should show: using main provider deepseek (deepseek-v4-pro)
```

### Manual Patch for < 0.12.0

```python
_DEEPSEEK_CANONICAL_MODELS = frozenset({
    "deepseek-chat", "deepseek-reasoner",
    "deepseek-v4-flash", "deepseek-v4-pro",  # ← ADD
})
```

## Test DeepSeek API Directly

```bash
# Without thinking
curl -s https://api.deepseek.com/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${DEEPSEEK_API_KEY}" \
  -d '{
    "model": "deepseek-v4-pro",
    "messages": [{"role": "user", "content": "1+1="}],
    "max_tokens": 50
  }' | python3 -c "
import sys, json
d = json.load(sys.stdin)
msg = d['choices'][0]['message']
print('content:', msg.get('content','')[:200])
print('reasoning_content:', msg.get('reasoning_content','NONE')[:200] if msg.get('reasoning_content') else 'NONE')
print('reasoning:', msg.get('reasoning','NONE')[:200] if msg.get('reasoning') else 'NONE')
"

# With thinking
curl -s https://api.deepseek.com/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${DEEPSEEK_API_KEY}" \
  -d '{
    "model": "deepseek-v4-pro",
    "messages": [{"role": "user", "content": "1+1="}],
    "max_tokens": 50,
    "thinking": {"type": "enabled"},
    "reasoning_effort": "high"
  }' | python3 -c "
import sys, json
d = json.load(sys.stdin)
msg = d['choices'][0]['message']
print('content:', msg.get('content','')[:200])
print('reasoning_content:', msg.get('reasoning_content','')[:200] if msg.get('reasoning_content') else 'NONE')
print('reasoning:', msg.get('reasoning','')[:200] if msg.get('reasoning') else 'NONE')
"
```

## v0.12.0+ Status: Built-In

**As of Hermes v0.12.0 (2026.4.30), DeepSeek thinking mode is handled natively through message-building code paths, not through `extra_body` dispatch.** The `_supports_reasoning_extra_body()` function remains OpenRouter-only, but DeepSeek thinking works because:

1. **`_build_assistant_message()`** — Adds `reasoning_content=" "` (space placeholder) on tool-call turns when `_needs_thinking_reasoning_pad()` returns True (which detects DeepSeek direct API).

2. **`_copy_reasoning_content_for_api()`** — Correctly echoes `reasoning_content` back on replay turns, handles empty-string→space upgrade, cross-provider poisoned history detection.

3. **DeepSeek v4-pro is inherently a thinking model** — It always returns `reasoning_content` without needing `thinking: {type: "enabled"}` sent.

### Verify in 0.12.0+

```bash
# Check version
hermes --version  # Should show >= v0.12.0

# Verify agent.log shows auxiliary using v4-pro (not deepseek-chat)
grep "Auxiliary auto-detect" ~/.hermes/logs/agent.log | tail -3
# Should show: "using main provider deepseek (deepseek-v4-pro)"
```

### If You're Still on < 0.12.0

Upgrade first — the model mapping bug that maps `deepseek-v4-pro` → `deepseek-chat` is also fixed in 0.12.0. See `references/hermes-upgrade-workflow.md` in the `hermes-agent` skill.

### Legacy Fixes (for v0.9.0 and earlier)

#### Option A: Route DeepSeek Through OpenRouter

```yaml
model:
  provider: openrouter
  default: deepseek/deepseek-v4-pro
```

#### Option B: Modify Source (`_supports_reasoning_extra_body`)

Edit `_supports_reasoning_extra_body()` in `run_agent.py`:
```python
if "api.deepseek.com" in self._base_url_lower:
    return True
```

#### Option C: Set agent.reasoning_effort (for OpenRouter path)

```bash
hermes config set agent.reasoning_effort high
```

Valid levels: `none`, `minimal`, `low`, `medium`, `high`, `xhigh`

## Note on display.show_reasoning

To see reasoning content in Hermes output, also set:
```yaml
display:
  show_reasoning: true
```

Without this, reasoning content is extracted but not displayed to the user in CLI mode.
