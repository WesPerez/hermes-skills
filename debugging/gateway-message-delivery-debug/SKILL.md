---
name: gateway-message-delivery-debug
description: "Systematic approach for diagnosing message truncation, silent delivery failures, and platform API limits across all Hermes gateway messaging platforms (WeChat, Telegram, Discord, QQ, etc.). Applies when users report messages 'cut off mid-sentence', 'only half the message arrived', or 'replies suddenly stop'."
version: 1.1.0
author: Hermes Agent
tags: [debugging, gateway, weixin, telegram, discord, message-delivery, ilink, truncation]
---

# Gateway Message Delivery Debugging

Diagnose why messages delivered through a gateway platform are truncated, failing silently, or stopping mid-stream. This skill covers the systematic approach to root-causing delivery issues that look like "the agent stopped talking" but are actually sending failures.

## Common Symptoms

| Symptom | Likely Cause |
|---------|-------------|
| First chunk arrives, rest vanish | A later chunk exceeds API byte limit → fails → old code stops sending remaining chunks |
| Messages arrive out of order | No ordering guarantee on some platforms; chunk delay too short |
| Short messages work, long ones fail | Byte vs character counting mismatch (see below) |
| Only plain text arrives, markdown broken | Platform markdown support differs from what Hermes sends |
| Messages arrive but garbled mid-text | Byte-level split cutting multi-byte char (UTF-8) |

## Step-by-Step Diagnostic

### 1. Check MAX_MESSAGE_LENGTH in the platform adapter

```bash
grep -n "MAX_MESSAGE_LENGTH" gateway/platforms/<platform>.py
```

Key question: **Is this limit in characters or bytes?**

| Platform | MAX_MESSAGE_LENGTH | Unit | Notes |
|----------|-------------------|------|-------|
| WeChat (weixin.py) | 2000 | **bytes** (pre-v1.1: chars) | iLink API byte limit. Chinese = 3 bytes/char |
| Telegram | 4096 | chars (but uses utf16_len) | |
| Discord | 2000 | chars | |
| DingTalk | 20000 | chars | |
| WeCom | 4000 | chars | |
| Email | 50000 | chars | |

### 2. Check if the split function uses the right length measure

Look at how `len()` is used in the splitting functions:

```python
# BAD (char count - fails for multi-byte content):
if len(content) <= max_length:

# GOOD (byte count - correct for byte-limited APIs):
if len(content.encode("utf-8")) <= max_length:
```

For platforms with **byte-level** API limits (WeChat iLink, some SMS gateways), character-count splitting produces chunks that exceed the API limit.

### 3. Check per-chunk error handling

```python
# BAD: single chunk failure stops ALL remaining chunks
for chunk in chunks:
    await self._send_text_chunk(...)  # raises on failure → rest skipped

# GOOD: per-chunk try/except logs and continues
for idx, chunk in enumerate(chunks):
    try:
        await self._send_text_chunk(...)
    except Exception as exc:
        logger.warning("chunk %d/%d failed: %s", idx+1, len(chunks), exc)
```

### 4. Check streaming consumer (stream_consumer.py)

The stream consumer also uses `MAX_MESSAGE_LENGTH` for cursor updates and final
send. Verify its `safe_limit` calculation matches the adapter's byte awareness:

```python
_raw_limit = getattr(self.adapter, "MAX_MESSAGE_LENGTH", 4096)
_safe_limit = max(500, _raw_limit - len(self.cfg.cursor) - 100)
```

### 5. Enable per-chunk logging

Set `WEIXIN_SEND_CHUNK_RETRIES` and `WEIXIN_SEND_CHUNK_DELAY_SECONDS` via
config or env to control retry behavior. Monitor gateway logs:

```bash
grep "chunk\|send.*fail\|rate limit" ~/.hermes/logs/gateway.log | tail -20
```

## iLink (WeChat) Specific

Reference: `references/weixin-ilink-byte-limit.md`

| Config | Default | Env variable |
|--------|---------|-------------|
| `send_chunk_delay_seconds` | 1.5 | `WEIXIN_SEND_CHUNK_DELAY_SECONDS` |
| `send_chunk_retries` | 4 | `WEIXIN_SEND_CHUNK_RETRIES` |
| `split_multiline_messages` | false | `WEIXIN_SPLIT_MULTILINE_MESSAGES` |
| MAX_MESSAGE_LENGTH | 2000 bytes | hardcoded |

## Pitfalls

- **`len()` vs `len(text.encode("utf-8"))`**: Always verify which one the
  platform API actually uses. If your messages have Chinese/Japanese/Korean
  characters (3 bytes/char), char-count vs byte-count matters enormously.
- **Silent API rejection**: Some APIs (iLink) silently drop chunks over the
  byte limit rather than returning a clear error. The only symptom is "chunks
  after position N never arrive."
- **Rate limiting on burst sends**: Even when each chunk is under the byte
  limit, sending 10+ chunks with only 1.5s delay may trigger rate limits.
  Increase `send_chunk_delay_seconds` or split into fewer, denser chunks.
- **Double-splitting**: The `stream_consumer.py` splits before passing to
  `adapter.send()`, which splits again. Verify the two splitting stages don't
  fight each other (e.g., byte-aware vs char-aware).
- **Sites-enabled vs sites-available**: On Ubuntu, these may be SEPARATE files
  rather than a symlink. Edit the one nginx actually reads.

## Verification Script

```bash
# Quick test: does your byte-length split produce valid chunks?
python3 -c "
def byte_len(t): return len(t.encode('utf-8'))
text = '你好' * 350  # 2100 bytes, should split
chunks = []
# ... insert your platform's split logic here ...
assert all(byte_len(c) <= 2000 for c in chunks), 'Oversized chunk!'
print(f'{len(text)} chars -> {len(chunks)} chunks, all OK')
"
```

## Related Skills

- `hermes-agent` — general Hermes configuration and platform setup
- `remote-desktop-setup` — for cases needing human pre-authentication
