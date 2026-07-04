# autobot — Autonomous PM orchestrator skill for Claude Code

`autobot` is a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that
takes a natural-language task (`/autobot PROMPT`) and drives it from intent to complete,
autonomous execution. It runs an interactive preparation phase (analysis → clarifications
→ brainstorming → plan), then builds a `LOOP_PROMPT` from `LOOP_PROMPT_TEMPLATE.md` and
launches the `/loop` skill for autonomous TDD execution.

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed and authenticated
- The `/loop` skill available (built into Claude Code) — used for autonomous execution

## Contents

```
autobot/
  SKILL.md                    # the skill instructions (Phase A + Phase B)
  assets/
    LOOP_PROMPT_TEMPLATE.md   # the loop template, source for building LOOP_PROMPT
```

> **Important:** always copy the `assets/` directory too. Without
> `assets/LOOP_PROMPT_TEMPLATE.md`, Phase B cannot build `LOOP_PROMPT` unless a copy of
> the template exists in the project root.

## Installation

Clone this repository, then copy the skill:

### Option 1 — Global (all projects)

```bash
# Windows (Git Bash)
mkdir -p "$USERPROFILE/.claude/skills/autobot/assets"
cp autobot/SKILL.md "$USERPROFILE/.claude/skills/autobot/SKILL.md"
cp autobot/assets/LOOP_PROMPT_TEMPLATE.md "$USERPROFILE/.claude/skills/autobot/assets/LOOP_PROMPT_TEMPLATE.md"

# macOS / Linux
mkdir -p ~/.claude/skills/autobot/assets
cp autobot/SKILL.md ~/.claude/skills/autobot/SKILL.md
cp autobot/assets/LOOP_PROMPT_TEMPLATE.md ~/.claude/skills/autobot/assets/LOOP_PROMPT_TEMPLATE.md
```

### Option 2 — Per project

```bash
mkdir -p .claude/skills/autobot/assets
cp autobot/SKILL.md .claude/skills/autobot/SKILL.md
cp autobot/assets/LOOP_PROMPT_TEMPLATE.md .claude/skills/autobot/assets/LOOP_PROMPT_TEMPLATE.md
```

> After installing, **restart Claude Code** (or start a new session). Skills are
> discovered at session startup; a freshly installed skill does not appear in the
> current session.

## Usage

Open Claude Code in your project directory and type:

```
/autobot <your task description>
```

Examples:
```
/autobot build a Python script that validates CSV files against a schema
/autobot add JWT authentication to the existing API, with tests
```

Or in natural language:
- "execute this autonomously: ..."
- "build me X end to end"
- "run a loop that implements the whole plan"

## What it does

1. **Phase A — Preparation (interactive):**
   - Stage 1: analyzes the prompt and asks clarifying questions if needed (`AskUserQuestion`).
   - Stage 2: brainstorming → a plan structured into milestones and tasks, with
     acceptance criteria (verifiable status format `- [ ]` / `- [~]` / `- [x]` / `- [!]`).
   - Stage 3: asks the new clarifications that surfaced during planning.
   - Stage 4: writes the plan to `EXECUTION_PLAN.md` (in the current directory) — asking
     first if one already exists (resume or archive) — then starts automatically.
2. **Phase B — Execution (autonomous):**
   - Stage 5a: fills in the template preamble → writes `LOOP_PROMPT.md`.
   - Stage 5b: launches `/loop` with `LOOP_PROMPT`, "one task per cycle" mode, until a
     terminal state (objective reached or blocked).

## Files generated at runtime

In the current working directory:
```
EXECUTION_PLAN.md  # the execution plan (source of truth for status)
LOOP_PROMPT.md     # the loop prompt built from the template
vault/             # persistent memory vault (initialized by the loop at STEP 0;
                   # includes vault/ENV.md with the discovered test/build commands)
```

## Updating

If you modify the skill's source, re-run the copy steps to propagate the change to the
active installation (global or per project).

## Uninstalling

```bash
# Global
rm -rf ~/.claude/skills/autobot

# Or per project
rm -rf .claude/skills/autobot
```

## License

[MIT](LICENSE)
