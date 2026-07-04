---
name: autobot
description: >
  Autonomous orchestrator that takes a natural-language task (/autobot PROMPT) and drives
  it from intent to full execution. Use this skill whenever the user types /autobot
  followed by a request, or asks to "execute this autonomously", "build X end to end",
  "run a loop that implements the whole plan". The skill analyzes the prompt, asks
  clarifying questions if needed, brainstorms, writes an execution plan structured into
  milestones and tasks, then builds a LOOP_PROMPT from LOOP_PROMPT_TEMPLATE.md and
  launches the /loop skill for autonomous TDD execution.
---

# Skill: /autobot — Autonomous PM orchestrator with a TDD loop

## Invocation

`/autobot PROMPT` — where `PROMPT` is the user's task in natural language.
If `PROMPT` is missing or empty, ask the user for it before anything else; do not fail.

## Principle: two phases

1. **Phase A — Preparation (interactive, in the current session).** You analyze the
   prompt, ask clarifying questions ONLY here, brainstorm and write the plan. All
   questions to the user are asked in this phase.
2. **Phase B — Execution (autonomous).** You build `LOOP_PROMPT` from the template and
   hand it to the `/loop` skill. In this phase you NO longer ask questions; decisions are
   made autonomously, aligned with the plan (rule R5 in the template).

Golden rule: **never interrupt the user in Phase B.** If you realize in Phase A that
something is ambiguous, ask then — the goal is a final plan that requires no further
decisions.

## Phase A — Preparation

### Stage 1 — Analysis + initial clarifications
Analyze `PROMPT`: what exactly is being asked, what is ambiguous, which assumptions are
risky, what constraints (including security) might exist.
- If you find ambiguities that materially change what gets implemented → ask questions
  via the `AskUserQuestion` tool (structured, multiple choice when possible).
- If the prompt is clear → continue without interrupting unnecessarily.
Do not ask questions for form's sake; ask only what you cannot decide autonomously
without risking a departure from the user's intent.

### Stage 2 — Brainstorming + plan
Determine the best execution path for the clarified prompt. Produce a plan structured
into **milestones → tasks**, each task with **acceptance criteria**. Scale depth with
complexity: simple prompt → 1 milestone, a few tasks; complex prompt → multiple
milestones. Every task uses the R6 status markers:
`- [ ]` not started, `- [~]` in progress, `- [x]` done, `- [!]` blocked.

### Stage 3 — Clarifications arising from brainstorming
If planning surfaces new decisions that depend on the user and cannot be taken
autonomously without risk → ask now (again via `AskUserQuestion`). Objective: after
Phase A, the plan **requires no further decisions** from the user.

### Stage 4 — Write the plan
If `EXECUTION_PLAN.md` already exists in the cwd, ask the user (via `AskUserQuestion`,
still in Phase A): **resume** the existing plan, or **archive** it (rename with a date,
e.g. `EXECUTION_PLAN.<date>.md`) and write a new one. Never overwrite silently.

Write the plan to `EXECUTION_PLAN.md` in the current working directory. Minimal format:

~~~markdown
# Execution plan — <PROJECT_NAME>

## Objective
<SESSION_OBJECTIVE>

## Established decisions and constraints
- <the user's clarifications from Stages 1/3 and the brainstorming decisions;
  the loop must not contradict them (rule R5)>

## Milestone 1: <title>
- [ ] <task> — Acceptance criteria: <verifiable criteria>
- [ ] <task> — Acceptance criteria: <verifiable criteria>
~~~

After writing, do **NOT** ask for confirmation — move automatically to Phase B.

## Phase B — Autonomous execution

### Stage 5a — Build LOOP_PROMPT
Read the template: first `LOOP_PROMPT_TEMPLATE.md` in the cwd root; if missing, use
`assets/LOOP_PROMPT_TEMPLATE.md` from this skill. Fill in **only SECTION 1 — Preamble**,
with these mappings:

| Template field | Value |
|----------------|-------|
| `{{PROJECT_NAME}}` | short name derived from the prompt |
| `{{PROJECT_DESCRIPTION}}` | context from the clarified prompt |
| `{{SESSION_OBJECTIVE}}` | the objective from the prompt |
| `{{TASK_SPECS}}` | the concrete requirements from the prompt + brainstorming |
| `{{SECURITY_CONSTRAINTS}}` | from the analysis; "N/A" if not applicable |
| `{{SUBAGENT_MECHANISM}}` | "the Agent tool with subagent_type: general-purpose, one role prompt per call" |
| `{{TEST_COMMAND}}`, `{{BUILD_LINT_COMMAND}}`, `{{TEST_TYPES}}`, `{{COVERAGE_THRESHOLD}}` | leave a "to be discovered" marker; add the note: "STEP 0 (bootstrap) discovers these commands from the codebase and records them in `vault/ENV.md`; the GATE (STEP 3) uses the discovered ones. If the project has no tests, mark N/A and base the GATE on the acceptance criteria from the plan." |
| `{{GIT_CONVENTION}}` | "1 commit per task, message `[task-id] description`" |
| `{{PROCESSING_MODE}}` | "one task per cycle" |
| `{{MAX_ITERATIONS}}` | `5` |

Copy **SECTION 2 (Rules) and SECTION 3 (Decision diagram) UNCHANGED**. Write the result
to `LOOP_PROMPT.md` in the cwd. Verify: no `{{...}}` marker remains unfilled (except the
environment ones intentionally left as "to be discovered").

### Stage 5b — Launch the loop
Invoke the `/loop` skill (via the `Skill` tool) with the contents of `LOOP_PROMPT.md`, in
self-paced/autonomous mode (no interval). Each wake-up resumes the state machine
(STEPS 0–8). With `PROCESSING_MODE = "one task per cycle"`, one wake-up = exactly one
task driven to `done` (STEP 7 → resume from STEP 1) + a short R8 report, then `/loop`
wakes up for the next task. The loop stops on its own at a terminal state: STEP 3 →
objective reached, or STEP 8 → blocked (no further replanning). After launching, report
to the user that the loop has started and which files were created (`EXECUTION_PLAN.md`,
`LOOP_PROMPT.md`).

## Edge cases
- **Non-software prompt / no tests** (e.g. "write documentation"): the GATE relies on
  the acceptance criteria from the plan; test fields = "N/A"; TDD applies only where it
  makes sense.
- **Blocked loop (STEP 8)**: `/loop` stops; the blocker is in the report + vault; the
  user intervenes and can relaunch.
- **Empty PROMPT**: ask for the prompt, do not fail.
- **`LOOP_PROMPT_TEMPLATE.md` missing from the root**: use the copy from `assets/`.
