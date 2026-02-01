---
description: "Sync task completion status to Jira"
tools:
  # Server name is configurable via mcp_server in jira-config.yml (default: "atlassian")
  - '{mcp_server}/issue_update'
  - '{mcp_server}/issue_get'
  - '{mcp_server}/transition_list'
  - '{mcp_server}/issue_transition'
---

# Sync Task Completion Status to Jira

This command syncs task completion status from your local tasks.md file to Jira, updating issue statuses and progress.

## Purpose

As you implement tasks locally, you can mark them as completed in tasks.md. This command:

1. Reads the local task status from `specs/<spec-name>/tasks.md`
2. Reads the Jira issue mapping from `specs/<spec-name>/jira-mapping.json`
3. Updates Jira issue statuses to match local completion
4. Optionally transitions issues through workflow states

## Prerequisites

1. MCP server providing Jira tools configured and running (server name configured in jira-config.yml)
2. Issues already created via `/speckit.jira.specstoissues`
3. Mapping file exists: `specs/<spec-name>/jira-mapping.json`
4. tasks.md file with completion markers

## User Input

$ARGUMENTS

Accepts optional `--spec <name>` argument to specify which specification to sync.
If not provided, auto-detects from current directory or available specs.

## Steps

### 1. Detect Specification Directory

Determine which specification to sync (in order of priority):

1. `--spec <name>` argument
2. Git branch name (if matches a spec with Jira mapping)
3. Current directory (if inside `specs/<name>/`)
4. Single spec with Jira mapping (if only one exists)

```bash
# Parse --spec argument if provided
spec_name=""
for arg in "$@"; do
  if [[ "$prev_arg" == "--spec" ]]; then
    spec_name="$arg"
  fi
  prev_arg="$arg"
done

# Auto-detection logic
if [ -z "$spec_name" ]; then
  # Try to detect from git branch name
  # Common patterns: feature/005-spec-name, 005-spec-name, spec/005-spec-name
  if command -v git &> /dev/null && git rev-parse --git-dir &> /dev/null; then
    branch_name=$(git branch --show-current 2>/dev/null)
    if [ -n "$branch_name" ]; then
      # Extract spec name from branch (remove common prefixes like feature/, spec/, etc.)
      potential_spec=$(echo "$branch_name" | sed -E 's#^(feature|spec|bugfix|hotfix|release)/##')
      if [ -f "specs/$potential_spec/jira-mapping.json" ]; then
        spec_name="$potential_spec"
        echo "📍 Auto-detected spec from git branch: $spec_name"
      fi
    fi
  fi
fi

if [ -z "$spec_name" ]; then
  # Check if current directory is inside a spec folder
  current_dir=$(pwd)
  if [[ "$current_dir" =~ specs/([^/]+) ]]; then
    spec_name="${BASH_REMATCH[1]}"
    echo "📍 Auto-detected spec from current directory: $spec_name"
  # Check if there's exactly one spec with a jira-mapping.json
  elif [ -d "specs" ]; then
    mapped_specs=$(find specs -maxdepth 2 -name "jira-mapping.json" 2>/dev/null | wc -l | tr -d ' ')
    if [ "$mapped_specs" -eq 1 ]; then
      spec_name=$(dirname "$(find specs -maxdepth 2 -name "jira-mapping.json")" | xargs basename)
      echo "📍 Auto-detected spec with Jira mapping: $spec_name"
    elif [ "$mapped_specs" -eq 0 ]; then
      echo "❌ Error: No specifications with Jira mapping found"
      echo "Run /speckit.jira.specstoissues first to create Jira issues"
      exit 1
    else
      echo "❌ Error: Multiple specifications have Jira mappings. Please specify which one to sync:"
      echo ""
      find specs -maxdepth 2 -name "jira-mapping.json" | while read mapping; do
        echo "  --spec $(basename "$(dirname "$mapping")")"
      done
      echo ""
      echo "Usage: /speckit.jira.sync-status --spec <name>"
      exit 1
    fi
  else
    echo "❌ Error: No specs/ directory found"
    exit 1
  fi
fi

# Validate spec directory and mapping exist
spec_dir="specs/$spec_name"
mapping_file="$spec_dir/jira-mapping.json"

if [ ! -d "$spec_dir" ]; then
  echo "❌ Error: Specification directory not found: $spec_dir"
  exit 1
fi

if [ ! -f "$mapping_file" ]; then
  echo "❌ Error: Jira mapping not found: $mapping_file"
  echo "Run /speckit.jira.specstoissues --spec $spec_name first to create Jira issues"
  exit 1
fi

echo "📂 Using specification: $spec_name"
echo "   Directory: $spec_dir"
echo ""
```

### 2. Load Configuration

```bash
config_file=".specify/extensions/jira/jira-config.yml"

if [ ! -f "$config_file" ]; then
  echo "❌ Error: Jira configuration not found at $config_file"
  exit 1
fi

# Read configuration values
mcp_server=$(yq eval '.mcp_server // "atlassian"' "$config_file")
project_key=$(yq eval '.project.key' "$config_file")

# Apply environment variable overrides
mcp_server="${SPECKIT_JIRA_MCP_SERVER:-$mcp_server}"
project_key="${SPECKIT_JIRA_PROJECT_KEY:-$project_key}"

echo "🔌 MCP Server: $mcp_server"
echo "🔄 Syncing task status to Jira project: $project_key"
echo ""
```

### 3. Parse tasks.md for Completion Status

Read tasks.md and identify completed tasks. Expected format:

```markdown
# Tasks

## Task 1: Implement login endpoint ✅
Description of completed task

## Task 2: Add password hashing
Description of in-progress task

## Task 3: Create user registration ✅
Description of completed task
```

Parse tasks and extract:

- Task number/identifier
- Completion status (✅ or checkbox [x] indicates completed)

```bash
tasks_file="$spec_dir/tasks.md"

if [ ! -f "$tasks_file" ]; then
  echo "❌ Error: Tasks file not found: $tasks_file"
  exit 1
fi

# Parse completed tasks (pseudo-code)
# completed_tasks=$(grep -E "^## Task.*✅|^## Task.*\[x\]" "$tasks_file" | sed 's/^## Task \([0-9]*\).*/\1/')

echo "📝 Parsing task completion from $tasks_file..."
```

### 4. Load Jira Issue Mapping

Read the mapping file to get Jira issue keys for each task:

```bash
# Read mapping (pseudo-code)
# epic_key=$(jq -r '.epic.key' "$mapping_file")
# task_keys=$(jq -r '.tasks[].key' "$mapping_file")

echo "📋 Loading issue mappings from $mapping_file..."
```

### 5. Get Available Transitions

For each issue, query available workflow transitions:

```markdown
Call the configured MCP server to get available transitions:
- Tool: transition_list
- Parameters: { "issue_key": "$task_key" }

Common transitions:
- "To Do" → "In Progress" → "Done"
- "Open" → "In Progress" → "Resolved" → "Closed"

Identify the transition ID for moving to "Done" or "Closed" state.
```

### 6. Update Jira Issue Statuses

For each task in the mapping:

1. Check if task is marked as completed locally
2. Get current Jira issue status
3. If local status is completed but Jira status is not:
   - Transition issue to "Done" or appropriate completion state
   - Update issue with completion comment

```bash
echo "🔄 Syncing statuses..."
echo ""

# For each task (pseudo-code):
# for i in "${!task_keys[@]}"; do
#   task_num=$((i + 1))
#   task_key="${task_keys[$i]}"
#
#   # Check if task is completed locally
#   if [[ " ${completed_tasks[@]} " =~ " ${task_num} " ]]; then
#     echo "  ✓ Task $task_num ($task_key) - Marking as Done"
#
#     # Call MCP tool to transition issue
#     # Tool: issue_transition
#     # Parameters: { "issue_key": "$task_key", "transition": "Done" }
#
#     # Add comment about completion
#     # Tool: issue_update
#     # Parameters: { "issue_key": "$task_key", "comment": "Task completed locally via spec-kit" }
#   else
#     echo "  ○ Task $task_num ($task_key) - In Progress"
#   fi
# done
```

### 7. Update Epic Progress

Calculate overall completion and update epic:

```bash
# Calculate completion percentage
# total_tasks=${#task_keys[@]}
# completed_count=${#completed_tasks[@]}
# completion_pct=$((completed_count * 100 / total_tasks))

echo ""
echo "📊 Overall Progress: $completed_count / $total_tasks tasks ($completion_pct%)"

# Update epic description with progress (optional)
# Tool: issue_update
# Parameters: { "issue_key": "$epic_key", "description": "...\\n\\nProgress: $completion_pct% ($completed_count/$total_tasks tasks completed)" }
```

### 8. Display Summary

```bash
echo ""
echo "✅ Status sync completed!"
echo ""
echo "Epic: $epic_key"
echo "  Progress: $completion_pct% ($completed_count / $total_tasks tasks)"
echo ""
echo "Updated Tasks:"
for task_key in "${updated_tasks[@]}"; do
  echo "  • $task_key - Transitioned to Done"
done
echo ""

if [ ${#skipped_tasks[@]} -gt 0 ]; then
  echo "Skipped (already done):"
  for task_key in "${skipped_tasks[@]}"; do
    echo "  • $task_key"
  done
  echo ""
fi
```

### 9. Log Sync Activity

Save sync activity to a log file in the specification directory:

```bash
log_file="$spec_dir/jira-sync-log.json"

cat >> "$log_file" <<EOF
{
  "synced_at": "$(date -u +"%Y-%m-%dT%H:%M:%SZ")",
  "spec": "$spec_name",
  "project": "$project_key",
  "epic": "$epic_key",
  "total_tasks": $total_tasks,
  "completed_tasks": $completed_count,
  "completion_pct": $completion_pct,
  "updated_issues": [$(printf '"%s",' "${updated_tasks[@]}" | sed 's/,$//')],
  "skipped_issues": [$(printf '"%s",' "${skipped_tasks[@]}" | sed 's/,$//')] }
EOF

echo "💾 Sync log saved to $log_file"
```

## Task Completion Markers

This command recognizes several formats for marking tasks as complete:

1. **Checkmark emoji**: `## Task 1: Title ✅`
2. **Checkbox syntax**: `## Task 1: Title [x]`
3. **Status prefix**: `## Task 1: [DONE] Title`

Example tasks.md:

```markdown
# Tasks

## Task 1: Implement authentication ✅
Completed task description

## Task 2: Add error handling [x]
Another completed task

## Task 3: [DONE] Write tests
Yet another completed task

## Task 4: Deploy to staging
In-progress task (not marked)
```

## Workflow Transitions

Different Jira projects have different workflows. Common patterns:

**Simple workflow:**

- To Do → Done

**Standard workflow:**

- To Do → In Progress → Done

**Complex workflow:**

- Backlog → Selected for Development → In Progress → Code Review → Testing → Done

The command automatically detects available transitions and uses the appropriate "completion" transition (Done, Closed, Resolved, etc.).

## Configuration

Edit `.specify/extensions/jira/jira-config.yml` to customize:

```yaml
# Add workflow customization (optional)
workflow:
  # Name of the "completed" status in your workflow
  done_status: "Done"  # or "Closed", "Resolved", etc.

  # Transition name to use
  done_transition: "Done"  # or "Close Issue", "Resolve", etc.
```

## Notes

- This command is idempotent - running it multiple times is safe
- Only incomplete → complete transitions are performed (not reversible)
- The epic progress is calculated from the mapping, not queried from Jira
- Use this after implementing tasks and before final review

## Troubleshooting

### "Mapping file not found"

Run `/speckit.jira.specstoissues --spec <name>` first to create the initial issue hierarchy for the specification.

### "Multiple specifications have Jira mappings"

Use `--spec <name>` to specify which specification to sync:

```bash
/speckit.jira.sync-status --spec 005-python-endpoint-alignment
```

### "Transition not found"

Your Jira workflow may use different transition names. Check your project's workflow configuration in Jira.

### "Issue not found"

The issue may have been deleted in Jira. Re-run `/speckit.jira.specstoissues --spec <name>` to recreate.
