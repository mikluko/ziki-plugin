# Ziki Plugin for Claude Code

A Claude Code plugin that manages an AI-owned personal knowledge base inspired by
Zettelkasten and Andrej Karpathy's [LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) pattern.

Instead of re-deriving answers from raw documents on every query, an LLM incrementally
builds and maintains a persistent wiki: a structured, interlinked collection of markdown
files that compounds over time. The LLM owns the wiki; the human curates sources and
asks questions.

## Commands

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

## Installation

```bash
claude plugin install ziki@mikluko/ziki-plugin
```

For development/testing (session-scoped):

```bash
claude --plugin-dir /path/to/ziki-plugin/plugins/ziki
```

## Configuration

The vault repository is currently hardcoded to `github.com/mikluko/ziki`. Future
versions will make this configurable.

### Prerequisites

- GitHub MCP server configured (preferred access method)
- Alternatively, `gh` CLI authenticated with access to the vault repository

## Architecture

All vault access goes through the remote git repository via GitHub MCP tools
(`mcp__github__get_file_contents`, `mcp__github__push_files`, etc.). The plugin never
touches the local filesystem for vault operations. This makes it work from Claude
Desktop, Claude Code CLI, and Claude Mobile.

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
