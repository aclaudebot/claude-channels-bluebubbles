# BlueBubbles — Access & Delivery

Your messaging account is publicly addressable — anyone with your phone number or Apple ID email can text you. Without a gate, every inbound message flows straight into your assistant session. The access model described here decides who gets through.

By default, a DM from an unknown sender triggers **pairing**: the channel replies with a 6-character code and drops the message. You run `/bluebubbles:access pair <code>` from your assistant session to approve them. Once approved, their messages pass through.

All state lives in `~/.claude/channels/bluebubbles/access.json`. The `/bluebubbles:access` skill commands edit this file; the server re-reads it on every inbound message, so changes take effect without a restart. Set `BLUEBUBBLES_ACCESS_MODE=static` to pin config to what was on disk at boot (pairing is unavailable in static mode since it requires runtime writes).

## At a glance

| | |
| --- | --- |
| Default policy | `pairing` |
| Sender ID | Phone number (`+15555551234`) or email (`user@example.com`) |
| Group key | Chat GUID (e.g. `iMessage;+;chat123456`) |
| `ackReaction` | Tapbacks: `love`, `like`, `dislike`, `laugh`, `emphasize`, `question` |
| Config file | `~/.claude/channels/bluebubbles/access.json` |

## DM policies

`dmPolicy` controls how DMs from senders not on the allowlist are handled.

| Policy | Behavior |
| --- | --- |
| `pairing` (default) | Reply with a pairing code, drop the message. Approve with `/bluebubbles:access pair <code>`. |
| `allowlist` | Drop silently. No reply. Recommended once all users are paired. |
| `disabled` | Drop everything, including allowlisted users and groups. |

```
/bluebubbles:access policy allowlist
```

## Sender IDs

BlueBubbles identifies senders by **phone number** (E.164 format like `+15555551234`) or **email address**. The allowlist stores these handles, normalized:

- Phone numbers: stripped to digits with `+` prefix
- Emails: lowercased, `mailto:` prefix stripped

Pairing captures the handle automatically. To add one manually:

```
/bluebubbles:access allow +15555551234
/bluebubbles:access allow user@example.com
/bluebubbles:access remove +15555551234
```

## Chat GUIDs

BlueBubbles uses chat GUIDs to identify conversations:

- **DMs**: `iMessage;-;+15555551234` (`;-;` = individual)
- **Groups**: `iMessage;+;chat123456` (`;+;` = group)

You typically don't need to use these directly — the channel extracts them from webhooks automatically.

## Groups

Groups are off by default. Opt each one in individually using its chat GUID.

```
/bluebubbles:access group add iMessage;+;chat123456
```

With the default `requireMention: true`, the bot responds only when a mention pattern matches the message text. Pass `--no-mention` to process every message, or `--allow handle1,handle2` to restrict which members can trigger it.

```
/bluebubbles:access group add iMessage;+;chat123456 --no-mention
/bluebubbles:access group add iMessage;+;chat123456 --allow +15555551234,+15555559876
/bluebubbles:access group rm iMessage;+;chat123456
```

## Mention detection

In groups with `requireMention: true`, a message must match at least one regex in `mentionPatterns` to be processed.

```
/bluebubbles:access set mentionPatterns '["^hey claude\\b", "\\bassistant\\b"]'
```

## Delivery

Configure outbound behavior with `/bluebubbles:access set <key> <value>`.

**`ackReaction`** reacts to inbound messages on receipt using a tapback. Must be one of: `love`, `like`, `dislike`, `laugh`, `emphasize`, `question`. Empty string disables.

```
/bluebubbles:access set ackReaction like
/bluebubbles:access set ackReaction ""
```

**`sendReadReceipts`** controls whether read receipts are sent automatically for delivered messages. Default: `true`. Requires Private API.

```
/bluebubbles:access set sendReadReceipts false
```

**`textChunkLimit`** sets the split threshold for long messages. Default and max: 4000 characters.

**`chunkMode`** chooses the split strategy: `length` cuts exactly at the limit; `newline` (default) prefers paragraph boundaries.

## Skill reference

| Command | Effect |
| --- | --- |
| `/bluebubbles:access` | Print current state: policy, allowlist, pending pairings, enabled groups. |
| `/bluebubbles:access pair a4f91c` | Approve pairing code `a4f91c`. Adds the sender to `allowFrom` and sends a confirmation message. |
| `/bluebubbles:access deny a4f91c` | Discard a pending code. The sender is not notified. |
| `/bluebubbles:access allow +15555551234` | Add a sender handle directly. |
| `/bluebubbles:access remove +15555551234` | Remove from the allowlist. |
| `/bluebubbles:access policy allowlist` | Set `dmPolicy`. Values: `pairing`, `allowlist`, `disabled`. |
| `/bluebubbles:access group add <chatGuid>` | Enable a group. Flags: `--no-mention`, `--allow handle1,handle2`. |
| `/bluebubbles:access group rm <chatGuid>` | Disable a group. |
| `/bluebubbles:access set ackReaction like` | Set a config key: `ackReaction`, `sendReadReceipts`, `textChunkLimit`, `chunkMode`, `mentionPatterns`. |

## Config file

`~/.claude/channels/bluebubbles/access.json`. Absent file is equivalent to `pairing` policy with empty lists, so the first DM triggers pairing.

```jsonc
{
  // Handling for DMs from senders not in allowFrom.
  "dmPolicy": "pairing",

  // Phone numbers or emails allowed to DM.
  "allowFrom": ["+15555551234", "user@example.com"],

  // Groups the channel is active in. Empty object = DM-only.
  "groups": {
    "iMessage;+;chat123456": {
      // true: respond only when mentionPatterns match.
      "requireMention": true,
      // Restrict triggers to these senders. Empty = any member (subject to requireMention).
      "allowFrom": []
    }
  },

  // Case-insensitive regexes that count as a mention.
  "mentionPatterns": ["^hey claude\\b"],

  // Tapback reaction on receipt: love, like, dislike, laugh, emphasize, question. "" disables.
  "ackReaction": "like",

  // Send read receipts automatically. Requires Private API.
  "sendReadReceipts": true,

  // Split threshold for long messages. Max: 4000.
  "textChunkLimit": 4000,

  // length = cut at limit. newline = prefer paragraph boundaries.
  "chunkMode": "newline"
}
```
