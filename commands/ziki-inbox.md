---
description: Process pending files in the Ziki knowledge base inbox, enrich with research, and promote quality content into the wiki
---

Use the `ziki-inbox` skill to process all pending files in the Ziki vault
`_inbox/` directory. The skill handles everything: reading `~/.claude/ziki.md`
for vault access, orienting against `AGENTS.md` / `wiki/_index.md` /
`.manifest.json` / `_meta/taxonomy.md`, classifying and assessing each inbox
file, enriching with research, re-verifying against ground truth, promoting to
`wiki/` pages, updating `.manifest.json` / `wiki/_index.md` / `wiki/_log.md`,
and committing. Do NOT reimplement any of that logic here.

This is a fully autonomous operation. Do not ask the user to confirm. Do not
prompt for input. If `~/.claude/ziki.md` is missing, tell the user to run
`/ziki-setup` and stop.
