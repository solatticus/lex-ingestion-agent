# Lex Ingestion Agent — Implementation Plan

> Automated agent that keeps `.lex/` knowledge packs in sync with code changes.
> Triggers on push to `dev`, diffs changed files against `.lex/` docs, and opens a PR with updates.

---

## Overview

When code is pushed to `dev`, this agent:
1. Identifies what source files changed
2. Maps those files to relevant `.lex/` documents (using `FILE_MAP.md` as the routing table)
3. Reads the changed source + corresponding `.lex/` sections
4. Determines if `.lex/` content is stale (line refs shifted, functions renamed, params changed, logic altered)
5. Generates updated `.lex/` content
6. Opens a PR on the source repo for human review

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│  GitHub Action (or Gitea Action)                         │
│  Trigger: push to dev                                    │
│                                                          │
│  1. Checkout repo                                        │
│  2. Run: node agent/index.ts                             │
│     ├── git diff HEAD~1..HEAD --name-only                │
│     ├── Read FILE_MAP.md → build source↔lex mapping      │
│     ├── Filter: only .lex docs affected by changed files │
│     ├── For each affected .lex doc:                      │
│     │   ├── Read current .lex content                    │
│     │   ├── Read changed source files                    │
│     │   ├── Call Claude API with operator prompt          │
│     │   └── Collect updated .lex content                 │
│     ├── Write updated .lex files                         │
│     ├── git checkout -b lex/auto-update-{sha}            │
│     ├── git commit + push                                │
│     └── Open PR via gh/API                               │
└──────────────────────────────────────────────────────────┘
```

---

## Components

### 1. Source-to-Lex Mapper (`agent/mapper.ts`)

Parses `FILE_MAP.md` to build a bidirectional mapping:

```
source file path → [list of .lex documents that reference it]
```

**Logic:**
- Parse FILE_MAP.md tables — each row maps a source file to its purpose
- Parse each .lex doc for explicit file path references (e.g., `flute-runtime/src/server/input_handler.rs:214`)
- Given a list of changed files, return the set of .lex documents that need review

**Edge cases:**
- New source files with no .lex mapping → flag for manual review, don't auto-generate
- Deleted source files → flag references in .lex docs as potentially stale
- Renamed files → detect via git diff --diff-filter=R, update path references

### 2. Diff Analyzer (`agent/diff.ts`)

Extracts structured change information from git:

```typescript
interface FileChange {
  path: string;
  status: 'added' | 'modified' | 'deleted' | 'renamed';
  oldPath?: string;          // for renames
  hunks: string[];           // raw diff hunks
  changedFunctions: string[]; // function names touched (heuristic: lines starting with fn/pub fn)
  lineShift: number;         // net lines added - removed (for line ref updates)
}
```

**Key capability:** For Rust files, extract function signatures from diff context lines to know which functions were modified. This lets the agent scope updates to just the relevant .lex sections.

### 3. Operator Prompt (`agent/prompts.ts`)

The system prompt that scopes the Claude API call:

```
You are a codebase knowledge maintainer for the Flute project.

Given:
- A git diff showing what changed in source files
- The current content of a .lex knowledge document
- The FILE_MAP.md showing the codebase structure

Your job:
1. Identify which sections of the .lex document are affected by the code changes
2. Update ONLY those sections. Do not rewrite unaffected content.
3. Specifically update:
   - Line number references (e.g., input_handler.rs:214 → input_handler.rs:228 if 14 lines were added above)
   - Function signatures if they changed
   - Flow descriptions if control flow changed
   - New parameters, return types, or error conditions
   - New functions/types that should be documented
4. Do NOT:
   - Add speculative content about code you haven't seen
   - Remove sections unless the corresponding code was deleted
   - Change the document structure or formatting style
   - Add commentary about the changes themselves

Output the complete updated .lex document content.
If no updates are needed, respond with exactly: NO_CHANGES_NEEDED
```

### 4. Agent Core (`agent/index.ts`)

Orchestration logic:

```typescript
async function main() {
  // 1. Get changed files
  const changes = await getChangedFiles();  // git diff

  // 2. Load source→lex mapping
  const mapper = await loadFileMap('.lex/FILE_MAP.md');

  // 3. Find affected .lex docs
  const affectedDocs = mapper.getAffectedDocs(changes.map(c => c.path));

  if (affectedDocs.length === 0) {
    console.log('No .lex documents affected by this push.');
    return;
  }

  // 4. For each affected doc, call Claude API
  const updates: Map<string, string> = new Map();

  for (const lexDoc of affectedDocs) {
    const currentContent = await readFile(lexDoc);
    const relevantChanges = changes.filter(c => mapper.isRelevant(c.path, lexDoc));
    const relevantDiffs = await getDiffs(relevantChanges);

    const result = await callClaude({
      system: OPERATOR_PROMPT,
      messages: [{
        role: 'user',
        content: buildUpdateRequest(currentContent, relevantDiffs, relevantChanges)
      }]
    });

    if (result !== 'NO_CHANGES_NEEDED') {
      updates.set(lexDoc, result);
    }
  }

  if (updates.size === 0) {
    console.log('All .lex documents are up to date.');
    return;
  }

  // 5. Write updates, branch, commit, PR
  await createUpdatePR(updates);
}
```

### 5. GitHub Action (`.github/workflows/lex-sync.yml`)

```yaml
name: Lex Knowledge Sync

on:
  push:
    branches: [dev]
    paths-ignore:
      - '.lex/**'  # Don't trigger on .lex changes (prevents loops)

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2  # Need HEAD~1 for diff

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install agent deps
        working-directory: .lex-agent  # or wherever the agent lives in the target repo
        run: npm ci

      - name: Run Lex Ingestion Agent
        working-directory: .lex-agent
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: npx tsx agent/index.ts

      - name: Create PR if changes
        if: steps.run.outputs.has_changes == 'true'
        run: |
          gh pr create \
            --title "lex: auto-sync knowledge docs" \
            --body "Automated .lex update triggered by $(git rev-parse --short HEAD)" \
            --base dev \
            --head lex/auto-update-$(git rev-parse --short HEAD)
```

---

## Deployment Options

### Option A: Embedded in target repo (Recommended for v1)

Drop the agent into the target repo as `.lex-agent/`:
```
flute/
├── .lex/                    # Knowledge docs
├── .lex-agent/              # This agent
│   ├── agent/
│   │   ├── index.ts
│   │   ├── mapper.ts
│   │   ├── diff.ts
│   │   └── prompts.ts
│   ├── package.json
│   └── tsconfig.json
├── .github/workflows/
│   └── lex-sync.yml
└── crates/...
```

**Pros:** Simple, self-contained, action has direct repo access.
**Cons:** Agent code lives in the project repo.

### Option B: Standalone service (Post-v1)

This repo (`lex-ingestion-agent`) runs as a webhook listener or scheduled job that can target multiple repos.

**Pros:** One agent serves all repos with .lex packs.
**Cons:** Needs webhook config, repo access tokens, more infra.

---

## File Map Parsing Spec

`FILE_MAP.md` has a consistent structure:

```markdown
## Section Name (`crate/path/`)

| File | Purpose | Key Types |
|------|---------|-----------|
| `filename.rs` | Description | Types |
```

**Parser rules:**
1. Section headers contain the base path in backtick-parens: `` (`crate/path/`) ``
2. Table rows contain the filename in backticks in column 1
3. Full path = section base path + filename
4. Quick Lookup table at bottom maps concepts to full paths

**Output:** `Map<string, string[]>` — source file path → list of .lex doc paths that reference it.

---

## Scoping Rules

The agent should be conservative:

| Change Type | Agent Action |
|-------------|-------------|
| Line number shift (code added/removed above referenced line) | Auto-update line refs |
| Function signature change | Update signature in .lex |
| Function renamed | Update name + all references in .lex |
| Function deleted | Flag section for review, don't auto-delete |
| New function added | Flag for manual addition (agent doesn't know if it's .lex-worthy) |
| New file added | Flag for FILE_MAP.md update |
| Logic/flow change | Update flow descriptions in affected .lex sections |
| Comment-only changes | Skip (NO_CHANGES_NEEDED) |

---

## Guard Rails

1. **Never auto-merge** — always open a PR for human review
2. **paths-ignore on `.lex/**`** — prevents infinite trigger loops
3. **Diff scope limit** — if more than 20 files changed (large refactor), skip and flag for manual review
4. **Token budget** — cap Claude API input at ~30K tokens per .lex doc update. If the diff + .lex content exceeds this, summarize the diff first
5. **Rate limit** — max 1 run per 10 minutes (debounce rapid pushes via action concurrency)
6. **Dry run mode** — `LEX_DRY_RUN=true` logs what would change without committing

---

## Dependencies

```json
{
  "dependencies": {
    "@anthropic-ai/sdk": "^0.39.0",
    "simple-git": "^3.27.0"
  },
  "devDependencies": {
    "typescript": "^5.7.0",
    "tsx": "^4.19.0",
    "@types/node": "^22.0.0"
  }
}
```

---

## Implementation Order

1. **`agent/mapper.ts`** — FILE_MAP.md parser + source↔lex mapping
2. **`agent/diff.ts`** — git diff extraction + Rust function heuristics
3. **`agent/prompts.ts`** — operator prompt + request builder
4. **`agent/index.ts`** — orchestration, file I/O, PR creation
5. **Action YAML** — GitHub/Gitea action config
6. **Test against flute repo** — run manually first, verify .lex updates are sane
7. **Deploy** — add to flute repo, enable action

---

## Open Questions (resolve during implementation)

- [ ] Should the agent also update MEMORY.md (Claude Code auto-memory) or just .lex docs?
- [ ] Multi-commit pushes: diff against last known .lex sync point vs always HEAD~1?
- [ ] Should the PR body include a summary of what changed and why?
- [ ] Gitea Actions vs GitHub Actions syntax differences (if moving to Gitea)
