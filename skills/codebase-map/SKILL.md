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

## Step 0 — Detect primary language

Run:
  find . -name "*.go" | wc -l
  find . -name "*.ts" -o -name "*.tsx" | wc -l
  find . -name "*.py" | wc -l
  find . -name "*.java" | wc -l
  find . -name "*.rs" | wc -l

The language with the most files is the primary language. Use language-specific patterns below.

## Step 1 — File inventory

For each source file, output one line:
  <path> | size: <lines> | exports: <top exported names> | imports: <top imported modules>

Use language-appropriate grep:

**Go:**
  # Exports (capitalized identifiers at top level)
  grep -n "^func [A-Z]\|^type [A-Z]\|^var [A-Z]\|^const [A-Z]" <file> | head -5
  # Internal imports
  grep -n "\".*internal/\|\".*pkg/" <file> | head -5

**TypeScript / JavaScript:**
  grep -n "^export " <file> | head -5
  grep -n "^import\|require(" <file> | head -5

**Python:**
  grep -n "^def \|^class " <file> | head -5
  grep -n "^from \|^import " <file> | head -5

**Java / Kotlin:**
  grep -n "^public class\|^public interface\|^public fun\|^public " <file> | head -5
  grep -n "^import " <file> | head -5

**Rust:**
  grep -n "^pub fn\|^pub struct\|^pub enum\|^pub trait" <file> | head -5
  grep -n "^use " <file> | head -5

## Step 2 — Size stats

- Total file count by extension
- All files sorted by line count, descending (show ALL files, not just top 10):
    find . -name "*.<ext>" -not -path "*/vendor/*" -not -path "*/.git/*" \
      -exec wc -l {} + | sort -rn

Output max 250 lines. Prioritize files with the most exports/definitions.
End your output with: ## AGENT-A COMPLETE
```

---

**Agent B — Patterns**

```
You are a codebase patterns agent. Your only job is to identify conventions.

Use ONLY Glob, Grep, and Bash. You may Read up to 2 config files:
  1. The manifest file: package.json / go.mod / Cargo.toml / pyproject.toml / pom.xml
  2. The build file if present: Makefile / Taskfile.yml / build.gradle

Find and report:

1. **Test framework & files:**
   - Go:    find . -name "*_test.go" | head -20 ; grep -rn "func Test\|func Bench" --include="*_test.go" | wc -l
   - JS/TS: grep -rn "describe\|it(\|test(\|expect(" --include="*.test.*" --include="*.spec.*" -l | head -10
   - Python: find . -name "test_*.py" -o -name "*_test.py" | head -20
   Report: file count, naming pattern, approximate test count

2. **Config files present:**
   List all of: *.config.*, .env*, Makefile, Dockerfile, docker-compose.*, *.yaml, *.toml in project root

3. **Build/run scripts:**
   From manifest or Makefile targets (make help or grep "^[a-zA-Z].*:" Makefile | head -20)

4. **Directory structure:**
   Top 2 levels only (find . -maxdepth 2 -type d | sort)

5. **Naming conventions:**
   - File naming: snake_case? camelCase? PascalCase?
   - Function/class naming patterns from grep

6. **Environment variables used:**
   grep -rn "os.Getenv\|process.env\.\|os.environ\|std::env::var" --include="*.<ext>" | \
   grep -o '"[A-Z_]*"' | sort -u | head -20

Output max 120 lines.
End your output with: ## AGENT-B COMPLETE
```

---

**Agent C — Module Graph**

```
You are a module graph agent. Your only job is to find the dependency hubs.

Use ONLY Grep and Bash. Do NOT use the Read tool.

## Step 0 — Detect primary language (same as Agent A)

## Step 1 — Hub files (most imported)

Use language-appropriate import pattern:

**Go:**
  grep -rh "\"" --include="*.go" | grep -o '"[^"]*internal[^"]*"' | sort | uniq -c | sort -rn | head -25

**TypeScript / JavaScript:**
  grep -rh "from ['\"]" --include="*.ts" --include="*.tsx" --include="*.js" | \
  grep -o "from '[^']*'\|from \"[^\"]*\"" | sort | uniq -c | sort -rn | head -25

**Python:**
  grep -rh "^from \|^import " --include="*.py" | sort | uniq -c | sort -rn | head -25

**Rust:**
  grep -rh "^use " --include="*.rs" | sort | uniq -c | sort -rn | head -25

## Step 2 — Entry points

Files that import many others but are not themselves imported often (likely entry points or orchestrators):
  # Count how many times each file is referenced as an import target
  grep -rh "import\|require\|use " --include="*.<ext>" | grep -o '[a-zA-Z0-9_/.-]*' | sort | uniq -c | sort -rn | head -20

## Step 3 — Module boundaries

Directories that contain an index file (index.ts/index.go/mod.rs/__init__.py) — these are likely public API surfaces:
  find . -name "index.*" -o -name "__init__.py" -o -name "mod.rs" | grep -v vendor | grep -v node_modules

## Step 4 — Circular dependency signals

For Go: look for pairs where pkg A imports pkg B and pkg B imports pkg A:
  # List all internal import relationships
  grep -rn "\"" --include="*.go" | grep "internal/" | \
  sed 's/.*"\(.*internal\/[^"]*\)".*/\1/' | sort | uniq -c | sort -rn | head -20

Output max 120 lines. Focus on the top 20 most-imported files.
End your output with: ## AGENT-C COMPLETE
```

---

Wait for all three agents to complete (confirm all outputs end with their COMPLETE marker) before starting Phase 2.

## Phase 2 — Synthesis

You now have ~490 lines of structured facts across three agents. This fits in context.

**Step 1:** Read all three agent outputs in full.

**Step 2 — Scale-aware file reading:**

Count total source files from Agent A. Then read files according to this scale:

| Scale | Files to Read | Strategy |
|-------|--------------|----------|
| < 20 files | All of them | Read everything |
| 20–60 files | 15–18 files | Top hub files + at least 2 per major directory |
| 60–100 files | 22–28 files | Top hubs + spread across all directories evenly |
| 100+ files | 30–40 files | Top hubs + representative file per subdirectory |

**Selection priority:**
1. Files referenced most in Agent C's import graph (hub files)
2. Files > 100 lines that haven't appeared in import graph (may be standalone business logic)
3. At least one file from each top-level directory that contains source code
4. Entry point files identified by Agent C

**Do NOT skip files just because they weren't in the import graph** — Go, Python, and other languages have many significant files that are launched by goroutines/processes rather than imported.

**Step 3 — Write `.superpowers/codebase/OVERVIEW.md`:**

```markdown
# Codebase Overview
**Generated:** [date]
**Scale:** [N source files, approx N total lines]

## Tech Stack
[languages, frameworks, major libraries — include versions if visible in manifest]

## Architecture
[how the codebase is organized — key directories, module boundaries, layering pattern]

## Entry Points
[where execution begins — main files, CLI entry, server start, cron jobs, etc.]

## Core Abstractions
[ALL files > 100 lines or that appear in the import graph, one sentence each.
Do NOT cap this list — completeness matters more than brevity here.]

## Testing
[test file count, naming pattern, how to run tests, approximate coverage signal]

## How to Write Code Here
[conventions: file naming, test patterns, how to add a new feature, import style]

## Key Dependencies Between Modules
[cross-cutting relationships that affect where new code should go]

## Gotchas
[non-obvious things that could trip up a new contributor.
IMPORTANT: every Gotcha must include a file:line reference as evidence.
Do NOT write security or data-loss warnings without grep-verified proof.]
```

Create `.superpowers/codebase/` directory if it does not exist.

**Step 4:** Tell the user: "Codebase map written to `.superpowers/codebase/OVERVIEW.md`. [2-3 sentence summary of what you found]"

## Refresh

To update the map after significant changes: invoke this skill again. It overwrites OVERVIEW.md.

## Integration

- **superpowers:auto-develop** — checks for OVERVIEW.md at startup, offers to run this skill if absent
- **superpowers:brainstorming** — reads OVERVIEW.md in the "Explore project context" step if present
- **superpowers:writing-plans** — reads OVERVIEW.md before defining file structure; offers to run this skill if absent and codebase is non-trivial
- **superpowers:quick-task** — read OVERVIEW.md in the "Locate" step if present
