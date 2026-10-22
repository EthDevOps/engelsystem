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

To rebuild with a security fix without touching existing tags, set `suffix` (e.g. `-php8.5.12`): only `<version><suffix>` is pushed and no release is created.

## Caveats

- Only the newest tag is checked each run. If upstream publishes two releases within 6 hours, build the older one manually.
- GitHub disables scheduled workflows in public repos after 60 days without commits. Re-enable it in the Actions tab, or commit something occasionally.
