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
| **[engineering-loop](skills/engineering-loop/)** | A complete human-gated engineering pipeline on a GitHub Projects board: agents triage issues, implement them behind a maker/checker gate, run CI with verifiable deliverables, learn from human rejections, and publish approved releases — humans keep every merge, approval, and emergency stop. 8 skills + templates. | [README](skills/engineering-loop/README.md) |

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
- 🚦 **Humans own the judgment calls** — merges, rejections, CI approval, and
  the emergency STOP are always yours.
- 🧾 **Full cost transparency** — every stage books its token spend onto the
  issue it worked on; one command aggregates the bill.

## 🚀 Quick start

```bash
git clone https://github.com/sicoreai-com/Skills.git

# Install a skill suite into your project
cp -r Skills/skills/engineering-loop/skills/* your-repo/.claude/skills/

# Then follow that skill's README for configuration
```

Open your project in Claude Code and the skills appear as slash commands
(`/triage`, `/implementation`, `/ci`, …).

## 🗂️ Repository layout

```
Skills/
├── skills/                  # one folder per skill (or suite)
│   └── engineering-loop/
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
