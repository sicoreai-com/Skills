# Loop Config

> Every skill reads this file first. Args override
> (e.g. `/triage org=myorg project="My Project"`).

org: YOUR_ORG
project: YOUR_PROJECT_NAME
repo: YOUR_ORG/YOUR_REPO
default_branch: main

# CI-Sweeper: maximum fix→CI rounds before escalation
max_ci_sweeper_loops: 10

# Output directories (relative to repo root).
# Both are LOCAL-ONLY: they are git-ignored and must never be committed or
# pushed to the GitHub repo. CI run folders and published artifacts live only
# on this machine.
ci_output_dir: CI
publish_dir: publish

# Specification folder in the product repo (one md file per feature/module,
# kept truthful by implementation and ci-sweeper in the same PR as the code).
spec_dir: specs

# Files
implementation_queue: IMPLEMENTATION.md
status_file: STATUS.md

# --- Project-specific CI hints ---
# Replace with YOUR project's checks. The CI agent runs these, in order, on a
# clean checkout of the default branch. Example (bun/TypeScript monorepo):
#   bun install --frozen-lockfile
#   bun run lint
#   bun run typecheck
#   per-package tests: `bun test` inside each changed/core package
# ci_deliverables: what CI must copy into <run folder>/deliverables/ —
#   the build output when the project has a build step, otherwise the
#   runtime source of the affected package(s). Always include
#   deliverables/MANIFEST.md (validated SHA, sha256 list, run notes).
