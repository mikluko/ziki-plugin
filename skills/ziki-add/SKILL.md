---
name: ziki-add
description: This skill should be used when the user asks to "add to ziki", "save to ziki", "file this in ziki", "add to inbox", or "save this to the knowledge base". Also used internally by the Stop and PreCompact hooks to autonomously file inbox-worthy content from the current session.
version: 1.3.0
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

- **URL**: fetch full page content, extract clean text, use page title as filename
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

6. **Cache reference material** (when filing a URL or external source): if the content
   is substantial and worth preserving for future verification, save a snapshot to
   `_refs/`. See "Reference snapshots" below. Add the `_refs/` path to the inbox
   file's `sources:` field.

7. **Write files** to the vault in a single commit:
   - `_inbox/<filename>.md`
   - `_refs/<snapshot>.md` (if caching a reference)

   Commit message: `ziki: add to inbox — <title>`

## Reference snapshots (`_refs/`)

When filing content from a URL or external document, you may cache a content snapshot
in `_refs/` to save bandwidth on future re-fetches and preserve content that might
disappear.

**When to cache**: substantial articles, documentation pages, or reference material
that wiki pages will cite as ground truth. Do NOT cache trivial pages, search results,
or ephemeral content.

**Naming convention**: `<domain>--<kebab-title>.md`

Examples:
- `docs-nats-io--jetstream-overview.md`
- `extendedbrain-substack-com--wiki-that-writes-itself.md`
- `kubernetes-io--pod-lifecycle.md`

**Frontmatter format**:

```markdown
---
url: "https://docs.nats.io/nats-concepts/jetstream"
fetched: YYYY-MM-DD
content_type: documentation | article | specification | transcript
---

# Page Title

Extracted content here (clean markdown, not raw HTML).
```

**Rules for `_refs/`**:
- Content should be clean, extracted markdown (not raw HTML)
- Include only the substantive content, not navigation or ads
- One file per source URL
- Deduplicate: check `_refs/` before creating a new snapshot

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
