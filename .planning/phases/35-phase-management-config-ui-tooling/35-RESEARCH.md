# Phase 35: Phase Management + Config + UI Tooling - Research

**Researched:** 2026-04-10
**Domain:** Seraphim plugin command development — markdown lifecycle management, config system, UI auditing
**Confidence:** HIGH

## Summary

Phase 35 builds 13 commands across three domains: phase lifecycle management (add/insert/remove/complete-milestone/pr-branch/health/workstreams/manager), configuration (settings, set-profile extension), and UI/quality tooling (ui-spec, ui-review, add-tests). All commands follow the established Seraphim `.md` skill format with YAML frontmatter, `node -e` inline scripts calling lib files, and subagent dispatch for complex operations.

The key technical challenge is ROADMAP.md manipulation — the planning markdown file (not roadmap.json) must be parsed, mutated, and written back atomically. Phase numbering, dependency references, and the progress table all need updating together. This is a text transformation problem, not a database operation.

The configuration domain is well-served by the existing `lib/config.js` (read/write/validate) and `set-profile.md` patterns. The `/seraphim:settings` command is a UI wrapper that reads config, presents toggles, and writes back — following the same pattern as `/seraphim:set-profile`.

**Primary recommendation:** Group plans by domain (phase-lifecycle lib + ROADMAP.md manipulation, config commands, UI commands) and implement in 4 waves: a shared lib module for ROADMAP.md parsing (reused by add/insert/remove/complete), then three parallel waves of commands.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** add/insert/remove-phase directly manipulate ROADMAP.md — parse markdown, insert/remove phase section, renumber subsequent phases, update dependency references. No intermediate JSON.
- **D-02:** `/seraphim:complete-milestone` archives to `.planning/milestones/vX.Y-ROADMAP.md` + git tag + cost attribution from session JSONL. Cleans up phase directories. Matches GSD complete-milestone pattern.
- **D-03:** `/seraphim:pr-branch` cherry-picks non-.planning/ commits to a clean branch. User gets a PR-ready branch without planning artifacts.
- **D-04:** Workstreams use `.planning/workstreams/` directory with state files per workstream, each tracking independent phase progress.
- **D-05:** Model profiles stored as presets in config.json — quality/balanced/budget/inherit profiles map to model selections per command type (planner, executor, reviewer). `/seraphim:settings` writes profile choice.
- **D-06:** Workflow settings as toggle map — `skip_discuss`, `auto_advance`, `ui_phase`, `ui_review`, `plan_checker`, `research_enabled`, `parallelization`. All read via config-get, written via `/seraphim:settings`.
- **D-07:** `/seraphim:health` validates `.planning/` structure — checks for ROADMAP.md, STATE.md, REQUIREMENTS.md, phase directories, orphaned plans. Reports issues with repair suggestions.
- **D-08:** `/seraphim:ui-spec` produces UI-SPEC.md with layout wireframes (ASCII), component list, interaction patterns, responsive breakpoints, accessibility notes. Consumed by executors as design contract.
- **D-09:** `/seraphim:ui-review` runs 6-pillar audit — layout, typography, color, spacing, accessibility, responsiveness. Produces scored UI-REVIEW.md. Advisory, not blocking.
- **D-10:** `/seraphim:add-tests` generates test files based on VERIFICATION.md acceptance criteria and PLAN.md done-criteria. Auto-detects test framework (jest/vitest).

### Claude's Discretion
- Internal prompt structures, error messages, manager UI layout, and display formatting.

### Deferred Ideas (OUT OF SCOPE)
- None.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| MGMT-01 | User can add a phase to end of milestone via `/seraphim:add-phase` | ROADMAP.md append + renumber pattern |
| MGMT-02 | User can insert urgent decimal phase between existing phases via `/seraphim:insert-phase` | ROADMAP.md insert + decimal numbering |
| MGMT-03 | User can remove an unstarted phase via `/seraphim:remove-phase` with renumbering | ROADMAP.md remove + renumber pattern |
| MGMT-04 | User can complete milestone via `/seraphim:complete-milestone` with archival and git tagging | Archive to .planning/milestones/, git tag, cost from JSONL |
| MGMT-05 | User can create clean PR branch filtering .planning/ via `/seraphim:pr-branch` | git cherry-pick pattern with .planning/ exclusion |
| MGMT-06 | User can validate .planning/ directory integrity via `/seraphim:health` | Directory scan + presence checks |
| MGMT-07 | User can manage parallel workstreams via `/seraphim:workstreams` | .planning/workstreams/ state files |
| MGMT-08 | User can manage phases from interactive command center via `/seraphim:manager` | Interactive TUI-style display in terminal |
| CFG-01 | Model profiles (quality/balanced/budget/inherit) control agent routing per command | config.json profile presets |
| CFG-02 | User can configure workflow settings via `/seraphim:settings` | config.json write via lib/config.js |
| UI-01 | User can generate UI design contract via `/seraphim:ui-spec` for frontend phases | UI-SPEC.md with ASCII wireframes |
| UI-02 | User can run retroactive 6-pillar UI audit via `/seraphim:ui-review` | UI-REVIEW.md with scores |
| UI-03 | User can generate tests for completed phase via `/seraphim:add-tests` | Test file generation from VERIFICATION.md |
</phase_requirements>

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js | v22.22.0 (installed) | All lib and inline scripts | Existing runtime — all hooks and commands use it |
| `fs` (built-in) | — | Atomic file writes (tmp+rename) | Zero deps, pattern established in all lib files |
| `path` (built-in) | — | Cross-platform path resolution | Established pattern throughout codebase |
| `child_process` (built-in) | — | `git` subprocess calls (pr-branch, complete-milestone) | Used in existing hooks and commands |

### Established Lib Files (Reuse)

| Lib | Export | Phase 35 Use |
|-----|--------|--------------|
| `lib/config.js` | `read(root)`, `write(root, config)`, `validate(config)` | Settings command reads/writes workflow toggles and profile |
| `lib/roadmap.js` | `readRoadmap(root)`, `writeRoadmap(root, data)` | complete-milestone reads .seraphim/roadmap.json for cost attribution |
| `lib/requirements.js` | REQ-ID CRUD | remove-phase checks for orphaned requirement references |

**No new npm dependencies.** This is a zero-dependency constraint from project decisions.

## Architecture Patterns

### Established Command File Pattern

Every command in `~/.claude/plugins/seraphim/commands/` follows this structure:

```markdown
---
description: "Short description"
argument-hint: "[args]"
allowed-tools: ["Read", "Write", "Bash"]
---

## Step 1 — Parse arguments
## Step 2 — Resolve project root (walk up for .seraphim/config.json)
## Step 3 — [Operation via node -e calling lib]
## Step 4 — Confirm output
```

### Project Root Resolution (Required in Every Command)

```bash
PROJECT_ROOT=""
DIR="$(pwd)"
while [ "$DIR" != "/" ]; do
  if [ -f "$DIR/.seraphim/config.json" ]; then
    PROJECT_ROOT="$DIR"
    break
  fi
  DIR="$(dirname "$DIR")"
done
```

### Atomic File Write Pattern (All lib writes)

```javascript
const tmpPath = filePath + '.tmp';
fs.writeFileSync(tmpPath, JSON.stringify(data, null, 2), 'utf8');
fs.renameSync(tmpPath, filePath);
```

### Dynamic Phase Directory Discovery (From Phase 34 pitfall)

Do NOT hardcode directory names. Use:

```javascript
const dirs = fs.readdirSync(phasesDir).sort();
const match = dirs.find(d => d.startsWith(phaseNum + '-'));
```

### ROADMAP.md Parse-Mutate-Write Pattern (New for Phase 35)

ROADMAP.md is plain markdown. Manipulation strategy:
1. `fs.readFileSync(roadmapPath, 'utf8')` to get full content
2. Split into lines array
3. Find phase section boundaries by regex: `/^### Phase \d+/`
4. Insert/remove/renumber lines within boundaries
5. Update dependency references (string replace on `Depends on: Phase N`)
6. Update the Progress Table section
7. Atomic write back (tmp+rename)

Renumbering rule for `insert-phase`: use decimal notation (e.g., 35.1) rather than shifting all subsequent phases. The ROADMAP.md section heading becomes `### Phase 35.1: ...` and the directory is `35.1-slug/`.

### Config Toggle Map (D-06)

The workflow settings writable via `/seraphim:settings`:

```json
{
  "workflow": {
    "skip_discuss": false,
    "auto_advance": true,
    "ui_phase": true,
    "ui_review": false,
    "plan_checker": true,
    "research_enabled": true,
    "parallelization": true
  }
}
```

Existing config.json already has most of these — `nyquist_validation`, `auto_advance`, `parallelization`, `ui_phase`, `ui_safety_gate`. The settings command maps toggle names to their config.json paths.

### Model Profile Presets (CFG-01)

Four presets need to be defined. The existing `set-profile.md` reads from `${CLAUDE_PLUGIN_ROOT}/config/profiles.json`. The `/seraphim:settings` command adds profile-switching to the workflow toggle UI — no new file; delegate to the same profiles.json + config write.

Profile names: `quality` (Opus for planner/reviewer, Sonnet for executor), `balanced` (Sonnet everywhere), `budget` (Haiku/Flash), `inherit` (use whatever is currently set per-command).

### complete-milestone Pattern (D-02)

1. Read `.planning/ROADMAP.md` — locate milestone section, extract phase list
2. For each phase: check `.planning/phases/{N}-*/` directories exist, read PLAN.md completion status
3. Read `~/.claude/projects/<hash>/<session>.jsonl` — sum `cost_usd` for session tokens attributed to this milestone
4. Archive: write `.planning/milestones/vX.Y-ROADMAP.md` (copy of milestone section + cost summary)
5. Run `git tag vX.Y -m "Milestone vX.Y complete"`
6. Update ROADMAP.md: mark milestone as complete with archive link
7. Optionally clean up phase directories (ask user)

Cost attribution source: `token-log.jsonl` at `.planning/token-log.jsonl` is the right file to check (present in `.planning/` directory per `ls` output). Alternatively scan `~/.claude/projects/` for session JSONL.

### pr-branch Pattern (D-03)

```bash
# Get all commits on current branch not in main
git log main..HEAD --oneline --format="%H"

# Filter commits that only touch .planning/
# A commit is "planning-only" if all changed files start with .planning/
for COMMIT in $COMMITS; do
  FILES=$(git diff-tree --no-commit-id -r --name-only $COMMIT)
  if echo "$FILES" | grep -qv "^\.planning/"; then
    NON_PLANNING_COMMITS="$NON_PLANNING_COMMITS $COMMIT"
  fi
done

# Create clean branch and cherry-pick
git checkout -b pr/feature-name main
git cherry-pick $NON_PLANNING_COMMITS
```

### health Check Structure (D-07)

Required files to check:
- `.planning/ROADMAP.md` — present?
- `.planning/STATE.md` — present, phase frontmatter parseable?
- `.planning/REQUIREMENTS.md` — present?
- `.planning/phases/` — exists?
- Each phase dir: has at least one `*-PLAN.md`?
- Orphaned plans: PLAN.md files in phases not listed in ROADMAP.md?
- Dependency integrity: phases referenced in `Depends on:` exist?

### workstreams Pattern (D-04)

`.planning/workstreams/` directory. Each workstream is `{name}.json`:

```json
{
  "name": "auth-track",
  "phases": [35, 36],
  "status": "active",
  "current_phase": 35,
  "created_at": "ISO",
  "updated_at": "ISO"
}
```

The `/seraphim:workstreams` command: list, create, switch, status.

### ui-spec Pattern (D-08)

Output: `.planning/phases/{N}-{slug}/UI-SPEC.md`

Structure:
```markdown
# UI Spec: [Phase Name]

## Layout Wireframe (ASCII)
[ASCII box diagram]

## Component List
| Component | Props | State | Notes |
|-----------|-------|-------|-------|

## Interaction Patterns
## Responsive Breakpoints
## Accessibility Notes
```

### ui-review Pattern (D-09)

6-pillar audit generates `.planning/ui-reviews/{phase}-UI-REVIEW.md`:

| Pillar | Score (1-5) | Notes |
|--------|-------------|-------|
| Layout | N | ... |
| Typography | N | ... |
| Color | N | ... |
| Spacing | N | ... |
| Accessibility | N | ... |
| Responsiveness | N | ... |

The command reads source files (React/HTML/CSS) in the phase's implementation scope, audits them against each pillar, assigns scores.

### add-tests Pattern (D-10)

Framework detection:
```bash
# Check package.json for jest/vitest
node -e "
  const pkg = JSON.parse(fs.readFileSync('package.json','utf8'));
  const deps = {...pkg.dependencies,...pkg.devDependencies};
  if (deps.vitest) console.log('vitest');
  else if (deps.jest) console.log('jest');
  else console.log('unknown');
"
```

Input: `.planning/phases/{N}-{slug}/VERIFICATION.md` + `{N}-*-PLAN.md` done-criteria.
Output: test files in `tests/` or `__tests__/` matching the detected framework's conventions.

### manager Command Pattern (D-04 / MGMT-08)

Interactive terminal command center. Since Seraphim commands cannot do true interactive prompts (they run in Claude Code context), the `/seraphim:manager` command:
1. Reads ROADMAP.md and STATE.md
2. Displays a rich formatted summary (phases, statuses, actions available)
3. Prompts the user to pick an action by number
4. Executes the chosen action (calls other commands inline)

This is the "TUI without a TUI" pattern — sequential prompts with formatted output.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Config read/write | Custom JSON parser | `lib/config.js` read/write/validate | Already handles defaults, merging, validation |
| Roadmap data access | Parse roadmap.json manually | `lib/roadmap.js` readRoadmap/writeRoadmap | Atomic write, error handling built-in |
| Git operations | Shell string manipulation | Standard `git` subprocess with error checking | Child process with stderr capture is reliable |
| Atomic writes | Direct `fs.writeFileSync` | tmp+rename pattern (established) | Prevents partial writes on crash |
| Profile management | New profile system | Extend existing profiles.json + set-profile pattern | Already has Qwen check, active marker, table display |

**Key insight:** This phase is almost entirely plumbing — existing lib files handle the hard parts. The commands are mostly UI shells calling lib functions and formatting output.

## Common Pitfalls

### Pitfall 1: ROADMAP.md Section Boundary Detection
**What goes wrong:** Regex for `### Phase N:` matches lines inside phase description text if they quote phase numbers. Renumbering updates wrong lines.
**Why it happens:** Markdown has no structural delimiter — sections end at the next `###` heading.
**How to avoid:** Parse section boundaries as ranges: start = line matching `/^### Phase \d/`, end = next line matching `/^### Phase \d/` or end of "Phase Details" section. Never regex-replace within a section body.
**Warning signs:** Dependency references in descriptions getting corrupted.

### Pitfall 2: Phase Directory Names vs Phase Numbers
**What goes wrong:** `remove-phase 35` finds no directory because directory is `35-phase-management-config-ui-tooling/`. Command fails silently.
**Why it happens:** Directory names include slugs; code assumes numeric-only names.
**How to avoid:** Use the Phase 34 dynamic discovery pattern: `dirs.find(d => d.startsWith(phaseNum + '-'))`. Established pitfall, already documented in STATE.md (Phase 34 pattern).

### Pitfall 3: complete-milestone Git Tag Collision
**What goes wrong:** `git tag vX.Y` fails if tag already exists.
**Why it happens:** Command run twice or tag created manually.
**How to avoid:** Check `git tag -l "vX.Y"` before tagging. If exists, warn and skip — don't abort the whole archival.

### Pitfall 4: pr-branch Planning Commit Detection
**What goes wrong:** A commit that touches both `.planning/` and source files gets excluded from the PR branch, dropping real code.
**Why it happens:** Treating any `.planning/` touch as "planning-only".
**How to avoid:** A commit is excluded ONLY if ALL changed files are under `.planning/`. Use `git diff-tree` per-commit, check if non-`.planning/` files exist. Include mixed commits.

### Pitfall 5: settings Command Writing Unknown Keys
**What goes wrong:** `/seraphim:settings` writes a toggle key that config.js doesn't know about, cluttering config.json.
**Why it happens:** Mismatch between the toggle list in the command and `CONFIG_DEFAULTS` in config.js.
**How to avoid:** Define the allowed workflow toggle keys in one place (config.js `CONFIG_DEFAULTS.workflow`). Settings command validates against that list before writing.

### Pitfall 6: ui-review Running on Non-UI Phases
**What goes wrong:** `/seraphim:ui-review` on a backend phase produces nonsensical scores for non-existent UI files.
**Why it happens:** No guard against running on phases without frontend code.
**How to avoid:** Check for UI files (`.tsx`, `.jsx`, `.css`, `.html`) in the phase scope before running. If none found, print: "No UI files detected in this phase scope. Run on a frontend phase."

### Pitfall 7: add-tests Generating Tests for Already-Tested Code
**What goes wrong:** Running twice creates duplicate test blocks in the same file.
**Why it happens:** Command doesn't check if test file already exists.
**How to avoid:** Check if target test file exists before generating. If exists, append only missing test cases (compare against VERIFICATION.md acceptance criteria already covered).

## Code Examples

### ROADMAP.md Phase Section Parse

```javascript
// Source: established pattern from lib/roadmap.js atomic write
const content = fs.readFileSync(roadmapPath, 'utf8');
const lines = content.split('\n');

// Find phase section start indices
const phaseSectionStarts = [];
lines.forEach((line, i) => {
  if (/^### Phase \d+/.test(line)) {
    phaseSectionStarts.push(i);
  }
});
```

### Config Read-Modify-Write

```javascript
// Source: lib/config.js pattern, verified from source
const config = require(`${PLUGIN_ROOT}/lib/config`);
const cfg = config.read(projectRoot);
cfg.workflow = cfg.workflow || {};
cfg.workflow.skip_discuss = true;
config.write(projectRoot, cfg);
```

### Git Tag with Existence Check

```bash
if git tag -l "v3.2" | grep -q "v3.2"; then
  echo "Tag v3.2 already exists — skipping tag creation"
else
  git tag v3.2 -m "Milestone v3.2: Idea-to-Shipped Journey complete"
  echo "Tagged v3.2"
fi
```

### Workstream State File

```javascript
// .planning/workstreams/{name}.json
const workstreamsDir = path.join(projectRoot, '.planning', 'workstreams');
if (!fs.existsSync(workstreamsDir)) {
  fs.mkdirSync(workstreamsDir, { recursive: true });
}
const wsPath = path.join(workstreamsDir, `${name}.json`);
const tmpPath = wsPath + '.tmp';
fs.writeFileSync(tmpPath, JSON.stringify(state, null, 2), 'utf8');
fs.renameSync(tmpPath, wsPath);
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `close-milestone.md` (archives to `.seraphim/milestones/`) | `complete-milestone` (archives to `.planning/milestones/`) | Phase 35 | Planning artifacts stay in .planning/, not .seraphim/ |
| `set-profile.md` (standalone profile switcher) | `/seraphim:settings` (unified toggle + profile) | Phase 35 | One command for all configuration |

**Deprecated/outdated:**
- `close-milestone.md` (seraphim pipeline concept): replaced by `complete-milestone` which operates on the GSD-style `.planning/` structure not the pipeline `.seraphim/` structure.

## Recommended Plan Structure

Based on command count and dependencies, recommend 4 plans:

**Plan 35-01:** ROADMAP.md manipulation lib + add/insert/remove-phase commands (MGMT-01, MGMT-02, MGMT-03)
- New `lib/planning-roadmap.js` with `readPlanningRoadmap`, `writePlanningRoadmap`, `addPhase`, `insertPhase`, `removePhase`, `renumberPhases`
- Three commands call this lib

**Plan 35-02:** Milestone lifecycle commands + workstreams (MGMT-04, MGMT-05, MGMT-06, MGMT-07, MGMT-08)
- `complete-milestone.md` (archive + git tag)
- `pr-branch.md` (cherry-pick filter)
- `health.md` (integrity check)
- `workstreams.md` (state files)
- `manager.md` (interactive center)

**Plan 35-03:** Configuration commands (CFG-01, CFG-02)
- Extend `lib/config.js` with workflow toggle definitions
- `settings.md` command (unified profile + workflow toggle UI)

**Plan 35-04:** UI and quality tooling (UI-01, UI-02, UI-03)
- `ui-spec.md` (design contract generator)
- `ui-review.md` (6-pillar audit)
- `add-tests.md` (test generation)

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | All lib scripts | ✓ | v22.22.0 | — |
| git CLI | complete-milestone (tag), pr-branch | ✓ | (system git) | — |
| jest/vitest | add-tests detection | Project-dependent | Detect at runtime | Output skeleton with TODO if neither found |

**Missing dependencies with no fallback:** None.

**Missing dependencies with fallback:**
- jest/vitest: if neither detected, add-tests generates a framework-agnostic test skeleton and notes "Install jest or vitest before running".

## Validation Architecture

nyquist_validation is `false` in `.planning/config.json` — section skipped per instructions.

## Open Questions

1. **cost attribution for complete-milestone**
   - What we know: `.planning/token-log.jsonl` exists but its schema is unclear without reading it
   - What's unclear: Does token-log.jsonl have per-session costs? Or does it need cross-referencing with `~/.claude/projects/` JSONL?
   - Recommendation: Read `.planning/token-log.jsonl` during planning. If it has `cost_usd` per entry, sum entries since milestone start. If not, fall back to reporting "cost tracking unavailable — check ~/.claude/projects/ JSONL manually."

2. **insert-phase decimal numbering**
   - What we know: D-02 says "urgent decimal phase" — e.g., 35.1
   - What's unclear: Does the phase directory use `35.1-slug/` or does the slug include the decimal? How does phase discovery handle decimals given `startsWith('35.1-')`?
   - Recommendation: Use `35.1-slug/` as directory name. The `startsWith` check with `'35.1-'` works correctly. Document this convention in the command.

3. **manager command interactivity**
   - What we know: Seraphim commands run as Claude prompts, not true interactive terminals
   - What's unclear: Should manager ask for input and loop, or display a one-shot dashboard then suggest commands?
   - Recommendation: One-shot display with numbered action list. User picks by responding to Claude. Claude executes. No shell loop needed — Claude handles the interactivity.

## Sources

### Primary (HIGH confidence)
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/lib/config.js` — verified CONFIG_DEFAULTS, read/write/validate exports
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/lib/roadmap.js` — verified readRoadmap/writeRoadmap, atomic write pattern
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/commands/close-milestone.md` — verified milestone archive pattern, cost attribution from decisions.jsonl
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/commands/next.md` — verified dynamic phase directory discovery pattern
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/commands/set-profile.md` — verified profile switching pattern
- `/home/alucardmessangeroflight/projects/seraphim/.planning/ROADMAP.md` — verified ROADMAP.md structure and phase section format
- `/home/alucardmessangeroflight/projects/seraphim/.planning/config.json` — verified workflow toggle keys, nyquist_validation=false

### Secondary (MEDIUM confidence)
- Phase 35 CONTEXT.md — all D-01 through D-10 decisions locked by user

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all libs verified from source files
- Architecture patterns: HIGH — extracted from existing command files
- ROADMAP.md parsing: MEDIUM — pattern inferred from structure, no existing parser lib exists yet
- Pitfalls: HIGH — derived from existing pitfall notes in STATE.md + code inspection
- Cost attribution: LOW — token-log.jsonl schema not verified

**Research date:** 2026-04-10
**Valid until:** 2026-05-10 (stable internal codebase)
