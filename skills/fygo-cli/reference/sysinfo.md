# System information reference

## Overview
System information covers machine identity, version/hardware details, host/time settings, fan configuration and scheduled power plans.

## Conventions
- Use the `appcgi.sysinfo.*` namespace.
- Keep read-only information endpoints separate from configuration mutations.

## Endpoint index
- Implemented:
  - `appcgi.sysinfo.getTrimMachineType`
  - `appcgi.sysinfo.getTrimVersion`
  - `appcgi.sysinfo.getKernelVersion`
  - `appcgi.sysinfo.getHostName`
  - `appcgi.sysinfo.getMachineId`
  - `appcgi.sysinfo.getUptime`
  - `appcgi.sysinfo.getAllVolsInfo`
  - `appcgi.sysinfo.getVolsList`
  - `appcgi.sysinfo.getDockerInfo`
  - `appcgi.sysinfo.getUnixTime`
  - `appcgi.sysinfo.getBootOnPowerFlag`
  - `appcgi.sysinfo.getHardwareInfo`
  - `appcgi.sysinfo.getReservedPartition`
  - `appcgi.sysinfo.isTrimMachine`
  - `appcgi.sysinfo.getTrimMachineFeature`
  - `appcgi.sysinfo.getFanMode`
  - `appcgi.sysinfo.getTrimDisk`
  - `appcgi.sysinfo.getTimeSetting`
  - `appcgi.sysinfo.getTimezoneList`
  - `appcgi.sysinfo.listPowerPlan`
  - `appcgi.sysinfo.nextPowerOffTime`
  - `appcgi.sysinfo.getPowerPlanStatus`
  - `appcgi.sysinfo.getPowerOnSeconds`
  - `appcgi.sysinfo.getSystemInitedTimestamp`
  - `appcgi.sysinfo.*` generic requests (via `system request`)
- Not exposed as dedicated commands:
  - Host/time configuration writes: `setHostName`, `setTimeSetting`
  - Power-plan writes: `setPowerPlanStatus`, `addPowerPlan`, `deletePowerPlan`, `modifyPowerPlan`
  - Fan-mode writes: `setFanMode`

## Endpoint details

### appcgi.sysinfo.getTrimMachineType

#### Endpoint
`appcgi.sysinfo.getTrimMachineType`

#### Purpose
Return NAS machine-type identification.

#### CLI mapping
```
fygo-cli system info
```

CLI behavior:
- A saved session is not required.
- Print `response.data` as JSON; missing `data` is an error.

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `appcgi.sysinfo.getTrimMachineType` | `appcgi.sysinfo.getTrimMachineType` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |

#### Response
| Field | Always Present | Type | Meaning | Conditions / Notes | Example |
| --- | --- | --- | --- | --- | --- |
| `data` | no | object | Machine-type information | The CLI extracts this field | `{"type":"trim-xx"}` |
| `result` | no | string | Terminal marker | `succ`/`fail` | `succ` |
| `errno` | no | number | Error code | Present on failure | `65534` |
| `errmsg` | no | string | Error description | Present on failure | `permission denied` |

#### Protocol Notes
- Use a plain JSON request.

#### Field Semantics
- The backend defines the returned fields, which may vary by model and firmware version.

#### Errors
- Backend errors use the common error formatter.
- Missing `data` makes the command fail with a nonzero exit code.

### Other read-only endpoints

These endpoints are read-only, require no saved session, and print the response `data` object.

| CLI command | Endpoint | Request parameters | Purpose |
| --- | --- | --- | --- |
| `fygo-cli system trim-version` | `appcgi.sysinfo.getTrimVersion` | None | System version |
| `fygo-cli system kernel-version` | `appcgi.sysinfo.getKernelVersion` | None | Kernel version |
| `fygo-cli system host-name` | `appcgi.sysinfo.getHostName` | None | Host name |
| `fygo-cli system machine-id` | `appcgi.sysinfo.getMachineId` | None | Machine ID |
| `fygo-cli system uptime` | `appcgi.sysinfo.getUptime` | None | System uptime |
| `fygo-cli system all-vols` | `appcgi.sysinfo.getAllVolsInfo` | None | All volume information |
| `fygo-cli system vols` | `appcgi.sysinfo.getVolsList` | None | Volume list |
| `fygo-cli system docker` | `appcgi.sysinfo.getDockerInfo` | None | Docker-related system information |
| `fygo-cli system unix-time` | `appcgi.sysinfo.getUnixTime` | None | System Unix time |
| `fygo-cli system boot-on-power` | `appcgi.sysinfo.getBootOnPowerFlag` | None | Boot-on-power flag |
| `fygo-cli system hardware` | `appcgi.sysinfo.getHardwareInfo` | None | Hardware information |
| `fygo-cli system reserved-partition` | `appcgi.sysinfo.getReservedPartition` | None | Reserved-partition information |
| `fygo-cli system is-trim-machine` | `appcgi.sysinfo.isTrimMachine` | None | Whether this is a TRIM machine |
| `fygo-cli system trim-feature` | `appcgi.sysinfo.getTrimMachineFeature` | None | Machine capabilities |
| `fygo-cli system fan-mode` | `appcgi.sysinfo.getFanMode` | None | Fan mode |
| `fygo-cli system trim-disk` | `appcgi.sysinfo.getTrimDisk` | None | System/machine disk information |
| `fygo-cli system time-setting --lang en-US` | `appcgi.sysinfo.getTimeSetting` | Optional `lang` | Time settings |
| `fygo-cli system timezone-list --lang en-US` | `appcgi.sysinfo.getTimezoneList` | Optional `lang` | Timezone list |
| `fygo-cli system power-plan` | `appcgi.sysinfo.listPowerPlan` | None | Power-plan list |
| `fygo-cli system next-power-off-time` | `appcgi.sysinfo.nextPowerOffTime` | None | Next power-off time |
| `fygo-cli system power-plan-status` | `appcgi.sysinfo.getPowerPlanStatus` | None | Power-plan enabled state |
| `fygo-cli system power-on-seconds` | `appcgi.sysinfo.getPowerOnSeconds` | None | Powered-on seconds |
| `fygo-cli system inited-timestamp` | `appcgi.sysinfo.getSystemInitedTimestamp` | None | System initialization timestamp |

### Generic requests

| CLI command | Endpoint | Request parameters | Risk controls |
| --- | --- | --- | --- |
| `fygo-cli system request appcgi.sysinfo.* --json '<object>'` | User-selected `appcgi.sysinfo.*` endpoint | JSON object; must not contain `req` or `reqid` | Confirmation required by default; `--yes` skips the prompt |

Generic request constraints:
- The endpoint must start with `appcgi.sysinfo.`.
- `--json` must be a JSON object.
- The CLI generates `req` and `reqid`; omit both fields from user JSON.
- `system request` is generic and does not distinguish reads from writes; it always requires confirmation by default.
- `system request` authenticates with the saved session. A target/session mismatch fails before any anonymous configuration request is sent.

## Notes
- Returned field structures may vary by NAS model and firmware version.
- All current dedicated subcommands are read-only queries.
- Scheduled power operations depend on the hardware capabilities declared by the device. For a successful `system power-plan` response without `data`, the CLI prints `{"plan":[]}`, matching an empty plan list. An empty list does not prove scheduled power-on support. Other system queries still reject responses without `data`.
- `errno 100060031` indicates support only on official hardware; the CLI displays `this feature is supported only on official hardware`. Hardware-related queries such as `boot-on-power`, `trim-disk` and `power-on-seconds` may return this error.
- Power-plan, host-name, time-setting and fan-mode configuration endpoints have no dedicated commands yet; use `system request` only after confirming the risks.
