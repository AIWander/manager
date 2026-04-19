# Manager — Per-Machine Setup

This guide covers everything you need to do on each machine where you want to run the `manager` MCP server.

## Per-Machine Checklist

| Item | Per-machine? | How to set up |
|---|---|---|
| MCP binary | Yes | Download from GitHub release → `C:\CPC\servers\manager.exe`. Pick right arch (`_arm64.exe` or `_x64.exe`). |
| Claude Desktop config | Yes | Edit `%APPDATA%\Claude\claude_desktop_config.json` — copy entry from `claude_desktop_config.example.json` in this repo into your `mcpServers` object. |
| Per-machine paths | Yes | Will be auto-detected by `cpc-paths` (forthcoming). For now, set env vars or hardcode in your config. See "Path Configuration" below. |
| User preferences | Yes | Open Claude Desktop → Settings → Profile → paste your preferences. (UI-only, can't script.) |
| Skills (optional) | Yes | If using CPC skills system, mirror from your Drive's `Volumes/skills/{skill}/` to `%LOCALAPPDATA%\claude-skills\{skill}\`. |
| Volumes / knowledge base | No (Drive-synced) | If Volumes is on Google Drive, just verify Drive is syncing on each machine. No copy needed. |

## Manager-Specific Notes

- **Codex backend:** Requires an OpenAI API key. Set via env var `OPENAI_API_KEY` in your system environment or Claude Desktop config, OR configure at runtime via `manager:configure(openai_api_key="...")`. Must be set before `task_submit` with `backend="codex"`.
- **Gemini CLI backend:** Requires Gemini CLI installed and authenticated. Check the expected path via `manager:configure` output — look for `gemini_cmd`. You must log into `gemini` in an interactive terminal before manager can use it.
- **Claude Code backend:** Requires `claude.exe` (Claude Code CLI) installed and in PATH, or at the path shown by `manager:configure` (`claude_code_cmd`). Log in interactively first — manager assumes the CLI is already authenticated.
- **All backends — auth first:** Manager shells out to backend CLIs. Every CLI must be logged in via an interactive terminal before manager's first delegation call. If you skip this, the first `task_submit` will hang or fail with an auth error. This is the single most common first-run issue.
- **Sessions across restarts:** Sessions persist across Claude Desktop restarts (since v1.2.3). State is stored in `C:\CPC\state\manager\` — per-machine, not synced.
- **Toast notifications (v1.2.6+):** `notify_on_complete` / `notify_on_fail` / `notify_on_destroy` hooks on `session_start` send Windows toast notifications. Uses Windows Runtime APIs — works out of the box on Windows 10/11.

**Test post-install:** `manager:status_bar` should return a clean state summary with no errors.

## Path Configuration

**Coming in `cpc-paths` (next release):** automatic detection of Volumes path, install path, backups path. Auto-writes `.cpc-config.toml` on first run. Until then, paths are detected via env vars with fallbacks:

| Path | Env var | Default fallback |
|---|---|---|
| Volumes (knowledge base) | `CPC_VOLUMES_PATH` | `C:\My Drive\Volumes` (Windows) |
| Install (server binaries) | `CPC_INSTALL_PATH` | `C:\CPC\servers` (Windows) |
| Backups | `CPC_BACKUPS_PATH` | `%LOCALAPPDATA%\CPC\backups` (Windows) |

If you're on a different platform or your Drive is mounted elsewhere, set the env vars in your shell profile or system environment before launching Claude Desktop.

## Future: cpc-setup.exe (planned)

A single-binary helper that automates this entire per-machine setup is planned. It will:
- Detect platform + architecture
- Download the right MCP server binary from GitHub releases
- Auto-detect Volumes / install / backup paths and write `.cpc-config.toml`
- Mirror skills from your Drive (if using CPC skills system)
- Generate a `claude_desktop_config.json` snippet ready to paste

Until cpc-setup.exe ships, follow the manual steps above.

## Two-Tier Storage

Manager writes data to exactly two places. Knowing which is which prevents accidental Drive-sync corruption and makes multi-machine setups straightforward.

**TL;DR: cross-machine data goes on Volumes (Drive-syncable); machine-bound data goes in local storage (never sync).**

### Tier 1 — Volumes (cross-machine, Drive-syncable)

Resolved by `cpc_paths::volumes_path()`. Default: `C:\My Drive\Volumes\` on Windows.
Override: `CPC_VOLUMES_PATH` env var.

| What | Why it lives here |
|---|---|
| Knowledge base (Operating files, CATALOG, skills) | Write-once or write-rarely; no file locks; meaningful across machines |
| Breadcrumb archive (`breadcrumbs/completed/`) | Completed breadcrumbs are write-once; safe to sync |
| Handoffs, transcripts, shared reference patterns | Pure data; read from any machine |

### Tier 2 — Local data (per-machine, never sync)

Resolved by `cpc_paths::data_path("manager")`. Default: `%LOCALAPPDATA%\CPC\data\manager-data\` on Windows.
Override: `CPC_MANAGER_DATA_DIR` env var.

| What | Why it lives here |
|---|---|
| Session `meta.json` (PIDs, alive flags, heartbeats) | PIDs are machine-specific; would mislead on other machines |
| Active breadcrumb state (`active.index.json`) | File-locked during writes; concurrent cloud sync corrupts |
| Task store (`.json` per task) | References local process state |
| Logs | High churn; machine context only |

### What NOT to sync

- **Session directories** (`meta.json`, heartbeat files) — reference local PIDs; dead on any other machine.
- **Active breadcrumb state** (`active.index.json`, `projects/*.jsonl`) — flock-sensitive; cloud sync will corrupt if two machines write simultaneously.
- **Chrome debug profiles** (hands server) — Chrome holds a file lock on the profile directory while running; syncing a live profile will corrupt it.
- **Build artifacts** (`target/`) — architecture-specific binaries; never sync.

### Legacy paths

If you already have data at `C:\CPC\` (legacy install), it keeps working. Manager's legacy-fallback logic checks `C:\temp\manager-sessions\` first; if it contains session data it stays there. New installs use the cpc-paths default (`%LOCALAPPDATA%\CPC\data\manager-data\`). No migration needed.

### Setting up a second machine

1. Verify Google Drive is syncing `Volumes/` on the new machine.
2. Install `manager.exe` (right arch) to `C:\CPC\servers\` or your preferred install path.
3. Add the manager entry to `claude_desktop_config.json` (see checklist above).
4. Log into each backend CLI interactively (Codex, Gemini, Claude Code).
5. Re-enter credentials per machine — credentials are OS-keyring-bound and do not sync.
6. Active session state starts fresh on the new machine. Previous sessions from your other machine are not visible here (expected — they reference that machine's processes).
