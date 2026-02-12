# Spec Kit - Jira Integration Extension

[![Spec Kit](https://img.shields.io/badge/spec--kit-extension-blue?logo=github)](https://github.com/github/spec-kit)
[![Version](https://img.shields.io/badge/version-2.1.0-green)](https://github.com/mbachorik/spec-kit-jira/releases)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Issues](https://img.shields.io/github/issues/mbachorik/spec-kit-jira)](https://github.com/mbachorik/spec-kit-jira/issues)

Create Jira Epics, Stories, and Issues directly from your spec-kit specifications and task breakdowns.

## Features

- **3-Level Hierarchy**: Convert SPEC.md → Epic, Phase headers → Stories, Tasks → Task issues
- **2-Level Mode**: Optional simplified mode (Epic → Stories with embedded task checklists)
- **Custom Field Discovery**: Discover and configure Jira custom fields
- **Status Synchronization**: Keep local task status in sync with Jira
- **Flexible Configuration**: Project-level config with local overrides and environment variables
- **MCP Integration**: Works with any MCP server providing Jira/Atlassian tools (configurable)

## Installation

### Prerequisites

1. **Spec Kit** version 0.1.0 or higher
2. **MCP server providing Jira tools** configured in your AI agent (e.g., "atlassian", "jira-mcp-server")
3. Valid Jira account with project access

### Install Extension

```bash
# From within a spec-kit project
specify extension add jira

# Or install from local development directory
specify extension add --dev /path/to/spec-kit-jira
```

## Configuration

### 1. Set MCP Server and Jira Project Key

Edit `.specify/extensions/jira/jira-config.yml`:

```yaml
# MCP server providing Jira tools (default: "atlassian")
mcp_server: "atlassian"

project:
  key: "MYPROJECT"  # Replace with your Jira project key
```

### 2. Discover Custom Fields (Optional)

```bash
claude
> /speckit.jira.discover-fields
```

This will show all available custom fields in your Jira instance and generate configuration snippets.

### 3. Customize Configuration

```yaml
# .specify/extensions/jira/jira-config.yml

project:
  key: "MSATS"

hierarchy:
  epic_type: "Epic"       # Issue type for SPEC.md
  story_type: "Story"     # Issue type for Phase headers
  task_type: "Task"       # Issue type for tasks (set to "" for 2-level mode)

  relationships:
    epic_story: "Epic Link"   # How Story links to Epic
    story_task: "Relates"     # How Task links to Story
    epic_task: "Epic Link"    # Direct Task-Epic link

defaults:
  epic:
    labels: ["spec-driven", "automated"]
    custom_fields: {}

  story:
    labels: []
    custom_fields: {}

  task:
    labels: ["implementation"]
    custom_fields:
      customfield_10002: 2  # Story points
```

## Usage

### Create Jira Issues from Spec and Tasks

After creating SPEC.md and TASKS.md with spec-kit:

```bash
claude
> /speckit.jira.specstoissues
```

This will:

1. Auto-detect spec from git branch name, current directory, or prompt if multiple exist
2. Create a Jira Epic from `specs/<spec-name>/spec.md`
3. Create Stories from Phase headers in `specs/<spec-name>/tasks.md` (e.g., `## Phase 1: ...`)
4. Create Tasks from task items under each Phase (e.g., `- [ ] T001 ...`)
5. Link all issues according to configured relationships
6. Save mapping to `specs/<spec-name>/jira-mapping.json`

**Hierarchy Modes:**

- **3-level mode** (default): Epic → Stories → Tasks
- **2-level mode** (set `task_type: ""`): Epic → Stories with task checklists embedded in descriptions

To specify a particular spec:

```bash
> /speckit.jira.specstoissues --spec 005-python-endpoint-alignment
```

### Discover Custom Fields

```bash
claude
> /speckit.jira.discover-fields
```

Outputs:

- All available custom fields in your Jira instance
- Configuration snippets for common mappings
- Examples of how to use custom fields

### Sync Task Completion Status

After completing tasks locally, sync status to Jira:

```bash
claude
> /speckit.jira.sync-status
```

This will:

1. Read task completion from TASKS.md
2. Update Jira issue statuses
3. Transition issues to "Done" state
4. Update epic progress

## Commands

### `/speckit.jira.specstoissues`

Create complete Jira issue hierarchy from spec and tasks.

**Arguments:**

- `--spec <name>` (optional): Specification name to use. Auto-detects if not provided.

**Prerequisites:**

- Specification directory exists: `specs/<spec-name>/`
- `spec.md` file exists in the specification directory
- `tasks.md` file exists in the specification directory
- Jira project key configured

**Output:**

- Epic created from spec
- Tasks created from task list
- All tasks linked to epic
- Mapping file: `specs/<spec-name>/jira-mapping.json`

### `/speckit.jira.discover-fields`

Discover available custom fields in Jira instance.

**Prerequisites:**

- Jira project key configured
- MCP server providing Jira tools configured

**Output:**

- List of custom fields
- Configuration snippets
- Usage examples
- Reference file: `.specify/extensions/jira/discovered-fields.json`

### `/speckit.jira.sync-status`

Sync local task completion to Jira.

**Arguments:**

- `--spec <name>` (optional): Specification name to sync. Auto-detects if not provided.

**Prerequisites:**

- Issues created via `/speckit.jira.specstoissues`
- Mapping file exists: `specs/<spec-name>/jira-mapping.json`
- `tasks.md` has completion markers

**Output:**

- Updated Jira issue statuses
- Epic progress calculation
- Sync log: `specs/<spec-name>/jira-sync-log.json`

## Configuration Reference

### Full Configuration Example

```yaml
# .specify/extensions/jira/jira-config.yml

# MCP Server Configuration
mcp_server: "atlassian"  # or "jira-mcp-server", "jira", etc.

# Jira Project Configuration
project:
  key: "MSATS"

# Issue Hierarchy
hierarchy:
  # Issue types to create
  epic_type: "Epic"       # Issue type for SPEC.md
  story_type: "Story"     # Issue type for Phase headers in TASKS.md
  task_type: "Task"       # Issue type for task items
                          # Set to "" or "none" for 2-level mode (Epic → Stories only)

  # Relationships between issues
  # Options: "Parent", "Epic Link", "Relates", "Blocks", "Implements", "is child of", "none"
  relationships:
    epic_story: "Epic Link"   # How Story connects to Epic
    story_task: "Relates"     # How Task connects to Story
    epic_task: "Epic Link"    # Direct Task-to-Epic link

# Default Values
defaults:
  epic:
    labels: ["spec-driven", "microservice"]
    custom_fields:
      customfield_10001: "Sprint 1"

  story:
    labels: []
    custom_fields: {}

  task:
    labels: ["implementation"]
    custom_fields:
      customfield_10002: 2  # Story points

# Field Mappings (discovered via /speckit.jira.discover-fields)
field_mappings:
  spec_version: "customfield_10005"
  team: "customfield_10006"

# Status Mapping for sync-status command
status_mapping:
  completed: "Done"           # [x] in TASKS.md
  pending: "To Do"            # [ ] in TASKS.md
  in_progress: "In Progress"  # [~] in TASKS.md (optional)
```

### Environment Variable Overrides

```bash
# Override MCP server name
export SPECKIT_JIRA_MCP_SERVER="atlassian"

# Override project key
export SPECKIT_JIRA_PROJECT_KEY="DEVTEST"

# Override issue types
export SPECKIT_JIRA_EPIC_TYPE="Epic"
export SPECKIT_JIRA_STORY_TYPE="Story"
export SPECKIT_JIRA_TASK_TYPE="Task"

# Override relationships
export SPECKIT_JIRA_EPIC_STORY_RELATIONSHIP="Epic Link"
export SPECKIT_JIRA_STORY_TASK_RELATIONSHIP="Relates"
export SPECKIT_JIRA_EPIC_TASK_RELATIONSHIP="Epic Link"
```

### Local Overrides (Gitignored)

Create `.specify/extensions/jira/jira-config.local.yml` for local testing:

```yaml
project:
  key: "MYTEST"  # Override for local development
```

## Task Completion Markers

Mark tasks in TASKS.md using checkbox syntax:

| Marker  | Status      | Jira Status (default) |
| ------- | ----------- | --------------------- |
| `- [x]` | Completed   | Done                  |
| `- [ ]` | Pending     | To Do                 |
| `- [~]` | In Progress | In Progress           |

Example:

```markdown
# Tasks

## Phase 1: Authentication

- [x] T001: Implement login endpoint
- [~] T002: Add session management
- [ ] T003: Write authentication tests

## Phase 2: Error Handling

- [ ] T004: Add global error handler
- [ ] T005: Implement retry logic
```

Configure status mappings in `jira-config.yml`:

```yaml
status_mapping:
  completed: "Done"
  pending: "To Do"
  in_progress: "In Progress"
```

## Troubleshooting

### "Jira configuration not found"

**Solution**: Run `specify extension add jira` to install the extension and create config template.

### "Jira project key not configured"

**Solution**: Edit `.specify/extensions/jira/jira-config.yml` and set `project.key`.

### "MCP tool not available"

**Solution**: Ensure your MCP server providing Jira tools is configured in your AI agent's MCP settings, and verify the `mcp_server` name in jira-config.yml matches.

### "Issue not found" or "Permission denied"

**Solution**: Verify your Jira credentials and project permissions in your MCP server configuration.

### Custom fields not working

**Solution**:

1. Run `/speckit.jira.discover-fields` to find correct field IDs
2. Verify field IDs in configuration
3. Check field is available for your issue type

## Examples

### Example 1: Simple Project

```yaml
# Minimal configuration
project:
  key: "DEMO"
```

Then:

```bash
> /speckit.jira.specstoissues
```

### Example 2: With Custom Fields and 3-Level Hierarchy

```yaml
project:
  key: "MSATS"

hierarchy:
  epic_type: "Epic"
  story_type: "Story"
  task_type: "Task"
  relationships:
    epic_story: "Epic Link"
    story_task: "Relates"
    epic_task: "Epic Link"

defaults:
  task:
    custom_fields:
      customfield_10002: 3  # Story points
      customfield_10004: "Backend Team"
```

### Example 3: 2-Level Mode (Epic → Stories Only)

```yaml
project:
  key: "SIMPLE"

hierarchy:
  epic_type: "Epic"
  story_type: "Story"
  task_type: ""  # Empty = 2-level mode, tasks embedded as checklists

defaults:
  story:
    labels: ["auto-generated"]
```

### Example 4: Complete Workflow

```bash
# 1. Create spec and tasks
> /speckit.spec
> /speckit.tasks

# 2. Discover Jira fields
> /speckit.jira.discover-fields

# 3. Configure jira-config.yml
# (edit file)

# 4. Create Jira issues
> /speckit.jira.specstoissues

# 5. Implement tasks locally
# (mark tasks complete in TASKS.md)

# 6. Sync status to Jira
> /speckit.jira.sync-status
```

## Development

### Repository Structure

```text
spec-kit-jira/
├── README.md
├── LICENSE
├── CHANGELOG.md
├── extension.yml              # Extension manifest
├── jira-config.template.yml   # Config template
├── commands/
│   ├── specstoissues.md
│   ├── discover-fields.md
│   └── sync-status.md
└── docs/
    └── examples/
```

### Testing Locally

```bash
# Install in development mode
cd /path/to/your/project
specify extension add --dev /path/to/spec-kit-jira

# Make changes to extension
# Commands automatically reload

# Remove and reinstall to test install flow
specify extension remove jira
specify extension add --dev /path/to/spec-kit-jira
```

## License

MIT License - see LICENSE file

## Contributing

Contributions welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## Support

- **Issues**: <https://github.com/mbachorik/spec-kit-jira/issues>
- **Spec Kit Docs**: <https://github.com/github/spec-kit>

## Related Extensions

- **spec-kit-linear**: Linear integration
- **spec-kit-azure-devops**: Azure DevOps integration
- **spec-kit-github**: GitHub Issues integration
