# Codespaces configs — notes for future adjustments

These four configs exist only in the fork `jkemp814/images`, branch
`codespaces-config` (pushed to the fork's `main`). They are not part of
upstream `devcontainers/images`.

| Directory           | Builds from                                            | Base image                                            |
| ------------------- | ------------------------------------------------------ | ----------------------------------------------------- |
| `rust/`             | `../../src/rust/.devcontainer/Dockerfile` + features   | `rust:1-trixie@sha256:09027f87…`                      |
| `python/`           | `../../src/python/.devcontainer/Dockerfile` + features | `python:3-trixie@sha256:f34e3385…`                    |
| `rust-published/`   | nothing, image only                                    | `devcontainers/rust:2-1-trixie@sha256:2bf8d694…`      |
| `python-published/` | nothing, image only                                    | `devcontainers/python:3-3.15-trixie@sha256:eb39f073…` |

Why `.devcontainer/<sub>/`: Codespaces only discovers
`.devcontainer/devcontainer.json`, `.devcontainer.json`, or
`.devcontainer/<subdir>/devcontainer.json`. The per-image configs under
`src/<image>/.devcontainer` are invisible to the "Create codespace" flow.

The root `.devcontainer/devcontainer.json` (javascript-node + docker-in-docker)
stays the default; the four above appear as choices.

The build configs share the upstream Dockerfile via relative `build.dockerfile`
and `build.context`, so there is only one Dockerfile to maintain. Only the
`devcontainer.json` and the (generated) `devcontainer-lock.json` are local.

## Bumping base image digests

Both build configs pin through `build.args.VARIANT`; the Dockerfiles take
`ARG VARIANT` straight into `FROM`, so `VARIANT=trixie@sha256:…` pins the base.

```sh
# Docker Hub base images (needs the image pulled, or pull by tag first)
podman image inspect --format '{{index .RepoDigests 0}}' docker.io/library/rust:1-trixie
podman image inspect --format '{{index .RepoDigests 0}}' docker.io/library/python:3-trixie

# MCR published images: index digest without pulling
curl -sI -H 'Accept: application/vnd.oci.image.index.v1+json' \
  https://mcr.microsoft.com/v2/devcontainers/rust/manifests/2-1-trixie |
  grep -i docker-content-digest

# confirm a pin still resolves
podman manifest inspect docker://mcr.microsoft.com/devcontainers/rust:2-1-trixie@sha256:…
```

After `podman pull tag@sha256:<index>`, `RepoDigests` can report the
**platform** manifest digest rather than the index digest you pinned. Both
are valid; what matters is that the pin in `devcontainer.json` resolves.

## Known issue: features leave `/tmp` at 0755 (podman/buildah)

Two symptoms, same cause:

- runtime: `/tmp` is `0755`, so `mkdir /tmp/.X11-unix` and other non-root
  writes fail;
- build: a feature that runs `apt-get update` _after_ the offending feature
  dies with `Unable to mkstemp /tmp/apt.sig.* - GetTempFile (13: Permission
denied)` and `E: The repository … InRelease is not signed` — which reads
  like a signing or network fault.

Seen with `git:1`, `java:1` and `node:2` (2.1.1), always under rootless
podman/buildah. Upstream's Docker-built images ship `/tmp 1777`, so Docker
(Codespaces, WSL) is probably unaffected. Tracked upstream in
<https://github.com/devcontainers/features/issues/1480>.

Workarounds:

- Drop the feature when it is only there for metadata. `node:2` with
  `"version": "none"` installs nothing — that is why `python/` has no
  `node:2` and carries a comment saying so.
- Runtime only: `"runArgs": ["--tmpfs=/tmp:rw,exec,nosuid,nodev,mode=1777"]`.
  This does not help the build-time failure, because the bad mode is already
  in the image layer.

## Verified 2026-11-03

podman 5.8.9 / buildah 1.43.5, `@devcontainers/cli` 0.90.0, Debian trixie amd64:

- `rust/`: builds. rustc/cargo 1.99.1, git 2.56.0, zsh 5.9, uid 1000,
  metadata carries `capAdd: SYS_PTRACE` and `remoteUser: vscode`.
- `python/`: builds (without `node:2`). Python 3.15.2, pip 26.2.2,
  `/tmp` 1777, `flake8` present.
- both published configs: images pulled and run, `/tmp` 1777, remoteUser
  `vscode` taken from the image label.
- `build.args` really reach the Dockerfile: the CLI's command line contains
  `--build-arg VARIANT=trixie@sha256:…`.

## Local clone conventions

- `main` tracks `origin/main` (devcontainers/images, read-only) and is kept
  clean.
- `codespaces-config` tracks `fork/main` and holds these files.
- `git checkout codespaces-config` for local dev container work, then
  `git checkout main` again so the weekly `git pull --ff-only` stays green.
- On another machine: `git remote add fork git@github.com:jkemp814/images.git`,
  `git fetch fork`, `git checkout codespaces-config`.

## Gotchas

- The CLI derives the build image name from the workspace, not the config, so
  `devcontainer build` on two configs overwrites the same tag. Harmless for
  `up` and Codespaces.
- The published `devcontainers/python:3-3.15-trixie` advertises the Node
  feature in its metadata and puts `/usr/local/share/nvm/current/bin` on
  `PATH`, but has no node installed. Do not expect Node tooling there.
- `devcontainer read-configuration` validates the file but does not pull the
  image, so `remoteUser`, extensions and `capAdd` inherited from
  `devcontainer.metadata` show as undefined. Only `up` merges them.
- This host has no `docker` CLI; the Dev Containers extension reaches podman
  through `podman-wrapper`. For CLI work a `docker → podman` shim is needed.
  `podman buildx` accepts `--build-context` and `--iidfile` but not `--load`,
  and the CLI passes `--load`, so the shim must drop it.

## Test paths

- WSL / VS Code: checkout `codespaces-config`, "Reopen in Container", pick a
  config.
- Codespaces: `gh codespace create -R jkemp814/images`, pick a config.
- Sanity checks inside: `stat -c %a /tmp` → 1777, `id -u` → 1000,
  `rustc --version` or `python --version`, `git --version` → 2.56.0.
