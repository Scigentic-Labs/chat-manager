# chat-manager

A skill for managing AI conversation history across **Claude Code** and **Codex CLI**.

Scan, search, inspect, resume, redact secrets, and clean up session transcripts — invokable as a natural-language skill inside Claude Code or Codex, or directly from the command line.

## Features

- **Multi-source**: reads Claude Code (`~/.claude/projects/`) and Codex CLI (`~/.codex/sessions/`) in one table
- **Secret detection & redaction**: finds API keys, GitHub tokens, and other credentials; redacts in-place with backup
- **Cleanup analysis**: flags short sessions, low-signal openers, auto-generated task runs, and duplicate topics
- **Quarantine & restore**: safely moves sessions out of active storage; restore with one command
- **Purge quarantine**: built-in TTL deletion for old quarantined files (no cron needed)
- **Auto-update**: checks remote version on every invocation; semver-aware (won't downgrade)

---

## Using inside Claude Code (recommended)

chat-manager is designed to be invoked as a Claude Code skill. Once installed, you talk to it in plain language — Claude handles translating your intent into the right commands.

### Installation

Copy `chat_manager.py` and `SKILL.md` into your skills directory:

```bash
mkdir -p ~/.claude/skills/chat-manager
curl -sf https://raw.githubusercontent.com/Scigentic-Labs/chat-manager/main/chat_manager.py \
  -o ~/.claude/skills/chat-manager/chat_manager.py
curl -sf https://raw.githubusercontent.com/Scigentic-Labs/chat-manager/main/SKILL.md \
  -o ~/.claude/skills/chat-manager/SKILL.md
```

Or if you manage dotfiles, clone there and symlink:

```bash
# From your dotfiles directory
ln -s $(pwd)/claude-code/skills/chat-manager ~/.claude/skills/chat-manager
```

### Usage

Just type `/chat-manager` in Claude Code, then describe what you want:

```
/chat-manager
```

Claude will ask what you'd like to do, or you can be direct:

```
/chat-manager show me all my conversations
/chat-manager search for "langgraph"
/chat-manager clean up old sessions
/chat-manager find any API keys in my history
/chat-manager I want to resume the conversation about paper trading
```

Claude handles the workflow end-to-end: it runs the commands, shows you results, asks for confirmation before any destructive action, and presents options in plain language rather than raw paths.

### What the skill does automatically

- **On every invocation**: checks for updates and prompts if a newer version is available
- **Before quarantine/delete**: always shows a dry run and asks for confirmation
- **On cleanup**: groups candidates by reason (duplicates, low-signal, auto-generated tasks, secrets) so you can review before acting
- **On resume**: prints the exact terminal command to continue a session, including SSH variants for remote machines

---

## Using inside Codex CLI

chat-manager works equally well from Codex. The `SKILL.md` file doubles as an AGENTS.md-compatible instruction set that Codex picks up automatically when present in the project.

### Setup

Same installation as above. Then in any Codex session:

```
manage my chat history
show all conversations
search history for "fastapi"
clean up duplicate sessions
```

Codex reads `SKILL.md` for the skill definition and runs `chat_manager.py` directly. The workflow is the same — dry runs before destructive actions, confirmation required for `--apply` steps.

### Resuming a Codex session found by chat-manager

When `resume` finds a Codex session, it prints:

```
cd /your/project && codex --resume <session-id>
```

Note: Claude Code and Codex sessions cannot resume each other — each tool only resumes its own format.

---

## Direct CLI usage

You can also run `chat_manager.py` directly without any AI layer:

```bash
python3 ~/.claude/skills/chat-manager/chat_manager.py <command>
```

### `scan` — list all sessions

```bash
python3 chat_manager.py scan
python3 chat_manager.py scan --json   # also writes row→path map to stderr
```

### `search` — full-text search

```bash
python3 chat_manager.py search "langgraph"
```

### `inspect` — print full conversation

```bash
python3 chat_manager.py inspect /path/to/session.jsonl
```

### `resume` — get resume command

```bash
python3 chat_manager.py resume /path/to/session.jsonl
# prints: cd /project && claude --resume <session-id>
# or:     cd /project && codex --resume <session-id>
```

### `cleanup` — find cleanup candidates

```bash
python3 chat_manager.py cleanup
```

Flags sessions matching any of:
- 0–2 user messages
- Low-signal opening message (hi, test, 你好, …)
- Auto-generated task (Analyze this codebase for…, Analyze test coverage…)
- Duplicate topic (same first 3 user messages as another session)
- Sensitive content (API key, token)

### `secrets` — list sessions with credentials

```bash
python3 chat_manager.py secrets
```

### `redact-secrets` — redact credentials in-place

```bash
python3 chat_manager.py redact-secrets           # dry run
python3 chat_manager.py redact-secrets --apply   # redact + backup originals
```

Backups saved to `~/.claude/chat-manager-redaction-backups/<timestamp>/`.

### `quarantine` — move session out of active storage

```bash
python3 chat_manager.py quarantine /path/to/session.jsonl           # dry run
python3 chat_manager.py quarantine /path/to/session.jsonl --apply   # move
```

Sessions are moved to `~/.claude/chat-manager-quarantine/<timestamp>/`.

### `restore` — restore from quarantine

```bash
python3 chat_manager.py restore /path/in/quarantine.jsonl           # dry run
python3 chat_manager.py restore /path/in/quarantine.jsonl --apply   # restore
```

### `purge-quarantine` — permanently delete old quarantined files

```bash
python3 chat_manager.py purge-quarantine --days 7           # dry run (default: 7 days)
python3 chat_manager.py purge-quarantine --days 7 --apply   # delete
```

---

## Multi-machine setup

To see sessions from multiple machines in one table, create `~/.claude/chat-manager.config.json`:

```json
{
  "sources": [
    {"type": "claude-code", "path": "~/.claude/projects/",                       "machine": "local"},
    {"type": "codex",       "path": "~/.codex/sessions/",                        "machine": "local"},
    {"type": "claude-code", "path": "/Volumes/remote/.claude/projects/",         "machine": "workstation"},
    {"type": "codex",       "path": "/Volumes/remote/.codex/sessions/",          "machine": "workstation"}
  ]
}
```

When multiple sources are configured, the scan table gains a `Machine` and `Source` column.

---

## Supported formats

| Tool | Session format | Discovery pattern |
|------|---------------|-------------------|
| Claude Code | JSONL, one message per line | `~/.claude/projects/**/*.jsonl` |
| Codex CLI | JSONL, `rollout-*.jsonl`, `source=cli` only | `~/.codex/sessions/**/rollout-*.jsonl` |

## Safety rules

- Never deletes without explicit `--apply`
- Active session cannot be quarantined
- Prefer `redact-secrets` before quarantining sessions with credentials
- `restore` fails if the destination path already exists

## Changelog

| Version | Changes |
|---------|---------|
| 2.4.0 | `restore` command; `purge-quarantine` command; auto-generated task detection in `cleanup`; semver-aware update check |
| 2.3.0 | Codex CLI support; multi-source config; secret redaction with backups |
| 2.1.0 | Initial release: scan, search, inspect, resume, quarantine, cleanup |
