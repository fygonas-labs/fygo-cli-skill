---
name: trim-monitor
description: Inspect runtime CPU, memory, network, disk, process, service, battery, fan, GPU/NPU and alarm-beeper metrics.
---

# Monitoring

Read `../reference/resmon.md`. Runtime metrics use `monitor`; static system
identity, model and configuration use `trim-system.md`.

## Rules

- `monitor cpu` / `monitor memory` return time-series samples. Explain the returned
  `usage`, `cores` and related fields in context.
- `monitor gen --item` accepts comma-separated or repeated values, limited to
  `storeSpeed`, `netSpeed`, `cpuBusy` and `memPercent`.
- `monitor proc-info --pids` accepts positive integer PIDs, separated by commas.
- Most commands are read-only; `monitor mute-beeper` and `monitor request`
  require confirmation or explicit `--yes`.
- `monitor request` allows only `appcgi.resmon.*`; omit `req` and `reqid` from JSON.

## Commands

```bash
fygo-cli monitor cpu
fygo-cli monitor memory
fygo-cli monitor net
fygo-cli monitor disk
fygo-cli monitor gen --item storeSpeed,netSpeed --item cpuBusy
fygo-cli monitor proc-list
fygo-cli monitor proc-info --pids 123,456
fygo-cli monitor proc-srv
fygo-cli monitor sys-warn
fygo-cli monitor battery
fygo-cli monitor sys-fan
fygo-cli monitor beep-supported
fygo-cli monitor beep-events
fygo-cli monitor beep-reasons
fygo-cli monitor mute-beeper --yes
fygo-cli monitor request appcgi.resmon.gen --json '{"item":["cpuBusy"]}' --yes
```
