# AskLoop — the Engineering Loop

A set of [Claude Code](https://claude.com/claude-code) skills that runs an
autonomous, **human-gated** engineering pipeline on top of a GitHub Projects
(v2) board:

```
          agent                agent               HUMAN            agent              HUMAN           agent
Open ──► triage ──► queue ──► implement ──► PR ──► merge ──► CI ──► pass ──► Review ──► approve ──► publish ──► Closed
                                                     │                │
                                                  reject            fail
                                                     │                │
                                              retrospective      ci-sweeper ──► (fix PR | Escalation+STOP)
                                             (improves skills)
```

Agents do the repetitive work — triage, implementation, CI, publishing, cost
accounting. Humans keep the judgment calls — every merge, every rejection, CI
approval, and lifting emergency stops. When a human rejects the loop's work,
a retrospective skill updates the *skills themselves* so the same mistake
isn't repeated.

## What's in this folder

| Path | What it is |
|---|---|
| `skills/triage/` | Classify Open Feature/Bug issues: Priority, Effort, ambiguity → Feedback, and generate `IMPLEMENTATION.md` (the ordered work queue) |
| `skills/implementation/` | Work the queue: one worktree/branch/PR per issue, an implementer sub-agent checked by an **independent verifier** sub-agent, spec updated in the same PR |
| `skills/ci/` | Validate merged code on a clean checkout via a separate CI agent; logs + validated deliverables per run; CI runs are tracked as Task tickets |
| `skills/ci-sweeper/` | Fix Failed CI runs and loop until green (bounded); escalates with a STOP after too many rounds |
| `skills/retrospective/` | Learn from human-Rejected issues: update skills, reopen the issue |
| `skills/ci-retrospective/` | Same, for rejected CI-fix attempts |
| `skills/publish/` | Turn human-Approved CI runs into published artifacts (local `publish/` folder) and close everything they cover |
| `skills/cost-report/` | Read-only token-cost summary across all issues (`COST-REPORT.md`) |
| `skills/shared/github-project-ops.md` | The single source of truth for all `gh` CLI / GraphQL board operations — every skill follows it |
| `templates/LOOP.md` | The loop's contract: stages, binding status machines, control files, rules |
| `templates/loop-config.md` | Per-project configuration (org, project, repo, CI hints) — fill in the placeholders |
| `templates/STATUS.md` | Control file for `STOP` / `CI` gate words |
| `templates/IMPLEMENTATION.md` | Empty starting queue |

## Prerequisites

- **Claude Code** with the `gh` CLI installed and authenticated
  (`gh auth status`) with rights to your repo and organization project.
- A **GitHub Projects (v2)** board owned by your org (or user).
- **Issue types** `Feature`, `Bug`, and `Task` enabled on the repository
  (GitHub org setting) — the loop routes work by issue type.
- The board needs these custom fields (names must match exactly):

| Field | Type | Notes |
|---|---|---|
| `Status` | Single select | Options, exactly: `Create`, `Open`, `Pending merge`, `Merged`, `Resolved`, `Closed`, `Feedback`, `Rejected`, `Review`, `Approved`, `Failed`, `Escalation` |
| `Priority` | Single select | `Urgent`, `High`, `Medium`, `Low` |
| `Effort` | Single select | `Low`, `Medium`, `High` |
| `Estimate Token`, `Budget Token` | Number | Optional planning fields |
| `Analysis Token Usage`, `Implementation Token Usage`, `Retrospective Token Usage`, `CI Token Usage` | Number | Cumulative per-stage token accounting, written by the skills |
| `Detailed Description` | Text | Human ↔ agent handoff (rejection reasons, CI summaries). 1024-char limit — long histories go to issue comments |
| `Failure Log` | Text | CI failure evidence for the sweeper |

## Setup

1. **Install the skills** into the repo the loop should run on:

   ```bash
   cp -r skills/* /path/to/your-repo/.claude/skills/
   ```

2. **Install the control files** at the repo root:

   ```bash
   cp templates/LOOP.md templates/loop-config.md templates/STATUS.md templates/IMPLEMENTATION.md /path/to/your-repo/
   ```

3. **Fill in `loop-config.md`**: `org`, `project` (the board's title), `repo`,
   `default_branch`, your `spec_dir`, and — important — the project-specific
   CI hints block (the exact install/lint/typecheck/test commands and what
   counts as deliverables). Skills refuse to guess when placeholders like
   `YOUR_ORG` are still present.

4. **Git-ignore the local output folders** (they must never be pushed):

   ```gitignore
   # Loop output folders (local-only, never pushed to the repo)
   /CI/
   /publish/
   ```

5. **Create the board** with the fields above and add your Feature/Bug issues
   with Status `Open`.

## Running the loop

Each stage is a slash command — run them manually while you build trust:

```
/triage            # classify + queue
/implementation    # queue → PRs
/ci                # validate merged code
/ci-sweeper        # fix failed CI
/retrospective     # learn from rejections
/publish           # release approved runs
/cost-report       # what did it all cost?
```

…then put them on a cadence with Claude Code's `/loop` command
(see `templates/LOOP.md` for a starting cadence):

```
/loop 30m /implementation
/loop 1h  /triage
/loop 2h  /ci
```

Skipped rounds (empty queue, gate set) are normal and exit in a few lines.

### What the human does

The loop stops and waits for you at four points:

1. **Review & merge PRs** (issue Status `Pending merge`). Merge → set the
   issue to `Merged`. Reject → set `Rejected` and write the reason into
   `Detailed Description` (the retrospective reads it verbatim).
2. **Clarify ambiguous issues** (Status `Feedback`): answer the questions the
   triage skill commented on the issue, set back to `Open`.
3. **Approve CI runs** (CI Task ticket Status `Review`): check the run folder
   (`CI/<timestamp>/` — summary, logs, `deliverables/MANIFEST.md`), then set
   `Approved` (publish picks it up) or `Rejected` with a reason.
4. **Handle escalations**: if the CI sweeper gives up it sets `Escalation`
   and writes `STOP` into `STATUS.md` — everything gated halts until a human
   fixes the problem and removes the STOP line. Only humans remove `STOP`.

## Safety properties

- **Maker/checker everywhere**: implementations are verified by a separate
  agent with a reject-by-default stance before any PR is opened; CI executes
  in a separate agent whose logs are the evidence.
- **No auto-close**: GitHub closing keywords are banned in PR bodies; every
  status transition follows the state machines in `LOOP.md`, and only the
  loop (or you) moves tickets.
- **Bounded retries**: implement→verify is capped at 3 attempts per issue,
  CI fix rounds at `max_ci_sweeper_loops`; past the cap the loop escalates to
  a human instead of thrashing.
- **Specs stay truthful**: code changes without a matching spec update in the
  same PR are rejected by the verifier.
- **Local-only artifacts**: `CI/` and `publish/` never leave the machine.
- **Full cost transparency**: every skill books its tokens onto the issue it
  worked on; `/cost-report` aggregates them per ticket type and stage.

## Layout after setup

```
your-repo/
├── .claude/skills/           # the 8 skills + shared/
├── LOOP.md                   # the contract (stages, state machines, rules)
├── loop-config.md            # your project's settings
├── STATUS.md                 # gate file (STOP / CI control words)
├── IMPLEMENTATION.md         # triage → implementation queue
├── specs/                    # per-feature specifications (grows with PRs)
├── CI/                       # local-only CI run folders (git-ignored)
└── publish/                  # local-only published artifacts (git-ignored)
```

## License

Same license as this repository — see [LICENSE](../../LICENSE).
