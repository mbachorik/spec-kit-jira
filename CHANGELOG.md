# Changelog

All notable changes to the Jira Integration Extension will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
- jira-mcp-server: >=1.0.0

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

[1.0.0]: https://github.com/statsperform/spec-kit-jira/releases/tag/v1.0.0
