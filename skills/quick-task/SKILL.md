---
name: quick-task
description: Use for focused tasks under 30 minutes — bug fixes, small additions, config changes. Skips brainstorming and plan docs. TDD required.
---

# Quick Task

Execute a focused task without brainstorming or plan documents.

**Boundary:** If the task touches more than 2 files, has unclear requirements, or takes > 30 minutes — use superpowers:brainstorming instead.

**Announce at start:** "Using quick-task skill to execute this directly."

## When to Use

- Bug fix with a clear reproduction
- Single-function refactor
- Config or dependency update
- Small feature addition (one file, known approach)

## When NOT to Use

- "I'm not sure how to approach this" → use superpowers:brainstorming
- Task touches 3+ files → use superpowers:brainstorming
- Requires understanding system behavior first → use superpowers:codebase-map then brainstorming
- Involves architecture decisions → use superpowers:brainstorming

## Process

### 1. Understand

State the task in one sentence. If you can't, stop — it needs brainstorming.

### 2. Locate

Find the exact files and lines affected. Read them. Do not skip this step.

### 3. Write Failing Test (mandatory — TDD iron law)

Use superpowers:test-driven-development.

Write one test that captures the desired behavior. Run it. Confirm it fails for the right reason — not a syntax error, not an existing failure. A failure that proves the feature is missing.

```bash
# Example
npm test path/to/test.test.ts
# Expected: FAIL — [reason matching what you're about to implement]
```

### 4. Minimal Implementation

Write the smallest code that makes the test pass. No extras, no "while I'm here" improvements.

### 5. Verify Green

Run the full test suite. All tests pass. No new failures.

```bash
npm test
# Expected: all pass, output pristine
```

### 6. Commit

Atomic commit with a conventional message.

```bash
git add [specific files only]
git commit -m "fix: [what was fixed and why]"
```

## Red Flags

| Signal | Action |
|--------|--------|
| Can't state task in one sentence | Stop. Use brainstorming. |
| Test passes immediately | Wrong test. Fix it first. |
| "Just this one thing that's related..." | Stop. Separate task. |
| 3rd file needs touching | Stop. Use brainstorming. |
| Fix doesn't work after 2 attempts | Stop. Use systematic-debugging. |

## Integration

- **Needs debugging first:** Use superpowers:systematic-debugging to find root cause, then return here to implement the fix.
- **Part of a larger feature:** Use superpowers:auto-develop for the full pipeline instead.
- **Unfamiliar codebase:** Run superpowers:codebase-map first.
