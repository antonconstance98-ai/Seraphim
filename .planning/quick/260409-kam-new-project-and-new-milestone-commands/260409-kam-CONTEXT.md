# Quick Task 260409-kam: new-project and new-milestone commands - Context

**Gathered:** 2026-04-09
**Status:** Ready for planning

<domain>
## Task Boundary

Create /seraphim:new-project and /seraphim:new-milestone commands. Refactor add-feature to require an active milestone. Update close-milestone to prompt for next milestone creation.

</domain>

<decisions>
## Implementation Decisions

### New Project Flow
- /seraphim:new-project collects: project name, one-line description, AND first milestone (version + name) in one step
- Creates .seraphim/ directory, config.json with project name and description, and roadmap.json with first milestone

### Close Milestone Flow
- After archival completes, prompt user: "Create next milestone?" with option to skip
- If yes, ask for version and name, then create it in roadmap.json
- If no, just show completion summary

### Add Feature Guard
- When no active milestone exists, offer to create one inline: "No milestone found — create one now?"
- If user says yes, collect milestone version + name, create it, then continue adding the feature
- If user says no, show the /seraphim:new-milestone command and exit

### Claude's Discretion
- Config.json schema additions (project_name, project_description fields)
- Exact prompting flow and validation for new-project and new-milestone
- How "active milestone" is determined (first non-archived milestone in roadmap.json)

</decisions>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches

</specifics>
