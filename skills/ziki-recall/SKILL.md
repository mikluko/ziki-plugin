---
name: ziki-recall
description: This skill should be used to fetch pages from the Ziki wiki when the current task touches a topic that may already be documented there. Use it whenever a topic in the conversation matches (or plausibly matches) an entry in the wiki index that was primed at session start, or when the user asks to "recall from ziki", "check the wiki", "look it up in ziki", or similar. Prefer wiki pages over answering from memory when a match exists.
version: 1.0.0
---

# Ziki Recall

Fetch pages from the Ziki wiki so the current session can ground its answers
in accumulated knowledge instead of re-deriving things from scratch.

This skill is the read-path counterpart to `ziki-add`. The SessionStart hook
primes each session with `wiki/_index.md`; this skill is how the model actually
pulls a page body when it needs one.

## CRITICAL: vault access method

**You MUST read `~/.claude/ziki.md` FIRST, before doing anything else.** This
file contains the vault repository coordinates (YAML frontmatter) and the exact
tools or commands to use for reading vault files (markdown body).

**Do NOT read from the local filesystem.** The vault is a remote git
repository. Even if you see a local checkout of the vault (e.g.
`~/Documents/ziki/`), do not use it. All reads go through the remote access
method described in `~/.claude/ziki.md`.

If `~/.claude/ziki.md` does not exist, stop and tell the user to run
`/ziki-setup`.

## When to invoke

Invoke this skill autonomously whenever ANY of the following is true:

- The current task touches a topic listed (or plausibly listed) in the wiki
  index that was primed at session start
- The user asks a question whose answer would benefit from stable, verified
  knowledge rather than fresh derivation
- You are about to answer from memory on a topic that falls inside the user's
  domain as reflected in the wiki
- The user explicitly says "check ziki", "recall from the wiki", "look it up",
  or runs `/ziki-recall`

Do NOT invoke this skill for:

- Trivial factual questions unrelated to the user's wiki
- Code execution tasks that don't involve domain knowledge
- Questions the user has just answered themselves in this conversation

## Input

The caller provides one or more of:

- **Page path** (preferred): e.g. `wiki/jetstream-overview.md`
- **Page title**: e.g. `JetStream Overview`
- **Topic**: e.g. `nats durable consumers` — use this if the index has nothing
  that matches exactly; you will search the index for the best match
- **`_index`**: fetch the full `wiki/_index.md` (useful if the session was
  primed with a truncated index)

## How to recall

1. **Read `~/.claude/ziki.md`**: parse frontmatter for vault coordinates, read
   body for access instructions.

2. **Resolve the target**: if given a path, use it directly. If given a title
   or topic, scan the primed index (or re-fetch `wiki/_index.md` if needed) and
   pick the closest match. If there is no plausible match, return a short
   "no match in wiki" result so the caller can fall back to other strategies.

3. **Fetch the primary page** from the vault using the configured access
   method.

4. **Follow wikilinks** up to a small budget to pull in neighbours:
   - Extract `[[wikilinks]]` from the primary page
   - Fetch up to **2 additional** directly-linked pages that appear relevant
     to the caller's topic
   - Do not recurse past depth 1 — one hop only
   - Skip links to pages already fetched in this recall round

5. **Fetch ground-truth snapshots** if needed: if the primary page's
   frontmatter lists `_refs/...` paths in `ground_truth:` and the caller needs
   to cite specific facts, fetch those snapshots too. Prefer `_refs/` over
   refetching the original URL.

6. **Return results** as a compact block the model can quote from:

   ```
   ## Recalled from Ziki wiki

   ### wiki/<primary>.md
   <primary page body, verbatim>

   ### wiki/<neighbour-1>.md (linked from primary)
   <neighbour page body, verbatim>

   ### _refs/<snapshot>.md (ground truth for primary)
   <snapshot excerpt, if needed>
   ```

7. **Ground the answer**: when using recalled content in the response, cite
   the wiki page path (e.g. `per wiki/jetstream-overview.md`). Respect
   provenance markers on the page — if a claim is marked `^[inferred]`,
   `^[ambiguous]`, or `^[stale]`, carry that uncertainty into the answer.

## Budget

- At most **3 pages** fetched per recall round (1 primary + 2 linked)
- At most **1 `_refs/` snapshot** per recall round
- Total returned content under **32KB** — truncate body sections if larger and
  note the truncation

If the caller needs more, they can invoke the skill again with a more specific
target.

## Rules

- **NEVER** prompt the user for confirmation or input
- **NEVER** read the local filesystem for vault operations
- **NEVER** use `~/Documents/ziki/` or any other local path
- **NEVER** invent wiki pages or citations — if a page does not exist, say so
- **PREFER** recalled wiki content over answering from memory when both are
  available and cover the same ground
- If the vault is unreachable, return a one-line notice and let the caller
  decide whether to proceed without recall
- Quote page content verbatim in the return block; do not paraphrase
  ground-truth-anchored claims
