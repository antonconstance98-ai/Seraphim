# Claude Workflow Installer — Design Spec

**Date:** 2026-04-04
**Purpose:** Package the entire Claude Code + Codex + MiniMax multi-model workflow into a single installable repo that can be deployed on another Ubuntu/Linux machine with Claude Code already installed.

---

## Overview

A Node.js-based layered installer that reads a `manifest.json` to copy configuration files, hook scripts, the GSD engine, and project state to the correct locations on a target machine. Plugins are pulled from marketplaces (not bundled). API keys are prompted interactively during install.

## Target Environment

- Ubuntu/Linux with Claude Code already installed
- Node.js v22+ already available
- User has API keys for OpenAI, MiniMax, and optionally Perplexity

## Repo Structure

```
claude-workflow-installer/
├── manifest.json              # What goes where (every file mapped)
├── install.js                 # Main installer entry point (bin)
├── lib/
│   ├── layers.js              # Layer execution logic
│   ├── prompts.js             # Interactive prompts (API keys, paths)
│   ├── validators.js          # Pre-flight checks
│   └── fs-utils.js            # Safe copy/backup helpers
├── layers/
│   ├── core/                  # settings.json, settings.local.json, CLAUDE.md
│   ├── hooks/                 # All 28 hook scripts
│   ├── gsd/                   # get-shit-done/ engine (v1.30.0)
│   ├── plugins/               # Plugin registry config
│   ├── project/               # Claude_X_Codex project snapshot
│   │   ├── .claude/           # Project-level settings
│   │   ├── .planning/         # Full planning history
│   │   ├── AGENTS.md
│   │   ├── CLAUDE.md
│   │   └── .mcp.json
│   └── secrets/               # .env.example
├── skills/                    # Custom skills (dispatching-parallel-agents)
├── commands/                  # Custom commands (gsd/workstreams.md)
├── package.json
└── README.md
```

## Manifest Format

```json
{
  "version": "1.0.0",
  "layers": [
    {
      "name": "core",
      "description": "Claude Code global configuration",
      "order": 1,
      "files": [
        { "src": "layers/core/settings.json", "dest": "~/.claude/settings.json", "backup": true },
        { "src": "layers/core/settings.local.json", "dest": "~/.claude/settings.local.json", "backup": true },
        { "src": "layers/core/CLAUDE.md", "dest": "~/.claude/CLAUDE.md", "backup": true, "template": true }
      ]
    },
    {
      "name": "hooks",
      "description": "Lifecycle hook scripts (28 files)",
      "order": 2,
      "files": [
        { "src": "layers/hooks/", "dest": "~/.claude/hooks/", "glob": "*.js", "backup": true }
      ]
    },
    {
      "name": "gsd",
      "description": "Get Shit Done workflow engine v1.30.0",
      "order": 3,
      "files": [
        { "src": "layers/gsd/", "dest": "~/.claude/get-shit-done/", "recursive": true, "backup": true }
      ]
    },
    {
      "name": "skills",
      "description": "Custom skills",
      "order": 4,
      "files": [
        { "src": "skills/", "dest": "~/.claude/skills/", "recursive": true, "backup": true }
      ]
    },
    {
      "name": "commands",
      "description": "Custom commands",
      "order": 5,
      "files": [
        { "src": "commands/", "dest": "~/.claude/commands/", "recursive": true, "backup": true }
      ]
    },
    {
      "name": "plugins",
      "description": "Install plugins from marketplaces",
      "order": 6,
      "marketplace": [
        "superpowers",
        "context7",
        "code-review",
        "frontend-design",
        "github",
        "feature-dev",
        "code-simplifier",
        "ralph-loop",
        "playwright",
        "claude-md-management",
        "agent-sdk-dev",
        "claude-code-setup",
        "plugin-dev",
        "pinecone",
        "vercel",
        "skill-creator",
        "codex@openai-codex"
      ]
    },
    {
      "name": "project",
      "description": "Claude_X_Codex project with full planning history",
      "order": 7,
      "files": [
        { "src": "layers/project/", "dest": "~/projects/Claude_X_Codex/", "recursive": true }
      ]
    },
    {
      "name": "secrets",
      "description": "API key configuration",
      "order": 8,
      "envVars": [
        { "key": "OPENAI_API_KEY", "description": "OpenAI API key for Codex/GPT-5.4", "required": true },
        { "key": "MINIMAX_API_KEY", "description": "MiniMax M2.7 API key", "required": true },
        { "key": "PERPLEXITY_SESSION_TOKEN", "description": "Perplexity MCP session token", "required": false },
        { "key": "PERPLEXITY_CSRF_TOKEN", "description": "Perplexity MCP CSRF token", "required": false }
      ],
      "target": "~/.bashrc"
    }
  ]
}
```

## Installer Behavior

### Pre-flight Checks (`validators.js`)

Before any installation:
1. Node.js >= 22 installed
2. `~/.claude/` directory exists (Claude Code is installed)
3. `claude` CLI is on PATH
4. Sufficient disk space (~500MB)
5. Not running as root

Fail fast with clear error messages if any check fails.

### Layer Execution (`layers.js`)

Each layer runs in order. For each layer:

1. Print layer name and description
2. For file layers:
   - If `backup: true` and destination exists, copy to `{dest}.backup-{ISO-timestamp}`
   - Copy source to destination, creating parent dirs as needed
   - If `template: true`, print a reminder that the file needs manual editing
3. For plugin layers:
   - Run `claude plugin install {name}` for each plugin
   - Skip if already installed (check exit code)
   - Report which plugins were installed vs skipped
4. For secret layers:
   - Check if env var already exists in target file
   - If not, prompt user for value
   - Append `export KEY="value"` to target file
   - Skip optional vars if user presses Enter

### Progress Output

Each step prints:
```
[1/8] Core configuration...
  -> Backed up ~/.claude/settings.json
  -> Copied settings.json
  -> Copied settings.local.json
  -> Copied CLAUDE.md (TEMPLATE - edit machine specs after install)
  [OK]
```

### Error Handling

- File copy failures: print error, ask to continue or abort
- Plugin install failures: log warning, continue (plugins can be installed manually)
- Secret prompt failures: skip, remind user to set manually
- All backups are timestamped so multiple runs don't overwrite previous backups

### Post-Install Summary

```
Installation complete!

Installed:
  [OK] Core configuration (3 files)
  [OK] Hooks (28 scripts)
  [OK] GSD engine (v1.30.0)
  [OK] Skills (1 custom skill)
  [OK] Commands (1 custom command)
  [OK] Plugins (17 installed, 0 skipped)
  [OK] Project (Claude_X_Codex with planning history)
  [OK] Secrets (2 configured, 2 skipped)

Action items:
  1. Edit ~/.claude/CLAUDE.md — update machine specs for this system
  2. Run: source ~/.bashrc
  3. Test: claude --version
  4. Test: codex --version
```

## What Gets Copied (Source Inventory)

### From `~/.claude/`
- `settings.json` (176 lines — permissions, plugins, hooks config)
- `settings.local.json` (local permission overrides)
- `CLAUDE.md` (workstation context — marked as template)

### From `~/.claude/hooks/` (28 scripts)
**GSD hooks:** gsd-check-update.js, gsd-context-monitor.js, gsd-prompt-guard.js, gsd-statusline.js, gsd-workflow-guard.js
**Codex hooks:** codex-cost-reporter.js, codex-global-aggregator.js, codex-token-logger.js, codex-router.js, codex-wave-validator.js, codex-wave-validator-worker.js, codex-exec.js, codex-plan-reviewer.js, codex-superpowers-plan-reviewer.js, codex-review-gate.js, codex-handoff.js, codex-multi-round-reviewer.js, codex-dashboard-generator.js
**MiniMax hooks:** minimax-exec.js, minimax-post-scan.js, minimax-compress.js, minimax-connectivity-test.js
**Utility hooks:** claude-settings-guard.js, decision-logger.js, hook-signal.js, migrate-opus-pricing.js

### From `~/.claude/get-shit-done/` (133 files)
- `bin/lib/` — 18 core CJS modules
- `commands/` — 72 workflow commands
- `templates/` — 43 templates
- `references/` — 16 guides
- `package.json`, `VERSION`, `gsd-file-manifest.json`

### From `~/.claude/skills/`
- `dispatching-parallel-agents/SKILL.md`

### From `~/.claude/commands/`
- `gsd/workstreams.md`

### From `~/projects/Claude_X_Codex/`
- `.claude/settings.json` (project-level Codex routing config)
- `.planning/` (full directory with all phases, decision logs, session reports)
- `AGENTS.md`, `CLAUDE.md`, `.mcp.json`, `.codex`
- `docs/`, `research/`, `*.md` research files

## What Does NOT Get Copied

- `~/.claude/.credentials.json` — encrypted, machine-specific
- `~/.claude/history.jsonl` — session history, not portable
- `~/.claude/sessions/` — session archives
- `~/.claude/projects/` — other project contexts (only Claude_X_Codex)
- `~/.claude/plugins/cache/` — plugins reinstalled from marketplace
- `~/.claude/file-history/`, `shell-snapshots/`, `debug/` — ephemeral
- `~/.codex/` — OpenClaw config (separate system)
- `~/.npm-global/` — Vercel CLI etc (installed separately)
- Any `.env` files or plaintext secrets

## Dependencies

The installer itself needs only Node.js built-in modules:
- `fs` / `fs/promises` — file operations
- `path` — path resolution
- `os` — home directory
- `child_process` — plugin installs via `claude` CLI
- `readline` — interactive prompts

Zero npm dependencies. Runs with `node install.js`.
