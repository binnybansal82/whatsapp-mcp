# WhatsApp MCP Project

## Git
- Always push to the fork: `origin` = `https://github.com/binnybansal82/whatsapp-mcp.git`

## Architecture
- The MCP server is a thin bridge — it only reads/writes data from the WhatsApp SQLite DB and the WhatsApp API.
- **Do NOT add business logic to the MCP server.** All intelligence (filtering, analysis, follow-up detection) should be handled by Claude using the existing tools.

## Handling "awaiting reply" / "messages waiting for my reply"

When the user asks about messages awaiting their reply, use the existing MCP tools — do NOT modify the server.

### Step 1: Get recent chats
Call `list_chats(limit=50, include_last_message=true, sort_by="last_active")` to get chats with their last message info. Each chat includes `last_is_from_me`, `last_message`, `last_sender`, and `last_message_time`.

### Step 2: Identify chats needing attention
From the results, identify two categories:

**AWAITING REPLY** — chats where `last_is_from_me = false`:
- For direct chats (`@s.whatsapp.net`): always include
- For group chats (`@g.us`): only include if relevant (recent activity from me)

**DEFERRED** — chats where `last_is_from_me = true` BUT the last message is a promise to follow up. Look for patterns like:
- "I'll get back", "will get back", "let me check", "let me look into", "will follow up", "brb", "one sec", "hold on", "let me think", "I'll check", "will revert", "reply later"

### Step 3: Get context
For important chats, call `list_messages(chat_jid=<jid>, limit=5)` to fetch recent conversation context.

### Step 4: Calendar reminders for deferred items
For any DEFERRED chats detected, automatically create a Google Calendar reminder (next morning) so the user doesn't forget to follow up. Use the chat name and deferred message as the event description.

### Step 5: Present results
Show results grouped by category (AWAITING REPLY vs DEFERRED), sorted by most recent, with conversation context.
