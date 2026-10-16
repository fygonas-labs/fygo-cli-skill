---
name: trim-download
description: List, search, create and control download tasks; inspect task files, statistics and state transitions.
---

# Downloads

Read `../reference/download.md` for command signatures and endpoint fields.
Use for URI, magnet and torrent tasks, save directories, priorities and task retries.

## Rules

- Creating tasks uses `download add-uri <uri> <saveDir>` or
  `download add-path <path> <saveDir>`; save directories are concrete `/vol{n}/...` paths.
- `download stat` reports aggregate statistics, not one file's metadata.
- `download pause|resume|retry|rm <id...>` affects existing tasks and does not infer paths.
- Task creation and control require explicit `--yes`.
- Paths in `download info` / `download files` directly reflect Fygo volume
  definitions; use the returned paths rather than deriving replacements.
- Unwrapped `appcgi.downloadcenter.*` endpoints use `download request`; omit
  `req` and `reqid` from JSON.

## Commands

```bash
fygo-cli download ls
fygo-cli download ls installer
fygo-cli download info <id>
fygo-cli download files <id>
fygo-cli download add-uri https://example.com/file.iso /vol1/download --yes
fygo-cli download add-path /vol1/source.torrent /vol1/download --yes
fygo-cli download pause <id...> --yes
fygo-cli download resume <id...> --yes
fygo-cli download retry <id...> --yes
fygo-cli download rm <id...> --yes
fygo-cli download stat
fygo-cli download request appcgi.downloadcenter.tracker.query --json '{}' --yes
```
