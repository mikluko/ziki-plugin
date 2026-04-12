---
name: ziki-add
description: This skill should be used when the user asks to "add to ziki", "save to ziki", "file this in ziki", "add to inbox", or "save this to the knowledge base". Also used internally by the Stop and PreCompact hooks to autonomously file inbox-worthy content from the current session.
version: 1.0.0
---

# Ziki Add

File content into the Ziki knowledge base inbox via the remote vault repository.

This skill is used both explicitly (user request or `/ziki-add` command) and
autonomously (Stop/PreCompact hooks).

## Vault access

All reads and writes go through the **remote git repository**, never the local
filesystem. The vault repo is `mikluko/ziki` on GitHub, branch `main`.

### How to read vault files

Use `mcp__github__get_file_contents` with `owner: "mikluko"`, `repo: "ziki"`, and
the file path. Example: read `wiki/_index.md` to check for duplicates.

### How to write vault files

Use `mcp__github__push_files` to write multiple files in a single commit, or
`mcp__github__create_or_update_file` for a single file.

When updating an existing file, you must provide its SHA. Get it from the
`get_file_contents` response.

### Fallback

If GitHub MCP tools are unavailable, use `gh api repos/mikluko/ziki/contents/<path>`
for reads and `gh api -X PUT repos/mikluko/ziki/contents/<path>` for writes.

## Input sources

Handle any of the following (infer from context, no user input required):

- **URL**: fetch full page content with `WebFetch`, extract clean text, use page title
  as filename
- **File path**: read the file, use its name as filename
- **Topic / conversation excerpt**: synthesize a draft page from current session context
- **Combination**: a URL discussed in context plus surrounding analysis

## How to file

1. **Determine content**: fetch URL if needed, read file if needed, or synthesize from
   conversation
2. **Deduplicate**: read `wiki/_index.md` and list `_inbox/` directory via
   `mcp__github__get_file_contents`. If a near-duplicate exists, skip silently.
3. **Choose filename**: `<kebab-case-title>.md` (short, descriptive, no dates)
4. **Prepare the inbox file** with this format:

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

5. **Read current `wiki/_log.md`** to get its SHA for updating
6. **Push both files** using `mcp__github__push_files`:
   - `_inbox/<filename>.md` (new file)
   - `wiki/_log.md` (append: `## [YYYY-MM-DD] inbox | <title>`)

   Commit message: `ziki: add to inbox — <title>`

7. **Do not** commit separately or push separately. Use a single `push_files` call.

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
