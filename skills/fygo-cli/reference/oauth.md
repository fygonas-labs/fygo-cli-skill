# OAuth reference

## Fixed parameters

- `client_id`: `YJNMPJUGA9`
- `scope`: `trim.user.all trim.file.read trim.file.write trim.download.all trim.appcenter.all trim.photo.all trim.system.all trim.log.all trim.storage.all trim.docker.all`; product-specific scopes are described in the applicable references.
- `code_challenge_method`: `S256`
- `device_model`: `CLI`; see the device naming rules below.
- Authorization code lifetime: 5 minutes.

## Sign in

```bash
fygo-cli --profile home --host <host> --port <port> login --authorize-only --agent-name '<agent>'
fygo-cli --profile home --host <host> --port <port> login --code '<code>'
```

The Agent uses two separate calls; the user does not need a terminal:

1. Confirm the user's NAS target and profile. Reuse a valid session; ask for an unknown new target.
2. Provide the Agent identity as described below, run `login --authorize-only --agent-name '<agent>'` and share the returned `authorization_url` unchanged as a link. `authorization_pending=true` means authorization is pending, not that login succeeded. This call does not wait for terminal input.
3. The user opens the link, enters credentials and approves access in their browser, then sends the displayed one-time code to the Agent.
4. Only after receiving that explicitly provided code, run `login --code '<code>'` on the same execution host, as the same system user, with the same configuration directory, profile and connection options. Pass code as a separate argument; quote it correctly in a shell, never interpolate it as shell syntax.
5. Confirm `token_persisted=true` and `refresh_token_persisted=true`, then make a read-only request such as `system info` using that profile. Report success and continue the task only after verification.

### Authorization device name

When starting authorization, pass only the current Agent tool's short name with `--agent-name`: use `codex` for Codex or `claude` for Claude. Names must contain 1–32 ASCII letters, digits or hyphens, starting with a letter or digit; the CLI converts them to lowercase. If the Agent identity cannot be determined, ask the user for a name.

On macOS, Linux and Windows, the CLI reads the execution environment's hostname through native system APIs, preserves its case and builds `{hostname}-{agent}`, for example `macbook-codex`. The Agent does not need to read or guess the hostname. Lookup does not depend on shell commands or hostname environment variables. In SSH sessions or containers, the hostname belongs to that execution environment. The CLI keeps at most the first 20 hostname characters (Unicode scalar values, not bytes), preserving the full Agent suffix. For example, `abcdefghijklmnopqrstuv` with `codex` becomes `abcdefghijklmnopqrst-codex`.

The name appears in NAS authorization management and is saved with the profile's pending authorization. `login --code` reuses it automatically; omit naming options on that call. `--agent-name` conflicts with `--device-name`, `--code` and `--refresh`. If native lookup fails, the user can specify the full name with `--device-name` (nonblank, no control characters, at most 128 UTF-8 input bytes; surrounding whitespace is trimmed). Manual login without either naming option defaults to `fygo-cli`. The name applies to this authorization; the CLI does not proactively rename historical authorizations.

### Authorization handling

Do not enter passwords, click consent, or read a code from the browser on the user's behalf. Accept only the code the user actively sends. Chat and tool records, and potentially local process arguments, can retain it; do not promise no retention. Do not repeat it in replies, additionally write it to files or logs, enable shell tracing, or forward it. Access/refresh tokens and PKCE verifiers remain in CLI storage, outside chat.

If device access-code protection is enabled, the CLI also stores the token response's `x_access_code` in the same controlled profile session. Do not accept or repeat this value. It is already encoded and is used unchanged in request headers. Protected HTTP requests automatically include `X-Access-Code` and `X-Access-Source: app`, without `x-device-id`. The user handles the browser's access-code page.

Authorization URLs include PKCE challenge, client, scope and device parameters, with no `redirect_uri`; do not rewrite them. Each profile keeps one pending verifier. Do not generate another link while awaiting a code: it replaces that pending state. Explicitly preserve host, port, scheme and TLS options in both calls; a new unsaved session cannot supply target defaults. Remote plain HTTP requires `--scheme http --allow-insecure-http` in both calls.

For supported Connect device subdomains, the CLI adds a `mode=relay` Cookie to target OAuth and proxy HTTP requests; other targets do not receive it. This affects CLI requests only, not the browser authorization link. Support for a particular device or Connect route must be verified rather than inferred.

A code can be exchanged once and expires after 5 minutes. Stop and explain an expired, missing or target-mismatched pending state, or a failed exchange, then restart authorization. Do not repeatedly retry an old code. If tokens were saved but user-summary lookup or probing failed, first try a read-only request with the saved session; do not exchange the consumed code again. Parallel authorizations use distinct profiles.

Use the same package-provided launcher for both calls; no interactive TTY is needed. Reauthorize in the actual execution environment after changing host, SSH shell or container; local sessions are not automatically shared. macOS Keychain access may require the user's approval of a system prompt.

If the user chooses manual terminal authorization, plain `login` prompts for the code. `--no-open` only disables automatic browser opening; it does not change that manual flow. The Agent flow does not require a user terminal. `trim.basic` is not in the current requestable scope list and is not explicitly requested. Old `file photo` scopes or a session missing a module's current scopes require authorization again; refresh cannot expand the originally authorized scopes.

## Refresh

```bash
fygo-cli --profile home login --refresh
```

Before a command, an access token expiring within 60 seconds refreshes automatically. HTTP 401 from proxy `/ogh/ac/w` or native `/ogh/ac/h/*` triggers a profile-locked refresh and one retry; a second 401 stops. A new refresh token replaces the old one; otherwise the existing token is kept. Before refreshing, the CLI writes the current session back to the selected profile to verify actual storage is writable. Failed preflight sends no refresh request and does not consume a potentially rotating refresh token.

The `ask-file` storage choice prompts once and applies to this refresh commit. After acquiring the lock, concurrent processes reread the session and reuse a newer token instead of overwriting it. Refresh preserves the device access code and sends its saved headers; it does not rotate that code.

An `x-trim-safe-code-challenge` response indicates device access-code interception and does not trigger OAuth 401 refresh. HTTP 429 means verification is rate-limited: wait before retrying. HTTP 503 indicates unavailable verification; 403 may indicate policy rejection or temporary blocking. A gateway challenge page with status 200 is ambiguous; the CLI asks for a new login to obtain a valid code.

Storage may still fail after the request. If refresh succeeds but final persistence fails, do not repeatedly reuse the old refresh token: the server may have rotated it. Restart OAuth authorization.

## Sessions and logout

Sessions hold access/refresh tokens, optional device access code, expiry, OAuth client/device/scopes and a user summary. `logout` removes the current profile's session and pending files; it does not promise remote token revocation.

Reauthentication uses the flow above, never CLI password/2FA or old-token recovery. Only user-provided codes go into `login --code`; verifiers and access/refresh tokens must not become command arguments, chat output or logs.

## Upgrading

This product uses only `FYGO_CLI_*` configuration variables and independent session storage. Reauthorize after upgrading; old credentials are not migrated, read or deleted. A same-named profile in another product does not share state.
