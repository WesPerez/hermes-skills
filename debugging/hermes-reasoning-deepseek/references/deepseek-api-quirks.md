# DeepSeek API Quirks (verified empirically 2026-05-07)

## Endpoint: GET /v1/models

Only returns 2 models:
```json
{"data": [
  {"id": "deepseek-v4-flash", "owned_by": "deepseek"},
  {"id": "deepseek-v4-pro", "owned_by": "deepseek"}
]}
```

## Silent Model Remapping

| Request | Response `model` | Actual behavior |
|---------|-----------------|-----------------|
| `deepseek-v4-pro` | `deepseek-v4-pro` | Pro model ✅ |
| `deepseek-v4-flash` | `deepseek-v4-flash` | Flash model ✅ |
| `deepseek-chat` | `deepseek-v4-flash` | **Flash model** ⚠️ |
| `deepseek-reasoner` | (not tested) | Presumed R1 |

Any `deepseek-chat` request is silently served by `deepseek-v4-flash`.

## Reasoning Content

- `deepseek-v4-pro`: Always returns `reasoning_content` in response (inherent thinking model)
- `deepseek-v4-flash`: May not return reasoning_content
- No need to send `thinking: {type: "enabled"}` — v4-pro thinks by default

## API Format

OpenAI-compatible Chat Completions API:
```
POST https://api.deepseek.com/v1/chat/completions
Authorization: Bearer $DEEPSEEK_API_KEY
Content-Type: application/json
```

Endpoint: `api.deepseek.com` → CDN behind `cloudfront.net`

## Model Normalization Chain (Hermes)

For Hermes with `model.provider: deepseek`:

1. Config: `model.default: <value>`
2. Runtime resolves provider as `deepseek`
3. `normalize_model_for_provider(<value>, "deepseek")` called
4. `_normalize_for_deepseek()` checks canonical set + V-series regex
5. Result sent to API as `model` field

**Pre-0.12.0**: Missing V-series support → `deepseek-v4-pro` → `deepseek-chat` → `deepseek-v4-flash` on backend
**0.12.0+**: V-series regex `^deepseek-v\d+([-.].+)?$` catches all v4/v5/v6 variants
