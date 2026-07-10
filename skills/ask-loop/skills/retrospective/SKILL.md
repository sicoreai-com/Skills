---
name: retrospective
description: >
  Learn from Rejected Feature/Bug issues: read the rejection reason from the
  Detailed Description field, update or create skills so the mistake is not
  repeated, record what changed back onto the issue, update Retrospective
  Token Usage, and reopen the issue (Status → Open) for the next
  implementation round.
user_invocable: true
---

# Retrospective Skill

You are the learning stage of the implementation loop. A human rejected a PR;
your job is to convert the rejection reason into a durable improvement of the
loop's skills, then send the issue back through implementation. Skill updates
are the output of this loop — the code fix comes on the next round.

Follow `.claude/skills/shared/github-project-ops.md` for all project reads and
writes.

## Inputs

- All project items where issue Type is **Feature** or **Bug** and Status is
  **Rejected**. If none, retrospective is finished — exit with a one-line
  note.
- Per issue: the `Detailed Description` field (human's rejection reason), the
  issue body (includes the PR link), and the rejected PR's diff and review
  comments.

## Procedure — per rejected issue

1. **Understand the rejection.** Read Detailed Description first; it is the
   authoritative reason. Read the PR review comments and diff for specifics.
2. **Root-cause it against the loop, not the code.** Ask: which skill let
   this happen?
   - Wrong thing built / misread requirements → `triage` (ambiguity rubric)
   - Correct intent, bad execution → `implementation` (implementer
     instructions)
   - Defect the checker should have caught → `implementation` (verifier
     checklist)
   - A project convention the loop didn't know → a new project-specific skill
     or an addition to an existing one
3. **Update or create skills** under `.claude/skills/`. Make the smallest
   durable change that would have prevented this rejection: a new checklist
   line, a sharpened rubric row, a new rule. When creating a new skill, follow
   the structure of the existing ones (frontmatter with name/description, then
   Inputs / Procedure / Rules). Generalize — encode the class of mistake, not
   the single instance.
4. **Record on the issue:**
   - Append to `Detailed Description`: which skill files were updated/created
     and a one-line description of each change (append, never overwrite —
     shared ref §3).
   - Add tokens spent on this retrospective to `Retrospective Token Usage`
     (cumulative).
5. **Reopen.** Set Status → **Open** so the issue enters the next triage/
   implementation round with the improved skills in place.

## Rules

- Every Rejected issue processed must result in at least one concrete skill
  change, or an explicit appended note that the rejection was
  one-off/non-generalizable and why.
- Never edit product code here. Never touch the rejected PR.
- Don't bloat skills: prefer sharpening an existing rule over adding a new
  one; merge duplicates when you see them.
- If two rejections share a root cause, make one skill change and reference
  it from both issues.
- End-of-run summary: issues processed, skill files changed, issues reopened.
