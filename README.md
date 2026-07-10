<div align="center">

# 🛠️ SicoreAI Skills

**Open-source skills for [Claude Code](https://claude.com/claude-code) —
teach your agent how your team ships software.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-skills-d97757.svg)](https://docs.claude.com/en/docs/claude-code)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](#-contributing)

</div>

---

A **skill** is a markdown playbook (`SKILL.md`) that Claude Code loads on
demand — invoked as a slash command or triggered automatically when a task
matches. Skills turn one-off prompting into a repeatable, reviewable,
version-controlled process: the agent follows the same procedure every time,
and when it makes a mistake, you fix the *skill*, not the chat.

## 📦 Skill catalog

| Skill | What it does | Docs |
|---|---|---|
| **[ask-loop](skills/ask-loop/)** | A complete human-gated engineering pipeline on a GitHub Projects board: agents triage issues, implement them behind a maker/checker gate, run CI with verifiable deliverables, learn from human rejections, and publish approved releases — humans keep every merge, approval, and emergency stop. 8 skills + templates. | [README](skills/ask-loop/README.md) |

> More skills are on the way. Watch the repo to catch new releases.

### ✨ Spotlight: the engineering loop

```
          agent                agent               HUMAN            agent              HUMAN           agent
Open ──► triage ──► queue ──► implement ──► PR ──► merge ──► CI ──► pass ──► Review ──► approve ──► publish ──► Closed
                                                     │                │
                                                  reject            fail
                                                     │                │
                                              retrospective      ci-sweeper ──► (fix PR | Escalation+STOP)
                                             (improves skills)
```

Why teams like it:

- 🔁 **Self-improving** — human rejections feed retrospectives that edit the
  skills themselves, so the same mistake isn't made twice.
- 🧑‍⚖️ **Maker/checker built in** — every implementation is verified by an
  independent agent with a reject-by-default stance before a PR exists.
- 📜 **Specs as a first-class citizen** — every change ships with its spec in
  the same PR, so the system's evolution stays trackable and explainable:
  living documentation that agents and developers collaborate through, not
  stale docs bolted on later.
- 🔗 **Traceable end to end** — every PR is linked to a GitHub issue, so each
  code change traces back to the Feature or Bug that motivated it: from idea
  to issue to diff to release, the whole evolution of the system is one
  connected, auditable thread.
- 🚦 **Humans own the judgment calls** — merges, rejections, CI approval, and
  the emergency STOP are always yours.
- 🧾 **Full cost transparency** — every stage books its token spend onto the
  issue it worked on; one command aggregates the bill.

## 🔄 Issue lifecycle — and where humans decide

Every piece of work is a GitHub issue moving through a **binding status
machine**. Agents move issues between statuses as they work; the transitions
marked 🧑 are yours and yours alone — the loop stops and waits for you there.

**Feature/Bug issues** (the work you want done):

```
Create ─► Open ─► Pending merge ─🧑─► Merged ─► Resolved ─► Closed      the happy path
                       │                agent CI ▲      publish ▲
          ▲            🧑
          │            ▼
          └───────  Rejected      (retrospective learns from your reason, reopens)
Create ─► Open ─🧑?─► Feedback ─🧑─► Open      (triage found it ambiguous; you clarify)
```

- `Open → Pending merge` — agent implemented it and opened a PR
- 🧑 `Pending merge → Merged` — **you** reviewed and merged the PR
- 🧑 `Pending merge → Rejected` — **you** rejected it (reason goes in
  `Detailed Description`; the retrospective skill turns it into a skill fix
  and reopens the issue)
- 🧑 `Feedback → Open` — **you** answered triage's questions on the issue

**Task issues** (one per CI run, created by the CI skill):

```
Open ─► Review ─🧑─► Approved ─► Closed                        CI green on first try
Open ─► Failed ─► Pending merge ─🧑─► Review ─🧑─► Approved ─► Closed    sweeper fixed it, you merged
Open ─► Failed ─► Pending merge ─🧑─► Rejected ─► Failed       you rejected the fix; ci-retrospective learns, sweeper retries
Open ─► Failed ─► … ─► Escalation 🚨                           fix cap reached; STOP written, humans take over
```

### 🧑 Your four duties

| When you see | What you do |
|---|---|
| Feature/Bug or Task in **Pending merge** | Review the PR. Merge → set Feature/Bug to `Merged`, Task to `Review`. Reject → set `Rejected` **and write the reason into `Detailed Description`** — that text is what the retrospective learns from. |
| Feature/Bug in **Feedback** | Triage found the issue too ambiguous to implement. Answer its questions in the issue comments, then set the issue back to `Open`. |
| Task in **Review** | A CI run passed and awaits sign-off. Check the run folder (`CI/<timestamp>/` — summary, logs, `deliverables/MANIFEST.md`), then set `Approved` (publish picks it up) or `Rejected` with a reason. |
| Task in **Escalation** (+ `STOP` in `STATUS.md`) 🚨 | The CI sweeper hit its retry cap and halted the loop. Fix the CI problem, put your solution into `Detailed Description`, set the ticket to `Rejected` (so ci-retrospective learns from it), and **remove the `STOP` line** — only humans may remove a STOP. |

Everything else — prioritizing, coding, verifying, running CI, sweeping
failures, publishing, cost accounting — the agents handle.

## ✅ Prerequisites

Before installing ask-loop, have these three things in place:

1. **Connect Claude Code to GitHub.** Claude Code drives everything through
   the GitHub CLI (`gh`), so it must be authenticated with access to your
   repository and organization project. Just ask Claude Code — it will walk
   you through the detailed procedure. Note: granting the required org
   permissions typically needs your **org admin's** help.

2. **A GitHub repository** for your source code — use an existing repo or
   create a new one. The skills, templates, and loop state files live inside
   it.

3. **A GitHub Project (v2)** where issues track every change. The project
   must meet the requirements below — and here's a hint: you can simply ask
   Claude Code to create the project with these requirements for you.

   **Issue types** (repository/org setting): `Feature`, `Bug`, `Task`

   **Status options**: `Create`, `Open`, `Pending merge`, `Merged`,
   `Resolved`, `Closed`, `Feedback`, `Rejected`, `Review`, `Approved`,
   `Failed`, `Escalation`

   **Fields**:

   | Field | Kind | Notes |
   |---|---|---|
   | Title, Issue body, Type, Issue author, Assignees, Updated | built-in | plus the project activity log & issue timeline |
   | `Priority` | built-in single select | `Urgent` / `High` / `Medium` / `Low` |
   | `Effort` | built-in single select | `Low` / `Medium` / `High` |
   | `Status` | single select | the 12 options above |
   | `Estimate Token`, `Budget Token` | number | planning fields |
   | `Analysis Token Usage`, `Implementation Token Usage`, `Retrospective Token Usage`, `CI Token Usage` | number | per-stage cost accounting, written by the skills |
   | `Detailed Description` | text | human ↔ agent handoff (rejection reasons, CI summaries) |
   | `Failure Log` | text | CI failure evidence for the sweeper |

## 🚀 Quick start

```bash
git clone https://github.com/sicoreai-com/Skills.git

# Install a skill suite into your project
cp -r Skills/skills/ask-loop/skills/* your-repo/.claude/skills/

# Then follow that skill's README for configuration
```

Open your project in Claude Code and the skills appear as slash commands
(`/triage`, `/implementation`, `/ci`, …).

## 🗂️ Repository layout

```
Skills/
├── skills/                  # one folder per skill (or suite)
│   └── ask-loop/
│       ├── README.md        # setup & usage guide
│       ├── skills/          # the SKILL.md playbooks
│       └── templates/       # config & control files to copy into your repo
├── LICENSE                  # MIT
└── README.md                # you are here
```

## 🤝 Contributing

Contributions are welcome! To add a skill:

1. Create `skills/<meaningful-english-name>/` containing a `SKILL.md`
   (frontmatter: `name`, `description`; body: the procedure).
2. Suites with multiple skills get a `README.md` covering setup, usage, and
   any templates they need.
3. Keep skills **project-agnostic**: placeholders (`YOUR_ORG`) over
   hard-coded values, configuration in a template file, no secrets, no
   machine-specific paths.
4. Open a PR describing what the skill automates and how you validated it.

## 📄 License

[MIT](LICENSE) © 2026 SicoreAI

<div align="center">
<sub>Built with Claude Code · Skills over prompts</sub>
</div>
