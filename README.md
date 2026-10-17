# Fygo CLI Skill

Usage guidance and public integration content for Fygo. The runtime CLI and Skill come from the same npm package version.

## Install the Skill

```bash
npx skills add fygonas-labs/fygo-cli-skill --skill fygo-cli -g
```

## Standalone Skill launcher

Resolve `scripts/fygo-cli` (macOS/Linux) or `scripts/fygo-cli.cmd` (Windows) from the loaded Skill directory and use its absolute path. Replace the `fygo-cli` prefix in Skill examples with that path, preserving each argument and its quoting. Run `--version` and the relevant `--help` first.

First execution automatically downloads one fixed-version CLI for this host; progress goes to stderr and later calls reuse the current user's cache. Standalone execution needs no Node/npm or global CLI. macOS/Linux require system curl, gzip, a SHA-256 tool (sha256sum or shasum), and standard shell utilities. For download failures, damaged caches or mirror configuration, read [first-use guidance](skills/fygo-cli/runtime-setup.md), follow its recovery steps and retry the original command. Skill updates are explicit; existing entries never upgrade themselves.

On Windows, call `scripts/fygo-cli.ps1` from PowerShell 5.1 or 7; the `.cmd` entry uses system Windows PowerShell. Quote arguments for the active shell; single quotes are recommended for JSON in PowerShell. Windows downloads and decompresses with system .NET and needs no Unix utilities. If PowerShell is missing or execution policy blocks the script, resolve the local environment issue and retry the same entry.

## Authorize and use

Start with [the Skill](skills/fygo-cli/SKILL.md). The Agent runs `login --authorize-only` and shares a link. Sign in and authorize in your browser, then explicitly return the one-time code. The Agent preserves target, profile and execution environment, runs `login --code`, verifies token persistence and makes a read-only request. No user terminal is needed; never provide passwords or tokens. Follow the [OAuth guide](skills/fygo-cli/reference/oauth.md).

## License

Standard MIT, copyright AXISSTORE TECHNOLOGY PTE. LTD., covering the distributed CLI, Skill and integration code, not the NAS operating system, server software or API access authorization. See [LICENSE](LICENSE) and [third-party notices](THIRD_PARTY_NOTICES.txt).
