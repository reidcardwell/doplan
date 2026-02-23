# doplan

> Designed and built by [Reid Cardwell](https://reidcardwell.ai) — AI-native development systems.

**doplan** is an automated software development pipeline that transforms a written development plan into fully implemented, tested, reviewed, and version-controlled code — with minimal human intervention.

This repository documents the architecture, design decisions, and operational details of the system. It is one of several workflow artifacts from my AI-native development practice, where I design and operate the systems rather than writing code directly.

---

## Context

This pipeline runs on top of [Claude Code](https://claude.ai/code) (Anthropic's agentic coding tool) and integrates with [Obsidian](https://obsidian.md) as the knowledge management layer where development plans are stored and updated. It is designed for Laravel and Python projects but the architecture is language-agnostic.

**What you need to understand this document:**
- Familiarity with Git branching and pull requests
- Basic understanding of how AI coding agents work
- No installation required — this is a documentation and architecture reference

**Metrics** from live pipeline executions are tracked automatically and published at [reidcardwell.ai](https://reidcardwell.ai).

---

## Why This Exists

Software development has always been inconsistent by nature. Even experienced developers approach the same kind of task differently each time — different branching strategies, different testing habits, different levels of documentation rigor. Doplan was built to change that. It codifies the entire development process into a repeatable, measurable pipeline: the same steps, the same quality gates, the same handoff documentation, every time. What used to vary by developer, by day, and by deadline now executes the same way whether it's 9am on a Monday or the end of a long sprint.

## Overview

At its core, doplan takes a structured checklist of development tasks and orchestrates an entire team of specialized AI agents to execute them end-to-end: writing code, running tests, committing changes, creating pull requests, performing code reviews, and even auto-fixing issues found during review.

The system was built to solve a real problem: executing multi-phase development plans is tedious and error-prone when done manually, even with AI assistance. Each phase requires dozens of repetitive steps — creating branches, reading task requirements, implementing code, writing tests, committing changes, opening PRs, requesting reviews. Doplan automates the entire sequence while maintaining the safety guardrails and quality gates that professional software development demands.

---

## How It Works: The Big Picture

A development plan is a structured document stored in Obsidian that breaks work into **phases**, and each phase into **tasks**. For example, a plan to build a customer management feature might have:

- **Phase 1:** Database setup (3 tasks)
- **Phase 2:** API endpoints (4 tasks)
- **Phase 3:** User interface (5 tasks)
- **Phase 4:** Testing and polish (3 tasks)

When doplan runs, it creates a dedicated workspace (a Git branch) for the entire plan, then executes each phase one at a time. Each phase gets its own sub-workspace, its own pull request, its own code review, and its own merge back into the plan workspace. When all phases complete, a final pull request rolls everything up for human review.

```
Development Plan (Obsidian)
         |
         v
  ┌──────────────┐
  │   doplan     │  Plan-level orchestrator
  │  (team lead) │  Reads the plan, creates the workspace,
  └──────┬───────┘  manages the team
         |
    ┌────┴────┐────────┐─────────┐
    v         v        v         v
 Phase 1    Phase 2   Phase 3   Phase 4
 ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐
 │dophase│ │dophase│ │dophase│ │dophase│   Phase-level executors
 └──┬────┘ └───┬───┘ └───┬───┘ └───┬───┘   (one per phase, sequential)
    |          |         |         |
    v          v         v         v
  Code    →  Test   →  Review  →  Merge   (repeated per phase)
```

---

## Step 1: Plan Reading and Initialization

When the user invokes doplan with a plan name, the system first locates and reads the plan file from Obsidian. It parses all phase headers to discover how many phases exist and what each one is titled.

### What Happens

1. The plan name is resolved to a file in the Obsidian vault (e.g., `Customer` becomes `CustomerTodo.md`)
2. The plan file is read and all phases are discovered by scanning for structured headers
3. A summary of discovered phases is displayed to the user

### Why This Matters

This step validates that the plan exists and is well-formed before any work begins. If the plan file is missing or has no recognizable phases, the system halts immediately with a clear error rather than failing partway through execution.

---

## Step 2: Workspace Preparation

Before any code is written, doplan prepares a clean, isolated workspace using Git branches.

### What Happens

1. **Branch safety check** — The system verifies it's starting from the correct base branch and refuses to proceed from protected environments (e.g., staging or production)
2. **Uncommitted work detection** — If there are any unsaved changes in the current workspace, the user is prompted to either commit them or abort
3. **Plan branch creation** — A dedicated long-lived branch is created for the entire plan (e.g., `plan/customer`), keeping all plan work isolated from other development

### Resume Support

If the plan branch already exists (from a previous interrupted run), the system detects this and offers to resume from where it left off rather than starting over. This is critical for reliability — if a session is interrupted for any reason, no work is lost.

### Branch Architecture

```
dev (main development branch)
 |
 +-- plan/customer  (long-lived plan branch)
      |
      +-- feat/customer-phase-1-setup     → PR into plan/customer
      +-- feat/customer-phase-2-api       → PR into plan/customer
      +-- feat/customer-phase-3-ui        → PR into plan/customer
      |
      +-- Final PR: plan/customer → dev   (manual review required)
```

Each phase creates its own short-lived branch from the plan branch. Work flows upward through pull requests: phase branches merge into the plan branch, and the plan branch eventually merges into the main development branch. This layered approach ensures that incomplete plan work never contaminates the main codebase.

---

## Step 3: Team Assembly

Doplan uses an **Agent Teams** architecture — it creates a virtual team where the doplan orchestrator acts as team lead and spawns specialized teammate agents to execute each phase.

### What Happens

1. A named team is created (e.g., `doplan-customer`)
2. Tasks are registered for each phase with sequential dependencies (Phase 2 cannot start until Phase 1 completes)
3. A progress tracking file is initialized for crash recovery

### Why Agent Teams?

Early versions of this system used child processes to execute phases, which suffered from cold-start overhead — each phase would take extra time spinning up connections to external services. The Agent Teams architecture solved this by allowing teammates to inherit the parent session's connections, dramatically reducing overhead.

---

## Step 4: Phase Execution Loop

This is the heart of the system. For each phase, doplan spawns a teammate agent that executes the complete phase workflow.

### What Happens Per Phase

The orchestrator spawns a single teammate with all the context needed to execute the phase independently. The teammate then runs through the full phase workflow and reports back with a structured result.

### Retry Logic

If a phase fails, the system retries automatically (up to 3 attempts by default). However, if the same task fails twice in a row, the system recognizes a persistent problem and aborts rather than wasting time on repeated failures. This balances resilience with efficiency.

### Result Handling

After each phase completes:
- **On success:** The phase's pull request is merged into the plan branch, and the plan branch is synced with the new changes
- **On failure:** The error is logged, the failed teammate is shut down, and a retry is attempted with a fresh teammate
- **Progress is saved** after every phase, enabling resume from any point

---

## Step 5: Phase Execution (dophase) — The Inner Workflow

Each phase execution is a self-contained workflow with 9 internal stages. This is where the actual development work happens.

### Stage 0: Environment Validation

The system confirms it can connect to the Obsidian knowledge base where plans are stored. If the connection is unavailable, execution halts immediately.

### Stage 1: Argument Parsing

All configuration parameters (plan name, phase number, target branches, merge policy) are validated. The system includes intelligent error detection — for example, if a plan name looks like a branch name, it warns the user about a likely mistake.

### Stage 2: Phase Content Retrieval

A specialized **Phase Getter** agent reads the specific phase from the plan file. It's smart enough to handle partially-completed phases — if some tasks were already done (from a previous run), it returns only the remaining incomplete tasks.

### Stage 3: Pre-flight and Branch Setup

A **Pre-flight** agent handles all the Git preparation work:

1. Validates the base branch for safety
2. Pulls the latest changes to ensure the workspace is current
3. Creates the phase branch from the plan branch
4. Switches the workspace to the new branch

### Stage 4: Task Execution Loop

For each task in the phase, a pipeline of specialized agents runs in sequence:

1. **Task Getter** — Reads the full task requirements from the plan file, providing complete context to the executor
2. **Task Executor** — Implements the code changes required by the task
3. **Learning Recorder** — Captures observations about the implementation for future reference
4. **Tester** — Creates and runs automated tests to validate the implementation
5. **Git Flow** — Commits the changes with a structured commit message
6. **Plan Updater** — Marks the task complete in the plan file with a timestamp and result summary

This pipeline runs for every task, ensuring consistent quality and documentation regardless of task complexity.

### Stage 5: Pull Request Creation

After all tasks in a phase complete:

1. A **PR Creator** agent opens a pull request from the phase branch to the plan branch
2. The PR is populated with a structured description summarizing all tasks completed
3. Updates the workflow tracking file with the PR number
4. Marks the phase as complete in the progress tracker

A **protected branch guard** prevents auto-merging into sensitive branches (main, dev, staging) regardless of configuration, ensuring human oversight where it matters most.

### Stage 6: Automated Code Review

A **Code Reviewer** agent (running on the most capable AI model available) performs a thorough review of the pull request:

1. Reads the full diff of all changes
2. Analyzes code quality, correctness, and adherence to best practices
3. Consults framework documentation for accuracy
4. Posts a detailed review as a comment on the pull request
5. Categorizes issues by severity (Critical, High, Medium, Low)

Issues are formatted with structured markers that enable automated extraction in the next stage.

### Stage 6.5: Auto-fix PR Issues

After the code review, the system automatically attempts to resolve identified issues:

1. A script extracts all Critical, High, and Medium severity issues from the review
2. For each issue, a specialized resolution workflow is invoked
3. The fix is committed and the resolution is documented in the plan
4. A summary reports how many issues were resolved vs. remaining

This creates a rapid feedback loop: code is written, reviewed, and improved — all within a single automated pass.

### Stage 6.6: Merge Gate Evaluation

Before merging, a quality gate evaluates whether the code meets the minimum standard:

- Were all review issues successfully resolved?
- Is the target branch safe for auto-merge?
- Did the code review even complete successfully?

If all gates pass, the PR is merged automatically. If any gate fails, the PR is left open for manual review. This fail-safe design ensures that questionable code never merges without human eyes.

### Stage 7: Results Summary

A concise summary is displayed:

```
Phase 3 complete: 5 tasks | PR #142 → github.com/.../pull/142
Review: 0 blocking, 2 warnings
Auto-fix: 2 resolved, 0 failed
Merge: merged
```

### Stage 8: Session Handoff

A **Session Handoff Writer** agent creates a context document that preserves the full state of the work — what was accomplished, what decisions were made, any failures encountered, and recommended next steps. This document is stored in Obsidian and can be loaded in a future session to resume work with full context, even across different conversations.

---

## Step 6: Plan Completion

After all phases have executed successfully:

1. A final summary pull request is created from the plan branch to the main development branch
2. This final PR is **never auto-merged** — it always requires human review, as it represents the cumulative output of the entire plan
3. The team is disbanded and tracking files are cleaned up
4. An optional audio notification announces completion

---

## Safety and Reliability Features

### Crash Recovery at Every Level

- **Plan level:** Progress is tracked across phases in a persistent file. If a session is interrupted, doplan resumes from the last incomplete phase
- **Phase level:** Progress is tracked across tasks within each phase. If a phase is interrupted, it resumes from the last incomplete task
- **Cross-verification:** Before each phase, the system compares its progress records against the actual state of the plan file in Obsidian, detecting and resolving any discrepancies

### Protected Branch Guardrails

The system maintains a strict list of protected branches (main, master, dev, staging) that can never be auto-merged into. This applies regardless of user configuration, providing a hardware-like safety interlock.

### Continuous Learning

Every task execution feeds observations into a learning system. Over time, this produces:

- **Instincts** — Quick heuristic guidance (e.g., "this project uses a specific testing pattern")
- **Skills** — Detailed procedural knowledge for recurring task types

Future executions consult these learned patterns, creating a feedback loop that improves quality with each plan executed.

### Metrics and Observability

Pipeline performance is tracked automatically via Claude Code hooks at the end of each execution — capturing timing, throughput, task success rates, and phase durations. This data feeds into the metrics dashboard at [reidcardwell.ai](https://reidcardwell.ai), providing an empirical record of pipeline performance over time.

### Event Tracking

A system of lifecycle hooks fires at key moments throughout execution:

| Event | What It Tracks |
|-------|---------------|
| Before each agent runs | Records pending task in progress file |
| After each agent completes | Updates task status (success/failure) |
| File reads and writes | Tracks which files were consulted and modified |
| Session start/stop | Initializes and finalizes session metadata |
| Before session ends | Validates that the workflow completed properly |

---

## Key Design Decisions

### Agent Teams vs. Child Processes

The original architecture launched each phase as an independent child process. This worked but was slow — each process needed to establish fresh connections to external services. The Agent Teams refactor replaced child processes with teammates that share the parent session's connections, dramatically reducing overhead.

### Separation of Concerns in the Task Pipeline

Rather than having one agent implement and test code, the pipeline uses dedicated specialists. Generalist agents tend to cut corners — skipping edge cases in tests, or writing tests that don't actually validate behavior. By enforcing separation, each agent focuses on doing its job well.

### Two-Tier Orchestration

The doplan/dophase split allows each tier to be used independently. A user can run `dophase` directly for a single phase when they don't need full plan automation. This modular design means the system is useful at multiple scales — from a single phase to a multi-phase plan with dozens of tasks.

### Fail-Safe Merge Policy

Auto-merge is opt-in and blocked by multiple safety gates. The default behavior is always to create a PR and wait for human review. Even when auto-merge is enabled, protected branches override the setting. This "safe by default" philosophy prevents accidental deployment of unreviewed code.

---

## Summary of Specialized Agents

| Agent | Role | Analogy |
|-------|------|---------|
| Phase Getter | Reads phase requirements from the plan | Project manager reading the spec |
| Pre-flight | Validates environment and creates workspace | DevOps engineer setting up infrastructure |
| Task Getter | Retrieves individual task details | Developer reading their ticket |
| Task Executor | Writes the code implementation | Senior developer coding |
| Learning Recorder | Records patterns for future improvement | Team retrospective notes |
| Tester | Creates and runs automated tests | QA engineer |
| Git Flow | Commits and pushes code changes | DevOps handling version control |
| Plan Updater | Marks tasks complete with audit trail | Project manager updating the board |
| PR Creator | Opens pull requests with tracking | Developer submitting for review |
| Code Reviewer | Performs automated code review | Senior engineer doing code review |
| Session Handoff Writer | Preserves context for future sessions | Team member writing handoff notes |
| Vault Validator | Confirms Obsidian connectivity | IT checking system availability |
| Context Generator | Builds project documentation for agents | Technical writer creating onboarding docs |

---

## Metrics and Scale

- **Agents orchestrated per phase:** Up to 13 specialized agents
- **Agents per plan:** 13 agents × N phases (e.g., a 5-phase plan orchestrates ~65 agent invocations)
- **Automated steps per task:** 6 (read → implement → learn → test → commit → update)
- **Safety checkpoints per phase:** 4+ (branch validation, protected branch guard, merge gate, completion validation)
- **Recovery mechanisms:** 3 tiers (plan-level, phase-level, cross-verification)
- **Metrics captured:** Execution time, throughput, task success rates, phase durations — tracked automatically and published at [reidcardwell.ai](https://reidcardwell.ai)

---

## About

Built and maintained by [Reid Cardwell](https://reidcardwell.ai) — software developer and AI pipeline architect based in Graham, NC.
