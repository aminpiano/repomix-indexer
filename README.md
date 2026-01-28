# Repomix Indexer

Claude Code skill for creating searchable indexes from repomix XML output.

## Installation

```bash
# Clone to Claude skills directory
git clone https://github.com/aminpiano/repomix-indexer.git ~/.claude/skills/repomix-indexer
```

## Requirements

- **Node.js** (v18+) with npm
- **Claude Code CLI**

**Note:** repomix is NOT required to be pre-installed. The skill uses `npx repomix@latest` which automatically downloads and runs the latest version.

## Usage

In Claude Code, say:
- "repomix index"
- "index this repo"
- "인덱스 만들어줘" (Korean)

## What It Does

1. Asks which folders/files to exclude
2. Creates `.repomixignore` file
3. Runs repomix to pack codebase into XML
4. Extracts file boundaries using Claude's Grep tool
5. Spawns parallel Task agents to analyze files
6. Merges results into searchable index with validation

## Features

- **Cross-platform**: Works on Windows, Mac, Linux
- **No Unix dependencies**: Uses Node.js instead of grep/sed/wc
- **Parallel analysis**: Multiple Task agents for large codebases
- **Validation**: Catches JSON errors, missing files, format issues
- **Context-efficient**: Doesn't load chunk files into conversation

## Output Files

| File | Description |
|------|-------------|
| `repomix-output.xml` | Packed codebase with line numbers |
| `index-final.json` | Searchable index |

### index-final.json Structure

```json
{
  "generated": "2026-01-28T12:00:00Z",
  "total_files": 71,
  "total_lines": 5912,
  "analyzed_files": 71,
  "files": {
    "app/page.tsx": {
      "lines": [307, 361],
      "exports": ["Home:7"],
      "imports": ["react", "next/navigation"],
      "summary": "Main landing page with 3D background"
    }
  }
}
```

## Using the Index

```bash
# Find files by keyword
grep "button" index-final.json

# Search by export (requires jq)
jq '.files | to_entries[] | select(.value.exports[] | contains("useAuth"))' index-final.json
```

## License

MIT
