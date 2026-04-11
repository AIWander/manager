# Manager — Recommended CLAUDE.md Instructions

Copy this block into your CLAUDE.md to enforce manager delegation discipline.

---

```markdown
## Delegation via Manager MCP Server

### The 33-Line Rule (Hard Cutoff)
If a task requires writing more than ~33 lines of code, delegate it via
`manager:task_submit`. Claude's tokens are for reasoning and orchestration.
Coding agents (Codex, Claude Code, Gemini) have their own sandboxes and
token budgets — let them write code.

- Under 33 lines: write inline
- Over 33 lines: `task_submit(prompt="...", auto_route=true)`
- Multi-file changes: `task_run_parallel` with dependency gates

### Auto-Route as Default
When unsure which backend to use, always use `auto_route=true`.
The router considers task shape, historical success rates, and learned
patterns. Only override when you have specific knowledge:
- Codex: single-file write-and-run
- Claude Code: multi-step toolchain, corrections, complex refactors
- Gemini: one-shot Q&A, large-context analysis
- GPT: pure reasoning, structured output

### Archive-First Rule
Every delegated task that writes files creates backups automatically.
If a task fails:
1. `task_rollback(task_id=...)` to restore pre-task file state
2. `task_retry(task_id=..., additional_context="...")` with error context
Never manually clean up failed delegation output without checking
`task_rollback` first.

### Post-Delegation Extraction
After any delegated task completes, scan the output for:
- **Corrections:** The backend fixed something unexpected
- **Decisions:** The backend made a non-obvious architectural choice
- **Discoveries:** The backend found a bug, pattern, or constraint

If the finding passes the 3Q gate (Reusable? Specific? New?), extract it
to the knowledge base. Delegation without extraction loses half the value.

### Task Health (v1.1.1)
- Read `health` on `task_status`, not `stall_detected`. `health` is an enum:
  `done`, `failed`, `queued`, `cancelled`, `paused`, `running_long_tool`,
  `stalled`, `idle`, `running`. `stall_detected` is legacy.
- When `health` says `running_long_tool`, **keep waiting** — a backend tool
  is mid-flight. Do not cancel. Long Write/Edit operations can take 90+ seconds.
- Use `task_rerun` when a completed task needs another pass with tweaked
  context, injected files, or a different backend. Reuses the original prompt
  instead of writing a new one from scratch.

### Coordination
- 2+ related subtasks → create a Project Loaf first (`create_loaf`)
- Use `task_watch` for blocking waits (zero polling overhead)
- Use `wait=true` on `task_submit` only for short tasks (< 5 min)
```
