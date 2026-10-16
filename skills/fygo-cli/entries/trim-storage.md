---
name: trim-storage
description: Inspect storage pools, disks and SMART data, or perform mounting, expansion, disk replacement, formatting and other high-risk storage writes.
---

# Storage

Read `../reference/workflows/storage-dangerous-ops.md` before writes, and
`../reference/stor.md` for the API contract.

## Rules

- Pools use UUIDs or `trim_*` identifiers; disks use plain device names such as
  `sda` or `nvme0n1`.
- Before high-risk writes, run `storage pools` and `storage disks`.
- Before creating a pool, run `storage free-disks`. For free space on the system
  disk, use the returned `part`, not the whole-disk name.
- Stop when a required password is unavailable; do not assume a default.
- Prefer named commands. `storage request` rejects endpoints already covered by named commands.

## Commands

```bash
fygo-cli storage overview
fygo-cli storage pools
fygo-cli storage disks
fygo-cli storage free-disks
fygo-cli storage disk-info <disk>
fygo-cli storage disk-mount <disk> --yes
fygo-cli storage disk-umount <disk> --yes
fygo-cli storage smart <disk>
fygo-cli storage health <disk>
fygo-cli storage request stor.getConf --json '{"uuid":"trim_pool"}' --yes
```
