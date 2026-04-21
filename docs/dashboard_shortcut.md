# Opening the dashboard

Manager's dashboard is served on `localhost` the moment manager starts. The port is chosen dynamically (defaults to `9218`, falls back to the next free port in a 100-port range if that's taken), so the URL changes between manager restarts.

Rather than keeping bookmarks in sync, use one of these approaches:

## Option 1 — Call the MCP tool from your chat

In any Claude session that has manager loaded:

> `manager:dashboard_open`

Returns the URL and opens it in your default browser.

> `manager:dashboard_status`

Returns `{ port, running, url }` without opening a window.

## Option 2 — Use the included batch script

`scripts/open_dashboard.cmd` reads the current URL from `%LOCALAPPDATA%\manager-mcp\dashboard_url.txt` (written by manager v1.4.2+ at every startup) and opens it in your default browser. It has retry logic for the race window where manager is restarting but hasn't written the URL file yet.

### Install the script

Copy `scripts/open_dashboard.cmd` anywhere convenient — `C:\CPC\scripts\` is a common spot. It's self-contained, no dependencies.

### Make a desktop shortcut (optional)

Right-click your desktop → **New → Shortcut**. Paste the full path to the script (e.g. `C:\CPC\scripts\open_dashboard.cmd`), name it `Dashboard`, finish. Double-click it anytime to open the current dashboard URL.

You can change the shortcut's icon in its properties if the default batch-file icon bothers you — `shell32.dll,14` is a common choice.

## Why not ship a `.lnk` file?

Windows shortcuts embed absolute target paths, so a `.lnk` in the repo would only work for users who placed the script at the exact same path. Distributing the script and letting users create their own shortcut is more portable.

## Troubleshooting

If the script reports *"Dashboard URL file not found after 15 attempts"*:

1. Is manager running? (Task Manager → Details → look for `manager.exe`)
2. Is manager listed in `%APPDATA%\Claude\claude_desktop_config.json`?
3. Try calling `manager:dashboard_status` directly — if it errors, manager isn't loaded in your current session.

The URL file lives at `%LOCALAPPDATA%\manager-mcp\dashboard_url.txt`. Manager writes it on every startup in v1.4.2 and later.
