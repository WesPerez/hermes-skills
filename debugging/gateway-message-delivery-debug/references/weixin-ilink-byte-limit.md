# WeChat iLink Byte-Limit Investigation

Root cause analysis for "messages cut off mid-sentence" on WeChat through Hermes gateway.

## Timeline

1. User reports: long replies "suddenly stop talking" mid-sentence
2. Investigation reveals `MAX_MESSAGE_LENGTH = 2000` in `weixin.py`
3. The value 2000 was used as **character count** (`len()` in Python)
4. But iLink API `sendmessage` text field limit is **2000 bytes** (UTF-8)
5. Chinese characters = 3 bytes each in UTF-8 → ~666 CJK chars max per chunk
6. Hermes original code split at 2000 chars → a chunk with ~1000 Chinese chars
   = 3000 bytes → API silently rejected the chunk → no error, no delivery

## Affected Code Locations

| File | Lines | Issue |
|------|-------|-------|
| `gateway/platforms/weixin.py` | 845-908 | `len()` in `_pack_markdown_blocks_for_weixin` and `_split_text_for_weixin_delivery` used char count |
| `gateway/platforms/weixin.py` | 861 | `truncate_message()` called without `len_fn` → defaulted to char count |
| `gateway/platforms/weixin.py` | 1764-1775 | `send()` loop used bare `await` → exception on one chunk killed all remaining chunks |

## Fix Applied (v1.1)

Three changes to `gateway/platforms/weixin.py`:

### 1. Add `_utf8_byte_len()`

```python
def _utf8_byte_len(text: str) -> int:
    """Return UTF-8 byte length (iLink API limits by bytes, not characters)."""
    return len(text.encode("utf-8"))
```

### 2. Replace all `len()` with `_utf8_byte_len()` in splitting functions

In `_pack_markdown_blocks_for_weixin()` and `_split_text_for_weixin_delivery()`:

```python
_len = _utf8_byte_len
if _len(content) <= max_length:  # was: len(content) <= max_length
```

Also pass `len_fn=_utf8_byte_len` to `BasePlatformAdapter.truncate_message()`.

### 3. Per-chunk error tolerance in `send()`

```python
for idx, chunk in enumerate(chunks):
    try:
        await self._send_text_chunk(...)
        last_message_id = client_id
    except Exception as exc:
        logger.warning(
            "[%s] chunk %d/%d failed for %s, continuing: %s",
            self.name, idx + 1, len(chunks), _safe_id(chat_id), exc,
        )
```

## Byte-Length Calculation Reference

| Content Type | 2000 chars = bytes | Max chars for 2000 bytes |
|-------------|-------------------|------------------------|
| Pure ASCII | 2000 bytes | 2000 |
| Mixed Chinese/English | ~3000-4000 bytes | ~800-1000 |
| Pure Chinese (CJK) | ~6000 bytes | ~666 |
| Emoji-heavy | ~8000+ bytes | ~500 |

## Test Results

| Scenario | Chars | UTF-8 Bytes | Chunks (before) | Chunks (after fix) |
|----------|-------|-------------|-----------------|-------------------|
| English "Hello " x400 | 2400 | 2400 | 1 (failed) | 2 (✓) |
| Chinese "你好" x350 | 700 | 2100 | 1 (failed) | 2 (✓) |
| Mixed text | 1013 | ~3000 | 1 (failed) | 2 (✓) |

## iLink API Notes

- Endpoint: `ilinkai.weixin.qq.com/ilink/bot/sendmessage`
- Per-message text limit: **2000 bytes** (confirmed empirically)
- Silent failure mode: bytes > 2000 → message not delivered, no error returned
- Rate limit: `errcode=-2` + "rate limited" in errmsg → backoff and retry
- Session expired: `errcode=-14` or `ret=-2` + "unknown error" → retry w/o token
- Chunk delay: 1.5s default between chunks to avoid rate limiting
- Max retries: 4 per chunk with exponential backoff
