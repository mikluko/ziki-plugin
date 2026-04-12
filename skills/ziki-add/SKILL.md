---
name: ziki-add
description: This skill should be used when the user asks to "add to ziki", "save to ziki", "file this in ziki", "add to inbox", or "save this to the knowledge base". Also used internally by the Stop and PreCompact hooks to autonomously file inbox-worthy content from the current session.
version: 1.0.0
---

# Ziki Add

File content into the Ziki knowledge base inbox via the remote vault repository.

This skill is used both explicitly (user request or `/ziki-add` command) and
autonomously (Stop/PreCompact hooks).

## Vault configuration

Read `.claude/ziki.local.md` in the current project directory to get vault settings.
Parse the YAML frontmatter to extract `vault_owner`, `vault_repo`, and `vault_branch`.

If the file does not exist, stop and tell the user to run `/ziki-setup` first.

Example settings file:

```markdown
---
vault_owner: mikluko
vault_repo: ziki
vault_branch: main
---
```

## Vault access

All reads and writes go through the **remote git repository**, never the local
filesystem.

### How to read vault files

Use `mcp__github__get_file_contents` with `owner` and `repo` from settings, and
the file path. Example: read `wiki/_index.md` to check for duplicates.

### How to write vault files

Use `mcp__github__push_files` to write multiple files in a single commit, or
`mcp__github__create_or_update_file` for a single file. Use `branch` from settings.

When updating an existing file, you must provide its SHA. Get it from the
`get_file_contents` response.

### Fallback

If GitHub MCP tools are unavailable, use `gh api repos/<owner>/<repo>/contents/<path>`
for reads and `gh api -X PUT repos/<owner>/<repo>/contents/<path>` for writes,
substituting owner and repo from settings.

## Input sources

Handle any of the following (infer from context, no user input required):

- **URL**: fetch full page content with `WebFetch`, extract clean text, use page title
  as filename
- **File path**: read the file, use its name as filename
- **Topic / conversation excerpt**: synthesize a draft page from current session context
- **Combination**: a URL discussed in context plus surrounding analysis

## How to file

1. **Read settings**: parse `.claude/ziki.local.md` for vault owner, repo, and branch
2. **Determine content**: fetch URL if needed, read file if needed, or synthesize from
   conversation
3. **Deduplicate**: read `wiki/_index.md` and list `_inbox/` directory via
   `mcp__github__get_file_contents`. If a near-duplicate exists, skip silently.
4. **Choose filename**: `<kebab-case-title>.md` (short, descriptive, no dates)
5. **Prepare the inbox file** with this format:

```markdown
---
title: "<Title>"
type: concept | entity | comparison | synthesis | reference
tags: [tag1, tag2]
sources: []
created: YYYY-MM-DD
status: inbox
---

# Title

Content here.
```

6. **Read current `wiki/_log.md`** to get its SHA for updating
7. **Push both files** using `mcp__github__push_files` with branch from settings:
   - `_inbox/<filename>.md` (new file)
   - `wiki/_log.md` (append: `## [YYYY-MM-DD] inbox | <title>`)

   Commit message: `ziki: add to inbox — <title>`

8. **Do not** commit separately or push separately. Use a single `push_files` call.

## Tags

Read `_meta/taxonomy.md` from the vault to select tags. Extend the vocabulary there if
a new tag is clearly needed.

## Rules

- **NEVER** prompt the user for confirmation or input
- **NEVER** announce what you are doing unless the user explicitly invoked `/ziki-add`
- **NEVER** read or write the local filesystem for vault operations
- **NEVER** invent sources or citations
- If content is too thin (stub, one-liner with no substance), skip silently
- One `_inbox/` file per distinct concept; do not bundle unrelated things
- Deduplicate: check `wiki/_index.md` and existing `_inbox/` files before creating
  a near-duplicate
