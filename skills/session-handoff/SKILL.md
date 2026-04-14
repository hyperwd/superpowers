---
name: session-handoff
description: Save current work state to .superpowers/SESSION.md so it can be restored in the next session. Use when context is getting full, work is interrupted, or a phase completes.
---

# Session Handoff

Capture the current work state so it can be restored in the next session without losing context.

**Announce:** "Saving session state to .superpowers/SESSION.md."

## When to Use

- Context window is getting full (many tool calls, dense conversation)
- Pausing work mid-task
- End of a work session
- After completing a phase in a multi-phase workflow (subagent-driven-development, systematic-debugging)
- Auto-triggered by other skills at key checkpoints

## What to Write

Create or overwrite `.superpowers/SESSION.md` with this exact structure:

```markdown
# Session State
**Date:** [YYYY-MM-DD]
**Skill:** [skill currently in use]
**Task:** [one sentence — what are we building/fixing]
**Branch:** [git branch name, from: git branch --show-current]
**Status:** [what is done ✓, what is in-progress, what is pending]
**Next:** [exact next action — specific enough to execute without re-reading the conversation]
**Decisions:** [key decisions made this session that would take time to re-derive]
**Files:** [comma-separated list of files modified or being worked on]
```

Create the `.superpowers/` directory if it does not exist.

## Rules

**"Next" must be specific.** Bad: "Continue implementing". Good: "Dispatch implementer subagent for Task 4 — retry logic. Test already written at tests/api-client.test.ts:47. Use exponential backoff."

**"Decisions" captures non-obvious choices.** Only record things someone re-reading the codebase couldn't easily derive. Auth approach, performance tradeoffs, rejected alternatives.

**Keep it under 20 lines.** If you need more, you're writing a novel. Summarize.

**Always overwrite.** No history needed — the previous state is in git.

## Example

```markdown
# Session State
**Date:** 2026-04-14
**Skill:** subagent-driven-development
**Task:** Add retry logic to the API client for transient 503 errors
**Branch:** feat/api-retry
**Status:** Tasks 1-3 done ✓, Task 4 (exponential backoff implementation) in-progress
**Next:** Dispatch implementer subagent for Task 4. Test already written at tests/api-client.test.ts:47. The test expects 3 retries with backoff delays of 100ms, 200ms, 400ms.
**Decisions:** Exponential backoff chosen over fixed delay (better for burst traffic). Max 3 retries. Jitter ±100ms to prevent thundering herd.
**Files:** src/api-client.ts, tests/api-client.test.ts
```

## Resuming in Next Session

The session-start hook automatically injects SESSION.md into context when a new session starts. The AI will surface it and offer to continue.

To resume manually: say "Continue from SESSION.md" and follow the "Next" action.

## Integration

Called automatically by:
- **superpowers:subagent-driven-development** — after each task completes, and when context pressure is detected
- **superpowers:quick-task** — after commit
- **superpowers:auto-develop** — at stage transitions and on abort
- **superpowers:systematic-debugging** — after each hypothesis round
