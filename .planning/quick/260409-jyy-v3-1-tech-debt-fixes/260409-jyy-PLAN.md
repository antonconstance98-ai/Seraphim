---
phase: quick
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - ~/.claude/plugins/seraphim/dashboard/components/MilestoneTree.tsx
  - ~/.claude/plugins/seraphim/dashboard/components/TabBar.tsx
  - ~/.claude/plugins/seraphim/dashboard/app/project/[name]/page.tsx
  - ~/.claude/plugins/seraphim/dashboard/app/error.tsx
  - ~/.claude/plugins/seraphim/dashboard/app/project/[name]/error.tsx
  - ~/.claude/plugins/seraphim/commands/run.md
autonomous: true
requirements: [tech-debt]
must_haves:
  truths:
    - "Empty milestone tree shows /seraphim:add-feature command"
    - "Tab links have aria-controls attributes and tab panels have role=tabpanel"
    - "All dashboard font weights are 400 or 600 (no font-medium)"
    - "Error boundaries exist at app root and project route"
    - "run.md decisions-logger passes feature_id to buildRecord"
    - "Plan and Tasks tabs have Suspense skeletons"
  artifacts:
    - path: "~/.claude/plugins/seraphim/dashboard/app/error.tsx"
      provides: "Root error boundary"
    - path: "~/.claude/plugins/seraphim/dashboard/app/project/[name]/error.tsx"
      provides: "Project route error boundary"
  key_links: []
---

<objective>
Fix 6 tech debt items from v3.1 milestone audit across the Seraphim dashboard and plugin commands.

Purpose: Clean up UI inconsistencies, accessibility gaps, missing error boundaries, and a data tracking bug.
Output: All 6 items resolved in existing files + 2 new error boundary files.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@~/.claude/plugins/seraphim/dashboard/components/MilestoneTree.tsx
@~/.claude/plugins/seraphim/dashboard/components/TabBar.tsx
@~/.claude/plugins/seraphim/dashboard/app/project/[name]/page.tsx
@~/.claude/plugins/seraphim/commands/run.md
</context>

<tasks>

<task type="auto">
  <name>Task 1: Text fixes, font weights, and ARIA attributes</name>
  <files>
    ~/.claude/plugins/seraphim/dashboard/components/MilestoneTree.tsx
    ~/.claude/plugins/seraphim/dashboard/components/TabBar.tsx
    ~/.claude/plugins/seraphim/dashboard/app/project/[name]/page.tsx
  </files>
  <action>
1. In MilestoneTree.tsx, find the empty state text referencing `/seraphim:open-milestone` and replace it with `/seraphim:add-feature`.

2. In TabBar.tsx, add `aria-controls="tab-panel-{tab}"` to each tab Link element (where {tab} is the tab identifier/slug).

3. In dashboard/app/project/[name]/page.tsx, wrap the tab content area with a div that has `role="tabpanel" id="tab-panel-{activeTab}"` where activeTab is the currently selected tab.

4. Search all files under ~/.claude/plugins/seraphim/dashboard/ for `font-medium` and replace every occurrence with `font-semibold`. Use grep to find all instances first, then edit each file.
  </action>
  <verify>
    <automated>grep -r "font-medium" ~/.claude/plugins/seraphim/dashboard/ | wc -l | grep "^0$" && grep -r "open-milestone" ~/.claude/plugins/seraphim/dashboard/components/MilestoneTree.tsx | wc -l | grep "^0$" && grep "aria-controls" ~/.claude/plugins/seraphim/dashboard/components/TabBar.tsx | head -1 && grep "tabpanel" ~/.claude/plugins/seraphim/dashboard/app/project/\[name\]/page.tsx | head -1</automated>
  </verify>
  <done>No font-medium remains in dashboard. Empty state shows add-feature. ARIA attributes present on tabs and tab panel.</done>
</task>

<task type="auto">
  <name>Task 2: Error boundaries and loading skeletons</name>
  <files>
    ~/.claude/plugins/seraphim/dashboard/app/error.tsx
    ~/.claude/plugins/seraphim/dashboard/app/project/[name]/error.tsx
    ~/.claude/plugins/seraphim/dashboard/app/project/[name]/page.tsx
  </files>
  <action>
1. Create ~/.claude/plugins/seraphim/dashboard/app/error.tsx as a 'use client' component that displays "Could not load panel. Check your database connection and try refreshing." with a retry button that calls reset(). Follow Next.js error boundary conventions (export default function Error({ error, reset })).

2. Create ~/.claude/plugins/seraphim/dashboard/app/project/[name]/error.tsx with the same pattern and message.

3. In dashboard/app/project/[name]/page.tsx, wrap the Plan tab content and Tasks tab content each in a React Suspense boundary. The fallback should be a skeleton: `<div className="animate-pulse h-32 rounded bg-zinc-800" />`. Import Suspense from React if not already imported.
  </action>
  <verify>
    <automated>test -f ~/.claude/plugins/seraphim/dashboard/app/error.tsx && test -f ~/.claude/plugins/seraphim/dashboard/app/project/\[name\]/error.tsx && grep "Suspense" ~/.claude/plugins/seraphim/dashboard/app/project/\[name\]/page.tsx | head -1 && grep "use client" ~/.claude/plugins/seraphim/dashboard/app/error.tsx | head -1</automated>
  </verify>
  <done>Both error boundary files exist with correct message. Suspense skeletons wrap Plan and Tasks tab content.</done>
</task>

<task type="auto">
  <name>Task 3: Pass feature_id in run.md decisions-logger</name>
  <files>~/.claude/plugins/seraphim/commands/run.md</files>
  <action>
In commands/run.md, locate the decisions-logger section where buildRecord() is called. Add the current feature's feature_id (available from the pipeline context as the feature slug) as a parameter to the buildRecord() call. The feature slug should already be available in the surrounding context from the pipeline execution — find it and pass it through.
  </action>
  <verify>
    <automated>grep "feature_id\|feature\.id\|featureId" ~/.claude/plugins/seraphim/commands/run.md | head -3</automated>
  </verify>
  <done>buildRecord() call in decisions-logger includes feature_id from pipeline context.</done>
</task>

</tasks>

<verification>
All 6 tech debt items resolved:
1. grep confirms no "open-milestone" in MilestoneTree.tsx
2. grep confirms aria-controls in TabBar.tsx and tabpanel in page.tsx
3. grep confirms zero font-medium in dashboard/
4. error.tsx files exist at both paths
5. grep confirms feature_id in run.md buildRecord call
6. grep confirms Suspense in page.tsx
</verification>

<success_criteria>
- Zero instances of font-medium in dashboard directory
- Both error boundary files created with correct message
- ARIA attributes on tab navigation
- Suspense skeletons on Plan and Tasks tabs
- feature_id passed to buildRecord in run.md
</success_criteria>

<output>
After completion, create `.planning/quick/260409-jyy-v3-1-tech-debt-fixes/260409-jyy-SUMMARY.md`
</output>
