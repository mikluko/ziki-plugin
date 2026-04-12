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

Read `.claude/ziki.local.md` in the current project directory. This file contains:
- YAML frontmatter with `vault_owner`, `vault_repo`, `vault_branch`
- Markdown body with vault access instructions (how to read and write files)

If the file does not exist, stop and tell the user to run `/ziki-setup` first.

**Follow the access instructions in the markdown body exactly.** They describe which
tools or CLI commands to use for reading and writing vault files. These instructions
were tested during setup and are specific to the user's environment.

## Input sources

Handle any of the following (infer from context, no user input required):

- **URL**: fetch full page content with `WebFetch`, extract clean text, use page title
  as filename
- **File path**: read the file, use its name as filename
- **Topic / conversation excerpt**: synthesize a draft page from current session context
- **Combination**: a URL discussed in context plus surrounding analysis

## How to file

1. **Read settings**: parse `.claude/ziki.local.md` for vault config and access method
2. **Determine content**: fetch URL if needed, read file if needed, or synthesize from
   conversation
3. **Deduplicate**: read `wiki/_index.md` and list `_inbox/` directory from the vault.
   If a near-duplicate exists, skip silently.
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

6. **Read current `wiki/_log.md`** from the vault
7. **Write both files** to the vault in a single commit (follow the write instructions
   from settings):
   - `_inbox/<filename>.md` (new file)
   - `wiki/_log.md` (append: `## [YYYY-MM-DD] inbox | <title>`)

   Commit message: `ziki: add to inbox — <title>`

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
