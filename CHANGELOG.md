# Changelog

All notable changes to the Jira Integration Extension will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [2.0.0] - 2026-02-01

### Added

- **3-level hierarchy support**: Now creates Epic → Stories → Tasks
  - Epic: Created from SPEC.md (overall specification)
  - Stories: Created from Phase headers in TASKS.md (`## Phase X: ...`)
  - Tasks: Created from task items under each Phase (`- [ ] TXXX ...`)
- **Configurable relationships** for Team-managed and Company-managed Jira projects:
  - `hierarchy.relationships.epic_story`: How Story links to Epic (default: "Epic Link")
  - `hierarchy.relationships.story_task`: How Task links to Story (default: "Parent")
  - `hierarchy.relationships.epic_task`: Direct Task-Epic link (default: "none")
  - Options: "Parent", "Epic Link", "Relates", "Blocks", "Implements", "is child of", "none"
- New hierarchy configuration options:
  - `hierarchy.epic_type`: Issue type for SPEC.md (default: "Epic")
  - `hierarchy.story_type`: Issue type for Phases (default: "Story")
  - `hierarchy.task_type`: Issue type for Tasks (default: "Task")
- Enhanced jira-mapping.json structure with nested stories and tasks
- Status mapping configuration for sync-status command

### Changed

- **Breaking**: Config structure changed from `hierarchy.issue_type` to `epic_type`, `story_type`, `task_type`
- **Breaking**: `hierarchy.link_type` replaced by `hierarchy.relationships` section
- Default `task_type` changed from "Sub-task" to "Task"
- Default Epic-Story relationship is "Epic Link" (Company-managed compatible)
- jira-mapping.json now includes stories array with nested tasks
- Updated specstoissues command to parse Phase headers from TASKS.md

### Backward Compatibility

Old v1.x configs are automatically supported:
- `hierarchy.issue_type` → maps to `hierarchy.task_type`
- `hierarchy.link_type` → maps to `hierarchy.relationships.story_task`
- Missing relationship configs use defaults

### Migration (optional)

To use new config format:
1. Update `jira-config.yml`:
   - Replace `hierarchy.issue_type` with `hierarchy.task_type`
   - Replace `hierarchy.link_type` with `hierarchy.relationships.story_task`
   - Add `hierarchy.relationships` section for full control
2. Delete existing `jira-mapping.json` files (will be recreated with new structure)

## [1.2.0] - 2026-02-01

### Added

- **Multi-specification support**: Each specification now has its own Jira mapping
- `--spec <name>` argument for `specstoissues` and `sync-status` commands
- **Git branch auto-detection**: Automatically detects spec from branch name (e.g., `feature/005-spec-name`)
- Auto-detection of specification from current directory or single-spec projects
- Helpful error messages listing available specifications when multiple exist

### Changed

- Mapping files now stored in `specs/<spec-name>/jira-mapping.json` instead of `.specify/jira-mapping.json`
- Sync logs now stored in `specs/<spec-name>/jira-sync-log.json` instead of `.specify/jira-sync-log.json`
- Commands read `spec.md` and `tasks.md` from specification directories
- Updated all documentation to reflect spec-aware file locations

## [1.1.0] - 2026-01-29

### Added

- Configurable MCP server name via `mcp_server` setting in jira-config.yml
- Default MCP server changed to "atlassian" to match common setup
- New environment variable `SPECKIT_JIRA_MCP_SERVER` for server name override

### Changed

- Commands now use configurable MCP server instead of hardcoded "jira-mcp-server"
- Updated all documentation to reflect configurable server name
- Removed hardcoded tool requirement from extension.yml

## [1.0.0] - 2026-01-28

### Added

- Initial release of Jira integration extension
- Command: `/speckit.jira.specstoissues` - Create Jira hierarchy from spec and tasks
- Command: `/speckit.jira.discover-fields` - Discover Jira custom fields
- Command: `/speckit.jira.sync-status` - Sync task completion status to Jira
- Configuration system with project, local, and environment variable overrides
- Extension manifest with hooks support
- Comprehensive README with examples and troubleshooting
- MIT License

### Features

- Automatic Epic creation from SPEC.md
- Automatic Task/Subtask creation from TASKS.md
- Hierarchical linking (tasks → epic)
- Custom field discovery and configuration
- Status synchronization with workflow transitions
- Mapping file persistence (.specify/jira-mapping.json)
- Sync activity logging (.specify/jira-sync-log.json)

### Requirements

- Spec Kit: >=0.1.0
- MCP server providing Jira tools (e.g., "atlassian", "jira-mcp-server")

## [Unreleased]

### Planned

- Bi-directional sync (Jira → local tasks)
- Bulk issue updates
- Custom workflow configurations
- Sprint assignment support
- Comment synchronization
- Attachment handling
- Issue filtering and search

---

[2.0.0]: https://github.com/statsperform/spec-kit-jira/releases/tag/v2.0.0
[1.2.0]: https://github.com/statsperform/spec-kit-jira/releases/tag/v1.2.0
[1.1.0]: https://github.com/statsperform/spec-kit-jira/releases/tag/v1.1.0
[1.0.0]: https://github.com/statsperform/spec-kit-jira/releases/tag/v1.0.0
