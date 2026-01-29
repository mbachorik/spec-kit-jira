---
description: "Create Jira hierarchy from spec and tasks"
tools:
  - 'jira-mcp-server/epic_create'
  - 'jira-mcp-server/issue_create'
  - 'jira-mcp-server/issue_link'
  - 'jira-mcp-server/issue_list'
---

# Create Jira Issues from Spec and Tasks

This command creates a complete Jira issue hierarchy from your specification and task breakdown:

- **Epic**: Created from SPEC.md (overall specification)
- **Tasks/Subtasks**: Created from TASKS.md (individual implementation tasks)
- **Links**: Automatic linking of tasks to epic

## Prerequisites

1. Jira MCP server configured and running
2. Jira configuration file exists: `.specify/extensions/jira/jira-config.yml`
3. Project has SPEC.md and TASKS.md files

## User Input

$ARGUMENTS

## Steps

### 1. Load Jira Configuration

Load the Jira configuration from the extension directory:

```bash
config_file=".specify/extensions/jira/jira-config.yml"

if [ ! -f "$config_file" ]; then
  echo "❌ Error: Jira configuration not found at $config_file"
  echo "Run 'specify extension add jira' to install and configure the extension"
  exit 1
fi

# Read configuration values
project_key=$(yq eval '.project.key' "$config_file")
issue_type=$(yq eval '.hierarchy.issue_type // "subtask"' "$config_file")
link_type=$(yq eval '.hierarchy.link_type // "Relates"' "$config_file")

# Apply environment variable overrides
project_key="${SPECKIT_JIRA_PROJECT_KEY:-$project_key}"
issue_type="${SPECKIT_JIRA_ISSUE_TYPE:-$issue_type}"
link_type="${SPECKIT_JIRA_LINK_TYPE:-$link_type}"

# Validate required fields
if [ -z "$project_key" ]; then
  echo "❌ Error: Jira project key not configured"
  echo "Edit $config_file and set 'project.key'"
  exit 1
fi

echo "📋 Jira Project: $project_key"
echo "🔗 Issue Type: $issue_type"
echo "🔗 Link Type: $link_type"
```

### 2. Read Specification

Read the SPEC.md file to extract the epic information:

```bash
spec_file="SPEC.md"

if [ ! -f "$spec_file" ]; then
  echo "❌ Error: Specification file not found: $spec_file"
  echo "Create a specification first with /speckit.spec"
  exit 1
fi

echo "📄 Reading specification from $spec_file..."
```

### 3. Read Tasks

Read the TASKS.md file to extract the task list:

```bash
tasks_file="TASKS.md"

if [ ! -f "$tasks_file" ]; then
  echo "❌ Error: Tasks file not found: $tasks_file"
  echo "Generate tasks first with /speckit.tasks"
  exit 1
fi

echo "📝 Reading tasks from $tasks_file..."
```

### 4. Create Epic from Specification

Use the jira-mcp-server MCP tool to create an Epic for the specification:

1. Extract the spec title and summary from SPEC.md
2. Call the jira-mcp-server tool to create an epic:
   - Summary: The spec title (first H1 heading)
   - Description: The spec overview/summary section
   - Project: The configured Jira project key
   - Labels: From jira-config.yml defaults.epic.labels

Example structure:

- **Epic Summary**: "User Authentication System"
- **Epic Description**: Full spec content or summary section

Store the created Epic key (e.g., "MSATS-123") for linking tasks.

### 5. Parse Tasks from TASKS.md

Parse the TASKS.md file to extract individual tasks. Expected format:

```markdown
# Tasks

## Task 1: Implement login endpoint
Description of task 1

## Task 2: Add password hashing
Description of task 2

...
```

Extract:

- Task title (from H2 headings)
- Task description (content under each heading)
- Task number/order

### 6. Create Jira Issues for Each Task

For each task parsed from TASKS.md:

1. Call jira-mcp-server to create an issue:
   - Issue Type: From configuration (default: "subtask" or "task")
   - Summary: Task title
   - Description: Task description
   - Project: The configured Jira project key
   - Parent: The Epic key (if using subtasks)
   - Labels: From jira-config.yml defaults.task.labels

2. Store the created issue key for each task

### 7. Link Tasks to Epic

For each created task issue:

1. Use jira-mcp-server to create a link:
   - From: Task issue key
   - To: Epic issue key
   - Link Type: From configuration (default: "Relates")

This ensures all tasks are properly associated with the epic.

### 8. Display Summary

Output a summary of created issues:

```bash
echo ""
echo "✅ Jira issues created successfully!"
echo ""
echo "Epic: $project_key-XXX - [Epic Title]"
echo "URL: https://your-jira-instance.atlassian.net/browse/$project_key-XXX"
echo ""
echo "Created ${task_count} tasks:"
for task_key in "${task_keys[@]}"; do
  echo "  • $task_key - [Task Title]"
  echo "    https://your-jira-instance.atlassian.net/browse/$task_key"
done
echo ""
echo "All tasks linked to epic with '${link_type}' relationship"
```

### 9. Save Issue Mapping

Save a mapping file for future reference:

```bash
mapping_file=".specify/jira-mapping.json"

cat > "$mapping_file" <<EOF
{
  "created_at": "$(date -u +"%Y-%m-%dT%H:%M:%SZ")",
  "project": "$project_key",
  "epic": {
    "key": "$epic_key",
    "summary": "$epic_summary"
  },
  "tasks": [
    $(for i in "${!task_keys[@]}"; do
      echo "    {\"key\": \"${task_keys[$i]}\", \"summary\": \"${task_summaries[$i]}\"}"
      [ $i -lt $((${#task_keys[@]} - 1)) ] && echo ","
    done)
  ]
}
EOF

echo "💾 Mapping saved to $mapping_file"
```

## Configuration Reference

Edit `.specify/extensions/jira/jira-config.yml` to customize:

- **project.key**: Your Jira project key (required)
- **hierarchy.issue_type**: Type of issues to create for tasks ("subtask", "task", or "story")
- **hierarchy.link_type**: How to link tasks to epic ("Relates", "Blocks", "Implements")
- **defaults.epic.labels**: Labels to apply to epic
- **defaults.task.labels**: Labels to apply to all tasks

## Troubleshooting

### "Jira configuration not found"

Run `specify extension add jira` to install the extension and create the config template.

### "Jira project key not configured"

Edit `.specify/extensions/jira/jira-config.yml` and set `project.key` to your Jira project key.

### "MCP tool not available"

Ensure jira-mcp-server is configured in your AI agent's MCP settings.

### Custom Fields

Use `/speckit.jira.discover-fields` to discover available custom fields in your Jira instance, then add them to `jira-config.yml`.

## Notes

- This command requires the jira-mcp-server MCP tool to be configured
- Epic and task creation happen in sequence to ensure proper linking
- The mapping file (.specify/jira-mapping.json) can be used for status syncing
- You can re-run this command to create a new hierarchy (it won't update existing issues)
