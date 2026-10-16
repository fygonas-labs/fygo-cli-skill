---
name: fygo-cli
description: Use fygo-cli to authorize and manage Fygo devices: files, photos, applications, Docker, downloads, logs, monitoring, networking, storage, and users. Use for Fygo or international-edition devices. Media APIs and mainland-only services are excluded.
---

# Fygo

## CLI setup

Requires Node.js 18+. Install or explicitly upgrade the CLI:

```bash
npm install -g @fygonas-labs/fygo-cli@latest
fygo-cli --version
```

Without a global installation, replace `fygo-cli` with `npx -y --package=@fygonas-labs/fygo-cli@latest fygo-cli`. Check the relevant `--help` before execution. During npm processing, `latest` may still select the previous release; existing installations never update automatically. Installing a Skill does not install the CLI or sign in. Follow the authorization flow below: the Agent generates a link, the user authorizes in their browser and explicitly returns the one-time code.


Manage devices with `fygo-cli`. Installation and upgrades are explicit; check the installed version and command help before use.


## Authorization and connections

For login, profiles, connection failures or recovery, read the [connection entry](entries/trim-shared.md) and [OAuth guide](reference/oauth.md). Read [conventions](reference/_conventions.md) for fields and the [index](reference/_index.md) for module guidance.

1. The Agent runs `login --authorize-only` for the user's target and profile, then shares the returned link.
2. The user signs in and authorizes in their browser, then explicitly sends the one-time code back. No user terminal is required.
3. The Agent runs `login --code` on the same execution host, as the same system user, with the same configuration environment, profile and connection options. Confirm token persistence and a read-only request before reporting success.

```bash
fygo-cli --profile home --host <host> --port <port> login --authorize-only
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
