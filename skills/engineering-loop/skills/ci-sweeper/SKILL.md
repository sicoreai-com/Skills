---
name: ci-sweeper
description: >
  Fix Failed CI Task tickets: correct main based on the Failure Log, re-run CI,
  and loop until green (max loops configurable, default 10). On success write a
  PR and set the CI ticket to Pending merge; after max loops set Escalation and
  write STOP into STATUS.md. Gated by STATUS.md STOP.
user_invocable: true
---

# CI-Sweeper Skill

You get a red main branch back to green. You work on Task-type CI tickets in
**Failed** status, fixing whatever the failure logs implicate and re-running
CI until it passes — with a hard loop cap so you never burn tokens on a
problem the loop cannot solve.

Follow `.claude/skills/shared/github-project-ops.md` for all project reads and
writes. Re-use the CI skill's run procedure (separate CI agent, per-run folder
under `ci_output_dir`) for every re-run.

## Gate — check before starting

Read `STATUS.md`. If any line starts with the control word **STOP** (shared
ref §5), skip this round: log one line and exit. Re-check between tickets.

## Inputs

- All project items where issue Type is **Task** and Status is **Failed**.
  If none, the sweeper is finished — exit with a one-line note.
- Per ticket: `Failure Log` field, `Detailed Description` field (may contain
  human notes or prior sweeper attempts), current `CI Token Usage`.
- `loop-config.md`: `max_ci_sweeper_loops` (default **10**).

## Procedure — per Failed CI ticket

Work in a fresh worktree on a branch `ci-fix-<ticket-number>` off the default
branch. Then loop, counting rounds (rounds already recorded in Detailed
Description by previous sweeper runs count toward the cap):

1. **Diagnose** from the `Failure Log` and `Detailed Description` fields plus
   the CI run folders. Identify the minimal root cause.
2. **Correct.** Make the smallest fix that addresses the failure. Never
   disable tests, skip assertions, or loosen checks to go green. When the
   failure was introduced by an identifiable commit, express the fix as
   `git revert <sha>` (plus follow-up commits if the revert alone isn't
   enough) rather than hand-editing the same lines back — history must show
   cause and remedy.
   **Specification:** every code change updates the affected spec in the
   product repo's `<spec_dir>` (see the implementation skill) in the same
   branch/PR — at minimum a Change Log entry (date, CI ticket, PR, what the
   fix changed); if the fix alters or restores documented behavior, correct
   the behavior section too. If the offending commit had changed behavior
   without touching the spec, say so in the entry — that omission is part of
   the root cause.
3. **Record the attempt.** Append to `Detailed Description`: round number,
   what was diagnosed, what was changed (files + one-line rationale).
4. **CI again** via a separate CI agent (same procedure and output-folder
   convention as the `ci` skill — including the `deliverables/` subfolder
   with the validated artifacts + MANIFEST.md, not only logs), run against
   the fix branch. A green re-run whose folder lacks populated
   `deliverables/` is not green.
5. **Evaluate:**
   - **Green** → exit the loop, go to "On success".
   - **Red and rounds < max** → append the new failure logs to
     `Detailed Description`, add this round's tokens to `CI Token Usage`
     (cumulative), and loop back to step 1. If the error is identical to the
     previous round's, treat the diagnosis as wrong — change approach, don't
     retry the same fix.
   - **Red and rounds = max** → go to "On escalation".

### On success

- Add the final round's tokens to `CI Token Usage`.
- **Write a PR** from the fix branch against the default branch — the PR is
  the output of this loop. Body: the CI ticket link, per-round summary, and
  the green CI run folder as evidence.
- CI ticket: Status → **Pending merge** (a human reviews and merges).

### On escalation (max loops exhausted)

- CI ticket: append the latest CI failure info to `Detailed Description`;
  Status → **Escalation**.
- Append to `STATUS.md`:
  `STOP: CI escalation on Task #<n> — <one-line failure summary>` — this
  halts implementation, CI, and further sweeping until a human fixes the
  problem and removes the STOP line.

## Rules

- The loop cap is a circuit breaker — never raise it mid-run or "just try one
  more" past it. Escalation is a feature, not a failure.
- Each round must change something meaningful; two identical consecutive
  attempts means escalate your approach within the round budget.
- Never push fixes directly to the default branch — always via PR.
- End-of-run summary per ticket: rounds used, outcome (PR link or
  Escalation), tokens added.
