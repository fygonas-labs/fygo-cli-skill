# Users and authentication reference

## OAuth login

fygo-cli establishes a proxy session using OAuth 2.0 Authorization Code + PKCE. It does not use NAS username/password login, 2FA login or a legacy token restoration chain. The CLI requests `trim.*` scopes for users, file reads/writes, downloads, app center, photos, system, logs, storage and Docker; see `oauth.md` for the complete list.

Agents use two-step authorization; read `oauth.md` for boundaries and failure handling:

```bash
fygo-cli --profile home --host <host> --port <port> login --authorize-only
fygo-cli --profile home --host <host> --port <port> login --code '<code>'
```

First share the CLI's authorization link. The user signs in and authorizes in the browser, then sends the one-time code back to the Agent. Only then execute the second command, in the same execution environment, profile and connection options. The user needs no terminal. Credentials and consent remain in the user's browser: do not fill passwords, approve access or read a code from the page. Accept only a code the user actively provides; do not repeat it or save additional copies. The authorization URL omits `redirect_uri`; do not add one. Successful token exchange and persistence clear the one-time pending state.

The session stores:

| Field | Meaning |
| --- | --- |
| `accessToken` | Bearer token for proxy business requests |
| `refreshToken` | Credential for refreshing the access token |
| `accessTokenExpiresAt` | Access-token expiration as Unix seconds |
| `oauthClientId` | OAuth client ID used by the CLI |
| `oauthDeviceId` | Locally persisted OAuth device ID |
| `oauthScope` | Authorized scopes |

Authenticated commands automatically refresh an access token approaching expiration. Explicit refresh is also available:

```bash
fygo-cli --profile home login --refresh
```

If the refresh response does not contain a new refresh token, the CLI keeps the existing one. Before refreshing, it checks that the current profile's session storage is writable. A failed preflight stops the command without sending refresh or silently switching to username/password login. Concurrent processes reuse a newly saved session instead of overwriting newer tokens.

`logout` clears only the current product/profile's local session; it does not imply server-side token revocation:

```bash
fygo-cli --profile home logout
```

## Current user and user lists

```bash
fygo-cli user info
fygo-cli user list
fygo-cli user list --mode group --group Users
fygo-cli user request user.list --json '{"limit":100,"offset":0}' --yes
```

| Command | Endpoint | Important output |
| --- | --- | --- |
| `user info` | `user.info` | `userInfo.user`, `uid`, `admin` |
| `user list` | `user.list` | User array, total count and pagination fields |
| `user list --mode group` | `user.listUG` | Users in the specified group |

For `user request`, the endpoint must start with `user.`. `--json` must be a JSON object without manually supplied `req` or `reqid`. The generic command cannot reliably infer whether an endpoint reads or writes from its name, so it requires confirmation by default. Automation must explicitly pass `--yes`. Dedicated read-only commands such as `user info` and `user list` do not require it.

## User mutations

```bash
fygo-cli user add <user> [--password <initial-password>] [--groups Users] [--set-admin] --yes
fygo-cli user mod <user> [--new-name <name>] [--password <new-password>] [--groups Users] --yes
fygo-cli user del <user> --yes
fygo-cli user change-password <user> --yes
fygo-cli user unfreeze <user> --yes
fygo-cli user set-admin <user> --admin true --yes
```

| Command | Endpoint | Main request fields |
| --- | --- | --- |
| `user add` | `user.add` | `user`, `password`, `groups`, optional contact information and administrator flag |
| `user mod` | `user.mod` | `user` and explicitly supplied changes |
| `user del` | `user.del` | `user` |
| `user change-password` | `user.changePassword` | `user`, new password entered interactively |
| `user unfreeze` | `user.unfreeze` | `user` |
| `user set-admin` | `user.setAdmin` | `user`, `admin` |

Here `--password` sets a new user's password or changes the target user's password; it is not a CLI login credential. Do not turn user-management passwords into login arguments. Keep passwords out of replies, logs and persistent command records; omit password options when interactive entry is available.

Deleting users, changing administrator privileges, changing passwords and unfreezing accounts are writes. Obtain explicit user authorization first.

## Groups

```bash
fygo-cli user group list
fygo-cli user group info <group>
fygo-cli user group users <group>
fygo-cli user group add <group> [--comment <text>] --yes
fygo-cli user group mod <group> [--new-name <name>] [--comment <text>] --yes
fygo-cli user group del <group> --yes
fygo-cli user group set-users <group> --users <user> --yes
fygo-cli user group add-users <group> --users <user> --yes
fygo-cli user group del-users <group> --users <user> --yes
```

| Command | Endpoint | Main request fields |
| --- | --- | --- |
| `group list` | `user.groupList` | None |
| `group info` | `user.groupInfo` | `group` |
| `group users` | `user.groupUsers` | `group` |
| `group add` | `user.groupAdd` | `group`, optional `comment` |
| `group mod` | `user.groupMod` | `group`, optional new name and comment |
| `group del` | `user.groupDel` | `group` |
| `group set-users` | `user.groupSetUsers` | `group`, `users` |
| `group add-users` | `user.groupAddUsers` | `group`, `users` |
| `group del-users` | `user.groupDelUsers` | `group`, `users` |

`--users` and `--groups` accept repeated options or comma-separated values. Empty lists, control characters and clearly invalid names are rejected before any request is sent.

## Signed-in devices

```bash
fygo-cli user login-devices
fygo-cli user request user.listLoginDevice --yes
```

Signed-in-device lists are user-management data, not part of CLI OAuth PKCE login. Do not use these lists to attempt the old trusted-device/2FA login flow.

## Errors

- `saved proxy session is required`: complete OAuth login first.
- Missing or expired OAuth pending state: follow `oauth.md`, run `login --authorize-only` again, and wait for a new user-supplied code before exchanging. Do not retry the old code.
- Refresh failure or missing refresh token: restart OAuth login without falling back to username/password.
- Final session write failure after refresh: do not blindly reuse the old refresh token; it may have rotated. Sign in again.
- Permission errors: use `user info` to inspect `admin`; do not automatically elevate privileges.
- Unconfirmed write: explain the target and effect, obtain authorization, then pass `--yes`.
- Business error codes: report their actual semantics; HTTP status alone does not establish success.
