## First use and cache

The package carries a lightweight launcher bound to a fixed version. The first `fygo-cli --version`, `--help`, or business command downloads only the CLI matching the execution host, verifies its bundled SHA-256, and runs it. Download messages go to stderr and do not pollute business JSON. Installing Skill documentation does not download a runtime; first use also works when npm lifecycle scripts are disabled.

First use needs network access to an available download source. Once prepared, the same product, version, and platform run from the current user's system cache without contacting the download site. Cache directories are separated by command, version, and platform. No administrator privileges or PATH changes are required, and existing product login state is preserved.

Explicitly updating the npm package or reinstalling the Skill selects a new version. Existing entries retain their own version; the launcher does not automatically upgrade or remove other versions. Retry the original command after a download failure. Failed checks never execute the downloaded file. If the launcher reports a damaged cache, confirm that initialization has stopped, remove the reported cache directory, and retry. Only remove a leftover lock after confirming its initializer has stopped.

### Mirror configuration

Downloads try GitHub Release, your configured compatible npm binary mirror, and public npmmirror in that order. The launcher prefers the `npm_config_registry` environment variable. When it is unset and npm is available, the launcher reads the effective value from `npm config get registry`. A cache hit does not read npm configuration or contact a download site.

Use an HTTPS base URL without credentials, query parameters, whitespace, or backslashes. A corporate path prefix is preserved. The default npmjs registry is skipped because it does not provide the binary interface. Mirror paths use `/-/binary/fygo-cli/v<version>/<asset-name>`. An ordinary npm registry may not implement this interface; a failed request proceeds to the next source. Set the environment variable to an empty string to skip a custom mirror.

Every source supplies the same fixed version and is checked against the bundled digest. Network and HTTP failures permit fallback. A completed response with the wrong size or checksum stops initialization immediately without trying another source. Retry the original command after all sources fail. Public mirror availability depends on product onboarding and version synchronization; client fallback support does not prove that onboarding is complete.
