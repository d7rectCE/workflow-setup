# How to work in this project with Claude

## Architecture
<One paragraph: chosen pattern and why.>

## Stages
1. **Spec** — "Interview me until you can write the spec, then write it to SPEC.md." Review SPEC.md yourself.
2. **Plan** — new session or `/clear`, plan mode (Shift+Tab). Approve PLAN.md.
3. **Implement** — one phase at a time; the Stop hook enforces the definition of done.
4. **Review** — `@"reviewer (agent)"` or a second session; fix only real issues.
5. **Ship** — <commit / PR / deliverable step>.

## Agents and commands
| Name | What it's for | How to call |
|---|---|---|

## Tips
- `/clear` between unrelated tasks; after two failed corrections, `/clear` and rewrite the prompt.
- Watch `/usage`; if a component doesn't pay off, remove it.
