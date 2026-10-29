---
name: fygo-cli
description: "Use fygo-cli to authorize and manage Fygo devices: files, photos, applications, Docker, downloads, logs, monitoring, networking, storage, and users. Use for Fygo or international-edition devices. Media APIs and mainland-only services are excluded."
---

# Fygo

Manage devices with `fygo-cli`. Installation and upgrades are explicit; check the installed version and command help before use.

<!-- standalone:start -->
## Standalone Skill launcher

Resolve `scripts/fygo-cli` (macOS/Linux) or `scripts/fygo-cli.cmd` (Windows) from the loaded Skill directory and use its absolute path. Replace the `fygo-cli` prefix in Skill examples with that path, preserving each argument and its quoting. Run `--version` and the relevant `--help` first.

First execution automatically downloads one fixed-version CLI for this host; progress goes to stderr and later calls reuse the current user's cache. Standalone execution needs no Node/npm or global CLI. macOS/Linux require system curl, gzip, a SHA-256 tool (sha256sum or shasum), and standard shell utilities. For download failures, damaged caches or mirror configuration, read [first-use guidance](runtime-setup.md), follow its recovery steps and retry the original command. Skill updates are explicit; existing entries never upgrade themselves.

On Windows, call `scripts/fygo-cli.ps1` from PowerShell 5.1 or 7; the `.cmd` entry uses system Windows PowerShell. Quote arguments for the active shell; single quotes are recommended for JSON in PowerShell. Windows downloads and decompresses with system .NET and needs no Unix utilities. If PowerShell is missing or execution policy blocks the script, resolve the local environment issue and retry the same entry.
<!-- standalone:end -->

## Authorization and connections

For login, profiles, connection failures or recovery, read the [connection entry](entries/trim-shared.md) and [OAuth guide](reference/oauth.md). Read [conventions](reference/_conventions.md) for fields and the [index](reference/_index.md) for module guidance.

1. The Agent runs `login --authorize-only --agent-name '<agent>'` (use `codex` for Codex) for the user's target and profile. The CLI builds `{hostname}-{agent}` automatically; share the returned link.
2. The user signs in and authorizes in their browser, then explicitly sends the one-time code back. No user terminal is required.
3. The Agent runs `login --code` on the same execution host, as the same system user, with the same configuration environment, profile and connection options. Confirm token persistence and a read-only request before reporting success.

```bash
fygo-cli --profile home --host <host> --port <port> login --authorize-only --agent-name '<agent>'
fygo-cli --profile home --host <host> --port <port> login --code '<code>'
fygo-cli --profile home system info
```

The user handles passwords, 2FA and browser consent. Accept only the code they explicitly send; do not repeat, additionally store or forward it. Chat, tool records and process arguments may retain the code; do not promise otherwise. Tokens and PKCE verifiers remain in controlled CLI storage, outside chat.

## Usage rules

- Product commands, configuration, profiles and sessions are independent. This release requires fresh authorization; it does not read, migrate or automatically delete old credentials.
- Use only this product's `FYGO_CLI_*` configuration variables. Without explicit connection options, restore only this product's selected profile target; otherwise use `localhost:5666`.
- File writes normally require concrete `/vol{n}/...` paths. Confirm user intent and the command's `--yes` requirement for risky operations; device validation does not authorize writes by default.
- `raw` omits `--yes` only for built-in audited read-only endpoints; an unknown endpoint name does not prove read-only behavior.
- CLI and Skill come from the same package version. After installation, upgrades or public documentation changes, check actual `--version` and relevant `--help`; do not assume `latest` has propagated.

## Command and task overview

- [Connections and authentication](entries/trim-shared.md): Sign in, select connection targets or profiles, troubleshoot sessions and connections, validate devices, and apply common safety rules.
- [Files](entries/trim-file.md): List and search files, upload local files, inspect shared directories and ACLs, or determine current-user and NAS path semantics.
- [Photos](entries/trim-photos.md): Browse photo folders, search photos, inspect details, and obtain thumbnail or original preview links.
- [App Center](entries/trim-app.md): Manage installed applications, App Center installation, FPK packages, updates, application entries and lifecycle operations.
- [Docker](entries/trim-docker.md): Inspect and manage Docker images, containers, Compose projects and long-running lifecycle operations.
- [Downloads](entries/trim-download.md): List, search, create and control download tasks; inspect task files, statistics and state transitions.
- [Logs](entries/trim-log.md): Browse and filter logs, enumerate modules, clear or export logs, configure archives, and copy or cancel diagnostic logs.
- [Monitoring](entries/trim-monitor.md): Inspect runtime CPU, memory, network, disk, process, service, battery, fan, GPU/NPU and alarm-beeper metrics.
- [Networking](entries/trim-network.md): Manage network service switches, including enabling or disabling SSH, and check write confirmation requirements.
- [Storage](entries/trim-storage.md): Inspect storage pools, disks and SMART data, or perform mounting, expansion, disk replacement, formatting and other high-risk storage writes.
- [System and power](entries/trim-system.md): Inspect static system identity, model, versions, volumes, hardware and power schedules, or reboot and power off a device.
- [Users and permissions](entries/trim-user.md): Inspect users, groups, signed-in devices and current permissions, manage users, or choose the correct authenticated profile.
