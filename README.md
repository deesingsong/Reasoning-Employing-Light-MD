# Multi-Agent Codex Build Protocol

This project uses a cost-conscious multi-agent workflow. The goal is to keep the large-model windows short and purposeful while delegating long implementation context to Luna sub-agents.

## Agent roles

Use these roles consistently:

- **Planner:** GPT-5.6-Sol Max. Converts the user request into an implementation plan, architecture, acceptance criteria, risks, and a sequence of delegated tasks.
- **Orchestrator:** GPT-5.6-Sol Max. Owns the build from the plan onward. It coordinates sub-agents, maintains state, reviews results, and decides what happens next.
- **Worker:** Luna 5.6 Light. Performs focused implementation, investigation, testing, documentation, or review tasks in a clean context window.
- **Bulldozer:** A fresh GPT-5.6-Sol Max instance. Unblocks stalled work, resolves contradictory worker results, repairs integration failures, or takes over when the orchestrator is stuck.

## Required startup sequence

### 1. Plan before editing

The Planner must inspect the repository and produce a concise plan before making substantial changes. The plan must include:

- the intended user-visible outcome;
- relevant existing files and constraints;
- architecture and data-flow decisions;
- task decomposition into independently verifiable units;
- acceptance criteria and test commands;
- known risks and fallback options.

The Planner should avoid implementing the whole project. Its primary output is a durable handoff for the Orchestrator.

### 2. Start the Orchestrator

Give the Orchestrator the Planner's handoff and instruct it to own the complete build. The Orchestrator must not repeatedly reconstruct the entire project history from chat. It should use the repository handoff files as the source of truth.

At the beginning of its run, the Orchestrator must:

1. read `AGENTS.md`;
2. read `docs/agent/PLAN.md` if it exists;
3. read `docs/agent/STATE.md` if it exists;
4. inspect the current git diff and repository status;
5. update `docs/agent/STATE.md` before delegating work.

### 3. Delegate to Luna with clean contexts

Each Luna call must receive one narrowly scoped task. Do not send the entire conversation or an unbounded project history when a repository handoff is sufficient.

Every worker handoff must specify:

- task ID and objective;
- files or directories in scope;
- files that must not be changed;
- relevant constraints and interfaces;
- expected deliverables;
- exact validation commands;
- what to report if blocked;
- the instruction to inspect the current repository state before editing.

Use `docs/agent/tasks/TASK-<id>.md` for the task brief and `docs/agent/results/TASK-<id>.md` for the worker's result. The worker should update only the files necessary for its task and should leave a concise result report.

The Orchestrator should prefer several small, independent worker tasks over one broad task. Parallelize only when tasks do not touch the same files or depend on each other's unfinished work.

## Handoff templates

### Planner handoff: `docs/agent/PLAN.md`

```md
# Build Plan

## Objective
<!-- One clear sentence describing the desired outcome. -->

## Current repository state
<!-- Existing architecture, relevant files, constraints, and detected issues. -->

## Proposed design
<!-- Components, interfaces, data flow, and important decisions. -->

## Work breakdown
<!-- Ordered task IDs with dependencies. -->

## Acceptance criteria
<!-- Observable conditions that must be true when complete. -->

## Validation
<!-- Exact lint, typecheck, test, build, and run commands. -->

## Risks and fallback options
<!-- Important uncertainties and safe alternatives. -->
```

### Orchestrator state: `docs/agent/STATE.md`

```md
# Agent Build State

## Current objective

## Current phase

## Completed tasks

## Active tasks

## Blocked tasks

## Decisions

## Validation status

## Next action

## Last updated
```

### Luna task brief: `docs/agent/tasks/TASK-<id>.md`

```md
# Task <id>

## Objective

## Scope

### May edit

### Must not edit

## Context
<!-- Only the facts needed for this task. Read the repository for details. -->

## Deliverables

## Validation commands

## Completion requirements

1. Inspect the current repository state before editing.
2. Make the smallest coherent change that satisfies the objective.
3. Run the listed validation commands, or explain precisely why one cannot run.
4. Write a concise result to `docs/agent/results/TASK-<id>.md`.
5. Report changed files, tests run, failures, and any follow-up needed.
```

## Orchestrator operating rules

The Orchestrator is responsible for integration quality, not merely task dispatch.

- Keep `STATE.md` current after every meaningful milestone.
- Before assigning a task, check for overlapping active work.
- After a worker finishes, inspect its diff and result report before accepting it.
- Run integration tests after combining related tasks.
- Do not trust a worker's claim that tests pass without checking the relevant output when practical.
- Resolve ambiguity from repository evidence and the user's objective; do not invent requirements silently.
- Preserve unrelated user changes. Never reset, discard, or overwrite them without explicit authorization.
- Keep task briefs and result reports concise. Durable repository state is preferable to repeating context in prompts.
- Stop and ask the user when a decision changes scope, security posture, data handling, destructive behavior, or public API compatibility.

## Fresh-context rule

When the Orchestrator is nearing context saturation, it must checkpoint before continuing:

1. update `docs/agent/STATE.md`;
2. record unresolved questions and exact next steps;
3. record validation results and relevant file paths;
4. start a fresh Orchestrator context with `AGENTS.md`, `PLAN.md`, `STATE.md`, and the current repository state.

Do not paste a large conversation transcript into the fresh context when the same information is already represented in the handoff files.

## Bulldozer escalation

Start a fresh GPT-5.6-Sol Max Bulldozer when any of the following occurs:

- the Orchestrator is unable to make progress after two focused attempts;
- a worker is blocked by an unclear interface or contradictory repository state;
- integration tests fail and the cause is not isolated;
- multiple worker results conflict;
- the build has drifted from the acceptance criteria;
- a long-running worker appears stalled or repeatedly returns incomplete work.

The Bulldozer must receive only a compact recovery handoff containing:

- the original objective and acceptance criteria;
- the current `STATE.md`;
- the relevant task briefs and result reports;
- the exact failing command and output;
- the smallest decision needed to unblock progress.

The Bulldozer must first diagnose, then either repair the issue directly or create a replacement task for Luna. It must not broadly rewrite the project merely because the existing implementation is unfamiliar.

After recovery, it must update `STATE.md` with the diagnosis, action taken, and recommended next step before returning control to the Orchestrator.

## Completion gate

The build is complete only when:

- every acceptance criterion is satisfied;
- relevant tests, linting, type checks, and builds pass;
- no active task remains unexplained;
- the final diff has been reviewed for accidental changes;
- `STATE.md` records the final validation and any known limitations;
- the Orchestrator provides a concise final summary with changed files and verification results.

## Cost and context discipline

- Keep large-model prompts focused on decisions, coordination, review, and recovery.
- Keep Luna prompts focused on bounded execution with repository-local context.
- Prefer durable files over repeated prose.
- Avoid asking any agent to summarize information that is already available in a handoff file.
- Do not delegate trivial one-line edits if the Orchestrator can safely perform them while already working in the same context.
- Do not split tightly coupled edits into separate workers if doing so creates repeated integration overhead.

## Safety and authorization

Agents must not expose secrets, commit credentials, weaken authentication, disable security controls, or perform destructive operations without explicit authorization. If a task requires external access, paid services, irreversible migration, data deletion, or a scope change, pause and request confirmation.
