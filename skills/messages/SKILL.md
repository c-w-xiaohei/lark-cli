---
name: messages
description: Retrieve chat message history from Lark - get messages from group chats, private chats, and threads. Use when user asks about chat messages, conversation history, or what was discussed in a group.
---

# Messages Skill

Retrieve chat message history and search for chats/groups via the `lark` CLI.

## Running Commands

Ensure `lark` is in your PATH, or use the full path to the binary. Set the config directory if not using the default:

```bash
lark msg <command>
lark chat <command>
# Or with explicit config:
LARK_CONFIG_DIR=/path/to/.lark lark msg <command>
```

## Commands Reference

### Search for Chats/Groups
```bash
# Search for chats by name
lark chat search "project"

# Search with Chinese characters
lark chat search "团队"

# List all visible chats (no query)
lark chat search

# Limit results
lark chat search "team" --limit 10
```

Available flags:
- `--limit`: Maximum number of chats to retrieve (0 = no limit)

Output:
```json
{
  "chats": [
    {
      "chat_id": "oc_d8e62a81cd188199ab994080f1e0804f",
      "name": "project-team",
      "description": "Project discussion group",
      "owner_id": "ou_46538f635a314d611cdd028c9c293d21",
      "external": false,
      "chat_status": "normal"
    }
  ],
  "count": 1,
  "query": "project"
}
```

The search supports:
- Group name matching (including internationalized names)
- Group member name matching
- Multiple languages
- Fuzzy search (pinyin, prefix, etc.)

### Get Chat History
```bash
# Get messages from a group chat
lark msg history --chat-id oc_xxxxx

# Get messages with limit
lark msg history --chat-id oc_xxxxx --limit 50

# Get messages in a time range (Unix timestamp)
lark msg history --chat-id oc_xxxxx --start 1704067200 --end 1704153600

# Get messages in a time range (ISO 8601)
lark msg history --chat-id oc_xxxxx --start 2024-01-15 --end 2024-01-16

# Sort by newest first
lark msg history --chat-id oc_xxxxx --sort desc

# Get thread messages
lark msg history --chat-id thread_xxxxx --type thread
```

Available flags:
- `--chat-id` (required): Chat ID or thread ID
- `--type`: Container type - `chat` (default) or `thread`
- `--start`: Start time (Unix timestamp or ISO 8601)
- `--end`: End time (Unix timestamp or ISO 8601)
- `--sort`: Sort order - `asc` (default) or `desc`
- `--limit`: Maximum number of messages (0 = no limit)

Output:
```json
{
  "messages": [
    {
      "message_id": "om_dc13264520392913993dd051dba21dcf",
      "msg_type": "text",
      "content": "{\"text\":\"Hello world\"}",
      "sender": {
        "id": "ou_155184d1e73cbfb8973e5a9e698e74f2",
        "type": "user"
      },
      "create_time": "2024-01-15T09:00:00+08:00",
      "mentions": [],
      "is_reply": false,
      "thread_id": "",
      "deleted": false
    }
  ],
  "count": 1,
  "chat_id": "oc_xxxxx"
}
```

## Reading Thread Replies

When a message has a `thread_id` field, it means the message is part of a thread (or is the root of a thread with replies). To fetch all replies in that thread:

1. Get chat history and note the `thread_id` from any message that has one
2. Use that `thread_id` with `--type thread` to get all messages in the thread

Example workflow:
```bash
# Get recent messages from a chat
lark msg history --chat-id oc_xxxxx --limit 10 --sort desc

# If a message has thread_id: "omt_1a3b99f9d2cfd982", fetch the thread
lark msg history --chat-id omt_1a3b99f9d2cfd982 --type thread
```

Thread messages will have `is_reply: true` for replies (the root message has `is_reply: false`).

## Message Types

The `msg_type` field indicates the message format:
- `text` - Plain text message
- `post` - Rich text post
- `image` - Image
- `file` - File attachment
- `audio` - Audio message
- `media` - Video/media
- `sticker` - Sticker/emoji
- `interactive` - Interactive card
- `share_chat` - Shared chat
- `share_user` - Shared user contact

## Parsing Message Content

The `content` field is a JSON string. Parse it based on `msg_type`:

### Text Messages
```json
{"text": "Hello world"}
```

### Post Messages (Rich Text)
```json
{
  "title": "Post Title",
  "content": [[{"tag": "text", "text": "paragraph text"}]]
}
```

### Image Messages
```json
{"image_key": "img_xxxx"}
```

### File Messages
```json
{"file_key": "file_xxxx", "file_name": "document.pdf"}
```

### Audio Messages
```json
{"file_key": "file_xxxx", "duration": 5000}
```

### Media (Video) Messages
```json
{"file_key": "file_xxxx", "image_key": "img_xxxx", "file_name": "video.mp4", "duration": 10000}
```

## Downloading Resource Files

Download images, files, audio, and video from messages using `msg resource`:

```bash
# Download an image
lark msg resource --message-id om_xxx --file-key img_v3_xxx --type image --output ./image.png

# Download a file, audio, or video
lark msg resource --message-id om_xxx --file-key file_v2_xxx --type file --output ./document.pdf
```

Available flags:
- `--message-id` (required): Message ID containing the resource
- `--file-key` (required): Resource key from message content (`image_key` or `file_key`)
- `--type` (required): `image` for images, `file` for files/audio/video
- `--output` (required): Output file path

Output:
```json
{
  "success": true,
  "message_id": "om_xxx",
  "file_key": "img_v3_xxx",
  "output_path": "./image.png",
  "content_type": "image/png",
  "bytes_written": 62419
}
```

### Workflow: View Images from Chat

1. Get message history to find image messages:
```bash
lark msg history --chat-id oc_xxxxx --limit 20
```

2. Find messages with `msg_type: "image"` and parse the content to get `image_key`

3. Download the image:
```bash
lark msg resource --message-id om_xxx --file-key img_v3_xxx --type image --output /tmp/image.png
```

4. View the image using the Read tool on the downloaded file

### Limitations

- Maximum file size: 100MB
- Emoji resources cannot be downloaded
- Resources from card messages, merged messages, or forwarded messages cannot be downloaded (API error 234043)
- The `message_id` and `file_key` must match (the file must belong to that message)

## Integration with Contacts

Enrich sender information by looking up user details:

```bash
# Get message history
lark msg history --chat-id oc_xxxxx --limit 10

# Look up sender details
lark contact get ou_sender_id
```

## Output Format

All commands output JSON. Format appropriately when presenting to user.

## Error Handling

Errors return JSON:
```json
{
  "error": true,
  "code": "ERROR_CODE",
  "message": "Description"
}
```

Common error codes:
- `AUTH_ERROR` - Need to run `lark auth login`
- `VALIDATION_ERROR` - Missing required fields (e.g., chat-id)
- `API_ERROR` - Lark API issue (e.g., bot not in group, missing permissions)

## Permissions Required

- The bot must be in the group chat
- For group chat messages, the app needs the "Read all messages in associated group chat" permission scope
- Private chat messages only require `im:message:readonly` scope

## Notes

- Chat IDs typically start with `oc_` (for chats) or `thread_` (for threads)
- Time filters don't work for thread container type
- Messages are sorted by creation time ascending by default
- The `deleted` field indicates if a message was recalled
