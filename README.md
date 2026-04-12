# Ziki Plugin for Claude Code

A Claude Code plugin that manages an AI-owned personal knowledge base inspired by
Zettelkasten and Andrej Karpathy's [LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) pattern.

Instead of re-deriving answers from raw documents on every query, an LLM incrementally
builds and maintains a persistent wiki: a structured, interlinked collection of markdown
files that compounds over time. The LLM owns the wiki; the human curates sources and
asks questions.

## Commands

### `/ziki-setup`

Configure the vault repository for this project. Asks for the GitHub `owner/repo` and
branch, verifies access, and writes settings to `.claude/ziki.md`.

### `/ziki-add`

File content from the current conversation into the vault inbox. Handles URLs, file
paths, topics, and conversation excerpts. Runs silently by default.

### `/ziki-inbox`

Fully autonomous inbox processing. Reads pending files from `_inbox/`, classifies,
assesses quality, enriches with web research, and promotes to wiki pages. Commits and
pushes results.

## Hooks

- **Stop**: reviews the conversation before the agent exits and files any durable
  knowledge into the inbox
- **PreCompact**: same logic, triggered before context compaction to preserve knowledge
  that would otherwise be compressed away

Hooks are inactive until `/ziki-setup` has been run. If `~/.claude/ziki.md` does
not exist, hooks return immediately without doing anything.

## Scheduled Processing

During setup, the plugin offers to create an hourly remote agent that runs inbox
processing automatically. This closes the loop: hooks capture knowledge into `_inbox/`
during sessions, and the scheduled agent promotes it to wiki pages in the background.
Knowledge captured in one session becomes available as compiled wiki pages within the
hour.

## Installation

```bash
claude plugin marketplace add mikluko/ziki-plugin
claude plugin install ziki@ziki-plugin
```

For development/testing (session-scoped):

```bash
claude --plugin-dir /path/to/ziki-plugin
```

## Configuration

After installing, run `/ziki-setup` in any project where you want Ziki active. Setup
will:

1. Ask for your vault repository (`owner/repo` and branch)
2. Discover how to access it (GitHub MCP tools, `gh` CLI, `git` CLI, or custom)
3. Test that read access works
4. Write `.claude/ziki.md` with vault config and access instructions

The settings file stores both the repo coordinates (in YAML frontmatter) and the
tested access instructions (in the markdown body) that all other commands follow.

```markdown
---
vault_owner: mikluko
vault_repo: ziki
vault_branch: main
---

# Vault Access Instructions

(access method details written by /ziki-setup)
```

The settings file is user-level, shared across all projects.

### Prerequisites

You need some way for Claude to access a GitHub repository. Any of these work:
- GitHub MCP server
- `gh` CLI (authenticated)
- `git` CLI with SSH or HTTPS credentials
- Any other method you can describe to `/ziki-setup`

## Architecture

All vault access goes through the remote git repository, never the local filesystem.
The specific access method is configured per-project during `/ziki-setup` and stored
in `~/.claude/ziki.md`. This makes the plugin work regardless of which tools are
available in the user's environment.

### Vault structure

```
_inbox/        # Drafts and raw sources (agent writes here)
wiki/          # LLM-compiled wiki pages (agent owns this layer)
wiki/_index.md # Human-facing entry point
wiki/_log.md   # Append-only operation log
_meta/         # Taxonomy and metadata
.manifest.json # Tracks ingested inbox files
AGENTS.md      # Agent contract and hard rules
```

## License

MIT
