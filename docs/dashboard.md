# CPC Dashboard

Manager includes a built-in HTTP dashboard for monitoring task state, breadcrumbs, session health, and service status across the CPC stack.

## Opening the dashboard

Open a browser to `http://localhost:9100` while manager is running.

Or use the MCP tool:

```json
{ "tool": "dashboard_open", "arguments": {} }
```

## Default ports

| Server     | Default port | Env override                  |
|------------|--------------|-------------------------------|
| manager    | 9100         | `CPC_DASHBOARD_PORT` or `CPC_MANAGER_PORT` |
| local      | 9101         | `CPC_DASHBOARD_PORT_LOCAL`    |
| hands      | 9102         | `CPC_DASHBOARD_PORT_HANDS`    |
| workflow   | 9103         | `CPC_DASHBOARD_PORT_WORKFLOW` |
| autonomous | 9104         | *(no HTTP endpoint; polled by JS)* |

Each server binds `127.0.0.1` only. If the primary port is taken, it tries the next 5 consecutively. Port collisions between servers are detected client-side via the `server` discriminator field in each `/api/status` response.

## What each panel shows

| Panel | Source | Data |
|-------|--------|------|
| Sessions & Tasks | `manager:9100/api/status` | Running/queued/done/failed tasks, backend, elapsed time |
| Breadcrumbs | `local:9101/api/status` | Active breadcrumbs with progress bar, staleness, owner |
| Scorecard | manager + local | Running count, done today, failed, orphaned, extractions, .exe.old count |
| Hands | `hands:9102/api/status` | Browser status, current URL, tab count, last action |
| Workflow | `workflow:9103/api/status` | Credential count, TOTP entries, flows, active watches |
| Autonomous | `autonomous:9104/api/status` | Extractions today, reminders pending, index status |

## Partial install behavior

The dashboard auto-collapses any server that fails 2 consecutive polls **and has never been seen**. After 10 seconds with only manager + local running, the Hands/Workflow/Autonomous panels disappear and a footer shows:

> `3 additional servers not detected (hands, workflow, autonomous) · Show all`

Clicking **Show all** forces all panels visible. If a collapsed server comes online later, its panel re-appears automatically.

The health strip hides dots for collapsed servers. Only servers that have responded at least once get a dot.

## live_status.json piggyback

Every 30 seconds, manager polls all 5 server endpoints and writes a snapshot to:

```
C:\My Drive\Volumes\dashboard\live_status.json
```

This file is used by mobile status cards and external monitors that cannot reach `localhost` directly. It contains the same data as the browser dashboard, serialized as JSON with a `timestamp` field.

## Action bar

| Action | What it does |
|--------|-------------|
| Clean .exe.old | POSTs to `local:9101/api/action/clean_old` — deletes `*.exe.old` from the server install directory |
| GitHub ▾ | Opens the GitHub release page for any of the 5 servers |
| Path copyable | Copies `C:\My Drive\Volumes` to clipboard |

## Views

- **Full** — all panels visible (default)
- **Ops** — hides service panels, shows task/breadcrumb/scorecard only
- **Summary** — scorecard only

## Poll intervals

Manager and local poll every **5 seconds**. Hands/workflow/autonomous poll every **42 seconds** (lower priority, heavier payloads).
