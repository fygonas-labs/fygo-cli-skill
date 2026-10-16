# Fygo CLI Skill

Usage guidance and public integration content for Fygo. The runtime CLI and Skill come from the same npm package version.

## Install the Skill

```bash
npx skills add fygonas-labs/fygo-cli-skill --skill fygo-cli -g
```

## Install or upgrade the CLI

Requires Node.js 18+. Install or upgrade explicitly:

```bash
npm install -g @fygonas-labs/fygo-cli@latest
fygo-cli --version
fygo-cli --help
```

During publication processing, `latest` may still point to an older release; installed copies never update automatically. Installing a Skill does not install the CLI or sign in. Product state is independent and this release requires fresh authorization; old credentials remain untouched and are neither read nor migrated.

## Authorize and use

Start with [the Skill](skills/fygo-cli/SKILL.md). The Agent runs `login --authorize-only` and shares a link. Sign in and authorize in your browser, then explicitly return the one-time code. The Agent preserves target, profile and execution environment, runs `login --code`, verifies token persistence and makes a read-only request. No user terminal is needed; never provide passwords or tokens. Follow the [OAuth guide](skills/fygo-cli/reference/oauth.md).

## License

Standard MIT, copyright AXISSTORE TECHNOLOGY PTE. LTD., covering the distributed CLI, Skill and integration code, not the NAS operating system, server software or API access authorization. See [LICENSE](LICENSE) and [third-party notices](THIRD_PARTY_NOTICES.txt).
