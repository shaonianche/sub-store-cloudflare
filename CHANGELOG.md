# Changelog

All notable changes to this project will be documented in this file.

This project follows semantic versioning where practical.

## [Unreleased]

### Added

- Added local Playwright visual snapshots for the admin UI (`pnpm run check:visual`) so layout, overlay, and theme regressions can be compared automatically.

### Changed

- Built-in Mihomo templates now take their rule data from [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) instead of the ACL4SSR `.list`, Loyalsoldier `.txt`, and blackmatrix `.yaml` files: `acl4ssr-mihomo`, `acl4ssr-mihomo-no-emoji`, and `ai-streaming-mihomo` use compiled `.mrs` rule sets (`format: mrs`) served from `cdn.jsdelivr.net`, which also fixes the `SteamCN` provider that pointed at a 404 URL. `settings.rulesetCdn` switches the CDN host. The `loyalsoldier-*` templates keep their own lists because `reject`, `direct`, and `tld-not-cn` have no MetaCubeX equivalent.
- The `sing-box` output consumes those same rule sets: `RULE-SET` rules that reference a MetaCubeX provider become remote `.srs` rule sets (`format: binary`, `download_detour: "DIRECT"`) and `GEOIP,<country>` maps to the matching country rule set, so sing-box now follows the template's routing instead of only its groups. Rule sets are emitted once per file, a `REJECT` policy becomes `action: "reject"`, and `settings.rulesetCdn` applies here as well.
- Clash rule providers that sing-box cannot read (the Loyalsoldier presets, custom `.txt` / `.yaml` / `.list` URLs) are converted by the Worker and served as sing-box source rule sets: the profile points at `/download/collection/<id>/ruleset/<provider>?token=<download-token>`, which fetches the provider, translates `payload` YAML or Surge-style lines into sing-box rule items (ports become numbers, `1000-2000` becomes a `port_range`), and caches the result for the provider's `interval`. Mihomo keeps using the upstream lists directly, and sing-box now follows the same routing.
- Removed unused bottom TabBar and SideBar chrome now that navigation lives in the top segmented control.

### Fixed

- `sing-box` output now reads the collection's routing template instead of always emitting the built-in `PROXY` / `AUTO` groups: template groups are mirrored as `selector` / `urltest` outbounds (`fallback` / `load-balance` degrade to `selector`, since sing-box 1.13 removed them), `$all` and `filter` expand to the nodes that exist, `PASS` and dangling members are dropped, and `mixed-port` / `allow-lan` land on the `mixed` inbound. Rules that sing-box can express (`DOMAIN`, `DOMAIN-SUFFIX`, `DOMAIN-KEYWORD`, `DOMAIN-REGEX`, `IP-CIDR`, `IP-CIDR6`, `SRC-IP-CIDR`, `DST-PORT`, `SRC-PORT`, `PROCESS-NAME`, `PROCESS-PATH`) are translated and `MATCH` becomes `route.final`; `RULE-SET` / `GEOSITE` / `GEOIP` are skipped because sing-box only accepts `.srs` or source-format rule-sets.
- sing-box output now emits a complete VPN profile: a `tun` inbound with `auto_route`, a DNS section with a proxy resolver plus a direct bootstrap resolver, a `hijack-dns` rule, and `route.default_domain_resolver`. Without a `tun` inbound iOS and Android clients only ran a local proxy, so the VPN indicator never appeared and no traffic was reported. The profile now requires sing-box 1.12+.
- Replaced `vite-plugin-svg-icons` with a small first-party sprite plugin (`frontend/plugins/svg-sprite.ts`), which removes the `svg-baker` chain (`postcss@5`, `micromatch@3`, `braces`, `decode-uri-component`) from the build tree; `pnpm audit` now reports no known vulnerabilities in any install root, and the sprite only embeds the icons `<svg-icon>` resolves, trimming the entry bundle from 206 kB to 191 kB (72 kB to 66 kB gzip).
- Upgraded the Cloudflare tooling to the current releases (`wrangler` 4.141.0, `@cloudflare/vitest-pool-workers` 0.22.0, `@cloudflare/workers-types` 5.20260926.1), which brings a `miniflare` that ships patched `sharp` and `undici`, and pinned `sharp` 0.35.4 for the `miniflare` that still declares 0.35.2 exactly.
- Patched the dependency advisories that failed `pnpm run check:audit`: `hono` moves to 4.13.9, `js-yaml` to 4.3.2, and pnpm overrides pin the patched `postcss`, `nanoid`, and `brace-expansion` lines that only ship transitively.
- Bumped build tooling within its existing ranges (`vitest` 4.1.11, `svgo` 2.8.4, `brace-expansion` 2.1.7) to clear the remaining in-range advisories.
- sing-box output no longer emits legacy inbound fields or a WireGuard outbound, so profiles decode on sing-box 1.13+ clients that removed them.
- Repo checks pass again: documentation token snippets use an ESM one-liner instead of CommonJS, and the stale npm `frontend/package-lock.json`, unused `surgeformac_icon.png` asset, and tracked root `.dev.vars.example` are removed.
- Editor save/preview bar no longer covers the common-options and action blocks.
- Settings profile no longer duplicates the nav language switcher.
- Workers Builds install now sees `@playwright/test` in the root lockfile and skips Playwright browser download during deploy.
- Built-in ACL4SSR rule providers now set `format: text` so Mihomo can load `.list` files instead of treating them as YAML.
- Template rendering now drops dangling proxy-group references and uses the first rendered group for an empty-rule fallback MATCH.
- VLESS and trojan share links keep their transport parameters (`type`, `path`, `host`, `serviceName`), so WebSocket, gRPC and HTTP/2 nodes render with their real path and host instead of an empty default. The sing-box output writes them into `transport`, because sing-box only accepts `tcp` / `udp` in `network` and otherwise rejects the whole profile with `decode config: outbounds[n].network: unknown network: ws`.

## [1.1.0] - 2026-07-11

### Added

- Added a guided interactive CLI setup, an explicit empty quick-install mode, and cross-platform deployment token generation.
- Added a first-run admin checklist for Source, Collection, and client-link creation.
- Added five-minute quick-start and upgrade guides for Deploy Button, Agent/CLI, D1 migrations, backup, and rollback.
- Added installer helper tests and a deployment-experience release check.

### Changed

- Rebuilt the Chinese and English README files around a three-step ordinary-user deployment path.
- Non-interactive installs without private setup now stop before deployment instead of risking example source import.
- Deploy Button documentation now explains repository copies, required secrets, and upstream upgrade behavior.

### Security

- Removed the root `.dev.vars.example` that caused Cloudflare's Deploy form to prefill public placeholder values for required Worker Secrets.

## [1.0.0] - 2026-07-11

### Added

- Added JSON5 subscription input, a distinct Surge Mac output target, and structured Snell, SSH, and HTTP/2 CONNECT client-line compatibility.
- Added authenticated one-shot proxy/subscription conversion and Mihomo, Surge, Loon, and Quantumult X rule conversion APIs plus an admin Tools page.
- Added allowlisted remote subscription metadata propagation, hashed Cache API keys, configurable edge-cache TTL, forced refresh, conditional requests, and stale-on-error fallback.
- Added scoped download grants with one-time plaintext tokens, D1-stored SHA-256 hashes, target restrictions, expiration, revocation, and deployment-token compatibility.
- Added a bounded 50-entry recycle bin for deleted sources, collections, custom templates, and scoped download grants.
- Added configurable HTTPS node IP, location, organization, and ASN lookup from the preview node panel.
- Added an upstream compatibility matrix and a D1 migration for the new compatibility resources.

### Changed

- Repositioned the project as a tested Cloudflare-native compatibility edition while keeping Workers Static Assets + Worker API + D1 + Worker Secrets.
- Updated product and Agent boundaries to preserve scoped private links and the bounded recycle bin while continuing to reject public sharing, unbounded archives, runtime scripts, files, artifacts, queues, cron, and persistent logs.
- Download responses now propagate safe subscription metadata and expose a non-sensitive edge-cache status header.

### Verification

- Added rule conversion, JSON5, Surge Mac, response metadata, scoped authorization, and recycle/restore tests.
- Kept the full release gate, dry-run bundle measurement, startup profiling, and deployed smoke verification as release requirements.

## [0.3.0] - 2026-07-11

### Added

- Added build-time JavaScript Filter / Operator support without runtime `eval()` or `new Function()`.
- Added code-owned script metadata at `/api/scripts`, metadata-driven admin UI controls, and two Free-verified built-ins: TLS fingerprint and name regex filter.
- Added gitignored personal script manifests and source directories that the Agent/CLI installer compiles into the Worker registry.
- Added Worker/D1 integration coverage for script metadata, validation, execution, arguments, unavailable scripts, and per-stage limits.

### Changed

- Aligned source and collection validation, immutable IDs, partial updates, and empty-collection membership semantics across the Worker, frontend, installer, and documentation.
- Removed the advertised `surge-mac` target because it had no independent renderer; use `surge` or a supported YAML target instead.

### Documentation

- Documented Free-compatible build-time JavaScript filters and operators, personal deployment steps, upstream compatibility levels, security boundaries, and performance gates.

## [0.2.0] - 2026-07-10

### Added

- Added Workers runtime and D1 migration integration tests for auth, storage restore, downloads, parsers, and payload limits.
- Added a real `wrangler dev` startup smoke test and production dependency audits to the release gate.
- Added bounded readers for API bodies, remote subscriptions, flow metadata, and DoH responses.

### Changed

- Moved built-in routing templates from D1 seed rows into Worker-owned code so template fixes reach existing deployments immediately.
- Removed request-time schema creation and default seeding; D1 migrations are now the only schema path.
- Consolidated the repository into one pnpm workspace and one root lockfile.
- Updated Wrangler, Workers types, Hono, Axios, Vite, Vue tooling, TypeScript, and YAML dependencies.
- Lazy-loaded the settings route and CodeMirror editor, and removed redundant precompression output.
- Updated Wrangler compatibility settings to `2026-07-08` with `nodejs_compat` across all generated and checked configs.

### Security

- Added CSP, frame denial, no-referrer, no-sniff, and permissions-policy response headers.
- Changed unhandled failures to structured server logs plus generic client-facing 500 responses.
- Removed the admin token from the browser URL after ingest and changed backup export to authenticated blob download.
- Stopped the installer from printing private D1 database IDs and strengthened ignored-file privacy verification.

### Documentation

- Documented the new runtime limits, code-owned template model, release checks, and header-authenticated backup export.

## [0.1.1] - 2026-06-28

### Changed

- Removed repository GitHub Actions and Dependabot so the upstream project does not depend on GitHub automation.
- Kept lightweight GitHub issue forms and a pull request template for contributor intake; they do not run CI/CD.
- Clarified that the Cloudflare Deploy Button is the Cloudflare-hosted template import path, while `pnpm run install:cloudflare` is the local Agent/CLI deployment path.
- Updated Worker compatibility dates and documented the Node 22 + pnpm local development baseline.

### Verification

- Local release checks and Wrangler dry-run deployment remain the release gate.

## [0.1.0] - 2026-06-28

### Added

- Cloudflare-native Worker application with Static Assets, Worker API, D1, and Worker Secrets.
- Source and collection management for remote subscription URLs and local node text.
- Node filters for include/exclude, rename, delete-field, dedupe, sort, regex-sort, flag handling, quick options, and DNS resolve workflows.
- Built-in routing templates for Mihomo-compatible YAML output.
- Output targets for Mihomo, Stash, Surge, Surfboard, Loon, Egern, Shadowrocket, Quantumult X, sing-box, v2ray, URI, and JSON.
- Preview, backup/restore, temporary `url` / `content` / `ua` conversion parameters, and subscription usage metadata.
- Deploy to Cloudflare button support with root `wrangler.jsonc`.
- Agent/CLI installer via `pnpm run install:cloudflare`.
- Release checks for Worker/frontend builds, agent setup, deployment config, worker contract, module format, open-source hygiene, and git history privacy.

### Documentation

- Added deployment, AI agent install, architecture, and product-scope documentation.
- Added contributing, support, security, code of conduct, release notes, and local release checks.
