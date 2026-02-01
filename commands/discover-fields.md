---
description: "Discover Jira custom fields for configuration"
tools:
  # Server name is configurable via mcp_server in jira-config.yml (default: "atlassian")
  - '{mcp_server}/field_list'
  - '{mcp_server}/project_get'
---

# Discover Jira Custom Fields

This command helps you discover custom fields in your Jira instance and generates configuration snippets for `jira-config.yml`.

## Purpose

Different Jira instances have different custom fields based on your organization's configuration. This command:

1. Lists all available fields in your Jira project
2. Identifies custom fields (customfield_*)
3. Generates configuration snippets for common mappings

## Prerequisites

1. MCP server providing Jira tools configured and running (server name configured in jira-config.yml)
2. Jira configuration file exists: `.specify/extensions/jira/jira-config.yml`
3. Valid Jira project key configured

## User Input

$ARGUMENTS

## Steps

### 1. Load Jira Configuration

```bash
config_file=".specify/extensions/jira/jira-config.yml"

if [ ! -f "$config_file" ]; then
  echo "❌ Error: Jira configuration not found at $config_file"
  echo "Run 'specify extension add jira' to install and configure the extension"
  exit 1
fi

# Read configuration values
mcp_server=$(yq eval '.mcp_server // "atlassian"' "$config_file")
project_key=$(yq eval '.project.key' "$config_file")

# Apply environment variable overrides
mcp_server="${SPECKIT_JIRA_MCP_SERVER:-$mcp_server}"
project_key="${SPECKIT_JIRA_PROJECT_KEY:-$project_key}"

if [ -z "$project_key" ]; then
  echo "❌ Error: Jira project key not configured"
  echo "Edit $config_file and set 'project.key'"
  exit 1
fi

echo "🔌 MCP Server: $mcp_server"
echo "🔍 Discovering fields for Jira project: $project_key"
echo ""
```

### 2. Fetch Project Information

Use the configured MCP server to get project details:

```markdown
Call the the configured MCP server MCP tool to get project information:
- Tool: project_get
- Parameters: { "project_key": "$project_key" }

This verifies the project exists and is accessible.
```

### 3. List All Fields

Use the configured MCP server to fetch all available fields:

```markdown
Call the the configured MCP server MCP tool to list fields:
- Tool: field_list
- Parameters: {}

This returns all fields available in the Jira instance, including:
- Standard fields (summary, description, assignee, etc.)
- Custom fields (customfield_10001, customfield_10002, etc.)
```

### 4. Parse and Display Custom Fields

Parse the field list and display custom fields in a user-friendly format:

```bash
echo "📋 Available Custom Fields:"
echo ""
echo "Standard Fields:"
echo "  • summary           - Issue summary/title"
echo "  • description       - Issue description"
echo "  • assignee          - Issue assignee"
echo "  • priority          - Issue priority"
echo "  • labels            - Issue labels"
echo "  • issuetype         - Issue type"
echo "  • project           - Project key"
echo ""
echo "Custom Fields (available in your Jira instance):"
echo ""

# Example output (replace with actual discovered fields):
# echo "  • customfield_10001 - Sprint"
# echo "  • customfield_10002 - Story Points"
# echo "  • customfield_10003 - Epic Link"
# echo "  • customfield_10004 - Team"
# etc...
```

### 5. Generate Configuration Snippets

Based on discovered custom fields, generate configuration snippets for common use cases:

```bash
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "📝 Configuration Snippets"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "Add these to your jira-config.yml to use custom fields:"
echo ""
echo "# Field Mappings"
echo "field_mappings:"

# If Sprint field found
if [ -n "$sprint_field" ]; then
  echo "  # Map tasks to Sprint"
  echo "  sprint: \"$sprint_field\""
  echo ""
fi

# If Story Points field found
if [ -n "$story_points_field" ]; then
  echo "  # Map task complexity to Story Points"
  echo "  story_points: \"$story_points_field\""
  echo ""
fi

# If Epic Link field found
if [ -n "$epic_link_field" ]; then
  echo "  # Link tasks to Epic (alternative to parent relationship)"
  echo "  epic_link: \"$epic_link_field\""
  echo ""
fi

# If Team field found
if [ -n "$team_field" ]; then
  echo "  # Assign to Team"
  echo "  team: \"$team_field\""
  echo ""
fi

echo "# Apply custom field values to created issues"
echo "defaults:"
echo "  epic:"
echo "    custom_fields:"
if [ -n "$story_points_field" ]; then
  echo "      $story_points_field: 0  # Epic story points"
fi
echo ""
echo "  task:"
echo "    custom_fields:"
if [ -n "$story_points_field" ]; then
  echo "      $story_points_field: 1  # Default story points per task"
fi
if [ -n "$team_field" ]; then
  echo "      $team_field: \"Engineering\"  # Default team"
fi
echo ""
```

### 6. Show Examples

Provide examples of how to use the discovered fields:

```bash
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "📚 Usage Examples"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "1. To set custom fields for all epics:"
echo ""
echo "   defaults:"
echo "     epic:"
echo "       custom_fields:"
echo "         customfield_10001: \"Sprint 1\""
echo "         customfield_10002: 5"
echo ""
echo "2. To set custom fields for all tasks:"
echo ""
echo "   defaults:"
echo "     task:"
echo "       custom_fields:"
echo "         customfield_10002: 2"
echo "         customfield_10004: \"Backend Team\""
echo ""
echo "3. To map spec concepts to custom fields:"
echo ""
echo "   field_mappings:"
echo "     spec_version: \"customfield_10005\""
echo "     implementation_phase: \"customfield_10006\""
echo ""
```

### 7. Save Field List

Optionally save the discovered fields to a reference file:

```bash
output_file=".specify/extensions/jira/discovered-fields.json"

echo "{" > "$output_file"
echo "  \"discovered_at\": \"$(date -u +"%Y-%m-%dT%H:%M:%SZ")\"," >> "$output_file"
echo "  \"project\": \"$project_key\"," >> "$output_file"
echo "  \"fields\": [" >> "$output_file"

# Add field list (pseudo-code, replace with actual field data)
# for field in fields; do
#   echo "    {\"id\": \"$field_id\", \"name\": \"$field_name\", \"type\": \"$field_type\"}," >> "$output_file"
# done

echo "  ]" >> "$output_file"
echo "}" >> "$output_file"

echo "💾 Field list saved to $output_file"
echo ""
```

## Output Format

The command outputs:

1. **Standard Fields**: Core Jira fields available in all instances
2. **Custom Fields**: Organization-specific custom fields
3. **Configuration Snippets**: Ready-to-use YAML snippets
4. **Usage Examples**: How to apply custom fields
5. **Reference File**: JSON file with complete field list

## Next Steps

After discovering fields:

1. Copy relevant snippets to `.specify/extensions/jira/jira-config.yml`
2. Adjust values to match your project needs
3. Run `/speckit.jira.specstoissues` to create issues with custom fields

## Notes

- Custom fields vary by Jira instance and project type
- Some fields may be read-only or have restricted values
- Field IDs (customfield_*) are stable, but names may change
- You can re-run this command anytime to discover new fields
