---
description: Configure the Ziki vault repository and access method for this project
---

# Ziki Setup

Configure which GitHub repository to use as the Ziki vault and how to access it.

## Steps

### 1. Vault repository

Ask the user for their vault repository. They may provide:
- A GitHub `owner/repo` (e.g. `mikluko/ziki`)
- A full URL (e.g. `https://github.com/mikluko/ziki`)
- Or say they don't have one yet

If they don't have a vault repo, offer to create one on GitHub (if you have tools
to do so). See "Bootstrapping a new vault" below.

Ask for the branch (default: `main`).

### 2. Access method

Ask the user how you should access the repository. Common options:

- **GitHub MCP tools**: `mcp__github__get_file_contents`, `mcp__github__push_files`, etc.
- **`gh` CLI**: `gh api repos/owner/repo/contents/...`
- **`git` CLI**: clone/pull, edit locally, commit and push
- **Something else**: the user may have a custom setup

If the user isn't sure, try to auto-detect:
1. Try calling `mcp__github__get_file_contents` for `AGENTS.md` in the vault repo
2. If that fails, try `gh api repos/<owner>/<repo>/contents/AGENTS.md`
3. If that fails, try `git ls-remote` against the repo URL
4. Report what worked

### 3. Test access

Once an access method is chosen, verify it works end-to-end:
- **Read test**: fetch `AGENTS.md` (or any file) from the vault
- **Write test**: skip (don't modify the vault during setup)

If the read test fails, report the error and ask follow-up questions:
- Authentication issues? Ask the user to configure credentials.
- Permission issues? Check repo visibility.
- Wrong repo? Confirm the name.

Do not proceed until read access is confirmed.

### 4. Write settings

Write `~/.claude/ziki.md` (user-level settings) with:
- YAML frontmatter: `vault_owner`, `vault_repo`, `vault_branch`
- Markdown body: access instructions that other Ziki commands will follow

The access instructions should be specific and tested. Example for `gh` CLI:

```markdown
---
vault_owner: mikluko
vault_repo: ziki
vault_branch: main
---

# Vault Access Instructions

Use `gh` CLI for all vault operations against `mikluko/ziki` on branch `main`.

## Reading files

gh api repos/mikluko/ziki/contents/{path}?ref=main

The response includes `content` (base64-encoded) and `sha` (needed for updates).

## Writing a single file

gh api -X PUT repos/mikluko/ziki/contents/{path} \
  -f message="commit message" \
  -f content="$(echo -n 'file content' | base64)" \
  -f branch=main \
  -f sha={current_sha}  # only for updates, omit for new files

## Writing multiple files in one commit

Use the Git Data API via `gh api`:
1. Get current commit SHA: `gh api repos/mikluko/ziki/git/ref/heads/main`
2. Get the tree SHA from that commit
3. Create blobs for each file
4. Create a new tree with the blobs
5. Create a commit pointing to the new tree
6. Update the ref to point to the new commit
```

Example for GitHub MCP tools:

```markdown
---
vault_owner: mikluko
vault_repo: ziki
vault_branch: main
---

# Vault Access Instructions

Use GitHub MCP tools for all vault operations against `mikluko/ziki` on branch `main`.

## Reading files

Use `mcp__github__get_file_contents` with:
- owner: "mikluko"
- repo: "ziki"
- path: the file path
- ref: "refs/heads/main"

## Writing a single file

Use `mcp__github__create_or_update_file` with owner, repo, path, content, message,
branch, and sha (for updates).

## Writing multiple files in one commit

Use `mcp__github__push_files` with owner, repo, branch, files array, and message.
```

### 5. Confirm

Tell the user:
- Which vault repo is configured
- Which access method is being used
- That `/ziki-add` and `/ziki-inbox` will now work in this project
- That hooks (Stop, PreCompact) will auto-capture knowledge from sessions

## Bootstrapping a new vault

If the user doesn't have a vault repo yet, or the repo exists but is empty:

### 1. Create the repo (if needed)

Offer to create a private GitHub repository. Use whatever access method is available.

### 2. Scaffold the vault structure

Push the following files in one commit:

- **`AGENTS.md`**: the agent contract. Use the canonical version from
  `github.com/mikluko/ziki` as a template (read it during setup). Adapt the title
  and any repo-specific references.
- **`wiki/_index.md`**: empty index with a header and placeholder sections
- **`wiki/_log.md`**: empty log with a header
- **`.manifest.json`**: empty object `{}`
- **`_meta/taxonomy.md`**: starter taxonomy with a few universal tags
- **`_inbox/.gitkeep`**: ensure the directory exists

Commit message: `wiki: initialize vault`

### 3. Suggest a seed topic

An empty wiki is daunting. After scaffolding, suggest a first topic to get the wiki
started. The goal is to create one real wiki page immediately, so the user sees the
system working end-to-end.

**How to pick the topic:**

1. Look at what you already know about the user from this session and any accumulated
   context: their role, domain, current project, technologies they use, problems
   they're solving, interests they've mentioned.

2. If there's one obvious topic (e.g. the user has been deep in a specific technology
   or domain all session), suggest it directly:
   > "Based on our conversation, I'd suggest starting with a page on [topic].
   > Want me to create it?"

3. If there's no single obvious winner, offer 3-5 concrete topics drawn from what
   you know, and let the user pick one or more:
   > "Here are some topics that could seed your wiki based on what I know about
   > your work:
   > 1. [Technology X] — you use this daily
   > 2. [Concept Y] — came up in our discussion
   > 3. [Tool Z] — central to your workflow
   > 4. [Domain topic] — your area of expertise
   >
   > Pick any that interest you, or suggest your own."

4. If you know nothing about the user yet, suggest they pick a topic they're currently
   learning or researching, since that's where the wiki pattern adds the most value.

**When the user picks a topic:**

- Use web search to gather current, accurate information
- Create both an `_inbox/` draft and a promoted `wiki/` page (so the user sees the
  full pipeline result immediately)
- Update `wiki/_index.md`, `wiki/_log.md`, and `.manifest.json`
- Commit and push: `wiki: seed — <topic>`

This gives the user a working wiki with real content from the very first setup.

## Notes

- If `~/.claude/ziki.md` already exists, show current settings and ask whether
  to update them.
- The settings file is user-level (`~/.claude/`), shared across all projects.
- Tailor the access instructions to exactly what was tested and confirmed working.
  Do not include methods that weren't verified.
