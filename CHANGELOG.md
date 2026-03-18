# Changelog

All notable changes to the Jira Integration Extension will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [3.0.0] - 2026-03-06

### Changed

- **Breaking**: Adopted semantic `*_artifact` naming convention (consistent with spec-kit-linear)
  - `hierarchy` section renamed to `mapping`
  - `hierarchy.epic_type` → `mapping.spec_artifact`
  - `hierarchy.story_type` → `mapping.phase_artifact`
  - `hierarchy.task_type` → `mapping.task_artifact`
  - `hierarchy.relationships.epic_story` → `mapping.relationships.spec_phase`
  - `hierarchy.relationships.story_task` → `mapping.relationships.phase_task`
  - `hierarchy.relationships.epic_task` → `mapping.relationships.spec_task`
- **Breaking**: Default labels section renamed
  - `defaults.epic` → `defaults.spec`
  - `defaults.story` → `defaults.phase`
- **Breaking**: Environment variables renamed
  - `SPECKIT_JIRA_EPIC_TYPE` → `SPECKIT_JIRA_SPEC_ARTIFACT`
  - `SPECKIT_JIRA_STORY_TYPE` → `SPECKIT_JIRA_PHASE_ARTIFACT`
  - `SPECKIT_JIRA_TASK_TYPE` → `SPECKIT_JIRA_TASK_ARTIFACT`
  - `SPECKIT_JIRA_EPIC_STORY_RELATIONSHIP` → `SPECKIT_JIRA_SPEC_PHASE_RELATIONSHIP`
  - `SPECKIT_JIRA_STORY_TASK_RELATIONSHIP` → `SPECKIT_JIRA_PHASE_TASK_RELATIONSHIP`
  - `SPECKIT_JIRA_EPIC_TASK_RELATIONSHIP` → `SPECKIT_JIRA_SPEC_TASK_RELATIONSHIP`

### Backward Compatibility

Old v2.x configs are automatically supported:

- `hierarchy.epic_type` → maps to `mapping.spec_artifact`
- `hierarchy.story_type` → maps to `mapping.phase_artifact`
- `hierarchy.task_type` → maps to `mapping.task_artifact`
- `hierarchy.relationships.*` → maps to `mapping.relationships.*`
- `defaults.epic` → maps to `defaults.spec`
- `defaults.story` → maps to `defaults.phase`

### Migration

To use new config format:

1. Rename `hierarchy:` to `mapping:`
2. Rename `epic_type:` to `spec_artifact:`
3. Rename `story_type:` to `phase_artifact:`
4. Rename `task_type:` to `task_artifact:`
5. Update relationship keys: `epic_story` → `spec_phase`, `story_task` → `phase_task`, `epic_task` → `spec_task`
6. Rename `defaults.epic` → `defaults.spec`, `defaults.story` → `defaults.phase`

## [2.1.0] - 2026-02-03

### Added

- **2-level mode support**: Set `task_type: ""` or `"none"` for Epic → Stories only hierarchy
  - Tasks are embedded as checklists in Story descriptions
  - No individual Task issues created in Jira
  - Useful for simpler projects or when tasks don't need individual tracking
- Updated jira-mapping.json structure with `"mode": "2-level"` or `"mode": "3-level"` indicator
- `embedded_tasks` array in 2-level mode mapping (task metadata without Jira keys)

### Changed

- Story description now includes full task checklist when in 2-level mode
- Summary output indicates mode used (2-level vs 3-level)
- Config template includes 2-level mode documentation

## [2.0.0] - 2026-02-01

### Added

- **3-level hierarchy support**: Now creates Epic → Stories → Tasks
  - Epic: Created from SPEC.md (overall specification)
  - Stories: Created from Phase headers in TASKS.md (`## Phase X: ...`)
  - Tasks: Created from task items under each Phase (`- [ ] TXXX ...`)
- **Configurable relationships** for Team-managed and Company-managed Jira projects:
  - `hierarchy.relationships.epic_story`: How Story links to Epic (default: "Epic Link")
  - `hierarchy.relationships.story_task`: How Task links to Story (default: "Relates")
  - `hierarchy.relationships.epic_task`: Direct Task-Epic link (default: "Epic Link")
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

[3.0.0]: https://github.com/mbachorik/spec-kit-jira/releases/tag/v3.0.0
[2.1.0]: https://github.com/mbachorik/spec-kit-jira/releases/tag/v2.1.0
[2.0.0]: https://github.com/mbachorik/spec-kit-jira/releases/tag/v2.0.0
[1.2.0]: https://github.com/mbachorik/spec-kit-jira/releases/tag/v1.2.0
[1.1.0]: https://github.com/mbachorik/spec-kit-jira/releases/tag/v1.1.0
[1.0.0]: https://github.com/mbachorik/spec-kit-jira/releases/tag/v1.0.0
