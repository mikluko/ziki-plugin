---
description: Process pending files in the Ziki knowledge base inbox, enrich with research, and promote quality content into the wiki
---

Process all pending files in `_inbox/` of the Ziki vault. This is a fully autonomous
operation. No human review is needed.

## Vault access

All reads and writes go through the **remote git repository**, never the local
filesystem. The vault repo is `mikluko/ziki` on GitHub, branch `main`.

**Reading files**: use `mcp__github__get_file_contents` with `owner: "mikluko"`,
`repo: "ziki"`.

**Writing files**: use `mcp__github__push_files` to push all changes in a single
commit at the end.

**Fallback** (if GitHub MCP tools are unavailable): use `gh api` CLI against
`repos/mikluko/ziki/contents/<path>`.

## Setup

Before processing, orient yourself by reading these files from the vault:

1. `AGENTS.md` — vault schema, page format, hard rules, provenance markers
2. `wiki/_index.md` — existing wiki pages and their topics
3. `wiki/_log.md` (last 20 entries) — what has already been processed
4. `.manifest.json` — which `_inbox/` files are already ingested (skip these)
5. `_meta/taxonomy.md` — canonical tag vocabulary

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

To reject: note `status: rejected` and `rejection_reason: <one line>` for the
frontmatter update. Do not delete the file. Log it. Move on.

**Important**: do not modify the body content of `_inbox/` files. Only update
frontmatter status fields.

### 3. Research and enrich

For files that pass assessment:
- Search `wiki/` for related pages (read directory listing, then relevant files)
- For draft pages: use `perplexity_ask` or web search to verify facts, fill gaps, and
  find current information (drafts are often written from memory and may be stale)
- For raw sources: identify all concepts and entities; determine which existing wiki
  pages to update and which new pages to create
- Identify wikilinks the content missed; find connections to existing pages

### 4. Promote

Create or update `wiki/` pages following the AGENTS.md page format:

```markdown
---
title: "<Page Title>"
type: concept | entity | comparison | synthesis | reference
tags: [tag1, tag2]
sources: [_inbox/foo.md]
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

Minimum bar for a promoted page:
- Complete frontmatter (`title`, `type`, `tags`, `sources`, `created`, `updated`)
- `> [!abstract]` TL;DR callout
- At least one `[[wikilink]]` to an existing wiki page
- All claims sourced or marked `^[inferred]`

**Raw sources** may produce multiple wiki pages. Merge into existing pages where
appropriate. Prefer updating an existing page over creating a near-duplicate.

**Draft pages** should be refined and expanded to full wiki quality: complete
frontmatter, verified facts, wikilinks, provenance markers.

### 5. Update supporting files

After processing all files, prepare updates to:

1. **`.manifest.json`**: add an entry for each processed file with `sha256`, `ingested`
   date, and `pages` array listing wiki pages created/updated
2. **`wiki/_index.md`**: add new promoted pages to the hand-curated map under the
   appropriate section
3. **`wiki/_log.md`**: append one entry per file:
   - `## [YYYY-MM-DD] ingest | <title>` (promoted)
   - `## [YYYY-MM-DD] rejected | <title> — <reason>` (rejected)

### 6. Commit and push

Push all changes in a single commit using `mcp__github__push_files`:

- All new/updated `wiki/` pages
- Updated `_inbox/` files (frontmatter status changes only)
- Updated `.manifest.json`
- Updated `wiki/_index.md`
- Updated `wiki/_log.md`

Commit message: `wiki: process inbox [YYYY-MM-DD] — N promoted, M rejected`

## Hard rules

- **NEVER** read or write the local filesystem for vault operations
- **NEVER** edit content of `_inbox/` files (only update frontmatter status fields)
- **NEVER** invent sources or citations
- **NEVER** create a near-duplicate wiki page (update the existing one instead)
- **ALWAYS** use provenance markers: `^[inferred]`, `^[ambiguous]`, `^[stale]`
- **ALWAYS** log every file processed to `wiki/_log.md`
- **ALWAYS** push all changes in a single commit at the end
