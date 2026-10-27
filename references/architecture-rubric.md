# Architecture rubric

Distilled from Anthropic's docs (Building effective agents, the multi-agent research system, harness design, Claude Code best practices) and community practice as of late 2026.

## Core principles

1. **Start with the simplest thing that can work.** Climb the complexity ladder one rung at a time:
   - a single model call;
   - a chain of steps with programmatic gates;
   - routing;
   - parallelization;
   - orchestrator-workers;
   - evaluator-optimizer;
   - autonomous agent.
2. **Verification beats agent count.** The value sits in three things:
   - a pass/fail check the agent runs itself;
   - an independent reviewer in a clean context;
   - deterministic constraints (hooks, deny rules, sandbox).
3. **Parallelize reads, keep writes single-threaded.** Multiple writing agents are fine only if they touch different files (worktrees) and each has its own check.
4. **Multi-agent mostly buys token budget.** In Anthropic's system, 15× the tokens gave +90% on broad research. For coupled tasks (most coding) it's a bad trade.
5. **A subagent means context isolation and constraints, not role-play.** It gives a fresh view or parallelism, not a smarter answer.
6. **Every component encodes an assumption about what the model can't do on its own.** Remove components that don't pay off, and revisit the harness when the model changes.

## Decision matrix

| Task trait | Architecture |
|---|---|
| Coupled changes in one place (feature, bug, refactor, document) | Single agent + check + reviewer (read-only subagent or a second session) |
| Broad research, many independent sources, more information than fits in one window | Orchestrator-workers: lead (Opus/inherit) + 2–5 researchers (Sonnet/Haiku, low–medium effort); 1–2k-token summaries; final result to a file |
| Many independent units of the same kind (files, documents, records) | Fan-out: `/batch`, `claude -p` in a loop, dynamic workflow or Batch API; trial run on 3 units first |
| Many independently checkable subtasks with an oracle (tests, reference output) | Peers with no orchestrator, coordinating via git, lock files and tests |
| Review from several angles, or competing hypotheses | Parallel reviewers or agent teams (experimental, 3–5 members) |
| Untrusted external content + dangerous actions | Quarantine: unprivileged readers → a privileged executor that acts only on their summaries |
| Long task (hours) | Planner → generator → evaluator, handing off via files (SPEC, PLAN, progress); the evaluator is a separate agent with hard thresholds |
| No human in the loop (CI, schedule) | Headless `claude -p` / GitHub Action: `dontAsk` + `--allowedTools`, `--max-turns`, `--max-budget-usd`, protected branch |

Delegation scale for research tasks:
- simple fact: 1 agent, 3–10 tool calls;
- comparison: 2–4 subagents;
- complex research: more than 10 subagents, only when justified.

Current models delegate eagerly. Set limits explicitly.

## When to add each component

| Component | Add when |
|---|---|
| Line in CLAUDE.md | Claude can't learn it from the code, or it made the same mistake twice |
| `.claude/rules/` with `paths:` | The rule applies to only part of the codebase |
| Skill | The same prompt or procedure is needed a third time; a slash command is needed |
| Subagent | Side work floods the main context; a restricted tool set is needed; a fresh view is needed (review) |
| Hook | Something must happen **every time**, no exceptions |
| Deny rule | Access must be forbidden in every mode (secrets, prod) |
| MCP server | An external service is needed and has no good CLI |
| Plugin | The same setup needs to move to other projects |
| Agent teams | Members need to discuss work with each other, not just report to the lead |

## Models and effort by role

- **Orchestrator / planner / final synthesis:** strong model (inherit/Opus), effort medium–high.
- **Researchers, search, extraction, classification:** Sonnet or Haiku, effort low–medium.
- **Writing worker:** inherit or Sonnet, depending on difficulty.
- **Reviewer:** Sonnet or inherit, effort high, read-only tools. The instruction "skip style, correctness only" is mandatory.
- A weak lead with strong workers loses. If you economize, do it on the workers.

## Guarding against known failures (MAST and Anthropic's experience)

Every subagent brief includes:
- goal;
- output format;
- task boundaries (what not to do);
- sources and tools;
- **completion condition**;
- budget (`maxTurns`, number of calls).

Before "done", require independent verification of the overall goal (the largest measured gain, +15.6%).

Against duplicated work: explicitly split the zones between parallel agents.

Against requirements lost during compaction: keep requirements in a file (SPEC.md), not only in chat.

## Anti-patterns

- "Architect / coder / tester" personas with no real tool restrictions.
- Dozens of rules in CLAUDE.md, ALL CAPS, "think hard". Instead: hooks for hard rules, `/effort` for depth.
- Five reviewers on a routine edit.
- A Ralph loop with no iteration limit and no sandbox.
- MCP servers "just in case": their schemas eat context.
- Third-party skills and plugins from unknown sources installed without reading their code.
