---
name: ziki-add
description: This skill should be used when the user asks to "add to ziki", "save to ziki", "file this in ziki", "add to inbox", or "save this to the knowledge base". Also used internally by the Stop and PreCompact hooks to autonomously file inbox-worthy content from the current session.
version: 1.1.0
---

# Ziki Add

File content into the Ziki knowledge base inbox via the remote vault repository.

This skill is used both explicitly (user request or `/ziki-add` command) and
autonomously (Stop/PreCompact hooks).

## CRITICAL: vault access method

**You MUST read `~/.claude/ziki.md` FIRST, before doing anything else.** This file
contains the vault repository coordinates (YAML frontmatter) and the exact tools or
commands to use for reading and writing vault files (markdown body).

**Do NOT write to the local filesystem.** The vault is a remote git repository. Even
if you see a local checkout of the vault (e.g. `~/Documents/ziki/`), do not use it.
All reads and writes go through the remote access method described in `~/.claude/ziki.md`.

If `~/.claude/ziki.md` does not exist, stop and tell the user to run `/ziki-setup`.

## Input sources

Handle any of the following (infer from context, no user input required):

- **URL**: fetch full page content with `WebFetch`, extract clean text, use page title
  as filename
- **File path**: read the file, use its name as filename
- **Topic / conversation excerpt**: synthesize a draft page from current session context
- **Combination**: a URL discussed in context plus surrounding analysis

## How to file

1. **Read `~/.claude/ziki.md`**: parse frontmatter for vault coordinates, read body
   for access instructions
2. **Determine content**: fetch URL if needed, read file if needed, or synthesize from
   conversation
3. **Deduplicate**: read `wiki/_index.md` and list `_inbox/` directory from the vault
   (using the remote access method). If a near-duplicate exists, skip silently.
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

6. **Write the file** to the vault using the access method from `~/.claude/ziki.md`:
   - `_inbox/<filename>.md` (new file)

   Commit message: `ziki: add to inbox — <title>`

## Tags

Read `_meta/taxonomy.md` from the vault to select tags. Extend the vocabulary there if
a new tag is clearly needed.

## Rules

- **NEVER** prompt the user for confirmation or input
- **NEVER** announce what you are doing unless the user explicitly invoked `/ziki-add`
- **NEVER** read or write the local filesystem for vault operations
- **NEVER** use `~/Documents/ziki/` or any other local path
- **NEVER** invent sources or citations
- If content is too thin (stub, one-liner with no substance), skip silently
- One `_inbox/` file per distinct concept; do not bundle unrelated things
- Deduplicate: check `wiki/_index.md` and existing `_inbox/` files before creating
  a near-duplicate
