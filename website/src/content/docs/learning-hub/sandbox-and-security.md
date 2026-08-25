---
title: 'Sandbox and Security in Copilot CLI'
description: 'Understand the OS-level sandbox that restricts file and network access for shell commands in GitHub Copilot CLI sessions, and how to configure it for your workflow.'
authors:
  - GitHub Copilot Learning Hub Team
lastUpdated: 2026-08-25
estimatedReadingTime: '8 minutes'
tags:
  - security
  - sandbox
  - configuration
  - fundamentals
relatedArticles:
  - ./copilot-configuration-basics.md
  - ./automating-with-hooks.md
  - ./using-copilot-coding-agent.md
prerequisites:
  - Basic understanding of GitHub Copilot CLI
---

When GitHub Copilot runs shell commands on your behalf, it uses an **OS-level sandbox** to restrict what those commands can access. The sandbox limits file paths, network connections, and system resources — preventing accidental writes to sensitive locations and limiting the impact if an agent runs an unexpected command.

This article explains how the sandbox works, how to configure it for your workflow, and how enterprise administrators can enforce sandbox policies across their organization.

## What Is the Copilot CLI Sandbox?

The Copilot CLI sandbox is an OS-native isolation layer (using macOS sandbox profiles and Windows job objects on those platforms, with a Linux equivalent) that wraps every shell command the agent executes. Rather than relying solely on the agent's judgment about what commands are safe, the sandbox enforces policy at the OS level.

**Key things the sandbox controls**:
- Which files and directories shell commands can read or write
- Whether outbound network connections are allowed
- Access to toolchain caches and package registries
- Access to git credentials and the `gh` CLI
- Access to macOS Keychain (on macOS)

Commands that violate the sandbox policy are blocked before they execute, and the CLI shows you what was denied and why.

## Viewing and Configuring the Sandbox

### The `/sandbox` Command

The `/sandbox` slash command opens an interactive configuration dialog where you can see and adjust all sandbox settings:

```
/sandbox            # open the sandbox configuration dialog
/sandbox policy     # show effective paths, denials, and network access
```

The `/sandbox policy` subcommand shows a read-only summary of the currently enforced policy — useful for understanding exactly which paths are writable, which are denied, and whether outbound network access is on.

### Enabling and Disabling the Sandbox

The first time you start a session, the CLI may show a first-run prompt offering to enable the sandbox (if it hasn't been configured before). You can also toggle it from `/sandbox`.

> **Note**: Disabling the sandbox from a bypass prompt applies only to that session. New sessions start sandboxed again — the sandbox cannot be globally disabled from a bypass.

### Key Sandbox Settings

These settings can be adjusted in the `/sandbox` dialog or in your `settings.json`:

| Setting | Description | Default |
|---------|-------------|---------|
| `allowDevToolAccess` | Grants sandboxed builds access to toolchain caches, registries, and installs (npm, cargo, pip, etc.) | `true` |
| `sandbox.auth.git` | Allow sandboxed git commands to use stored HTTPS credentials | configurable |
| `sandbox.auth.gh` | Allow sandboxed `gh` CLI commands to authenticate | configurable |
| `sandbox.auth.keychain` *(macOS)* | Allow sandboxed commands to access the system Keychain | `false` by default for tighter isolation |
| `network.allowOutbound` | Whether sandboxed commands can make outbound network connections | configurable |

> **BREAKING CHANGE (v1.0.79)**: The setting `allowDevToolCaches` was renamed to `allowDevToolAccess`. If you have `allowDevToolCaches: false` in your settings to opt out, rename it — the old key is silently ignored and the sandbox will behave as if it's enabled.

> **BREAKING CHANGE (v1.0.79)**: Sandbox authentication settings moved from `sandbox.gitAuth`/`sandbox.ghAuth` to `sandbox.auth.git`/`sandbox.auth.gh`. The old keys are ignored and must be updated in settings files and any managed/MDM policy.

### Sandboxed git Authentication

Sandboxed git can authenticate to non-GitHub remotes (Azure DevOps, GitHub Enterprise Server, GitLab, and other remotes with stored HTTPS credentials). Enable this from the **Auth** tab in the `/sandbox` dialog. Once enabled, git push/pull operations in sandboxed builds work without requiring manual token entry.

## Bypass Prompts

When the sandbox blocks a command, the CLI offers a **bypass prompt** — if bypass is permitted in your session:

- **Linux**: Blocked searches and most shell commands immediately offer to re-run outside the sandbox without asking the model again.
- **macOS**: Blocked commands show a bypass prompt with the denied path and reason.
- **Windows**: Similar bypass prompts appear for blocked commands.

Bypassing only affects the current session. A bypass from one session does not carry over to future sessions.

### Configuring Bypass Behavior

The `/permissions` command controls how the CLI handles tool permission prompts globally, including sandbox bypass prompts. The `/allow-all` command (or `--autopilot` startup flag) disables individual permission prompts — but note that unconditional autopilot approval also disables the sandbox for the current session when bypass is allowed.

## Dev Tool Access

By default, the sandbox grants sandboxed builds access to:
- **Package manager caches** (npm, pip, cargo, Maven, Gradle, etc.)
- **Package registries** (npmjs.com, PyPI, crates.io, etc.)
- **Toolchain installs** (Node.js, Python, Rust, Go, etc.)

This is controlled by the `allowDevToolAccess` setting (on by default). It ensures that commands like `npm install`, `cargo build`, or `go test` work without extra setup. Set `allowDevToolAccess: false` if you want stricter isolation that prevents any registry access.

### Workspace-Aware Build Caches

The sandbox inspects build manifests in your working directory (`package.json`, `Cargo.toml`, `go.mod`, etc.) and grants access to the specific caches and registries those manifests need. Sandboxed wrapper builds (`make` and similar) also get the dev tool caches their recipes require.

On Windows, sandbox scratch caches are created on first run — so `cargo`, `go`, `Gradle`, and `ccache` work even without a pre-warmed cache.

## MCP Servers in the Sandbox

MCP servers are affected by the sandbox too. If a sandboxed MCP server fails to start, the CLI:
- Fails fast (in seconds rather than stalling the session)
- Shows the stderr output from the server so you can diagnose the root cause
- Clearly states that the sandbox was at fault and how to fix or opt out

You can toggle individual MCP servers on or off without restarting the CLI — toggling disables or re-enables the server immediately. This is useful when a specific server is blocked by the sandbox and you want to temporarily disable it.

For more on troubleshooting MCP server startup, see [Understanding MCP Servers](../understanding-mcp-servers/).

## Enterprise Sandbox Policy

GitHub organization administrators can enforce a **managed sandbox floor** — a minimum sandbox policy that applies to all members of the organization. Managed settings can tighten (but never loosen) a user's sandbox configuration.

When a managed policy is active:
- The `/sandbox` dialog shows the org-configured values with **locked fields** so you can see exactly what is enforced.
- Managed filesystem paths appear labeled so you know which paths come from the org policy.
- If a managed sandbox policy arrives mid-session (e.g., a policy update while the CLI is running), the footer shows that commands are being restricted by a managed policy.

### Enforcing via MDM

Enterprise administrators can push the managed sandbox policy via macOS and Windows native MDM (Mobile Device Management) settings. This lets you enforce sandbox floors across a fleet of developer machines without requiring each developer to configure it themselves.

### Policy Enforcement Details

- Managed settings win **per entry** for `enabledPlugins` and `extraKnownMarketplaces` — plugins or marketplaces your organization pins cannot be overridden locally.
- When `forceRemoteSettingsRefresh` is set, the cached policy is never used as a fallback — a fresh policy must be fetched on startup. If the fetch fails, the session applies a restrictive undetermined-policy posture: non-default MCP servers are blocked, bypass permissions mode cannot be enabled, and policy-gated plugin updates are blocked.

### Viewing the Effective Policy

```
/sandbox policy
```

This shows the combined effective policy — your local settings merged with any managed settings — so you always know exactly what is and isn't permitted in the current session. Inactive settings are tagged `(disabled)` with an explanation of why they are locked.

## Security Considerations

The sandbox reduces risk but is not a complete security boundary. Some considerations:

- **The sandbox runs with your user account permissions.** It limits what the agent can do, but commands still run as you. Do not rely on the sandbox as the sole protection against malicious instructions.
- **Bypass disables the sandbox for the session.** If you approve a bypass, the rest of the session runs without sandbox restrictions. Use bypasses selectively.
- **Hook scripts run outside the sandbox by default.** Hooks (`preToolUse`, `postToolUse`, etc.) are trusted automation you control, so they are not sandboxed. For more on hooks, see [Automating with Hooks](../automating-with-hooks/).
- **Remote/cloud coding agent sessions** (on GitHub.com) run in an isolated cloud environment and don't use the local OS sandbox — they have their own cloud-based isolation. The local sandbox described here applies to local CLI sessions only.

## Troubleshooting

**Command fails with a permission error**

1. Run `/sandbox policy` to see the effective allowed and denied paths.
2. If the denied path is a build cache or registry, check whether `allowDevToolAccess` is enabled.
3. If the path is a project directory that should be writable, check if the worktree or project root is covered by the allowed paths.
4. Use a bypass prompt to re-run the command outside the sandbox while you diagnose.

**MCP server fails to start**

- Check the stderr output in the CLI warning — it contains the server's error message.
- Run `/mcp show <server-name>` to see the server status and connection details.
- If the server needs network access or file paths that are blocked, adjust the sandbox settings or toggle the server off.

**Build fails with missing cache/registry access**

- Verify `allowDevToolAccess` is `true` (the default).
- If it was previously set to `false` via the old `allowDevToolCaches` key, rename it to `allowDevToolAccess` in your settings.

## Next Steps

- **Configure permissions and approval modes**: [Copilot Configuration Basics](../copilot-configuration-basics/) — covers `/allow-all`, `/permissions`, and autopilot settings
- **Automate with hooks**: [Automating with Hooks](../automating-with-hooks/) — add guardrails to autonomous sessions alongside the sandbox
- **Understand MCP server troubleshooting**: [Understanding MCP Servers](../understanding-mcp-servers/) — diagnose MCP server connectivity and sandbox issues

---
