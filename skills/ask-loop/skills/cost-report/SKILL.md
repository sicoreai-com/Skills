---
name: cost-report
description: >
  Summarize token cost across all GIT issues in the project: totals and
  breakdowns per ticket type (Feature/Bug/Task) and per handling stage
  (Analysis, Implementation, Retrospective, CI). Read-only; writes
  COST-REPORT.md.
user_invocable: true
---

# Cost-Report Skill

You are the accounting stage of the loop. You aggregate the per-issue token
usage fields the other skills maintain into one report a human can read to
decide where the loop is spending.

Follow `.claude/skills/shared/github-project-ops.md` for the project read
(§2). This skill is **read-only** on the project — it edits no issue and no
field.

## Inputs

- ALL project items (every type, every status). If the project has no items,
  cost-report is finished — exit with a one-line note.
- Per item: Type, Status, Title, and the four stage fields —
  `Analysis Token Usage`, `Implementation Token Usage`,
  `Retrospective Token Usage`, `CI Token Usage` — plus `Estimate Token` and
  `Budget Token` when set. Treat empty fields as 0.

## Output — `COST-REPORT.md` at repo root (overwrite each run)

```markdown
# Cost Report
Generated: <ISO timestamp>
Project: <org> / <project>
Issues counted: N

## Totals
| Stage | Tokens | % of total |
|-------|--------|-----------|
| Analysis | … | … |
| Implementation | … | … |
| Retrospective | … | … |
| CI | … | … |
| **Total** | … | 100% |

## Per Ticket Type
| Type | Issues | Analysis | Implementation | Retrospective | CI | Total | Avg/issue |
|------|--------|----------|----------------|---------------|----|-------|-----------|
| Feature | … |
| Bug | … |
| Task | … |

## Type × Stage highlights
- Costliest stage overall and why (which issues drive it)

## Top 10 costliest issues
| Issue | Type | Status | Total tokens | Dominant stage |

## Budget check
- Issues where total usage exceeds Budget Token (if set): list with overrun
- Issues where usage exceeds Estimate Token: count + worst offenders

## Observations
- 2–4 bullets: trends worth human attention (e.g. "Rejected→retry cycles
  doubled Bug implementation cost", "CI-Sweeper rounds dominate Task cost")
```

## Rules

- Numbers come only from the project fields — never estimate or back-fill
  missing data; report field coverage instead (e.g. "12/40 issues have no
  usage recorded") so gaps are visible.
- Keep the report deterministic: same board state → same numbers.
- End with a one-line run summary: total tokens, costliest type, costliest
  stage.
