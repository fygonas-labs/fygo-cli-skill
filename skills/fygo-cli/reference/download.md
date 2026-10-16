# download module

## Overview
The Download Center module maps to the `appcgi.downloadcenter.*` namespace. It creates download tasks from URLs, magnet links, or torrent files, and supports task queries, control, and statistics.

Download Center requests use the same `{ req, reqid, ...params }` format as other CGI calls, but some endpoints return business data through `data.rsp`, `data.body`, or `data.block.data`.

## Protocol model

### Request model

Typical request format:

```json
{
  "reqid": "250318103000ABCDEF0123456789ABCD",
  "req": "appcgi.downloadcenter.task.query",
  "init_flag": true,
  "state_filter": 65535
}
```

Notes:
- Some parameters use camelCase at the API layer (such as `initFlag`) but snake_case at the CGI layer (such as `init_flag`).
- Most endpoints accept an optional `uid` parameter.
- `save_dir` is required in practice for `task.addUris`.

### Response model

Response data may be nested in different locations:

```json
{
  "reqid": "...",
  "result": "succ",
  "req": "appcgi.downloadcenter.task.query",
  "data": {
    "block": {
      "id": "20260318160514HlEnuPgC19HGBlXP",
      "total_size": 41653,
      "offset": 0,
      "data": "{\"tasks\":[],\"stats\":{}}"
    }
  }
}
```

Business data may reside in `data.rsp`, `data.body`, or `data.block.data` (decode JSON strings).

### Terminal states and error handling

Terminal values: `succ`, `fail`, `cancel`

Handling rules:
- Transport completion and business success are independent concerns.
- Preserve `errno`, `errmsg`, and `extra`.
- Some control endpoints return a simple `"ok"` on success.
- Some endpoints return the `{ code: 200, body: {} }` format.
- Batch creation endpoints may return `errs[]` even when the request itself succeeds.

### Block transfer

- A single packet must not exceed 1 MiB.
- Large payloads use the same `reqid` across blocks.
- Responses include `block.id`, `block.total_size`, `block.offset`, and `block.data`.

## Endpoint index

### Queries (implemented)
- `appcgi.downloadcenter.task.query` — Task list and statistics
- `appcgi.downloadcenter.task.search` — Keyword search
- `appcgi.downloadcenter.task.getInfo` — Task details
- `appcgi.downloadcenter.task.getFiles` — Task file list

### Creation (implemented)
- `appcgi.downloadcenter.task.addUris` — Create from a URL or magnet link
- `appcgi.downloadcenter.task.addPaths` — Create from a torrent file on the NAS

### Control (implemented)
- `appcgi.downloadcenter.task.pause` — Pause tasks
- `appcgi.downloadcenter.task.resume` — Resume tasks
- `appcgi.downloadcenter.task.retry` — Retry tasks
- `appcgi.downloadcenter.task.delete` — Delete tasks

### Statistics (implemented)
- `appcgi.downloadcenter.stat.all` — Aggregate statistics

### Generic request entry point

Use the following entry point for `appcgi.downloadcenter.*` endpoints without a dedicated subcommand:

```bash
fygo-cli download request appcgi.downloadcenter.<name> --json '<object>' --yes
```

- `--json` must be a JSON object without `req` or `reqid`; the CLI injects them.
- Interactive confirmation is required by default; use `--yes` for automation or after confirming the risks.
- Responses are unpacked from the usual Download Center locations: `data.rsp`, `data.body`, and `data.block.data`.

Endpoints available through `download request` include:
- Creation:`task.addfnShareLink`, `task.getfnShareLinkStatus`
- Control:`task.pin`, `task.setForceStart`, `task.setFilePriority`, `task.exportTorrentFile`, `task.setDownloadLimit`, `task.setUploadLimit`, `task.setShareLimit`
- Configuration:`config.getDefaultSaveDir`, `config.setDefaultSaveDir`, `config.getTransferCfg`, `config.setTransferCfg`, `config.getNetworkCfg`, `config.setNetworkCfg`, `config.getAltCfg`, `config.setAltCfg`, `config.getListColumns`, `config.setListColumns`
- Statistics:`stat.qbittorrent`, `stat.aria2`
- Trackers:`task.getTrackers`, `task.addTrackers`, `task.removeTrackers`, `task.editTracker`, `tracker.query`, `tracker.insert`, `tracker.delete`
- Peers:`task.getPeers`, `task.addPeers`, `transfer.banPeers`
- Scan directories:`scanDir.query`, `scanDir.insert`, `scanDir.delete`
- Utilities:`util.getDirFreeSpace`, `util.isTcpPortInUse`
- Administration:`admin.getUsers`

`task.addUploadFile` involves uploaded file contents and block transfer, not an ordinary JSON parameter request. Prefer uploading the local file with file-upload commands first, then call the Download Center endpoint according to device behavior.

## Endpoint details

### appcgi.downloadcenter.task.query

#### Endpoint
`appcgi.downloadcenter.task.query`

#### Purpose
Query the task list, aggregate statistics, and download-engine errors.

#### CLI mapping
```
fygo-cli download ls
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value | `appcgi.downloadcenter.task.query` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `engine_types` | body | no | array | Engine-type filter | Optional | `[1,2]` |
| `ids` | body | no | array | Specific task IDs | Optional | `[42,43]` |
| `state_filter` | body | no | number | State filter bitmask | CLI default `65535` (all states) | `65535` |
| `type_filter` | body | no | number | Type filter | Optional | `1` |
| `init_flag` | body | no | boolean | Initialization flag | CLI default `true` | `true` |
| `uid` | body | no | number | User ID | Optional | `1000` |

#### Response
Main response fields (after decoding `data.block.data`):

| Field | Always Present | Type | Meaning | Conditions / Notes | Example |
| --- | --- | --- | --- | --- | --- |
| `tasks` | no | array | Task list | Includes task summaries | `[...]` |
| `stats` | no | object | Aggregate statistics | Speed and size statistics | `{...}` |
| `errors` | no | array | Engine errors | Errors reported by download engines | `[...]` |

Common task fields:`id`, `type`, `name`, `state`, `total_size`, `size`, `dl_speed`, `up_speed`, `save_dir`, `uri`, `addition_time`, `error_code`, `error_message`

### appcgi.downloadcenter.task.search

#### Endpoint
`appcgi.downloadcenter.task.search`

#### Purpose
Search download tasks by task name or file name.

#### CLI mapping
```
fygo-cli download ls <keyword>
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value | `appcgi.downloadcenter.task.search` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `keyword` | body | yes | string | Search keyword | Match task name or file name | `ubuntu` |
| `uid` | body | no | number | User ID | Optional | `1000` |

### appcgi.downloadcenter.task.getInfo

#### Endpoint
`appcgi.downloadcenter.task.getInfo`

#### Purpose
Get detailed information about one download task.

#### CLI mapping
```
fygo-cli download info <id>
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value | `appcgi.downloadcenter.task.getInfo` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `id` | body | yes | number | Task ID | Positive integer | `42` |
| `uid` | body | no | number | User ID | Optional | `1000` |

#### Response
Main response fields (inside `body`):

| Field | Always Present | Type | Meaning | Conditions / Notes | Example |
| --- | --- | --- | --- | --- | --- |
| `name` | no | string | Task name | | `ubuntu-22.04.iso` |
| `total_size` | no | number | Total size (bytes) | | `4194304` |
| `save_path` | no | string | Save path | | `/vol1/downloads` |
| `dl_speed` | no | number | Download speed | | `1048576` |
| `up_speed` | no | number | Upload speed | | `0` |
| `eta` | no | number | Estimated remaining time | | `120` |
| `seeds` | no | number | Seed count | | `5` |
| `peers` | no | number | Peer count | | `12` |
| `share_ratio` | no | number | Share ratio | | `0.5` |
| `addition_date` | no | number | Addition time | | `1710000000` |
| `completion_date` | no | number | Completion time | | `1710100000` |

### appcgi.downloadcenter.task.getFiles

#### Endpoint
`appcgi.downloadcenter.task.getFiles`

#### Purpose
Get the file list of a download task, especially useful for BitTorrent tasks.

#### CLI mapping
```
fygo-cli download files <id>
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value | `appcgi.downloadcenter.task.getFiles` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `id` | body | yes | number | Task ID | Positive integer | `42` |
| `mode` | body | no | number | Retrieval mode | `1` retrieves immediately | `1` |
| `uid` | body | no | number | User ID | Optional | `1000` |

#### Response
Main response fields (inside `body[]`):

| Field | Always Present | Type | Meaning | Conditions / Notes | Example |
| --- | --- | --- | --- | --- | --- |
| `index` | no | number | File index | | `0` |
| `name` | no | string | File name | | `readme.txt` |
| `size` | no | number | File size | | `1024` |
| `progress` | no | number | Download progress | | `0.75` |
| `priority` | no | number | Priority | `0` skip, `1` normal, `7` highest | `1` |

### appcgi.downloadcenter.task.addUris

#### Endpoint
`appcgi.downloadcenter.task.addUris`

#### Purpose
Create download tasks from URLs or magnet links.

#### CLI mapping
```
fygo-cli download add-uri <uri> <saveDir> --yes
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value | `appcgi.downloadcenter.task.addUris` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `uris` | body | yes | string[] | URI list | Supports http/https/ftp/sftp/magnet | `["magnet:?xt=..."]` |
| `save_dir` | body | yes | string | Save directory | Must use the `/vol{v}/...` format | `/vol1/downloads` |
| `select_files` | body | no | array | Selected files | Relevant to BitTorrent inputs | `[0,1,2]` |
| `uris_type` | body | no | number | URI type | `0` default, `1` qBittorrent | `0` |

#### Response
| Field | Always Present | Type | Meaning | Conditions / Notes | Example |
| --- | --- | --- | --- | --- | --- |
| `ids` | no | number[] | Created task IDs | | `[42]` |
| `errs` | no | array | Per-item errors | Present when the request succeeds but individual items fail | `[{"uri":"...","err":"..."}]` |

### appcgi.downloadcenter.task.addPaths

#### Endpoint
`appcgi.downloadcenter.task.addPaths`

#### Purpose
Create download tasks from existing torrent files on the NAS.

#### CLI mapping
```
fygo-cli download add-path <path> <saveDir> --yes
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value | `appcgi.downloadcenter.task.addPaths` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `paths` | body | yes | string[] | Torrent-file path list | Must use the `/vol{v}/...` format | `["/vol1/torrents/demo.torrent"]` |
| `save_dir` | body | yes | string | Save directory | Must use the `/vol{v}/...` format | `/vol1/downloads` |
| `select_files` | body | no | array | Selected files | Optional | `[0,1]` |

#### Response
| Field | Always Present | Type | Meaning | Conditions / Notes | Example |
| --- | --- | --- | --- | --- | --- |
| `ids` | no | number[] | Created task IDs | | `[43]` |
| `errs` | no | array | Per-item errors | Present when the request succeeds but individual items fail | `[...]` |

### Control endpoints

#### appcgi.downloadcenter.task.pause / resume / retry / delete

These four endpoints share the same request structure.

#### CLI mapping
```
fygo-cli download pause <id...> --yes
fygo-cli download resume <id...> --yes
fygo-cli download retry <id...> --yes
fygo-cli download rm <id...> --yes
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Select the operation | `appcgi.downloadcenter.task.pause` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `ids` | body | yes | number[] | Task ID list | Supports batch operations | `[42, 43]` |
| `uid` | body | no | number | User ID | Optional | `1000` |
| `delete_files` | body | no | boolean | Also delete files | Only available for `delete`; the CLI preserves files by default | `false` |

#### Response
Usually returns `"ok"` on success.

### appcgi.downloadcenter.stat.all

#### Endpoint
`appcgi.downloadcenter.stat.all`

#### Purpose
Get aggregate Download Center statistics.

#### CLI mapping
```
fygo-cli download stat
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value | `appcgi.downloadcenter.stat.all` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `uid` | body | no | number | User ID | Optional | `1000` |

#### Response
| Field | Always Present | Type | Meaning | Conditions / Notes | Example |
| --- | --- | --- | --- | --- | --- |
| `dl_speed` | no | number | Total download speed | | `1048576` |
| `dl_total_size` | no | number | Total downloaded bytes | | `10737418240` |
| `up_speed` | no | number | Total upload speed | | `524288` |
| `up_total_size` | no | number | Total uploaded bytes | | `5368709120` |

## Notes
- File upload (`task.addUploadFile`) requires file contents and block-transfer support; do not invoke it as an ordinary `download request --json` call.
- `task.addfnShareLink` imports a task into Download Center; do not confuse it with the `appcgi.sharesvr.share.link.*` shared-link management endpoints.
- Specific response field structures may vary by backend version.
