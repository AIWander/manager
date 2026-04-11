# Changelog

All notable changes to the Manager MCP Server are documented here.

## [1.1.1] - 2026-04-11

### Added

- **`task_rerun` tool documented.** Re-submit a completed task with optional
  `additional_context`, `include_files` (array of file paths injected into
  the backend prompt), and `backend_override`. The new task record contains
  a `rerun_of` field pointing to the original, and the original's
  `parent_task_id` is set automatically on the new task. Use this instead
  of writing a new prompt from scratch when a completed task needs another
  pass with tweaks.

- **`health` enum on `task_status`.** New string field with 9 values:
  `done`, `failed`, `queued`, `cancelled`, `paused`, `running_long_tool`,
  `stalled`, `idle`, `running`. This replaces `stall_detected` as the
  field to read for behavior decisions. `stall_detected` remains for
  backward compatibility but `health` distinguishes `running_long_tool`
  (safe to wait) from `stalled` (actually stuck).

- **`active_tool_running` bool on `task_status`.** `true` when the
  backend's most recent step has a `"started"` event with no completion
  event yet. The stall detector reads this field to decide whether to skip.

- **Task lineage fields.** Three new fields on task records:
  `parent_task_id` (populated by `task_rerun`), `forked_from`, and
  `continuation_of`. Only `parent_task_id` is populated in this release.
  Fork and continuation handlers are planned for a follow-up release.

### Fixed

- **Stall detector false positives.** Threshold raised from 30 seconds to
  90 seconds. Detector now skips entirely when a tool is mid-flight
  (`active_tool_running == true`). A Write operation on a 12KB markdown
  file once took 99 seconds between visible step updates and was
  incorrectly flagged as stalled — this no longer happens.

## [1.1.0] - 2026-03-28

### Added

- Initial multi-backend orchestration (Claude Code, Codex, Gemini, GPT)
- Auto-route backend selection
- Project Loafs for durable coordination
- `task_run_parallel` with dependency gates
- Server-side `task_watch` blocking
- Archive-first file backups and `task_rollback`
- `get_analytics` for backend performance tracking
- Session tools for multi-turn interactions
- Workflow templates

## [1.0.0] - 2026-03-01

### Added

- Initial release with Claude Code backend support
- Basic task submission, status, and output retrieval
- Task cancellation and cleanup
