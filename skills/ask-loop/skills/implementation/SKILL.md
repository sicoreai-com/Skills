---
name: implementation
description: >
  Implement the issues queued in IMPLEMENTATION.md. For each issue: worktree →
  implementer sub-agent → independent verifier sub-agent → PR → update the GIT
  issue (Implementation Token Usage, status Pending merge, PR link in body) →
  remove from the queue. Gated by STATUS.md: skips the round if STOP or CI is
  present.
user_invocable: true
---

# Implementation Skill

You are the maker stage of the loop. You turn triaged issues into pull
requests. You never merge — humans review and merge every PR.

Follow `.claude/skills/shared/github-project-ops.md` for all project reads and
writes.

## Gate — check before starting

Read `STATUS.md`. If any line **starts with** the control word **STOP** or
**CI** (shared ref §5 — documentation inside the file never counts), skip
this entire round: log one line stating which word blocked you and exit.
The next scheduled run will retry. Re-check the gate before starting each
subsequent issue as well — a CI run may have started mid-round.

## Inputs

- `IMPLEMENTATION.md` at repo root. If missing or its Queue is empty,
  implementation is finished — exit with a one-line note.
- `loop-config.md` (repo, default branch)

## Procedure — repeat for each issue in queue order

1. **Freshness check.** `IMPLEMENTATION.md` may be stale — a triage run
   that overlapped a previous implementation round can restore an entry for
   an already-shipped issue. Before any work, query the issue's project
   Status on the board: proceed only if it is **Open**. Anything else
   (Pending merge, Merged, Resolved, ...) means the entry is stale: remove
   it from `IMPLEMENTATION.md`, note it in the run summary, and move to the
   next issue.
2. **Open a worktree.** Create an isolated git worktree on a fresh branch
   named `issue-<number>-<slug>` off the default branch. All work for this
   issue happens there.
3. **Implementer sub-agent.** Spawn a sub-agent in the worktree with: the
   issue title/body/comments, the Hint line from IMPLEMENTATION.md, and the
   project's build/test commands. Instructions: smallest change that fully
   resolves the issue, include/adjust tests, run them, no unrelated refactors,
   never disable tests to go green. When the change introduces new user-facing
   syntax (CLI flags/options, config keys, API params), near-miss inputs must
   fail with a message naming the actual problem (a mistyped `--descendng` is
   an "unknown option", not "not a number") — include a test for at least one
   near-miss. It returns a summary: files changed, diff summary, tests run +
   results.
   **Specification (mandatory, every code change).** Specs live in the
   product repo's `<spec_dir>` (from `loop-config.md`; default `specs/`, or
   the repo's existing spec folder such as `specification/`), one markdown
   file per feature/module (e.g. `sort-cli.md`). The implementer keeps them
   truthful in the SAME branch/PR as the code:
   - New Feature → create or extend the spec: overview, behavior (inputs,
     outputs, options, error handling), acceptance criteria.
   - Bug fix → update the spec to state the corrected behavior; if the spec
     was silent on the buggy case, that silence was part of the bug — spell
     the case out now.
   - Every spec ends with a **Change Log** table: date, issue, PR, one-line
     summary of what changed and why.
4. **Verifier sub-agent — must be a separate agent**, never the implementer
   and never yourself reusing the implementer's context. Give it the issue,
   the diff, and the implementer's claims. Its stance is REJECT unless
   evidence is strong: it re-runs the tests itself, checks the diff only
   touches relevant files, checks the change addresses the stated issue,
   checks nothing was disabled or weakened, and **checks the spec**: the
   `<spec_dir>` file for the touched feature was created/updated in this
   diff, describes the behavior the verifier actually observed (including
   error cases), and has a Change Log entry for this issue. Code change
   without a matching spec change is an automatic REJECT.
   Verdict: APPROVE / REJECT.
   - On REJECT: send the numbered reasons back to a fresh implementer pass in
     the same worktree. Maximum 3 implement→verify attempts per issue; after
     that, leave the issue in the queue, comment the verifier's findings on
     the GitHub issue, and move on to the next issue.
5. **Write the PR** (only after APPROVE). Push the branch and open a PR
   against the default branch: title `#<issue>: <issue title>`, body with
   summary, test evidence, verifier verdict — and NO GitHub closing keyword
   anywhere in the body: never `Closes/Fixes/Resolves #N` or `<keyword>
   owner/repo#N` (all of them auto-close the issue on merge; say
   "Implements #N" / "for #N" instead). Status flow is managed by the loop,
   not by auto-close. The PR is the
   output of this skill.
6. **Update the GIT issue:**
   - Add tokens spent by implementer + verifier for this issue to the
     `Implementation Token Usage` field (cumulative).
   - Set Status → **Pending merge**.
   - Append the PR link to the issue body (shared ref §3).
7. **Dequeue.** Remove this issue's entry from `IMPLEMENTATION.md`.
8. **Clean up** the worktree (the branch stays for the PR).

## Rules

- Maker/checker is non-negotiable: the verifier is a separate sub-agent with
  no stake in the implementation.
- One issue = one worktree = one branch = one PR. Never batch issues.
- If a `gh`/git step fails after the PR exists, finish the remaining issue
  updates manually where possible and report exactly what state the issue was
  left in.
- End-of-run summary: issues implemented (with PR links), issues skipped and
  why, gate status.
