---
description: Configure the Ziki vault repository for this project
---

# Ziki Setup

Configure which GitHub repository to use as the Ziki vault.

## Steps

1. **Ask the user** for their vault repository in `owner/repo` format.
   Default suggestion: `mikluko/ziki`. Also ask for the branch (default: `main`).

2. **Verify access**: use `mcp__github__get_file_contents` to read `AGENTS.md` from
   the specified repo and branch. If it fails, warn the user and ask whether to
   proceed anyway.

3. **Write settings** to `.claude/ziki.local.md` in the current project directory:

```markdown
---
vault_owner: <owner>
vault_repo: <repo>
vault_branch: <branch>
---
```

4. **Confirm** to the user that the vault is configured. Remind them that other
   Ziki commands (`/ziki-add`, `/ziki-inbox`) will now use this vault.

## Notes

- The settings file is local to this project and should not be committed to git.
- If `.claude/ziki.local.md` already exists, show the current settings and ask
  whether to update them.
- If the vault repo has `_meta/taxonomy.md`, mention that tags will be sourced
  from there.
