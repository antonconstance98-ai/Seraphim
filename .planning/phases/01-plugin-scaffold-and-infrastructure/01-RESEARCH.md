# Phase 1: Plugin Scaffold and Infrastructure - Research

**Researched:** 2026-04-04
**Domain:** Claude Code plugin system, Node.js file I/O, JSON config management
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**D-01:** `/seraphim:new-project` uses a guided setup flow with 4 questions: profile selection, project type (code/research/writing/science/mixed), opus_enabled toggle, and max_loops preference. Creates `.seraphim/config.json` with full config.

**D-02:** Users can create multiple named custom profiles with completely different model wiring. Not limited to one "custom" profile — users can create, name, and save as many profiles as they want.

**D-03:** The five built-in profiles (Performance, Balanced, Moderate, Budget, Frugal) are presets that ship with the plugin. Users can also create a "naked" profile where every slot is empty and the user fills in which model goes where.

**D-04:** Custom profiles are stored per-project in `.seraphim/profiles/` as individual JSON files (e.g., `my-research-profile.json`). Built-in profiles live in the plugin at `config/profiles.json`.

**D-05:** `/seraphim:set-profile` lists both built-in and user-created profiles for selection.

### Claude's Discretion

- Plugin directory layout (research found `.claude-plugin/plugin.json` for manifest — Claude follows research findings)
- dispatch.js internal architecture (resolution order: override > opus_enabled > profile is locked from roadmap)
- phase-state.js persistence format
- models.json schema design

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| PLUG-01 | Plugin loads in Claude Code with `/seraphim:` namespace via plugin manifest | Plugin manifest schema verified from 14 installed plugins; `.claude-plugin/plugin.json` path confirmed |
| PLUG-02 | `dispatch.js` routes any phase to the correct model executor based on profile, overrides, and opus_enabled flag | Resolution order locked; pseudo-code pattern verified against design spec |
| PLUG-03 | `profiles.json` defines all five profiles with ten sub-phase slots each | Ten sub-phase names extracted from design spec; all five profile tables verified |
| PLUG-04 | Per-project config at `.seraphim/config.json` stores profile, opus_enabled, overrides, and max_loops | Config schema confirmed in design spec; config.json format fully specified |
| PLUG-05 | `config.js` reads/writes `.seraphim/config.json` with validation and defaults | Node.js fs/promises pattern confirmed; validation approach from existing codex-pricing.js pattern |
| PLUG-06 | `phase-state.js` persists loop counters, retry counts, and phase completion to `.seraphim/phases/{N}/state.json` | Pitfall 6 documents exactly why in-memory fails; per-increment write pattern required |
| PLUG-07 | `models.json` defines all nine models with mechanism, pricing tier, and capability flags | Nine-model roster fully enumerated from design spec; schema fields identified |
| PROF-06 | Users can create named custom profiles with fully custom model wiring, stored per-project in `.seraphim/profiles/` | D-04 locked; per-project `.seraphim/profiles/` directory confirmed |
| PROF-07 | A "naked" empty profile template is available where every model slot is unassigned and the user fills in each one | D-03 locked; naked profile is a built-in template with all slots set to null |
</phase_requirements>

---

## Summary

Phase 1 delivers the foundational infrastructure for the Seraphim plugin: the manifest that makes `/seraphim:` commands appear in Claude Code, the dispatch router that translates phase names into executor calls, and the per-project config and state persistence layer that survives session restarts. All of these are pure Node.js with no npm dependencies beyond what already exists on this machine.

The Claude Code plugin system uses a specific directory convention: the plugin manifest lives at `.claude-plugin/plugin.json` inside the plugin root. This is verified by inspecting 14 installed plugins on this machine — every single one uses the `.claude-plugin/plugin.json` path, not `plugin.json` at root. Commands are `.md` files with YAML frontmatter in a `commands/` directory. Hooks are declared in `hooks/hooks.json` (auto-discovered by Claude Code — NOT referenced from `plugin.json`). The `${CLAUDE_PLUGIN_ROOT}` environment variable is available in all hook commands to reference the plugin's own files.

This phase creates only static config and pure-Node utility modules. No model executors are built here (Phase 2). The plugin loads into Claude Code and passes validation, dispatch resolves correctly for all three config levels, per-project config persists across sessions, and phase state survives crashes. The GSD fork at `~/.claude/hooks/` provides the codex-exec.js and minimax-exec.js patterns to replicate and several existing utilities to fork.

**Primary recommendation:** Build the `.claude-plugin/plugin.json` manifest first, validate it loads with `claude plugin validate`, then build dispatch.js + config.js + phase-state.js as a tight unit before adding commands.

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js built-ins | v22.22.0 (confirmed) | fs/promises, path, os | No install; already on machine; v22 has `fs.glob()` |
| JSON files | — | profiles.json, models.json, config.json, state.json | Native to Node; human-readable; zero parsing dependencies |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `openai` npm | 6.33.0 (at `~/.npm-global`) | Used by minimax-exec.js later | Phase 2 only; reference the global install path |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Plain JSON config | SQLite, YAML | JSON is zero-dependency, already established in this codebase |
| `fs.readFileSync` | `fs/promises` async | Async preferred for hooks; sync acceptable in dispatch.js since Node single-threaded |

**Installation:** None required for Phase 1. All dependencies are Node.js built-ins already on the machine.

**Version verification:** Node.js v22.22.0 confirmed live. No npm packages needed in Phase 1.

---

## Architecture Patterns

### Recommended Plugin Structure

```
~/.claude/plugins/seraphim/
├── .claude-plugin/
│   └── plugin.json              # MANIFEST — must be here, not at root
├── config/
│   ├── profiles.json            # Five preset profiles (ten slots each)
│   └── models.json              # Nine models with mechanism, pricing, capabilities
├── commands/
│   └── new-project.md           # /seraphim:new-project (Phase 1 scope)
├── executors/
│   └── dispatch.js              # Central router (Phase 1 scope)
├── hooks/
│   ├── hooks.json               # Hook declarations (auto-discovered — NOT in plugin.json)
│   └── session-start.js         # SessionStart hook stub
└── lib/
    ├── config.js                # Read/write .seraphim/config.json
    └── phase-state.js           # Phase progress tracker
```

Note: `agents/`, `executors/` (model executors), `tools/`, and `lib/pricing.js` are Phase 2+. Phase 1 creates the scaffold directories but only implements the files above.

### Pattern 1: Plugin Manifest

**What:** `plugin.json` declares only `name`, `version`, `description`, and `author`. It does NOT declare hooks.

**When to use:** Required for Claude Code to recognize the plugin and register `/seraphim:*` commands.

**Example (verified from 14 installed plugins on this machine):**
```json
{
  "name": "seraphim",
  "version": "3.0.0",
  "description": "Six-phase multi-model creative pipeline. Discover → Envision → Judge → Architect → Forge → Crucible.",
  "author": {
    "name": "Dragos",
    "email": ""
  }
}
```

Installed at: `~/.claude/plugins/seraphim/.claude-plugin/plugin.json`

### Pattern 2: Command Frontmatter

**What:** Each `/seraphim:*` command is a `.md` file with YAML frontmatter. The file body is the prompt Opus reads.

**When to use:** Every slash command the plugin exposes.

**Example (verified from feature-dev, pinecone, ralph-loop on this machine):**
```markdown
---
description: "Initialize a new Seraphim project with guided profile setup"
argument-hint: "[project-name]"
allowed-tools: ["Read", "Write", "Bash"]
---

# New Project Setup
...prompt body...
```

Optional frontmatter fields (verified from pinecone plugin):
- `model: claude-haiku-4-5` — overrides the model for this command
- `hide-from-slash-command-tool: "true"` — hides from slash command list

### Pattern 3: hooks/hooks.json (auto-discovered, NOT declared in plugin.json)

**What:** Claude Code auto-discovers `hooks/hooks.json` inside any installed plugin. Declaring it again in `plugin.json` causes a `conflicting manifests` error where hooks silently never fire.

**When to use:** For SessionStart and PostToolUse hooks.

**Example (verified from ralph-loop 1.0.0 on this machine):**
```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node \"${CLAUDE_PLUGIN_ROOT}/hooks/session-start.js\"",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

`${CLAUDE_PLUGIN_ROOT}` is set by Claude Code to the plugin's install directory. Also available: `${CLAUDE_PROJECT_ROOT}` for the user's project root.

### Pattern 4: Hook Output Contract (SessionStart)

**What:** Hooks output JSON to stdout. SessionStart hooks can inject additional context for Opus.

**When to use:** session-start.js to report pipeline status at session open.

**Example (verified from superpowers 5.0.7 and vercel plugin on this machine):**
```javascript
// SessionStart hook output — injects context visible to Opus
process.stdout.write(JSON.stringify({
  hookSpecificOutput: {
    hookEventName: 'SessionStart',
    additionalContext: '## Seraphim Status\n...'
  }
}));
process.exit(0);
```

Observer-only hooks (no output needed): `process.stdout.write('{}'); process.exit(0);`

### Pattern 5: dispatch.js Resolution Chain

**What:** Three-level config resolution. Resolution order is LOCKED: per-phase override > opus_enabled flag > profile preset.

**When to use:** Every phase execution call routes through dispatch.js.

```javascript
// dispatch.js — resolution chain (from design spec, approved)
function resolveExecutorId(phase, config) {
  const profiles = require('../config/profiles.json');
  const models   = require('../config/models.json');

  // Level 1: per-phase override in .seraphim/config.json
  if (config.overrides && config.overrides[phase]) {
    return config.overrides[phase];
  }

  // Level 2: profile preset (built-in or custom)
  let profileDef;
  if (config.profile === 'custom' || !profiles[config.profile]) {
    // Load named custom profile from .seraphim/profiles/{name}.json
    const customPath = path.join(config._projectRoot, '.seraphim', 'profiles', config.profile + '.json');
    profileDef = JSON.parse(fs.readFileSync(customPath, 'utf8'));
  } else {
    profileDef = profiles[config.profile];
  }

  let modelId = profileDef.phases[phase];

  // Level 3: opus_enabled:false → use profile's opusFallback
  if (!config.opus_enabled && models[modelId] && models[modelId].isOpus) {
    modelId = profileDef.opusFallback;
  }

  return modelId;  // executor ID string, e.g. 'minimax-m2.7'
}
```

### Pattern 6: phase-state.js Persistence

**What:** Loop counters and completion flags written to disk at every increment — never kept only in memory.

**When to use:** Any code path that increments a loop counter or marks phase completion.

**Example (required by Pitfall 6):**
```javascript
// phase-state.js — write at every increment
function incrementLoop(projectRoot, phaseId, loopType) {
  const statePath = path.join(projectRoot, '.seraphim', 'phases', phaseId, 'state.json');
  const state = readState(statePath);
  state.loops = state.loops || {};
  state.loops[loopType] = (state.loops[loopType] || 0) + 1;
  state.last_updated = new Date().toISOString();
  fs.writeFileSync(statePath, JSON.stringify(state, null, 2));
  return state.loops[loopType];
}
```

State file schema:
```json
{
  "phase_id": "01-feature-name",
  "created_at": "2026-04-04T00:00:00Z",
  "last_updated": "2026-04-04T01:00:00Z",
  "completed": false,
  "loops": {
    "judge_envision": 0,
    "crucible_forge": 0
  },
  "retries": {},
  "reset_at": "2026-04-04T00:00:00Z"
}
```

The `reset_at` field prevents old loop counts from prior milestone runs carrying over.

### Pattern 7: config.js Read/Write

**What:** Reads `.seraphim/config.json` relative to the project root, provides defaults, validates required fields.

**Example:**
```javascript
// config.js
const CONFIG_DEFAULTS = {
  profile: 'moderate',
  opus_enabled: true,
  overrides: {},
  max_loops: 2,
  project_type: 'code'
};

function read(projectRoot) {
  const configPath = path.join(projectRoot, '.seraphim', 'config.json');
  if (!fs.existsSync(configPath)) {
    return { ...CONFIG_DEFAULTS, _projectRoot: projectRoot };
  }
  const raw = JSON.parse(fs.readFileSync(configPath, 'utf8'));
  return { ...CONFIG_DEFAULTS, ...raw, _projectRoot: projectRoot };
}

function write(projectRoot, config) {
  const configPath = path.join(projectRoot, '.seraphim', 'config.json');
  const toWrite = { ...config };
  delete toWrite._projectRoot;  // don't persist internal field
  fs.mkdirSync(path.dirname(configPath), { recursive: true });
  fs.writeFileSync(configPath, JSON.stringify(toWrite, null, 2));
}
```

### Pattern 8: profiles.json Schema (Ten Sub-Phase Slots)

The ten sub-phase slot names used in `profiles.json` (extracted from design spec):

```json
{
  "performance": {
    "phases": {
      "discover_external":          "perplexity-sonar",
      "discover_internal":          "gemini-3.1-pro",
      "envision":                   "claude-opus-4-6",
      "judge":                      "gemini-3-flash",
      "architect":                  "claude-opus-4-6",
      "forge":                      "codex-gpt-5.4",
      "forge_checkpoint_runtime":   "claude-sonnet-4-6",
      "forge_checkpoint_static":    "minimax-m2.7",
      "crucible_verify":            "claude-opus-4-6",
      "crucible_adversarial":       "minimax-m2.7"
    },
    "opusFallback": "gemini-3.1-pro"
  }
}
```

The `"naked"` profile has all slots set to `null`. Users fill them in with `/seraphim:override`.

### Pattern 9: models.json Schema

Nine model entries, each with mechanism, pricing key, and capability flags:

```json
{
  "claude-opus-4-6": {
    "name": "Claude Opus 4.6",
    "mechanism": "native_session",
    "pricingKey": "claude-opus-4-6",
    "isOpus": true,
    "capabilities": ["tool_use", "streaming", "vision"]
  },
  "minimax-m2.7": {
    "name": "MiniMax M-2.7",
    "mechanism": "api_openai_sdk",
    "pricingKey": "minimax-m2.7",
    "isOpus": false,
    "temperature": 0.01,
    "capabilities": ["adversarial_review", "bug_scan"]
  },
  "codex-gpt-5.4": {
    "name": "Codex GPT-5.4",
    "mechanism": "cli_full_auto",
    "pricingKey": "codex-gpt-5.4",
    "isOpus": false,
    "capabilities": ["code_execution", "autonomous_forge"],
    "timeout_ms": 300000
  },
  "qwen-3.5-27b": {
    "name": "Qwen 3.5-27B",
    "mechanism": "local_ollama",
    "pricingKey": "qwen-3.5-27b",
    "isOpus": false,
    "capabilities": ["code_generation", "instruction_following"],
    "timeout_ms": 180000,
    "requiresGPU": true
  }
}
```

All nine models: `claude-opus-4-6`, `claude-sonnet-4-6`, `claude-haiku-4-5`, `codex-gpt-5.4`, `minimax-m2.7`, `gemini-3.1-pro`, `gemini-3-flash`, `qwen-3.5-27b`, `perplexity-sonar`.

### Pattern 10: /seraphim:new-project Guided Setup Flow

Four questions per D-01 (locked):
1. Profile selection (list: Performance, Balanced, Moderate, Budget, Frugal, or custom name)
2. Project type (code / research / writing / science / mixed)
3. opus_enabled toggle (yes/no)
4. max_loops preference (default: 2)

Output: creates `.seraphim/config.json` and `.seraphim/profiles/` directory.

### Anti-Patterns to Avoid

- **Declaring hooks in plugin.json:** Only use `hooks/hooks.json` for hook registration. Declaring them in both causes silent hook failure with no error at runtime.
- **Keeping loop counters in memory only:** Must write to `state.json` at every increment. In-memory counters reset on crash.
- **Sharing an `_projectRoot` field in serialized config:** Internal runtime field; always strip before writing to disk.
- **Using a single `computeCost()` across all nine models:** Each provider has an incompatible token schema. This is Phase 2 scope but the architecture decision must be made now in models.json.
- **Plugin manifest at root:** `plugin.json` at `~/.claude/plugins/seraphim/plugin.json` does not work. Must be at `~/.claude/plugins/seraphim/.claude-plugin/plugin.json`.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Profile validation | Custom schema validator | JSON schema check at read time + CONFIG_DEFAULTS merge | Profiles are small; full schema validation library is overkill |
| Config file watching | File watcher / inotify | Re-read config.json on each dispatch.js call | Config changes per-session; fresh read is simpler and correct |
| Hook registration | Custom settings.json editor | `hooks/hooks.json` auto-discovery | Claude Code handles registration; manual editing risks breaking existing hooks |
| Custom profile storage | Database, encrypted store | Flat JSON files in `.seraphim/profiles/` | Per D-04 (locked); simple, readable, diffable |

**Key insight:** The plugin system handles command registration, hook registration, and `${CLAUDE_PLUGIN_ROOT}` injection automatically. Don't replicate this in code.

---

## Common Pitfalls

### Pitfall 1: Plugin Manifest in Wrong Location (Silent Failure)

**What goes wrong:** Plugin appears in `enabled_plugins` but `/seraphim:` commands never appear in Claude Code's command list. No error is surfaced.

**Why it happens:** `plugin.json` placed at `~/.claude/plugins/seraphim/plugin.json` (root) instead of `~/.claude/plugins/seraphim/.claude-plugin/plugin.json`. Verified: ALL 14 installed plugins on this machine use the `.claude-plugin/` subdirectory.

**How to avoid:** Create `.claude-plugin/` directory first. Run `claude plugin validate` after creating the manifest and before testing anything else.

**Warning signs:** Plugin appears in `enabledPlugins` in settings.json but no `/seraphim:` prefix shows in Claude Code.

---

### Pitfall 2: Hook Double-Registration Silences All Hooks

**What goes wrong:** `plugin.json` references `hooks/hooks.json`. Claude Code also auto-discovers `hooks/hooks.json`. Duplicate registration causes `conflicting manifests` error — all plugin hooks silently drop.

**Why it happens:** Developers coming from the global `~/.claude/settings.json` hook system assume hooks must be explicitly declared.

**How to avoid:** `plugin.json` declares NOTHING about hooks. `hooks/hooks.json` is the only hook registration point. Verify with `claude --debug` on first install.

**Warning signs:** Hooks never fire despite being syntactically correct and executable.

---

### Pitfall 3: dispatch.js Resolves Wrong Profile for Named Custom Profiles

**What goes wrong:** User creates `~/.seraphim/profiles/my-research.json` but dispatch.js only checks the built-in `profiles.json`. User sets `"profile": "my-research"` and gets "profile not found" error.

**Why it happens:** dispatch.js treats `config.profile` as a key into the bundled `profiles.json` only.

**How to avoid:** dispatch.js must check if `config.profile` exists in the bundled profiles. If not, it falls back to loading from `.seraphim/profiles/{config.profile}.json` in the project root. If that also fails, return a clear error: `"Profile 'my-research' not found in plugin config or .seraphim/profiles/"`.

**Warning signs:** Works for Performance/Balanced/Moderate/Budget/Frugal, fails for any custom profile name.

---

### Pitfall 4: state.json Loop Counter Carries Over Across Pipeline Runs

**What goes wrong:** A previous pipeline run for phase N hit the loop cap. Next run, phase-state.js reads the persisted count and immediately surfaces "loop cap exceeded" before any work is done.

**Why it happens:** state.json is keyed by phase directory (`phases/01-feature-name/`), not by pipeline run number. A new `/seraphim:run` invocation on the same phase number reuses the old state.

**How to avoid:** `/seraphim:new-project` and new pipeline runs must reset state.json by writing a fresh file with `reset_at` updated. phase-state.js should expose a `reset(projectRoot, phaseId)` function that the run command calls at pipeline start.

---

### Pitfall 5: Opus-Disabled Fallback Missing for a Profile

**What goes wrong:** `opus_enabled: false` is set. dispatch.js looks up `profileDef.opusFallback` but the field doesn't exist in a custom profile JSON the user created. dispatch.js crashes with `Cannot read property of undefined`.

**Why it happens:** Custom profile JSON validation is not enforced at creation time.

**How to avoid:** `config.js` validates custom profile JSON at load time. Required fields: `phases` object with all ten keys, `opusFallback` string. Missing fields produce a clear error message at config load, not a crash in dispatch.js.

---

## Code Examples

### Verified: Command Frontmatter (ralph-loop, confirmed on this machine)

```markdown
---
description: "Initialize a new Seraphim project with guided profile setup"
argument-hint: "[project-name]"
allowed-tools: ["Read", "Write", "Bash", "TodoWrite"]
---
```

### Verified: hooks.json Format (ralph-loop 1.0.0, confirmed on this machine)

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node \"${CLAUDE_PLUGIN_ROOT}/hooks/session-start.js\"",
            "timeout": 10
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Bash|Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "node \"${CLAUDE_PLUGIN_ROOT}/hooks/token-logger.js\"",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

### Verified: SessionStart Hook Output (superpowers 5.0.7, confirmed on this machine)

```javascript
// Node.js hook — session-start.js
'use strict';
const stdinTimeout = setTimeout(() => { process.stdout.write('{}'); process.exit(0); }, 10000);
let input = '';
process.stdin.setEncoding('utf8');
process.stdin.on('data', chunk => { input += chunk; });
process.stdin.on('end', () => {
  clearTimeout(stdinTimeout);
  try {
    const data = JSON.parse(input || '{}');
    const cwd = data.cwd || process.cwd();
    // ... load config, build status message ...
    const context = buildSeraphimStatus(cwd);
    process.stdout.write(JSON.stringify({
      hookSpecificOutput: {
        hookEventName: 'SessionStart',
        additionalContext: context
      }
    }));
  } catch (e) {
    process.stdout.write('{}');
  }
  process.exit(0);
});
```

### Verified: Plugin Manifest Fields (inspected 14 plugins on this machine)

Minimal valid `plugin.json`:
```json
{
  "name": "seraphim",
  "version": "3.0.0",
  "description": "Six-phase multi-model creative pipeline.",
  "author": { "name": "Dragos" }
}
```

Optional fields observed in other plugins: `homepage`, `repository`, `license`, `keywords` (array of strings).

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Register hooks in `plugin.json` | `hooks/hooks.json` auto-discovery | Claude Code plugin system v2+ | Explicit declaration in plugin.json causes double-registration error |
| Hooks registered in global `~/.claude/settings.json` | Plugin-scoped `hooks/hooks.json` | Plugin system adoption | Plugin hooks are sandboxed to plugin root; `${CLAUDE_PLUGIN_ROOT}` available |
| Plain `plugin.json` at repo root | `.claude-plugin/plugin.json` subdirectory | Claude Code plugin convention | All 14 installed plugins on this machine use `.claude-plugin/` |

**Deprecated/outdated:**
- Root-level `plugin.json` (from early Claude Code plugin docs): replaced by `.claude-plugin/plugin.json` in actual installed plugins
- Explicit `"hooks"` key in `plugin.json`: causes `conflicting manifests` error; hooks.json is auto-discovered

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | All hook scripts, dispatch.js, config.js, phase-state.js | Yes | v22.22.0 | — |
| npm | Package management (Phase 2+) | Yes | 10.9.4 | — |
| `openai` npm global | minimax-exec.js (Phase 2) | Yes | 6.33.0 at `~/.npm-global` | — |
| Claude Code plugin system | Plugin manifest, command registration | Yes | Confirmed (14 plugins installed) | — |
| `claude plugin validate` | Manifest validation | Yes | Available in current Claude Code | — |

**Missing dependencies with no fallback:** None for Phase 1.

**Missing dependencies with fallback:** None for Phase 1.

---

## Open Questions

1. **Does `claude plugin validate` check command `.md` file syntax or only `plugin.json`?**
   - What we know: `claude plugin validate` is the prescribed validation command; verified it exists
   - What's unclear: Whether it validates command frontmatter fields or only the manifest JSON
   - Recommendation: Run it after each file creation; treat failures as blockers

2. **Does `enabledPlugins` in `~/.claude/settings.json` need to be manually edited for a locally installed plugin, or does `claude plugin install` handle it?**
   - What we know: `enabledPlugins` contains entries for all 14 installed marketplace plugins
   - What's unclear: Whether a manually placed plugin directory auto-registers or requires a settings.json edit
   - Recommendation: Plan a task to add `"seraphim@local": true` to `enabledPlugins` and test; if `claude plugin install ./path` handles it automatically, that is simpler

3. **Is there a `claude plugin install` command for local directories?**
   - What we know: Marketplace plugins install via `claude plugin install name@marketplace`
   - What's unclear: Whether local path installation is supported (e.g., `claude plugin install ./` from plugin root)
   - Recommendation: Test both approaches; manual settings.json edit is the reliable fallback

---

## Sources

### Primary (HIGH confidence)

- Live inspection of 14 installed plugins at `~/.claude/plugins/cache/claude-plugins-official/` on this machine — plugin.json schema, hooks.json format, command frontmatter, directory layout
- `/home/alucard/.claude/plugins/cache/claude-plugins-official/ralph-loop/1.0.0/` — hooks.json auto-discovery pattern, hooks/hooks.json schema
- `/home/alucard/.claude/plugins/cache/claude-plugins-official/superpowers/5.0.7/` — SessionStart hook output format, `additionalContext` field
- `/home/alucard/.claude/plugins/cache/claude-plugins-official/vercel/f7814a911ef2/docs/hook-lifecycle.md` — `CLAUDE_PLUGIN_ROOT` env var, Hook I/O contract, hook event names
- `docs/specs/2026-04-04-seraphim-v3-design.md` — five profile tables, ten sub-phase slot names, config.json schema, opus-disabled fallback table
- `.planning/research/ARCHITECTURE.md` — component responsibility table, unified executor interface contract
- `.planning/research/PITFALLS.md` — Pitfall 1 (manifest path), Pitfall 2 (double-registration), Pitfall 6 (loop counter persistence)
- `/home/alucard/.claude/hooks/codex-exec.js` — executor module pattern to fork
- `/home/alucard/.claude/hooks/minimax-exec.js` — OpenAI SDK baseURL swap pattern, exponential backoff pattern
- `/home/alucard/.claude/hooks/codex-pricing.js` — per-provider pricing function pattern
- `/home/alucard/.claude/hooks/codex-token-logger.js` — stdin/stdout/timeout guard hook pattern

### Secondary (MEDIUM confidence)

- `.planning/research/STACK.md` — Node.js v22 built-ins verification, Chart.js pattern (not Phase 1 scope but confirms no npm needed)
- `/home/alucard/.claude/settings.json` — current hook registration format in `hooks.SessionStart`/`hooks.PostToolUse`, confirming the timeout field syntax

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — verified against live Node.js installation and existing codebase
- Architecture: HIGH — verified against 14 installed plugin structures on this machine + design spec
- Pitfalls: HIGH — three of five pitfalls come directly from the existing `.planning/research/PITFALLS.md` which was verified against official Claude Code docs; two additional pitfalls from code inspection

**Research date:** 2026-04-04
**Valid until:** 2026-05-04 (plugin manifest schema is stable; hooks.json convention unlikely to change)
