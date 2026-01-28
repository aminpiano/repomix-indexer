# Repomix Indexer

Claude Code skill for creating searchable indexes from repomix XML output.

## Installation

```bash
# Clone to Claude skills directory
git clone https://github.com/YOUR_USERNAME/repomix-indexer.git ~/.claude/skills/repomix-indexer
```

## Usage

In Claude Code, say:
- "repomix index"
- "index this repo"

## Features

- Cross-platform (Windows/Mac/Linux)
- Uses Node.js instead of Unix commands
- Parallel analysis with Task agents
- Validation and error reporting
- Automatic cleanup

## Output

- `repomix-output.xml` - Packed codebase
- `index-final.json` - Searchable index with exports, imports, summaries

## Requirements

- Node.js (for repomix and scripts)
- Claude Code CLI
