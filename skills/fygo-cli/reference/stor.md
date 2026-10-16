# Storage reference

## Overview
Storage management covers pools, disks, SMART operations, mount/eject flows, capacity/space reports and user storage allocation.

## Conventions
- Distinguish operations that may wake disks from those that do not.
- Destructive operations and preflight endpoints are identified in error and protocol notes.
- Pool-target requests use `uuid`, not the legacy `name` field.

## Password verification for high-risk operations
fygo-cli uses `appcgi.usersrv.authUser` for password preflight checks on high-risk storage writes.

| Policy | Commands |
| --- | --- |
| Verify and forward the password | `storage umount`, `storage create`, `storage stop`, `storage add-disk`, `storage remove-disk`, `storage replace-disk`, `storage resize` |
| Verify without forwarding the password | `storage extend`, `storage format`, `storage eject` |
| Confirmation only, without password verification | `storage mount`, `storage disk-mount`, `storage disk-umount` |
| Generic requests | `storage request` requires confirmation by default; an explicit `--password` is verified before forwarding |

## Task routing

First distinguish reading from changing state:

| User intent | Preferred command | Notes |
| --- | --- | --- |
| Inspect storage overview | `fygo-cli storage overview` | Suitable starting point for read-only probes |
| List pools | `fygo-cli storage pools` | Confirm the pool UUID or `trim_*` identifier |
| List disks | `fygo-cli storage disks` | Confirm the plain device name |
| Inspect SMART / health | `fygo-cli storage smart <disk>` / `health <disk>` | Confirm that the target is a disk, not a pool |
| Inspect removable devices | `fygo-cli storage removable` / `disk-info <disk>` | Inspect before `format` / `eject` / `disk-mount` / `disk-umount` |
| Perform a high-risk write | Read `workflows/storage-dangerous-ops.md` first | Probe read-only state before deciding whether to proceed |
| Call an endpoint without a dedicated command | `fygo-cli storage request stor.<name> --json '<object>'` | Prefer dedicated commands; use only for uncovered `stor.*` endpoints |

## Common mistakes

- `mount` does not reverse `stop`; confirm the current pool state first.
- `extend` and `resize` are different operations.
- `format` / `eject` target removable devices, not ordinary pool management.
- Pool operations use UUIDs or `trim_*`; disk operations use plain device names.
- `--level` for `storage create` / `resize` accepts only `0`, `1`, `4`, `5`, `6`, `10`.

## High-risk precautions

- Run `storage pools` and `storage disks` before any write.
- Before creating a pool, also run `storage free-disks` to identify eligible candidates. For remaining space on a system disk, create using the returned `part`, not the whole-disk `name`.
- Stop if a command requires password verification and no password is available; do not guess.
- If read-only probes disagree with the user's description, stop and confirm the target.
- Operations such as `format`, `stop`, `remove-disk` and `replace-disk` have broad effects; do not batch them across multiple targets.
- `storage request` is a controlled fallback, not a way to bypass dedicated command validation and workflows.

## Endpoint index
- Pool and disk discovery (implemented):
  - `stor.listStor` — List storage pools
  - `stor.listDisk` — List disks
  - `stor.listFreeDisk` — List disk/partition candidates eligible for pool creation
  - `stor.general` — Storage overview
  - `stor.calcSpace` — Aggregated space
  - `stor.listRemovable` — Removable devices
  - `stor.diskInfo` — Selected disk details
- Pool lifecycle (implemented):
  - `stor.create` — Create a storage pool
  - `stor.mount` — Mount a storage pool
  - `stor.umount` — Unmount a storage pool
  - `stor.stop` — Stop/delete a storage pool
- Expansion and disk management (implemented):
  - `stor.addDisk` — Add disks
  - `stor.removeDisk` — Remove disks
  - `stor.replaceDisk` — Replace disks
  - `stor.extend` — Extend a storage pool
  - `stor.resize` — Resize/reshape a storage pool
- SMART and health (implemented):
  - `stor.diskHealth` — Disk health
  - `stor.diskSmart` — SMART information
- Removable devices (implemented):
  - `stor.diskMount` — Mount a removable disk
  - `stor.diskUmount` — Unmount a removable disk
  - `stor.format` — Format
  - `stor.eject` — Eject
- Generic requests (implemented):
  - `stor.*` — Use `storage request` for storage endpoints without a dedicated command
- Not exposed as dedicated commands:
  - `stor.state` / `stor.state2`, `stor.getSysPartInfo`
  - `stor.delRec`, `stor.lvActive`
  - `stor.readdDisk`, `stor.removeDiskCheck`, `stor.addSpareDisk`
  - `stor.enableSmart`, `stor.runSmartTest`, `stor.abortSmartTest`
  - Dedicated configuration commands: `stor.getConf`, `stor.setConf`, `stor.removeWarn`, `stor.setTrashbin`, and others
  - User storage: `stor.getUserStorage`, `stor.setUserStorage`

## Endpoint details

### stor.general

#### Endpoint
`stor.general`

#### Purpose
Get the storage overview.

#### CLI mapping
```
fygo-cli storage overview
```

#### Protocol Notes
- The legacy endpoint name `stor.getStorGeneral` is deprecated.

### stor.listStor

#### Endpoint
`stor.listStor`

#### Purpose
List pools and related disk metadata.

#### CLI mapping
```
fygo-cli storage pools
```

#### Protocol Notes
- Responses may have multiple stages: a basic pool list followed by removed-disk, hot-spare and cache details.
- Connected administrators may receive `{"sysNotify":"storageChanged"}`; refresh the output when this occurs.

#### Field Semantics
- Some firmware pool responses include `degraded` and `fixNeedDisks`, but their precise meaning and consistency across firmware are not established. A nonzero value alone does not establish disk failure, required repair or a data-risk level.
- SMART, `md[].arrayState` and `md[].syncAction` offer related observations, but their relationship to those fields is not established. Preserve raw values and verify them in the Fygo storage page.
- The CLI passes these fields through without interpreting them or automatically repairing anything.

### stor.listDisk

#### Endpoint
`stor.listDisk`

#### Purpose
List disks known to the storage service.

#### CLI mapping
```
fygo-cli storage disks
```

#### Protocol Notes
- Disk data frames may precede the successful terminal frame. The CLI collects `disk` arrays from all frames and merges duplicates by `name`.
- If no frame contains disk data, the output contains an empty `disk` array.

#### Field Semantics
- Returned disk identifiers are the canonical disk names used by `stor.diskHealth`, `stor.diskSmart`, `stor.format` and `stor.eject`.

### stor.diskInfo

#### Endpoint
`stor.diskInfo`

#### Purpose
Query a disk's details, commonly to confirm removable-disk partitions, capacity and filesystem.

#### CLI mapping
```
fygo-cli storage disk-info <disk>
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `stor.diskInfo` | `stor.diskInfo` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `disk` | body | yes | string | Disk device name | Plain device name such as `sdc` or `nvme0n1` | `sdc` |

### stor.diskMount / stor.diskUmount

#### Endpoint
`stor.diskMount` and `stor.diskUmount`

#### Purpose
Mount or unmount the selected removable disk.

#### CLI mapping
```
fygo-cli storage disk-mount <disk> [--yes]
fygo-cli storage disk-umount <disk> [--yes]
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | `stor.diskMount` or `stor.diskUmount` | `stor.diskMount` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `disk` | body | yes | string | Removable disk device name | Confirm using `storage removable` / `storage disk-info` | `sdc` |

#### Protocol Notes
- Interactive confirmation is required by default; `--yes` skips the prompt.
- No password verification is performed.

### stor.listFreeDisk

#### Endpoint
`stor.listFreeDisk`

#### Purpose
List disks or partitions eligible for creating a pool.

#### CLI mapping
```
fygo-cli storage free-disks
```

#### Protocol Notes
- This endpoint identifies creation candidates; it differs from the general disk inventory returned by `storage disks`.
- Ordinary data-disk candidates usually use `name` directly for creation.
- Available space on a system disk is usually represented by `sys: 1`, `part` and `partSize`. Pass `part` to `--disks` when creating; use `partSize` for capacity.

#### Field Semantics
- The top-level response usually contains a `disk` array.
- `name` is the whole-disk device name.
- `part` names a partition eligible for pool creation, appearing only in cases such as remaining system-disk space.
- `partSize` is the capacity of the eligible partition.

### stor.calcSpace

#### Endpoint
`stor.calcSpace`

#### Purpose
Report aggregated space usage for system and storage partitions.

#### CLI mapping
```
fygo-cli storage space
```

### stor.listRemovable

#### Endpoint
`stor.listRemovable`

#### Purpose
List mounted removable storage devices.

#### CLI mapping
```
fygo-cli storage removable
```

### stor.diskHealth / stor.diskSmart

#### Endpoint
`stor.diskHealth` and `stor.diskSmart`

#### Purpose
Query disk health metrics and SMART details.

#### CLI mapping
```
fygo-cli storage health <disk>
fygo-cli storage smart <disk>
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | `stor.diskHealth` or `stor.diskSmart` | `stor.diskHealth` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `disks` | body | yes | string[] | Disk device name list for `stor.diskHealth` | The CLI's single-disk command sends a one-element array | `["nvme0n1"]` |
| `disk` | body | yes | string | Disk device name for `stor.diskSmart` | The CLI sends a plain device name | `nvme0n1` |
| `wakeup` | body | no | boolean | Wake disks before querying | `diskHealth` sends `true`; `diskSmart` uses the device default | `true` |

#### Protocol Notes
- A sleeping disk may return `diskStandby` from `stor.diskSmart` instead of full SMART data.
- `listDisk` aggregates data frames preceding the terminal frame and returns `disk: []` for a genuinely empty list. Successful `listFreeDisk` without candidates adds `disk: []`; successful `listRemovable` without devices adds `removable: []`.
- `diskHealth` uses the `disks` array; `diskSmart` uses singular `disk`.

### stor.create

#### Endpoint
`stor.create`

#### Purpose
Create a pool from the specified RAID/storage level and disk set.

#### CLI mapping
```
fygo-cli storage create --level <level> --disks <disk> [--disks <disk>...] [--comment <comment>] [--fstype <fstype>] [--check-disk] [--password <password>] [--yes]
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `stor.create` | `stor.create` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `level` | body | yes | number | RAID/storage level | Allowed: `0`, `1`, `4`, `5`, `6`, `10`; `0` means RAID 0 and requires at least two disks | `1` |
| `disks` | body | yes | string[] | Disk device name list | Repeat `--disks` or use comma-separated values; duplicates are removed and disk counts validated by level | `["sdb","sdc"]` |
| `comment` | body | no | string | Pool comment | Optional | `pool-a` |
| `fstype` | body | no | string | Filesystem type | Lowercase identifier such as `ext4` | `ext4` |
| `checkDisk` | body | no | boolean | Disk preflight check | Sent when `--check-disk` is set | `true` |
| `password` | body | no | string | Password | Forwarded after preflight verification | |

#### Field Semantics
- `level` is numeric, matching the backend request contract.
- `disks` must contain plain device names; `/dev/sdb` is not accepted.
- Local disk-count validation runs before creation: `0` needs at least two, `5` at least three, `6` and `10` at least four; `10` also requires an even number.

### stor.mount / stor.umount

#### Endpoint
`stor.mount` and `stor.umount`

#### Purpose
Mount or unmount a storage pool.

#### CLI mapping
```
fygo-cli storage mount <uuid> [-y]
fygo-cli storage umount <uuid> [--password <password>] [-y]
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | `stor.mount` or `stor.umount` | `stor.mount` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `uuid` | body | yes | string | Storage pool UUID | Do not use legacy `name` | `trim_pool` |
| `password` | body | conditional | string | Password | Required only for `umount` | |

#### Protocol Notes
- `storage mount` requires confirmation only, without password verification.
- `storage umount` verifies the password before forwarding it.
- `stor.mount` is not a general undo operation for `stor.stop`.

### stor.stop

#### Endpoint
`stor.stop`

#### Purpose
Stop and delete a storage pool.

#### CLI mapping
```
fygo-cli storage stop <uuid> [--password <password>] [-y]
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `stor.stop` | `stor.stop` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `uuid` | body | yes | string | Storage pool UUID | | `trim_pool` |
| `password` | body | no | string | Password | Forwarded after preflight verification | |

#### Field Semantics
- `stor.stop` is the primary delete-storage-space operation, applicable to ordinary pools, inactive pools and pools with damaged caches.

### stor.addDisk

#### Endpoint
`stor.addDisk`

#### Purpose
Add disks to a pool.

#### CLI mapping
```
fygo-cli storage add-disk <uuid> --disks <disk> [--disks <disk>...] [--password <password>] [-y]
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `stor.addDisk` | `stor.addDisk` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `uuid` | body | yes | string | Storage pool UUID | | `trim_pool` |
| `disks` | body | yes | string[] | Disks to add | Repeat the option or use comma-separated values | `["sdd"]` |
| `password` | body | no | string | Password | Forwarded after preflight verification | |

### stor.removeDisk

#### Endpoint
`stor.removeDisk`

#### Purpose
Remove disks from a pool.

#### CLI mapping
```
fygo-cli storage remove-disk <uuid> --disks <disk> [--password <password>] [-y]
```

### stor.replaceDisk

#### Endpoint
`stor.replaceDisk`

#### Purpose
Replace disks in a pool.

#### CLI mapping
```
fygo-cli storage replace-disk <uuid> --disks <disk> --new-disks <disk> [--password <password>] [-y]
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `stor.replaceDisk` | `stor.replaceDisk` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `uuid` | body | yes | string | Storage pool UUID | | `trim_pool` |
| `disks` | body | yes | string[] | Old disks to replace | | `["sdb"]` |
| `newDisks` | body | yes | string[] | Replacement disks | Must have the same length as `disks` | `["sdg"]` |
| `password` | body | yes | string | Password | Forwarded after preflight verification | |

#### Field Semantics
- `disks[i]` and `newDisks[i]` form a replacement pair.

### stor.extend

#### Endpoint
`stor.extend`

#### Purpose
Start the pool expansion flow.

#### CLI mapping
```
fygo-cli storage extend <uuid> [--password <password>] [-y]
```

#### Protocol Notes
- Verify the password without forwarding it; the target payload contains only `uuid`.

### stor.resize

#### Endpoint
`stor.resize`

#### Purpose
Resize a pool or reshape its layout.

#### CLI mapping
```
fygo-cli storage resize <uuid> --disks <disk> [--vd-name <name>] [--level <level>] [--password <password>] [-y]
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `stor.resize` | `stor.resize` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `uuid` | body | yes | string | Storage pool UUID | | `trim_pool` |
| `disks` | body | yes | string[] | Disks participating in the resize | | `["sdh"]` |
| `vdName` | body | no | string | Target virtual device name | Optional | `raidz1-0` |
| `level` | body | no | number | Target RAID level | Allowed: `0`, `1`, `4`, `5`, `6`, `10` | `5` |
| `password` | body | no | string | Password | Forwarded after preflight verification | |

### stor.format

#### Endpoint
`stor.format`

#### Purpose
Format a removable disk or partition.

#### CLI mapping
```
fygo-cli storage format <disk> --fstype <fstype> [--partition <no>] [--password <password>] [-y]
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `stor.format` | `stor.format` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `disk` | body | yes | string | Disk device name | Plain device name | `sdb` |
| `no` | body | yes | number | Partition number | `0` means the whole disk | `0` |
| `fstype` | body | yes | string | Filesystem type | Lowercase | `ext4` |

### stor.eject

#### Endpoint
`stor.eject`

#### Purpose
Safely eject a removable disk.

#### CLI mapping
```
fygo-cli storage eject <disk> [--password <password>] [-y]
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `stor.eject` | `stor.eject` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `disk` | body | yes | string | Removable disk device name | Use an identifier returned by `stor.listRemovable` | `sdc` |

### stor.* generic request

#### Endpoint
Any endpoint with the `stor.` prefix.

#### Purpose
Send a controlled generic storage request when no dedicated command covers the endpoint.

#### CLI mapping
```
fygo-cli storage request stor.<name> [--json '<object>'] [--password <password>] [--yes]
```

#### Request Rules
- `endpoint` must start with `stor.`.
- Endpoints covered by dedicated commands are rejected; use the matching dedicated command.
- `--json` must be a JSON object.
- `--json` must not contain `req`, `reqid` or `password`. The CLI generates `req` / `reqid`; passwords must come through `--password`.
- Interactive confirmation is required by default; `--yes` skips the prompt.
- With `--password`, the CLI verifies it before adding it to the target `stor.*` request.

#### Usage Notes
- Prefer dedicated commands when available: they provide more precise local validation and dangerous-operation workflows.
- Treat endpoints with unknown mutation behavior as dangerous: probe read-only state and confirm the target first.

## Notes
- Exact response-field differences between pool-state endpoints (`state` vs `state2`) are not fully established.
- Some advanced endpoints, including cache-device operations, `stor.delRec` and `stor.lvActive`, are not exposed as dedicated commands.
- Common responses carry `result` (`succ`/`fail`), `errno` and `errmsg` fields.
