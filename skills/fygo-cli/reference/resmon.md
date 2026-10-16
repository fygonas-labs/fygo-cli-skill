# Resource monitoring reference

## Overview
Resource monitoring covers CPU, memory, network, GPU, disks, battery, system fans and process/service views.

## Conventions
- Distinguish aggregated metrics (`gen`) from category metrics (`cpu`, `mem`, and others).
- Endpoints use the `appcgi.resmon.*` namespace.

## Endpoint index
- Implemented:
  - `appcgi.resmon.cpu`
  - `appcgi.resmon.mem`
  - `appcgi.resmon.gen` (aggregated metrics)
  - `appcgi.resmon.net` (network)
  - `appcgi.resmon.disk` (disks)
  - `appcgi.resmon.gpu` (GPU)
  - `appcgi.resmon.npu` (NPU)
  - `appcgi.resmon.proc.info` (process details)
  - `appcgi.resmon.proc.list` (process list)
  - `appcgi.resmon.proc.srv` (service list)
  - `appcgi.resmon.sysWarn` (system warnings)
  - `appcgi.resmon.battery` (battery)
  - `appcgi.resmon.sysFan` (system fans)
  - `appcgi.resmon.alert.getSupportedBeepEvents` (supported beeper events)
  - `appcgi.resmon.alert.getBeepEvents` (configured beeper events)
  - `appcgi.resmon.alert.getBeepReasons` (beeper trigger reasons)
  - `appcgi.resmon.alert.muteBeeper` (mute beeper)
  - `appcgi.resmon.*` generic requests (via `monitor request`)
- Not exposed as dedicated commands:
  - Write endpoints without a dedicated command; use `monitor request` only after confirming the risks.

## Endpoint details

### appcgi.resmon.cpu

#### Endpoint
`appcgi.resmon.cpu`

#### Purpose
Return CPU monitoring metrics.

#### CLI mapping
```
fygo-cli monitor cpu
```

CLI behavior:
- Reuse the session restoration flow.
- Print endpoint data as JSON.

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `appcgi.resmon.cpu` | `appcgi.resmon.cpu` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |

#### Response
| Field | Always Present | Type | Meaning | Conditions / Notes | Example |
| --- | --- | --- | --- | --- | --- |
| `data` | no | object | CPU metrics | The CLI prints this object directly | `{"usage":37,"cores":4}` |
| `result` | no | string | Terminal marker | `succ`/`fail` | `succ` |
| `errno` | no | number | Error code | Present on failure | `5001` |
| `errmsg` | no | string | Error description | Present on failure | `monitor backend unavailable` |

#### Protocol Notes
- Use signed requests when a session secret is available.
- Signing follows the contract in `_conventions.md`.

#### Field Semantics
- The backend defines the response structure; the CLI does not remap fields.

#### Errors
- Backend errors are formatted consistently on stderr, including errno.
- The command exits with a nonzero status on failure.

### appcgi.resmon.mem

#### Endpoint
`appcgi.resmon.mem`

#### Purpose
Return memory monitoring metrics.

#### CLI mapping
```
fygo-cli monitor memory
```

CLI behavior:
- Reuse the session restoration flow.
- Print endpoint data as JSON.

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `appcgi.resmon.mem` | `appcgi.resmon.mem` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |

#### Response
| Field | Always Present | Type | Meaning | Conditions / Notes | Example |
| --- | --- | --- | --- | --- | --- |
| `data` | no | object | Memory metrics | The CLI prints this object directly | `{"total":1024,"used":768}` |
| `result` | no | string | Terminal marker | `succ`/`fail` | `succ` |
| `errno` | no | number | Error code | Present on failure | `5001` |
| `errmsg` | no | string | Error description | Present on failure | `monitor backend unavailable` |

#### Protocol Notes
- Use signed requests when a session secret is available.
- Signing follows the contract in `_conventions.md`.

#### Field Semantics
- The backend defines the response structure; the CLI does not remap fields.

#### Errors
- Backend errors are formatted consistently on stderr, including errno.
- The command exits with a nonzero status on failure.

### Other read-only endpoints

The endpoints below reuse the saved session, send authenticated requests and print the response `data` object.

| CLI command | Endpoint | Request parameters | Purpose |
| --- | --- | --- | --- |
| `fygo-cli monitor gen --item storeSpeed,netSpeed` | `appcgi.resmon.gen` | `item`: only `storeSpeed`, `netSpeed`, `cpuBusy`, `memPercent`; repeat `--item` or use comma-separated values | Aggregated metrics |
| `fygo-cli monitor net` | `appcgi.resmon.net` | None | Network metrics |
| `fygo-cli monitor disk` | `appcgi.resmon.disk` | None | Disk metrics |
| `fygo-cli monitor gpu` | `appcgi.resmon.gpu` | None | GPU metrics |
| `fygo-cli monitor npu` | `appcgi.resmon.npu` | None | NPU metrics |
| `fygo-cli monitor proc-info --pids 123,456` | `appcgi.resmon.proc.info` | `pids`: array of positive integers | Details for selected processes |
| `fygo-cli monitor proc-list` | `appcgi.resmon.proc.list` | None | Process list; responses may arrive in batches, which the CLI aggregates into a complete list |
| `fygo-cli monitor proc-srv` | `appcgi.resmon.proc.srv` | None | Service view |
| `fygo-cli monitor sys-warn` | `appcgi.resmon.sysWarn` | None | System warnings |
| `fygo-cli monitor battery` | `appcgi.resmon.battery` | None | Battery status |
| `fygo-cli monitor sys-fan` | `appcgi.resmon.sysFan` | None | System fans |
| `fygo-cli monitor beep-supported` | `appcgi.resmon.alert.getSupportedBeepEvents` | None | Supported beeper events |
| `fygo-cli monitor beep-events` | `appcgi.resmon.alert.getBeepEvents` | None | Configured beeper events |
| `fygo-cli monitor beep-reasons` | `appcgi.resmon.alert.getBeepReasons` | None | Current or historical trigger reasons |

Parameter constraints:
- `--item` accepts only `storeSpeed`, `netSpeed`, `cpuBusy`, `memPercent`; empty values are ignored and invalid metrics are rejected before sending a request.
- `--pids` accepts only positive integers; duplicate PIDs are removed.
- `beep-supported`, `beep-events`, `beep-reasons` apply only to TRIM machines with beeper capability. If a successful response lacks `data`, the CLI checks machine type and `beep` capability. It emits `[]` only for a confirmed non-TRIM machine or `beep: false`; unknown capability, `beep: true`, or missing `data` on other monitor commands remains an error.

### Writes and generic requests

| CLI command | Endpoint | Request parameters | Risk controls |
| --- | --- | --- | --- |
| `fygo-cli monitor mute-beeper` | `appcgi.resmon.alert.muteBeeper` | None | Confirmation required by default; `--yes` skips the prompt |
| `fygo-cli monitor request appcgi.resmon.* --json '<object>'` | User-selected `appcgi.resmon.*` endpoint | JSON object; must not contain `req` or `reqid` | Confirmation required by default; `--yes` skips the prompt |

Generic request constraints:
- The endpoint must start with `appcgi.resmon.`.
- `--json` must be a JSON object.
- The CLI generates `req` and `reqid`; omit both fields from user JSON.
- `monitor request` is generic and does not distinguish reads from writes; all endpoints require confirmation by default.

## Notes
- Read-only commands do not change device state.
- Before using `--yes` for an automated write or generic request, establish the endpoint semantics and JSON contents.
