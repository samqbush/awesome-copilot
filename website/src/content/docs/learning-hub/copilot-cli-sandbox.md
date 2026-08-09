---
title: 'Copilot CLI Sandbox'
description: 'Understand the OS-level shell sandbox in GitHub Copilot CLI — how it protects your system, how to configure it, and how enterprise admins can enforce policies.'
authors:
  - GitHub Copilot Learning Hub Team
lastUpdated: 2026-08-09
estimatedReadingTime: '8 minutes'
tags:
  - sandbox
  - security
  - copilot-cli
  - fundamentals
relatedArticles:
  - ./automating-with-hooks.md
  - ./understanding-mcp-servers.md
  - ./copilot-configuration-basics.md
prerequisites:
  - GitHub Copilot CLI installed
  - Basic understanding of Copilot CLI sessions
---

The GitHub Copilot CLI includes an OS-level shell sandbox that restricts what commands agents can run during a session. The sandbox limits filesystem access, network egress, and process spawning so that agents work within a defined boundary — protecting your system even if an agent follows unexpected instructions.

This article explains what the sandbox does, how to configure it, and how enterprise administrators can enforce sandbox policies.

## What the Sandbox Does

When the sandbox is active, Copilot CLI creates an OS-level isolation layer around shell commands and tool executions:

- **Filesystem access**: Agent-run commands can only read and write to the paths you permit
- **Network egress**: Outbound network access is controlled — commands can be blocked from making external requests
- **Dev tool access**: Build tools, package managers, and their caches are allowed by default so builds work without extra setup
- **MCP servers**: Locally-spawned MCP servers that run inside the sandbox are labeled `connected (sandboxed)` in `/mcp list`

The sandbox is enforced at the OS level (using platform-specific mechanisms), not just at the application level — commands that try to access restricted paths fail at the OS boundary.

> **Note**: The sandbox applies to shell commands and tool executions run by the agent. It does not restrict the Copilot CLI process itself, your terminal, or commands you run manually outside of the agent.

## Enabling the Sandbox

Starting with v1.0.74, a first-run splash prompts you to opt into the default sandbox. You can also manage it with the `/sandbox` command inside a session, or via command-line flags.

### Checking Sandbox Status

```
/sandbox
```

Opens the sandbox configuration dialog, showing which sandbox settings are active and where settings are stored.

### Toggling the Sandbox for a Session

Use the `--sandbox` and `--no-sandbox` flags to turn the sandbox on or off for a single session without changing your saved settings (v1.0.70+):

```bash
# Start a session with the sandbox enabled
copilot --sandbox

# Start a session with the sandbox disabled (e.g., for scripts that need full access)
copilot --no-sandbox

# Useful with -p for one-off scripted tasks
copilot -p "run my deployment script" --no-sandbox
```

These flags apply only to the current session. New sessions will use your saved sandbox setting.

> **Per-session only**: If you disable the sandbox via a bypass prompt during a session, that change applies only to that session — new sessions start sandboxed again (v1.0.78+).

## Configuring the Sandbox

Sandbox settings are managed through the `/sandbox` dialog or `settings.json`. Key settings:

### Dev Tool Access

By default, the sandbox grants agent commands access to toolchain caches, registries, and installs so builds work without extra setup (`allowDevToolAccess`, on by default since v1.0.79-1):

```json
{
  "sandbox": {
    "allowDevToolAccess": false
  }
}
```

Set to `false` to opt out and run with tighter isolation — note that this may break builds that rely on cached dependencies.

> **Renamed in v1.0.79-1**: This setting was called `allowDevToolCaches` in v1.0.78. If you set `allowDevToolCaches: false` in your settings, rename it to `allowDevToolAccess: false`.

### Authentication Inside the Sandbox

By default, sandboxed commands don't have access to your git or `gh` CLI credentials, which can block operations like cloning private repos or authenticating API calls. You can opt in to allow these (v1.0.72+):

From the `/sandbox` dialog, enable the **Auth** tab settings:
- **Git auth** (`sandbox.auth.git`): lets sandboxed git commands use your stored credentials
- **gh auth** (`sandbox.auth.gh`): lets sandboxed `gh` commands authenticate with GitHub
- **macOS Keychain** (`sandbox.auth.keychain`, macOS only): access macOS Keychain for credential storage (off by default for tighter isolation)

```json
{
  "sandbox": {
    "auth": {
      "git": true,
      "gh": true
    }
  }
}
```

### Worktree Base Reference

Control whether `/worktree`, `/worktree new`, and `--worktree` start from HEAD or the remote default branch (v1.0.79-8+):

```json
{
  "worktreeBaseRef": "HEAD"
}
```

All three default to HEAD. Previously, `--worktree` started from the remote default branch.

## Bypassing the Sandbox

When a sandboxed command is blocked, the CLI offers an immediate bypass prompt so you can re-run it outside the sandbox without involving the model. On Linux, the CLI also offers to re-run blocked searches and most shell commands outside the sandbox.

If your organization allows sandbox bypass, you can:
- Respond to the bypass prompt inline
- Disable the sandbox for the current session from `/sandbox`

## Enterprise Sandbox Policies

Enterprise administrators can enforce a sandbox floor — a minimum level of sandboxing that users cannot relax below.

### Managed Settings

Admins can deploy sandbox policies via:
- **GitHub organization managed settings**: Configure via organization settings in GitHub
- **macOS MDM** (v1.0.77+): Deploy via your MDM profile
- **Windows MDM** (v1.0.77+): Deploy via Windows registry or MDM policy

The `/sandbox` dialog surfaces org-configured managed values with locked fields and managed filesystem paths, so both admins and users can confirm what is enforced.

### What Admins Can Control

- **Sandbox floor**: Managed settings tighten (but never loosen) the user's sandbox policy
- **Proxy enforcement**: Enforce a proxy URL for sandboxed outbound requests while leaving credentials user-controlled (v1.0.79-8+)
- **Allow-auto-only policy**: Allow `/allow-all auto` (AI-judged approvals) while keeping full `/allow-all` blocked (v1.0.79-8+)

### User Experience with Managed Policies

When a managed policy is active:
- The `/sandbox` dialog shows locked fields with the managed value
- If a configured MCP server is blocked by policy, a warning is shown at startup
- Managed settings fall back to the cached policy if a settings fetch fails (fail-open — no unconfirmed restrictions)

## Sandbox and MCP Servers

When you toggle the sandbox mid-session with `/sandbox`, only **local MCP servers** are restarted — remote MCP servers remain connected without interruption (v1.0.72+).

MCP servers that run inside the sandbox are labeled in `/mcp list` as `connected (sandboxed)` so you can see their isolation status at a glance (v1.0.70+).

## Common Questions

**Q: What happens to a build when the sandbox is enabled?**

A: Most builds work fine because `allowDevToolAccess` is on by default, giving sandboxed commands access to package manager caches and registries. If a specific tool fails, check the stderr output for permission errors and consider enabling relevant auth settings or adjusting the denied path list.

**Q: Can I use the sandbox with `-p` (prompt mode)?**

A: Yes. Use `--sandbox` or `--no-sandbox` flags with `-p` to control sandboxing for scripted runs. The `--no-sandbox` flag is useful when you're running a deployment script that needs broader access.

**Q: How does the sandbox interact with hooks?**

A: Hook scripts run as commands, so they are subject to sandbox restrictions. If a hook needs broader filesystem or network access, you may need to adjust sandbox settings or have the hook script bypass specific restrictions. See [Automating with Hooks](../automating-with-hooks/) for hook configuration.

**Q: Will the sandbox block my git operations?**

A: By default, sandboxed commands do not have access to git credentials. Enable `sandbox.auth.git: true` to allow sandboxed git commands to authenticate using your stored credentials.

**Q: What if a managed policy breaks my workflow?**

A: Contact your organization administrator to adjust the policy, or ask about approved alternatives (like specific allowed paths or approved proxy configurations).

## Next Steps

- **Hooks and Security**: [Automating with Hooks](../automating-with-hooks/) — Use `preToolUse` hooks for additional security gating on top of the sandbox
- **MCP Servers**: [Understanding MCP Servers](../understanding-mcp-servers/) — Understand how MCP server isolation interacts with the sandbox
- **Configuration**: [Copilot Configuration Basics](../copilot-configuration-basics/) — Explore other repository and user-level configuration options

---
