# AGENTS.md

Guidance for AI coding agents (Claude Code, Cursor, Copilot, etc.) working in this repository.

## Purpose

Single-image repo. Builds `iwfwebsolutions/ci-build-dind` — a Docker-in-Docker (DinD) Jenkins agent image used by IWF pipelines (e.g. `publica-storybook`). Image bundles `bash`, `git`, `openssh-client`, `ca-certificates` on top of `docker:dind`, with `git.iwf.io` SSH host key pre-trusted.

The image is consumed in Jenkinsfiles via:

```groovy
agent {
    docker {
        image 'iwfwebsolutions/ci-build-dind:latest'
        args  '--privileged -u 0:0'   // dockerd needs root + privileged
        reuseNode true
    }
}
```

`-u 0:0` is mandatory: it overrides Jenkins's auto-injected `-u 1000:1000` (last `-u` wins). Without it, `dockerd` cannot start.

## Layout

- `src/Dockerfile` — the image. Uses BuildKit heredoc syntax (`# syntax=docker/dockerfile:1`) and exposes `ARG DOCKER_VERSION=dind` so consumers can pin (e.g. `28-dind`) without editing the file.
- `src/` is the build context (`docker build ./src`). Keep build inputs inside `src/` so the GitHub Actions `paths:` trigger on `src/**` stays meaningful.
- `.github/workflows/ci.yml` — multi-arch (`linux/amd64,linux/arm64`) Buildx build + push to Docker Hub. Push is gated on `vars.DOCKERHUB_USERNAME != '' && env.dockerhub_token != ''` so PRs from forks build but don't push.
- `test/workflows/` — fixture event payloads for running workflows locally with `act`.

## Common commands

```bash
# Build locally
docker build -t ci-build-dind:test ./src

# Build with pinned Docker version
docker build --build-arg DOCKER_VERSION=28-dind -t ci-build-dind:28 ./src

# Smoke test (must run privileged so dockerd works)
docker run --rm --privileged ci-build-dind:test sh -c \
  'git --version && ssh -V && bash --version && grep git.iwf.io /root/.ssh/known_hosts'

# Lint
hadolint src/Dockerfile

# Run CI workflow locally (devcontainer pre-installs `act`)
act -W .github/workflows/ci.yml workflow_dispatch
act -W .github/workflows/ci.yml pull_request -e test/workflows/event-pr.json
act -W .github/workflows/ci.yml push        -e test/workflows/event-tag.json
```

## Release / tagging

`docker/metadata-action` derives tags from the ref:
- push to `main` → `latest`
- semver tag `X.Y.Z` → `X.Y.Z` + `latest`
- branch push → branch name

Required repo config for pushes to Docker Hub:
- Variable `DOCKERHUB_USERNAME` (e.g. `iwfwebsolutions`)
- Variable `IMAGE_NAME` (e.g. `iwfwebsolutions/ci-build-dind`)
- Secret `DOCKERHUB_TOKEN` (push-scoped Docker Hub access token)

Base image is unpinned by default. To pick up upstream `docker:dind` patches without a code change, trigger **Actions → Continuous Integration → Run workflow** manually.

## Conventions

- Default branch is `main`. Dependabot targets `develop` for PR churn isolation; `develop` is merged into `main` to release.
- `CODEOWNERS` makes `@D3strukt0r` the default reviewer.
- Dependabot auto-merge (`.github/workflows/dependabot-automerge.yml`) is gated on `github.repository == 'iwf-web/ci-build-dind'` — update if the repo is forked or moved.
