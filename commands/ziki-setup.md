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
to do so). The repo needs at minimum: `_inbox/`, `wiki/_index.md`, `wiki/_log.md`,
`.manifest.json`, and `AGENTS.md`.

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

Write `.claude/ziki.local.md` in the current project directory with:
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

## Notes

- If `.claude/ziki.local.md` already exists, show current settings and ask whether
  to update them.
- The settings file is local to this project and should not be committed to git.
- Tailor the access instructions to exactly what was tested and confirmed working.
  Do not include methods that weren't verified.
