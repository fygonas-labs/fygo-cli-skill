# Photos reference

## Authentication

Photos uses the current profile's system access token to call native HTTP APIs through `/ogh/ac/h/*uri`. The current command target must match the target stored in that profile's session. Without a valid session, complete OAuth `login` first. Photos needs no separate application login or signature.

## Command index

| Command | Output |
| --- | --- |
| `photos folders` | Watched folders, scan status and photo/video counts |
| `photos search <keyword>` | Conditional-search matches, timeline and total |
| `photos info <id>` | File, capture, location, tag and thumbnail fields |
| `photos preview <id>` | Absolute preview URL at the requested size |
| `photos request <method> <path>` | Raw HTTP and business response from an unwrapped endpoint |

## Response rules

Named commands validate HTTP status and the business envelope: non-2xx HTTP, `code != 0` or missing `data` causes failure. Named search and detail commands also convert relative `*Url` fields to absolute URLs.

`photos request` accesses raw endpoints without named-command business-envelope validation. Its output shape is:

```json
{
  "httpStatus": 200,
  "response": { "code": 0, "msg": "", "data": {} }
}
```

The Agent must confirm both `httpStatus` in `200..299` and `response.code == 0`; CLI exit status alone does not establish success for a generic request.

## Watched folders

```bash
fygo-cli photos folders
```

Requests `GET /p/api/v1/photo/folder/list` with no business parameters. Main `list[]` fields:

| Field | Type | Meaning |
| --- | --- | --- |
| `folderId` | integer | Photos watched-folder ID |
| `folderPath` | string | Fygo directory path |
| `photoCount` | integer | Photo count in the current index |
| `videoCount` | integer | Video count in the current index |
| `status` | integer | `0` scanning, `1` complete, `2` directory error, `4` unknown |
| `disable` | integer | Whether the directory may be removed from the scan scope |
| `hasWriteAccess.hasWriteAccess` | boolean | Whether the current user can write |
| `hasWriteAccess.quotaCurr` | integer | Current quota usage |
| `hasWriteAccess.quotaMax` | integer | Maximum quota |
| `isDefault` | boolean | Whether this is the default watched folder |

Counts may change while scanning. Do not present counts as final when the directory has an error. These counts cover the Photos index, not every image in the filesystem.

## Conditional search

```bash
fygo-cli photos search IMG --limit 20
fygo-cli photos search Beijing --filter file_type=photo
fygo-cli photos search Beijing --exclude-filter not_geo=1
```

Requests `POST /p/api/v2/search/results`. Request fields:

| Field | Required | Type | Meaning |
| --- | --- | --- | --- |
| `keyword` | yes | string | Conditional-search keyword; empty strings are rejected by the CLI |
| `limit` | yes | integer | Returned sample limit; CLI accepts `1..200`, default `20` |
| `filters` | yes | array | Positive `{filterName, filterValue}` conditions |
| `antiFilters` | no | array | Exclusion `{filterName, filterValue}` conditions |

`--filter` and `--exclude-filter` take repeatable `name=value` arguments. Common filter names include `file_type`, `is_collect`, `person`, `photo_tag` and `file_dir`. Common `file_type` values include `photo`, `video`, `live_photo`, `gif`, `raw`, `360`, `panorama` and `screenshot`. Other values depend on the device's Photos version and indexed content.

Main response fields:

| Field | Type | Meaning |
| --- | --- | --- |
| `list[]` | array | Photo/video samples returned by this call |
| `timeline[]` | array | `itemCount` grouped by `year`, `month` and `day` |
| `total` | integer | All matches reported by the server, not the length of `list` |
| `list[].id` | integer | Photo ID used by detail and preview commands |
| `list[].category` | string | `photo` or `video` |
| `list[].fileName` | string | Filename |
| `list[].filePath` | string | Fygo file path |
| `list[].photoUUID` | string | UUID used in photo stream URLs |
| `list[].additional.thumbnail.*Url` | string | Size-specific preview fields converted to absolute URLs |

## Photo detail

```bash
fygo-cli photos info <photoId>
```

Requests `GET /p/api/v1/gallery/getOne?id=<photoId>`. In addition to search-entry fields, common detail fields include `fileSize`, `width`, `height`, `description`, `photoDateTime`, `timeZoneOffset`, `make`, `model`, `fumber`, `exposureTime`, `isoSpeedRatings`, `focalLength`, `latitude`, `longitude`, `ownerId`, `ownerName`, `isCollect`, `isLive` and tags.

## Preview

```bash
fygo-cli photos preview <photoId> --size m
fygo-cli photos preview <photoId> --size o --open
```

Preview first reads `gallery/getOne`, then selects a URL from `additional.thumbnail`. Supported sizes are `xxs`, `xs`, `s`, `m` and `o`; `o` means original. Videos prefer `videoUrl`, falling back to `mUrl` if absent.

Output fields include `id`, `fileName`, `category`, `size`, `previewUrl`, `originalUrl`, `videoUrl` and `requiresAuthenticatedBrowserSession`. URLs are absolute, but the browser usually still needs a signed-in Fygo cookie; do not describe them as public unauthenticated links.

## AI semantic search

Check the model before running a natural-language query:

```bash
fygo-cli photos request post /api/v1/magic-search/ready --yes
fygo-cli photos request post /api/v1/magic-search/do \
  --json '{"keyword":"sunset at the beach","antiFilters":[]}' --yes
```

| Endpoint | Request | Response |
| --- | --- | --- |
| `POST /p/api/v1/magic-search/ready` | No body | `code == 0` indicates that the current model is available |
| `POST /p/api/v1/magic-search/do` | Required `keyword`; optional `antiFilters` | `data.list[]` contains this call's AI matches |

In generic-request output, the AI list is at `response.data.list`. This endpoint has no stable `total`, so `list` length means only the number returned by this call, not an exact library-wide total. For previews, select `id` from AI results and run `photos preview <id> --size m`; relative URLs from raw AI responses must not be treated directly as accessible links.

Business code `4044` means the current model does not support the query language: switch models first. Report other nonzero business codes unchanged and stop. Do not fall back to filename search and continue claiming AI results.

## Generic request

```bash
fygo-cli photos request get /api/v1/server/sys_info
fygo-cli photos request get /api/v1/gallery/getOne --query id=123
fygo-cli photos request post /api/v2/search/results \
  --json '{"filters":[],"keyword":"IMG","limit":5}' --yes
```

Paths must begin with `/api/...` or `/p/api/...`, without an origin, query string or fragment. Query parameters use repeatable `--query key=value`; request bodies must be JSON objects or arrays. Non-GET requests require interactive confirmation or explicit `--yes`.

See [workflows/photos-routing.md](workflows/photos-routing.md) for statistics, AI search and sampled-preview workflows.
