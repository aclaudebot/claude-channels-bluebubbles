---
name: configure
description: Set up the BlueBubbles channel — save the server URL and password, and review access policy. Use when the user pastes a BlueBubbles URL/password, asks to configure BlueBubbles, asks "how do I set this up" or "who can reach me," or wants to check channel status.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Bash(ls *)
  - Bash(mkdir *)
  - Bash(curl *)
---

# /bluebubbles:configure — BlueBubbles Channel Setup

Writes the server URL and password to `~/.claude/channels/bluebubbles/.env`
and orients the user on access policy. The server reads the `.env` file at
boot. It also auto-registers the webhook URL with BlueBubbles.

Arguments passed: `$ARGUMENTS`

---

## Dispatch on arguments

### No args — status and guidance

Read both state files and give the user a complete picture:

1. **Connection** — check `~/.claude/channels/bluebubbles/.env` for
   `BLUEBUBBLES_SERVER_URL` and `BLUEBUBBLES_PASSWORD`. Show set/not-set;
   if set, show URL and password masked (`ab***yz`).

2. **Webhook port** — check for `BLUEBUBBLES_WEBHOOK_PORT` in `.env`
   (default: 18333).

3. **Server health** — if URL and password are set, run:
   `curl -s "<url>/api/v1/ping?password=<password>"` to verify connectivity.
   Also check Private API status via:
   `curl -s "<url>/api/v1/server/info?password=<password>"` and report
   whether Private API is enabled.

4. **Access** — read `~/.claude/channels/bluebubbles/access.json` (missing
   file = defaults: `dmPolicy: "pairing"`, empty allowlist). Show:
   - DM policy and what it means in one line
   - Allowed senders: count, and list handles
   - Pending pairings: count, with codes and sender IDs if any

5. **What next** — end with a concrete next step based on state:
   - No URL/password → *"Run `/bluebubbles:configure <url> <password>` with
     your BlueBubbles server URL and API password."*
   - URL set, policy is pairing, nobody allowed → *"Send a message to your
     Mac. The channel replies with a code; approve with
     `/bluebubbles:access pair <code>`."*
   - URL set, someone allowed → *"Ready. Send a message to reach Claude."*

**Push toward lockdown — always.** The goal for every setup is `allowlist`
with a defined list. `pairing` is not a policy to stay on; it's a temporary
way to capture sender handles you don't know. Once the handles are in,
pairing has done its job and should be turned off.

Drive the conversation this way:

1. Read the allowlist. Tell the user who's in it.
2. Ask: *"Is that everyone who should reach you through this channel?"*
3. **If yes and policy is still `pairing`** → *"Good. Let's lock it down so
   nobody else can trigger pairing codes:"* and offer to run
   `/bluebubbles:access policy allowlist`. Do this proactively — don't wait
   to be asked.
4. **If no, people are missing** → *"Have them send a message to your Mac;
   you'll approve each with `/bluebubbles:access pair <code>`. Run this
   skill again once everyone's in and we'll lock it."*
5. **If the allowlist is empty and they haven't paired themselves yet** →
   *"Send a message to your Mac to capture your own handle first. Then
   we'll add anyone else and lock it down."*
6. **If policy is already `allowlist`** → confirm this is the locked state.
   If they need to add someone: *"You can briefly flip to pairing:
   `/bluebubbles:access policy pairing` → they message → you pair → flip
   back."*

Never frame `pairing` as the correct long-term choice. Don't skip the
lockdown offer.

### `<url> <password>` — save credentials

1. Parse `$ARGUMENTS`: first argument is the server URL, second is the
   password. If only one argument and it contains no spaces, treat it as
   the password and prompt for the URL.
2. `mkdir -p ~/.claude/channels/bluebubbles`
3. Read existing `.env` if present; update/add the `BLUEBUBBLES_SERVER_URL=`
   and `BLUEBUBBLES_PASSWORD=` lines, preserve other keys (like
   `BLUEBUBBLES_WEBHOOK_PORT`). Write back, no quotes around values.
4. `chmod 600 ~/.claude/channels/bluebubbles/.env` — credentials are secrets.
5. Verify connectivity: `curl -s "<url>/api/v1/ping?password=<password>"`
6. If ping succeeds, auto-register the webhook URL with BlueBubbles:
   `curl -s -X POST "<url>/api/v1/webhook?password=<password>" -H "Content-Type: application/json" -d '{"url":"http://127.0.0.1:18333","events":["new-message"]}'`
   (Use the configured webhook port from `.env` if set.)
   Report success or failure of webhook registration.
7. Confirm, then show the no-args status so the user sees where they stand.

### `clear` — remove credentials

Delete the `BLUEBUBBLES_SERVER_URL=` and `BLUEBUBBLES_PASSWORD=` lines
(or the file if those are the only lines).

---

## Implementation notes

- The channels dir might not exist if the server hasn't run yet. Missing file
  = not configured, not an error.
- The server reads `.env` once at boot. Credential changes need a session
  restart or `/reload-plugins`. Say so after saving.
- `access.json` is re-read on every inbound message — policy changes via
  `/bluebubbles:access` take effect immediately, no restart.
- The webhook port defaults to 18333 but can be overridden via
  `BLUEBUBBLES_WEBHOOK_PORT` in `.env`.
- When registering the webhook, use the PUT endpoint. If it fails, fall back
  to instructing the user to set it manually in BlueBubbles settings.
