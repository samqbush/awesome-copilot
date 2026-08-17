---
title: 'Multi-Session Development and Agent Host Protocol'
description: 'Learn how to manage multiple concurrent sessions and share them across terminals using the Agent Host Protocol (AHP) in GitHub Copilot CLI.'
authors:
  - GitHub Copilot Learning Hub Team
lastUpdated: 2026-08-17
estimatedReadingTime: '10 minutes'
tags:
  - sessions
  - ahp
  - collaboration
  - advanced
relatedArticles:
  - ./github-copilot-app.md
  - ./using-copilot-coding-agent.md
  - ./copilot-configuration-basics.md
prerequisites:
  - GitHub Copilot CLI installed (v1.0.79+)
  - Basic familiarity with Copilot CLI sessions
---

GitHub Copilot CLI v1.0.79 introduced the **Agent Host Protocol (AHP)** — a way to run sessions on a shared host so that multiple terminals, devices, or team members can attach to and collaborate on the same session in real time. This article explains what AHP is, how to use the Sessions tab, and how to manage multiple sessions effectively.

> **Note**: AHP (`--ahp` and `/ahp` commands) is currently gated on the `AHP_CLIENT` feature flag and available to staff users first. The Sessions tab and multi-session management are available to all users.

## Managing Multiple Sessions

The **Sessions tab** lets you work on multiple separate tasks simultaneously from within one CLI window. Each session is independent — its own context, history, and working state.

### Switching Between Sessions

Use the Sessions tab (accessible from the tab bar in the CLI) to:

- View all your open sessions
- Switch between sessions with the arrow keys and `Enter`
- Create a new session with `n`
- Close a session by selecting it and pressing the close key

Sessions that are actively running (agent is working) sort to the top and show their current status. Idle sessions show whether they are waiting for input.

### Worktree Sessions

Use `/worktree new` to start a new session in a freshly created git worktree — ideal for working on a separate branch without disrupting your current session:

```
/worktree new
```

By default, the new worktree starts from `HEAD`. You can change this with the `worktreeBaseRef` setting in your config to always start from the remote default branch instead.

## What Is the Agent Host Protocol?

The **Agent Host Protocol (AHP)** separates the *host* (where sessions live and run) from the *client* (the terminal you type in). Normally both are the same CLI process. With AHP, you can:

- Connect multiple terminals to the **same running session** — useful when pairing or monitoring from another machine
- Run a host daemon on one machine and attach from another (Codespace, remote server, Mission Control cloud environment)
- Let sessions survive terminal disconnects — the session keeps running on the host even if the client goes away

```
Terminal A ──┐
             ├──► AHP Host (sessions live here) ──► Copilot Agent
Terminal B ──┘
```

### Starting an AHP Session

Launch the CLI in AHP mode to automatically attach to (or start) a local host daemon:

```bash
copilot --ahp
```

If a local AHP host is already running, the CLI attaches to it. If not, it starts one in the current directory.

### The `/ahp` Commands

Inside an AHP-connected session, use the `/ahp` family of commands to manage hosts and sessions:

| Command | Description |
|---------|-------------|
| `/ahp status` | Show the connected host's identity, version, and health |
| `/ahp start [port]` | Start a new local AHP daemon serving the current directory |
| `/ahp stop <host>` | Stop a running AHP daemon |
| `/ahp restart <host>` | Restart a daemon on the same workspace |
| `/ahp connect <url>` | Connect to an additional host by URL |
| `/ahp hosts` | List all connected hosts |
| `/ahp use <host>` | Switch the active host |
| `/ahp sessions` | List sessions on the active host |
| `/ahp attach <session>` | Join a session running on the host |
| `/ahp new` | Create a new session on the host |
| `/ahp codespace <name>` | Forward a Codespace's daemon port and add it to the Sessions tab |
| `/ahp cloud <environment-id>` | Put a Mission Control cloud environment in the Sessions tab |

### Connecting to a Codespace

To work in a Codespace session from your local terminal, use `/ahp codespace`:

```
/ahp codespace my-codespace-name
```

This forwards the Codespace's `copilotd` port to your machine and adds it to the Sessions tab's source picker. Select it with `h` to switch your Sessions tab to that host.

> **Prerequisite**: Your `gh` CLI must have the `codespace` scope. If it's missing, the command shows you the exact `gh auth refresh` line to run.

### Connecting to a Mission Control Cloud Environment

Similarly, `/ahp cloud <environment-id>` lets you reach the compute environment a `--cloud` session was provisioned on:

```
/ahp cloud env-abc123
```

The environment appears in the Sessions tab marked `CLOUD`. Mission Control wakes the environment on connect — the CLI cannot start or stop it directly.

### Sharing Sessions

When another CLI attaches to a session you're in, the Sessions tab shows `2 clients` (or more) on that row. Both clients see the same live turn stream:

- **Typing while a turn is streaming**: `Enter` steers the prompt into the running turn; `Ctrl+Q` queues it for the next one; `Ctrl+C` takes it back
- **Presence** is announced on attach and refreshed on a heartbeat, so a joining client shows up immediately and one that leaves stops being counted

The `/ahp status` command reports the number of attached clients as well.

### Session Persistence

AHP sessions live on the host, not in your terminal. This means:

- **Closing your terminal** does not end the session — it keeps running on the host
- **Reconnecting** from the same machine: `--ahp` auto-discovers running daemons and lists them in the Sessions tab
- **Auto-discovery**: `--ahp` scans for AHP daemons already running on your machine and puts them in the source picker automatically. Disable with `COPILOT_AHP_DISCOVER=0`

### Multiple Hosts

`--ahp` and `COPILOT_AHP_URL` accept a comma-separated list of host URLs. You can also add hosts live with `/ahp connect`. The Sessions tab shows each host's sessions separately, and `h` switches the active source. The current source is highlighted, and each host row shows health status (responding / not responding).

Where the Sessions tab cannot display all sources (e.g., a narrow sidebar), it collapses to the current source and shows a count of hidden sources.

### Security

Connection tokens in AHP URLs (e.g., `wss://host:8765?tkn=…`) are automatically redacted from:
- `/ahp status` output
- Session lists
- Connection error messages
- Host-status notices

This means you can safely share transcripts or screenshots without leaking credentials.

## How Sessions Appear in the Tab Bar

Each session entry shows:
- **Status**: running (agent working), waiting (waiting for input), or idle
- **Host**: which daemon the session lives on, if using AHP (marked with `AHP`)
- **Client count**: `2 clients` or more when multiple terminals are attached
- **Health indicator**: a `CLOUD` or `CS` badge for cloud/Codespace-hosted sessions

Busy sessions sort to the top of each host's list. A session you've joined from a remote host is listed once — not duplicated as a local copy.

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| "no AHP host is listening" | No daemon running and auto-start failed | Run `/ahp start` manually |
| Status stuck on "Loading: still waiting on extensions" | Session lives in another process | Normal for `--ahp` sessions — the CLI reports this instead of waiting forever |
| Session runs at workspace root instead of current directory | Daemon serving a parent directory | Expected; the session uses the directory you started the CLI from, if within the host's workspace |
| Host skills vanish shortly after session opens | Stale snapshot from relay reconnect | Reconnect; this is fixed in v1.0.80 |
| Permission denied when creating new session | Host workspace mismatch | Run `/ahp start` in the correct directory to start a new daemon |

## Best Practices

- **One daemon per project**: Start an AHP daemon in your project root so all sessions share the same workspace context
- **Use `/ahp status` after connecting**: Confirm you're on the right host and workspace before starting work
- **Stop unused daemons**: Run `/ahp stop <host>` when done to free resources
- **Don't share connection tokens**: AHP URLs with `tkn=` parameters are credentials — use `/ahp codespace` and `/ahp cloud` instead of raw URLs where possible

## Further Reading

- [GitHub Copilot CLI Releases](https://github.com/github/copilot-cli/releases) — full AHP release notes in v1.0.79 and v1.0.80
- [Using the Copilot Coding Agent](../using-copilot-coding-agent/) — for remote-controlling cloud agent sessions
- [Getting Started with the GitHub Copilot app](../github-copilot-app/) — the desktop app alternative for managing parallel sessions
