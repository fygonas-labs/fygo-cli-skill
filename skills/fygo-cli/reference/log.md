# Log reference

## Overview
The log center supports queries, module enumeration, clearing, export and archive policy management.

## Conventions
- Keep operational logs, audit logs and log-service controls distinct.
- The current fygo-cli log-center endpoints use `appcgi.eventlogger.common.*`.

## Endpoint index
- Implemented:
  - `appcgi.eventlogger.common.list`
  - `appcgi.eventlogger.common.moduleList`
  - `appcgi.eventlogger.common.clear`
  - `appcgi.eventlogger.common.export`
  - `appcgi.eventlogger.common.archive` (set archive policy)
  - `appcgi.eventlogger.common.archive.get` (query archive policy)
  - `appcgi.eventlogger.debuglog.copyStart` (start copying diagnostic logs)
  - `appcgi.eventlogger.debuglog.copyStop` (cancel diagnostic-log copying)
- Not exposed as dedicated commands:
  - `log.*` (legacy log-query family)

## Endpoint details

### appcgi.eventlogger.common.list

#### Endpoint
`appcgi.eventlogger.common.list`

#### Purpose
Query paginated log-center records, optionally filtered by level and module.

#### CLI mapping
```
fygo-cli logger list [--page <n>] [--page-size <n>] [--level <n>] [--module <n>] [--locale <locale>]
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value | `appcgi.eventlogger.common.list` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `pageSize` | body | no | number | Records per page | Positive integer | `50` |
| `page` | body | no | number | Page number | Positive integer | `1` |
| `level` | body | no | number | Log level filter | Numeric level identifier | `3` |
| `module` | body | no | number | Log module filter | Numeric module identifier | `1` |
| `locale` | body | no | string | Language/locale | Controls the language of returned text | `en-US` |

#### Response
| Field | Always Present | Type | Meaning | Conditions / Notes | Example |
| --- | --- | --- | --- | --- | --- |
| `data` | no | object | Log data | Contains records and pagination | `{...}` |
| `result` | no | string | Terminal marker | `succ`/`fail` | `succ` |
| `errno` | no | number | Error code | Present on failure | `65534` |
| `errmsg` | no | string | Error description | Present on failure | `invalid filter` |

### appcgi.eventlogger.common.moduleList

#### Endpoint
`appcgi.eventlogger.common.moduleList`

#### Purpose
List available log modules.

#### CLI mapping
```
fygo-cli logger modules [--locale <locale>]
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value | `appcgi.eventlogger.common.moduleList` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `locale` | body | no | string | Language/locale | Controls the language of module names | `en-US` |

### appcgi.eventlogger.common.clear

#### Endpoint
`appcgi.eventlogger.common.clear`

#### Purpose
Clear log records by level and module.

#### CLI mapping
```
fygo-cli logger clear --level <level> --module <module> --yes
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value | `appcgi.eventlogger.common.clear` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `level` | body | yes | number | Log level | Required | `3` |
| `module` | body | yes | number | Log module | Required | `1` |

### appcgi.eventlogger.common.export

#### Endpoint
`appcgi.eventlogger.common.export`

#### Purpose
Export log records by level and module.

#### CLI mapping
```
fygo-cli logger export --level <level> --module <module> [--locale <locale>] --yes
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value | `appcgi.eventlogger.common.export` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `level` | body | yes | number | Log level | Required | `3` |
| `module` | body | yes | number | Log module | Required | `1` |
| `locale` | body | no | string | Language/locale | Controls the language of exported content | `en-US` |

### appcgi.eventlogger.common.archive

#### Endpoint
`appcgi.eventlogger.common.archive`

#### Purpose
Set the log archive policy.

#### CLI mapping
```
fygo-cli logger archive set --switch <0|1> --file-path <path> [--size-gt <n>] [--date-unit <n>] [--date-before <n>] --yes
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value | `appcgi.eventlogger.common.archive` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `switch` | body | yes | number | Archive switch | `0` disables; `1` enables | `1` |
| `filePath` | body | yes | string | Archive file path | May be an empty string when archiving is disabled | `/vol1/logs` |
| `sizeGt` | body | no | number | File size threshold | Optional | `1024` |
| `dateUnit` | body | no | number | Date unit | `0` day, `1` week, `2` month, `3` year | `2` |
| `dateBefore` | body | no | number | Date offset | Optional | `3` |

#### Field Semantics
- `switch: 0` disables archiving; `filePath` may then be empty.
- `dateUnit` selects granularity: `0` day, `1` week, `2` month, `3` year.

### appcgi.eventlogger.common.archive.get

#### Endpoint
`appcgi.eventlogger.common.archive.get`

#### Purpose
Query the current log archive configuration.

#### CLI mapping
```
fygo-cli logger archive query
```

#### Protocol Notes
- The response contains the current archive configuration; the CLI prints the backend payload directly.
- If present, `validPath` reports whether the path is valid.

### appcgi.eventlogger.debuglog.copyStart

#### Endpoint
`appcgi.eventlogger.debuglog.copyStart`

#### Purpose
Copy backend diagnostic logs (Debug Log) to the specified output directory and return task progress.

#### CLI mapping
```
fygo-cli logger debuglog copy-start --output-dir <path> [--srv-type <0|1|2>]... --yes
fygo-cli diagnostic-log export --output-dir <path> [--srv-type <0|1|2>]... --yes
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value | `appcgi.eventlogger.debuglog.copyStart` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `outputDir` | body | yes | string | Output directory | Requires a mounted storage or external-disk path, such as `/vol1/<dir>` or `/vol00/<mount>/<dir>`; system directories such as `/tmp`, `/var/tmp` and `/home/...` are not valid targets | `/vol00/RemovableDisk/debug-log` |
| `srvType` | body | no | number[] | Copy categories | `0` system, `1` built-in applications, `2` built-in services; omission selects all categories on the backend. Repeat the option or use comma-separated values; duplicates are removed in order. | `[0,1,2]` |

#### Response
| Field | Always Present | Type | Meaning | Conditions / Notes | Example |
| --- | --- | --- | --- | --- | --- |
| `taskId` | no | string | Copy task ID | Returned in `data` of doing/success progress frames | `ftask_1` |
| `fileNumTotal` | no | number | Total files to copy | Returned in progress frames | `51` |
| `fileNumDone` | no | number | Files already copied | Returned in progress frames | `2` |

#### Protocol Notes
- The backend may return `result:"doing"` progress frames before the terminal frame. The CLI prefers a progress payload containing `taskId`, so a later `diagnostic-log stop` can target it.
- `outputDir` must already exist under mounted storage or an external disk on the device. Typical storage paths are `/volN/<uid>/...`; external disks commonly use `/vol00/<mount-name>/...`.

#### Errors
- Common errors include missing `outputDir`, nonexistent directory, directory creation failure, unwritable directory, insufficient disk space, duplicate task and missing task.

### appcgi.eventlogger.debuglog.copyStop

#### Endpoint
`appcgi.eventlogger.debuglog.copyStop`

#### Purpose
Cancel an ongoing diagnostic-log copy task.

#### CLI mapping
```
fygo-cli logger debuglog copy-stop --task-id <taskId> --yes
fygo-cli diagnostic-log stop --task-id <taskId> --yes
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value | `appcgi.eventlogger.debuglog.copyStop` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `taskId` | body | yes | string | Copy task ID | Must be nonempty and contain no whitespace | `ftask_1` |

## Notes
- Legacy `log.*` endpoints and current `appcgi.eventlogger.common.*` endpoints may coexist on different firmware versions.
- Choose a mounted storage or external-disk path for `outputDir` in `logger debuglog copy-start` / `diagnostic-log export`; system-directory paths are not valid diagnostic-log export targets.
