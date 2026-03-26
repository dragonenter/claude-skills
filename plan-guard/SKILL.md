---
name: plan-guard
description: Use after writing-plans generates a plan and before subagent-driven-development starts execution — validates plan quality, enforces file ownership, and ensures context is passed between tasks
---

# Plan Guard

Overlay skill that patches known issues in `writing-plans` + `subagent-driven-development` workflow. Use this BETWEEN plan creation and plan execution.

**Announce at start:** "I'm using the plan-guard skill to validate the plan and prepare context injection."

## When to Trigger

This skill MUST be invoked:
1. After `writing-plans` produces a plan document, BEFORE user approves execution
2. During `subagent-driven-development` execution, BEFORE each subagent dispatch

## Phase 1: Plan Validation (after writing-plans)

After the plan document is written, perform these checks BEFORE asking the user to approve:

### Check 1: File Ownership (BLOCKING)

Scan all tasks in the plan. Extract every file path from `Create:`, `Modify:`, and `Test:` lines.

**Rule: Each non-test source file MUST appear in exactly ONE task as its primary owner.**

If a file appears in multiple tasks:
- Flag it as a conflict
- Propose resolution: merge the conflicting tasks, or split the file so each task owns a distinct file
- Do NOT proceed until resolved

**Example of a conflict:**
```
Task 4: Data Fetcher Service
  - Create: backend/app/services/data_fetcher.py

Task 8: Data Collector
  - Modify: backend/app/services/data_fetcher.py   ← CONFLICT
```

**Resolution options:**
- Merge Task 8's data_fetcher changes into Task 4
- Create a separate file (e.g., `collector_fetcher.py`) for Task 8's needs
- Add Task 8's data_fetcher changes as a sub-step at the END of Task 4

**Exception:** Test files may be touched by multiple tasks if they test different things. Shared utility files (e.g., `config.py`, `constants.py`) may be touched by multiple tasks only if each task adds independent content (not modifying the same functions).

### Check 2: Task Dependency Order (BLOCKING)

Verify that tasks are ordered so that:
- A task that imports from a file created by another task comes AFTER that task
- Database models come before services that use them
- Services come before API routes that use them
- Backend comes before frontend that calls it

If ordering is wrong, reorder tasks.

### Check 3: Small Task Merging (WARNING)

If a task has fewer than 3 steps and shares context with an adjacent task, suggest merging them. Too many tiny tasks = too many subagents = overhead.

**Target: 5-10 tasks per plan.** If the plan has more than 12 tasks, suggest merging related ones.

### Check 4: Interface Boundaries (WARNING)

For each task that produces files imported by later tasks, verify the plan explicitly defines:
- Function signatures / class interfaces that downstream tasks will use
- Data structures (schemas, models) that are shared

If not defined, add an "Interface Contract" section to the task.

### Validation Output

After all checks, output a summary:

```
Plan Guard Validation:
  File Ownership: ✅ No conflicts (or ❌ N conflicts found — see above)
  Task Order: ✅ Correct (or ❌ Reordered — see above)
  Task Count: ✅ N tasks (or ⚠️ N tasks — consider merging)
  Interfaces: ✅ Defined (or ⚠️ N tasks missing interface contracts)
```

Only proceed to execution after all BLOCKING checks pass.

## Phase 2: Context Injection (during subagent-driven-development)

When `subagent-driven-development` dispatches each implementer subagent, the `## Context` section of the prompt MUST include:

### 2a: Completed Task Summary

For every previously completed task, include:

```
## Previously Completed Tasks

### Task 1: [name] ✅
- Files created: path/to/file1.py, path/to/file2.py
- Key interfaces exposed:
  - `function_name(params) -> return_type` in file1.py
  - `ClassName` with methods: method1(), method2() in file2.py
- Key decisions: [any architectural choices made]

### Task 2: [name] ✅
- Files created: ...
- Key interfaces exposed: ...
```

### 2b: File Change Registry

Maintain a running list of all files created/modified so far:

```
## File Registry (do NOT modify these files unless your task owns them)
- backend/app/models.py (Task 2) — DO NOT MODIFY
- backend/app/schemas.py (Task 3) — DO NOT MODIFY
- backend/app/services/data_fetcher.py (Task 4) — DO NOT MODIFY
- backend/app/services/scanner.py (your task) — YOU OWN THIS
```

### 2c: Relevant Interface Contracts

If the current task imports from files created by previous tasks, include the ACTUAL interface (read the file and extract function signatures, not what the plan said):

```
## Interfaces You Will Use (read from actual code)

From backend/app/models.py:
  class Stock(Base):
      __tablename__ = "stocks"
      id: int, symbol: str, name: str

From backend/app/schemas.py:
  class StockCreate(BaseModel):
      symbol: str, name: str
```

### 2d: Error Prevention Rules

Add to every implementer prompt:

```
## Rules
- Do NOT create or modify files outside your task's file list
- Do NOT re-implement functions that already exist in completed tasks
- If you need to import from a file listed in the File Registry, use the interfaces provided above
- If an interface doesn't match what you need, report NEEDS_CONTEXT — do NOT modify the other file
- If Read tool fails with token limit, use offset/limit parameters to read in chunks
- If Bash command might take >60 seconds, add timeout parameter
```

## Phase 3: Post-Task Verification (after each subagent completes)

After each implementer subagent reports DONE, before dispatching spec reviewer:

1. Run `git diff --name-only` to verify the subagent only touched files in its task's file list
2. If it touched files it doesn't own → reject, re-dispatch with explicit warning
3. Update the File Registry and Completed Task Summary for the next task

## Integration

This skill works WITH (not instead of) the existing workflow:

```
writing-plans → plan-guard (Phase 1: validate) → user approves →
subagent-driven-development → plan-guard (Phase 2: inject context per task) →
  each task: dispatch → plan-guard (Phase 3: verify) → review → next
```
