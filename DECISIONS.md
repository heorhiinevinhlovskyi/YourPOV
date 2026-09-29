# Decision log

Record every significant decision here, newest at the bottom. Never delete an entry; if a decision changes, add a new one that supersedes it.

## Template

```
## NNN — Title
- Date: YYYY-MM-DD
- Status: Proposed | Accepted | Superseded by NNN
- Decided by:
- Context: what problem or question prompted this?
- Decision: what we chose.
- Alternatives considered: what else we looked at and why not.
- Consequences: what this makes easier or harder.
```

---

## 001 — Pull-request workflow instead of enforced branch protection
- Date: 2026-09-28
- Status: Accepted
- Decided by: George
- Context: The repo is private on a personal GitHub account. GitHub does not enforce rulesets or branch protection there without a paid organization plan.
- Decision: The team agrees never to push directly to `main`. All changes go through a branch and a pull request with at least 1 approval. The rule is written in `CLAUDE.md` so Claude Code follows it too. A ruleset is saved in GitHub settings so it starts enforcing if the repo later moves to a paid organization.
- Alternatives considered: Making the repo public (free protection, but exposes the project); moving to a paid GitHub Team organization (cost not justified yet).
- Consequences: Protection depends on discipline. Revisit when the team grows or starts paying for GitHub.
