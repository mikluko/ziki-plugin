---
description: Process pending files in the Ziki knowledge base inbox, enrich with research, and promote quality content into the wiki
---

Process all pending files in `_inbox/` of the Ziki vault. This is a fully autonomous
operation. No human review is needed.

## CRITICAL: vault access method

**You MUST read `~/.claude/ziki.md` FIRST, before doing anything else.** This file
contains the vault repository coordinates (YAML frontmatter) and the exact tools or
commands to use for reading and writing vault files (markdown body).

**Do NOT write to the local filesystem.** The vault is a remote git repository. Even
if you see a local checkout of the vault (e.g. `~/Documents/ziki/`), do not use it.
All reads and writes go through the remote access method described in `~/.claude/ziki.md`.

If `~/.claude/ziki.md` does not exist, stop and tell the user to run `/ziki-setup`.

## Setup

Before processing, orient yourself by reading these files from the vault:

1. `AGENTS.md` — vault schema, page format, hard rules, provenance markers
2. `wiki/_index.md` — existing wiki pages and their topics
3. `.manifest.json` — which `_inbox/` files are already ingested (skip these)
4. `_meta/taxonomy.md` — canonical tag vocabulary

Then list the `_inbox/` directory to discover pending files.

## Processing loop

For each file in `_inbox/` not present in `.manifest.json`:

### 1. Classify

Determine the file type:
- **Raw source**: article, transcript, data dump, web clip (externally sourced,
  unstructured or lightly structured)
- **Draft page**: Claude-authored stub with frontmatter and partial content, intended
  to become a wiki page

### 2. Assess

Reject the file if ANY of the following apply:
- Content too thin to extract a single coherent fact or concept
- Subject or intention is unclear
- Near-duplicate of an existing wiki page with nothing new to add

To reject: set `status: rejected` and `rejection_reason: <one line>` in frontmatter.
Do not delete rejected files. Move on.

### 3. Research and enrich

For files that pass assessment:
- Search `wiki/` for related pages (read directory listing, then relevant files)
- For draft pages: use web search to verify facts, fill gaps, and find current
  information (drafts are often written from memory and may be stale)
- For raw sources: identify all concepts and entities; determine which existing wiki
  pages to update and which new pages to create
- Identify wikilinks the content missed; find connections to existing pages
- **Identify ground truth sources**: find authoritative URLs for the topic (official
  docs, specifications, primary sources). These go into the `ground_truth:` field.

### 4. Re-verify before writing

**Before creating or updating any wiki page**, re-verify the claims you are about to
write against the page's `ground_truth:` sources:

- If the page already exists and has `ground_truth:` URLs, fetch them (or read from
  `_refs/` if cached) and compare the incoming content against them.
- If creating a new page, fetch the ground truth URLs you identified in step 3 and
  verify your synthesis is consistent with them.
- If a claim contradicts a ground truth source, do NOT write it. Mark it with
  `^[ambiguous]` and note the discrepancy.
- If a ground truth URL is unreachable, note `^[stale]` on claims that depend on it.

This prevents error propagation: each write is anchored to authoritative sources,
not just to other wiki pages or inbox drafts.

### 5. Promote

Create or update `wiki/` pages following this format:

```markdown
---
title: "<Page Title>"
type: concept | entity | comparison | synthesis | reference
tags: [tag1, tag2]
sources: [_inbox/foo.md]
ground_truth: [https://docs.example.com/topic, _refs/example-com--topic.md]
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

# Page Title

Short definition or TL;DR.

> [!abstract]
> Expanded TL;DR. 2-4 sentences.

## Body

Content with [[wikilinks]] to related pages.

## Sources

- [[_inbox/foo.md]]

## See also

- [[Related Page]]
```

**Frontmatter fields**:
- `sources:` — where content came from (inbox files, other wiki pages)
- `ground_truth:` — authoritative references to verify claims against. URLs and/or
  `_refs/` paths. These are the "source of truth" for the page's claims.

Minimum bar for a promoted page:
- Complete frontmatter including `ground_truth:` (at least one authoritative URL)
- `> [!abstract]` TL;DR callout
- At least one `[[wikilink]]` to an existing wiki page
- All claims sourced or marked `^[inferred]`

**Raw sources** may produce multiple wiki pages. Merge into existing pages where
appropriate. Prefer updating an existing page over creating a near-duplicate.

**Draft pages** should be refined and expanded to full wiki quality: complete
frontmatter, verified facts, wikilinks, provenance markers.

### 6. Cache reference material

When you fetch a URL during enrichment that serves as ground truth for a wiki page,
consider caching it in `_refs/` to preserve the content and save future bandwidth.

**When to cache**: official documentation, specifications, substantial articles that
wiki pages will cite as ground truth. Do NOT cache trivial pages or search results.

**Naming**: `<domain>--<kebab-title>.md`
Examples: `docs-nats-io--jetstream-overview.md`, `kubernetes-io--pod-lifecycle.md`

**Frontmatter**:
```markdown
---
url: "https://docs.nats.io/nats-concepts/jetstream"
fetched: YYYY-MM-DD
content_type: documentation | article | specification | transcript
---
```

Store clean extracted markdown, not raw HTML. One file per source URL. Check `_refs/`
first to avoid duplicates.

When a `_refs/` snapshot is created, use the `_refs/` path in the wiki page's
`ground_truth:` field alongside the original URL.

### 7. Clean up inbox

After promoting an inbox file, you may:
- **Delete it** if all its content has been fully absorbed into wiki pages
- **Edit it** to mark `status: promoted` and add `promoted_to: [wiki/page.md]`
- **Merge** multiple related inbox files into one before promoting
- **Split** a large inbox file into multiple before promoting

The inbox is a mutable staging area. Do not treat it as an archive.

### 8. Update supporting files

After processing all files, prepare updates to:

1. **`.manifest.json`**: add an entry for each processed file with `sha256`, `ingested`
   date, and `pages` array listing wiki pages created/updated
2. **`wiki/_index.md`**: add new promoted pages to the hand-curated map under the
   appropriate section

### 9. Commit and push

Write all changes to the vault in a single commit (follow the write instructions from
`~/.claude/ziki.md`):

- All new/updated `wiki/` pages
- Any new `_refs/` snapshots
- Updated or deleted `_inbox/` files
- Updated `.manifest.json`
- Updated `wiki/_index.md`

Commit message: `wiki: process inbox [YYYY-MM-DD] — N promoted, M rejected`

## Hard rules

- **NEVER** read or write the local filesystem for vault operations
- **NEVER** use `~/Documents/ziki/` or any other local path
- **NEVER** invent sources or citations
- **NEVER** create a near-duplicate wiki page (update the existing one instead)
- **ALWAYS** populate `ground_truth:` with at least one authoritative URL per wiki page
- **ALWAYS** use provenance markers: `^[inferred]`, `^[ambiguous]`, `^[stale]`
- **ALWAYS** write all changes in a single commit at the end
