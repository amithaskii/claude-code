# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is **not** the Claude Code CLI source (that's closed-source and distributed via npm/curl installer). This repo is the public-facing companion repo for Claude Code: bundled plugins, example configs, GitHub automation scripts, and docs. Treat it as config/tooling/automation, not an application with its own build.

## Repository layout

- `plugins/` — official Claude Code plugins (commands, agents, hooks, skills). See `plugins/README.md` for the full table of what each plugin provides.
- `.claude-plugin/marketplace.json` — the marketplace manifest listing every plugin in `plugins/` with name, version, author, and category. **Any new or renamed plugin under `plugins/` must be added/updated here too**, or it won't be installable via `/plugin`.
- `.claude/commands/` — slash commands used for *this repo's own* issue/PR automation (`dedupe.md`, `triage-issue.md`, `commit-push-pr.md`), invoked by the GitHub Actions workflows below.
- `scripts/` — TypeScript (run with **Bun**) and bash scripts backing the GitHub issue-automation workflows (dedup detection, lifecycle comments, auto-close, label editing).
- `.github/workflows/` — GitHub Actions that wire `@claude` mentions, issue triage, dedup, sweep/lifecycle automation, and issue locking.
- `examples/` — reference `settings.json` variants (strict/lax/bash-sandbox), an MDM-managed-settings example, and a hooks example (`bash_command_validator_example.py`).
- `.devcontainer/` — sandboxed devcontainer (with firewall init script) for running Claude Code itself in a container.

## Working with plugins

Each plugin under `plugins/<name>/` follows:
```
plugin-name/
├── .claude-plugin/plugin.json   # name, description, version, author (required)
├── commands/                    # slash commands (markdown, optional)
├── agents/                      # specialized subagents (markdown, optional)
├── skills/                      # Agent Skills (optional)
├── hooks/                       # event handlers, e.g. hooks.json + Python scripts (optional)
└── README.md                    # required — document all commands/agents/skills
```
When adding or modifying a plugin:
1. Update `plugin.json` (bump `version` on meaningful changes).
2. Update the plugin's own `README.md`.
3. Update the summary table in `plugins/README.md`.
4. Update the entry in `.claude-plugin/marketplace.json`.

The `hookify` plugin is the most complex example (Python `core/`, `matchers/`, `utils/` modules plus `hooks/*.py` driven by `hooks/hooks.json`) — look at it as a reference if building a plugin with real hook logic rather than just prompt-only commands/agents.

## Repo-automation scripts (`scripts/`)

- TypeScript files (`*.ts`) are run directly with Bun in CI: `bun run scripts/sweep.ts`, `bun run scripts/lifecycle-comment.ts`, etc. There is no `package.json`/build step — Bun executes the `.ts` files as-is.
- `scripts/gh.sh` is a restricted wrapper around the `gh` CLI: it only allows `issue view`, `issue list`, `search issues`, `label list` (plus a small allowed-flags list), and always scopes to the repo via `GH_REPO`/`GITHUB_REPOSITORY`. Repo-automation slash commands (`.claude/commands/*.md`) call this instead of raw `gh` or WebFetch, and declare it via `allowed-tools: Bash(./scripts/gh.sh:*)` in their frontmatter.
- `scripts/edit-issue-labels.sh` and `scripts/comment-on-duplicates.sh` are the other allowed write-paths for the triage/dedupe commands — issue number/labels are read from the workflow event, not passed as free-form args.

To run/test one of the automation scripts locally, use Bun directly, e.g.:
```bash
GITHUB_TOKEN=... GITHUB_REPOSITORY_OWNER=anthropics GITHUB_REPOSITORY_NAME=claude-code bun run scripts/sweep.ts
```

## Conventions for `.claude/commands/*.md` (repo automation)

- Declare `allowed-tools` in frontmatter scoped to the exact scripts needed (e.g. `Bash(./scripts/gh.sh:*)`), and instruct the command/agents not to use any other tool (no MCP, no direct file edits, no raw `gh`).
- These commands are designed to run unattended from GitHub Actions (see `.github/workflows/claude-dedupe-issues.yml`, `claude-issue-triage.yml`) — they must only take label/comment actions explicitly allowed by the workflow's `permissions:`.
- Label/category logic in `triage-issue.md` treats the issue body as authoritative over any dropdown/template field that might contradict it.

## GitHub Actions conventions

- Third-party actions are pinned to a commit SHA with a version comment (e.g. `actions/checkout@11bd71901bb...  # v4`), not a floating tag.
- The `@claude` mention workflow (`claude.yml`) authenticates via Workload Identity Federation (OIDC token exchange), not a static API key — see the `anthropic_federation_rule_id` / `anthropic_organization_id` / etc. inputs.
- Bun-based scripts always run via `oven-sh/setup-bun` then `bun run scripts/<file>.ts`.
