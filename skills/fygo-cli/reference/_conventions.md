# API conventions

This directory describes the Fygo API modules supported by fygo-cli.

## Document structure

Each module includes an overview, module-wide conventions, endpoint index, endpoint details and usage limitations.

## Endpoint details

Each endpoint lists its full name (for example `appcgi.file.ls`), purpose and CLI mapping. Unimplemented mappings are explicitly marked as unavailable in fygo-cli.

### Request

| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |

Locations are `body` (top-level JSON), `query` (URL parameters), `path` (URL variables), `header` (protocol headers), or `wrapped body` (nested fields such as `data.body.*`).

### Response

| Field | Always Present | Type | Meaning | Conditions / Notes | Example |
| --- | --- | --- | --- | --- | --- |

### Protocol notes

Document authentication requirements (token, session, signature), terminal packets (`result`, streaming responses), and special envelopes such as `data.body`.
Where a session contains `secret` and an endpoint is not in the passthrough set, signing uses HMAC-SHA256. The payload is `<base64-hmac-signature><json-body>`, using the base64-decoded session `secret` as key. `si` is injected into JSON before signing when available.

### Field semantics and errors

Describe field meanings, constraints and common error codes and messages. Device-specific limits must be explicit.

## Paths

Fygo file paths use canonical `/vol{v}/...` form, for example `/vol1/downloads` or `/vol3/1000/media`.

## Common response envelope

Most endpoints include `result` (`succ` or `fail`) as a terminal marker, and `errno` (numeric error code) plus `errmsg` (description) on failure.
