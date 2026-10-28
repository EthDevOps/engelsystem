# Engelsystem Docker builds

Stable, versioned Docker images of [Engelsystem](https://github.com/engelsystem/engelsystem), built automatically from upstream releases.

The goal is a stable Engelsystem image that our deployments can pin to a version and auto-update from. Every stable upstream release becomes an image in our registry and a matching GitHub release here. There are no code changes or patches. The images are upstream's code, built with upstream's own `docker/Dockerfile`.

## Images

```
ghcr.io/ethdevops/engelsystem:<version>
```

| Tag      | Points to                        |
|----------|----------------------------------|
| `3.7.1`  | exactly that upstream release    |
| `3.7`    | latest patch of 3.7              |
| `3`      | latest release of major 3        |
| `latest` | newest stable upstream release   |

Only stable `vX.Y.Z` upstream tags are built. Pre-releases and upstream `main` are ignored.

## How it works

The only file here is [`.github/workflows/docker.yaml`](.github/workflows/docker.yaml). Every 6 hours it:

1. Finds the newest `vX.Y.Z` tag in `engelsystem/engelsystem`.
2. Stops if this repo already has a release with that tag.
3. Otherwise checks out upstream at that tag, builds `docker/Dockerfile` (with the tag passed as `VERSION`), and pushes the image to GHCR.
4. Creates a GitHub release with the same tag here. Update tooling that watches this repo's releases picks up the new version from it.

The release is created only after the image has been pushed, so a release always has a pullable image behind it.

## Manual runs

Actions → Docker → *Run workflow*. Leave the tag empty to build the latest upstream release, or enter a specific tag (e.g. `v3.6.0`) to build an older one. Manual builds of older tags do not move `latest`.

To rebuild a version that has already been released, delete its GitHub release first.

To rebuild with a security fix without touching existing tags, set `suffix` (e.g. `-php8.5.12`): only `<version><suffix>` is pushed and no release is created. `php_ref` builds the PHP base from that [docker-library/php](https://github.com/docker-library/php) commit instead of pulling `php:8.5-fpm-alpine`, for when a patched PHP is not on Docker Hub yet.

**Temporary:** every build currently appends php2shell (CVE-2026-109618/109619) hardening to the image (`conf.d/zz-php2shell.ini`, `php-fpm.d/zzz-php2shell.conf` with `pm.max_requests = 1`). Remove that step once the base ships PHP >= 8.5.12.

## Devcon build

The Telegram plugin needs upstream's plugin system, which no release has yet, so the Devcon image is built from a pinned upstream main commit. Run the workflow manually with:

- `tag`: the upstream main commit SHA (e.g. `f09f95ecd012217d3a91c7db5e9e1be4c5177d32`)
- `suffix`: `-devcon`
- `plugin_ref`: the commit SHA of a [maggo/engelsystem-telegram](https://github.com/maggo/engelsystem-telegram) release (e.g. `9e835fe0051dcaff7de8ec44d2748284eabca61b` = `v1.0.0`). Its CHANGELOG names the Engelsystem commit each release was tested against; use that as `tag`.

This pushes only `ghcr.io/ethdevops/engelsystem:<sha>-devcon` and creates no release. The plain tags stay pure upstream. With `plugin_ref` set, the build also locks `symfony/http-foundation` ≥ 7.4.21 and `symfony/yaml` ≥ 8.1.9, because upstream main still locks vulnerable versions. Drop that step once upstream bumps them. After deploying, run Admin › Plugins › Telegram › Install (first time) or Update (new release) so that its migrations run. Never Uninstall: it drops the plugin's tables. Platform needs (public webhook, outbound HTTPS to Telegram, an optional `/telegram/tick` pinger): see the plugin's README.

## Caveats

- Only the newest tag is checked each run. If upstream publishes two releases within 6 hours, build the older one manually.
- GitHub disables scheduled workflows in public repos after 60 days without commits. Re-enable it in the Actions tab, or commit something occasionally.
