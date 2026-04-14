---
name: auto-develop
description: Full development pipeline — automatically chains brainstorming → writing-plans → subagent-driven-development. Human approves at design, spec, and execution gates. No manual skill invocations needed between stages.
---

# Auto Development Pipeline

One command for the complete development workflow. You are the orchestrator — you call each skill in sequence. The user only makes decisions at approval gates.

**Announce at start:** "Using auto-develop pipeline: brainstorming → writing-plans → subagent-driven-development."

**Context health check:** If the current session already has a dense history (many tool calls, long conversation), warn the user: "This session has significant context usage. For best results with auto-develop, consider starting a fresh session — each subagent gets an isolated context window, but the orchestrator (this session) benefits from a clean start." Offer to continue anyway or wait for a fresh session.

Check for `.superpowers/codebase/OVERVIEW.md`. If it does not exist and this is a non-trivial codebase, offer to run superpowers:codebase-map first: "No codebase map found. Run codebase-map for better context? (recommended for unfamiliar codebases)"

## Stage 1 — Design

Invoke superpowers:brainstorming. Follow it completely without skipping steps.

**Gate 1 — Design approval (user must approve):**
Present design sections. Wait for explicit approval before writing the spec.

**Gate 2 — Spec approval (user must approve):**
After writing the spec document, ask the user to review it. Do NOT proceed until they approve or request changes.

If the user says "stop", "pause", or "not yet" at either gate:
→ Invoke superpowers:session-handoff recording `Stage: design, Gate: [1 or 2]`
→ Stop. Do not continue to Stage 2.

## Stage 2 — Plan

After Gate 2 passes, immediately invoke superpowers:writing-plans. **Do not ask the user whether to proceed** — this transition is automatic.

**Gate 3 — Execution approval (user must confirm):**
After the plan file is written, announce: "Plan complete at `[path]`. Ready to start execution with subagent-driven-development?"

Do NOT proceed to Stage 3 unless the user confirms.

If the user says "stop" or "not yet":
→ Invoke superpowers:session-handoff recording `Stage: plan-complete, Plan: [path]`
→ Stop.

## Stage 3 — Execute

After Gate 3 passes:

1. **Set up isolated workspace:** Invoke `superpowers:using-git-worktrees` to create a dedicated worktree. This is required by subagent-driven-development.
2. **Invoke superpowers:subagent-driven-development.** Follow it completely.

After all tasks are marked complete, invoke superpowers:session-handoff recording completion.

## Summary of What's Automated

| Transition | Automated? |
|-----------|------------|
| brainstorming → writing-plans | ✅ Automatic after Gate 2 |
| writing-plans → subagent-driven-development | ✅ Automatic after Gate 3 |
| session-handoff on completion | ✅ Automatic |
| session-handoff on abort | ✅ Automatic |

## Summary of What Requires Human Approval

| Gate | What user decides |
|------|------------------|
| Gate 1 | Design sections look correct |
| Gate 2 | Written spec is accurate and complete |
| Gate 3 | Ready to start code execution |

## For Quick Tasks

If the task is clearly small (< 30 min, single file), suggest superpowers:quick-task instead: "This looks like a quick task — use quick-task for a lighter workflow?"
