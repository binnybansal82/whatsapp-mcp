# WhatsApp MCP Project

## Git
- Always push to the fork: `origin` = `https://github.com/binnybansal82/whatsapp-mcp.git`

## Architecture
- The MCP server is a thin bridge — it only reads/writes data from the WhatsApp SQLite DB and the WhatsApp API.
- **Do NOT add business logic to the MCP server.** All intelligence (filtering, analysis, follow-up detection) should be handled by Claude using the existing tools.

## Handling "awaiting reply" / "messages waiting for my reply"

When the user asks about messages awaiting their reply, use the `get_chats_with_context` tool for efficient single-call retrieval.

### Step 1: Fetch chats with context (single call)
Call `get_chats_with_context(only_incoming=True, since=<4 weeks ago ISO-8601>, context_messages=5)` to get all chats where the last message is from someone else, along with 5 recent messages per chat for context. Locked chats and inactive groups are automatically excluded.

### Step 2: Check for deferred replies
Also call `get_chats_with_context(only_incoming=False, since=<4 weeks ago ISO-8601>, context_messages=3)` to scan for **DEFERRED** chats — where `last_is_from_me = true` BUT the last message is a promise to follow up. Look for patterns like:
- "I'll get back", "will get back", "let me check", "let me look into", "will follow up", "brb", "one sec", "hold on", "let me think", "I'll check", "will revert", "reply later"

### Step 3: Categorize and prioritize
From the results, categorize:

**AWAITING REPLY** (from Step 1):
- HIGH priority: direct chats with recent activity
- MEDIUM priority: group chats where I was recently active
- LOW priority: older chats

**DEFERRED** (from Step 2): chats where I promised to follow up

### Step 4: Calendar reminders for deferred items
For any DEFERRED chats detected, automatically create a Google Calendar reminder (next morning) so the user doesn't forget to follow up. Use the chat name and deferred message as the event description.

### Step 5: Present results
Show results grouped by category (AWAITING REPLY vs DEFERRED), sorted by most recent, with conversation context from the messages included in each chat.

## Locked chats
Locked chats are automatically tracked in the DB (`chats.locked = 1`). Always exclude them from queries by adding `AND c.locked = 0` or equivalent.
