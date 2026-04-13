---
name: ziki-inbox
description: This skill should be used to process pending files in the Ziki vault `_inbox/` directory — classify, assess, enrich with research, re-verify against ground truth, and promote quality content into `wiki/` pages. Invoked by the `/ziki-inbox` command and by the scheduled hourly inbox-processing agent. Fully autonomous; no human review.
version: 1.0.0
---

# Ziki Inbox

Process all pending files in `_inbox/` of the Ziki vault. This is a fully
autonomous operation. No human review is needed.

This skill is used both explicitly (user request or `/ziki-inbox` command) and
autonomously (the hourly scheduled inbox-processing agent described in
`README.md`).

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
- **Start from the inbox file's `ground_truth:` field** — the `ziki-add` skill is
  supposed to have captured the primary sources at filing time. Read each
  `_refs/` snapshot listed there and treat them as the authoritative baseline.
- For draft pages: use web search to verify facts, fill gaps, and find current
  information (drafts are often written from memory and may be stale)
- For raw sources: identify all concepts and entities; determine which existing wiki
  pages to update and which new pages to create
- Identify wikilinks the content missed; find connections to existing pages
- **Supplement ground truth if needed**: if the inbox file's `ground_truth:` is
  empty or incomplete, find additional authoritative URLs (official docs,
  specifications, primary sources) and cache substantial ones to `_refs/`.
  Missing ground truth on a factual draft is a quality issue — note it in
  `wiki/_log.md` so the `ziki-add` pathway can be improved.
- **Never replace ground truth with your own synthesis** — the whole point is
  that the wiki page is derivative and must be traceable back to primary sources.

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

### 6. Cache additional reference material

`_refs/` caching conventions (naming, frontmatter, when-to-cache, when-not-to,
web/repo-file/session origin types) are defined in the `ziki-add` skill under
"Reference snapshots (`_refs/`)". Follow them here too — do not invent a
different scheme.

At this stage you are only responsible for **enrichment-time** snapshots: any
extra ground-truth sources you discovered during research (step 3) that were
not already cached by the original `ziki-add` call. The draft's existing
`_refs/` snapshots should be preserved as-is, not re-fetched or renamed.

Before creating a new snapshot:
- List `_refs/` and reuse an existing file if it covers the same source
- For repo files, always pin to a commit SHA — never reference `main`/`HEAD`
- One file per source (URL, repo-file-at-commit, or pasted artifact)

When a new `_refs/` snapshot is created, add its path to the wiki page's
`ground_truth:` field alongside any canonical URL.

### 7. Clean up inbox

After promoting an inbox file, you may:
- **Delete it** if all its content has been fully absorbed into wiki pages
- **Edit it** to mark `status: promoted` and add `promoted_to: [wiki/page.md]`
- **Merge** multiple related inbox files into one before promoting
- **Split** a large inbox file into multiple before promoting

The inbox is a mutable staging area. Do not treat it as an archive.

**Never delete `_refs/` snapshots as part of inbox cleanup.** Once a snapshot
has been referenced by a promoted wiki page's `ground_truth:` field (or by any
remaining inbox file), it is a live dependency of the wiki and must be
preserved. The rule is simple: `_inbox/` is staging; `_refs/` is permanent
ground-truth infrastructure. Removing a `_refs/` file orphans every page that
cites it and breaks the re-verification chain on all future inbox passes.

A `_refs/` snapshot may be removed only if it has zero inbound references
from both `wiki/` and `_inbox/`. Do not perform that sweep as part of routine
inbox processing; leave stale-refs cleanup to a dedicated future pass.

### 8. Update supporting files

After processing all files, prepare updates to:

1. **`.manifest.json`**: add an entry for each processed file with:
   - `sha256` — content hash of the inbox file
   - `ingested` — date promoted (YYYY-MM-DD)
   - `pages` — array of wiki pages created or updated from this file
   - `refs` — array of `_refs/` paths the inbox file contributed or relied on
     (its own `ground_truth:` entries plus any snapshots added during step 6
     enrichment). This makes orphan detection tractable: a future pass can
     compute `{all _refs/} − {refs referenced by any manifest entry or any
     live inbox file}` to find stale snapshots.
2. **`wiki/_index.md`**: add new promoted pages to the map under the appropriate
   section. **This file is not just for humans — it is the recall surface that
   the `ziki-recall` skill and the SessionStart hook load to make wiki content
   discoverable from a fresh session.** Treat it with care:
   - Every promoted wiki page must appear in the index
   - Every entry must have: the page path, the title, and a one-sentence summary
     that names the concept clearly enough that a model scanning the index can
     decide whether to recall the page
   - Group entries by topic area; keep top-level headings stable so the primed
     context remains coherent across sessions
   - Remove entries for deleted/merged pages
   - If the index grows past ~6KB, split sub-sections into linked sub-indices
     (e.g. `wiki/_index-nats.md`) and keep `wiki/_index.md` as a terse table of
     contents — the SessionStart hook truncates anything larger than 6KB
   - Never let `wiki/_index.md` drift out of sync with `wiki/`; a page that
     exists but is not indexed is effectively invisible to future sessions
3. **`wiki/_log.md`**: append a single dated entry for this inbox pass recording
   what was processed. Format:

   ```markdown
   ## YYYY-MM-DD — inbox pass

   - Promoted: `_inbox/foo.md` → `wiki/foo.md` (N refs)
   - Rejected: `_inbox/bar.md` (reason: too thin)
   - Quality issues: `_inbox/baz.md` arrived with empty `ground_truth:`
     (supplemented during enrichment) — feedback for `ziki-add` pathway
   ```

   This log is append-only and gives a running audit trail of what the wiki
   ingested, what was rejected, and where the upstream `ziki-add` capture
   pipeline needs improvement. Keep entries terse — one bullet per file.

### 9. Commit and push

Write all changes to the vault in a single commit (follow the write instructions from
`~/.claude/ziki.md`):

- All new/updated `wiki/` pages
- Any new `_refs/` snapshots (never delete existing ones)
- Updated or deleted `_inbox/` files
- Updated `.manifest.json`
- Updated `wiki/_index.md`
- Updated `wiki/_log.md`

Commit message: `wiki: process inbox #N — X promoted, Y rejected`

`#N` is the inbox-pass sequence number. Determine it by counting prior commits
on the vault branch whose message matches `^wiki: process inbox` and adding 1.
The first pass is `#1`. Do not use the date here — git already records it in
the commit metadata.

## Hard rules

- **NEVER** read or write the local filesystem for vault operations
- **NEVER** use `~/Documents/ziki/` or any other local path
- **NEVER** invent sources or citations
- **NEVER** create a near-duplicate wiki page (update the existing one instead)
- **NEVER** delete a `_refs/` snapshot during routine inbox processing — refs
  are permanent wiki dependencies, not inbox staging
- **ALWAYS** populate `ground_truth:` with at least one authoritative URL per wiki page
- **ALWAYS** record the inbox file's `refs` in the `.manifest.json` entry so
  orphan detection stays tractable
- **ALWAYS** append a single dated entry to `wiki/_log.md` per inbox pass
- **ALWAYS** use provenance markers: `^[inferred]`, `^[ambiguous]`, `^[stale]`
- **ALWAYS** write all changes in a single commit at the end
