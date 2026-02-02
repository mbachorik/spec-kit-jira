# Spec Kit - Jira Integration Extension

[![Spec Kit](https://img.shields.io/badge/spec--kit-extension-blue?logo=github)](https://github.com/github/spec-kit)
[![Version](https://img.shields.io/badge/version-2.0.0-green)](https://github.com/mbachorik/spec-kit-jira/releases)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Issues](https://img.shields.io/github/issues/mbachorik/spec-kit-jira)](https://github.com/mbachorik/spec-kit-jira/issues)

Create Jira Epics, Stories, and Issues directly from your spec-kit specifications and task breakdowns.

## Features

- **Automatic Hierarchy Creation**: Convert SPEC.md → Epic and TASKS.md → Tasks/Subtasks
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
  issue_type: "subtask"  # or "task", "story"
  link_type: "Relates"   # or "Blocks", "Implements"

defaults:
  epic:
    labels: ["spec-driven", "automated"]
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
3. Create Tasks/Subtasks from `specs/<spec-name>/tasks.md`
4. Link all tasks to the epic
5. Save mapping to `specs/<spec-name>/jira-mapping.json`

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
  issue_type: "subtask"
  link_type: "Relates"

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

# Field Mappings
field_mappings:
  spec_version: "customfield_10005"
  team: "customfield_10006"

# Workflow Configuration
workflow:
  done_status: "Done"
  done_transition: "Done"
```

### Environment Variable Overrides

```bash
# Override MCP server name
export SPECKIT_JIRA_MCP_SERVER="atlassian"

# Override project key
export SPECKIT_JIRA_PROJECT_KEY="DEVTEST"

# Override issue type
export SPECKIT_JIRA_ISSUE_TYPE="task"

# Override link type
export SPECKIT_JIRA_LINK_TYPE="Blocks"
```

### Local Overrides (Gitignored)

Create `.specify/extensions/jira/jira-config.local.yml` for local testing:

```yaml
project:
  key: "MYTEST"  # Override for local development
```

## Task Completion Markers

Mark tasks as complete in TASKS.md using:

1. **Checkmark emoji**: `## Task 1: Title ✅`
2. **Checkbox**: `## Task 1: Title [x]`
3. **Status prefix**: `## Task 1: [DONE] Title`

Example:

```markdown
# Tasks

## Task 1: Implement authentication ✅
Description of completed task

## Task 2: Add error handling
In-progress task

## Task 3: [DONE] Write tests
Another completed task
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

### Example 2: With Custom Fields

```yaml
project:
  key: "MSATS"

defaults:
  task:
    custom_fields:
      customfield_10002: 3  # Story points
      customfield_10004: "Backend Team"
```

### Example 3: Complete Workflow

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
