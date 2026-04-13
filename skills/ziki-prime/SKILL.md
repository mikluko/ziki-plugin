---
name: ziki-prime
description: This skill should be used at the start of a session, or whenever the wiki may have changed mid-session, to load the Ziki wiki index into the current context so Claude knows what knowledge is available and can call the ziki-recall skill to fetch specific pages. Also used internally by the SessionStart hook. Invoke this skill whenever the user runs /ziki-prime, asks to "reload the wiki", "refresh ziki", or when you suspect the primed index is stale.
version: 1.0.0
---

# Ziki Prime

Load the Ziki wiki index into the current session so Claude is aware of what
knowledge has been accumulated and can pull specific pages on demand via the
`ziki-recall` skill.

This skill is the discovery-surface loader for Ziki. It's used both by the
`SessionStart` hook (which wraps the output as `additionalContext`) and
explicitly by the user via `/ziki-prime` to re-prime a long-running session
after the wiki has been updated.

## CRITICAL: vault access method

**You MUST read `~/.claude/ziki.md` FIRST, before doing anything else.** This
file contains the vault repository coordinates (YAML frontmatter) and the
exact tools or commands to use for reading vault files (markdown body).

**Do NOT read from the local filesystem.** The vault is a remote git
repository. Even if you see a local checkout of the vault (e.g.
`~/Documents/ziki/`), do not use it. All reads go through the remote access
method described in `~/.claude/ziki.md`.

If `~/.claude/ziki.md` does not exist, return an empty string (when called
from a hook) or a one-line notice "Ziki not configured — run /ziki-setup"
(when called interactively). Do NOT raise an error; Ziki is simply inactive
for this user.

## What to load

1. **`wiki/_index.md`** (required) — the curated map of wiki pages
2. **`_meta/taxonomy.md`** (optional) — the tag vocabulary; include only the
   tag list, not the descriptions

That's it. Do not fetch individual wiki page bodies — those are pulled on
demand by `ziki-recall`. The index is deliberately lightweight so that
priming is cheap and every session can afford it.

## How to prime

1. **Read `~/.claude/ziki.md`**: parse frontmatter for `vault_owner`,
   `vault_repo`, `vault_branch`. Read the body for access instructions.

2. **Fetch `wiki/_index.md`** from the remote vault using the configured
   access method.

3. **Fetch `_meta/taxonomy.md`** from the remote vault (best-effort — if it
   doesn't exist or fails, skip it silently).

4. **Assemble the primer block** (see format below).

5. **Enforce the size budget** (see below).

6. **Return the primer block** as plain markdown. The caller (hook or
   command) is responsible for any additional wrapping.

## Primer block format

Return a single markdown block in this exact shape:

```
# Ziki wiki primer

A Ziki wiki is available for this session.

- Vault: `<owner>/<repo>` on branch `<branch>`
- Primed: <YYYY-MM-DD HH:MM UTC>

Use the `ziki-recall` skill to fetch a wiki page whenever the current task
touches a topic listed in the index below. Prefer recalled wiki content over
answering from memory when a matching page exists. The `ziki-recall` skill
accepts a page path, title, or topic, and will follow `[[wikilinks]]` one
hop to pull in neighbour pages.

If the wiki has been updated during this session, run `/ziki-prime` to
refresh this primer.

## wiki/_index.md

<contents of wiki/_index.md, verbatim, inside a fenced ```markdown block>

## _meta/taxonomy.md (tags)

<flattened tag list, one per line, if taxonomy was fetched>
```

## Size budget

Keep the total primer block under **8KB**. Apply the following truncation
rules in order:

1. If `wiki/_index.md` is **≤ 6KB**, include it verbatim.
2. If `wiki/_index.md` is **> 6KB**, include only:
   - Top-level headings (`#`, `##`)
   - The first sentence of each bulleted entry (truncate at `. ` or 120 chars)
   - A footer line: `<full index available via: ziki-recall _index>`
3. If the taxonomy file would push the total over 8KB, drop it.
4. If even the truncated index exceeds 8KB, keep only the top-level headings
   and note that `wiki/_index.md` must be split into sub-indices (this is a
   signal to the operator that `/ziki-inbox` needs to maintain the index
   better).

## Failure modes

- **`~/.claude/ziki.md` missing**: return an empty string (hook context) or
  "Ziki not configured — run `/ziki-setup`" (interactive context). Do not
  raise.
- **Vault unreachable** (auth, network, 404): return a one-line primer that
  says `Ziki vault configured but unreachable — recall unavailable this
  session.` with the vault coordinates for debugging. Do not raise.
- **`wiki/_index.md` does not exist** (fresh vault with no pages yet):
  return a short primer that explains the vault is configured but empty,
  and that `ziki-recall` will have nothing to retrieve until content is
  added via `/ziki-add` and processed by `/ziki-inbox`.

## Rules

- **NEVER** prompt the user for confirmation or input
- **NEVER** read the local filesystem for vault operations
- **NEVER** use `~/Documents/ziki/` or any other local path
- **NEVER** fetch individual wiki page bodies — that is `ziki-recall`'s job
- **ALWAYS** return a string, never raise — Ziki is best-effort infrastructure
  and must not block session startup
- **ALWAYS** respect the 8KB budget — the primer competes with real work for
  context space
