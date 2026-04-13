---
description: Reload the Ziki wiki index into the current session
---

Use the `ziki-prime` skill to load the Ziki wiki index into the current
session. This is useful when the wiki has been updated mid-session (for
example, after the hourly `/ziki-inbox` trigger has promoted new pages) and
you want Claude to re-discover what's available.

The skill fetches `wiki/_index.md` from the vault and returns a primer block.
Print the primer block in your response so it becomes part of the conversation
context; future turns can then use the `ziki-recall` skill to pull specific
pages.

Do not ask the user to confirm. If `~/.claude/ziki.md` is missing, tell them
to run `/ziki-setup` and stop.
