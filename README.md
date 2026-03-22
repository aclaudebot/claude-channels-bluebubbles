# BlueBubbles

Connect your Mac's messaging to your Claude Code session via [BlueBubbles](https://bluebubbles.app).

The MCP server receives messages from a BlueBubbles server via webhooks and provides tools to Claude to reply, react, edit, unsend, and send attachments. When someone texts your Mac, the server forwards the message to your Claude Code session.

## Prerequisites

- **[Bun](https://bun.sh)** — the MCP server runs on Bun. Install with `curl -fsSL https://bun.sh/install | bash`.
- **[BlueBubbles](https://bluebubbles.app)** — a macOS app that acts as a messaging bridge.

## Quick Setup

> Default pairing flow for a single-user DM setup. See [ACCESS.md](./ACCESS.md) for groups and multi-user setups.

**1. Install and configure BlueBubbles.**

Download from [bluebubbles.app/install](https://bluebubbles.app/install). Open it, sign in with your Apple ID, and:

- Go to **Settings > API/Web Server**
- Enable the REST API
- Set a password
- Note the server URL (e.g. `http://localhost:1234` or your LAN IP)

**2. Install the plugin.**

Add the marketplace and install:

```bash
claude plugin marketplace add aclaudebot/claude-channels-bluebubbles
claude plugin install bluebubbles
```

Or install directly from a local clone:

```bash
claude plugin install --plugin-dir /path/to/claude-channels-bluebubbles
```

> **Note:** Development channels require the `--dangerously-load-development-channels` flag when launching Claude Code (see steps 3–4).

**3. Give the server your BlueBubbles credentials.**

Start Claude Code with the development channels flag, specifying the plugin:

```bash
claude --dangerously-load-development-channels plugin:bluebubbles@claude-channels-bluebubbles
```

Then configure your BlueBubbles credentials:

```
/bluebubbles:configure http://localhost:1234 your-password-here
```

This writes `BLUEBUBBLES_SERVER_URL` and `BLUEBUBBLES_PASSWORD` to `~/.claude/channels/bluebubbles/.env` (mode 0600) and auto-registers the webhook with BlueBubbles.

> To run multiple instances (different BlueBubbles servers, separate allowlists), set `BLUEBUBBLES_STATE_DIR` to a different directory per instance.

**4. Relaunch Claude Code.**

Exit your session and start a new one with the same flag:

```bash
claude --dangerously-load-development-channels plugin:bluebubbles@claude-channels-bluebubbles
```

The MCP server starts automatically on launch.

**5. Pair.**

Send a message to the number associated with the Mac running BlueBubbles. The channel replies with a 6-character pairing code. In your Claude Code session:

```
/bluebubbles:access pair <code>
```

Your next message reaches the assistant.

**6. Lock it down.**

Pairing is for capturing sender handles. Once you're in, switch to `allowlist` so strangers don't get pairing-code replies:

```
/bluebubbles:access policy allowlist
```

## Access control

See **[ACCESS.md](./ACCESS.md)** for DM policies, groups, mention detection, delivery config, skill commands, and the `access.json` schema.

Quick reference: sender IDs are **phone numbers** (E.164 format like `+15555551234`) or **email addresses**. Default policy is `pairing`.

## Tools exposed to the assistant

| Tool | Purpose |
| --- | --- |
| `reply` | Send a text message and/or file attachments. Takes `chat_guid` + `text`, optionally `reply_to` (message GUID for threading), `files` (absolute paths), and `effect` (message effect). Auto-chunks text at 4000 chars. |
| `react` | Add or remove a tapback reaction: `love`, `like`, `dislike`, `laugh`, `emphasize`, `question`. Accepts aliases like `heart`, `thumbsup`, emoji. Requires Private API. |
| `edit_message` | Edit a previously sent message (macOS 13+, requires Private API). |
| `unsend_message` | Unsend a previously sent message (macOS 13+, requires Private API). |
| `download_attachment` | Download an attachment by GUID and return the local file path. |
| `send_attachment` | Send a local file as an attachment. Max 50 MB. |
| `mark_read` | Send a read receipt for a chat (requires Private API). |

## Message effects

When the Private API is enabled, the `reply` tool supports message effects via the `effect` parameter:

`slam` `loud` `gentle` `invisible` `echo` `spotlight` `balloons` `confetti` `fireworks` `lasers` `celebration`

## Reactions (Tapbacks)

The six tapbacks with their accepted aliases:

| Tapback | Aliases |
| --- | --- |
| `love` | `heart`, `loved` |
| `like` | `thumbs_up`, `thumbsup`, `liked` |
| `dislike` | `thumbs_down`, `thumbsdown`, `disliked` |
| `laugh` | `haha`, `lol`, `laughed` |
| `emphasize` | `!!`, `emphasized` |
| `question` | `?`, `questioned` |

Inbound tapback reactions from others are forwarded to Claude as events.

## Attachments

Inbound attachments include their GUID in the `<channel>` notification. Claude calls `download_attachment` to fetch the file to `~/.claude/channels/bluebubbles/inbox/` and then reads it.

Outbound files can be sent via the `files` parameter on `reply` or via `send_attachment`. Max 50 MB per file.

## Private API

Some features require BlueBubbles' [Private API](https://docs.bluebubbles.app/server/advanced/private-api):

- Typing indicators
- Read receipts
- Tapback reactions
- Reply threading
- Message effects
- Message editing (macOS 13+)
- Message unsending (macOS 13+)

The server probes Private API status at boot and every 5 minutes. Features that require it gracefully degrade — text messaging always works.

## Environment variables

| Variable | Default | Description |
| --- | --- | --- |
| `BLUEBUBBLES_SERVER_URL` | *(required)* | BlueBubbles REST API URL |
| `BLUEBUBBLES_PASSWORD` | *(required)* | BlueBubbles API password |
| `BLUEBUBBLES_WEBHOOK_PORT` | `18333` | Local webhook listener port |
| `BLUEBUBBLES_STATE_DIR` | `~/.claude/channels/bluebubbles` | State directory |
| `BLUEBUBBLES_ACCESS_MODE` | *(unset)* | Set to `static` to pin config at boot (disables pairing) |
