# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is the public `claude-code` repository. It does **not** contain the Claude Code CLI source. It holds:

- **`plugins/`** + **`.claude-plugin/marketplace.json`**: the bundled plugin marketplace (`claude-code-plugins`).
- **`scripts/`** + **`.github/workflows/`**: GitHub issue-management automation (triage, dedupe, lifecycle, sweep).
- **`examples/`**: reference configs (enterprise/MDM managed settings, settings presets, a hook example, a gateway setup).
- **`.devcontainer/`**: a sandboxed devcontainer with a firewall (`init-firewall.sh`) for running Claude Code.
- `CHANGELOG.md` / `feed.xml`: generated release notes. They are updated by automated `chore:` commits, so don't hand-edit them.

There is no top-level `package.json`, build or test suite.

## Plugins

Every plugin under `plugins/<name>/` follows the standard Claude Code plugin layout:

- `.claude-plugin/plugin.json`: the manifest (name, version, description, author).
- Optional `commands/*.md`, `agents/*.md`, `skills/<skill>/SKILL.md`, and `hooks/hooks.json` (hook commands reference `${CLAUDE_PLUGIN_ROOT}`).
- `README.md`.

When you add or rename a plugin, also update **both** `.claude-plugin/marketplace.json` (the `plugins` array, `source: "./plugins/<name>"`) and the table in `plugins/README.md`.

Plugins with real code (not just markdown prompts):
- `hookify`: Python. `hooks/{pretooluse,posttooluse,stop,userpromptsubmit}.py` load user rules from `.local.md` files (`core/config_loader.py`) and evaluate them in `core/rule_engine.py`. Example rules are in `examples/`.
- `security-guidance`: a Python PreToolUse hook (`hooks/security_reminder_hook.py`, patterns in `patterns.py`).
- `ralph-wiggum`: a bash Stop hook (`hooks/stop-hook.sh`) that re-feeds the prompt to loop.
- `explanatory-output-style` / `learning-output-style`: SessionStart hooks in `hooks-handlers/`.

Validation helpers live in `plugins/plugin-dev/skills/*/scripts/`:
```bash
plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh path/to/hooks.json
plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh <hook-script> <test-input.json>
plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh <hook-script.sh>
plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh path/to/agent.md
plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh <file>
```

## Issue automation

These scripts run in GitHub Actions with **Bun** (`bun run scripts/<name>.ts`) and need `GITHUB_TOKEN`:

- `issue-lifecycle.ts`: the single source of truth for lifecycle labels (`invalid`, `needs-repro`, `needs-info`, `stale`, `autoclose`), their timeouts and nudge messages. `sweep.ts` (runs twice daily) and `lifecycle-comment.ts` (runs on `issues: labeled`) both import from it, so change label behavior here.
- `auto-close-duplicates.ts` (daily) and `backfill-duplicate-comments.ts` (manual; `DRY_RUN=true` by default).
- `sweep.ts` and `lifecycle-comment.ts` accept `--dry-run`. Test changes locally that way, e.g. `GITHUB_TOKEN=... bun run scripts/sweep.ts --dry-run`.

The Claude-driven workflows (`claude-issue-triage.yml`, `claude-dedupe-issues.yml`) run `anthropics/claude-code-action` using the prompts in `.claude/commands/` (`triage-issue.md`, `dedupe.md`). Those prompts may call only:
- `scripts/gh.sh`: a locked-down `gh` wrapper that allows only `issue view|list`, `search issues` and `label list`, plus a small flag allowlist. It requires `GH_REPO` or `GITHUB_REPOSITORY`.
- `scripts/edit-issue-labels.sh`: it reads the issue number from `$GITHUB_EVENT_PATH`, not from arguments.
- `scripts/comment-on-duplicates.sh`.

Keep this least-privilege design: don't widen `gh.sh`'s allowlist or the commands' `allowed-tools` casually. PRs that touch `.github/**` trigger `non-write-users-check.yml`.
