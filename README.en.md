# Lark Safe Write — Claude Code Skill

A Claude Code Skill for safe writing to Lark/Feishu wiki. Enforces a four-step closed loop: **pre-check → backup → write → verify**, preventing accidental overwrites or silent write failures.

> [中文版](README.md)

---

## Problems Solved

When writing to Feishu knowledge base via lark-cli, you may have encountered:

- **Accidental overwrites**: Direct `update` calls wipe existing content with no recovery path
- **False success**: API returns 200 but the content wasn't actually updated
- **Path-as-content bug**: Using `--markdown /path/to/file.md` writes the path string instead of the file content
- **Permission trial-and-error**: Calling APIs repeatedly without pre-checking bot permissions

This Skill eliminates these issues through mandatory workflow constraints.

---

## Installation

### Claude Code

```bash
# 1. Clone to your local skills directory
git clone https://github.com/duo-lark/lark-safe-write.git ~/.claude/skills/lark-safe-write

# 2. Trigger the Skill
/lark-safe-write
```

### Claude Chat / Claude CoWork

Upload the `SKILL.md` file directly into the conversation.

---

## Core Workflow

```
Pre-check → Backup → Write → Verify
```

### 1. Pre-check (mandatory)

- Confirm `--bot` identity flag
- Run `lark-cli auth test` to validate token
- Parse wiki_token / node_token
- For first-time space access, confirm bot has permission

### 2. Backup (required for updates/overwrites)

- Read existing document content
- Save to `.backups/YYYYMMDD-HHMMSS-{token}.md`
- Report backup path to user

### 3. Write

- **Create new doc**: Identify `parent_node_token`, use `lark-cli wiki +node-create`
- **Update existing doc**: Use **stdin pipe** (never `--markdown /path`)
- Wait 2 seconds after write (Feishu async indexing delay)

### 4. Verify (mandatory)

- Re-read the document
- Compare title, first paragraph, key markers
- Return explicit verification result and document URL

---

## Error Handling

| Error Scenario | Handling |
|---------------|----------|
| `auth test` failure | Halt execution, prompt to check env vars and bot permissions |
| Backup read failure (doc not found) | Continue as "create new doc" flow |
| Backup read failure (permission denied) | Halt execution, prompt to add permissions in Feishu console |
| Write failure (4xx/5xx) | Halt execution, preserve backup, skip verification |
| Verification failure (content mismatch) | Halt execution, preserve backup, do not auto-retry overwrite |

---

## Dependencies

- [lark-cli](https://open.larksuite.com/document/uAjLw4CM/ukTMukTMukTM/cli/overview) — configured with bot identity and permissions

---

## License

MIT License
