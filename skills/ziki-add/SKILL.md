---
name: ziki-add
description: This skill should be used when the user asks to "add to ziki", "save to ziki", "file this in ziki", "add to inbox", or "save this to the knowledge base". Also used internally by the Stop and PreCompact hooks to autonomously file inbox-worthy content from the current session.
version: 1.4.0
---

# Ziki Add

File content into the Ziki knowledge base inbox via the remote vault repository.

This skill is used both explicitly (user request or `/ziki-add` command) and
autonomously (Stop/PreCompact hooks).

## Synthesis vs ground truth — read this first

Every inbox entry has two layers, and **you must capture both in the same pass**:

1. **Synthesis** — the draft page you write. Your interpretation, summary, or
   explanation of a topic. Derived, interpretive, and potentially wrong. Goes into
   `_inbox/<slug>.md`.
2. **Ground truth** — the primary sources your synthesis is anchored to. Stable,
   authoritative, and verifiable. Goes into `_refs/<snapshot>.md` (for cacheable
   content) and/or stable URLs recorded in the inbox file's `ground_truth:` field.

**The synthesis is worthless without ground truth.** A later `/ziki-inbox` pass —
possibly days later, in a different session — will re-verify your claims against
the ground-truth sources before promoting the draft to a wiki page. If you file a
synthesis with no ground truth, the re-verification step has nothing to check
against, and the knowledge you captured degrades into unverifiable hearsay.

**Capture ground truth NOW, not later.** This skill runs inside a live session
that has full context: which files were read, which commits were referenced,
which docs were fetched, which repo was explored. That context evaporates the
moment the session ends. `/ziki-inbox` can re-fetch external URLs, but it
**cannot** recover "the state of mikluko/yosh at commit abc1234 that the earlier
session was looking at". Ground truth that is not captured here is lost
permanently.

### What counts as ground truth

- **Source code read during the session**: snapshot the relevant files (README,
  key source files, config schemas, Helm values). Record the repo URL pinned to
  a commit SHA (`https://github.com/owner/repo/tree/<sha>`) — never bare `main`.
- **External documentation consulted**: official docs, RFCs, specifications,
  vendor guides. Cache substantial pages to `_refs/`.
- **URLs quoted or paraphrased in the conversation**: fetch and cache them.
- **Transcripts, papers, articles**: cache the clean extracted text to `_refs/`.
- **Code artifacts produced by the user in-session** (design docs, ADRs, config
  files they pasted): snapshot them verbatim into `_refs/` with a `session:`
  origin marker.

### What is NOT ground truth

- Your own reasoning, synthesis, or explanation — that is the draft, not the
  source.
- Other wiki pages — those are derived too.
- Search-result snippets or chat messages about a topic — fetch the primary
  source instead.
- Ephemeral pages (login walls, auto-generated dashboards, search results).

### When in doubt

Ask: *"If a future session had to verify every claim in my draft, what would it
need to read?"* Capture exactly that. If you cannot name a single ground-truth
source for a draft, the draft is too speculative — file it with an explicit
`ground_truth: []` and a `status: unverified` marker, or skip it.

## CRITICAL: vault access method

**You MUST read `~/.claude/ziki.md` FIRST, before doing anything else.** This file
contains the vault repository coordinates (YAML frontmatter) and the exact tools or
commands to use for reading and writing vault files (markdown body).

**Do NOT write to the local filesystem.** The vault is a remote git repository. Even
if you see a local checkout of the vault (e.g. `~/Documents/ziki/`), do not use it.
All reads and writes go through the remote access method described in `~/.claude/ziki.md`.

If `~/.claude/ziki.md` does not exist, stop and tell the user to run `/ziki-setup`.

## Input sources

Handle any of the following (infer from context, no user input required). Each
input implies a different ground-truth capture duty — see the right column.

| Input | Synthesis | Ground truth to capture alongside |
| --- | --- | --- |
| **URL** | Extract clean text as the draft body, or summarize it | Cache the page verbatim to `_refs/` and record the URL |
| **File path** | Summarize or extract key ideas | Snapshot the file contents to `_refs/` with a session-origin marker |
| **Topic / conversation excerpt** | Synthesize a draft from session context | List every primary source the synthesis leans on and cache each one (see below) |
| **Codebase exploration** (you read files from a repo before filing) | Synthesis of architecture, design, behaviour | Repo URL pinned to commit SHA + `_refs/` snapshots of the specific files you relied on |
| **Combination** | Draft that blends a URL with surrounding analysis | All sources above, combined |

For the **Topic / conversation excerpt** and **Codebase exploration** cases —
the ones where you are writing from memory or derivation rather than a single
obvious source — you are personally responsible for reconstructing the ground
truth from the session. Before writing the draft, list out:

1. Every file you read (with repo + commit SHA).
2. Every URL you fetched.
3. Every external doc or spec you relied on.
4. Any pasted content from the user.

Then cache each of those to `_refs/` and record them in the inbox file's
`ground_truth:` field. Only after that should you write the synthesis.

## How to file

1. **Read `~/.claude/ziki.md`**: parse frontmatter for vault coordinates, read body
   for access instructions.
2. **Inventory ground truth first**: before writing any synthesis, enumerate every
   primary source the draft will rely on (files read from a repo with their commit
   SHA, URLs fetched, docs consulted, pasted content). If the list is empty, the
   draft is unverifiable — either find sources or skip.
3. **Cache each ground-truth source to `_refs/`** (see "Reference snapshots" below).
   Do this BEFORE writing the draft, so the draft can reference the cached paths.
4. **Determine draft content**: fetch URL if needed, read file if needed, or
   synthesize from conversation now that the ground truth is locked in.
5. **Deduplicate**: read `wiki/_index.md` and list `_inbox/` directory from the vault
   (using the remote access method). If a near-duplicate exists, skip silently.
   Also list `_refs/` and reuse any existing snapshot instead of creating a duplicate.
6. **Choose filename**: `<kebab-case-title>.md` (short, descriptive, no dates).
7. **Prepare the inbox file** with this format:

```markdown
---
title: "<Title>"
type: concept | entity | comparison | synthesis | reference
tags: [tag1, tag2]
sources:
  - <where the synthesis came from: session context, conversation excerpt, a user message, etc.>
ground_truth:
  - _refs/<snapshot>.md
  - https://github.com/owner/repo/tree/<commit-sha>
  - https://docs.example.com/spec  # stable canonical URL
created: YYYY-MM-DD
status: inbox
---

# Title

Content here. Claims that lean on a specific ground-truth source should say so
inline (e.g. "per `_refs/sing-box--vless-inbound.md`" or "per yosh `main.go` at
`abc1234`"), so the `/ziki-inbox` re-verification step can trace each claim to a
source without guessing.
```

The `ground_truth:` list is **required** and must be non-empty for any draft
that makes factual claims. An empty list is allowed only for pure opinion,
preference, or personal-journal entries — and those should also carry
`status: unverified`.

8. **Write files** to the vault in a single commit:
   - `_inbox/<filename>.md`
   - One `_refs/<snapshot>.md` per ground-truth source you cached
   - Commit message: `ziki: add to inbox — <title>`

   Prefer a single commit that contains the draft and all of its ground-truth
   snapshots together. A draft that lands without its refs breaks the
   verification chain.

## Reference snapshots (`_refs/`)

`_refs/` is the ground-truth cache. Every inbox draft that makes factual claims
should land with at least one `_refs/` snapshot in the same commit. This is not
optional sugar — it is the mechanism that makes the re-verification step in
`/ziki-inbox` possible at all.

**When to cache**: any primary source the draft depends on — web pages, docs,
source files from a repo, specifications, transcripts, pasted user content.
Do NOT cache trivial pages, search-result SERPs, or auto-generated dashboards.

**When NOT to cache (reference the URL only instead)**: content already available
at a stable canonical URL that will not rot (RFCs on `rfc-editor.org`, W3C specs,
pinned git permalinks). In these cases record the stable URL directly in
`ground_truth:` and skip the `_refs/` snapshot.

**Naming convention**:

- Web content: `<domain>--<kebab-title>.md`
  (e.g. `docs-nats-io--jetstream-overview.md`)
- Repo file snapshots: `<owner>-<repo>--<path-kebab>@<short-sha>.md`
  (e.g. `mikluko-yosh--cmd-bot-main-go@abc1234.md`)
- User-pasted content: `session--<kebab-title>.md`
  (e.g. `session--yosh-design-notes.md`)

**Frontmatter format** (adapt to origin):

```markdown
---
# For web content:
url: "https://docs.nats.io/nats-concepts/jetstream"
fetched: YYYY-MM-DD
content_type: documentation | article | specification | transcript

# For repo file snapshots (in addition to or instead of url):
repo: "mikluko/yosh"
path: "cmd/bot/main.go"
commit: "abc1234"
fetched: YYYY-MM-DD
content_type: source-code

# For session-origin content:
origin: session
captured: YYYY-MM-DD
content_type: design-note | pasted-text | transcript
---

# Page Title or File Path

Extracted content here (clean markdown / verbatim source, no HTML noise).
```

**Rules for `_refs/`**:
- Clean, extracted markdown (or verbatim source code) — never raw HTML
- Include substantive content only; strip navigation, ads, chrome
- One file per source (URL, repo-file-at-commit, or pasted artifact)
- Deduplicate: list `_refs/` and reuse an existing snapshot before creating a new one
- For repo files, always pin to a commit SHA — never reference `main`/`HEAD`
- When the same topic spans multiple files of a repo, it is fine to create
  multiple snapshots; keep each one focused on a single file or section

## Tags

Read `_meta/taxonomy.md` from the vault to select tags. Extend the vocabulary there if
a new tag is clearly needed.

## Rules

- **NEVER** prompt the user for confirmation or input
- **NEVER** announce what you are doing unless the user explicitly invoked `/ziki-add`
- **NEVER** read or write the local filesystem for vault operations
- **NEVER** use `~/Documents/ziki/` or any other local path
- **NEVER** invent sources or citations
- **NEVER** file a synthesis draft without its ground-truth snapshots in the same
  commit — the draft is not useful without the sources it was derived from
- **ALWAYS** inventory ground truth BEFORE writing the synthesis, not after
- **ALWAYS** pin repo references to a commit SHA, never `main`/`HEAD`
- **ALWAYS** populate the `ground_truth:` field; the only acceptable empty list
  is on opinion/preference entries, which must also carry `status: unverified`
- If content is too thin (stub, one-liner with no substance), skip silently
- If you cannot identify any ground-truth source for a factual draft, skip it —
  unverifiable synthesis rots the wiki
- One `_inbox/` file per distinct concept; do not bundle unrelated things
- Deduplicate: check `wiki/_index.md` and existing `_inbox/` files before creating
  a near-duplicate; check `_refs/` before creating a near-duplicate snapshot
