---
name: workflow-setup
description: Builds a Claude workflow tailored to a specific task. Parallel Sonnet researchers search the web for domain literature and methods, existing skills, plugins and MCP servers, environment requirements, and verification approaches; then the coordinator picks an architecture (single agent vs multi-agent) and, after user approval, generates CLAUDE.md, subagents, skills, hooks and settings in the project. Usage - /workflow-setup "task description".
argument-hint: "<task description>"
disable-model-invocation: true
---

# Workflow Setup

User's task: **$ARGUMENTS**

You are the coordinator. Delegate research to Sonnet subagents; you design the architecture and build the configuration yourself. This skill's base directory (`${CLAUDE_SKILL_DIR}`) contains:
- `references/briefs.md` — researcher briefs and note format;
- `references/architecture-rubric.md` — how to choose the architecture;
- `templates/` — config starting points: `PLAN.md`, `CLAUDE.md`, `WORKFLOW.md`, `settings.json`, `agent-reviewer.md`, `agent-researcher.md`.

Talk to the user in the language they wrote the task in. Generated files may be in that language too, except frontmatter keys and code.

Core principle every decision is measured against: the workflow is built around a **verifiable definition of done**, not around the number of agents. Every extra component costs tokens and must pay for itself.

## Phase 0. Intake

1. If `$ARGUMENTS` is empty, ask what task to set up and stop.
2. Inspect the target folder (current working directory):
   - existing `CLAUDE.md`, `AGENTS.md`, `.claude/` (agents, skills, settings.json), `.mcp.json`;
   - git;
   - languages and package managers (package.json, pyproject.toml, etc.);
   - OS;
   - available CLIs: `git`, `node`, `python`, `jq`, `gh`, `uv`, `docker`.

   Write this up briefly as the "environment profile".
3. Ask at most three clarifying questions with AskUserQuestion, and only ones whose answers change the architecture. Typical ones:
   - interactive vs autonomous/CI operation;
   - budget sensitivity;
   - external services the agent needs access to;
   - what counts as the deliverable.

   If the task is clear enough, ask nothing.
4. Pick a short kebab-case slug for the task and create `.workflow-setup/<slug>/research/`. This folder is the only place you write before approval.

## Phase 1. Parallel research (Sonnet)

Read `references/briefs.md`. It has five briefs:
- `domain` — literature and methods of the subject area;
- `ecosystem` — existing skills, plugins, MCP servers, subagents;
- `architecture` — how similar tasks are solved with agents;
- `environment` — tools, dependencies, environment setup;
- `verification` — how to check results, and evals.

How many researchers to launch:
- narrow, well-understood task → 3 (`domain`, `ecosystem`, `verification`; fold environment and architecture questions into them);
- typical task → all 5;
- large task spanning several domains → 5, with `domain` split into 2–3 researchers by subdomain.

Launch them all **in a single message** via the Agent tool with `subagent_type: "general-purpose"` and `model: "sonnet"`. Give each this prompt:

```
You are a researcher helping set up a Claude workflow for a task.
User's task: <task + answers to clarifying questions>
Environment profile: <profile from phase 0>
Your brief: section "<brief name>" in ${CLAUDE_SKILL_DIR}/references/briefs.md (expand to the absolute path) — read it and the "Shared rules" section first.
Save your notes to <absolute path>/.workflow-setup/<slug>/research/<brief name>.md
Return a summary of at most 300 words: key findings, recommendations, gaps.
```

Wait for all researchers to finish. Don't report or guess at anything that isn't in their summaries yet. Don't do research yourself in this phase. If a critical question is left open, you may run **one** extra round of 1–2 researchers.

## Phase 2. Design

1. Read all notes in `research/` and `references/architecture-rubric.md`.
2. Choose the architecture using the rubric and write down why it beats the neighboring options. The default is single agent + verification + clean-context reviewer. Multi-agent is allowed only when the rubric's conditions are met.
3. Write `.workflow-setup/<slug>/PLAN.md` from `templates/PLAN.md`. Fill in:
   - definition of done;
   - architecture and work flow;
   - model and effort per role;
   - files to create or modify (purpose of each);
   - what to install: plugins, MCP servers, packages, CLIs — with source, install command and trust level;
   - checks and hooks;
   - cost estimate;
   - risks;
   - what was deliberately **not** added.
4. Select components by the rubric's triggers, not "just in case". A good plan usually has 1–4 subagents and 0–3 skills. Justify each one beyond that.

## Phase 3. Approval (mandatory)

Show the user a short plan summary: architecture, file list, install list, cost estimate. Ask with AskUserQuestion:
- "Apply everything";
- "Config files only, no installs";
- "Revise the plan" (then apply the edits and ask again).

Install nothing and write nothing outside `.workflow-setup/` without explicit approval. Approval of the plan does not cover actions that aren't in it.

## Phase 4. Build

Use `templates/` as a base, but adapt every file to the task. Leave no empty placeholders.

- **Git.** If the project is a git repo, add `.workflow-setup/` to `.gitignore` unless it's already there.
- **Existing files.** Read them first. Merge into `CLAUDE.md` and `.claude/settings.json`; never overwrite them. Back up any file before changing it, to `.workflow-setup/<slug>/backup/`.
- **CLAUDE.md** — under 200 lines. Only what Claude can't learn from the code: commands, definition of done, gotchas, decisions made outside the code. No architecture description, no ALL-CAPS commands.
- **Subagents** (`.claude/agents/*.md`):
  - minimal `tools`;
  - `model` by role: sonnet/haiku for workers, inherit where depth is needed;
  - explicit completion criteria and output format;
  - reviewers get read-only tools and "skip style";
  - writing agents that run in parallel get `isolation: worktree`.
- **Skills** (`.claude/skills/<name>/SKILL.md`) — for repeated procedures and slash commands. The `description` says when to use the skill.
- **Hooks and permissions** (`.claude/settings.json`):
  - deny rules for secrets;
  - a Stop hook for the definition-of-done check, only if the check is fast (roughly under 30 seconds) and deterministic. A Stop hook runs at the end of every turn, including turns that only answered a question. For slow checks, enforce them through the reviewer agent or a skill instead;
  - PostToolUse formatting, if the project has a formatter.

  Hook commands must work in the shell Claude Code uses on this machine. On native Windows that is usually Git Bash, so POSIX syntax works but tools like `jq`, `node` or `python` may be missing. Use only commands you saw in the environment profile. A blocking hook exits with code 2; exit 1 does not block.
- **Installs.** Only what was approved, one command at a time. Sources: the official Anthropic marketplace, the MCP registry, or repos with a reputation established in the `ecosystem` notes. Show each command before running it. Don't blindly retry a failed install: record it and move on.
- **Work process.** `WORKFLOW.md` in the project root (template `templates/WORKFLOW.md`) — a short guide for the human: how to give tasks, the order of stages, which commands and agents exist.

## Phase 5. Verification

1. Check mechanically:
   - every JSON file parses (with whatever is available: `node`, `python`, `jq`, or PowerShell `ConvertFrom-Json`);
   - every agent and skill has valid YAML frontmatter with `name` and `description`;
   - each hook command runs: execute it once in the current state and confirm it exits 0 when the work is green;
   - CLAUDE.md is under 200 lines.
2. Run one read-only Sonnet reviewer subagent (`model: "sonnet"`). Have it compare the generated files against PLAN.md and the rubric and return only real discrepancies: contradictions, a missing check, redundant components, unsafe permissions.
3. Fix what it finds.

## Phase 6. Report

Tell the user briefly:
- which architecture was chosen and why (1–2 sentences);
- files created and modified, with links;
- what was installed and what wasn't (with reasons);
- how to start: the first command or prompt to kick off work;
- what to measure to see whether the setup pays off: cost (`/usage`), share of tasks passing the check, a set of 20–50 reference tasks for evals.

Separately name anything you couldn't verify, such as a plugin that was found but not installed.
