---
description: "Create Jira hierarchy from spec and tasks"
tools:
  # Server name is configurable via mcp_server in jira-config.yml (default: "atlassian")
  - '{mcp_server}/createJiraIssue'
  - '{mcp_server}/editJiraIssue'
  - '{mcp_server}/searchJiraIssuesUsingJql'
  - '{mcp_server}/getJiraIssue'
---

# Create Jira Issues from Spec and Tasks

This command creates a complete Jira issue hierarchy from your specification and task breakdown:

- **Epic**: Created from SPEC.md (overall specification)
- **Stories**: Created from Phase headers in TASKS.md (e.g., `## Phase 1: Setup`)
- **Tasks/Subtasks**: Created from task items under each Phase (e.g., `- [ ] T001 ...`)

## Prerequisites

1. MCP server providing Jira tools configured and running (server name configured in jira-config.yml)
2. Jira configuration file exists: `.specify/extensions/jira/jira-config.yml`
3. Specification directory with `spec.md` and `tasks.md` files in `specs/<spec-name>/`

## User Input

$ARGUMENTS

Accepts optional `--spec <name>` argument to specify which specification to use.
If not provided, auto-detects from current directory or available specs.

## Steps

### 1. Detect Specification Directory

Determine which specification to use (in order of priority):

1. `--spec <name>` argument
2. Git branch name (if matches a spec directory)
3. Current directory (if inside `specs/<name>/`)
4. Single spec (if only one exists)

Read the specification directory and validate that both `spec.md` and `tasks.md` exist.

### 2. Load Jira Configuration

Load the Jira configuration from `.specify/extensions/jira/jira-config.yml`:

**Artifact Mapping:**
- `mapping.spec_artifact`: Issue type for SPEC.md (default: "Epic")
- `mapping.phase_artifact`: Issue type for Phase headers (default: "Story")
- `mapping.task_artifact`: Issue type for task items (default: "Task"). Set to `""` or `"none"` for 2-level mode (Spec → Phases only)

**2-Level Mode:**

When `task_artifact` is empty (`""`) or `"none"`, the extension operates in 2-level mode:

- Only Spec and Phase issues are created in Jira
- Tasks are embedded as a checklist in the Phase description
- No individual Task issues are created
- Useful for simpler projects or when tasks don't need individual tracking

**Relationships:**
- `mapping.relationships.spec_phase`: How Phase links to Spec (default: "Epic Link")
- `mapping.relationships.phase_task`: How Task links to Phase (default: "Relates")
- `mapping.relationships.spec_task`: Direct Task-Spec link (default: "Epic Link")

Relationship options: `"Parent"`, `"Epic Link"`, `"Relates"`, `"Blocks"`, `"Implements"`, `"is child of"`, `"none"`

**Backward Compatibility:**

If old config structure is found:
- `hierarchy.epic_type` → maps to `mapping.spec_artifact`
- `hierarchy.story_type` → maps to `mapping.phase_artifact`
- `hierarchy.task_type` → maps to `mapping.task_artifact`
- `hierarchy.relationships.epic_story` → maps to `mapping.relationships.spec_phase`
- `hierarchy.relationships.story_task` → maps to `mapping.relationships.phase_task`
- `hierarchy.relationships.epic_task` → maps to `mapping.relationships.spec_task`

**Environment variable overrides:**
- `SPECKIT_JIRA_PROJECT_KEY` → `project.key`
- `SPECKIT_JIRA_SPEC_ARTIFACT` → `mapping.spec_artifact`
- `SPECKIT_JIRA_PHASE_ARTIFACT` → `mapping.phase_artifact`
- `SPECKIT_JIRA_TASK_ARTIFACT` → `mapping.task_artifact`
- `SPECKIT_JIRA_SPEC_PHASE_RELATIONSHIP` → `mapping.relationships.spec_phase`
- `SPECKIT_JIRA_PHASE_TASK_RELATIONSHIP` → `mapping.relationships.phase_task`

### 3. Parse SPEC.md

Read and parse the specification file to extract:

1. **Title**: First H1 heading (e.g., `# User Authentication System`)
2. **Summary**: Content under the first heading or "Overview" section
3. **Full content**: Entire spec for the Epic description

Example SPEC.md structure:
```markdown
# User Authentication System

## Overview
This specification defines the implementation of...

## Goals
- Goal 1
- Goal 2
```

Extract:
- Epic title: "User Authentication System"
- Epic description: Full spec content (or truncated if too long for Jira)

### 4. Parse TASKS.md for Phases and Tasks

Read and parse the tasks file to extract the phase/task hierarchy:

1. **Phases**: H2 headings starting with "Phase" (e.g., `## Phase 1: Setup`)
2. **Tasks**: List items under each phase (e.g., `- [x] T001 Initialize pnpm workspace...`)

Example TASKS.md structure:
```markdown
# Tasks: User Authentication System

## Phase 1: Setup (Shared Infrastructure)

- [x] T001 Initialize pnpm workspace with Nx and NestJS presets
- [x] T002 Add root tsconfig.base.json with path aliases
- [ ] T003 Configure root eslint.config.mjs

## Phase 2: Foundational (Blocking Prerequisites)

- [x] T010 Generate libs/core scaffold
- [ ] T011 Generate libs/config scaffold
```

Extract into a structure like:
```json
{
  "phases": [
    {
      "name": "Phase 1: Setup (Shared Infrastructure)",
      "tasks": [
        {"id": "T001", "description": "Initialize pnpm workspace with Nx and NestJS presets", "status": "completed"},
        {"id": "T002", "description": "Add root tsconfig.base.json with path aliases", "status": "completed"},
        {"id": "T003", "description": "Configure root eslint.config.mjs", "status": "pending"}
      ]
    },
    {
      "name": "Phase 2: Foundational (Blocking Prerequisites)",
      "tasks": [
        {"id": "T010", "description": "Generate libs/core scaffold", "status": "completed"},
        {"id": "T011", "description": "Generate libs/config scaffold", "status": "pending"}
      ]
    }
  ]
}
```

Task status mapping:
- `[x]` → "completed"
- `[ ]` → "pending"
- `[~]` → "in_progress" (optional convention)

### 5. Check for Existing Issues

Before creating issues, check if a mapping file already exists at `specs/<spec-name>/jira-mapping.json`.

If it exists:
1. Display existing mapping summary
2. Ask user whether to:
   - Skip existing issues and only create missing ones
   - Re-create all issues (creates duplicates)
   - Abort and review existing mapping

### 6. Create Epic from SPEC.md

Use the configured MCP server to create the Spec issue:

```
Tool: {mcp_server}/createJiraIssue
Parameters:
  - projectKey: {project.key}
  - issueTypeName: {mapping.spec_artifact}
  - summary: {spec_title}
  - description: {spec_content}
  - additional_fields: {defaults.spec.custom_fields}
```

Store the created Epic key (e.g., "PROJ-100") for linking stories.

Display:
```
✅ Created Epic: PROJ-100 - User Authentication System
   URL: https://your-jira.atlassian.net/browse/PROJ-100
```

### 7. Create Stories for Each Phase

For each phase extracted from TASKS.md, create a Story and link it to the Epic.

**First, check if 2-level mode is enabled:**

```
is_two_level_mode = (task_artifact == "" OR task_artifact == "none" OR task_artifact is not set)
```

**Step 7a: Create the Phase Issue**

The Phase description varies based on mode:

**3-Level Mode (default):** Brief description with task summary
```
Tool: {mcp_server}/createJiraIssue
Parameters:
  - projectKey: {project.key}
  - issueTypeName: {mapping.phase_artifact}
  - summary: {phase_name}
  - description: "Phase from spec: {spec_name}\n\nTasks:\n- T001: ...\n- T002: ..."
  - additional_fields: {defaults.phase.custom_fields}
```

**2-Level Mode:** Full task checklist embedded in description
```
Tool: {mcp_server}/createJiraIssue
Parameters:
  - projectKey: {project.key}
  - issueTypeName: {mapping.phase_artifact}
  - summary: {phase_name}
  - description: |
      Phase from spec: {spec_name}

      ## Tasks

      - [x] T001: Initialize pnpm workspace with Nx and NestJS presets
      - [x] T002: Add root tsconfig.base.json with path aliases
      - [ ] T003: Configure root eslint.config.mjs
      ...
  - additional_fields: {defaults.phase.custom_fields}
```

**Step 7b: Link Phase to Spec based on `relationships.spec_phase`**

| spec_phase value | Action |
|------------------|--------|
| `"Parent"` | Set Phase's parent field to Spec key |
| `"Epic Link"` | Set Epic Link custom field on Phase to Spec key |
| `"Relates"` / `"Blocks"` / etc. | Create issue link from Phase to Spec |
| `"none"` | No link created |

Store each Phase key for linking tasks (if 3-level mode).

Display (3-level mode):
```
✅ Created Phase: PROJ-101 - Phase 1: Setup (Shared Infrastructure)
   URL: https://your-jira.atlassian.net/browse/PROJ-101
   Linked to Spec via: {relationships.spec_phase}
   Tasks: 9 tasks to create
```

Display (2-level mode):
```
✅ Created Phase: PROJ-101 - Phase 1: Setup (Shared Infrastructure)
   URL: https://your-jira.atlassian.net/browse/PROJ-101
   Linked to Spec via: {relationships.spec_phase}
   Tasks: 9 tasks (embedded in description)
```

### 8. Create Individual Jira Issues for EACH Task

**⚠️ SKIP THIS STEP IF 2-LEVEL MODE IS ENABLED**

If `task_artifact` is empty (`""`) or `"none"`, skip this entire step and proceed to Step 9.
In 2-level mode, tasks are already embedded in Phase descriptions.

---

#### 3-Level Mode Only

**CRITICAL: This step is MANDATORY in 3-level mode. You MUST create a separate Jira issue for EVERY task listed in TASKS.md.**

DO NOT skip this step in 3-level mode. DO NOT just put tasks in the Phase description. Each `- [ ] T001 ...` line in TASKS.md becomes its own Jira issue.

**For each task item** (e.g., `- [x] T001 Initialize pnpm workspace...`):

**Step 8a: Create the Jira Task issue**

Call the MCP tool to create the task:

```
Tool: {mcp_server}/createJiraIssue
Parameters:
  - projectKey: {project.key}
  - issueTypeName: {mapping.task_artifact}
  - summary: "{task_id}: {task_description}"
  - description: "Task from spec: {spec_name}\nPhase: {phase_name}\nStatus in spec-kit: {task_status}"
  - additional_fields: {defaults.task.custom_fields}
```

**Step 8b: Link Task to Phase based on `relationships.phase_task`**

| phase_task value | Action |
|------------------|--------|
| `"Parent"` | Set Task's parent field to Phase key |
| `"Relates"` / `"Blocks"` / etc. | Create issue link from Task to Phase |
| `"none"` | No link created |

**Step 8c: Link Task to Spec based on `relationships.spec_task`**

| spec_task value | Action |
|-----------------|--------|
| `"Epic Link"` | Set Epic Link custom field on Task to Spec key |
| `"Relates"` / `"Blocks"` / etc. | Create issue link from Task to Spec |
| `"none"` | No direct Task-Spec link |

**Repeat steps 8a-8c for EVERY task** in the phase before moving to the next Phase.

Example: If Phase 1 has 9 tasks (T001-T009), you create 9 Jira issues:
```
Creating tasks for Story PROJ-101 (Phase 1: Setup):
  ├── ✅ PROJ-110 - T001: Initialize pnpm workspace
  ├── ✅ PROJ-111 - T002: Add root tsconfig.base.json
  ├── ✅ PROJ-112 - T003: Configure root eslint.config.mjs
  ├── ✅ PROJ-113 - T004: Configure prettier
  ├── ✅ PROJ-114 - T005: Add root vitest.config.ts
  ├── ✅ PROJ-115 - T006: Add .npmrc
  ├── ✅ PROJ-116 - T007: Add Nx workspace config
  ├── ✅ PROJ-117 - T008: Add workspace lint/test scripts
  └── ✅ PROJ-118 - T009: Add .gitignore updates

9 tasks created for Phase 1
```

**IMPORTANT**: The jira-mapping.json must include ALL created task keys. If tasks are missing from the mapping, you have not completed this step correctly.

### 9. Save Issue Mapping

Save a comprehensive mapping file at `specs/<spec-name>/jira-mapping.json`.

**Include `"mode": "2-level"` or `"mode": "3-level"`** to indicate the hierarchy type used.

#### 3-Level Mode Mapping

```json
{
  "created_at": "2026-01-29T10:30:00Z",
  "updated_at": "2026-01-29T10:35:00Z",
  "spec": "001-user-auth",
  "project": "PROJ",
  "jira_base_url": "https://your-jira.atlassian.net",
  "epic": {
    "key": "PROJ-100",
    "summary": "User Authentication System",
    "url": "https://your-jira.atlassian.net/browse/PROJ-100"
  },
  "stories": [
    {
      "key": "PROJ-101",
      "summary": "Phase 1: Setup (Shared Infrastructure)",
      "url": "https://your-jira.atlassian.net/browse/PROJ-101",
      "tasks": [
        {
          "key": "PROJ-110",
          "id": "T001",
          "summary": "Initialize pnpm workspace with Nx and NestJS presets",
          "status": "completed",
          "url": "https://your-jira.atlassian.net/browse/PROJ-110"
        },
        {
          "key": "PROJ-111",
          "id": "T002",
          "summary": "Add root tsconfig.base.json with path aliases",
          "status": "completed",
          "url": "https://your-jira.atlassian.net/browse/PROJ-111"
        }
      ]
    },
    {
      "key": "PROJ-102",
      "summary": "Phase 2: Foundational (Blocking Prerequisites)",
      "url": "https://your-jira.atlassian.net/browse/PROJ-102",
      "tasks": [
        {
          "key": "PROJ-120",
          "id": "T010",
          "summary": "Generate libs/core scaffold",
          "status": "completed",
          "url": "https://your-jira.atlassian.net/browse/PROJ-120"
        }
      ]
    }
  ],
  "mode": "3-level",
  "summary": {
    "total_stories": 10,
    "total_tasks": 94,
    "completed_tasks": 87,
    "pending_tasks": 7
  }
}
```

#### 2-Level Mode Mapping

```json
{
  "created_at": "2026-01-29T10:30:00Z",
  "updated_at": "2026-01-29T10:35:00Z",
  "spec": "001-user-auth",
  "project": "PROJ",
  "jira_base_url": "https://your-jira.atlassian.net",
  "mode": "2-level",
  "epic": {
    "key": "PROJ-100",
    "summary": "User Authentication System",
    "url": "https://your-jira.atlassian.net/browse/PROJ-100"
  },
  "stories": [
    {
      "key": "PROJ-101",
      "summary": "Phase 1: Setup (Shared Infrastructure)",
      "url": "https://your-jira.atlassian.net/browse/PROJ-101",
      "embedded_tasks": [
        {"id": "T001", "summary": "Initialize pnpm workspace", "status": "completed"},
        {"id": "T002", "summary": "Add root tsconfig.base.json", "status": "completed"},
        {"id": "T003", "summary": "Configure root eslint.config.mjs", "status": "pending"}
      ]
    }
  ],
  "summary": {
    "total_stories": 10,
    "total_embedded_tasks": 94,
    "completed_tasks": 87,
    "pending_tasks": 7
  }
}
```

Note: In 2-level mode, `embedded_tasks` contains task metadata without Jira keys (since no Jira issues were created for tasks).

### 10. Display Summary

Output a complete summary based on the mode used.

#### 3-Level Mode Summary

```
═══════════════════════════════════════════════════════════════
✅ Jira Hierarchy Created Successfully! (3-level mode)
═══════════════════════════════════════════════════════════════

📋 Project: PROJ
📁 Spec: 001-user-auth

Epic: PROJ-100 - User Authentication System
  └── https://your-jira.atlassian.net/browse/PROJ-100

Stories (10):
  ├── PROJ-101 - Phase 1: Setup (9 tasks)
  ├── PROJ-102 - Phase 2: Foundational (17 tasks)
  ├── PROJ-103 - Phase 3: User Story 1 (10 tasks)
  └── ... (7 more)

Summary:
  • Total Stories: 10
  • Total Tasks: 94 (as Jira issues)
  • Completed: 87 (93%)
  • Pending: 7 (7%)

💾 Mapping saved to: specs/001-user-auth/jira-mapping.json

Next steps:
  • View Epic in Jira: https://your-jira.atlassian.net/browse/PROJ-100
  • Sync status later: /speckit.jira.sync-status --spec 001-user-auth
═══════════════════════════════════════════════════════════════
```

#### 2-Level Mode Summary

```text
═══════════════════════════════════════════════════════════════
✅ Jira Hierarchy Created Successfully! (2-level mode)
═══════════════════════════════════════════════════════════════

📋 Project: PROJ
📁 Spec: 001-user-auth

Epic: PROJ-100 - User Authentication System
  └── https://your-jira.atlassian.net/browse/PROJ-100

Stories (10):
  ├── PROJ-101 - Phase 1: Setup (9 tasks embedded)
  ├── PROJ-102 - Phase 2: Foundational (17 tasks embedded)
  ├── PROJ-103 - Phase 3: User Story 1 (10 tasks embedded)
  └── ... (7 more)

Summary:
  • Mode: 2-level (Epic → Stories only)
  • Total Stories: 10
  • Total Tasks: 94 (embedded in Story descriptions)

💾 Mapping saved to: specs/001-user-auth/jira-mapping.json

Next steps:
  • View Epic in Jira: https://your-jira.atlassian.net/browse/PROJ-100
  • Tasks are tracked as checklists within Stories
═══════════════════════════════════════════════════════════════
```

## Configuration Reference

Edit `.specify/extensions/jira/jira-config.yml` to customize:

| Config Key | Description | Default |
|------------|-------------|---------|
| `mcp_server` | MCP server name | "atlassian" |
| `project.key` | Jira project key | (required) |
| `mapping.spec_artifact` | Issue type for SPEC.md | "Epic" |
| `mapping.phase_artifact` | Issue type for Phases | "Story" |
| `mapping.task_artifact` | Issue type for Tasks. Set to `""` or `"none"` for 2-level mode | "Task" |
| `mapping.relationships.*` | Link types between issues | See docs |
| `defaults.spec.labels` | Labels for Spec | [] |
| `defaults.phase.labels` | Labels for Phases | [] |
| `defaults.task.labels` | Labels for Tasks (3-level only) | [] |

## Troubleshooting

### "Jira configuration not found"

Copy the template and configure:
```bash
cp .specify/extensions/jira/jira-config.template.yml .specify/extensions/jira/jira-config.yml
# Edit jira-config.yml with your project settings
```

### "Sub-task cannot have Epic as parent"

Some Jira configurations don't allow subtasks under Epics. The command handles this by:
1. Creating Stories under the Epic
2. Creating Sub-tasks under Stories (not directly under Epic)

### "Issue type not found"

Use `/speckit.jira.discover-fields` to discover available issue types in your Jira project, then update `jira-config.yml` accordingly.

### Custom Fields

If your Jira project requires custom fields (e.g., Team, Sprint), discover them with `/speckit.jira.discover-fields` and add to the config:

```yaml
defaults:
  epic:
    custom_fields:
      customfield_10001: "Platform Team"
  story:
    custom_fields:
      customfield_10002: "Sprint 1"
```

## Notes

- This command creates issues in sequence: Epic → Stories → Tasks
- The mapping file enables `/speckit.jira.sync-status` to sync completion status
- Re-running creates new issues unless you manually update the mapping
- Task IDs (T001, T002) are preserved in Jira summaries for traceability
