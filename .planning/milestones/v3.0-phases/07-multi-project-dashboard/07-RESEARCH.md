# Phase 7: Multi-Project Dashboard — Research

**Researched:** 2026-04-08
**Domain:** Vercel-hosted Next.js dashboard + real-time data sync from local Seraphim pipeline
**Confidence:** HIGH (architecture decisions clear; WebSocket on Vercel is the main technical risk to plan around)

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Vercel-hosted web app — always live at a URL, not localhost. Deployed to Vercel with proper domain.
- **D-02:** Fallback: if Vercel hosting proves impractical for data sync, fall back to always-on local daemon at 127.0.0.1 with auto-selected port.
- **D-03:** Data sync mechanism: hook fires after each phase completion, pushes metrics and progress data to a Vercel-hosted backend (database via Vercel Marketplace — e.g., Neon Postgres or Upstash Redis).
- **D-04:** Live push per phase — data pushes after EACH phase completes, not just full pipeline runs.
- **D-05:** WebSocket connection — dashboard auto-refreshes when new data arrives.
- **D-06:** Seraphim pipeline data only. Clean slate — no import of v1.0-v2.0 historical data.
- **D-07:** Data sources per project: `.seraphim/config.json`, `.seraphim/token-log.jsonl`, `.seraphim/decisions.jsonl`, `.seraphim/phases/*/state.json`, phase output files.
- **D-08:** Full drill-down with rendered phase outputs: overview -> project card -> phase list -> per-phase details -> rendered markdown of actual phase output files.
- **D-09:** Multi-project overview shows: project name, active profile, current phase, progress bar, total cost, last activity date.
- **D-10:** Workflow metrics panel: cross-project model performance, cost trends over time, savings vs Opus-only baseline.

### Claude's Discretion

- Frontend framework choice (Next.js App Router on Vercel is the natural fit)
- Database choice for data persistence (Neon Postgres via Marketplace likely)
- WebSocket implementation (Vercel Functions + WebSocket or external service)
- Dashboard visual design and layout
- Authentication (if any — this is a personal tool, may not need auth)

### Deferred Ideas (OUT OF SCOPE)

- Import v1.0-v2.0 historical data into the new dashboard
- Authentication/access control
- Mobile-responsive design — focus on desktop first

</user_constraints>

---

## CRITICAL: REQUIREMENTS.md vs CONTEXT.md Conflict

REQUIREMENTS.md DASH-04 states: "Localhost web dashboard at 127.0.0.1:PORT, Node.js HTTP server, no frameworks."
CONTEXT.md D-01 states: "Vercel-hosted web app — always live at a URL, not localhost."

**Resolution:** CONTEXT.md is the user's locked decision from the discuss phase. It supersedes the original requirements text. The planner MUST implement the Vercel-hosted approach (D-01 through D-10), not the localhost approach in DASH-04. DASH-04 is effectively replaced by D-01/D-02.

The fallback (D-02) means: if Vercel deployment fails or data-sync architecture proves impractical during implementation, fall back to always-on local daemon. This fallback should be designed in from the start.

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DASH-01 | Multi-project scanner discovers `.seraphim/` directories across `~/projects/` and configured paths | Existing `codex-global-aggregator.js` has this exact pattern (find + mtime-gated incremental reads). Reuse and adapt. |
| DASH-02 | Progress extractor parses phase-state.json, blueprint.md, forge-log.md to surface per-project completion | `phase-state.js` schema known: `{phaseId, loops, retries, completed, completed_at}`. Task counting requires blueprint.md `TASK` marker parsing (existing `markers.js`). |
| DASH-03 | Workflow data aggregator merges token-log.jsonl and decisions.jsonl across all projects | Schema fully documented from `decisions-logger.js` and `token-logger.js`. Multi-project merge pattern exists in `codex-global-aggregator.js`. |
| DASH-04 | **OVERRIDDEN by D-01:** Vercel-hosted dashboard (not localhost). D-02 fallback: localhost daemon. | Next.js 16 on Vercel. API Routes for data ingestion. See Architecture Patterns section. |
| DASH-05 | Overview: each project's name, profile, current phase, progress bar, total cost, last activity | All data available from `.seraphim/config.json` (name, profile) + decisions.jsonl (cost, activity) + phase states. |
| DASH-06 | Per-project drill-down: phase roadmap, tasks, model assignments, pipeline run history | Phase outputs already exist as .md files. Markdown rendering needed in browser. decisions.jsonl has full model/phase history. |
| DASH-07 | Workflow metrics panel: cross-project model performance, cost trends, savings vs Opus-only | `pattern-analyzer.js` already computes this. Extend to multi-project scope. Chart.js 4.5.1 already used in Phase 6 dashboard. |

</phase_requirements>

---

## Summary

Phase 7 builds a Vercel-hosted multi-project command center for Seraphim. The workstation runs a push hook after each phase completes, sending structured data to a Vercel-hosted database. The Next.js dashboard reads from this database and uses Server-Sent Events for real-time updates.

The phase has two distinct implementation tracks:
1. **Local agent (push hook)** — runs on the workstation, scans all `.seraphim/` directories, serializes and pushes metrics + phase output data to Vercel backend
2. **Vercel app (dashboard)** — Next.js App Router app that stores, queries, and renders the pushed data; serves the UI with real-time updates

The Phase 6 dashboard (`lib/dashboard-generator.js`) covers single-project adaptive intelligence metrics. Phase 7 absorbs those panels into a multi-project context and adds: project overview cards, progress bars, task drill-down, and cross-project cost trends.

**Primary recommendation:** Next.js 16 App Router + Neon Postgres (via Vercel Marketplace `vercel integration add neon`) + Server-Sent Events (not WebSocket) for real-time updates. SSE is simpler than WebSocket and fully supported on Vercel's edge infrastructure.

**Critical risk:** Vercel's free tier functions have a 10s timeout. Long-running WebSocket connections are NOT supported on Vercel Serverless Functions. Use Server-Sent Events (SSE) via Vercel Edge Runtime (no timeout limit) instead.

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Next.js | 16.2.3 (verified) | Full-stack React framework on Vercel | Vercel-native; App Router + API Routes; zero config deployment |
| @neondatabase/serverless | 1.0.2 (verified) | Neon Postgres driver (serverless-safe) | Uses HTTP transport — works in Vercel Edge Runtime where TCP is blocked; @vercel/postgres is sunset |
| chart.js | 4.5.1 (existing) | Charts for cost/performance metrics | Already used in Phase 6 dashboard; same version |
| react-chartjs-2 | ~5.x | React wrapper for Chart.js | Standard Vercel/React integration |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| @upstash/redis | 1.37.0 (verified) | Alternative to Neon for simple KV data | If Postgres schema proves overkill; simpler for raw JSONL storage |
| ws | 8.20.0 (verified) | WebSocket on local fallback daemon (D-02) | Only needed if Vercel SSE/polling rejected |
| marked | latest | Markdown-to-HTML rendering in browser | For rendering phase output .md files in drill-down view (D-08) |

### DEPRECATED — Do Not Use

| Package | Status | Replacement |
|---------|--------|-------------|
| `@vercel/postgres` | Sunset — no longer first-party | `@neondatabase/serverless` |
| `@vercel/kv` | Sunset — no longer first-party | `@upstash/redis` |

### Push Hook (Local — runs on workstation)

| Component | Notes |
|-----------|-------|
| Node.js 22.22.2 (installed) | Hook script runtime — no install needed |
| Built-in `fetch` (Node 22) | POST data to Vercel API route; no node-fetch needed — Node 22 has fetch built-in |
| Existing pattern: `codex-global-aggregator.js` | Adapt for Seraphim data; same mtime-gated incremental + atomic write pattern |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Neon Postgres | Upstash Redis | Redis simpler setup; Postgres better for relational cost/phase queries |
| SSE (recommended) | WebSocket (pusher.js) | Pusher $0 free tier limited; adds third-party dependency; SSE sufficient for read-only push |
| SSE (recommended) | Polling every 30s | Polling simpler but not "real-time" per D-05; SSE is low-overhead and Vercel Edge supports it |
| Next.js App Router | Remix / SvelteKit | Vercel deployment is native Next.js; App Router API routes eliminate separate backend |

**Installation (Vercel app):**
```bash
# 1. Install Vercel CLI and authenticate
npm install -g vercel
vercel login

# 2. Scaffold the Next.js app
npx create-next-app@latest seraphim-dashboard --typescript --app --tailwind
cd seraphim-dashboard

# 3. Install app dependencies
npm install @neondatabase/serverless chart.js react-chartjs-2 marked

# 4. Provision Neon Postgres via Vercel Marketplace (auto-injects DATABASE_URL)
vercel integration add neon

# 5. Pull env vars to local development
vercel env pull .env.local --yes

# 6. Deploy
vercel deploy
```

---

## Architecture Patterns

### Recommended Project Structure (Vercel App)

```
seraphim-dashboard/          # Colocated at ~/.claude/plugins/seraphim/dashboard/
├── app/
│   ├── page.tsx             # Multi-project overview (DASH-05)
│   ├── project/[name]/
│   │   └── page.tsx         # Per-project drill-down (DASH-06)
│   ├── api/
│   │   ├── ingest/
│   │   │   └── route.ts     # POST endpoint — receives push from workstation hook
│   │   └── events/
│   │       └── route.ts     # SSE endpoint — streams updates to browser (D-05)
│   └── layout.tsx
├── lib/
│   ├── db.ts                # Neon Postgres client (lazy init — see Pitfall 7)
│   ├── queries.ts           # SQL for project/phase/cost queries
│   └── types.ts             # Shared TypeScript types matching Seraphim schemas
└── components/
    ├── ProjectCard.tsx       # Overview card: name, profile, progress, cost (D-09)
    ├── PhaseRoadmap.tsx      # Phase list with statuses for drill-down (D-06)
    ├── MetricsPanel.tsx      # Cross-project cost/model charts (D-07)
    └── MarkdownRenderer.tsx  # Renders phase output .md content (D-08)
```

### Local Push Hook Structure

```
~/.claude/plugins/seraphim/
├── hooks/
│   └── phase-push.js        # NEW: fires after each phase; calls Vercel ingest API
└── lib/
    └── push-client.js       # NEW: serializes .seraphim/ data; HTTP POST to Vercel
```

### Pattern 1: Phase Completion Push Hook

The hook fires after each Seraphim phase completes (PostToolUse or custom phase-end event). It reads local `.seraphim/` data and pushes to Vercel.

```javascript
// phase-push.js — fires after phase completion
// Source: adapted from codex-global-aggregator.js patterns (existing)
'use strict';
const fs = require('fs');
const path = require('path');
const os = require('os');

async function pushPhaseData(projectRoot, phaseId) {
  const config = JSON.parse(fs.readFileSync(
    path.join(projectRoot, '.seraphim', 'config.json'), 'utf8'
  ));
  const decisions = readJsonl(path.join(projectRoot, '.seraphim', 'decisions.jsonl'));
  const phaseState = readJson(path.join(projectRoot, '.seraphim', 'phases', phaseId, 'state.json'));
  
  const payload = {
    project_name: path.basename(projectRoot),
    project_root: projectRoot,
    profile: config.profile,
    phase_id: phaseId,
    phase_state: phaseState,
    decisions,
    pushed_at: new Date().toISOString()
  };

  const endpoint = process.env.SERAPHIM_DASHBOARD_URL + '/api/ingest';
  const secret   = process.env.SERAPHIM_DASHBOARD_SECRET;
  
  // Fire-and-forget: do NOT await at hook top-level — see Pitfall 3
  fetch(endpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${secret}` },
    body: JSON.stringify(payload)
  }).catch(err => {
    fs.appendFileSync(
      path.join(os.homedir(), '.claude', 'plugins', 'seraphim', 'push-errors.log'),
      `${new Date().toISOString()} ${err.message}\n`
    );
  });
}
```

### Pattern 2: Vercel API Route — Ingest Endpoint

```typescript
// app/api/ingest/route.ts
// Source: Next.js 16 App Router API Routes (official docs)
// Note: neon() called inside function body, not at module top-level (avoids build crash — see Pitfall 7)
import { neon } from '@neondatabase/serverless';
import { NextRequest, NextResponse } from 'next/server';

export async function POST(req: NextRequest) {
  const authHeader = req.headers.get('authorization');
  if (authHeader !== `Bearer ${process.env.DASHBOARD_SECRET}`) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }
  
  // Safe: neon() inside function body, not module scope
  const sql = neon(process.env.DATABASE_URL!);
  const payload = await req.json();
  
  // Upsert project record + append phase data
  await sql`
    INSERT INTO projects (name, root_path, profile, last_pushed_at)
    VALUES (${payload.project_name}, ${payload.project_root}, ${payload.profile}, ${payload.pushed_at})
    ON CONFLICT (name) DO UPDATE
    SET profile = EXCLUDED.profile, last_pushed_at = EXCLUDED.last_pushed_at
  `;
  // ... insert decisions records
  return NextResponse.json({ ok: true });
}
```

### Pattern 3: Server-Sent Events for Real-Time Updates (D-05)

```typescript
// app/api/events/route.ts — SSE via Vercel Edge Runtime
// Source: Next.js Edge Runtime + SSE pattern
export const runtime = 'edge'; // No timeout limit on Edge vs Serverless

export async function GET() {
  const encoder = new TextEncoder();
  const stream = new ReadableStream({
    async start(controller) {
      // Poll DB for changes and emit events
      const interval = setInterval(async () => {
        const update = await getLatestActivity();
        controller.enqueue(encoder.encode(`data: ${JSON.stringify(update)}\n\n`));
      }, 5000); // 5s polling within SSE connection
      
      // Clean up
      return () => clearInterval(interval);
    }
  });
  
  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    }
  });
}
```

### Pattern 4: Multi-Project Scanner (for initial/manual sync)

Reuse and adapt `codex-global-aggregator.js`:

```javascript
// lib/multi-project-scanner.js
// Source: adapted from ~/.claude/hooks/codex-global-aggregator.js (existing)
const DEFAULT_ROOTS = [
  path.join(os.homedir(), 'projects'),
  path.join(os.homedir(), 'agent'),
];

function discoverSeraphimProjects(roots) {
  const projects = [];
  for (const root of roots) {
    if (!fs.existsSync(root)) continue;
    const dirs = fs.readdirSync(root, { withFileTypes: true });
    for (const dir of dirs) {
      if (!dir.isDirectory()) continue;
      const seraphimDir = path.join(root, dir.name, '.seraphim');
      if (fs.existsSync(seraphimDir)) {
        projects.push(path.join(root, dir.name));
      }
    }
  }
  return projects;
}
```

### Pattern 5: Progress Calculation from Phase State + Blueprint

To surface "tasks remaining / tasks done" for DASH-02:

```javascript
// lib/progress-extractor.js
function extractProgress(projectRoot) {
  const phases = ['discover','envision','judge','architect','forge','crucible'];
  const progress = { completed: 0, total: phases.length, tasks_done: 0, tasks_total: 0 };
  
  for (const phase of phases) {
    const statePath = path.join(projectRoot, '.seraphim', 'phases', phase, 'state.json');
    if (fs.existsSync(statePath)) {
      const state = JSON.parse(fs.readFileSync(statePath, 'utf8'));
      if (state.completed) progress.completed++;
      // Blueprint task counting: parse TASK markers from blueprint.md
      const blueprint = path.join(projectRoot, '.seraphim', 'phases', phase, 'blueprint.md');
      if (fs.existsSync(blueprint)) {
        const content = fs.readFileSync(blueprint, 'utf8');
        const tasks = (content.match(/<!-- TASK:/g) || []).length;
        const done = Object.keys(state.retries || {}).length; // tasks attempted
        progress.tasks_total += tasks;
        progress.tasks_done += done;
      }
    }
  }
  return progress;
}
```

### Pattern 6: Lazy DB Initialization (avoids build crash)

```typescript
// lib/db.ts — lazy initialization (safe for build time)
// Source: @neondatabase/serverless best practice (verified via Vercel skill)
import { neon } from '@neondatabase/serverless';

let _sql: ReturnType<typeof neon> | null = null;

export function getSql() {
  if (!_sql) _sql = neon(process.env.DATABASE_URL!);
  return _sql;
}

// Usage in API routes:
// const sql = getSql();
// const rows = await sql`SELECT * FROM projects`;
```

### Anti-Patterns to Avoid

- **WebSocket on Vercel Serverless Functions:** Vercel Serverless Functions are request/response, not persistent. The connection closes when the function returns. Use Edge Runtime for SSE.
- **`neon()` at module top-level:** Calling `neon(process.env.DATABASE_URL!)` at the top level of a module crashes `next build` when `DATABASE_URL` is not yet set (e.g., before Marketplace provisioning). Always use lazy initialization via `getSql()`.
- **Storing phase .md file content in database as blobs:** Expensive, defeats the point. Store file paths in DB; serve content via the ingest endpoint or a signed URL.
- **Direct DB access from push hook:** The hook should only call the API route. Never embed DB credentials in workstation files — credentials live in Vercel env vars only.
- **Regenerating full dashboard HTML on each push:** The Phase 6 pattern generates a static HTML file. Phase 7 should use React/Next.js dynamic rendering, not static file generation.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Markdown rendering | Custom parser | `marked` (or `react-markdown`) | Edge cases in GFM (tables, code blocks, nested lists) are numerous |
| Deployment + HTTPS | Manual nginx/cert | Vercel CLI (`vercel deploy`) | Automatic TLS, CI/CD, preview URLs — zero ops burden |
| DB connection pooling | Manual pool | `@neondatabase/serverless` HTTP transport | Serverless functions can't maintain TCP connection pools across invocations |
| Chart rendering | Custom SVG | Chart.js 4.5.1 (already in Phase 6) | Existing, verified, correct SHA hash already in codebase |
| Auth token generation | Custom HMAC | Random secret string in env var | For a single-user personal tool, a static bearer token is sufficient |
| Real-time state sync | Custom WebSocket server | SSE via Edge Runtime | SSE is one-directional push (all this dashboard needs); WebSocket adds bidirectional complexity for no gain |
| Storage provisioning | Manual Neon account + manual env var setup | `vercel integration add neon` | Auto-provisions account, creates DB, and injects `DATABASE_URL` into project env |

**Key insight:** This dashboard is read-only from the browser's perspective. All writes come from the workstation push hook. SSE (one-way server-to-client push) is exactly right — WebSocket's bidirectionality adds complexity for zero benefit.

---

## Common Pitfalls

### Pitfall 1: Vercel WebSocket Confusion

**What goes wrong:** Developer sets up a WebSocket endpoint in a Vercel Serverless Function; it times out immediately or throws "function exceeded limit."

**Why it happens:** Vercel Serverless Functions are request/response, not persistent. The connection closes when the function returns.

**How to avoid:** Use `export const runtime = 'edge'` for SSE endpoints. Edge Runtime runs on the Vercel edge network with streaming support and no hard timeout.

**Warning signs:** `FUNCTION_INVOCATION_TIMEOUT` errors in Vercel logs.

### Pitfall 2: Neon Postgres TCP on Edge Runtime

**What goes wrong:** Importing `pg` (standard node-postgres) in an Edge Runtime function throws "TCP sockets are not available in Edge Runtime."

**Why it happens:** Vercel Edge Runtime runs in a V8 isolate, not Node.js. No TCP access.

**How to avoid:** Always use `@neondatabase/serverless` (HTTP transport) for database calls in Edge Runtime. For Node.js Serverless Functions, `pg` or `@neondatabase/serverless` both work.

**Warning signs:** `TCP sockets are not available` error at runtime.

### Pitfall 3: Push Hook Blocks Pipeline

**What goes wrong:** Push hook does synchronous HTTP POST; if Vercel is slow or down, the Seraphim pipeline hangs.

**Why it happens:** The hook is in the critical path of phase completion.

**How to avoid:** Make push hook fire-and-forget. Use Node.js `fetch()` with `.catch()` but no top-level `await`. Log failures to a local file but never throw.

**Warning signs:** `/seraphim:run` taking unexpectedly long after phases complete.

### Pitfall 4: decisions.jsonl Contains Meta-Records

**What goes wrong:** Aggregator counts all JSONL lines as phase executions; metrics skewed.

**Why it happens:** `decisions-validator.js` and `pattern-analyzer.js` emit `type: "recommendation"` and `type: "recommendation_response"` meta-records into the same file.

**How to avoid:** Filter records where `type === 'recommendation' || type === 'recommendation_response'` before computing metrics. The existing `pattern-analyzer.js` already does this — reuse its filter logic.

**Warning signs:** Cost totals inflated; phase counts don't match expected.

### Pitfall 5: .seraphim Not Initialized in Most Projects

**What goes wrong:** Scanner finds zero projects; dashboard shows empty state.

**Why it happens:** `.seraphim/` is only created when `/seraphim:new-project` runs. As of research date, NO project in `~/projects/` has been initialized with Seraphim yet.

**How to avoid:** Dashboard should render a helpful "No Seraphim projects found. Run `/seraphim:new-project` in a project to get started." empty state. Do not assume .seraphim directories exist.

**Warning signs:** Multi-project scanner returns empty array on first run.

### Pitfall 6: Vercel Free Tier Limits

**What goes wrong:** Dashboard stops working after exceeding Vercel Hobby tier limits (100GB bandwidth, 100k function invocations/mo).

**Why it happens:** Personal project; auto-push on every phase completion could hit invocation limits if pipelines run frequently.

**How to avoid:** Batch multiple phase events if they fire close together (debounce). SSE polling interval at 5-30s, not 1s. Vercel Hobby tier is sufficient for personal use at reasonable frequency.

### Pitfall 7: neon() at Module Top-Level Crashes next build

**What goes wrong:** `next build` fails with an error about `DATABASE_URL` being undefined. The build succeeds locally after `vercel env pull` but fails in CI or on first deploy before Marketplace provisioning.

**Why it happens:** Next.js evaluates module-level code at build time. `neon(process.env.DATABASE_URL!)` at the top of a module throws immediately if the env var is not set.

**How to avoid:** Always use lazy initialization — call `neon()` inside a `getSql()` function (Pattern 6 above), never at module top-level. This defers the call to request time when the env var is guaranteed to be present.

**Warning signs:** Build error referencing `DATABASE_URL` or `neon` during `next build`/Vercel deployment.

---

## Code Examples

### decisions.jsonl Record Schema (verified from decisions-logger.js)

```javascript
// Source: ~/.claude/plugins/seraphim/lib/decisions-logger.js (existing)
{
  timestamp: "2026-04-08T23:14:09.136Z",
  phase: "envision",
  model: "claude-opus-4-6",
  profile: "performance",
  tokens_in: 12500,
  tokens_out: 3200,
  cost_usd: 0.0492,
  latency_ms: 8400,
  outcome: "success",
  retry_count: 0,
  loop_count: 0,
  quality_signals: {
    crucible_pass_rate: null,
    judge_kill_rate: 0,
    checkpoint_catch_rate: null,
    loop_trigger_reason: null
  }
}
```

### phase state.json Schema (verified from phase-state.js)

```javascript
// Source: ~/.claude/plugins/seraphim/lib/phase-state.js (existing)
{
  phaseId: "envision",
  loops: { "judge_envision": 0 },
  retries: { "task_1": 1 },
  completed: true,
  completed_at: "2026-04-08T23:14:09.136Z"
}
```

### Neon Postgres DB Schema (recommended)

```sql
-- projects table: one row per project
CREATE TABLE projects (
  name VARCHAR(255) PRIMARY KEY,
  root_path TEXT,
  profile VARCHAR(50),
  last_pushed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- decisions table: one row per phase execution (from decisions.jsonl)
CREATE TABLE decisions (
  id SERIAL PRIMARY KEY,
  project_name VARCHAR(255) REFERENCES projects(name),
  timestamp TIMESTAMPTZ,
  phase VARCHAR(50),
  model VARCHAR(100),
  profile VARCHAR(50),
  tokens_in INTEGER,
  tokens_out INTEGER,
  cost_usd NUMERIC(10,6),
  latency_ms INTEGER,
  outcome VARCHAR(50),
  retry_count INTEGER DEFAULT 0,
  loop_count INTEGER DEFAULT 0,
  quality_signals JSONB
);

-- phase_states table: latest state per project+phase
CREATE TABLE phase_states (
  id SERIAL PRIMARY KEY,
  project_name VARCHAR(255) REFERENCES projects(name),
  phase_id VARCHAR(50),
  completed BOOLEAN DEFAULT FALSE,
  completed_at TIMESTAMPTZ,
  loops JSONB,
  retries JSONB,
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(project_name, phase_id)
);
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Pages Router (`pages/api/`) | App Router (`app/api/route.ts`) | Next.js 13+ | New way to write API routes; Pages Router still works but App Router is standard |
| `@vercel/postgres` | `@neondatabase/serverless` | 2024 | `@vercel/postgres` is sunset — do not use. Direct Neon driver is the replacement. |
| `@vercel/kv` | `@upstash/redis` | 2024 | `@vercel/kv` is sunset — do not use. Direct Upstash Redis is the replacement. |
| WebSocket on Vercel | SSE via Edge Runtime | 2023-2024 | Vercel's architecture cannot sustain persistent WebSocket connections in Serverless; Edge Runtime + SSE is the recommended pattern |

---

## Open Questions

1. **Where does the Vercel app live? Same repo or separate repo?**
   - What we know: The Seraphim plugin lives at `~/.claude/plugins/seraphim/`. The Vercel app is a full Next.js project, which needs its own directory and package.json.
   - What's unclear: Should it be `~/.claude/plugins/seraphim/dashboard/` or a separate git repo at `~/projects/seraphim-dashboard/`?
   - Recommendation: Separate directory at `~/.claude/plugins/seraphim/dashboard/` to keep it colocated with the plugin. Deploy from there with `vercel deploy`.

2. **Neon Postgres vs Upstash Redis — which database?**
   - What we know: Neon Postgres is recommended in CONTEXT.md (D-03). Upstash Redis is an alternative.
   - What's unclear: Will the query patterns (cost aggregation by model/phase, trend over time) be better served by relational SQL or KV storage?
   - Recommendation: Neon Postgres. SQL aggregations (`GROUP BY model, phase`, `SUM(cost_usd)`, `DATE_TRUNC`) are exactly what the metrics panel needs. Redis would require manual aggregation in application code.

3. **Should push hook fire on every phase or only on phase completion?**
   - What we know: D-04 says "after EACH phase completes."
   - What's unclear: Does "completes" mean success only, or also failure/loop-trigger events?
   - Recommendation: Push on every phase state change including failures — the dashboard should show loops and retries, which are the interesting signal.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | Push hook (local) | Yes | v22.22.2 | — |
| npm | Package install | Yes | 10.9.7 | — |
| Vercel CLI | Deployment | No | — | `npm install -g vercel && vercel login` |
| Neon Postgres | Data persistence | No | — | `vercel integration add neon` (auto-provisions free account + injects DATABASE_URL) |
| SERAPHIM_DASHBOARD_URL | Push hook env var | Not set | — | Set after `vercel deploy` completes |
| SERAPHIM_DASHBOARD_SECRET | Push hook auth | Not set | — | Generate random string; add to Vercel env + `~/.bashrc` on workstation |

**Missing dependencies with no fallback:**
- None — all missing items have clear install steps.

**Missing dependencies with install steps:**
- Vercel CLI: `npm install -g vercel` + `vercel login`
- Neon database: `vercel integration add neon` — do NOT create manually at neon.tech (Marketplace provisioning auto-injects `DATABASE_URL` into Vercel project env)
- Env vars: set `SERAPHIM_DASHBOARD_URL` and `SERAPHIM_DASHBOARD_SECRET` on workstation after deploy

---

## Sources

### Primary (HIGH confidence)
- `~/.claude/plugins/seraphim/lib/decisions-logger.js` — decisions.jsonl schema (exact field names)
- `~/.claude/plugins/seraphim/lib/phase-state.js` — phase state.json schema
- `~/.claude/plugins/seraphim/lib/dashboard-generator.js` — Phase 6 dashboard pattern (reuse)
- `~/.claude/hooks/codex-global-aggregator.js` — Multi-project scanner pattern (reuse)
- `npm view next version` — verified Next.js 16.2.3
- `npm view @neondatabase/serverless version` — verified 1.0.2
- `.planning/phases/07-multi-project-dashboard/07-CONTEXT.md` — locked architectural decisions
- Vercel Storage skill (project-level) — confirmed `@vercel/postgres` and `@vercel/kv` are sunset; confirmed `@neondatabase/serverless` HTTP transport; confirmed `vercel integration add neon` provisioning path; confirmed lazy `neon()` initialization requirement

### Secondary (MEDIUM confidence)
- Vercel Edge Runtime SSE pattern — established pattern from Vercel docs; SSE is the recommended real-time approach for read-only streaming on Vercel

### Tertiary (LOW confidence)
- Specific Neon LISTEN/NOTIFY support in serverless mode — needs verification against current Neon docs before implementing; skill file did not confirm this pattern

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — versions verified via npm registry; Vercel Storage skill confirmed sunset packages and replacement
- Architecture: HIGH — patterns derived from existing working code in codebase; Vercel constraints verified via skill
- Pitfalls: HIGH — Vercel/Edge Runtime limitations confirmed by Vercel Storage skill; lazy init pattern confirmed

**Research date:** 2026-04-08
**Valid until:** 2026-07-08 (90 days — Next.js/Vercel API stable; Neon HTTP transport stable)
