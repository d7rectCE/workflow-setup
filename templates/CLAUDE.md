<!-- Template. Keep under 200 lines. Only what Claude can't learn from the code. Delete sections that don't apply. -->

# Commands
- Install: <command>
- Run one check/test: <command>
- Full check: <command>
- Lint / format: <command>

# Definition of done
- <pass/fail check that must be green before reporting done>
- Report the commands you ran and their output.

# Workflow
- Requirements live in SPEC.md; the plan and progress live in PLAN.md. Update PLAN.md as phases finish.
- For non-trivial work: spec → plan (plan mode) → implement phase by phase → review with the `reviewer` agent.

# Conventions
- <project-specific rule Claude would otherwise get wrong>

# Gotchas
- <non-obvious trap: env vars, flaky tests, legacy code not to copy>

# Decisions made outside the code
- <decision + date>
