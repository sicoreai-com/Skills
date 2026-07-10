# Engineering Loop

**Goal**: Autonomously take Feature/Bug issues from Open to Closed —
triage → implement → (human merge) → CI → (human approve) → publish — with
retrospectives that improve the skills whenever a human rejects the loop's
work.

Driven off the GitHub Project named in `loop-config.md`. All skills share the
`gh` recipes in `.claude/skills/shared/github-project-ops.md`.

## Skills (agent) and human gates

| # | Stage | Actor | Skill | Reads (Type/Status) | Writes status |
|---|-------|-------|-------|---------------------|---------------|
| 1 | Triage | agent | `triage` | Feature/Bug: Open | Open→Feedback (ambiguous); fills Priority/Effort; writes IMPLEMENTATION.md |
| 2 | Implementation | agent | `implementation` | IMPLEMENTATION.md queue | Open→Pending merge; opens PRs (code + spec together) |
| 3 | Code review & merge | **human** | — | Feature/Bug: Pending merge | →Merged (Feature/Bug), →Review (Task), or →Rejected + reason in Detailed Description |
| 4 | Retrospective | agent | `retrospective` | Feature/Bug: Rejected | Rejected→Open; updates skills |
| 5 | CI | agent | `ci` | Feature/Bug: Merged | creates CI Task (Open); Merged→Resolved + Task→Review on pass; Task→Failed on fail |
| 6 | CI-Sweeper | agent | `ci-sweeper` | Task: Failed | Failed→Pending merge (fix PR, git-revert preferred) or →Escalation + STOP |
| 7 | Fix escalated CI | **human** | — | Task: Escalation | solution into Detailed Description, →Rejected, remove STOP |
| 8 | CI-Retrospective | agent | `ci-retrospective` | Task: Rejected | Rejected→Failed; updates skills |
| 9 | Approve CI | **human** | — | Task: Review | →Approved |
| 10 | Publish | agent | `publish` | Task: Approved (+ earlier Review) | →Closed for Task tickets and covered Feature/Bug issues; copies CI run (logs + deliverables) to publish/ |
| 11 | Cost report | agent | `cost-report` | all issues | none (writes COST-REPORT.md) |

## Status machines (binding — no other transitions)

**Feature/Bug**:
- Create → Open → Pending merge → Merged → Resolved → Closed
- Create → Open → Feedback (ambiguous; human clarifies, back to Open)
- Create → Open → Pending merge → Rejected → Open (via retrospective)

**Task (CI tickets)**:
- Open → Review → Approved → Closed
- Open → Failed → Pending merge → Review → Approved → Closed
- Open → Failed → Pending merge → Rejected → Failed (via ci-retrospective)
- Open → Failed → ... → Escalation (sweeper cap hit; human intervenes)

## Control & handoff files

- `STATUS.md` — control words at line start only. `CI` = CI in progress
  (implementation skips); `STOP` = everything gated halts (written by
  ci-sweeper escalation, removed only by a human). Gated skills:
  implementation (STOP|CI), ci (STOP), ci-sweeper (STOP).
- `IMPLEMENTATION.md` — triage → implementation queue.
- `<spec_dir>/` — per-feature specifications; every code change updates its
  spec in the same PR (verifier rejects code-without-spec).
- `CI/<timestamp>/` — logs + `deliverables/` (+ MANIFEST.md) per CI run.
  LOCAL-ONLY: git-ignored, never pushed.
- `publish/<timestamp>/` — approved CI runs, copied by publish. LOCAL-ONLY:
  git-ignored, never pushed.
- `COST-REPORT.md` — cost-report output.

## Scheduling (recommended cadence)

```
/loop 4h  /triage             # 6/day — keep the queue fresh; stagger ahead of implementation
/loop 6h  /implementation     # 4/day — core delivery loop; yields to CI via STATUS.md
/loop 12h /ci                 # 2/day — main-branch validation, matches human merge rhythm
/loop 6h  /ci-sweeper         # 4/day — fix a failed CI within 6 hours
/loop 24h /retrospective      # 1/day — consolidated learning from human rejections
/loop 24h /ci-retrospective   # 1/day — distill escalation lessons daily
/loop 24h /publish            # 1/day — fixed daily release window
/loop 7d  /cost-report        # weekly — token-budget review
```

Stagger start times (triage a couple of hours before implementation). Use
`/schedule` (cloud) instead of `/loop` when the cadence must survive the
session closing. Skipped rounds (empty queue, STATUS gate) are normal and
must exit in a few lines — early-exit cheap, act expensive.

## Loop-wide rules

- Maker/checker: every implementation is verified by a **separate** agent
  before a PR is opened; every CI run executes in a **separate** CI agent.
- Humans own all merges, all rejections, CI approval, and STOP removal.
- No GitHub closing keywords (Closes/Fixes/Resolves) anywhere in PR bodies
  or commit messages — status flow is loop-owned.
- Every code change ships with a spec update in `<spec_dir>/` (same PR).
- Token accounting: every skill adds what it spent per issue to that issue's
  stage field, cumulatively. Cost-report only reads.
- Retrospectives change **skills**, not code.
