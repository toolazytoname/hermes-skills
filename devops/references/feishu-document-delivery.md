# Feishu Document Delivery Pattern

## User Preference

This user (in Feishu "服务器调试" group) reads documentation from files, not inline chat messages. They explicitly stated: "一大段，我不方便阅读，以后尽量都是这么发" (long inline messages are hard to read, send as files).

## Standard Delivery Pattern

### Step 1: Write the document
```python
write_file(
    content="# Document Title\n\n...",
    path="/opt/data/home/docs/<filename>.md"
)
```

### Step 2: Send via Feishu as media
```python
send_message(
    message="MEDIA:/opt/data/home/docs/<filename>.md",
    target="feishu:服务器调试"  # or appropriate channel
)
```

### Step 3: Optional context message (short)
```python
send_message(
    message="📚 Document Title — see attached file",
    target="feishu:服务器调试"
)
```

## Document Storage Location

All user-facing documents go to `/opt/data/home/docs/`. This path is confirmed as the user's home directory on the Hermes server (`/opt/data/home`).

## When to Use File Delivery

| Content type | Use file delivery? |
|-------------|-------------------|
| Tutorial / how-to guide | ✅ Yes |
| Quick command reference | Inline ok |
| Troubleshooting steps | Inline ok (short) |
| Multi-section doc (>5 paragraphs) | ✅ Yes |
| Technical reference docs | ✅ Yes |
| Diagrams / architecture | ✅ Yes |

## Feishu Channel Reference

Primary channel for server troubleshooting: `feishu:服务器调试`

Available channels can be listed with:
```python
send_message(action="list")  # returns all Feishu targets
```

## Why Not Just Inline?

- Feishu renders `.md` files as proper documents with formatting
- User can download and archive
- Long messages get truncated in Feishu
- Better for code blocks and tables
