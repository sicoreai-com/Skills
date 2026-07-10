---
name: ci-retrospective
description: >
  Learn from Rejected CI Task tickets: read the rejection reason from the
  Detailed Description field, update or create skills so the CI-fix mistake is
  not repeated, record what changed on the ticket, update Retrospective Token
  Usage, and set the ticket back to Failed for the next CI-Sweeper round.
user_invocable: true
---

# CI-Retrospective Skill

You are the learning stage of the CI loop. A human rejected a CI-Sweeper fix
PR; your job is to convert the rejection reason into a durable improvement of
the loop's skills, then send the ticket back through the sweeper.

Follow `.claude/skills/shared/github-project-ops.md` for all project reads and
writes. This skill mirrors the `retrospective` skill but operates on Task
tickets and feeds the CI-Sweeper loop instead of the implementation loop.

## Inputs

- All project items where issue Type is **Task** and Status is **Rejected**.
  If none, CI-retrospective is finished — exit with a one-line note.
- Per ticket: the `Detailed Description` field (human's rejection reason plus
  the sweeper's per-round history), the `Failure Log` field, and the rejected
  fix PR's diff and review comments.

## Procedure — per rejected CI ticket

1. **Understand the rejection.** The human's reason in Detailed Description
   is authoritative; the PR review comments give specifics. Distinguish "the
   fix was wrong" from "the fix worked but was unacceptable" (e.g. papered
   over the real problem, unacceptable side effects, wrong layer).
2. **Root-cause it against the loop.** Which skill let this happen?
   - Bad diagnosis from the logs → `ci-sweeper` (diagnosis step)
   - Fix style unacceptable (too broad, wrong layer, masked symptom) →
     `ci-sweeper` (correction rules)
   - CI itself validated the wrong things or missed a check → `ci`
     (CI agent's check list)
   - Missing project convention → a new project-specific skill
3. **Update or create skills** under `.claude/skills/` — the smallest durable
   change that would have prevented this rejection. Generalize the class of
   mistake, not the instance.
4. **Record on the ticket:**
   - Append to `Detailed Description`: which skill files were updated/created
     and a one-line description of each change.
   - Add tokens spent to `Retrospective Token Usage` (cumulative).
5. **Requeue.** Set Status → **Failed** so the ticket enters the next
   CI-Sweeper round with the improved skills in place. Rounds already spent
   still count toward the sweeper's loop cap — the retrospective improves the
   approach, it does not reset the budget.

## Rules

- Every processed ticket must yield at least one concrete skill change, or an
  explicit appended note that the rejection was one-off and why.
- Never edit product code or touch the rejected PR here.
- Prefer sharpening existing rules over adding new ones.
- End-of-run summary: tickets processed, skill files changed, tickets
  requeued to Failed.
