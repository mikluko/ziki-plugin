---
description: Add current context, a URL, or a topic to the Ziki knowledge base inbox
---

Use the `ziki-add` skill to file content from the current conversation into the Ziki
vault inbox.

Infer what to file from context:
- If a URL is visible in the conversation, fetch and file it
- If a file path was discussed, file its contents
- If a topic was explored in depth, synthesize and file it

Do not ask the user what to file. Do not confirm before filing. Do not announce
completion unless the user is watching.
