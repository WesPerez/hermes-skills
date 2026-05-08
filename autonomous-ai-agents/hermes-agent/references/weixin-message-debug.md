# WeChat (Weixin) Message Truncation Debugging

Diagnosis and fixes for the problem where Hermes Agent's replies to WeChat
suddenly stop mid-sentence when the response is long.

## Root Cause 1: Byte-length vs Character-length (Primary)

**iLink Bot API (`sendmessage`) enforces a 2000-byte limit on the `text` field,
but Hermes was using Python `len()` which counts Unicode characters, not bytes.**

Chinese characters in UTF-8 are 3 bytes each. A message with 700 Chinese
characters = 2100 bytes → exceeds API limit → API rejects the chunk silently
→ the user sees only the previously-successful chunks → "suddenly stops talking."

**The fix in `gateway/platforms/weixin.py`:**

1. Added `_utf8_byte_len(text)` that returns `len(text.encode("utf-8"))`
2. Changed all `len()` comparisons in `_pack_markdown_blocks_for_weixin` and
   `_split_text_for_weixin_delivery` to use `_utf8_byte_len`
3. Passed `len_fn=_utf8_byte_len` to `BasePlatformAdapter.truncate_message`

The `MAX_MESSAGE_LENGTH = 2000` constant remains the same value but is now
interpreted as UTF-8 bytes (not characters). A comment explains the rationale.

## Root Cause 2: Chunk Failure Stops Remaining Delivery (Secondary)

When one chunk failed (e.g. due to byte-limit rejection), the `send()` method's
outer `try/except` caught the exception and returned `SendResult(success=False)`,
skipping all remaining chunks. The already-delivered successful chunks were
already visible to the user, creating the appearance of an interrupted reply.

**The fix in `gateway/platforms/weixin.py` `send()` method:**

Wrapped each chunk send in an inner `try/except` so a single chunk failure
logs a warning but does NOT abort the remaining chunk delivery:

```python
for idx, chunk in enumerate(chunks):
    try:
        await self._send_text_chunk(...)
    except Exception as exc:
        logger.warning("chunk %d/%d failed, continuing: %s", idx+1, len(chunks), exc)
```

## Diagnosis Steps

When a user reports "messages suddenly stop mid-reply":

1. **Check gateway logs**: `grep "failed\|error\|rate limit" ~/.hermes/logs/gateway.log`
2. **Look for "chunk" failures**: if you see chunk log messages, the byte-length fix is needed
3. **Reproduce with known-long Chinese text**: Send a test message with ~700+ Chinese characters
4. **Check if chunks are under 2000 bytes**: Use `_utf8_byte_len()` on each chunk

## Configurable Parameters

These env vars / config keys affect message delivery reliability for WeChat:

| Parameter | Env Variable | Default | Notes |
|-----------|-------------|---------|-------|
| Chunk delay | `WEIXIN_SEND_CHUNK_DELAY_SECONDS` | 1.5 | Seconds between chunks. Increase if rate-limited. |
| Chunk retries | `WEIXIN_SEND_CHUNK_RETRIES` | 4 | Retries per chunk on failure |
| Retry delay | `WEIXIN_SEND_CHUNK_RETRY_DELAY_SECONDS` | 1.0 | Base backoff (rate-limit uses 3x) |
| Split multiline | `WEIXIN_SPLIT_MULTILINE_MESSAGES` | false | Split by line breaks (legacy) |

## Key Files

- `gateway/platforms/weixin.py` — main adapter with `MAX_MESSAGE_LENGTH`, `_send_text_chunk()`, `send()`
- `gateway/platforms/base.py` — `BasePlatformAdapter.truncate_message()` with `len_fn` support
- `gateway/stream_consumer.py` — streaming delivery path, uses `MAX_MESSAGE_LENGTH` for fallback
- `tests/gateway/test_weixin.py` — unit tests for split behavior

## Verification

```python
# Test that byte-length splitting works
from gateway.platforms.weixin import (
    _utf8_byte_len, _split_text_for_weixin_delivery
)

# Chinese text over the byte limit
text = "你好" * 350  # 700 chars, 2100 bytes
chunks = _split_text_for_weixin_delivery(text, 2000)
assert all(_utf8_byte_len(c) <= 2000 for c in chunks)
assert len(chunks) >= 2  # must split at least once
```
