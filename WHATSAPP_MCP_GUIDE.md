# How to pull WhatsApp data with Claude

This setup lets Claude Code read your WhatsApp chats on your own Mac.
It uses the open source project **whatsapp-mcp**: https://github.com/lharries/whatsapp-mcp

## How it works

There are two parts:

1. **The bridge** (a small Go program). It connects to WhatsApp like a linked device (same as WhatsApp Web) and saves every message into a local database file.
2. **The MCP server** (a small Python program). It lets Claude search and read that database.

Everything stays on your computer. Nothing is uploaded anywhere.

```
WhatsApp  -->  bridge (Go)  -->  messages.db (SQLite)  -->  MCP server (Python)  -->  Claude Code
```

## What you need

- A Mac (Windows and Linux also work, see the project README)
- Go: `brew install go`
- uv (Python runner): `brew install uv`
- Claude Code
- Your phone with WhatsApp

## Setup

### 1. Download the project

```bash
git clone https://github.com/lharries/whatsapp-mcp.git ~/whatsapp-mcp
```

### 2. Start the bridge

```bash
cd ~/whatsapp-mcp/whatsapp-bridge
go run main.go
```

A QR code shows in the terminal. On your phone open
**WhatsApp > Settings > Linked Devices > Link a Device** and scan it.

Leave this terminal window open. The bridge must keep running to get new messages.
The first sync of old chats can take a few minutes.

### 3. Connect it to Claude Code

```bash
claude mcp add whatsapp -- uv --directory ~/whatsapp-mcp/whatsapp-mcp-server run main.py
```

Restart Claude Code. Type `/mcp` and check that `whatsapp` is listed as connected.

### 4. Try it

Ask Claude things like:

- "List my WhatsApp chats from today"
- "Show the last 20 messages with Ahmed"
- "Which chats are waiting for my reply?"
- "Search my WhatsApp for the word invoice"

## Where the data lives

`~/whatsapp-mcp/whatsapp-bridge/store/`

- `messages.db` holds chats and messages
- `whatsapp.db` holds the login session and contact names

You can also read the database directly. This is faster for big questions:

```bash
cd ~/whatsapp-mcp/whatsapp-bridge/store
sqlite3 'file:messages.db?mode=ro' "
  select m.timestamp, c.name, m.is_from_me, m.content
  from messages m join chats c on c.jid = m.chat_jid
  where m.timestamp > '2026-01-01'
  order by m.timestamp desc
  limit 50;"
```

Main tables:

| Table | Columns that matter |
|---|---|
| `chats` | `jid` (chat id), `name`, `last_message_time` |
| `messages` | `chat_jid`, `sender`, `content`, `timestamp`, `is_from_me`, `media_type` |

Group chat ids end with `@g.us`.

## Things that go wrong

- **The bridge can stop without telling you.** Claude still answers, but from old data. Before you trust an answer, check that the newest message in `messages.db` is recent. If it is old, restart the bridge.
- **The bridge library gets out of date.** If the bridge fails to connect (for example error 405), update it and start again:
  ```bash
  cd ~/whatsapp-mcp/whatsapp-bridge
  go get -u go.mau.fi/whatsmeow@latest && go mod tidy
  go run main.go
  ```
- **You get logged out about every 20 days.** Scan the QR code again.
- **Dates use a space, not a T.** Timestamps look like `2026-01-15 11:49:29+04:00`. Use the same format when you filter by date.
- **Only messages from the day you linked (plus the history WhatsApp sends) are saved.** Very old chats may be missing.

## Safety

- The MCP server can also **send** messages. Tell Claude to only draft, and to send only when you say "send".
- Do not message many strangers from this setup. WhatsApp bans numbers for bulk cold messages.
- The `store` folder is your full chat history and login. Do not share it or put it in git.
