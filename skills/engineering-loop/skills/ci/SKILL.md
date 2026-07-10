---
name: ci
description: >
  Run CI over freshly Merged Feature/Bug issues: create a "CI <date> <time>"
  Task ticket listing them, set the CI word in STATUS.md, build/test/validate
  main via a separate CI agent, save logs AND the validated deliverables
  (deliverables/ + MANIFEST.md) under the CI/ run folder, then mark
  the Feature/Bug tickets Resolved and the CI ticket Review on success, or
  Failed with logs on failure. Gated by STATUS.md STOP.
user_invocable: true
---

# CI Skill

You validate merged code on the default branch. Each CI run is itself tracked
as a Task-type GIT issue so downstream stages (sweeper, publish, cost report)
can operate on it.

Follow `.claude/skills/shared/github-project-ops.md` for all project reads and
writes and for creating the Task issue (§4).

## Gate — check before starting

Read `STATUS.md`. If any line starts with the control word **STOP** (shared
ref §5), skip this round: log one line and exit. Wait for the next scheduled
run.

## Inputs

- All project items where issue Type is **Feature** or **Bug** and Status is
  **Merged**. These are exactly the issues merged since the last CI execution
  (a successful CI moves them onward to Resolved, so anything still in Merged
  is un-validated). If none, CI is finished — exit with a one-line note.
- `loop-config.md`: repo, default branch, `ci_output_dir` (default `CI`).

## Procedure

1. **Announce.** Append a line to `STATUS.md`:
   `CI: run started <ISO timestamp>` — this tells the implementation skill to
   skip its rounds while CI runs.
2. **Create the CI ticket.** Create a **Task**-type issue:
   - Title: `CI <YYYY-MM-DD> <HH:MM>` (UTC).
   - Issue body: the list of all included Feature/Bug tickets, one per line as
     `- #<number> — <title>`.
   - Add it to the project; initial Status → **Open**.
3. **Prepare the output folder**: `<ci_output_dir>/<YYYY-MM-DD_HHMM>/` — one
   folder per CI run, never reused. The `<ci_output_dir>` tree is LOCAL-ONLY:
   it is git-ignored and must never be committed or pushed to the repo. If a
   commit is ever needed while it exists, verify it is not staged.
4. **Run CI via a separate CI agent** (spawn a sub-agent; do not build/test in
   your own context). On a clean checkout of the default branch it runs the
   project's build, full test suite, lint/typecheck, and any other checks the
   project defines. It writes `build.log`, `test.log`, and a `summary.md`
   (overall PASS/FAIL + per-check results) into the run folder, and returns
   PASS or FAIL with the key evidence.
   **Deliverables — the run folder must contain the true deliverables, not
   only logs.** The CI agent copies the validated artifacts into
   `<run folder>/deliverables/`: the build output when the project has a
   build step, otherwise the runtime product files exactly as validated
   (for this project see `ci_deliverables` in `loop-config.md`). Alongside them it writes `deliverables/MANIFEST.md`: the validated
   commit SHA, the file list with sha256 checksums, and how to run/install.
   A PASS without a populated `deliverables/` folder is incomplete — treat
   it as a CI infrastructure failure, not a pass.
5. **On success:**
   - Every included Feature/Bug ticket: Status → **Resolved**.
   - CI ticket: append the validation summary and log locations to
     `Detailed Description`; add tokens spent to `CI Token Usage`;
     Status → **Review** (a human approves CI runs).
6. **On failure:**
   - CI ticket: append the failure logs (the failing commands and their
     output, trimmed to the failing parts) to the `Failure Log` field; add
     tokens spent to `CI Token Usage`; Status → **Failed** (the ci-sweeper
     skill picks it up). Included Feature/Bug tickets stay in **Merged**.
7. **Clear the gate.** Whether CI passed or failed, remove every `CI` line
   from `STATUS.md`. Never touch `STOP` lines.

## Rules

- The CI agent is a separate sub-agent — its logs are the evidence; never
  report PASS without a green run recorded in the run folder.
- Full outputs live in the run folder; issue fields get summaries plus the
  folder path, not megabytes of logs.
- Run folders stay local: never `git add`, commit, or push anything under
  `<ci_output_dir>/` (it is listed in `.gitignore`; keep it that way).
- If the run is interrupted, still perform step 7 (remove your CI lines)
  before exiting, and note the interruption on the CI ticket.
- End-of-run summary: CI ticket link, verdict, included issues, run folder.
