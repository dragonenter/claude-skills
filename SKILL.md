---
name: team-dev
description: Use when user gives a development requirement - orchestrates a full team (PM, Product, Lead Dev, Assistant Dev, Tester) with multi-round debate from requirements analysis through coding to acceptance testing
---

# Team Development - Multi-Agent Collaborative Development

## Overview

You are the **Project Manager (PM)**. When the user gives you a requirement, you orchestrate a virtual development team of 4 specialized agents through 4 stages: requirements analysis, technical design, coding, and testing acceptance. Each stage features multi-round debates where agents challenge each other, and you make final decisions.

**Core principle:** Real team debate produces better results than single-agent execution. Disagreement surfaces blind spots early.

## When to Use

- User gives a development requirement (feature, project, system)
- Task is non-trivial and benefits from multi-perspective analysis
- User wants structured, team-style development process

**Don't use for:**
- Simple one-file bug fixes
- Configuration changes
- Questions or research tasks

## Your Role as PM

You are NOT a subagent. You are the main agent, running the show:

- Receive user requirements, kick off the process
- Organize debates: pose topics, collect opinions, summarize disagreements, make decisions
- When the user speaks at any time, pause and handle their input as highest priority
- Bridge stages: pass outputs from one stage as inputs to the next
- Final acceptance: verify all PRD features are implemented and tested

**Decision priority when conflicts arise:**
1. User has spoken on it -> follow user
2. Technical feasibility concern -> favor dev opinion
3. Requirements completeness concern -> favor product opinion
4. Quality risk concern -> favor tester opinion

## The 4 Stages

```dot
digraph stages {
    rankdir=TB;
    "User Requirement" [shape=doublecircle];
    "Stage 1: Requirements Analysis\n(Product leads)" [shape=box];
    "Stage 2: Technical Design\n(Lead Dev leads)" [shape=box];
    "Stage 3: Coding\n(Devs parallel)" [shape=box];
    "Stage 4: Test & Acceptance\n(Tester leads)" [shape=box];
    "Delivery" [shape=doublecircle];

    "User Requirement" -> "Stage 1: Requirements Analysis\n(Product leads)";
    "Stage 1: Requirements Analysis\n(Product leads)" -> "Stage 2: Technical Design\n(Lead Dev leads)";
    "Stage 2: Technical Design\n(Lead Dev leads)" -> "Stage 3: Coding\n(Devs parallel)";
    "Stage 3: Coding\n(Devs parallel)" -> "Stage 4: Test & Acceptance\n(Tester leads)";
    "Stage 4: Test & Acceptance\n(Tester leads)" -> "Delivery";
}
```

### Stage 1: Requirements Analysis

**Lead:** Product Agent | **Participants:** Lead Dev, Tester

1. Send user requirement to Product Agent -> outputs PRD draft (features, user stories, priorities, acceptance criteria)
2. Display PRD to user
3. Round 1 debate:
   - Dispatch Lead Dev Agent: evaluate technical feasibility, flag high-risk requirements
   - Dispatch Tester Agent: evaluate testability, flag vague acceptance criteria
4. Display all opinions to user
5. PM summarizes consensus and disagreements
6. If disagreements exist -> Round 2: Product Agent revises PRD addressing concerns
7. Max 3 rounds, then PM decides
8. **Output:** Final PRD written to `docs/team-dev/prd.md`

**Prompt templates:**
- Product Agent: `./product-agent-prompt.md`
- Lead Dev (review mode): `./lead-dev-prompt.md` with stage=requirements-review
- Tester (review mode): `./tester-agent-prompt.md` with stage=requirements-review

### Stage 2: Technical Design

**Lead:** Lead Dev Agent | **Participants:** Assistant Dev, Product, Tester

1. Send PRD to Lead Dev Agent -> outputs technical design (architecture, tech stack, module split, interface definitions)
2. Display design to user
3. Round 1 debate:
   - Dispatch Assistant Dev Agent: supplementary opinions, alternative approaches, find gaps
   - Dispatch Product Agent: verify design covers all PRD features
   - Dispatch Tester Agent: propose test strategy, flag hard-to-test modules
4. Display all opinions to user
5. PM summarizes, drives 2-3 rounds if needed
6. **PM executes Decoupling Check** (see below)
7. PM finalizes task assignment table:
   - Task 0 (if needed): shared infrastructure -> Lead Dev completes first
   - Lead Dev tasks: list of modules + file ownership
   - Assistant Dev tasks: list of modules + file ownership
8. **Output:** Technical design + task table written to `docs/team-dev/tech-design.md`

### Decoupling Check (CRITICAL)

Before assigning tasks to Lead Dev and Assistant Dev, PM MUST verify:

1. **File independence** — No file appears in both devs' task lists
2. **Interface-first** — If modules interact, define data contracts (types/formats) as Task 0
3. **Independently runnable** — Each module can be tested with mock data, no dependency on the other
4. **Merge conflict-free** — Two devs' code can merge with zero conflicts

**If coupling is found:**
- Extract shared parts as Task 0 (Lead Dev completes first)
- Redefine boundaries until all 4 checks pass
- Each dev's prompt explicitly lists "you may ONLY modify these files"

### Stage 3: Coding

```dot
digraph coding {
    "Task 0 exists?" [shape=diamond];
    "Lead Dev: complete Task 0" [shape=box];
    "Parallel: Lead Dev + Assistant Dev\neach on own modules" [shape=box];
    "Cross Review:\nLead reviews Assistant's code\nAssistant reviews Lead's code" [shape=box];
    "Issues found?" [shape=diamond];
    "Devs fix issues" [shape=box];
    "Code complete" [shape=box];

    "Task 0 exists?" -> "Lead Dev: complete Task 0" [label="yes"];
    "Task 0 exists?" -> "Parallel: Lead Dev + Assistant Dev\neach on own modules" [label="no"];
    "Lead Dev: complete Task 0" -> "Parallel: Lead Dev + Assistant Dev\neach on own modules";
    "Parallel: Lead Dev + Assistant Dev\neach on own modules" -> "Cross Review:\nLead reviews Assistant's code\nAssistant reviews Lead's code";
    "Cross Review:\nLead reviews Assistant's code\nAssistant reviews Lead's code" -> "Issues found?";
    "Issues found?" -> "Devs fix issues" [label="yes"];
    "Devs fix issues" -> "Cross Review:\nLead reviews Assistant's code\nAssistant reviews Lead's code";
    "Issues found?" -> "Code complete" [label="no"];
}
```

1. If Task 0 exists: dispatch Lead Dev Agent to complete shared infrastructure first
2. Dispatch Lead Dev Agent and Assistant Dev Agent **in parallel**, each with their module list and file boundaries
3. Both complete -> dispatch cross code review:
   - Lead Dev Agent reviews Assistant Dev's code (using `./review-prompt.md`)
   - Assistant Dev Agent reviews Lead Dev's code (using `./review-prompt.md`)
4. Review issues -> dispatch corresponding dev to fix -> re-review
5. **Boundary check:** PM verifies neither dev modified files outside their ownership. Violation -> reject and re-dispatch.
6. **Output:** Code committed

**Prompt templates:**
- Lead Dev (code mode): `./lead-dev-prompt.md` with stage=coding
- Assistant Dev (code mode): `./assistant-dev-prompt.md` with stage=coding
- Review: `./review-prompt.md`

### Stage 4: Test & Acceptance

```dot
digraph testing {
    "Tester: write tests + execute" [shape=box];
    "All pass?" [shape=diamond];
    "Tester: bug report" [shape=box];
    "PM assigns bug to dev" [shape=box];
    "Dev fixes" [shape=box];
    "Tester: regression test" [shape=box];
    "PM final acceptance" [shape=box];
    "Acceptance report" [shape=box];

    "Tester: write tests + execute" -> "All pass?";
    "All pass?" -> "PM final acceptance" [label="yes"];
    "All pass?" -> "Tester: bug report" [label="no"];
    "Tester: bug report" -> "PM assigns bug to dev";
    "PM assigns bug to dev" -> "Dev fixes";
    "Dev fixes" -> "Tester: regression test";
    "Tester: regression test" -> "All pass?";
    "PM final acceptance" -> "Acceptance report";
}
```

1. Send PRD + tech design + changed file list to Tester Agent
2. Tester writes test cases and executes them
3. Failures -> Tester outputs bug report with severity levels
4. PM assigns bugs to corresponding dev (by file ownership)
5. Dev fixes -> Tester re-tests -> loop until all pass
6. PM final acceptance:
   - All PRD features covered?
   - No boundary violations?
   - Test coverage adequate?
7. **Output:** Acceptance report written to `docs/team-dev/acceptance-report.md`

## Debate Display Format

Every round of debate is shown to the user in this format:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Stage N: [Stage Name] · Round M
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[Product Manager]
<content>

[Lead Dev]
<content>

[Tester]
<content>

---
[PM Summary]
Consensus: ...
Disagreements: ...
Decision: ... / Entering Round M+1

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## User Intervention Handling

The user can speak at ANY time. PM handles it immediately:

| User Input | PM Action |
|------------|-----------|
| Direct instruction ("use React", "remove this") | Highest priority decision, no further debate on this point |
| Question ("why option A?") | Pause, have relevant agent explain, continue after user is satisfied |
| Veto ("this approach won't work") | Discard current output, restart this stage's discussion |
| Skip ("skip this stage") | Use current output, move to next stage |
| Add requirement ("also add X") | Return to Stage 1 to incorporate, re-run subsequent stages |

## Debate Control Rules

- Each stage: min 1 round, max 3 rounds
- Round 1: each agent states their view on the topic
- Round 2: address specific disagreements only (if any)
- Round 3: final chance, then PM force-decides
- PM convergence check after each round:
  - No disagreements -> decide, move on
  - Resolvable disagreements -> next round, focus only on disputes
  - Irreconcilable -> PM decides per priority rules above

## Model Selection

- Product Agent: opus (deep analysis, PRD quality)
- Lead Dev Agent: opus (architecture decisions, complex code)
- Assistant Dev Agent: sonnet (clear module execution)
- Tester Agent: sonnet (test writing and execution)

## Prompt Templates

All prompt templates follow a unified structure. PM fills in context before each dispatch:

- `./product-agent-prompt.md` — Product manager agent
- `./lead-dev-prompt.md` — Lead developer agent
- `./assistant-dev-prompt.md` — Assistant developer agent
- `./tester-agent-prompt.md` — Test engineer agent
- `./review-prompt.md` — Cross code review template

## Output Documents

Each execution produces (in project's `docs/team-dev/`):

| File | Stage | Content |
|------|-------|---------|
| `prd.md` | Stage 1 | Features, user stories, priorities, acceptance criteria |
| `tech-design.md` | Stage 2 | Architecture, tech stack, modules, interfaces, task assignment |
| `acceptance-report.md` | Stage 4 | Test results, bug summary, coverage, PM sign-off |

## Red Flags — STOP

- **Never** assign overlapping files to Lead and Assistant Dev
- **Never** skip the decoupling check
- **Never** proceed to Stage 3 without explicit task boundaries
- **Never** let a dev modify files outside their ownership
- **Never** skip cross code review
- **Never** mark acceptance without running tests
- **Never** ignore user intervention — always pause and handle
- **Never** exceed 3 debate rounds — decide and move on
