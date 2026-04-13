# Manager MCP Server

Multi-vendor AI orchestration from inside any MCP client. Manager routes
coding, reasoning, and toolchain tasks to **Claude Code**, **OpenAI Codex**,
**Google Gemini CLI**, or **OpenAI GPT API** â€” based on task shape, historical
success rates, and explicit user choice.

One MCP server. Four backends. Server-side blocking. Durable coordination.

---

## What's New in v1.2.0

**Critical fix** for ghost tasks after manager restart, plus architectural hardening for multi-instance scenarios.

### 1. Fixed: Ghost tasks no longer survive manager restart (CRITICAL)

Previously, restarting ``manager.exe`` — or a second manager instance starting — unconditionally marked all Running/Queued tasks as Failed with ``"Server restarted while task was running"``. This fired even when the child process was still alive and working. Tasks that actually *completed* also got stuck in "Running" forever across restarts.

v1.2.0 replaces the blanket clobber with per-task PID tracking and smart liveness verification:

- CLI tasks now persist their child process PID to disk
- On startup, manager checks each Running task's PID
- Alive → keep Running with observation ``"child pid N confirmed alive"``
- Dead → mark Failed with observation ``"child pid N dead on manager startup"``
- Legacy tasks (no PID stored) → marked Failed with observation ``"legacy task, no child_pid, cannot verify across restart"``

### 2. Added: Named-pipe singleton architecture

Multiple Claude Desktop worker processes no longer cause competing manager instances. The first manager to start acquires an exclusive lock at ``~/AppData/Local/manager-mcp/manager.lock`` and binds a named pipe server at ``\\.\pipe\cpc-manager``. Subsequent spawns proxy stdio through the pipe and exit when stdio closes. Only one real manager processes tasks at any time; state is shared safely via the pipe.

### 3. Added: Zombie reaper on startup

Detects stale ``manager.exe`` instances left over from previous Claude Desktop sessions and reaps them via named-pipe health check. Prevents accumulation of 5-10 orphan manager processes across long-running sessions.

### 4. Added: ``watchdog_observations`` field on task records

Read-only telemetry surfacing manager's observations about task state (restarts, PID liveness checks, singleton takeovers). These observations do NOT mutate task status — they only report what was seen.

**Architectural principle:** observation tools don't mutate state. Task success/failure is determined ONLY by child process stdio result or explicit timeout — never by "I noticed something was off." The previous auto-failing behavior has been removed entirely.

### 5. Added: ``child_pid`` field on task records

CLI tasks persist the spawned child's PID. Enables startup recovery to verify task liveness rather than assuming all Running tasks are dead.

---

## What's New in v1.1.1

### 1. `task_rerun` now documented

Re-submit a completed task with tweaked context, file injection, or backend
override. The new task links back to the original via `parent_task_id`.
Use this instead of writing a new prompt from scratch when a completed task
needs another pass.

### 2. Stall detector fix

The stall detector threshold has been raised from **30 seconds to 90 seconds**.
Additionally, the detector now **skips entirely** when a backend tool is
mid-flight (`active_tool_running == true`). This eliminates false positives
on long Write/Edit operations that previously triggered stall warnings after
just 30 seconds of no visible step updates.

### 3. `health` enum on `task_status`

New `health` field replaces `stall_detected` as the field to read for
behavior decisions. Nine values:

| Value | Meaning |
|-------|---------|
| `done` | Task completed successfully |
| `failed` | Task failed |
| `queued` | Waiting to be picked up |
| `cancelled` | Cancelled by user or system |
| `paused` | Paused by user |
| `running_long_tool` | Backend tool is mid-flight â€” keep waiting |
| `stalled` | No activity beyond threshold â€” actually stuck |
| `idle` | Session open but no active work |
| `running` | Normal execution in progress |

`stall_detected` remains for backward compatibility but `health` is strictly
more expressive.

### 4. `active_tool_running` on `task_status`

Boolean field â€” `true` when the backend's most recent step has a `"started"`
event with no completion event yet. A tool is mid-flight. This is what the
stall detector reads to decide whether to skip.

### 5. Task lineage scaffolding

Three new fields on task records: `parent_task_id`, `forked_from`, and
`continuation_of`. Only `parent_task_id` is populated today (set automatically
by `task_rerun` via the `rerun_of` relationship). Fork and continuation
handlers are planned for a follow-up release.

---

## Overview

Manager exists because of the **33-line rule**: if a task requires writing
more than ~33 lines of code, delegate it. Claude's context window is for
reasoning and orchestration. Coding agents have their own sandboxes and token
budgets â€” let them write code.

### Backends

| Backend | Status | Best For |
|---------|--------|----------|
| **Claude Code** | Full support | Multi-step toolchains, iterative implementation, complex refactors â€” the primary backend in v1.2.0 |
| **GPT** | Full support | Pure reasoning chains, structured output, classification |
| **Codex** | Compatibility â€” beta | One-shot script generation. Full functionality planned for v2. |
| **Gemini CLI** | Compatibility â€” beta | One-shot Q&A, large-context analysis. Full functionality planned for v2. |

### Key Capabilities

- **Auto-routing** â€” `auto_route=true` picks the best backend per task
- **Server-side blocking** â€” `task_watch` holds the connection, zero polling
- **Project Loafs** â€” durable JSON coordination files that survive crashes
- **Archive-first** â€” file backups before every write, `task_rollback` to restore
- **Analytics** â€” `get_analytics` shows backend success rates over time
- **Task lineage** â€” `task_rerun` links new tasks to originals via `parent_task_id`

---

## Installation

### Prerequisites

- At least one backend CLI installed:
  - **Claude Code**: `claude` CLI
  - **Codex**: `codex` CLI or `OPENAI_API_KEY`
  - **Gemini**: `gemini` CLI or `GEMINI_API_KEY`
  - **GPT**: `OPENAI_API_KEY`

### Claude Desktop Configuration

Add to your `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "manager": {
      "command": "C:\\CPC\\servers\\manager.exe",
      "args": []
    }
  }
}
```

See `claude_desktop_config.example.json` for ARM64 + x64 paths.

### Verify Installation

Run the included doctor script:

```powershell
.\doctor.ps1
```

This checks the binary, backend availability, and task state directory.

---

## Quickstart

### Submit a task

```
task_submit(
  prompt="Write a pytest suite for utils.py covering edge cases",
  auto_route=true,
  wait=true
)
```

`auto_route=true` picks the best backend automatically. To pick manually, pass
`backend="claude_code"` (or `codex`, `gemini`, `gpt`) instead.

### Re-run with tweaks (new in v1.1.1)

```
task_rerun(
  task_id="task_abc123",
  additional_context="Also handle the empty-array edge case",
  include_files=["tests/edge_cases.py"]
)
```

### Check task health (new in v1.1.1)

```
status = task_status(task_id="task_abc123")
# Read status.health â€” not stall_detected
# "running_long_tool" = backend is working, keep waiting
# "stalled" = actually stuck, consider cancelling
```

### Parallel workflow with dependencies

```
task_run_parallel(
  name="auth refactor",
  steps=[
    { id: "tests",    backend: "claude_code", prompt: "Write unit tests for auth.py" },
    { id: "docs",     backend: "claude_code", prompt: "Write docstrings for auth.py" },
    { id: "refactor", backend: "claude_code", prompt: "Refactor auth.py using new tests", depends_on: ["tests"] }
  ]
)
```

### Monitor with zero polling overhead

```
task_watch(task_ids=["task_1", "task_2"], timeout=600)
```

---

## Tool Reference

### Core Task Tools

| Tool | Purpose |
|------|---------|
| `task_submit` | Submit a one-shot task to a backend |
| `task_status` | Check task state, health, and active_tool_running |
| `task_watch` | Server-side block until tasks complete |
| `task_output` | Retrieve full output of a completed task |
| `task_cancel` | Cancel a running or pending task |
| `task_retry` | Re-run a failed task with error context injected |
| `task_rerun` | Re-submit a completed task with modifications |
| `task_rollback` | Restore files from pre-task backup |
| `task_explain` | Human-readable summary of what a task did |
| `task_list` | List recent tasks with optional filtering |
| `task_cleanup` | Remove old task records |
| `task_decompose` | Break a complex prompt into a subtask DAG |
| `task_route` | Preview routing decision without submitting |

### Session Tools

| Tool | Purpose |
|------|---------|
| `session_start` | Start a persistent multi-turn session |
| `session_send` | Send a message to an active session |
| `session_list` | List active sessions |

### Project Loaf Tools

| Tool | Purpose |
|------|---------|
| `create_loaf` | Create a coordination file for related subtasks |
| `loaf_update` | Update loaf state |
| `loaf_status` | Read current loaf state |
| `loaf_close` | Finalize a completed loaf |

### Workflow & Template Tools

| Tool | Purpose |
|------|---------|
| `task_run_parallel` | Execute tasks with dependency gates |
| `workflow_run` | Execute a saved workflow template |
| `template_save` | Save a reusable workflow template |
| `template_list` | List saved templates |
| `template_run` | Run a saved template |

### Analytics & Configuration

| Tool | Purpose |
|------|---------|
| `get_analytics` | Query historical task performance data |
| `configure` | Update manager settings at runtime |
| `role_create` | Define a named backend role |
| `role_delete` | Remove a role |
| `role_list` | List defined roles |

### Extraction Tools

| Tool | Purpose |
|------|---------|
| `review_extractions` | Review delegation output for patterns |
| `dismiss_extraction` | Dismiss a pending extraction |
| `extract_workflow` | Extract a workflow pattern from task history |

---

## Companion Skill: Manager + Local

If you also run the `local` MCP server, install the **manager-with-local**
skill for breadcrumb-tracked delegation chains. This wraps multi-step
manager orchestrations in local's breadcrumb system for crash recovery,
cross-context resumption, and audit trails.

See `skills/manager-with-local.md` for the full reference.

---

## Examples

- [`examples/delegate_a_coding_task.md`](examples/delegate_a_coding_task.md) â€” Single-task delegation walkthrough
- [`examples/task_rerun_workflow.md`](examples/task_rerun_workflow.md) â€” Re-running completed tasks with modifications
- [`examples/parallel_workflow.md`](examples/parallel_workflow.md) â€” DAG execution with dependency gates
- [`examples/health_enum_interpretation.md`](examples/health_enum_interpretation.md) â€” Reading the health enum correctly

---

---

### Prerequisites: log into your coding CLI first

Manager delegates to coding agents by shelling out to their command-line interfaces. **You must install and log into each CLI you want manager to use, before manager can call it.** Manager does not handle authentication â€” it assumes the CLI is already ready.

- **Claude Code** â€” run `claude` in PowerShell or your terminal, complete the login flow, confirm it works standalone. Requires an active Claude subscription; manager's usage counts against that subscription.
- **OpenAI Codex CLI** *(beta support)* â€” install `codex`, log in, verify. Requires an active OpenAI subscription.
- **Gemini CLI** *(beta support)* â€” install `gemini`, log in, verify. Requires an active Google AI subscription.

Each CLI must be authenticated in a real interactive terminal *before* manager's first delegation call. If you skip this step, manager's first `task_submit` will hang or fail with an auth error from the child process. This is the single most common first-run issue â€” check it before anything else.

## Compatible With

Works with any MCP client. Common install channels:

- **Claude Desktop** (the main chat app) â€” add to `claude_desktop_config.json`. See `claude_desktop_config.example.json` in this repo.
- **Claude Code** â€” add to `~/.claude/mcp.json`, or point your `CLAUDE.md` at `skills/manager.md` to load it as a skill instead.
- **OpenAI Codex CLI** â€” register via Codex's MCP config, or load the skill directly.
- **Gemini CLI** â€” register via Gemini's MCP config, or load the skill directly.

**Two install layouts:**

1. **Local folder** â€” clone or download this repo, then point your client at the local directory or the extracted `.exe` binary.
2. **Installed binary** â€” grab the `.exe` from the [Releases](https://github.com/josephwander-arch/manager/releases) page, place it wherever you keep your MCP binaries, then register its path in your client's config.

**Also ships as a skill** â€” if your client supports Anthropic skill files, load `skills/manager.md` directly. Skill-only mode gives you the behavioral guidance without running the server; useful for planning, review, or read-only workflows.

### First-run tip: enable "always-loaded tools"

For the smoothest experience, enable **tools always loaded** in your Claude client settings (Claude Desktop: Settings â†’ Tools, or equivalent in Claude Code / Codex / Gemini). This ensures Claude recognizes the tool surface on first use without needing to re-discover it every session. Most users hit friction on day one because this is off by default.

### Bootstrap the rest of the toolkit *(optional convenience)*

`manager` is not a required install path â€” each of the other four MCP servers can be installed directly using the steps in Compatible With above. But if you already have `manager` running, you can skip the manual work for the rest.

Once `manager` is running, you can delegate the remaining four installs to a fresh Claude Code session. Ask Claude:

> `task_submit with backend claude_code: install hands, local, echo, and workflow from github.com/josephwander-arch/, register them in Claude Desktop config, and verify each one started cleanly.`

The delegated session handles download, placement, and config updates in its own context â€” you monitor via `task_status` and pick up the results when it reports `health: done`. Good for users who already have Claude Code installed and want the full stack without manual steps.

## License

Apache License 2.0. See [LICENSE](LICENSE).

---

## Donations

If this project saves you time, consider supporting development:

**$NeverRemember** (Cash App)

---

## Contact

Joseph Wander
- GitHub: [github.com/josephwander-arch](https://github.com/josephwander-arch/)
- Email: protipsinc@gmail.com
