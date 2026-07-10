---
name: publish
description: >
  Publish human-Approved CI runs: copy the approved CI run folder — logs plus
  the validated deliverables (deliverables/ + MANIFEST.md) — into the
  publish/ subfolder, close every Feature/Bug issue included in the handled CI
  Task tickets, and close those Task tickets — including earlier Review-status
  CI tickets superseded by the approval.
user_invocable: true
---

# Publish Skill

You are the release stage of the loop. A human has approved a CI run (Task
ticket Status → Approved); you turn that approval into a published artifact
and close out everything it covers.

Follow `.claude/skills/shared/github-project-ops.md` for all project reads and
writes.

## Inputs

- All project items where issue Type is **Task** and Status is **Approved**.
  If none, publish is finished — exit with a one-line note.
- For each Approved ticket, all **Task**-type tickets in **Review** status
  created **before** that Approved ticket (compare issue `createdAt`). A later
  approved CI validates a superset of the code, so these earlier runs are
  superseded and are handled — and closed — in this round.
- `loop-config.md`: `ci_output_dir` (default `CI`), `publish_dir`
  (default `publish`). Both trees are LOCAL-ONLY: they are git-ignored and
  must never be committed or pushed to the GitHub repo — publishing means
  copying into the local `<publish_dir>/`, not creating a git artifact.

## Procedure — per Approved CI ticket (oldest first)

1. **Collect the handled set**: this Approved ticket + all Review-status Task
   tickets created before it.
2. **Publish the artifact.** Locate the Approved ticket's CI run folder under
   `<ci_output_dir>/` (identified by the run timestamp in the ticket
   title/Detailed Description). **Check it contains the true deliverables**:
   a non-empty `deliverables/` subfolder with `MANIFEST.md` (the validated
   product files/build output — not just logs). If it's
   missing (e.g. a run from before the deliverables convention), reconstruct
   it from the exact commit SHA recorded in the run's `summary.md` — checkout
   that SHA, copy the product files + write MANIFEST.md — before publishing;
   note the reconstruction in the run summary. Then copy the entire folder to
   `<publish_dir>/<same-folder-name>/` — logs + deliverables together are the
   approval record and the output of the publish loop. If the destination
   already exists, this run was already published: skip the copy, warn, and
   continue with status updates.
3. **Close the covered Feature/Bug issues.** Parse the Issue body of every
   ticket in the handled set for the included `#<number>` Feature/Bug
   references. For each: Status → **Closed** and close the GitHub issue
   (`gh issue close`).
4. **Close the Task tickets.** Every ticket in the handled set (the Approved
   one and the superseded Review ones): Status → **Closed** and close the
   GitHub issue.

## Rules

- Only human approval (Approved status) triggers publishing — never publish a
  Review-status run on its own.
- Never delete or modify anything under `<ci_output_dir>/` — publish copies,
  it doesn't move.
- Published folders stay local: never `git add`, commit, or push anything
  under `<publish_dir>/` or `<ci_output_dir>/` (both are in `.gitignore`;
  keep them there).
- If a Feature/Bug referenced by a handled ticket is not in Resolved status,
  close it anyway per the flow, but flag it in the run summary — it indicates
  a state-machine drift worth a human look.
- End-of-run summary: published folder(s), Task tickets closed, Feature/Bug
  issues closed, anomalies.
