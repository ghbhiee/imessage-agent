# imessage-agent

English | [中文](README.zh.md)

**Drive your AI agents from iMessage — with your own Apple ID, on your own Mac.**

A small relay that watches `chat.db`, hands your messages to an agent
([Claude Code](https://docs.anthropic.com/en/docs/claude-code) via the Agent SDK,
or [DeepSeek Harness / dsh](https://github.com/deepseek-ai/deepseek-harness)),
and sends the reply back through Messages.app — as *you*.

## What it is

Text an "agent contact" from your iPhone; the agent on your Mac picks it up, does the work, and replies with **your own Apple ID**.

```
iPhone Messages ──▶ iCloud sync ──▶ chat.db on the Mac ──▶ bridge.py ──▶ Claude / dsh
      ▲                                                                    │
      └──────────── Messages.app (AppleScript, your account) ◀─────────────┘
```

- One Apple ID, one Mac, one macOS user — shared by you and the agent.
- Each conversation (a 1:1 chat or a group) can route to a different agent: contact A → Claude, contact B → dsh.
- Allowlist routing: only the conversations listed in the config are handled; everything else is read and ignored.
- Claude's permission prompts arrive over iMessage too: reply `1` to allow, `2` to always allow, `3` to deny.
- dsh sessions can live in the dsh web UI, so the browser and your phone show the very same session.

## Why it is built this way

Most of this section is about iMessage quirks you will hit with any agent ↔ iMessage bridge.

### 1. Why not give the agent its own Apple ID

The obvious design is a separate Apple ID for the agent, so the two of you chat like two people. But then the agent is *someone else*:

- It has to act **as you**: reply to messages for you, use the websites and apps you are signed in to, read and write your files. A different account can't do that.
- It **shares the computer** with you. Within one macOS user, Messages can only be signed in to one Apple ID. Running two accounts at once means a second macOS user or a VM — and an agent living in another user can't reach your stuff.

So the premise here is: **the agent is you**, just with an extra pair of hands.

### 2. One account: telling your messages apart from the agent's

In `chat.db`, what you type on your iPhone and what the agent sends from the Mac are both `is_from_me = 1`. No column says which device sent a message.

The common workaround is to guess from text: remember what the agent just sent, drop anything identical as an echo, and add a rate limit as a safety net. In practice this swallows messages you really sent and produces duplicate replies.

This relay uses **deterministic rules** instead:

- Every message the agent sends starts with a fixed prefix (`🤖 ` for Claude, `🐳 ` for dsh by default). Anything with a prefix is skipped; anything without one is you.
- As a second safeguard it keeps fingerprints of the last 200 texts it sent.
- No rate limit, so nothing gets swallowed.

### 3. Why not just message yourself

Messaging yourself (your own number or email) works, and the relay supports it (`self_chat`). The catch: **every message produces two rows** in `chat.db`:

| Row | `is_from_me` | `reply_to_guid` |
|---|---|---|
| original | 1 | (previous message) |
| echo | 0 | the original's guid |

The relay folds the echo into the original and handles it once (`reply_to_guid` is a deterministic link), but Messages on the Mac shows every message twice, which gets messy.

### 4. Recommended: a "silent peer"

Pick an iMessage address **that you control but that never talks**, save it as a contact (say, "Claude"), and chat with it 1:1:

- Your messages: `is_from_me = 1`, a single row, no echo.
- The agent's replies: also sent from your account, with the 🤖 prefix.
- The peer never says anything — you get a clean thread where you ask and "you" answer.

Choosing the peer address:

- **Best: the email of a separate Apple ID.** Email addresses aren't recycled by carriers and can't fall back to SMS.
- A phone number works too, but a number you stop paying for may be reassigned, and its new owner would receive everything you and the agent say. If the number goes offline for long, messages may also turn into SMS.
- Want several agents? Use several such addresses, one per agent.

### 5. How iMessage delivery works (lessons learned)

- Whether you can iMessage an address depends on whether that address is **registered with iMessage at Apple**: at least one device must be signed in to Messages with that Apple ID and have the address checked under "Send & Receive". Merely having an Apple ID is not enough.
- If the address isn't registered, the iPhone decides at lookup time that it's "not iMessage" and sends it as **SMS** right away. An email address can't receive SMS, so the message just sits as "Not Delivered". `chat.db` then shows `service = 'SMS'`, `is_sent = 0`, `error = 0` — **no error code at all**.
- "Settings → Messages → Send as SMS" only controls whether a *failed* iMessage is retried as SMS. It has no say over the lookup-time decision.
- To check: start a new message on the iPhone and type the address. Blue means registered; red or grey means not.
- An address can drop off your iMessage addresses when you add or change a phone number or re-verify your account. Re-add and verify it under "Settings → Apple Account → Sign-In & Security → Email & Phone Numbers", then check it in "Send & Receive".
- Group chats have no echo — one row per message. But iMessage may split one group into two `chat guid`s depending on whether you send from your phone number or your email; add both to the allowlist.

### 6. Permissions: why the Agent SDK instead of `claude -p`

With `claude -p` there is nobody to click "Allow", so any tool outside your rules is simply denied. This relay uses the Python Claude Agent SDK instead:

- `setting_sources = user/project/local`: the allow rules in your `~/.claude/settings.json` apply as usual and never reset.
- Anything outside those rules goes through the `can_use_tool` callback, which posts "[Permission #N] … reply 1/2/3" to `approval_chat`. Answering `2` writes the CLI's suggested rule back to your settings, so you won't be asked again. A timeout counts as a denial.
- Approval answers are only accepted from you, in the approval conversation; someone else replying `1` in a group does nothing.
- `auto` mode is a good default: routine actions are allowed automatically and only risky ones are sent to you. Switch any time with `/mode`.

### 7. dsh: why through the web plugin rather than `dsh headless`

`dsh headless --session-id` can resume a session, but if `dsh web` is running at the same time, the web server takes over the write lock of every session it can see and never releases it. The next resume then fails with `already owned by an active write handle`, so every turn ends up in a new session.

Both fixes are implemented:

- `agent: "dsh"`: the headless CLI, with `dsh-sessions.patch.yml` storing sessions in a separate directory (`$DSH_HOME/sessions-imessage`). The web server can't see them, so it doesn't grab the lock. The trade-off: these sessions don't show up in the web UI.
- `agent: "dsh-web"` (recommended): goes through the `/plugins/imessage` API of [dsh-plugin-discord](https://github.com/ghbhiee/dsh-plugin-discord), which creates and prompts sessions inside the web server process. It reuses the same session code the Discord side runs (`SessionBridge`). Sessions are titled `[iMessage] MM-DD HH:MM`, appear in the web UI and can be continued there; `/new`, `/sessions`, `/use <id>`, `/current` and `/stop` work just like on Discord.

### 8. Full Disk Access: why a dedicated copy of Python

Reading `~/Library/Messages/chat.db` requires Full Disk Access. For a process started by launchd, the permission attaches to the executable it launches. Granting it to uv's shared Python would give every script on the machine that uses that Python full disk access. So the interpreter is copied into `runtime/`, and only that copy gets the permission.

## Install

Requirements: macOS, Messages signed in to iMessage, [uv](https://docs.astral.sh/uv/), the Claude Code CLI (for Claude), dsh (for dsh).

```bash
git clone https://github.com/ghbhiee/imessage-agent.git ~/imessage-agent
cd ~/imessage-agent

# dedicated Python (the only one that gets Full Disk Access)
uv python install 3.12
cp -R "$(dirname "$(dirname "$(uv python find 3.12)")")" runtime
uv venv --python runtime/bin/python3.12
uv sync

cp config.example.json config.json
.venv/bin/python bridge.py --chats      # list conversations and their chat guids for config.json
```

1. System Settings → Privacy & Security → Full Disk Access → `+` → pick `runtime/bin/python3.12`.
2. Try it in the foreground: `.venv/bin/python bridge.py`. The first time it sends a message, macOS asks whether it may control Messages — click Allow.
3. Keep it running: replace `YOU` in `com.example.imessage-agent.plist` with your username, put it in `~/Library/LaunchAgents/`, then run
   `launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.example.imessage-agent.plist`.
   After editing the config, restart with `launchctl kickstart -k gui/$(id -u)/com.example.imessage-agent`.

`.venv/bin/python bridge.py --send "hi"` sends a test message to `self_chat`.

## Configuration (config.json)

| Field | Meaning |
|---|---|
| `chats` | The allowlist; keys are chat guids (as listed by `--chats`) |
| `chats.*.agent` | `claude` (default) / `dsh` / `dsh-web` |
| `chats.*.allow_from` | Who may command it: `["me"]` means only you; you can add the other side's number/email, or `"*"`. **Anyone who can command the agent gets all of your permissions** — add people with care |
| `chats.*.trigger` | In groups, a message must start with this to count as a command (e.g. `"@c"`); `null` means every message counts |
| `chats.*.prefix` / `cwd` / `model` / `dsh_profile` | Per-conversation overrides |
| `self_chat` | Guid of your message-yourself conversation (used for echo folding; leave it out of `chats` if you don't want it routed) |
| `approval_chat` | Where Claude's permission prompts are sent |
| `agent_prefix` | Prefix for Claude's replies, `🤖 ` by default |
| `permission_mode` | Claude's default permission mode: `default` / `acceptEdits` / `auto` / `plan` / … |
| `max_backlog_age_s` | After a restart, messages older than this many seconds are skipped |
| `dsh_web_url` / `dsh_web_secret_file` | Address and secret file for the dsh-web backend |

## Commands in a conversation

| Command | Claude | dsh-web |
|---|---|---|
| `/new` | New session | New session (a new `[iMessage]` session appears in the web UI) |
| `/stop` | Interrupt the current task and deny pending approvals | Cancel the current turn |
| `/status` | Status, queue, session, permission mode | — |
| `/mode <mode>` | Switch permission mode | — |
| `/sessions` `/use <id>` `/current` | — | List / adopt / show sessions |
| `/help` | Help | Help |

## Known limitations

- On the iPhone, the agent's replies also appear on "your" side of the thread; only the prefix tells them apart.
- Only the final reply is sent, not intermediate progress. Long replies are split every 3000 characters.
- Claude loads all of your user settings, skills and MCP servers every turn, so even a trivial turn costs about $0.15. You can set a cheaper `model` per conversation.
- `chat.db` is polled (once a second by default), not pushed.
- Message text is decoded from `attributedBody` typedstream (the `text` column is often empty on recent macOS); this may need updating if Apple changes the format.

## License

MIT
