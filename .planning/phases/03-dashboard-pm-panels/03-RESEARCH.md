# Phase 3: Dashboard PM Panels - Research

**Researched:** 2026-04-09
**Domain:** Next.js 16 App Router UI — tabbed navigation, Server Components, Chart.js sparklines
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Tabbed project page with 3 tabs: Metrics | Plan | Tasks. Metrics = existing v3.0 pipeline view. Plan = roadmap/milestone/feature tree. Tasks = human tasks grouped by type. Cross-project overview goes on the home page.
- **D-02:** Plan tab shows milestone-feature tree with visual status indicators. Status colors: planned (gray), in-progress (blue), complete (green), blocked (red/amber). Feature entries show pipeline position (which of 6 phases complete).
- **D-03:** Tasks tab shows all human tasks for the project grouped by type (decision, research, review, validation, skills). Each task shows status, urgency, and linked feature. Matches the terminal inbox layout (by project then type).
- **D-04:** Home page extended with PM summary per project card: milestone progress bar, feature count (complete/total), pending human task count, cost trend sparkline. "Needs attention" badge on projects with blockers.
- **D-05:** Tabs are URL-based (e.g., `/project/[name]?tab=plan`) for shareability and deep linking. Default tab is Metrics (preserves v3.0 behavior). Server Components render tab content from Neon queries.

### Claude's Discretion

- Exact Tailwind styling and component layout within each tab
- Chart.js configuration for cost trend sparklines
- Responsive breakpoints and mobile layout
- Loading states and error boundaries per tab

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.

</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| ROAD-06 | Roadmap panel on web dashboard showing milestone-feature tree per project with visual status indicators | MilestoneTree + FeatureRow components; Neon queries for milestones and features tables (already upserted via Phase 2 ingest route) |
| TASK-06 | Human task dashboard panel showing all tasks across projects grouped by project and type | HumanTasksByType + TaskTypeGroup components; Neon query on human_tasks table; reuse TaskList human variant styling |
| OVER-02 | Dashboard cross-project panel showing all projects with PM status (milestone progress, feature counts, human tasks) | ProjectCard extension with MilestoneProgressBar, NeedsAttentionBadge, CostSparkline; queries already available via getCostTrend() |

</phase_requirements>

---

## Summary

Phase 3 adds three UI surfaces to an existing Next.js 16 App Router dashboard: a tabbed project detail page (Metrics | Plan | Tasks), and PM summary extensions to the home page ProjectCard. The database is already populated — Phase 2 wired the ingest route to accept and upsert `milestones`, `features`, `human_tasks`, and `cost_trend` tables. Phase 3 is a pure read/render task: query those tables, build components, wire them into the existing page structure.

The most consequential technical choice is how to implement URL-based tab switching. The project page is a Server Component (`export const dynamic = 'force-dynamic'`). Tab state must live in the URL query param (`?tab=plan`). The tab bar itself must be a Client Component to call `useRouter` or render `<Link>` elements that update `searchParams`, while tab content panels can remain Server Components — Next.js 16 handles this correctly via the `searchParams` prop on page Server Components.

The Chart.js sparkline for cost trend is the only client-side rendering concern. The existing codebase already uses Chart.js 4.5.1 via dynamic import inside `useEffect` to avoid SSR DOM crashes. The CostSparkline component must follow that same pattern.

**Primary recommendation:** Build TabBar as a single Client Component; keep all tab content panels (MilestoneTree, HumanTasksByType) as Server Components receiving data as props from the page server function. Add PM queries to `lib/queries.ts`. Extend ProjectCard with PM fields passed from the home page server function.

---

## Standard Stack

### Core (already installed — zero new packages required)

| Library | Version | Purpose | Why |
|---------|---------|---------|------|
| Next.js | 16.2.3 | App Router, Server Components, URL `searchParams` | Already installed; `searchParams` on page props is the canonical way to read query params server-side |
| React | 19.2.4 | Component model | Already installed |
| @neondatabase/serverless | ^1.0.2 | Neon SQL queries | Already installed; `getSql()` pattern established |
| chart.js | ^4.5.1 | Cost trend sparkline | Already installed; existing dynamic-import pattern in codebase |
| tailwindcss | ^4 | All styling | Already installed; all class names verified against existing component patterns |

### No new packages required

The STATE.md records an explicit project decision: "Zero new npm packages — builds entirely on existing infrastructure." This is confirmed by the package.json — all required libraries are present.

---

## Architecture Patterns

### Existing Patterns (MUST follow)

These are verified by reading the actual codebase:

1. **`export const dynamic = 'force-dynamic'`** on every page that queries Neon. Required to avoid prerender crash when `DATABASE_URL` is not set at build time. Both `app/page.tsx` and `app/project/[name]/page.tsx` use this. All new page-level files must include it.

2. **`getSql()` lazy singleton** in `lib/db.ts`. All queries go through this. New queries belong in `lib/queries.ts` following the existing pattern (named async export functions).

3. **Server Components for data pages** — no `use client` unless the component uses browser APIs, hooks, or event handlers. `TabBar` and `CostSparkline` are the only components in Phase 3 that require `use client`.

4. **Chart.js via dynamic import inside `useEffect`** — existing pattern. CostSparkline must follow this exactly:
   ```tsx
   'use client';
   import { useEffect, useRef } from 'react';
   
   export function CostSparkline({ data }: { data: { date: string; cost_usd: number }[] }) {
     const canvasRef = useRef<HTMLCanvasElement>(null);
     useEffect(() => {
       import('chart.js/auto').then(({ Chart }) => {
         // draw chart on canvasRef.current
       });
       return () => { /* destroy chart instance */ };
     }, [data]);
     return <canvas ref={canvasRef} width={96} height={48} aria-hidden="true" />;
   }
   ```

5. **Color/badge pattern** — every colored badge uses `bg-{color}/15 text-{color} border border-{color}/30`. Verified across ProjectCard, TaskList, project page. Never use solid fills.

### Tab Navigation Pattern (Next.js 16 App Router)

In Next.js 16 (App Router), `searchParams` is available as an async prop on page Server Components:

```tsx
// app/project/[name]/page.tsx
interface Props {
  params: Promise<{ name: string }>;
  searchParams: Promise<{ tab?: string }>;
}

export default async function ProjectPage({ params, searchParams }: Props) {
  const { name } = await params;
  const { tab = 'metrics' } = await searchParams;
  // ...
  return (
    <main>
      <TabBar activeTab={tab} projectName={name} />
      {tab === 'metrics' && <MetricsContent ... />}
      {tab === 'plan' && <MilestoneTree ... />}
      {tab === 'tasks' && <HumanTasksByType ... />}
    </main>
  );
}
```

The `TabBar` Client Component renders `<Link>` elements pointing to `?tab=plan` etc., which triggers a server re-render with the new tab value. This pattern requires no client-side state for tab content — it's a server navigation pattern.

```tsx
'use client';
import Link from 'next/link';
import { useSearchParams } from 'next/navigation';

export function TabBar({ projectName }: { projectName: string }) {
  const searchParams = useSearchParams();
  const activeTab = searchParams.get('tab') ?? 'metrics';
  const tabs = ['metrics', 'plan', 'tasks'];
  return (
    <div className="flex gap-6 border-b border-white/10 mb-8" role="tablist">
      {tabs.map((t) => (
        <Link
          key={t}
          href={`/project/${encodeURIComponent(projectName)}?tab=${t}`}
          role="tab"
          aria-selected={activeTab === t}
          className={activeTab === t
            ? 'text-white border-b-2 border-indigo-500 pb-2 capitalize'
            : 'text-gray-400 hover:text-gray-300 pb-2 capitalize'}
        >
          {t.charAt(0).toUpperCase() + t.slice(1)}
        </Link>
      ))}
    </div>
  );
}
```

Note: `useSearchParams()` in a Client Component inside a Server Component page requires a `<Suspense>` boundary wrapper at the call site in Next.js 15+. The TabBar must be wrapped in `<Suspense fallback={...}>` where it is rendered in the page.

### New Neon Queries (add to lib/queries.ts)

Phase 2 already wired the ingest route to upsert into `milestones`, `features`, `human_tasks`, and `cost_trend`. The dashboard just needs read queries:

```typescript
// Get milestones with features for Plan tab
export async function getMilestones(projectName: string) {
  const sql = getSql();
  return sql`
    SELECT version, name, status, feature_count, complete_count, cost_usd
    FROM milestones
    WHERE project_name = ${projectName}
    ORDER BY version ASC
  `;
}

export async function getFeatures(projectName: string) {
  const sql = getSql();
  return sql`
    SELECT feature_id, slug, name, status, milestone_version, pipeline_phase, cost_usd
    FROM features
    WHERE project_name = ${projectName}
    ORDER BY milestone_version ASC, name ASC
  `;
}

// Get human tasks for Tasks tab (all statuses — filter non-done for display)
export async function getHumanTasks(projectName: string) {
  const sql = getSql();
  return sql`
    SELECT task_id, type, status, feature_id, urgency
    FROM human_tasks
    WHERE project_name = ${projectName}
    ORDER BY urgency DESC, task_id ASC
  `;
}

// Get cost trend for sparkline (home page ProjectCard)
export async function getProjectCostTrend(projectName: string) {
  const sql = getSql();
  return sql`
    SELECT date, cost_usd
    FROM cost_trend
    WHERE project_name = ${projectName}
    ORDER BY date ASC
    LIMIT 30
  `;
}

// For home page: PM summary per project (milestone progress + blocked count)
export async function getProjectPmSummary(projectName: string) {
  const sql = getSql();
  const [milestones, tasks, costTrend] = await Promise.all([
    getMilestones(projectName),
    getHumanTasks(projectName),
    getProjectCostTrend(projectName),
  ]);
  return { milestones, tasks, costTrend };
}
```

### Recommended Project Structure (new files only)

```
dashboard/
├── app/
│   └── project/[name]/
│       └── page.tsx               # MODIFY: add searchParams, tab routing
├── components/
│   ├── TabBar.tsx                  # NEW: Client Component, URL-based tabs
│   ├── MilestoneTree.tsx           # NEW: Server Component, Plan tab content
│   ├── FeatureRow.tsx              # NEW: Server Component, child of MilestoneTree
│   ├── PipelineProgress.tsx        # NEW: Server Component, 6-dot pip row
│   ├── HumanTasksByType.tsx        # NEW: Server Component, Tasks tab content
│   ├── TaskTypeGroup.tsx           # NEW: Server Component, group per task type
│   ├── NeedsAttentionBadge.tsx     # NEW: Server Component, amber badge
│   ├── MilestoneProgressBar.tsx    # NEW: Server Component, card progress
│   ├── CostSparkline.tsx           # NEW: Client Component, Chart.js line
│   └── ProjectCard.tsx             # MODIFY: add PM summary fields
└── lib/
    ├── queries.ts                  # MODIFY: add PM query functions
    └── types.ts                    # Already has Milestone/Feature/HumanTask types
```

### Anti-Patterns to Avoid

- **Do not add `use client` to MilestoneTree, HumanTasksByType, or FeatureRow.** These are data display components with no interactivity — keep them Server Components to avoid unnecessary client bundles.
- **Do not fetch data inside components.** All Neon queries go in page server functions or `lib/queries.ts`; data is passed as props.
- **Do not use `router.push()` for tab switching from a Server Component.** TabBar must be Client Component; the page stays Server Component.
- **Do not render Chart.js canvas server-side.** The `<canvas>` element does not exist in Node.js. Chart.js initialization must live inside `useEffect` in a `use client` component.
- **Do not forget `destroy()` on Chart.js instance.** If the CostSparkline re-renders, the previous Chart instance leaks and attaches a second chart to the same canvas. Store the instance in a `useRef` and call `chartRef.current.destroy()` in the cleanup function of the useEffect.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Tab state management | Custom React context, Zustand, or local state | URL query param + Next.js `searchParams` | URL is shareable and survives page refresh; no client state needed; Server Components re-render with correct data |
| Sparkline chart | SVG path math, custom canvas drawing | Chart.js 4 (already installed) | Edge cases in bezier smoothing, axis clamping, and responsive resize are handled |
| SQL for PM tables | Raw template literals in component files | Named functions in `lib/queries.ts` | Keeps queries testable, co-located, and reusable across tabs |
| Status color logic | Inline ternary chains per component | Shared `STATUS_COLORS` constant object | Guarantees consistency; single source of truth for status → Tailwind class mapping |

---

## Common Pitfalls

### Pitfall 1: `useSearchParams()` missing Suspense boundary
**What goes wrong:** Next.js 15+ throws a build warning (or runtime error in strict mode) when `useSearchParams()` is called in a Client Component that is not wrapped in `<Suspense>`. The TabBar uses this hook.
**Why it happens:** `useSearchParams()` opts into dynamic rendering; without Suspense, Next.js cannot statically analyze the page.
**How to avoid:** Wrap `<TabBar>` in `<Suspense fallback={<div className="h-10" />}>` at the call site in the page.
**Warning signs:** Build output warning "useSearchParams() should be wrapped in a suspense boundary."

### Pitfall 2: Chart.js canvas double-mount leak
**What goes wrong:** If CostSparkline unmounts and remounts (e.g., navigating away and back to home page), a second Chart instance attaches to the same canvas element.
**Why it happens:** Chart.js stores instances by canvas element; without cleanup, the old instance remains registered.
**How to avoid:** Store chart instance in `useRef<Chart | null>`, call `chartRef.current?.destroy()` at the top of `useEffect` before creating a new instance, and also in the cleanup return.

### Pitfall 3: `searchParams` prop not awaited
**What goes wrong:** In Next.js 15+, `searchParams` on page Server Components is a Promise. Accessing `.tab` directly without `await` yields `undefined`, defaulting every tab to Metrics.
**Why it happens:** Next.js 15 made `params` and `searchParams` async (Promise-based). The existing project page already correctly awaits `params` — same pattern applies to `searchParams`.
**How to avoid:** `const { tab = 'metrics' } = await searchParams;`

### Pitfall 4: `force-dynamic` missing on modified page
**What goes wrong:** If the project detail page file is refactored without preserving `export const dynamic = 'force-dynamic'`, Next.js may attempt static prerendering and crash because `DATABASE_URL` is absent at build time.
**Why it happens:** Next.js defaults to static generation; Neon queries require a runtime connection string.
**How to avoid:** Verify `export const dynamic = 'force-dynamic'` is the first line of any page file that queries Neon.

### Pitfall 5: Human tasks table not yet migrated
**What goes wrong:** Queries against `milestones`, `features`, `human_tasks`, or `cost_trend` throw a "relation does not exist" Postgres error because the DDL was marked "pending manual application" in Phase 2.
**Why it happens:** The migrate.ts script only creates the three original tables (`projects`, `decisions`, `phase_states`). PM tables require a second migration run.
**How to avoid:** Wave 0 plan must include a migration task that adds the four PM tables before any dashboard query code is written. The ingest route already has the correct upsert SQL — the table column shapes are known (see ingest/route.ts lines 82-128).

---

## Code Examples

### Status color constant (shared, avoids duplication)
```typescript
// lib/status-colors.ts (new file)
export const STATUS_COLORS: Record<string, { dot: string; badge: string }> = {
  planned:     { dot: 'bg-gray-500',  badge: 'bg-gray-500/20 text-gray-400 border-gray-500/30' },
  'in-progress': { dot: 'bg-blue-500', badge: 'bg-blue-500/20 text-blue-400 border-blue-500/30' },
  complete:    { dot: 'bg-green-500', badge: 'bg-green-500/20 text-green-400 border-green-500/30' },
  blocked:     { dot: 'bg-red-500',   badge: 'bg-red-500/20 text-red-400 border-red-500/30' },
};
```

### MilestoneTree (Server Component)
```tsx
// components/MilestoneTree.tsx
import type { MilestoneSnapshot, FeatureSnapshot } from '@/lib/types';
import { STATUS_COLORS } from '@/lib/status-colors';
import { FeatureRow } from './FeatureRow';

interface Props {
  milestones: MilestoneSnapshot[];
  features: FeatureSnapshot[];
}

export function MilestoneTree({ milestones, features }: Props) {
  if (milestones.length === 0) {
    return (
      <div className="bg-[#1a1a1a] border border-white/10 rounded-xl p-6">
        <p className="text-gray-500 text-sm">No roadmap yet</p>
        <p className="text-gray-600 text-xs mt-1 font-mono">
          This project has no milestones defined. Run /seraphim:add-feature in a project directory to get started.
        </p>
      </div>
    );
  }
  return (
    <div className="bg-[#1a1a1a] border border-white/10 rounded-xl p-6">
      {milestones.map((m) => {
        const mFeatures = features.filter((f) => f.milestone_version === m.version);
        const colors = STATUS_COLORS[m.status] ?? STATUS_COLORS['planned'];
        return (
          <div key={m.version} className="border-b border-white/5 last:border-0">
            <div className="flex items-center gap-3 py-3">
              <span className={`w-2 h-2 rounded-full ${colors.dot}`} aria-label={m.status} />
              <span className="text-white font-semibold text-sm">{m.name}</span>
              <span className="text-gray-500 text-xs ml-auto">{m.complete_count}/{m.feature_count} features</span>
            </div>
            {mFeatures.map((f) => <FeatureRow key={f.feature_id} feature={f} />)}
          </div>
        );
      })}
    </div>
  );
}
```

### HumanTasksByType (Server Component)
```tsx
// components/HumanTasksByType.tsx
import type { HumanTaskSnapshot } from '@/lib/types';
import { TaskTypeGroup } from './TaskTypeGroup';

const TYPE_ORDER = ['decision', 'research', 'review', 'validation', 'skills'] as const;

export function HumanTasksByType({ tasks }: { tasks: HumanTaskSnapshot[] }) {
  const pending = tasks.filter((t) => t.status !== 'done');
  if (pending.length === 0) {
    return <p className="text-gray-500 text-sm">No pending tasks. All human tasks have been completed or none have been created yet.</p>;
  }
  return (
    <div>
      {TYPE_ORDER.map((type) => {
        const group = pending.filter((t) => t.type === type);
        if (group.length === 0) return null;
        return <TaskTypeGroup key={type} type={type} tasks={group} />;
      })}
    </div>
  );
}
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `searchParams` as sync object | `searchParams` as `Promise<{...}>` — must `await` | Next.js 15 | Existing project page already correct; apply same pattern to `searchParams` |
| `params` as sync object | `params` as `Promise<{...}>` | Next.js 15 | Already applied in project page |

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|-------------|-----------|---------|----------|
| Node.js | Next.js runtime | Yes | v22.22.0 | — |
| npm | Package management | Yes | 10.9.4 | — |
| next dev server | Dashboard rendering | Yes | 16.2.3 | — |
| Neon DATABASE_URL | PM table queries | Must verify | — | Queries return empty arrays gracefully |
| PM tables in Neon (milestones, features, human_tasks, cost_trend) | All 3 panels | Not verified — "DDL pending manual application" (Phase 2 note) | — | Wave 0 migration task required |

**Missing dependencies with no fallback:**
- PM database tables — if `milestones`, `features`, `human_tasks`, `cost_trend` tables do not exist in Neon, every new query will throw a Postgres error. A migration Wave 0 task is mandatory before implementing any dashboard query code.

---

## Open Questions

1. **Are the PM tables applied to Neon?**
   - What we know: Phase 2 added upsert SQL to the ingest route and defined the table schemas there. The `migrate.ts` script was NOT updated to create PM tables — it still only creates the original 3 tables.
   - What's unclear: Whether someone ran a manual DDL migration against the live Neon database.
   - Recommendation: Wave 0 task must run the PM table DDL (or extend migrate.ts and re-run it) before any other work.

2. **Does HumanTaskSnapshot have a `name` or `description` field?**
   - What we know: The `human_tasks` table schema (from ingest/route.ts) stores: `task_id`, `type`, `status`, `feature_id`, `urgency`. The `HumanTaskSnapshot` type in `lib/types.ts` matches. There is no `name` or `description` column.
   - What's unclear: The UI-SPEC says task cards show a task description. This may need to be derived from `task_id` or the table may need a `description` column added.
   - Recommendation: Check whether Phase 2 implementation added name/description to the table. If not, display `task_id` as the task name and leave description blank. Do not add a new column without Phase 2 schema review.

---

## Sources

### Primary (HIGH confidence)
- Direct code reading: `/home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/**` — all patterns verified from actual running codebase
- `app/project/[name]/page.tsx` — async `params` pattern confirmed
- `lib/queries.ts` — existing query function structure
- `lib/types.ts` — MilestoneSnapshot, FeatureSnapshot, HumanTaskSnapshot shapes confirmed
- `app/api/ingest/route.ts` — PM table column names confirmed (milestones, features, human_tasks, cost_trend)
- `package.json` — confirmed Next.js 16.2.3, React 19.2.4, chart.js 4.5.1, zero new packages needed

### Secondary (MEDIUM confidence)
- Next.js 15/16 `searchParams` async behavior — inferred from existing `params` await pattern in codebase + known Next.js 15 breaking change

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — verified from package.json, no new packages needed
- Architecture: HIGH — all patterns read directly from codebase; tab pattern derived from Next.js async searchParams which the codebase already uses for params
- Pitfalls: HIGH — PM table migration gap confirmed by reading migrate.ts (3 tables only); Chart.js cleanup is a known documented requirement; Suspense for useSearchParams is a Next.js 15+ requirement

**Research date:** 2026-04-09
**Valid until:** 2026-05-09 (stable Next.js + chart.js versions; no fast-moving APIs)
