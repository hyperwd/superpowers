---
name: codebase-map
description: Two-phase MapReduce codebase understanding. Phase 1 dispatches 3 parallel agents to extract facts (no file body reading). Phase 2 uses a single synthesis agent that sees all facts to produce coherent global understanding.
---

# Codebase Map

Build genuine global understanding of an unfamiliar or large codebase.

**Why two phases:** Parallel agents with isolated contexts cannot share findings — each sees only its slice. By separating *fact extraction* (parallel, cheap) from *synthesis* (single agent, coherent), Phase 2 can see all cross-cutting relationships at once.

**Announce at start:** "Using codebase-map: Phase 1 (parallel extraction) → Phase 2 (synthesis)."

## When to Use

- Starting work on an unfamiliar codebase
- Before brainstorming a large or cross-cutting feature
- After major refactoring (refresh the map)
- When auto-develop finds no OVERVIEW.md and you opt in

**Skip for:** greenfield projects with no code yet, or trivial codebases (< 10 files).

## Phase 1 — Parallel Fact Extraction

Dispatch these 3 subagents **in parallel** (single message, multiple Agent calls). Each uses ONLY Glob, Grep, and Bash — no Read of file bodies.

---

**Agent A — File Inventory**

```
You are a codebase inventory agent. Your only job is to produce a compact, structured fact list.

Use ONLY Glob, Grep, and Bash. Do NOT use the Read tool on any file.

Produce a file inventory with this format (one line per file):
  <path> | exports: <comma-separated exported names> | imports: <comma-separated imported modules>

Use grep to extract exports:
  grep -r "^export " --include="*.ts" --include="*.js" -l   (find files with exports)
  grep -n "^export " <file>                                  (get export names)

Use grep to extract imports:
  grep -n "^import\|require(" <file> | head -5              (top imports only)

Also report:
- Total file count by extension (find . -name "*.ts" | wc -l etc.)
- Largest files by line count (find . -name "*.ts" -exec wc -l {} + | sort -rn | head -10)

Output max 150 lines. Prioritize files with the most exports.
End your output with: ## AGENT-A COMPLETE
```

---

**Agent B — Patterns**

```
You are a codebase patterns agent. Your only job is to identify conventions.

Use ONLY Glob, Grep, and Bash. You may Read ONE file: package.json (or equivalent manifest).

Find and report:
1. Test framework: grep for "describe\|it(\|test(\|expect(" in test files
2. Test file naming pattern: what do test files look like? (*.test.ts, *.spec.js, __tests__/?)
3. Config files present: list *.config.*, .env*, Makefile, Dockerfile, docker-compose.*
4. Build/run scripts: from package.json "scripts" section only
5. Directory structure: top-level directories and their apparent purpose
6. Naming conventions: grep for common patterns (PascalCase components, camelCase utils, etc.)

Output max 80 lines.
End your output with: ## AGENT-B COMPLETE
```

---

**Agent C — Module Graph**

```
You are a module graph agent. Your only job is to find the dependency hubs.

Use ONLY Grep and Bash. Do NOT use the Read tool.

Find:
1. Hub files (most imported): 
   grep -r "from ['\"]" --include="*.ts" --include="*.js" -h | \
   grep -o "from '.*'" | sort | uniq -c | sort -rn | head -20

2. Entry points (files that import others but are not imported):
   Compare files that appear as import targets vs files that have imports

3. Circular dependency signals:
   Look for A imports B and B imports A patterns

4. Module boundaries:
   List directories that have an index.ts/index.js (likely public API boundaries)

Output max 80 lines. Focus on the top 15 most-imported files.
End your output with: ## AGENT-C COMPLETE
```

---

Wait for all three agents to complete (confirm all outputs end with their COMPLETE marker) before starting Phase 2.

## Phase 2 — Synthesis

You now have ~310 lines of structured facts across three agents. This fits in context.

**Step 1:** Read all three agent outputs in full.

**Step 2:** Based on Agent C's hub files and entry points, selectively Read 5-8 key files — the ones that are imported most and appear to be core abstractions. Do not read every file.

**Step 3:** Write `.superpowers/codebase/OVERVIEW.md`:

```markdown
# Codebase Overview
**Generated:** [date]
**Scale:** [N source files, approx N total lines]

## Tech Stack
[languages, frameworks, major libraries, test setup]

## Architecture
[how the codebase is organized — key directories, module boundaries, layering]

## Entry Points
[where execution begins — main files, CLI entry, server start, etc.]

## Core Abstractions
[top 5-8 hub files, one sentence each describing what they do]

## How to Write Code Here
[conventions: file naming, test patterns, how to add a new feature, import style]

## Key Dependencies Between Modules
[cross-cutting relationships that affect where new code should go]

## Gotchas
[non-obvious things: surprising patterns, known rough edges, things that look one way but work another]
```

Create `.superpowers/codebase/` directory if it does not exist.

**Step 4:** Tell the user: "Codebase map written to `.superpowers/codebase/OVERVIEW.md`. [2-3 sentence summary of what you found]"

## Refresh

To update the map after significant changes: invoke this skill again. It overwrites OVERVIEW.md.

## Integration

- **superpowers:auto-develop** — checks for OVERVIEW.md at startup, offers to run this skill if absent
- **superpowers:brainstorming** — reads OVERVIEW.md in the "Explore project context" step if present
- **superpowers:quick-task** — read OVERVIEW.md in the "Locate" step if present
