---
name: repomix-indexer
description: "Create searchable index from repomix output. Triggers: 'repomix index', 'index this repo'. For large codebases that can't be read at once - extracts file boundaries and uses parallel Task agents to analyze sections."
---

# Repomix Indexer

Creates a searchable index from repomix XML output, enabling efficient navigation of large codebases.

## When to Use

- Codebase > 10k lines packed with repomix
- Need to find specific files/functions without reading entire output
- Want structured overview of repository contents

## Important: Platform Compatibility

This skill uses **Node.js** and **Claude's built-in tools** (Grep, Read) instead of Unix commands for cross-platform compatibility (Windows/Mac/Linux).

## Workflow

### Step 0: Ask User for Exclusions

Before starting, ask the user what to exclude:
- Common exclusions: `.agent/**`, `node_modules/**`, `.next/**`, `.git/**`, `plan/**`, `dist/**`, `build/**`
- Ask if there are project-specific folders to exclude

### Step 1: Create .repomixignore File

**IMPORTANT**: The `--ignore` CLI flag is unreliable. Always create a `.repomixignore` file instead.

```javascript
// Use Write tool to create .repomixignore in project root
.agent/**
node_modules/**
.next/**
.git/**
.vercel/**
dist/**
build/**
*.log
*.png
*.jpg
repomix-output.xml
index-*.json
.repomixignore
```

### Step 2: Generate Repomix Output

**CRITICAL**: Use `node -e` wrapper because `npx` output capture fails in many environments.

```javascript
node -e "
const {execSync} = require('child_process');
const result = execSync('npx repomix . --output-show-line-numbers -o repomix-output.xml', {
  cwd: process.cwd(),
  encoding: 'utf8',
  stdio: ['pipe', 'pipe', 'pipe']
});
console.log(result);
"
```

If the command shows no output, check if `repomix-output.xml` was created with `ls` or Glob tool.

### Step 3: Extract File Boundaries

**Use Claude's Grep tool** (NOT bash grep):

```
Grep tool:
  pattern: <file path=
  path: repomix-output.xml
  output_mode: content
  -n: true
```

This returns lines like:
```
125:<file path="app/page.tsx">
179:<file path="app/globals.css">
```

**Get total line count** with Bash:
```bash
wc -l repomix-output.xml
```

If `wc` fails (Windows), use Node.js:
```javascript
node -e "const fs=require('fs'); console.log(fs.readFileSync('repomix-output.xml','utf8').split('\\n').length)"
```

### Step 4: Create Index Draft

Parse the Grep output directly with Node.js inline script:

```javascript
node -e "
const fs = require('fs');

// Paste the grep output here as a template literal
const grepOutput = \`
125:<file path=\"app/page.tsx\">
179:<file path=\"app/globals.css\">
...(paste all lines)...
\`;

const totalLines = 5912; // from wc -l result

const lines = grepOutput.trim().split('\\n').filter(l => l.trim());
const files = [];

for (let i = 0; i < lines.length; i++) {
  const match = lines[i].match(/^(\\d+):<file path=\"([^\"]+)\">/);
  if (!match) continue;

  const start = parseInt(match[1]);
  const path = match[2];
  const end = i < lines.length - 1
    ? parseInt(lines[i+1].match(/^(\\d+)/)[1]) - 1
    : totalLines;

  files.push({ path, start, end, lines: end - start + 1 });
}

const index = { total_lines: totalLines, total_files: files.length, files };
fs.writeFileSync('index-draft.json', JSON.stringify(index, null, 2));
console.log('Created index-draft.json with ' + files.length + ' files');
"
```

### Step 5: Plan Task Distribution

Read `index-draft.json` and plan parallel analysis:

| Scale | Files | Lines | Suggested Tasks |
|-------|-------|-------|-----------------|
| Small | <30 | <3k | 1 Task or direct analysis |
| Medium | 30-100 | 3k-10k | 2-3 Tasks |
| Large | 100-300 | 10k-50k | 5-10 Tasks |
| Huge | 300+ | 50k+ | 10+ Tasks |

**Distribution tips:**
- Balance by LINE COUNT, not file count
- Group logically (all components together, all lib together)
- Each chunk should be 1500-2500 lines

### Step 6: Execute Parallel Analysis

Spawn Task agents with `run_in_background: true`:

**Task agent prompt template:**
```
Analyze these files from {PROJECT_PATH}/repomix-output.xml and extract:
- Exported functions/classes with line numbers
- Key imports
- Brief summary (1 line)

Files to analyze (lines X-Y):
1. path/to/file1.tsx: START-END
2. path/to/file2.tsx: START-END
...

Use Read tool with offset/limit to read each section.
Example: For file starting at line 125, ending at 178:
  Read(file_path: "repomix-output.xml", offset: 124, limit: 54)

Output JSON format for each file:
{
  "path": "...",
  "exports": ["funcName:lineNum", "className:lineNum"],
  "imports": ["react", "next/navigation"],
  "summary": "..."
}

After analysis, write results to {PROJECT_PATH}/chunk{N}.json as JSON array.
```

### Step 7: Merge Results

After all Task agents complete, merge with Node.js.

**IMPORTANT**: Do NOT use Read tool on chunk files. Let Node.js read them directly to save context. The script includes validation to catch errors.

```javascript
node -e "
const fs = require('fs');

const draft = JSON.parse(fs.readFileSync('index-draft.json', 'utf8'));

// Auto-detect chunk files
const chunkFiles = fs.readdirSync('.').filter(f => /^chunk\d+\.json$/.test(f)).sort();
if (chunkFiles.length === 0) {
  console.error('ERROR: No chunk files found!');
  process.exit(1);
}
console.log('Found chunks:', chunkFiles.join(', '));

// Parse chunks with error handling
const errors = [];
const chunks = [];
chunkFiles.forEach(f => {
  try {
    const data = JSON.parse(fs.readFileSync(f, 'utf8'));
    if (!Array.isArray(data)) {
      errors.push(f + ': Not an array');
    } else {
      data.forEach((item, i) => {
        if (!item.path) errors.push(f + '[' + i + ']: Missing path');
        if (!Array.isArray(item.exports)) errors.push(f + '[' + i + ']: exports not array');
        if (!Array.isArray(item.imports)) errors.push(f + '[' + i + ']: imports not array');
      });
      chunks.push(...data);
    }
  } catch (e) {
    errors.push(f + ': ' + e.message);
  }
});

if (errors.length > 0) {
  console.error('VALIDATION ERRORS:');
  errors.forEach(e => console.error('  - ' + e));
  process.exit(1);
}

// Check coverage
const chunkMap = {};
chunks.forEach(c => chunkMap[c.path] = c);
const analyzed = Object.keys(chunkMap).length;
const expected = draft.total_files;
const missing = draft.files.filter(f => !chunkMap[f.path]).map(f => f.path);

console.log('Analyzed: ' + analyzed + '/' + expected + ' files');
if (missing.length > 0 && missing.length <= 5) {
  console.warn('Missing: ' + missing.join(', '));
} else if (missing.length > 5) {
  console.warn('Missing: ' + missing.length + ' files (run with validation to see list)');
}

// Build final index
const files = {};
draft.files.forEach(f => {
  const analysis = chunkMap[f.path] || { exports: [], imports: [], summary: '' };
  files[f.path] = {
    lines: [f.start, f.end],
    exports: analysis.exports,
    imports: analysis.imports,
    summary: analysis.summary
  };
});

const final = {
  generated: new Date().toISOString(),
  source: 'repomix-output.xml',
  total_files: draft.total_files,
  total_lines: draft.total_lines,
  analyzed_files: analyzed,
  files
};

fs.writeFileSync('index-final.json', JSON.stringify(final, null, 2));
console.log('Created index-final.json');

// Cleanup
chunkFiles.concat(['index-draft.json', '.repomixignore']).forEach(f => {
  try { fs.unlinkSync(f); } catch(e) {}
});
console.log('Cleaned up temp files');
"
```

**Script validates:**
- Chunk files exist and are valid JSON
- Each item has required fields (path, exports[], imports[])
- Coverage report (how many files analyzed vs expected)
- Lists missing files if any

### Step 8: Cleanup (Optional)

Delete temporary files if not done in merge step:
- `chunk*.json`
- `index-draft.json`
- `.repomixignore`

Keep:
- `repomix-output.xml` (source)
- `index-final.json` (searchable index)

## Using the Index

### Find files by keyword
```bash
grep "button" index-final.json
```

Or use Claude's Grep tool on index-final.json.

### Read specific file from repomix

From index: `"lines": [5305, 5620]`
```
Read tool:
  file_path: repomix-output.xml
  offset: 5304  (start - 1)
  limit: 316    (end - start + 1)
```

### Search by export (requires jq)
```bash
jq '.files | to_entries[] | select(.value.exports[] | contains("useAuth"))' index-final.json
```

## Troubleshooting

### repomix doesn't exclude files
- **Solution**: Use `.repomixignore` file instead of `--ignore` flag

### npx command shows no output
- **Solution**: Wrap with `node -e` and use `execSync`

### wc command not found (Windows)
- **Solution**: Use Node.js to count lines:
  ```javascript
  node -e "console.log(require('fs').readFileSync('file.xml','utf8').split('\\n').length)"
  ```

### grep/sed not working (Windows)
- **Solution**: Use Claude's built-in Grep tool instead of bash grep

## Quick Reference

| Task | Method |
|------|--------|
| Pack repo | `node -e` wrapper with `npx repomix` |
| Extract boundaries | Claude's Grep tool with `-n: true` |
| Count lines | `wc -l` or Node.js fallback |
| Read section | Read tool with `offset` and `limit` |
| Create/merge JSON | Node.js inline scripts |

## Output Files

- `repomix-output.xml` - Packed codebase with line numbers
- `index-final.json` - Searchable index with:
  - File paths and line ranges
  - Exported functions/classes
  - Import dependencies
  - One-line summaries
