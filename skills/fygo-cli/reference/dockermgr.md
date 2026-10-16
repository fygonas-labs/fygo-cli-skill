# dockermgr module

## Overview
Docker management covers images, containers, networks, Compose projects, and Docker daemon lifecycle operations.

## Conventions
- Endpoints are grouped into images, containers, networks, Compose, and system operations.
- Distinguish list/query endpoints from long-running mutation endpoints.
- Namespace convention: historical references use `appcgi.dockermgr.*`; some implementations use `dockermgr.*` (such as `dockermgr.containerList`). Prefer `appcgi.dockermgr.*` for automation.

## Task routing

First determine whether the task observes or changes state:

| User intent | Preferred command | Notes |
| --- | --- | --- |
| Inspect overall Docker state | `fygo-cli docker stats` | Use as the initial read-only Docker probe |
| Inspect images | `fygo-cli docker image ls` / `inspect` | Inspect an individual image only when details are needed |
| Pull an image | `fygo-cli docker image pull <imageRef> --yes` | Long-running write; verify state again afterward |
| Inspect containers | `fygo-cli docker container ls [--all]` / `inspect` / `stats` / `top` | `ls` lists running containers by default; stopped or created containers require `--all` |
| Start, stop, restart, kill, or remove containers | `start/stop/restart/kill/rm` | Inspect or list before changing state |
| Inspect Compose projects | `fygo-cli docker compose ls` | Prefer this read-only entry point |

## Common pitfalls

- `docker stats` returns aggregate statistics, not statistics for one container.
- `docker container ls` excludes stopped/created containers by default; use `docker container ls --all` or `inspect` by ID to determine whether a container still exists.
- A successful list response without `data` confirms request success only; it does not prove there are zero resources. The CLI preserves the original success envelope instead of turning an omitted backend field into an empty list.
- Operations such as `image pull`, `container stop`, and `container restart` may take substantially longer than ordinary reads.
- `image pull` and `container create/update/start/stop/restart/kill` require explicit `--yes`. Removal may prompt interactively, but Agent and noninteractive flows must also pass `--yes`. `--force` and `--yes` have different meanings.
- `container update` may return a new ID. The CLI inspects that ID and confirms resource changes and preserved network configuration before reporting success. If older firmware omits the ID, it performs the same check against the original ID. Use the command output ID for subsequent operations.
- Image references and container IDs are different identifiers; do not interchange them.

## High-risk operations

- Before a change, run a read-only command to confirm the target exists and its state matches expectations.
- After a long-running operation, confirm the result with a read-only command; the start message alone is insufficient.
- Do not bundle removal with unrelated image or container cleanup.
- For an unclear request such as "sort out Docker", first clarify whether it concerns images, containers, networks, or Compose.

## Endpoint index
- Images:
  - `appcgi.dockermgr.imageList`
  - `appcgi.dockermgr.imagePull`
  - `appcgi.dockermgr.imageInspect`
  - `appcgi.dockermgr.imageRemove`
  - `appcgi.dockermgr.imageDownloadList` (via `docker request`)
  - `appcgi.dockermgr.imageLoad` (via `docker request`)
  - `appcgi.dockermgr.imageSave` (via `docker request`)
  - `appcgi.dockermgr.imageUpgrade` (via `docker request`)
  - `appcgi.dockermgr.imageCancel` (via `docker request`)
- Containers:
  - `appcgi.dockermgr.containerList`
  - `appcgi.dockermgr.containerCreate`
  - `appcgi.dockermgr.containerInspect`
  - `appcgi.dockermgr.containerTop`
  - `appcgi.dockermgr.containerStats`
  - `appcgi.dockermgr.containerStart`
  - `appcgi.dockermgr.containerStop`
  - `appcgi.dockermgr.containerRestart`
  - `appcgi.dockermgr.containerKill`
  - `appcgi.dockermgr.containerRemove`
  - `appcgi.dockermgr.containerModify`
- Networks:
  - `appcgi.dockermgr.networkCreate`
  - `appcgi.dockermgr.networkList`
  - `appcgi.dockermgr.networkRemove`
  - `appcgi.dockermgr.networkConnect`
  - `appcgi.dockermgr.networkDisconnect`
- System and Compose:
  - `appcgi.dockermgr.stats`
  - `appcgi.dockermgr.composeList`
  - `appcgi.dockermgr.systemStart` (via `docker request`)
  - `appcgi.dockermgr.systemStop` (via `docker request`)
  - `appcgi.dockermgr.systemRestart` (via `docker request`)
  - `appcgi.dockermgr.composeContainers` (via `docker request`)
  - `appcgi.dockermgr.composeCreate` (via `docker request`)
  - `appcgi.dockermgr.composeStart` (via `docker request`)
  - `appcgi.dockermgr.composeStop` (via `docker request`)

Use the following entry point for `appcgi.dockermgr.*` endpoints without a dedicated subcommand:

```bash
fygo-cli docker request appcgi.dockermgr.<name> --json '<object>' --yes
```

`--json` must be a JSON object without `req` or `reqid`. Generic requests require explicit `--yes`; without it the CLI rejects before reading a session or accessing the network, without an interactive prompt. Docker system, network, Compose, and image import/export operations can affect running services; perform a read-only probe first.

## Endpoint details

### appcgi.dockermgr.containerList

#### Endpoint
`appcgi.dockermgr.containerList`

#### Purpose
List Docker containers and their runtime summaries.

#### CLI mapping
```
fygo-cli docker container ls [--all]
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value | `appcgi.dockermgr.containerList` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `all` | body | no | boolean | Include stopped and created containers | Send `true` only with `--all`; otherwise list running containers only | `true` |

#### Response
| Field | Always Present | Type | Meaning | Conditions / Notes | Example |
| --- | --- | --- | --- | --- | --- |
| `data` | no | array / object | Container list | Format may vary by Docker version | `[{"name":"nginx","state":"running"}]` |
| `result` | no | string | Terminal marker | `succ`/`fail` | `succ` |
| `errno` | no | number | Error code | Present on failure | `65534` |
| `errmsg` | no | string | Error description | | `docker not running` |

#### Protocol Notes
- Requests are signed when a session secret is available.
- Historical name alias: `dockermgr.containerList`.
- To check whether a container still exists after update/stop, use `--all` or `container inspect <id>` rather than relying on the default list.

### appcgi.dockermgr.stats

#### Endpoint
`appcgi.dockermgr.stats`

#### Purpose
Return aggregate Docker statistics.

#### CLI mapping
```
fygo-cli docker stats
```

#### Response
| Field | Always Present | Type | Meaning | Conditions / Notes | Example |
| --- | --- | --- | --- | --- | --- |
| `data` | no | object | Aggregate statistics | Includes utilization metrics | `{"cpu":12.5,"mem":47.1}` |
| `result` | no | string | Terminal marker | `succ`/`fail` | `succ` |

#### Field Semantics
- Aggregate summary endpoint; use `containerStats` for individual container metrics.

### appcgi.dockermgr.image.* family

#### Endpoint
`appcgi.dockermgr.imageList`, `imagePull`, `imageInspect`, `imageRemove`

#### Purpose
Manage Docker images (list/pull/inspect/remove).

#### CLI mapping
```
fygo-cli docker image ls
fygo-cli docker image pull <imageRef> --yes
fygo-cli docker image inspect <imageRef>
fygo-cli docker image rm <imageRef> [--force] [--yes]
```

#### Request
| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Select the operation | `appcgi.dockermgr.imageList` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `fromImage` | body | conditional | string | Image name (imagePull) | Excludes the tag | `nginx` |
| `tag` | body | conditional | string | Image tag (imagePull) | Defaults to `latest` when omitted | `latest` |
| `imageId` | body | conditional | string | Image ID or reference (inspect/rm) | A backend ID or a reference such as `nginx:latest` | `nginx:latest` |
| `force` | body | no | boolean | Force removal (imageRemove) | Sent only with `--force` | `true` |

#### Field Semantics
- `imagePull` splits `<imageRef>` into `fromImage` and `tag`, preserving registry ports; for example, `registry:5000/ns/app:1.0` becomes `fromImage="registry:5000/ns/app"`, `tag="1.0"`.
- `imageRemove` requires confirmation by default; `--yes` bypasses the prompt.
- Pull/upgrade can be long-running operations.

### appcgi.dockermgr.container.* mutation/details family

#### Endpoint
Container creation, lifecycle, and observation endpoints

#### Purpose
Create, start/stop/restart/kill/remove, and inspect containers; list processes and statistics.

#### CLI mapping
```
fygo-cli docker container create --image <imageRef> [--name <name>] [--start] [--restart] [--memory <mb>] [--cpu <shares>] [--env <key=value>] [--cmd <arg>] [--port <host:container[/proto]>] [--mount <source:target[:ro|rw]>] --yes
fygo-cli docker container update <containerId> [--restart] [--memory <mb>] [--cpu <shares>] --yes
fygo-cli docker container inspect <containerId>
fygo-cli docker container top <containerId>
fygo-cli docker container stats <containerId>
fygo-cli docker container start <containerId> --yes
fygo-cli docker container stop <containerId> --yes
fygo-cli docker container restart <containerId> --yes
fygo-cli docker container kill <containerId> --yes
fygo-cli docker container rm <containerId> [--force] [--yes]
```

#### Field Semantics

**containerCreate:**
- `--image` is required and maps to the image reference in the request.
- `--name` is an optional container name.
- `--start` maps to `startWhenCreated: true`.
- `--restart` maps to `restart: true`.
- `--memory <mb>` converts MB to bytes (for example, `128` to `134217728`).
- `--cpu <shares>` forwards positive integer CPU shares.
- Repeatable `--env <key=value>` maps to `env[]`.
- Repeatable `--cmd <arg>` maps to `cmd[]`.
- Repeatable `--port <host:container[/proto]>` maps to `port[]`; the protocol defaults to `tcp`.
- Repeatable `--mount <source:target[:ro|rw]>` maps to `mount[]`; permissions default to `rw`.
- Defaults: `restart=false`, `privileged=false`, `net=['bridge']`, `cpu=0`, `memory=0`.
- `create/update/start/stop/restart/kill` require explicit `--yes`; `rm` may prompt interactively, but Agent and noninteractive flows should pass `--yes`.

**containerModify:** The CLI reads the complete container configuration and merges resource changes before submission. If the response includes `rsp.id`, it inspects that ID and checks `name/image/restart/privileged/gpuEnabled/cpu/memory/env/cmd/capAdd/capDrop/link/mount/port/networks`. It reports success only when all fields match the submitted configuration; collection fields are compared without ordering, and network names and static addresses are checked. If older firmware omits `rsp.id`, the original ID receives the same verification.

**A successful containerCreate response** may use a top-level `rsp` object (Docker-style capitalized keys, such as `{"Id":"abcd","Warnings":[]}`) rather than nesting it inside `data.rsp`.

**containerInspect/top/stats/start/stop/restart/kill:** Identify the container with `containerId`.

`container top` requires a running target. The CLI inspects it first; if `State.Running=false`, it returns a clear local message without sending a top request that would produce a device errno.

**containerRemove:** Identify the container with `containerId`; `force` is sent only with `--force`. Confirmation is required by default; `--yes` bypasses it.

#### Errors
- Common errors: missing container, invalid state transition, or insufficient permissions.

### appcgi.dockermgr.composeList

#### Endpoint
`appcgi.dockermgr.composeList`

#### Purpose
List Docker Compose projects.

#### CLI mapping
```
fygo-cli docker compose ls
```

## Notes
- Image pull/upgrade and Compose operations are long-running and may involve progress/cancellation patterns.
- Starting/stopping/restarting the Docker daemon is a high-impact operation that may affect running containers.
- Detailed request/response schemas for some endpoints are not fully standardized.
