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

Required configuration:
- `mcp_server`: Name of the MCP server (default: "atlassian")
- `project.key`: Jira project key (required)
- `hierarchy.epic_type`: Issue type for spec (default: "Epic")
- `hierarchy.story_type`: Issue type for phases (default: "Story")
- `hierarchy.task_type`: Issue type for tasks (default: "Sub-task")
- `hierarchy.link_type`: Link type if not using subtasks (default: "Relates")

**Backward Compatibility (v1.x configs):**

If the old `hierarchy.issue_type` is found instead of the new structure:
1. Map `hierarchy.issue_type` → `hierarchy.task_type`
2. Use defaults: `epic_type: "Epic"`, `story_type: "Story"`
3. Display upgrade notice:
   ```
   ⚠️  Config upgrade: Found v1.x config with 'hierarchy.issue_type'
      Mapped to: task_type = "{issue_type}"
      Using defaults: epic_type = "Epic", story_type = "Story"
      Consider updating jira-config.yml to v2.0 format.
   ```

Environment variable overrides:
- `SPECKIT_JIRA_PROJECT_KEY` → `project.key`
- `SPECKIT_JIRA_EPIC_TYPE` → `hierarchy.epic_type`
- `SPECKIT_JIRA_STORY_TYPE` → `hierarchy.story_type`
- `SPECKIT_JIRA_TASK_TYPE` → `hierarchy.task_type`
- `SPECKIT_JIRA_ISSUE_TYPE` → `hierarchy.task_type` (legacy, for backward compat)

### 3. Parse SPEC.md

Read and parse the specification file to extract:

1. **Title**: First H1 heading (e.g., `# TypeScript MSA Framework Implementation`)
2. **Summary**: Content under the first heading or "Overview" section
3. **Full content**: Entire spec for the Epic description

Example SPEC.md structure:
```markdown
# TypeScript MSA Framework Implementation

## Overview
This specification defines the implementation of...

## Goals
- Goal 1
- Goal 2
```

Extract:
- Epic title: "TypeScript MSA Framework Implementation"
- Epic description: Full spec content (or truncated if too long for Jira)

### 4. Parse TASKS.md for Phases and Tasks

Read and parse the tasks file to extract the phase/task hierarchy:

1. **Phases**: H2 headings starting with "Phase" (e.g., `## Phase 1: Setup`)
2. **Tasks**: List items under each phase (e.g., `- [x] T001 Initialize pnpm workspace...`)

Example TASKS.md structure:
```markdown
# Tasks: TypeScript MSA Framework Implementation

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

Use the configured MCP server to create an Epic:

```
Tool: {mcp_server}/createJiraIssue
Parameters:
  - projectKey: {project.key}
  - issueTypeName: {hierarchy.epic_type}
  - summary: {spec_title}
  - description: {spec_content}
  - additional_fields: {defaults.epic.custom_fields}
```

Store the created Epic key (e.g., "MSATS-100") for linking stories.

Display:
```
✅ Created Epic: MSATS-100 - TypeScript MSA Framework Implementation
   URL: https://your-jira.atlassian.net/browse/MSATS-100
```

### 7. Create Stories for Each Phase

For each phase extracted from TASKS.md:

```
Tool: {mcp_server}/createJiraIssue
Parameters:
  - projectKey: {project.key}
  - issueTypeName: {hierarchy.story_type}
  - summary: {phase_name}
  - description: "Phase from spec: {spec_name}\n\nTasks:\n- T001: ...\n- T002: ..."
  - parent: {epic_key} (if Story can have Epic parent)
  - additional_fields: {defaults.story.custom_fields}
```

If the Jira project doesn't support Epic as parent for Stories, link them using:
```
Tool: {mcp_server}/editJiraIssue (to set Epic Link field)
```

Store each Story key for linking tasks.

Display:
```
✅ Created Story: MSATS-101 - Phase 1: Setup (Shared Infrastructure)
   URL: https://your-jira.atlassian.net/browse/MSATS-101
   Tasks: 9 tasks to create
```

### 8. Create Tasks Under Each Story

**IMPORTANT: Tasks belong to Stories, NOT to the Epic!**

The hierarchy is:
```
Epic (from SPEC.md)
  └── Story (from Phase header)        ← Story's parent = Epic
        └── Task (from task item)      ← Task's parent = Story (NOT Epic!)
```

For each Story created in Step 7, iterate through its tasks and create them:

**If `hierarchy.task_type` is "Sub-task":**

Sub-tasks MUST have the **Story** as their parent (not the Epic):

```
Tool: {mcp_server}/createJiraIssue
Parameters:
  - projectKey: {project.key}
  - issueTypeName: "Sub-task"
  - summary: {task_id}: {task_description}
  - description: "Task from: {spec_name}\nPhase: {phase_name}\nStatus in spec: {task_status}"
  - parent: {story_key}        ← MUST be the Story key, NOT the Epic key!
  - additional_fields: {defaults.task.custom_fields}
```

**If `hierarchy.task_type` is "Task" (not subtask):**

Create as standalone Task, then link to the **Story** (not Epic):

```
Tool: {mcp_server}/createJiraIssue
Parameters:
  - projectKey: {project.key}
  - issueTypeName: "Task"
  - summary: {task_id}: {task_description}
  - description: "Task from: {spec_name}\nPhase: {phase_name}\nStatus in spec: {task_status}"
  - additional_fields: {defaults.task.custom_fields}
```

Then link to **Story** (not Epic) using the configured link_type.

Display progress:
```
  ├── ✅ MSATS-110 - T001: Initialize pnpm workspace
  ├── ✅ MSATS-111 - T002: Add root tsconfig.base.json
  └── ✅ MSATS-112 - T003: Configure root eslint.config.mjs
```

### 9. Save Issue Mapping

Save a comprehensive mapping file at `specs/<spec-name>/jira-mapping.json`:

```json
{
  "created_at": "2026-01-29T10:30:00Z",
  "updated_at": "2026-01-29T10:35:00Z",
  "spec": "001-ts-msa-implementation",
  "project": "MSATS",
  "jira_base_url": "https://your-jira.atlassian.net",
  "epic": {
    "key": "MSATS-100",
    "summary": "TypeScript MSA Framework Implementation",
    "url": "https://your-jira.atlassian.net/browse/MSATS-100"
  },
  "stories": [
    {
      "key": "MSATS-101",
      "summary": "Phase 1: Setup (Shared Infrastructure)",
      "url": "https://your-jira.atlassian.net/browse/MSATS-101",
      "tasks": [
        {
          "key": "MSATS-110",
          "id": "T001",
          "summary": "Initialize pnpm workspace with Nx and NestJS presets",
          "status": "completed",
          "url": "https://your-jira.atlassian.net/browse/MSATS-110"
        },
        {
          "key": "MSATS-111",
          "id": "T002",
          "summary": "Add root tsconfig.base.json with path aliases",
          "status": "completed",
          "url": "https://your-jira.atlassian.net/browse/MSATS-111"
        }
      ]
    },
    {
      "key": "MSATS-102",
      "summary": "Phase 2: Foundational (Blocking Prerequisites)",
      "url": "https://your-jira.atlassian.net/browse/MSATS-102",
      "tasks": [
        {
          "key": "MSATS-120",
          "id": "T010",
          "summary": "Generate libs/core scaffold",
          "status": "completed",
          "url": "https://your-jira.atlassian.net/browse/MSATS-120"
        }
      ]
    }
  ],
  "summary": {
    "total_stories": 10,
    "total_tasks": 94,
    "completed_tasks": 87,
    "pending_tasks": 7
  }
}
```

### 10. Display Summary

Output a complete summary:

```
═══════════════════════════════════════════════════════════════
✅ Jira Hierarchy Created Successfully!
═══════════════════════════════════════════════════════════════

📋 Project: MSATS
📁 Spec: 001-ts-msa-implementation

Epic: MSATS-100 - TypeScript MSA Framework Implementation
  └── https://your-jira.atlassian.net/browse/MSATS-100

Stories (10):
  ├── MSATS-101 - Phase 1: Setup (9 tasks)
  ├── MSATS-102 - Phase 2: Foundational (17 tasks)
  ├── MSATS-103 - Phase 3: User Story 1 (10 tasks)
  └── ... (7 more)

Summary:
  • Total Stories: 10
  • Total Tasks: 94
  • Completed: 87 (93%)
  • Pending: 7 (7%)

💾 Mapping saved to: specs/001-ts-msa-implementation/jira-mapping.json

Next steps:
  • View Epic in Jira: https://your-jira.atlassian.net/browse/MSATS-100
  • Sync status later: /speckit.jira.sync-status --spec 001-ts-msa-implementation
═══════════════════════════════════════════════════════════════
```

## Configuration Reference

Edit `.specify/extensions/jira/jira-config.yml` to customize:

| Config Key | Description | Default |
|------------|-------------|---------|
| `mcp_server` | MCP server name | "atlassian" |
| `project.key` | Jira project key | (required) |
| `hierarchy.epic_type` | Issue type for SPEC.md | "Epic" |
| `hierarchy.story_type` | Issue type for Phases | "Story" |
| `hierarchy.task_type` | Issue type for Tasks | "Sub-task" |
| `hierarchy.link_type` | Link type (if not subtask) | "Relates" |
| `defaults.epic.labels` | Labels for Epic | [] |
| `defaults.story.labels` | Labels for Stories | [] |
| `defaults.task.labels` | Labels for Tasks | [] |

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
