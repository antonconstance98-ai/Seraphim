# Phase 37: Verification + Dashboard — Research

**Researched:** 2026-04-10
**Domain:** Seraphim plugin command authoring + Next.js 16/React 19 dashboard components
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** `/seraphim:verify` traces features to REQ-IDs via REQUIREMENTS.md traceability table + PLAN.md frontmatter. Goal-backward: start from goal, derive must-haves, check codebase. Every report includes at least one REQUIRES_HUMAN_JUDGMENT item.
- **D-02:** `/seraphim:uat` uses persistent UAT.md in phase directory. Accumulates test results across sessions. Each run reads existing state, presents next untested item, records result. YAML frontmatter tracks overall status.
- **D-03:** `/seraphim:validate` spawns nyquist-auditor subagent that reads VERIFICATION.md, identifies coverage gaps, generates additional test scenarios. Writes VALIDATION.md.
- **D-04:** Progress bars computed server-side from `.planning/` filesystem — phase completion % from plan summaries, overall milestone from phase count. Rendered as CSS gradient bars in Next.js dashboard.
- **D-05:** Velocity tracking uses rolling 7-day window from git log commit timestamps. Count commits per day, rolling average. Chart.js line chart.
- **D-06:** Roadmap tree view is hierarchical — milestone → phases → waves → tasks → costs. Expandable/collapsible. Data from ROADMAP.md + plan files + session JSONL for costs.
- **D-07:** Wave progress panels show per-wave breakdown within each phase — plans in wave, completion status, agent count.
- **D-08:** `/seraphim:audit-milestone` spawns integration-checker subagent that reads all phase VERIFICATION.md files, checks cross-phase data contracts, validates requirement coverage across the full milestone.
- **D-09:** `/seraphim:audit-uat` scans all UAT.md files across phases, filters for `status: pending` or `status: failed`, presents grouped by phase with links to debug sessions where applicable.
- **D-10:** `/seraphim:stats` shows terminal summary — phases completed/total, plans count, requirements coverage %, git metrics (commits, files changed), timeline (days elapsed, avg days/phase).

### Claude's Discretion

- Dashboard component styling, chart colors, tree expand/collapse UX, and internal subagent prompts at Claude's discretion.

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.

</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| VFY-01 | User can verify built features via `/seraphim:verify` with goal-backward traceability | D-01: verified approach — read REQUIREMENTS.md traceability + PLAN.md frontmatter `requirements:` field; spawn verifier subagent; write VERIFICATION.md |
| VFY-02 | Every verification report contains at least one REQUIRES_HUMAN_JUDGMENT item | D-01 decision; the verifier subagent prompt must mandate at least one human-judgment item per report |
| VFY-03 | User can validate phase completion via `/seraphim:validate` with Nyquist gap auditing | D-03: spawn nyquist-auditor subagent; read VERIFICATION.md; write VALIDATION.md in phase dir |
| VFY-04 | User can run conversational UAT via `/seraphim:uat` with persistent UAT.md state | D-02: persistent UAT.md per phase; YAML frontmatter tracks status; incremental run-to-run accumulation |
| VFY-05 | User can audit milestone completion via `/seraphim:audit-milestone` checking cross-phase integration | D-08: integration-checker subagent; reads all VERIFICATION.md files; validates cross-phase contracts |
| VFY-06 | User can run cross-phase UAT audit via `/seraphim:audit-uat` surfacing unresolved items | D-09: glob all UAT.md files; filter pending/failed; group by phase |
| VIZ-01 | Dashboard shows progress bars and completion % per phase and milestone | D-04: new `/api/planning/route.ts` filesystem API; `PhaseProgressPanel` server component |
| VIZ-02 | Dashboard shows velocity tracking (rolling 7-day completion rate) | D-05: new `/api/velocity/route.ts` executing `git log`; `VelocityChart` client component |
| VIZ-03 | Dashboard shows wave progress panels (per-wave breakdown) | D-07: new `WaveProgressPanel` server component; reads phase PLAN.md frontmatter `wave:` field |
| VIZ-04 | User can view comprehensive project statistics via `/seraphim:stats` | D-10: terminal command reading filesystem + git log; no dashboard required |
| VIZ-05 | Full roadmap tree view in dashboard with phases/waves/tasks/costs | D-06: new `/api/roadmap-tree/route.ts`; `RoadmapTree` client component with expand/collapse |

</phase_requirements>

---

## Summary

Phase 37 has two distinct delivery areas. The first is five command files (`verify.md`, `uat.md`, `validate.md`, `audit-milestone.md`, `audit-uat.md`) and one terminal stats command (`stats.md`) added to `~/.claude/plugins/seraphim/commands/`. The second is four new Next.js dashboard components (`PhaseProgressPanel`, `VelocityChart`, `RoadmapTree`, `WaveProgressPanel`) backed by three new API routes that read the `.planning/` filesystem and git log rather than Neon.

The existing codebase provides highly reusable scaffolding. Commands follow a well-worn `.md` file pattern with YAML frontmatter and inline Node.js bash blocks. The dashboard uses Next.js 16 / React 19 with `export const dynamic = 'force-dynamic'` on all server pages, `Promise.all` for parallel data fetches, dynamic `import('chart.js/auto')` inside `useEffect` for Chart.js (to avoid SSR breakage), and `Suspense` wrapping for any client component that uses navigation hooks. All new dashboard work must match these existing patterns precisely — no new npm dependencies are permitted (project constraint: zero new npm deps since Phase 32).

The VERIFICATION.md format already exists and is well-established (see Phase 36 as the gold standard). The UAT.md format is new — it must be designed to support incremental accumulation across sessions and be readable by both `uat.md` (interactive loop) and `audit-uat.md` (batch scanner).

**Primary recommendation:** Model all six new commands on the `debug.md` / `forensics.md` pattern (tool restriction frontmatter, atomic tmp+renameSync writes, subagent dispatch for analysis). Model all four dashboard components on the existing `MilestoneProgressBar`, `CostSparkline`, and `PipelineProgress` as direct style references.

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Next.js | 16.2.3 (installed) | Dashboard framework | Already in use — do not upgrade |
| React | 19.2.4 (installed) | Component model | Already in use |
| Tailwind CSS | v4 (`@import "tailwindcss"`) | Styling | Already in use — no `tailwind.config.js`, config lives in CSS |
| Chart.js | 4.5.1 (installed) | Velocity line chart | Already in dashboard via `chart.js/auto` dynamic import |
| `@neondatabase/serverless` | ^1.0.2 (installed) | Neon queries (existing panels only) | Already in use — new panels do NOT use Neon |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| Node.js `fs` | built-in (v22.22.0) | Read `.planning/` filesystem | All new API routes for progress/wave/roadmap data |
| Node.js `child_process` | built-in | Execute `git log` for velocity data | Velocity API route only |
| `marked` | ^18.0.0 (installed) | Markdown rendering (existing use) | Only if VERIFICATION.md content is surfaced in dashboard |

### Zero New NPM Dependencies

Project constraint since Phase 32: no new `npm install` entries. Every dependency needed for Phase 37 is already installed.

---

## Architecture Patterns

### Command File Pattern

All six new commands follow the exact same `.md` frontmatter + body structure used by every existing command:

```yaml
---
description: "One-line description"
argument-hint: "<phase-slug>"
allowed-tools: ["Read", "Write", "Bash"]
---
```

Key rules observed from existing commands:
- Commands that write files always use atomic `tmp+renameSync` pattern (from `requirements.js`, `roadmap.js`, `debug.md`)
- Subagent dispatch uses natural language instructions inline — no separate agent file needed
- Read-only commands restrict tools in frontmatter (`allowed-tools: ["Read", "Bash"]`) — see `forensics.md`
- All file paths computed relative to `PROJECT_ROOT` resolved by walking up to `.seraphim/config.json`

### PLAN.md Frontmatter — Source of Truth for Verification

Existing PLAN.md files carry machine-readable frontmatter. Phase 37 commands can rely on these fields:

```yaml
phase: 36-human-tasks-debugging
plan: 01
wave: 1
requirements: [HTASK-01, HTASK-02, HTASK-03]
must_haves:
  truths: [...]
  artifacts: [{ path, provides, contains }]
  key_links: [{ from, to, via, pattern }]
```

The `requirements:` array is the primary traceability link from plans to REQ-IDs. The `must_haves.truths` and `must_haves.artifacts` arrays are the pre-declared acceptance criteria for each plan — `verify.md` should use these as the checklist.

### VERIFICATION.md Format — Established Standard

The Phase 36 VERIFICATION.md (`36-VERIFICATION.md`) is the canonical template. Its structure:

```markdown
---
phase: <phase-slug>
verified: <ISO timestamp>
status: passed | failed | partial
score: N/N must-haves verified
---

## Goal Achievement

### Observable Truths
| # | Truth | Status | Evidence |

### Required Artifacts
| Artifact | Expected | Status | Details |

### Key Link Verification
| From | To | Via | Status | Details |
```

New `verify.md` writes this exact format to `.planning/phases/{slug}/{num}-VERIFICATION.md`.

### UAT.md Format — New Design

No existing UAT.md files exist. Must be designed for incremental accumulation. Recommended structure:

```markdown
---
phase: <phase-slug>
status: in_progress | complete | has_failures
last_updated: <ISO timestamp>
total: N
tested: N
passed: N
failed: N
---

## UAT Items

### [UAT-01] <feature description>
**status:** passed | failed | pending
**tested:** <ISO timestamp or blank>
**tester:** human
**notes:** <free text>
```

The `uat.md` command reads this file on each run, finds the first item with `status: pending`, presents it to the user, records the result, and updates the frontmatter counts. On resume, it picks up exactly where it left off.

### Dashboard API Routes — Filesystem-Based

New API routes read the `.planning/` filesystem directly — they do NOT query Neon. Pattern matches existing route structure:

```typescript
// app/api/planning/route.ts
export const dynamic = 'force-dynamic';

import { NextResponse } from 'next/server';
import fs from 'fs';
import path from 'path';

export async function GET(req: NextRequest) {
  const planningDir = process.env.PLANNING_DIR ?? path.join(process.cwd(), '../../.planning');
  // read phase dirs, parse PLAN.md frontmatter, compute %
  return NextResponse.json({ phases: [...] });
}
```

`PLANNING_DIR` env var must be set in dashboard's `.env.local` pointing to the project's `.planning/` directory. This is the only configuration needed.

### Chart.js in Dashboard — VelocityChart Pattern

VelocityChart must exactly follow the `CostSparkline.tsx` pattern:

```typescript
'use client';
import { useEffect, useRef } from 'react';

export function VelocityChart({ data }: { data: { date: string; commits: number }[] }) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const chartRef = useRef<any>(null);

  useEffect(() => {
    if (!data || data.length < 2) return;
    import('chart.js/auto').then(({ default: Chart }) => {
      chartRef.current?.destroy();
      if (!canvasRef.current) return;
      chartRef.current = new Chart(canvasRef.current, {
        type: 'line',
        data: {
          labels: data.map(d => d.date),
          datasets: [{
            data: data.map(d => d.commits),
            borderColor: 'rgb(99, 102, 241)',
            borderWidth: 1.5,
            pointRadius: 2,
            tension: 0.4,
            fill: false,
          }],
        },
        options: {
          responsive: true,
          plugins: { legend: { display: false } },
          scales: {
            x: { display: true, ticks: { color: '#6b7280', font: { size: 12 } } },
            y: { display: true, ticks: { color: '#6b7280', font: { size: 12 } } },
          },
        },
      });
    });
    return () => { chartRef.current?.destroy(); };
  }, [data]);

  if (!data || data.length < 2) {
    return <p className="text-gray-500 text-sm">Not enough commits to show velocity. Complete at least 2 days of work.</p>;
  }

  return (
    <>
      <canvas ref={canvasRef} style={{ width: '100%', height: '120px' }} aria-hidden="true" />
      <p className="sr-only">Velocity chart showing commits per day over the last 7 days.</p>
    </>
  );
}
```

Key differences from CostSparkline: axes ARE visible, `responsive: true`, `pointRadius: 2`, height 120px, `aria-hidden` + `sr-only` sibling.

### Recommended Project Structure — New Files

```
~/.claude/plugins/seraphim/
├── commands/
│   ├── verify.md          # VFY-01, VFY-02
│   ├── uat.md             # VFY-04
│   ├── validate.md        # VFY-03
│   ├── audit-milestone.md # VFY-05
│   ├── audit-uat.md       # VFY-06
│   └── stats.md           # VIZ-04
└── dashboard/
    ├── app/api/
    │   ├── planning/route.ts     # PhaseProgressPanel data
    │   ├── velocity/route.ts     # VelocityChart git data
    │   └── roadmap-tree/route.ts # RoadmapTree data
    └── components/
        ├── PhaseProgressPanel.tsx  # VIZ-01
        ├── VelocityChart.tsx       # VIZ-02
        ├── WaveProgressPanel.tsx   # VIZ-03
        └── RoadmapTree.tsx         # VIZ-05
```

The new panels are added to the project detail page (`app/project/[name]/page.tsx`) as a new tab or additional sections within the existing `metrics` tab.

### Anti-Patterns to Avoid

- **SSR Chart.js import:** Never `import Chart from 'chart.js/auto'` at module scope in a client component. Always inside `useEffect`. CostSparkline is the correct reference.
- **Synchronous fs reads in pages:** API routes use `fs.readFileSync` only. Pages import from API routes via `fetch` or pass data through server component props — never call `fs` directly in page components.
- **Mutable UAT state without atomics:** UAT.md writes must use tmp+renameSync. The file accumulates across sessions; a torn write corrupts all accumulated state.
- **Hardcoded `.planning/` path:** Always use `PLANNING_DIR` env var with a sensible default. The dashboard runs from `~/.claude/plugins/seraphim/dashboard/` which is not the project root.
- **`async: true` hook pattern:** Not relevant here — no hooks in this phase. Mentioned to reinforce that verify/uat commands are synchronous subagent dispatches, not hook-triggered flows.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| YAML frontmatter parsing | Custom regex parser | Inline Node.js bash block using existing `parseFrontmatter` pattern from Phase 33 | YAML edge cases (multi-line values, special chars) will break naive regex. The parseFrontmatter function already strips surrounding quotes (Phase 33 decision). |
| Commit counting for velocity | Manual git log parsing | `git log --oneline --since="7 days ago" --format="%ad" --date=short` piped to `sort \| uniq -c` | One-liner; battle-tested; handles timezone and merge commits correctly |
| Progress % from plan files | Counting files manually | Count `{num}-SUMMARY.md` files vs `{num}-PLAN.md` files in each phase dir | SUMMARY.md is written only when a plan completes — this ratio is the canonical completion signal |
| Tree expand/collapse state | External state manager (Zustand, Jotai) | Local `useState` per node | Existing pattern in codebase; MilestoneTree uses same approach; no new dep needed |
| Requirement coverage % | Custom traceability scanner | Read REQUIREMENTS.md traceability table + count `[x]` vs `[ ]` markers | The checkbox format is already machine-readable; regex on `\[x\]` vs `\[ \]` gives exact coverage |

**Key insight:** Every hard sub-problem in this phase already has a solved pattern in the codebase. The work is assembly, not invention.

---

## Common Pitfalls

### Pitfall 1: PLANNING_DIR Not Set in Dashboard Dev Environment
**What goes wrong:** `fs.readFileSync` throws ENOENT because the dashboard process runs from `~/.claude/plugins/seraphim/dashboard/` and `process.cwd()/../../.planning` resolves to a non-existent path on the dev machine.
**Why it happens:** The dashboard is a standalone Next.js app at a fixed path; `.planning/` belongs to the project being monitored, which is at a different path entirely.
**How to avoid:** API routes must require `PLANNING_DIR` env var. Dashboard's `.env.local` must set `PLANNING_DIR=/home/alucardmessangeroflight/projects/seraphim/.planning`. Add a fallback that returns empty data with a helpful error message rather than crashing.
**Warning signs:** Empty panels with no error, or 500s from API routes on first load.

### Pitfall 2: UAT.md Concurrent Write Corruption
**What goes wrong:** Two UAT sessions open simultaneously (e.g., user opens two terminal tabs) both read the same pending item and both try to write the result, leaving the file in a torn state.
**Why it happens:** No locking on markdown file writes.
**How to avoid:** Always use tmp+renameSync atomic write (the project-standard pattern). The last writer wins, which is acceptable for a single-user workflow. Document this limitation in the command.

### Pitfall 3: Verify Command Declaring False Positives
**What goes wrong:** The verifier subagent checks for file existence (`path: X`) and string presence (`contains: Y`) but declares VERIFIED without confirming the code actually functions correctly.
**Why it happens:** Static analysis is easier than runtime verification. Subagents default to the easier task.
**How to avoid:** The verifier subagent prompt must explicitly state: "Evidence must be traceable code path verification, not just string presence. For each artifact, trace from the call site through to the output."

### Pitfall 4: RoadmapTree ROADMAP.md Parse Failures
**What goes wrong:** ROADMAP.md uses freeform markdown structure that changes between milestones. A regex-based parser written for v3.2 format breaks silently on a different milestone's format.
**Why it happens:** ROADMAP.md is human-authored and not schema-enforced.
**How to avoid:** API route should parse defensively — return partial data and a `parse_warnings: []` array. RoadmapTree component should show whatever data is available rather than showing an error when some nodes fail to parse.

### Pitfall 5: git log in API Route on Vercel / Edge Runtime
**What goes wrong:** If the velocity API route is ever deployed to Vercel or edge runtime, `child_process.spawn` is unavailable and the route throws.
**Why it happens:** Edge runtime doesn't support Node.js `child_process`.
**How to avoid:** Velocity route must use Node.js runtime explicitly: `export const runtime = 'nodejs'` (or omit entirely, since Node.js is default). Do NOT use `export const runtime = 'edge'`. The events SSE route already demonstrates this pattern (it uses `export const runtime = 'edge'` only because it needs streaming — velocity doesn't).

### Pitfall 6: `audit-milestone` Subagent Reading Stale VERIFICATION.md
**What goes wrong:** A phase was re-executed after initial verification. The new code is correct but the VERIFICATION.md is outdated. The audit subagent reports the old evidence.
**Why it happens:** VERIFICATION.md is written once and not auto-updated.
**How to avoid:** `audit-milestone.md` command must display the `verified:` timestamp from each VERIFICATION.md frontmatter so the user can see whether re-verification is needed. Flag any VERIFICATION.md older than the most recent commit to that phase's files.

---

## Code Examples

### Atomic Write Pattern (used in all write commands)
```javascript
// Source: established pattern from lib/requirements.js, lib/roadmap.js, commands/debug.md
const fs = require('fs');
const path = require('path');
const target = path.join(phaseDir, 'UAT.md');
const tmp = target + '.tmp';
fs.writeFileSync(tmp, content, 'utf8');
fs.renameSync(tmp, target);
```

### Phase Progress Computation from Filesystem
```javascript
// Count PLAN.md vs SUMMARY.md files to compute completion %
// Source: established pattern — SUMMARY.md written only on plan completion
const fs = require('fs');
function getPhaseProgress(phaseDir) {
  const files = fs.readdirSync(phaseDir);
  const plans = files.filter(f => /^\d+-PLAN\.md$/.test(f)).length;
  const summaries = files.filter(f => /^\d+-SUMMARY\.md$/.test(f)).length;
  if (plans === 0) return { plans: 0, summaries: 0, percent: 0 };
  return { plans, summaries, percent: Math.round((summaries / plans) * 100) };
}
```

### Git Log for Velocity (7-day rolling window)
```bash
# Source: standard git invocation verified on this machine
git log --oneline --since="7 days ago" --format="%ad" --date=short \
  | sort | uniq -c \
  | awk '{print $2","$1}'
# Output: date,commit_count per line
```

### PLAN.md Frontmatter Requirements Extraction
```javascript
// Source: parseFrontmatter pattern established in Phase 33
// PLAN.md files have YAML frontmatter between --- delimiters
function extractRequirements(planContent) {
  const match = planContent.match(/^---\n([\s\S]*?)\n---/);
  if (!match) return [];
  const reqLine = match[1].match(/^requirements:\s*\[([^\]]*)\]/m);
  if (!reqLine) return [];
  return reqLine[1].split(',').map(s => s.trim().replace(/['"]/g, ''));
}
```

### MilestoneProgressBar-style Progress Bar (reference for PhaseProgressPanel)
```typescript
// Source: dashboard/components/MilestoneProgressBar.tsx (verified on disk)
<div className="w-full bg-white/10 rounded-full h-1.5">
  <div
    className="bg-indigo-500 h-1.5 rounded-full transition-all"
    style={{ width: `${percent}%` }}
    role="progressbar"
    aria-valuenow={percent}
    aria-valuemin={0}
    aria-valuemax={100}
  />
</div>
```

### WaveProgressPanel Dot Row (reference from PipelineProgress)
```typescript
// Source: dashboard/components/PipelineProgress.tsx (verified on disk)
<span className={`w-1.5 h-1.5 rounded-full ${complete ? 'bg-indigo-500' : 'bg-white/20'}`}
  aria-label={`Plan ${i + 1}: ${complete ? 'complete' : 'incomplete'}`}
/>
```

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | All commands + API routes | Yes | v22.22.0 | — |
| git CLI | Velocity API route (`git log`) | Yes (repo confirmed) | — | Return empty velocity data with message |
| `.planning/phases/` | All progress + wave computations | Yes (confirmed on disk) | — | Return empty with "No phases found" message |
| `PLANNING_DIR` env var | Dashboard API routes | Not set (must be configured) | — | Fallback to `process.cwd()/../../.planning` with warning log |
| Neon database | Existing panels (not new Phase 37 panels) | Yes (existing, working) | — | — |

**Missing dependencies with no fallback:**
- None that block execution.

**Missing dependencies with fallback:**
- `PLANNING_DIR` env var: must be added to dashboard `.env.local` — Wave 0 task.

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `params` as sync object | `params` as `Promise<{...}>` — must be `await`ed | Next.js 15+ | All new page/route files must `await params` and `await searchParams` — project page already does this correctly |
| `tailwind.config.js` | `@import "tailwindcss"` in CSS, no config file | Tailwind v4 | No JIT config, no `content:` array — classes are detected automatically. New components can use any Tailwind class without registration. |
| `chart.js` named imports | `chart.js/auto` dynamic import inside `useEffect` | chart.js v4 + Next.js SSR | Must use dynamic import pattern; CostSparkline is the reference. |
| `responsive: false` (sparkline) | `responsive: true` for full-width charts | Per component purpose | VelocityChart needs full container width; use `responsive: true` unlike the fixed-size sparkline. |

---

## Open Questions

1. **Where do new dashboard panels appear in the project page?**
   - What we know: The project detail page (`app/project/[name]/page.tsx`) has a tab bar with `metrics`, `plan`, and `tasks` tabs. New panels must integrate here.
   - What's unclear: Whether to add a new `progress` tab or add panels to the existing `metrics` tab.
   - Recommendation: Add a new `progress` tab — less disruption to existing metrics layout, cleaner separation of concerns. Tab label: "Progress".

2. **How does the velocity API route get the git repo path?**
   - What we know: `PLANNING_DIR` env var points to `.planning/`. The git repo root is one level up from `.planning/`.
   - What's unclear: Whether a `GIT_ROOT` env var should be separate or derived from `PLANNING_DIR`.
   - Recommendation: Derive `GIT_ROOT = path.dirname(PLANNING_DIR)`. No additional env var needed.

3. **Does `audit-milestone` need to write a report file, or is it terminal-only?**
   - What we know: D-08 says "spawns integration-checker subagent that reads all phase VERIFICATION.md files." No output file mentioned.
   - What's unclear: Whether the audit should persist its findings for later reference.
   - Recommendation: Write to `.planning/audit-milestone-{timestamp}.md` — consistent with how `forensics.md` writes to `.planning/debug/forensics/{slug}-{timestamp}.md`. Gives the user a record.

---

## Sources

### Primary (HIGH confidence)
- Existing codebase — `dashboard/components/CostSparkline.tsx`, `MilestoneProgressBar.tsx`, `PipelineProgress.tsx` (read directly, verified on disk 2026-04-10)
- Existing codebase — `dashboard/app/project/[name]/page.tsx`, `dashboard/lib/queries.ts`, `dashboard/lib/types.ts` (read directly)
- Existing codebase — `commands/debug.md`, `commands/forensics.md`, `commands/progress.md` (read directly)
- Existing codebase — `lib/requirements.js`, `lib/roadmap.js`, `lib/progress-extractor.js` (read directly)
- `.planning/phases/36-human-tasks-debugging/36-VERIFICATION.md` — canonical VERIFICATION.md template (read directly)
- `.planning/phases/36-human-tasks-debugging/36-01-PLAN.md` — canonical PLAN.md frontmatter structure (read directly)
- `dashboard/package.json` — confirmed dependencies and versions (read directly)
- `.planning/config.json` — `nyquist_validation: false` confirmed (read directly)

### Secondary (MEDIUM confidence)
- CONTEXT.md decisions D-01 through D-10 — user-locked architectural choices

### Tertiary (LOW confidence)
- None.

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all dependencies confirmed installed and in use
- Architecture patterns: HIGH — all patterns read directly from existing codebase
- Pitfalls: HIGH — derived from direct inspection of existing code and established project decisions
- UI spec: HIGH — read directly from 37-UI-SPEC.md

**Research date:** 2026-04-10
**Valid until:** 2026-05-10 (stable codebase; all dependencies pinned)
