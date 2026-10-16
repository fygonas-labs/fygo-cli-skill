# Power reference

The power module reboots or powers off a NAS. Both commands interrupt device services and CLI connections. Confirm the target address, current account permissions and service impact before execution.

## Commands

| Command | Endpoint | Effect |
| --- | --- | --- |
| `fygo-cli power reboot --yes` | `power.reboot` | Request a NAS reboot |
| `fygo-cli power poweroff --yes` | `power.poweroff` | Request NAS power-off |

## Confirmation

Interactive confirmation is required by default. Pass `--yes` for automation only after the target and operation have been authorized.

```bash
fygo-cli --host <host> --port <port> power reboot --yes
fygo-cli --host <host> --port <port> power poweroff --yes
```

## Session requirements

A saved authenticated session is required. The CLI authenticates using that session before sending the power request.

## Output

After the request is sent successfully and accepted by the backend, the command prints a short success message:

```text
Reboot request accepted
Poweroff request accepted
```

Acceptance normally means the reboot or power-off has been scheduled. It does not mean reboot has finished or the device is fully powered off.
