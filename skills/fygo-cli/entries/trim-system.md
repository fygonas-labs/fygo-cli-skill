---
name: trim-system
description: Inspect static system identity, model, versions, volumes, hardware and power schedules, or reboot and power off a device.
---

# System and power

Read `../reference/sysinfo.md` and `../reference/power.md`. Query the matching
`system` commands individually for hardware, volumes and power plans.

## Rules

- Static identity/configuration belongs here; CPU and memory runtime metrics use `trim-monitor.md`.
- Named `system` commands are read-only; generic `system request` requires
  confirmation or explicit `--yes` by default.
- `power reboot` / `power poweroff` interrupt services. Confirm the target and
  user authorization before passing `--yes`.
- `system request` allows only `appcgi.sysinfo.*`; omit `req` and `reqid` from JSON.
- Generic requests authenticate with the saved session; sign in to the target first.
- Machine type and version fields vary with CLI/firmware. Treat current
  `system info` output as authoritative; report unexpected or empty fields as
  possible version differences rather than inventing values.

## Commands

```bash
fygo-cli system info
fygo-cli system trim-version
fygo-cli system kernel-version
fygo-cli system host-name
fygo-cli system machine-id
fygo-cli system uptime
fygo-cli system all-vols
fygo-cli system vols
fygo-cli system docker
fygo-cli system unix-time
fygo-cli system boot-on-power
fygo-cli system hardware
fygo-cli system reserved-partition
fygo-cli system is-trim-machine
fygo-cli system trim-feature
fygo-cli system fan-mode
fygo-cli system trim-disk
fygo-cli system time-setting --lang en-US
fygo-cli system timezone-list --lang en-US
fygo-cli system power-plan
fygo-cli system next-power-off-time
fygo-cli system power-plan-status
fygo-cli system power-on-seconds
fygo-cli system inited-timestamp
fygo-cli system request appcgi.sysinfo.getTrimVersion --yes
```

## Power operations

```bash
fygo-cli power reboot --yes
fygo-cli power poweroff --yes
```
