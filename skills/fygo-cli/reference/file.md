# File module

## Overview
Filesystem operations: file lists, directory creation, deletion, copying, moving, finder search, shared-directory/ACL workflows, transfer tasks, trash, favorites, recent files and team files.

## Module conventions
- Paths must use canonical Fygo `/vol{v}/...` form; endpoint descriptions explicitly state aggregate-root behavior.
- List/search endpoints may stream multiple packets; their terminal-packet rules must be explicit.
- Shared-directory `file.share.*` endpoints belong here; share-link management under `appcgi.sharesvr.share.link.*` belongs to the separate share module.

## Task routing

Choose a command by user intent, then read its endpoint details below:

| Intent | Preferred command | Notes |
| --- | --- | --- |
| List currently visible directories | `fygo-cli file ls` | No argument and `/` both omit `path` |
| List a specific directory | `fygo-cli file ls /vol{v}/...` | Accepts only canonical paths |
| Search the current user's files | `fygo-cli file search <key>` | The CLI probes and derives the user's directory |
| Search an explicit path | `fygo-cli file search <key> /vol{v}/...` | Every path must be canonical |
| Search files shared with the current user | `fygo-cli file search-others <key>` | Finder search, not shared-directory listing |
| List directories, calculate path statistics or inspect access | `file ls-dir/calc/access <path>` | `calc` and `access` require concrete `/vol{v}/...` paths; `ls-dir` also allows no path |
| Inspect volume usage by category | `fygo-cli file usage <vol>` | `vol` is a storage-space ID; the CLI aggregates all streamed responses |
| Check for additional mount points within a directory | `fygo-cli file is-mount-point <path>` | `hasMount=false` means no additional mount points in this directory or descendants, not that its storage volume is unmounted |
| Inspect properties, size or a download URL | `file prop/size/download-url <path>` | Requires concrete `/vol{v}/...` paths |
| Inspect shared-directory metadata or lists | `file share info/list/list-others/admin-list/admin-list-others` | Shared directories and share links have different semantics |
| Inspect ACLs | `fygo-cli file acl get <path>` | Permissions are distinct from shared-directory state |
| Set ACLs or ownership | `file acl set` / `file chown` | Permission writes require `--yes` |
| Convert TrimACL or export its report | `file trimacl conv/report` | Permission-conversion writes require `--yes` |
| Create, rename, delete, copy or move | `file mkdir/rename/rm/cp/mv` | Writes normally require concrete `/vol{v}/...` paths and `--yes` |
| Compress or extract files | `file compress/extract` | Writes require `--yes` |
| Manage the current user's trash | `file trash list/search/restore/clear` | Restore and clear require `--yes` |
| Manage favorites | `file fav list/add/del` | `add` and `del` are writes and require explicit `--yes` |
| Manage team files and trash | `file team list/trash-list-trashbin/trash-list/trash-restore/trash-clear` | `trash-list` requires a concrete team-trash `path`; first use `trash-list-trashbin` if unknown. Restore and clear require `--yes`; clear requires the team volume number and directory name |
| Check a target before upload | `fygo-cli file check-upload /vol{v}/... <size> --yes` | Transfers no content but may allocate a temporary placeholder on the device |
| Upload one local file | `fygo-cli file upload /vol{v}/... <localFile> --yes` | The remote argument is a directory; the CLI appends the local filename |
| Download one or more files | `/multiple-download` HTTP flow | No named CLI command currently; this is the platform file-download service, not Download Center CGI |

If unsure between `ls`, `search`, `search-others` and `share`, first read `workflows/file-routing.md`.

## Common misunderstandings

- `file ls /` omits `path`, just like no argument; it does not send `/` directly to the backend.
- The current-user directory is derived from device responses and session data, not a fixed constant.
- `file share.*` manages shared directories, not share links.
- `file search-others` searches matching files shared with the current user; it does not list shared directories.
- `file cp` / `file mv` take a destination directory, not the final file path.

## High-risk operations

- Do not use aggregate roots, relative paths or ambiguous paths for writes.
- A user asking for a share link does not authorize a `file share.*` operation.
- For multi-path search, verify that every path is `/vol{v}/...`.
- When a task depends on the current-user directory, let the CLI probe it instead of guessing a payload.

## Endpoint index
- Core lists and metadata:
  - `file.ls`
  - `file.lsDir`
  - `file.calc`
  - `file.prop`
  - `file.access`
  - `file.usage`
- Mutations and transfers:
  - `file.mkdir`
  - `file.rename`
  - `file.rm`
  - `file.cp`
  - `file.mv`
  - `file.cancel`
- Ownership and ACLs:
  - `file.chown`
  - `file.getAcl`
  - `file.setAcl2`
  - `file.trimacl.conv`
  - `file.trimacl.conv.report`
  - `file.setAcl` (legacy ACL interface is not implemented)
- Upload, download and compression:
  - `file.checkUpload`
  - `file upload` CLI `/upload` flow; transport follows the current connection security policy
  - `/multiple-download` HTTP flow (no named CLI command currently)
  - `file.download`
  - `file.compress`
  - `file.extract`
  - `file.size`
- Search:
  - `appcgi.finder.fileSearch`
  - `appcgi.finder.searchOtherSharing`
  - `appcgi.finder.trashSearch`
  - `appcgi.finder.cancelSearch`
- Shared directories:
  - `file.share.info`
  - `file.share.list`
  - `file.share.listOthers`
  - `file.share.admin.list`
  - `file.share.admin.listOthers`
  - `file.share.add2`
  - `file.share.del2`
- Favorites and recent files:
  - `file.fav.list`, `file.fav.add`, `file.fav.del`
  - `file.recent.list`, `file.recent.add`, `file.recent.del`, `file.recent.clear`
- Application files and system partitions:
  - `appcgi.filestor.getAppDirList`
  - `appcgi.filestor.getSysPartInfo`
  - `appcgi.filestor.isMountPoint`
- Trash:
  - `file.trash.list`, `file.trash.clear`, `file.trash.restore`
- Team files:
  - `file.team.lsDir`
  - `file.team.trash.listTrashbin`
  - `file.team.trash.list`
  - `file.team.trash.clear`
  - `file.team.trash.restore`

## Endpoint details

### file.ls

#### Endpoint
`file.ls`

#### Purpose
List files in the backend's default view or a specified canonical Fygo path.

#### CLI mapping
```
fygo-cli file ls
fygo-cli file ls /
fygo-cli file ls /vol{v}/...
```

Path handling:
- No argument and `/` both normalize to omission of `path`.
- A non-root path must use `/vol{v}/...`; otherwise the CLI rejects it before sending a request.

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `file.ls` | `file.ls` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `path` | body | no | string | Concrete directory path | Omission selects the backend default view; an explicit path must be `/vol{v}/...` | `/vol3/1106/test3` |

#### Response
| Field | Always Present | Type | Meaning | Conditions / Notes | Example |
| --- | --- | --- | --- | --- | --- |
| `files` | no | array | File-entry block | May occur in several intermediate packets | `[{"name":"Photos","dir":1}]` |
| `result` | no | string | Terminal marker | `succ`/`fail`; may be omitted from intermediate packets | `succ` |
| `errno` | no | number | Error code | `errno: 0` means success | `65534` |
| `errmsg` | no | string | Error description | Present on failure | `path not found` |

#### Protocol Notes
- `file.ls` may return multiple packets: intermediate packets carry `files`; terminal packets contain `result`, `errno` or `errmsg`.
- Success is `result: succ` or `errno: 0`; `result: fail` or nonzero `errno` indicates failure.
- Requests are signed when a session secret is available; the format follows `_conventions.md`.

#### Field Semantics
- No argument and `/` omit `path` entirely; they do not set it to `/`.
- Device firmware determines the meaning of omitted `path`; some devices return the current-user directory rather than a separate volume list. Verify the actual target's behavior.
- `files[].dir` marks directories.
- The CLI exposes no pagination controls; it consumes the stream until a terminal packet.

#### Errors
- Invalid paths are rejected before sending, with guidance to use `/vol{v}/...`.
- Backend errors propagate through `errmsg`/`errno`.

### file.lsDir

#### Purpose
List file-manager directory entries or a specified directory. This endpoint may return `files` across multiple packets.

#### CLI mapping
```
fygo-cli file ls-dir
fygo-cli file ls-dir /vol{v}/...
```

#### Protocol Notes
- The CLI receives through the terminal packet and combines intermediate `files` into one JSON array.
- Terminal packets normally contain only metadata such as `result` and `uver`; they are not emitted as directory entries.

### file.usage

#### Purpose
Query capacity usage by category for a specified storage space.

#### CLI mapping
```
fygo-cli file usage <vol>
```

#### Protocol Notes
- Multiple packets are pushed; `category` identifies usage categories such as `user`, `team`, `app` and `docker`.
- The CLI outputs a JSON array, excluding terminal packets without `category`.
- A category may be pushed repeatedly: use the latest value, never sum repeated values. User/team categories are keyed by `category + id`; other categories by `category` alone.

### file.fav.add / file.fav.del

```bash
fygo-cli file fav add /vol{v}/... --yes
fygo-cli file fav del /vol{v}/... --yes
```

Both commands modify the current user's favorites and require explicit `--yes`. Without it, the CLI rejects the operation before reading a session or accessing the network.

### file.mkdir

#### Endpoint
`file.mkdir`

#### Purpose
Create a directory at a specified canonical Fygo path.

#### CLI mapping
```
fygo-cli file mkdir /vol{v}/... --yes
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `file.mkdir` | `file.mkdir` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `path` | body | yes | string | Target directory path | Must be `/vol{v}/...` | `/vol2/1106/new-dir` |

#### Response
| Field | Always Present | Type | Meaning | Conditions / Notes | Example |
| --- | --- | --- | --- | --- | --- |
| `result` | no | string | Terminal marker | `succ`/`fail` | `succ` |
| `errno` | no | number | Error code | Present on failure | `17` |
| `errmsg` | no | string | Error description | | `file exists` |

#### Field Semantics
- `path` is the complete target path, not a parent directory plus a separate name.
- Optional permission parameters are not currently supported.

### file.prop

#### Endpoint
`file.prop`

#### Purpose
Inspect properties of a specified file or directory.

#### CLI mapping
```
fygo-cli file prop /vol{v}/...
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `file.prop` | `file.prop` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `files` | body | yes | string[] | Paths to inspect | The CLI currently sends one path | `["/vol2/1106/a.txt"]` |

### file.size

#### Endpoint
`file.size`

#### Purpose
Inspect the size of one file or directory.

#### CLI mapping
```
fygo-cli file size /vol{v}/...
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `file.size` | `file.size` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `file` | body | yes | string | Path to calculate | One canonical path | `/vol2/1106/a.txt` |

### file.download

#### Endpoint
`file.download`

#### Purpose
Request a download URL for one or more files.

#### CLI mapping
```
fygo-cli file download-url /vol{v}/...
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `file.download` | `file.download` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `files` | body | yes | string[] | Download paths | The CLI currently sends one path | `["/vol2/1106/a.txt"]` |

### file.rename

#### Endpoint
`file.rename`

#### Purpose
Rename a specified file or directory.

#### CLI mapping
```
fygo-cli file rename /vol{v}/... <newName> --yes
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `file.rename` | `file.rename` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `path` | body | yes | string | Path to rename | Must be `/vol{v}/...` | `/vol2/1106/a.txt` |
| `newName` | body | yes | string | New name | A name component only; must not contain `/` | `b.txt` |

### file.checkUpload

#### Endpoint
`file.checkUpload`

#### Purpose
Check a complete target file path before uploading, returning hints for same-name conflicts, resume or the final filename.

#### CLI mapping
```
fygo-cli file check-upload /vol{v}/... <size> [--overwrite skip|replace|rename] --yes
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `file.checkUpload` | `file.checkUpload` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `path` | body | yes | string | Complete target file path | Must include a filename and use `/vol{v}/...` | `/vol2/1106/new.jpg` |
| `size` | body | yes | number | File size | Bytes | `12345` |
| `overwrite` | body | no | number | Same-name conflict policy | `0` skip, `1` replace, `2` rename | `2` |

#### Response
| Field | Always Present | Type | Meaning | Conditions / Notes | Example |
| --- | --- | --- | --- | --- | --- |
| `result` | no | string | Terminal marker | `succ`/`fail` | `succ` |
| `skip` | no | number | Skip hint | Returned by some versions | `1` |
| `uploadName` | no | string | HTTP upload/resume filename | Used in POST `/upload` header `Trim-Path`; it may differ from the final visible filename | `new.iso.~#2` |
| `from` | no | number | Resume offset | Returned by some versions | `1048576` |
| `completed` | no | number | Completed bytes | Returned by some versions | `1048576` |
| `errno` | no | number | Error code | Present on failure | `4386` |
| `errmsg` | no | string | Error description | | `no space left` |

#### Field Semantics
- `path` is the complete target file path, not the destination directory.
- `file check-upload` transfers no content, but the backend may allocate a temporary `.~#n` upload placeholder; it is therefore a write.
- `--overwrite` accepts `skip`, `replace`, `rename`, or the corresponding numeric values `0`, `1`, `2`.

### file upload CLI flow

#### Purpose
Upload one local file to a remote directory.

#### CLI mapping
```
fygo-cli file upload /vol{v}/... <localFile> [--overwrite skip|replace|rename] --yes
```

#### Flow
1. Normalize the remote directory to `/vol{v}/...`.
2. Append the local filename to form the complete remote target path.
3. Call `file.checkUpload`.
4. A nonzero numeric `skip` or `completed` value means no HTTP upload is needed; a missing field or zero does not skip transmission.
5. If `uploadName` is returned, HTTP `Trim-Path` uses the parent directory plus that name; the user-visible result remains the requested target path.
6. Upload multipart field `trim-upload-file` through `POST /ogh/ac/h/upload`. Transport follows the current HTTP/HTTPS connection and TLS policy: loopback defaults to HTTP, remote targets to HTTPS; remote plain HTTP requires explicit permission.
7. Files of 20 MiB or more cache the `uploadName` path for resume. Cache identity includes NAS endpoint, profile, signed-in user, remote target, file size, overwrite policy and local SHA-256; resume requires a complete match. Uploads to the same NAS endpoint and remote target serialize across processes. Cached resume retries on `328496` and falls back to a normal check on `4100`.

#### HTTP Headers
| Header | Required | Meaning |
| --- | --- | --- |
| `Trim-Path` | yes | URI-encoded HTTP upload path; may use backend `uploadName` |
| `Trim-From` | yes | Resume offset; `0` without resume |
| `Trim-Overwrite` | yes | `0` skip, `1` replace, `2` rename |
| `Trim-Mtim` | yes | Local file mtime in Unix seconds |
| `Authorization` | yes | OAuth `Bearer` authentication added by the proxy client; do not expose the token |
| `Trim-Token` | no | Legacy direct-connection field; not sent through the current HTTP proxy |
| `Trim-Sign` | no | Legacy direct-connection signature; not sent through the current HTTP proxy |

### multiple-download HTTP flow

#### Endpoint

```plaintext
POST /multiple-download
GET  /multiple-download?token=<download-token>
```

#### Purpose

Create a temporary download link for one or more Fygo files/directories. Single files can download directly; multiple targets are normally packaged by the platform first.

#### Request

| Field | Location | Required | Type | Meaning | Example |
| --- | --- | --- | --- | --- | --- |
| `accessToken` | POST body | yes | string | Current OAuth access token | `<access-token>` |
| `paths` | POST body | yes | string[] | Download target paths | `["/vol1/1000/a.txt"]` |
| `downloadLimit` | POST body | no | number | Download/packaging limit used by the current frontend | `2` |
| `noZipWhenSingleFile` | POST body | no | boolean | Avoid ZIP for a single file | `true` |
| `zipFileName` | POST body | no | string | Download or archive name | `a.txt and 1 other file` |
| `token` | GET query | yes | string | Temporary download token returned by POST | `<download-token>` |

#### Flow

1. Send `Bearer` authentication and POST `accessToken` plus `paths` to `/ogh/ac/h/multiple-download`; read temporary `token` from the successful response.
2. Send `Bearer` authentication when opening `GET /ogh/ac/h/multiple-download?token=<download-token>` to receive file or archive bytes.
3. Use `Range` for partial downloads, receiving HTTP 206 on success. A temporary token without `Bearer` authentication returns 401.
4. On HTTP 403, the response body identifies the first path lacking permission.

#### CLI support

- The Rust CLI currently has no named command for this flow. `download request` cannot call it because it is not an `appcgi.downloadcenter.*` CGI endpoint.
- The temporary GET link still requires `Bearer` authentication; it is not an anonymous share link.
- Access tokens and temporary download tokens are sensitive credentials; do not log or forward them.

### file.rm

#### Endpoint
`file.rm`

#### Purpose
Delete the file or directory at a specified canonical Fygo path.

#### CLI mapping
```
fygo-cli file rm /vol{v}/... --yes
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `file.rm` | `file.rm` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `files` | body | yes | string[] | Paths to delete | The CLI currently sends one path | `["/vol2/1106/old-dir"]` |

#### Field Semantics
- The backend uses `files: string[]`, not a single `path` field.
- The CLI currently exposes single-target deletion; the backend supports arrays.

### file.trash.list

#### Endpoint
`file.trash.list`

#### Purpose
List files in the current user's trash.

#### CLI mapping
```
fygo-cli file trash list
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `file.trash.list` | `file.trash.list` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |

### file.trash.restore

#### Endpoint
`file.trash.restore`

#### Purpose
Restore a specified file from trash.

#### CLI mapping
```
fygo-cli file trash restore /vol{v}/... --yes
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `file.trash.restore` | `file.trash.restore` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `files` | body | yes | string[] | Trash paths to restore | The CLI currently sends one path | `["/vol2/1106/a.txt"]` |
| `overwrite` | body | yes | number | Conflict policy | `2` keeps both by automatically renaming | `2` |

### file.trash.clear

#### Endpoint
`file.trash.clear`

#### Purpose
Clear the current user's trash.

#### CLI mapping
```
fygo-cli file trash clear --yes
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `file.trash.clear` | `file.trash.clear` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |

### file.cp

#### Endpoint
`file.cp`

#### Purpose
Copy a file or directory to a specified canonical Fygo destination directory.

#### CLI mapping
```
fygo-cli file cp <src> <destDir> --yes
```

`src` and `destDir` must both use `/vol{v}/...` form.

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `file.cp` | `file.cp` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `files` | body | yes | string[] | Source paths | The CLI currently sends one source path | `["/vol3/1106/source"]` |
| `pathTo` | body | yes | string | Destination directory path | Must be `/vol{v}/...` | `/vol4/1106/backup` |
| `overwrite` | body | yes | number | Conflict policy | CLI defaults to `2` (keep both/rename) | `2` |

#### Field Semantics
- The backend uses `files + pathTo`, not `src + dest`.
- `overwrite: 2` currently means keep both/rename.
- The CLI currently exposes a single source; the backend supports arrays.

### file.mv

#### Endpoint
`file.mv`

#### Purpose
Move a file or directory to a specified canonical Fygo destination directory.

#### CLI mapping
```
fygo-cli file mv <src> <destDir> --yes
```

`src` and `destDir` must both use `/vol{v}/...` form.

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `file.mv` | `file.mv` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `files` | body | yes | string[] | Source paths | The CLI currently sends one source path | `["/vol4/1106/source"]` |
| `pathTo` | body | yes | string | Destination directory path | Must be `/vol{v}/...` | `/vol2/1106/archive` |
| `overwrite` | body | yes | number | Conflict policy | CLI defaults to `2` (keep both/rename) | `2` |

#### Field Semantics
- Request structure and conflict policy match `file.cp`.

### appcgi.finder.fileSearch

#### Endpoint
`appcgi.finder.fileSearch`

#### Purpose
Use finder to search by keyword under one or more canonical Fygo paths.

#### CLI mapping
```
fygo-cli file search <key> [paths...]
```

Path handling:
- Every explicit path must be `/vol{v}/...`; the leading slash is removed when sent to finder.
- Without paths, the CLI first probes `file.ls` with omitted `path`, then derives the user's `vol{v}/{uid}` search root from returned entries' `v` field.
- Include `uid` when available in the session.

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value | `appcgi.finder.fileSearch` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69be...` |
| `key` | body | yes | string | Search keyword | Must not be empty | `.txt` |
| `path` | body | yes | string[] | Finder search roots | `vol{v}/...` form without a leading slash | `["vol2/1000"]` |
| `uid` | body | no | number | Current user ID | Included when available in the session | `1000` |

#### Response
| Field | Always Present | Type | Meaning | Conditions / Notes | Example |
| --- | --- | --- | --- | --- | --- |
| `data` | no | array | Search-result block | Streamed packets carry results in top-level `data` | `[{"path":"/vol2/1000/a.txt"}]` |
| `result` | no | string | Finder status | `doing` in progress; `succ` complete; `fail` failed | `doing` |
| `errno` | no | number | Error code | May terminate the request | `100000002` |
| `errmsg` | no | string | Error description | | `param invalid` |

#### Protocol Notes
- Finder search returns multiple packets. Intermediate packets may contain both `data` and `result: doing`.
- `result: doing` is not terminal. The CLI waits for a subsequent non-`doing` status.

### appcgi.finder.searchOtherSharing

#### Endpoint
`appcgi.finder.searchOtherSharing`

#### Purpose
Search files other users have shared with the current user.

#### CLI mapping
```
fygo-cli file search-others <key>
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value | `appcgi.finder.searchOtherSharing` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69be...` |
| `key` | body | yes | string | Search keyword | | `.txt` |

#### Protocol Notes
- Uses the same finder terminal semantics as `fileSearch`.
- Intermediate `result: doing` packets are not terminal.
- Streamed results use the `files` field.

### file.getAcl

#### Endpoint
`file.getAcl`

#### Purpose
Return ACL information for a specified canonical Fygo path, optionally including file properties.

#### CLI mapping
```
fygo-cli file acl get <path>
fygo-cli file acl get <path> --prop
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `file.getAcl` | `file.getAcl` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69be...` |
| `path` | body | yes | string | Target path | Must be `/vol{v}/...` | `/vol1/1000/docs` |
| `getProp` | body | no | boolean | Include properties | Sends `true` with `--prop` | `true` |

#### Field Semantics
- `permset` is the main raw ACL/shared-permission data and is preserved unchanged.
- `getProp` is passed through.

### file.share.info

#### Endpoint
`file.share.info`

#### Purpose
Return shared-directory metadata for a specified canonical Fygo path.

#### CLI mapping
```
fygo-cli file share info <path>
```

### file.share.list / file.share.admin.list

#### Endpoint
`file.share.list` and `file.share.admin.list`

#### Purpose
List shared directories: `list` uses the current user's view; `admin-list` uses the administrator's view.

#### CLI mapping
```
fygo-cli file share list [uid]
fygo-cli file share admin-list [uid]
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | `file.share.list` or `file.share.admin.list` | `file.share.list` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69be...` |
| `uid` | body | no | number | Source user/owner ID filter | Must be a positive integer | `1001` |

#### Protocol Notes
- Both endpoints stream multiple packets; intermediate packets contribute `share[]`.

### file.share.listOthers / file.share.admin.listOthers

#### Endpoint
`file.share.listOthers` and `file.share.admin.listOthers`

#### Purpose
List users who own visible shared directories.

#### CLI mapping
```
fygo-cli file share list-others
fygo-cli file share admin-list-others
```

### file.share.add2

#### Endpoint
`file.share.add2`

#### Purpose
Create a shared directory at the specified path, with explicit `permset` permissions.

#### CLI mapping
```
fygo-cli file share add <path> <shareName> --permset <json> [--sub] [--acl-mode <mode>] --yes
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `file.share.add2` | `file.share.add2` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69be...` |
| `path` | body | yes | string | Target path | Must be `/vol{v}/...` | `/vol1/1000/docs` |
| `shareName` | body | yes | string | Shared-directory display name | | `docs` |
| `permset` | body | yes | array | ACL/shared permissions | Must be a JSON array of objects | `[{"owner":32719,"inherit":7}]` |
| `sub` | body | no | boolean | Apply to descendants | Sent with `--sub` | `true` |
| `aclMode` | body | no | number | Explicit ACL mode | Sent with `--acl-mode` | `1` |

#### Field Semantics
- `permset` is passed through unchanged; raw data from `file acl get` can be reused directly.

### file.share.del2

#### Endpoint
`file.share.del2`

#### Purpose
Remove shared-directory status from the specified path.

#### CLI mapping
```
fygo-cli file share del <path> [--sub] [--acl-mode <mode>] --yes
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value `file.share.del2` | `file.share.del2` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69be...` |
| `path` | body | yes | string | Target path | Must be `/vol{v}/...` | `/vol1/1000/docs` |
| `sub` | body | no | boolean | Apply to descendants | Sent with `--sub` | `true` |
| `aclMode` | body | no | number | Explicit ACL mode | Sent with `--acl-mode` | `1` |

## Caveats
- Terminal packet behavior for long-running operations, such as large copies/moves, may vary with firmware.
- The full error-code taxonomy for ACLs and cross-volume transfers is not yet established.
- Exact conflict handling for `overwrite: 2` (keep both/rename) depends on the backend.
