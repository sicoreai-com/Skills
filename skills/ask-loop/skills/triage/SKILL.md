---
name: triage
description: >
  Triage Open Feature/Bug issues on the GitHub project: assign Priority and
  Effort, route ambiguous issues to Feedback status for human clarification,
  record Analysis Token Usage, and generate IMPLEMENTATION.md — the ordered
  queue consumed by the implementation skill.
user_invocable: true
---

# Triage Skill

You are the intake stage of the implementation loop. Your job is to turn the
raw Open backlog into an unambiguous, prioritized implementation queue. You
never write code and never change issue bodies — you only classify.

Follow `.claude/skills/shared/github-project-ops.md` for all project reads and
writes (config, field IDs, GraphQL item query, field-edit recipes).

## Inputs

- `loop-config.md` (org, project, repo) — args override
- All project items where issue Type is **Feature** or **Bug** and Status is
  **Open**

If no matching issues exist, write an empty queue section to
`IMPLEMENTATION.md` and exit — triage is finished.

## Procedure — per issue

1. **Read** the issue title, body, comments, and linked context.
2. **Ambiguity check first.** An issue is ambiguous if a competent engineer
   could not start it without asking a question: no acceptance criteria, no
   repro for a bug, contradictory requirements, or unstated scope. If
   ambiguous:
   - Set Status → **Feedback**.
   - Comment on the issue with the specific questions a human must answer.
   - Do NOT assign Priority/Effort and do NOT include it in IMPLEMENTATION.md.
3. **Priority** — only if the Priority field is currently empty (never
   overwrite a human-assigned priority):
   | Priority | Signals |
   |----------|---------|
   | Urgent | Security, data loss, prod breakage, blocks all other work |
   | High | High user impact, clear repro / clear spec, blocks other issues |
   | Medium | Valid feature or bug, no urgency signals |
   | Low | Polish, docs, nice-to-have |
4. **Effort** — evaluate and fill the Effort field (overwrite a stale value
   only if the issue changed since it was set):
   | Effort | Meaning |
   |--------|---------|
   | Low | Single well-understood change, ≤ ~3 files, existing test patterns |
   | Medium | Multiple files or a new component, moderate test work |
   | High | Cross-cutting change, design decisions, or migration required |
5. **Analysis Token Usage** — add the tokens you spent evaluating this issue
   to the issue's `Analysis Token Usage` field (cumulative; see shared ref §6).

## Output — `IMPLEMENTATION.md` at repo root

Regenerate the file each run from the issues that passed evaluation (Open,
unambiguous, Priority and Effort set). Order by Priority
(Urgent → Low), then by Effort ascending within a priority.

```markdown
# IMPLEMENTATION
Generated: <ISO timestamp> by triage
<!-- Consumed by the implementation skill, which removes entries as they ship.
     Do not hand-edit ordering unless you mean to re-prioritize. -->

## Queue
- [ ] #123 — <title> (Priority: High, Effort: Low, Type: Bug)
  URL: <issue url>
  Hint: <one line: where to start / suspected root cause / key acceptance criterion>
```

**Merge rule**: if `IMPLEMENTATION.md` already exists, preserve entries for
issues that are still Open (the implementation skill may be mid-queue) and
drop entries whose issue is no longer Open; then add newly triaged issues in
priority order.

**Write-time freshness**: perform this merge immediately before writing the
file — re-read `IMPLEMENTATION.md` and re-check the Status of every issue
about to be written at that moment, not from data gathered at the start of
the run. An issue shipped by a concurrent implementation round mid-triage
must not be re-queued: never write an entry whose Status is not Open at
write time.

## Rules

- Never change Status except Open → Feedback.
- Never overwrite a Priority that is already set.
- Be conservative on ambiguity: a wrong guess implemented is far more
  expensive than one round-trip through Feedback.
- One-line run summary at the end: issues scanned / prioritized / sent to
  Feedback / queued.
