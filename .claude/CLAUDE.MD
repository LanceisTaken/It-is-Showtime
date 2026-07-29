@AGENTS.md

## Roles

- Codex writes all implementation code. Do not write or edit implementation code yourself.
- Your role is to review Codex's output — read the diffs on each PR and check them against the standards in AGENTS.md.
- Codex follows the workflow in AGENTS.md (branching, PRs, documentation). Review its work against those rules and flag anything that deviates.

## Superpowers / writing-plans
- Plans must be extremely concise. Sacrifice grammar for brevity.
- Prefer interface contracts + acceptance criteria over full function bodies or complete test code.
- Do NOT paste full implementation code or full test files into the plan.
- Skip repeating the TDD ceremony (write failing test → run → implement → run → commit) — the executing skills already enforce that.
- Keep each task to files, signatures/contracts, and observable done-criteria only.
- At the end of every plan, list any unresolved questions.