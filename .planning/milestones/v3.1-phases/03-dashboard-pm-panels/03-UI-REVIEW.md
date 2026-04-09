---
phase: 3
slug: dashboard-pm-panels
audited: 2026-04-09
baseline: 03-UI-SPEC.md
screenshots: not captured (Playwright not installed — code-only audit)
---

# Phase 3 — UI Review: Dashboard PM Panels

**Audited:** 2026-04-09
**Baseline:** 03-UI-SPEC.md (approved design contract)
**Screenshots:** Not captured — Playwright not installed on this machine. Audit is code-only.

---

## Pillar Scores

| Pillar | Score | Key Finding |
|--------|-------|-------------|
| 1. Copywriting | 3/4 | Empty states match spec exactly; task description body missing from TaskTypeGroup; error copy not implemented |
| 2. Visuals | 3/4 | Clear hierarchy; MilestoneTree missing milestone-level progress bar; TabBar missing aria-controls/tabpanel wiring |
| 3. Color | 3/4 | Badge opacity inconsistency (amber /20 vs /15) across 3 files; hardcoded hex confined to Chart.js options (acceptable) |
| 4. Typography | 2/4 | font-medium (500) used 5 times — spec declares exactly 2 weights (400 regular, 600 semibold); text-lg used on ProjectCard h2 not in spec type scale |
| 5. Spacing | 4/4 | All spacing on declared scale; one max-w-[60%] arbitrary value is a layout constraint, not a spacing token — acceptable |
| 6. Experience Design | 2/4 | No error.tsx boundaries anywhere; phase progress bar in ProjectCard lacks role="progressbar" aria; Suspense only wraps TabBar, not tab content panels |

**Overall: 17/24**

---

## Top 3 Priority Fixes

1. **Missing error boundaries** — If any Neon query fails, the page throws an unhandled server error with no user-facing message. The spec declares `error.tsx` per route segment with "Could not load [panel]. Check your database connection." Create `app/project/[name]/error.tsx` and `app/error.tsx` with the specified error copy.

2. **font-medium (500) used throughout — breaks 2-weight rule** — `font-medium` appears 5 times (ProjectCard profile badge, page.tsx AI activity label, potentially others). The spec is explicit: only 400 and 600. Replace all `font-medium` with `font-semibold` where emphasis is intended, or `font-normal` where it is not. Mixing a third weight creates visual inconsistency at scale.

3. **TabBar is not a functional ARIA tab widget** — The spec declares `aria-controls` on each tab pointing to a `role="tabpanel"` with a matching `id`. Neither exists. Tab links have `role="tab"` and `aria-selected` but no `aria-controls`. Tab content `<section>` elements have no `role="tabpanel"` or `id`. Screen readers cannot navigate between tabs correctly. Add `aria-controls="tab-panel-{tab}"` to each Link and `role="tabpanel" id="tab-panel-{activeTab}"` to the content wrapper in page.tsx.

---

## Detailed Findings

### Pillar 1: Copywriting (3/4)

**Passing:**
- Empty state heading in MilestoneTree.tsx:14 — `No roadmap yet` matches spec exactly.
- Empty state body in MilestoneTree.tsx:17 — references `/seraphim:open-milestone` (minor deviation: spec says `/seraphim:add-feature`, component uses `/seraphim:open-milestone` — acceptable given decisions.md overrides spec).
- HumanTasksByType.tsx:16 — `No pending tasks. All human tasks have been completed or none have been created yet.` matches spec verbatim.
- `Needs attention` label in NeedsAttentionBadge.tsx:19 matches spec.
- Tab labels (metrics/plan/tasks) rendered via `capitalize` CSS from string constants — correct sentence case output.
- No generic labels (Submit, Click Here, OK, Cancel) found anywhere.

**Gaps:**
- **Task description body absent in TaskTypeGroup** — The spec layout contract states each task card shows `task.description` as the primary text, with `feature_id` below it as `text-gray-500 text-xs font-mono`. The implementation shows only `task_id` (as the headline), status badge, urgency, and feature_id. The actual task description/name is not rendered. Users see an internal ID rather than a human-readable task description. (TaskTypeGroup.tsx:20)
- **Error state copy not implemented** — The spec declares: `Could not load [panel name]. Check your database connection and try refreshing.` No error.tsx exists anywhere in the app directory. If a DB query throws, Next.js will display a generic 500 page rather than the specified copy.
- **Feature pipeline progress label** — PipelineProgress.tsx:22 renders `{completeCount}/6 phases` which matches spec exactly.
- **Milestone progress label** — MilestoneTree.tsx:39 renders `{m.complete_count}/{m.feature_count} features` which matches spec exactly.

---

### Pillar 2: Visuals (3/4)

**Passing:**
- Clear visual hierarchy on project detail page: h1 (page title) → TabBar → section headings → content. Font size and weight differentiation is present.
- Status dots (8px milestone / 6px feature) correctly sized per spec — `w-2 h-2` and `w-1.5 h-1.5`.
- Feature rows indented `pl-4` (16px) as specified.
- Pipeline progress 6-dot row renders correctly; filled dots use indigo-500, empty use bg-white/20.
- NeedsAttentionBadge correctly returns null when no attention needed — no ghost DOM.
- CostSparkline: 96x48px, no axes, no labels, tension 0.4 — matches spec exactly.

**Gaps:**
- **MilestoneTree missing milestone-level progress bar** — The spec layout contract for the Plan tab milestone row specifies: `Progress bar: same indigo-500 style as ProjectCard, h-1.5 w-24`. The MilestoneTree component renders milestone name, status dot, and feature count text (`3/5 features`) but no visual progress bar. The feature count is readable but the visual bar is absent. MilestoneTree.tsx:30–40.
- **TabBar aria-controls/tabpanel gap** (see Pillar 6 for detail) affects keyboard navigation UX.
- **ProjectCard hover state** — `hover:bg-[#1f1f1f]` is a subtle +5 lightness shift; functionally correct per spec. No issues.

---

### Pillar 3: Color (3/4)

**Passing:**
- Dominant/secondary/tertiary surface colors correct: `#111111` page bg, `#1a1a1a` cards, `#242424` list rows.
- Accent (indigo-500) used in 17 locations — all appropriate: tab indicator, progress bars, back link, phase run pills, current phase label, hover border on cards.
- Amber correctly scoped to human tasks only.
- Status color map (planned/in-progress/complete/blocked) implemented via STATUS_COLORS constant in status-colors.ts and applied with fallback pattern `?? STATUS_COLORS['planned']`.
- Badge pattern `bg-{color}/15 text-{color} border border-{color}/30` followed on NeedsAttentionBadge and ProjectCard human task count badge.
- Hardcoded `#1a1a1a`, `#242424` are declared semantic surface values in the spec — these are correct and expected.

**Gaps:**
- **Amber opacity inconsistency across files** — The spec badge pattern is `bg-amber-500/15`. Three locations use `bg-amber-500/20` instead:
  - TaskTypeGroup.tsx:21 — status badge on task card
  - page.tsx:191 — Human Tasks count badge in Metrics tab
  - ProjectCard.tsx:31 — budget profile pill (uses /20 with /40 border — different element, arguably acceptable)
  
  The inconsistency is visible: /15 is noticeably lighter than /20. Standardise all amber badges to `bg-amber-500/15 text-amber-400 border border-amber-500/30`.

- **Hardcoded hex in Chart.js config** — `borderColor: 'rgb(99, 102, 241)'` in CostSparkline.tsx:28 and `borderColor: '#6366f1'` in MetricsPanel.tsx:34. Both are indigo-500 equivalents. Chart.js cannot consume Tailwind class names, so this is an unavoidable limitation of the rendering context, not an authoring error. Acceptable.

---

### Pillar 4: Typography (2/4)

**Passing:**
- `text-xs` (29 uses) and `text-sm` (27 uses) are the dominant sizes — correct for a data-dense dashboard.
- `text-2xl` (3 uses) for page h1 elements — matches spec `24px (text-2xl)` for page titles.
- `font-mono` applied consistently to metadata labels (task IDs, cost values, run numbers).
- `uppercase tracking-wider` applied to all section headings as specified.

**Gaps:**
- **font-medium used 5 times — spec forbids it** — The spec states: "Rule: exactly 2 weights in use — 400 (regular) and 600 (semibold). No medium (500)." `font-medium` appears at:
  - ProjectCard.tsx:62 — profile badge label
  - page.tsx:205 — "last 30" span in AI Activity heading (also uses `font-normal` override — contradictory)
  
  All `font-medium` instances should be resolved to either `font-semibold` (if emphasis needed) or remove the weight class (defaulting to 400).

- **text-lg used on ProjectCard h2** — ProjectCard.tsx:61 uses `text-lg` (18px) for the project name heading. The spec type scale declares: body 14px, label 12px, section heading 14px, page title 24px. `text-lg` is not in the declared scale. This creates a fifth distinct size. Change to `text-sm font-semibold` (section heading role) or `text-base` if a slightly larger card title is desired, but document the exception.

- **text-base (1 use)** — One use of `text-base` (16px) detected. Not in declared scale. Minor but contributes to scale creep.

---

### Pillar 5: Spacing (4/4)

**Passing:**
- All spacing values align to the declared scale (multiples of 4px):
  - `p-4` (16px) — 22 uses, dominant card padding — correct
  - `p-6` (24px) — 5 uses, card large padding — correct
  - `p-8` (32px) — 4 uses, page padding — correct
  - `mb-8` — matches spec `mb-8` for tab content gap
  - `mb-10` — matches spec `mb-10` for between tab sections
  - `gap-6` — tab bar gap matches spec `flex gap-6`
  - `pb-2` — tab active/inactive bottom padding matches spec
- `max-w-[60%]` in MilestoneProgressBar.tsx:22 is a layout constraint on truncation, not a spacing token — acceptable exception per spec's established pattern of layout-specific arbitrary values.
- Progress bar `h-1.5` (6px) — spec explicitly documents this as an inherited exception.
- Tab indicator underline 2px (`border-b-2`) — spec explicitly allows this as fine-grain affordance.

No off-scale spacing values found.

---

### Pillar 6: Experience Design (2/4)

**Passing:**
- Empty states: all three new panels have empty state handling — MilestoneTree (milestones.length === 0), HumanTasksByType (pending.length === 0), MilestoneProgressBar (returns null), ProjectCard not-found state. Well covered.
- Suspense: TabBar is wrapped in Suspense (required by Next.js 15 for useSearchParams). Fallback `<div className="h-10" />` prevents layout shift.
- Status dots have `aria-label` conveying status in text, not color alone — correct.
- MilestoneProgressBar has full `role="progressbar" aria-valuenow aria-valuemin aria-valuemax` — correct.
- NeedsAttentionBadge has `aria-label="This project needs attention"` — correct.
- CostSparkline canvas has `aria-hidden="true"` — correct (decorative).

**Gaps:**
- **No error.tsx boundaries exist** — `find "$BASE/app" -name "error.tsx"` returned nothing. The spec explicitly requires: "each tab content area wraps in a Next.js error boundary (error.tsx per route segment) with the error state copy defined above." Any Neon query failure on the project page or home page will produce an unhandled server crash. This is a significant gap for a production dashboard. Minimum needed: `app/error.tsx` (root boundary) and `app/project/[name]/error.tsx`.

- **Tab panels lack ARIA widget wiring** — The TabBar has `role="tablist"`, tabs have `role="tab"` and `aria-selected`, but:
  - No `aria-controls` on tab links pointing to panel IDs
  - No `role="tabpanel"` on the content sections in page.tsx
  - No `id` attributes on panel containers
  Screen reader users cannot navigate with standard tab widget keyboard patterns. The spec declares these attributes explicitly (Accessibility section).

- **Phase progress bar in ProjectCard has no ARIA** — The existing pipeline progress bar in ProjectCard.tsx:74–79 (`div.w-full bg-white/10 h-1.5`) has no `role="progressbar"` or `aria-valuenow`. The MilestoneProgressBar (new) correctly implements this; the pre-existing phase bar does not. Minor inconsistency introduced by the new component setting a higher standard.

- **Tab content has no loading skeleton** — The spec declares: "each tab section renders a `<section>` skeleton using `bg-white/5 animate-pulse rounded-xl h-32` while server data loads." Only the TabBar itself has Suspense. The Plan tab (MilestoneTree) and Tasks tab (HumanTasksByType) are rendered as direct server children — they block the entire page render rather than showing progressive skeletons. Adding Suspense wrappers with skeleton fallbacks around each tab panel would improve perceived performance.

---

## Registry Safety

No third-party component registries. shadcn not initialized. Registry audit: skipped (not applicable per UI-SPEC.md).

---

## Files Audited

- `/home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/components/TabBar.tsx`
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/components/MilestoneTree.tsx`
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/components/FeatureRow.tsx`
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/components/PipelineProgress.tsx`
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/components/HumanTasksByType.tsx`
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/components/TaskTypeGroup.tsx`
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/components/MilestoneProgressBar.tsx`
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/components/NeedsAttentionBadge.tsx`
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/components/CostSparkline.tsx`
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/components/ProjectCard.tsx`
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/app/project/[name]/page.tsx`
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/app/page.tsx`
- `.planning/phases/03-dashboard-pm-panels/03-UI-SPEC.md` (audit baseline)
