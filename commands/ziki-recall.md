---
description: Pull pages from the Ziki wiki into the current session for grounding
---

Use the `ziki-recall` skill to fetch pages from the Ziki wiki so the current
session can ground its answers in accumulated knowledge.

Infer the target from context:
- If the user named a topic or page title, pass it to the skill
- If the user said "check the wiki" or "recall that", infer the topic from the
  most recent conversation turn
- If the session was primed with a wiki index, match the topic against it
- If nothing specific was asked, pass `_index` to refresh the full index

Do not ask the user to pick a page. Do not confirm before fetching. Return the
recalled content and use it to answer the user's question, citing the wiki
page paths you pulled from.
