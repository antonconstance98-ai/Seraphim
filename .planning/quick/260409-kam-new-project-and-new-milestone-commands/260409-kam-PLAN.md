---
phase: quick
plan: 260409-kam
type: execute
wave: 1
depends_on: []
files_modified:
  - ~/.claude/plugins/seraphim/commands/new-project.md
  - ~/.claude/plugins/seraphim/commands/new-milestone.md
  - ~/.claude/plugins/seraphim/commands/add-feature.md
  - ~/.claude/plugins/seraphim/commands/close-milestone.md
autonomous: true
requirements: []
must_haves:
  truths:
    - "/seraphim:new-project collects project name, description, AND first milestone in one flow"
    - "/seraphim:new-milestone creates a milestone in roadmap.json with version and name"
    - "/seraphim:add-feature blocks when no active milestone exists, offers inline creation"
    - "/seraphim:close-milestone prompts for next milestone after archival"
  artifacts:
    - path: "~/.claude/plugins/seraphim/commands/new-milestone.md"
      provides: "Standalone milestone creation command"
    - path: "~/.claude/plugins/seraphim/commands/new-project.md"
      provides: "Updated project init with first milestone collection"
    - path: "~/.claude/plugins/seraphim/commands/add-feature.md"
      provides: "Active milestone guard with inline creation offer"
    - path: "~/.claude/plugins/seraphim/commands/close-milestone.md"
      provides: "Next milestone prompt after archival"
  key_links:
    - from: "new-project.md"
      to: "lib/roadmap.js writeRoadmap"
      via: "Creates roadmap.json with first milestone during init"
    - from: "new-milestone.md"
      to: "lib/roadmap.js readRoadmap/writeRoadmap"
      via: "Reads existing roadmap, appends milestone, writes back"
    - from: "add-feature.md"
      to: "new-milestone.md"
      via: "References command in error message when user declines inline creation"
---

<objective>
Create /seraphim:new-milestone command, update /seraphim:new-project to collect first milestone during init, refactor /seraphim:add-feature to require an active milestone (with inline creation offer), and extend /seraphim:close-milestone to prompt for next milestone after archival.

Purpose: Complete the milestone lifecycle so users always have clear project/milestone context.
Output: Four updated/new command files in ~/.claude/plugins/seraphim/commands/
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@~/.claude/plugins/seraphim/commands/new-project.md
@~/.claude/plugins/seraphim/commands/add-feature.md
@~/.claude/plugins/seraphim/commands/close-milestone.md
@~/.claude/plugins/seraphim/commands/roadmap.md
@~/.claude/plugins/seraphim/lib/roadmap.js
@~/.claude/plugins/seraphim/lib/config.js

<interfaces>
From lib/roadmap.js:
- readRoadmap(projectRoot) -> { milestones: [] }
- writeRoadmap(projectRoot, data) -> void (atomic write)
- nextFeatureId(roadmap) -> string (e.g., "feat-004")
- milestoneProgress(milestone) -> { percent: number }
- findFeature(roadmap, ref) -> { feature, milestone } | null

From lib/config.js:
- read(projectRoot) -> config object (merged with defaults)
- write(projectRoot, config) -> void
- validate(config) -> { valid: boolean, errors: string[] }

Milestone object shape (from roadmap.json):
{ id: "ms-001", version: "v3.1", name: "...", status: "planned"|"in-progress"|"complete", features: [] }

"Active milestone" = first milestone where status !== "archived" and status !== "complete" (i.e., "planned" or "in-progress").
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Create new-milestone.md and update new-project.md</name>
  <files>~/.claude/plugins/seraphim/commands/new-milestone.md, ~/.claude/plugins/seraphim/commands/new-project.md</files>
  <action>
**new-milestone.md** (new file): Create a slash command following the exact pattern of existing commands (YAML frontmatter with description/argument-hint/allowed-tools, then numbered Steps).

- Frontmatter: description "Create a new milestone in the project roadmap", argument-hint "[version] [--name <name>]", allowed-tools ["Read", "Bash"]
- Step 1: Parse arguments — extract version (positional, e.g. "v4.0") and --name flag
- Step 2: Resolve project root (same bash snippet as add-feature.md Step 1)
- Step 3: Gather inputs interactively — if version not provided, ask "Milestone version? (e.g., v4.0)". If name not provided, ask "Milestone name? (e.g., Dashboard Redesign)"
- Step 4: Write milestone to roadmap using node script:
  - readRoadmap, check if version already exists (abort with error if duplicate)
  - Generate ms-ID: find max ms-NNN, increment
  - Create milestone object: { id, version, name, status: "planned", features: [] }
  - Push to milestones array, writeRoadmap
  - Output JSON with id, version, name
- Step 5: Confirm — print formatted summary like add-feature does:
  ```
  Milestone created.
    ID:      {id}
    Version: {version}
    Name:    {name}
    Status:  planned

  Add features with: /seraphim:add-feature --milestone {version}
  ```

**new-project.md** (update): After the existing Step 4 (write config.json) and before Step 5 (verify), add a new step that collects first milestone:

- Ask: "First milestone version? (e.g., v1.0)" then "Milestone name? (e.g., MVP Launch)"
- Use readRoadmap + writeRoadmap to create roadmap.json with one milestone (same logic as new-milestone.md Step 4)
- Also add project_description to config.json: after asking project name, ask "One-line project description? (leave blank to skip)" and include it in the config write
- Update the Step 6 summary to show the created milestone version and name
- Update the "Next steps" to suggest `/seraphim:add-feature` instead of jumping to `/seraphim:run`
  </action>
  <verify>
    <automated>grep -q "new-milestone" ~/.claude/plugins/seraphim/commands/new-milestone.md && grep -q "milestone version" ~/.claude/plugins/seraphim/commands/new-project.md && echo "PASS" || echo "FAIL"</automated>
  </verify>
  <done>new-milestone.md exists as a complete slash command. new-project.md collects project name, description, and first milestone in one flow.</done>
</task>

<task type="auto">
  <name>Task 2: Refactor add-feature.md with active milestone guard</name>
  <files>~/.claude/plugins/seraphim/commands/add-feature.md</files>
  <action>
Modify add-feature.md Step 3 (Gather inputs — Milestone section). Currently when NO_MILESTONES is found, it asks the user to create a milestone version inline. Replace this with an "active milestone guard":

1. After reading the roadmap, determine the "active milestone": first milestone where status is "planned" or "in-progress". Not "complete", not "archived".

2. If NO milestones exist at all OR no active (non-complete/non-archived) milestone exists:
   - Print: "No active milestone found."
   - Ask: "Create one now? (yes/no)"
   - If YES: collect version and name interactively (same prompts as new-milestone), create the milestone in roadmap.json using the same node script pattern (readRoadmap, create milestone object, writeRoadmap), then continue adding the feature to that milestone.
   - If NO: print "Run `/seraphim:new-milestone` to create one first." and stop.

3. If active milestones exist and --milestone flag was NOT provided:
   - List only active milestones (not complete/archived ones)
   - Ask which one to use
   - If user provides a version not in the list, warn and re-ask

4. If --milestone flag WAS provided, validate it exists and is active. If it's complete/archived, warn: "Milestone {version} is {status}. Choose an active milestone or create a new one."

Keep all other steps (slug, description, priority, write feature, confirm) unchanged.
  </action>
  <verify>
    <automated>grep -q "active milestone" ~/.claude/plugins/seraphim/commands/add-feature.md && grep -q "new-milestone" ~/.claude/plugins/seraphim/commands/add-feature.md && echo "PASS" || echo "FAIL"</automated>
  </verify>
  <done>add-feature.md blocks when no active milestone exists, offers inline creation, references /seraphim:new-milestone as fallback.</done>
</task>

<task type="auto">
  <name>Task 3: Extend close-milestone.md with next milestone prompt</name>
  <files>~/.claude/plugins/seraphim/commands/close-milestone.md</files>
  <action>
After the existing Step 4 (Confirm) in close-milestone.md, add a new Step 5:

**Step 5 — Prompt for next milestone**

After printing the archive summary:

1. Check if any active milestones remain:
   ```bash
   node -e "
     const r = require('${PLUGIN_ROOT}/lib/roadmap');
     const roadmap = r.readRoadmap('${PROJECT_ROOT}');
     const active = (roadmap.milestones || []).filter(m => m.status !== 'complete' && m.status !== 'archived');
     if (active.length > 0) {
       console.log('HAS_ACTIVE:' + active.map(m => m.version).join(','));
     } else {
       console.log('NO_ACTIVE');
     }
   "
   ```

2. If NO_ACTIVE: Ask "Create next milestone? (yes/no)"
   - If YES: collect version and name, create milestone in roadmap.json (same pattern as new-milestone.md), print confirmation
   - If NO: print "No active milestones. Create one later with `/seraphim:new-milestone`."

3. If HAS_ACTIVE: print "Active milestones remaining: {list}. Run `/seraphim:roadmap` to see the full roadmap." — do NOT prompt for creation.
  </action>
  <verify>
    <automated>grep -q "next milestone" ~/.claude/plugins/seraphim/commands/close-milestone.md && grep -q "new-milestone" ~/.claude/plugins/seraphim/commands/close-milestone.md && echo "PASS" || echo "FAIL"</automated>
  </verify>
  <done>close-milestone.md prompts for next milestone creation when no active milestones remain after archival. Skips prompt when active milestones still exist.</done>
</task>

</tasks>

<verification>
All four command files exist and follow Seraphim command patterns (YAML frontmatter, numbered steps, node scripts using lib/roadmap.js helpers). No new dependencies introduced.
</verification>

<success_criteria>
- new-milestone.md is a complete, standalone slash command
- new-project.md collects name + description + first milestone in one flow
- add-feature.md guards against missing active milestone with inline creation offer
- close-milestone.md prompts for next milestone after archival when none remain
- All commands reference lib/roadmap.js readRoadmap/writeRoadmap (no direct fs calls to roadmap.json)
</success_criteria>

<output>
After completion, create `.planning/quick/260409-kam-new-project-and-new-milestone-commands/260409-kam-SUMMARY.md`
</output>
