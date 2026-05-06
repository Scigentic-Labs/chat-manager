# chat-manager

A Claude Code skill for managing AI conversation history across **Claude Code** and **Codex CLI**.

Scan, search, inspect, resume, redact secrets, and clean up session transcripts — all from a single Python script.

## Features

- **Multi-source**: reads Claude Code (`~/.claude/projects/`) and Codex CLI (`~/.codex/sessions/`) in one table
- **Secret detection & redaction**: finds API keys, GitHub tokens, and other credentials; redacts in-place with backup
- **Cleanup analysis**: flags short sessions, low-signal openers, auto-generated task runs, and duplicate topics
- **Quarantine & restore**: safely moves sessions out of active storage; restore with one command
- **Purge quarantine**: built-in TTL deletion for old quarantined files (no cron needed)
- **Auto-update**: checks remote version on every invocation; semver-aware (won't downgrade)

## Installation

The skill file is `chat_manager.py`. All commands are thin wrappers around it.

Place `chat_manager.py` somewhere on your path, or invoke it directly:

```bash
python3 ~/.claude/skills/chat-manager/chat_manager.py <command>
```

### Optional: custom sources

Create `~/.claude/chat-manager.config.json` to add remote machines or non-default paths:

```json
{
  "sources": [
    {"type": "claude-code", "path": "~/.claude/projects/", "machine": "local"},
    {"type": "codex",       "path": "~/.codex/sessions/",  "machine": "local"},
    {"type": "claude-code", "path": "/Volumes/remote/.claude/projects/", "machine": "workstation"}
  ]
}
```

## Commands

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

## Supported formats

| Tool | Session format | Discovery pattern |
|------|---------------|-------------------|
| Claude Code | JSONL, one message per line | `~/.claude/projects/**/*.jsonl` |
| Codex CLI | JSONL, `rollout-*.jsonl`, `source=cli` only | `~/.codex/sessions/**/rollout-*.jsonl` |

## Safety rules

- Never delete without explicit `--apply`
- Active session cannot be quarantined
- Prefer `redact-secrets` before quarantining sessions with credentials
- `restore` fails if the destination path already exists

## Changelog

| Version | Changes |
|---------|---------|
| 2.4.0 | `restore` command; `purge-quarantine` command; auto-generated task detection in `cleanup`; semver-aware update check |
| 2.3.0 | Codex CLI support; multi-source config; secret redaction with backups |
| 2.1.0 | Initial release: scan, search, inspect, resume, quarantine, cleanup |
