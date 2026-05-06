---
name: chat-manager
description: List, inspect, search, and delete AI chat history across tools and machines
user_invocable: true
---

# Chat Manager

Manage conversation history from Claude Code, Codex CLI, and other AI tools.
Reads sources from `~/.claude/chat-manager.config.json` (defaults to `~/.claude/projects/`).

All logic lives in `chat_manager.py`. Each command below is a thin wrapper.

---

## Update Check (run first, every invocation)

```bash
_LOCAL_VER=$(python3 ~/.claude/skills/chat-manager/chat_manager.py version 2>/dev/null || echo "unknown")
_AUTO_FILE=$HOME/.claude/chat-manager.auto-update
_REMOTE_VER=""

# If auto-update enabled, skip the prompt and upgrade silently
if [ -f "$_AUTO_FILE" ] && [ "$(cat "$_AUTO_FILE")" = "true" ]; then
  _REMOTE_VER=$(curl -sf --max-time 3 \
    https://raw.githubusercontent.com/Scigentic-Labs/chat-manager/main/chat_manager.py \
    | grep '^__version__' | head -1 | cut -d'"' -f2 2>/dev/null || echo "")
  # semver-aware: only upgrade if remote > local
  if [ -n "$_REMOTE_VER" ] && [ "$_REMOTE_VER" != "$_LOCAL_VER" ]; then
    _semver_gt() {
      local IFS=.; local i; read -r -a a <<< "$1"; read -r -a b <<< "$2"
      for i in 0 1 2; do
        local x="${a[$i]:-0}" y="${b[$i]:-0}"
        [ "$x" -gt "$y" ] && return 0; [ "$x" -lt "$y" ] && return 1
      done; return 1
    }
    _semver_gt "$_REMOTE_VER" "$_LOCAL_VER" && echo "AUTO_UPGRADE $_LOCAL_VER $_REMOTE_VER"
  fi
else
  _REMOTE_VER=$(curl -sf --max-time 3 \
    https://raw.githubusercontent.com/Scigentic-Labs/chat-manager/main/chat_manager.py \
    | grep '^__version__' | head -1 | cut -d'"' -f2 2>/dev/null || echo "")
  if [ -n "$_REMOTE_VER" ] && [ "$_REMOTE_VER" != "$_LOCAL_VER" ]; then
    _semver_gt() {
      local IFS=.; local i; read -r -a a <<< "$1"; read -r -a b <<< "$2"
      for i in 0 1 2; do
        local x="${a[$i]:-0}" y="${b[$i]:-0}"
        [ "$x" -gt "$y" ] && return 0; [ "$x" -lt "$y" ] && return 1
      done; return 1
    }
    _semver_gt "$_REMOTE_VER" "$_LOCAL_VER" && echo "UPGRADE_AVAILABLE $_LOCAL_VER $_REMOTE_VER"
  fi
fi
```

**If `AUTO_UPGRADE` is printed:** run the inline upgrade silently, then tell user "chat-manager updated v{old} → v{new} (auto-update is on)." and continue.

**If `UPGRADE_AVAILABLE` is printed:**

Use AskUserQuestion:
- header: "Update available"
- question: "chat-manager **v{new}** is available (you're on v{old})."
- options:
  - "Yes, update now" → run the inline upgrade below, then continue with original command
  - "Enable auto-update" → run the inline upgrade below, then write `echo "true" > ~/.claude/chat-manager.auto-update`; tell user "Updated to v{new}. Future updates will install automatically."; continue with original command

**Inline upgrade flow:**

```bash
# Download new version
_DEST=~/.dotfiles/claude-code/skills/chat-manager/chat_manager.py
# Fallback if dotfiles path not found
[ ! -f "$_DEST" ] && _DEST=~/.claude/skills/chat-manager/chat_manager.py

curl -sf --max-time 15 \
  https://raw.githubusercontent.com/Scigentic-Labs/chat-manager/main/chat_manager.py \
  -o "$_DEST" && echo "UPDATE_OK" || echo "UPDATE_FAILED"
```

- **UPDATE_OK:** clear snooze (`rm -f ~/.claude/chat-manager.update-snoozed`), tell user "Updated to v{new}!"; continue with original command
- **UPDATE_FAILED:** tell user "Update failed — still on v{old}. Try manually: `curl -sf https://raw.githubusercontent.com/Scigentic-Labs/chat-manager/main/chat_manager.py -o ~/.claude/skills/chat-manager/chat_manager.py`"; continue with original command

**If no `UPGRADE_AVAILABLE` output:** continue silently.

---

## List all records

Run:
```bash
python3 ~/.claude/skills/chat-manager/chat_manager.py scan
```
Display the table, then wait for the user's instruction.

For a JSON row→path map alongside the table:
```bash
python3 ~/.claude/skills/chat-manager/chat_manager.py scan --json
```
The JSON map is written to stderr — read it separately if needed.

## Search

```bash
python3 ~/.claude/skills/chat-manager/chat_manager.py search "KEYWORD"
```

## Inspect a record

Look up the row's `path` from the previous scan output, then:
```bash
python3 ~/.claude/skills/chat-manager/chat_manager.py inspect "<path>"
```

## Resume a session

```bash
python3 ~/.claude/skills/chat-manager/chat_manager.py resume "<path>"
```
Script prints the exact `cd ... && claude --resume ...` command (or Codex equivalent).
For records from other machines, it also prints the SSH variant.

## Quarantine records

Confirm with user first, then dry-run:

```bash
python3 ~/.claude/skills/chat-manager/chat_manager.py quarantine "<path>"
```

Apply only after confirmation:

```bash
python3 ~/.claude/skills/chat-manager/chat_manager.py quarantine "<path>" --apply
```

The script moves the transcript under `~/.claude/chat-manager-quarantine/<timestamp>/`
and prints a restore command. Re-run scan after.

## Restore from quarantine

Dry-run first:

```bash
python3 ~/.claude/skills/chat-manager/chat_manager.py restore "<quarantine-path>"
```

Apply only after confirmation:

```bash
python3 ~/.claude/skills/chat-manager/chat_manager.py restore "<quarantine-path>" --apply
```

Moves the file back to its original location. Fails if destination already exists.

## Purge quarantine

Remove quarantined files older than N days (default 7). Dry-run first:

```bash
python3 ~/.claude/skills/chat-manager/chat_manager.py purge-quarantine --days 7
```

Apply only after confirmation:

```bash
python3 ~/.claude/skills/chat-manager/chat_manager.py purge-quarantine --days 7 --apply
```

## Cleanup candidates

```bash
python3 ~/.claude/skills/chat-manager/chat_manager.py cleanup
```
Present candidates grouped by reason. Always confirm before deleting.

## Secret scan

```bash
python3 ~/.claude/skills/chat-manager/chat_manager.py secrets
```

Lists sessions containing credential-shaped values without printing the values.

## Redact secrets

Dry-run first:

```bash
python3 ~/.claude/skills/chat-manager/chat_manager.py redact-secrets
```

Apply only after confirmation:

```bash
python3 ~/.claude/skills/chat-manager/chat_manager.py redact-secrets --apply
```

The script creates backups under `~/.claude/chat-manager-redaction-backups/<timestamp>/`
before modifying any transcript.

## Important

- NEVER delete without explicit confirmation
- The current active session cannot be deleted
- Prefer `redact-secrets` before deleting sensitive records
- Prefer quarantine over direct `rm`
- Always show updated table after deletions
