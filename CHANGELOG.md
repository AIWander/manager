# Changelog

All notable changes to the Manager MCP Server are documented here.

## [1.2.0] - 2026-04-13

### Fixed

- **CRITICAL: Ghost tasks no longer survive manager restart.** Previously, restarting `manager.exe` (or a second instance starting) unconditionally marked all Running/Queued tasks as Failed with the message `"Server restarted while task was running"` — even when the child process was still alive. Tasks that *actually* completed also got stuck in "Running" forever across restarts. v1.2.0 replaces the blanket clobber with per-task PID tracking and smart liveness verification on startup. Tasks with live child PIDs keep running; dead PIDs are marked Failed with a specific observation; legacy tasks (no PID) are marked Failed with `"legacy task, no child_pid"`.

### Added

- **`child_pid` field on task records.** CLI tasks persist the spawned child process PID to disk. This enables startup recovery to verify task liveness rather than assuming everything is dead.

- **`watchdog_observations` field on task records.** Read-only telemetry array surfacing manager's observations about task state — restarts, PID liveness checks, singleton takeovers. These observations do NOT mutate task status. They only report what was seen.

- **Named-pipe singleton architecture.** Multiple Claude Desktop worker processes no longer cause competing manager instances. First manager acquires an exclusive lock at `~/AppData/Local/manager-mcp/manager.lock` and binds a named pipe server at `\\.\pipe\cpc-manager`. Subsequent spawns proxy stdio through the pipe and exit when stdio closes.

- **Zombie reaper on startup.** Detects stale `manager.exe` instances from previous Claude Desktop sessions and reaps them via named-pipe health check. Prevents accumulation of orphan manager processes.

### Changed

- **Task status transitions are now driven ONLY by child stdio results or explicit timeouts.** Previous behavior of auto-failing tasks based on "I noticed the manager restarted" has been removed entirely. Architectural principle: observation tools don't mutate state.

### Architecture

This release codifies the **observation-action separation**: watchdog observations are strictly read-only telemetry; only the child-exit code path or explicit cancellation can change `task.status`. This pattern should apply to any future observability features added to manager.

---
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
