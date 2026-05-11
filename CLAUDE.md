# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is the official Claude Code repository — primarily a documentation and plugin showcase repo, not a compiled application. It contains:
- Example plugins demonstrating Claude Code's plugin system
- Example settings/MDM deployment configurations
- GitHub automation scripts (issue triage, deduplication, lifecycle management)
- A devcontainer setup for sandboxed development

There is no build step, no test suite, and no package.json at the root.

## Key Directories

- `plugins/` — Official plugin examples, each self-contained with their own `.claude-plugin/plugin.json`
- `examples/settings/` — Reference settings JSON files for enterprise deployments
- `examples/mdm/` — MDM deployment templates (Jamf, Intune, Group Policy)
- `examples/hooks/` — Example hook scripts
- `scripts/` — GitHub automation TypeScript scripts run via Bun
- `.claude/commands/` — Project-level slash commands (triage-issue, dedupe, commit-push-pr)
- `.github/workflows/` — CI/CD using `anthropics/claude-code-action@v1`

## GitHub Automation

The `scripts/` directory contains TypeScript scripts run via **Bun** (not Node):

```bash
bun run scripts/sweep.ts        # Enforce issue lifecycle timeouts (runs on cron)
bun run scripts/sweep.ts --dry-run
```

Scripts use the GitHub REST API directly via `GITHUB_TOKEN`. The `issue-lifecycle.ts` file is the single source of truth for lifecycle labels, timeouts, and messages — edit there when changing lifecycle behavior.

The restricted `./scripts/gh.sh` wrapper is used by Claude workflows to sandbox `gh` CLI access — only `issue view`, `issue list`, `search issues`, and `label list` subcommands are permitted.

## Plugin Architecture

Each plugin in `plugins/` follows this structure:
```
plugin-name/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest
├── commands/                # Slash commands as .md files with YAML frontmatter
├── agents/                  # Specialized agent .md files with YAML frontmatter
├── skills/                  # Skill .md files loaded contextually
├── hooks/
│   ├── hooks.json           # Hook registration (uses ${CLAUDE_PLUGIN_ROOT} for portable paths)
│   └── *.py / *.sh          # Hook scripts
└── README.md
```

**Hook scripts** receive JSON on stdin and communicate via exit codes:
- Exit 0: allow
- Exit 1: show stderr to user only (not blocked)
- Exit 2: block the operation and show stderr to Claude

**Hooks use `${CLAUDE_PLUGIN_ROOT}`** for portable paths — never hardcode absolute paths in `hooks.json`.

**Slash commands** are markdown files whose YAML frontmatter specifies `allowed-tools`, `description`, and optionally `argument-hint`. Dynamic context can be injected with `!` prefix shell commands.

**Skills** use a three-level progressive disclosure model: YAML metadata (always loaded) → core SKILL.md (triggered by keywords) → reference docs (on demand). Trigger phrases in the metadata determine when a skill auto-loads.

## Workflow Automation (GitHub Actions)

Workflows use `anthropics/claude-code-action@v1` to run Claude on GitHub events:

- **claude.yml**: Responds when `@claude` is mentioned in issues/PRs/comments (read-only permissions)
- **claude-issue-triage.yml**: Runs `/triage-issue` on new issues; uses `claude-opus-4-6`
- **claude-dedupe-issues.yml**: Runs `/dedupe` on new issues; uses `claude-sonnet-4-5-20250929`
- **sweep.yml**: Cron (twice daily) — runs `bun run scripts/sweep.ts` to enforce lifecycle timeouts
- **lock-closed-issues.yml**: Cron (daily) — locks closed issues after 7 days of inactivity

When adding new Claude-powered workflows, follow the existing pattern: use `allowed_non_write_users: "*"` only when necessary, and be aware the `non-write-users-check.yml` workflow will flag it for security review.

## Issue Lifecycle

Defined in `scripts/issue-lifecycle.ts`. Labels and their auto-close timeouts:
- `invalid` → 3 days
- `needs-repro` / `needs-info` → 7 days (bugs only)
- `stale` → 14 days
- `autoclose` → 14 days (then closed)

Issues with ≥10 upvotes (`STALE_UPVOTE_THRESHOLD`) are exempt from auto-staling.

## Adding a New Plugin

1. Create `plugins/your-plugin-name/` following the standard structure above
2. Add `.claude-plugin/plugin.json` with manifest metadata
3. Register it in `.claude-plugin/marketplace.json` at the repo root
4. Add an entry to the table in `plugins/README.md`
5. Use `${CLAUDE_PLUGIN_ROOT}` in `hooks.json` for all paths

## Devcontainer

The devcontainer (`/.devcontainer/`) provides a sandboxed Linux environment with Claude Code pre-installed, a network firewall via `init-firewall.sh`, and VS Code extensions (ESLint, Prettier, GitLens, Claude Code). The workspace is mounted at `/workspace`. Use this for testing plugins and hook scripts in isolation.
