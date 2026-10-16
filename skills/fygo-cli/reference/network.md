# Network reference

## Overview

The network module manages device network service switches. The CLI currently exposes the SSH switch.

## Endpoint index

- Implemented:
  - `appcgi.network.ssh.switch` (enable or disable SSH)

## appcgi.network.ssh.switch

### Endpoint

`appcgi.network.ssh.switch`

### Purpose

Enable or disable SSH on the device.

### CLI mapping

```bash
fygo-cli network ssh switch --enable [--yes]
fygo-cli network ssh switch --disable [--yes]
```

### Request

| Field | Location | Required | Type | Meaning | Constraints / Notes | Example |
| --- | --- | --- | --- | --- | --- | --- |
| `req` | body | yes | string | Endpoint selector | Fixed value | `appcgi.network.ssh.switch` |
| `reqid` | body | yes | string | Request correlation ID | Generated per request | `69ba...` |
| `enable` | body | yes | boolean | Whether to enable SSH | `true` enables; `false` disables | `true` |

### Notes

- Specify exactly one of `--enable` and `--disable`.
- Enabling SSH is security-sensitive and requires confirmation by default; use `--yes` only after explicit user authorization.
- The command requires a saved authenticated session and follows the authentication chain.
