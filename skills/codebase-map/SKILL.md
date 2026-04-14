---
name: codebase-map
description: Three-phase MapReduce codebase understanding. Phase 0 determines optimal parallelism based on codebase scale. Phase 1 dispatches 1-6 parallel agents to extract facts (no file body reading). Phase 2 uses a single synthesis agent that sees all facts to produce coherent global understanding.
---

# Codebase Map

Build genuine global understanding of an unfamiliar or large codebase.

**Why two phases:** Parallel agents with isolated contexts cannot share findings — each sees only its slice. By separating *fact extraction* (parallel, cheap) from *synthesis* (single agent, coherent), Phase 2 can see all cross-cutting relationships at once.

**Announce at start:** "Using codebase-map: Phase 0 (scale analysis) → Phase 1 (parallel extraction with N agents) → Phase 2 (synthesis)."

## When to Use

- Starting work on an unfamiliar codebase
- Before brainstorming a large or cross-cutting feature
- After major refactoring (refresh the map)
- When auto-develop finds no OVERVIEW.md and you opt in

**Skip for:** greenfield projects with no code yet, or trivial codebases (< 10 files).

## Phase 0 — Scale Analysis & Parallelism Planning

Before dispatching agents, determine optimal parallelism based on codebase scale.

**Step 1 — Count source files:**

Run a fast file count for common languages:

```bash
find . \( -name "*.go" -o -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" \
        -o -name "*.py" -o -name "*.java" -o -name "*.rs" -o -name "*.kt" \
        -o -name "*.rb" -o -name "*.php" -o -name "*.cs" \) \
        -not -path "*/node_modules/*" \
        -not -path "*/vendor/*" \
        -not -path "*/.git/*" \
        -not -path "*/dist/*" \
        -not -path "*/build/*" | wc -l
```

**Step 2 — Determine agent count:**

| File Count | Agent Count | Strategy |
|------------|-------------|----------|
| < 20 | 1 | Single merged agent |
| 20-59 | 2 | Split into inventory+graph vs patterns+config |
| 60-99 | 3 | Original 3-agent split (A/B/C) |
| 100-199 | 4 | 3 core + 1 deep file analysis |
| 200-399 | 5 | 3 core + 2 supplementary |
| 400+ | 6 (max) | 3 core + 3 specialized |

**Step 3 — Announce plan:**

Tell the user: "Detected [N] source files. Dispatching [M] agents in parallel for Phase 1."

## Phase 1 — Parallel Fact Extraction

Dispatch subagents **in parallel** (single message, multiple Agent calls) based on the agent count determined in Phase 0. Each uses ONLY Glob, Grep, and Bash — no Read of file bodies.

### Agent Count: 1

Use a single comprehensive agent that merges all tasks:

```
You are a comprehensive codebase analysis agent. Your job is to produce a complete structured fact list combining file inventory, patterns, and module relationships.

Use ONLY Glob, Grep, and Bash. Do NOT use the Read tool on any file (you may Read up to 2 config files: the manifest and build file if present).

## Step 0 — Detect primary language

Run:
  find . -name "*.go" | wc -l
  find . -name "*.ts" -o -name "*.tsx" | wc -l
  find . -name "*.py" | wc -l
  find . -name "*.java" | wc -l
  find . -name "*.rs" | wc -l

The language with the most files is the primary language.

## Step 1 — File inventory

For each source file, output one line:
  <path> | size: <lines> | exports: <top exported names> | imports: <top imported modules>

Use language-appropriate grep patterns (see Agent A definition below for details).

## Step 2 — Patterns & conventions

- Test framework & files (naming pattern, file count, approximate test count)
- Config files present (*.config.*, .env*, Makefile, Dockerfile, etc.)
- Build/run scripts (from manifest or Makefile targets)
- Directory structure (top 2 levels)
- Naming conventions (file and function/class patterns)
- Environment variables used

## Step 3 — Module graph

- Hub files (most imported) — top 25
- Entry points (files that import many but aren't imported often)
- Module boundaries (index files, __init__.py, mod.rs)
- Circular dependency signals (if applicable)

Output max 450 lines. Prioritize completeness over brevity.
End your output with: ## SINGLE-AGENT COMPLETE
```

### Agent Count: 2

**Agent 1 — Inventory & Graph**

```
You are a codebase inventory and module graph agent. Your job is to produce structured facts about file structure and dependencies.

Use ONLY Glob, Grep, and Bash. Do NOT use the Read tool on any file.

## Step 0 — Detect primary language

Run:
  find . -name "*.go" | wc -l
  find . -name "*.ts" -o -name "*.tsx" | wc -l
  find . -name "*.py" | wc -l
  find . -name "*.java" | wc -l
  find . -name "*.rs" | wc -l

The language with the most files is the primary language.

## Step 1 — File inventory

For each source file, output one line:
  <path> | size: <lines> | exports: <top exported names> | imports: <top imported modules>

Use language-appropriate grep (see Agent A definition below).

## Step 2 — Size stats

- Total file count by extension
- All files sorted by line count, descending

## Step 3 — Module graph

- Hub files (most imported) — top 25 with import counts
- Entry points (files that import many but aren't imported often)
- Module boundaries (index files, __init__.py, mod.rs)

Output max 300 lines.
End your output with: ## AGENT-1 COMPLETE
```

**Agent 2 — Patterns & Config**

```
You are a codebase patterns and configuration agent. Your job is to identify conventions and project setup.

Use ONLY Glob, Grep, and Bash. You may Read up to 2 config files:
  1. The manifest file: package.json / go.mod / Cargo.toml / pyproject.toml / pom.xml
  2. The build file if present: Makefile / Taskfile.yml / build.gradle

Find and report:

1. **Test framework & files:**
   - File count, naming pattern, approximate test count
   - Use language-appropriate patterns (see Agent B definition below)

2. **Config files present:**
   List all of: *.config.*, .env*, Makefile, Dockerfile, docker-compose.*, *.yaml, *.toml in project root

3. **Build/run scripts:**
   From manifest or Makefile targets

4. **Directory structure:**
   Top 2 levels only (find . -maxdepth 2 -type d | sort)

5. **Naming conventions:**
   File naming: snake_case? camelCase? PascalCase?
   Function/class naming patterns from grep

6. **Environment variables used:**
   grep -rn "os.Getenv\|process.env\.\|os.environ\|std::env::var" --include="*.<ext>" | \
   grep -o '"[A-Z_]*"' | sort -u | head -20

Output max 200 lines.
End your output with: ## AGENT-2 COMPLETE
```

### Agent Count: 3

Use the original 3-agent split with Agent A, B, and C defined below.

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

### Agent Count: 4+

For codebases with 100+ files, use Agent A/B/C as the core, plus additional specialized agents:

**Agent D — Large File Analysis** (4+ agents)

```
You are a large file analysis agent. Your job is to identify complexity hotspots in large files.

Use ONLY Grep and Bash. Do NOT use the Read tool on file bodies.

## Step 1 — Find large files (> 500 lines)

For each primary language:
  find . -name "*.<ext>" -not -path "*/vendor/*" -not -path "*/.git/*" -not -path "*/node_modules/*" \
    -exec wc -l {} + | awk '$1 > 500' | sort -rn | head -30

## Step 2 — Extract signatures from large files

For each large file, extract function/class signatures:

**Go:**
  grep -n "^func \|^type .*struct\|^type .*interface" <file> | head -20

**TypeScript / JavaScript:**
  grep -n "^export function\|^export class\|^function\|^class " <file> | head -20

**Python:**
  grep -n "^def \|^class \|^async def " <file> | head -20

**Java / Kotlin:**
  grep -n "^public class\|^public interface\|^public fun\|^private class" <file> | head -20

**Rust:**
  grep -n "^pub fn\|^fn \|^pub struct\|^struct \|^pub enum\|^impl " <file> | head -20

## Step 3 — Complexity signals

For each large file, count:
- Nested control flow depth (estimate from indentation)
- Number of imports/dependencies
- Comment density (comment lines / total lines)

Output: <file> | <lines> | <functions/classes> | <imports> | <comment%>

Output max 150 lines.
End your output with: ## AGENT-D COMPLETE
```

**Agent E — Documentation & Config Audit** (5+ agents)

```
You are a documentation and configuration audit agent. Your job is to assess project setup and documentation completeness.

Use ONLY Glob, Grep, and Bash. You may Read up to 3 documentation files: README, CONTRIBUTING, and one docs/index file.

## Step 1 — Documentation inventory

Find and categorize documentation:
  - README files (README.md, README.txt, etc.)
  - Developer docs (CONTRIBUTING.md, DEVELOPMENT.md, docs/ directory)
  - API documentation (OpenAPI specs, GraphQL schemas, proto files)
  - Architecture docs (ARCHITECTURE.md, ADRs, diagrams)

List each with approximate line count.

## Step 2 — Configuration files audit

Find and report:
  - CI/CD configs (.github/workflows/, .gitlab-ci.yml, Jenkinsfile, etc.)
  - Container configs (Dockerfile, docker-compose.yml, .dockerignore)
  - Deployment configs (k8s manifests, terraform, ansible)
  - Editor configs (.editorconfig, .vscode/, .idea/)
  - Linter/formatter configs (.eslintrc, .prettierrc, .pylintrc, rustfmt.toml)

## Step 3 — Comment density by directory

For each top-level source directory, estimate comment density:
  grep -rn "^[[:space:]]*#\|^[[:space:]]*//" <dir> --include="*.<ext>" | wc -l
  find <dir> -name "*.<ext>" -exec wc -l {} + | awk '{sum+=$1} END {print sum}'
  
Report: <directory> | <comment lines> | <total lines> | <ratio>

Output max 100 lines.
End your output with: ## AGENT-E COMPLETE
```

**Agent F — Quality Signals** (6 agents)

```
You are a code quality signals agent. Your job is to identify quality indicators and technical debt signals.

Use ONLY Grep and Bash. Do NOT use the Read tool on source files.

## Step 1 — Test coverage estimate

- Test file count vs source file count ratio
- Test line count vs source line count ratio
- Test framework configurations (coverage tools, thresholds)

## Step 2 — Linter and formatter presence

Check for:
  - Linter configs and which rules are enabled
  - Formatter configs (Prettier, Black, gofmt, rustfmt)
  - Pre-commit hooks (.pre-commit-config.yaml, .husky/)

## Step 3 — Code duplication patterns

Search for potential duplication signals:
  - Files with similar names (e.g., utils.ts, util.ts, helpers.ts)
  - Repeated function/class name patterns across files
  
Run:
  find . -name "*util*" -o -name "*helper*" -o -name "*common*" | grep -v node_modules | grep -v vendor

## Step 4 — TODO/FIXME/HACK comments

Count technical debt markers:
  grep -rn "TODO\|FIXME\|HACK\|XXX\|BUG" --include="*.<ext>" | wc -l
  
Sample top 10 most critical:
  grep -rn "FIXME\|HACK\|XXX" --include="*.<ext>" | head -10

## Step 5 — Dependency health

From manifest files:
  - Total dependency count (production + dev)
  - Outdated dependency signals (if lockfile has version ranges)

Output max 80 lines.
End your output with: ## AGENT-F COMPLETE
```

---

Wait for all dispatched agents to complete (confirm all outputs end with their COMPLETE marker) before starting Phase 2.

## Phase 2 — Synthesis

You now have structured facts across all dispatched agents. Typical output volume ranges from 450 lines (single agent) to 850 lines (6 agents). This fits comfortably in context.

**Step 1:** Read all agent outputs in full.

**Step 2 — Scale-aware file reading:**

Count total source files from the agent outputs (file inventory data). Then read files according to this scale:

| Scale | Files to Read | Strategy |
|-------|--------------|----------|
| < 20 files | All of them | Read everything |
| 20–60 files | 15–18 files | Top hub files + at least 2 per major directory |
| 60–100 files | 22–28 files | Top hubs + spread across all directories evenly |
| 100+ files | 30–40 files | Top hubs + representative file per subdirectory |

**Selection priority:**
1. Files referenced most in the module graph output (hub files)
2. Files > 100 lines that haven't appeared in import graph (may be standalone business logic)
3. At least one file from each top-level directory that contains source code
4. Entry point files identified by the agents

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
