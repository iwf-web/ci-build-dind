# ci-build-dind

Pre-built Docker-in-Docker (DinD) image used as a Jenkins build agent at IWF.

[![License](https://img.shields.io/github/license/iwf-web/ci-build-dind?label=License)](LICENSE.txt)
[![Contributor Covenant](https://img.shields.io/badge/Contributor%20Covenant-2.0-4baaaa)][code-of-conduct]
[![Docker Pulls](https://img.shields.io/docker/pulls/iwfwebsolutions/ci-build-dind)](https://hub.docker.com/r/iwfwebsolutions/ci-build-dind)

Based on [`docker:dind`](https://hub.docker.com/_/docker), with `bash`, `git`, `openssh-client` and `ca-certificates` pre-installed and the `git.iwf.io` SSH host key pre-trusted. Built and pushed to Docker Hub by GitHub Actions.

## Usage

### Pull

```bash
docker pull iwfwebsolutions/ci-build-dind:latest
```

### Jenkins (declarative pipeline)

```groovy
agent {
    docker {
        image 'iwfwebsolutions/ci-build-dind:latest'
        // --privileged: dockerd needs it. -u 0:0: overrides Jenkins's
        // auto-injected -u 1000:1000 (last -u wins). Required because
        // dockerd inside the agent needs root.
        args   '--privileged -u 0:0'
        reuseNode true
    }
}
```

Inside the stage, start dockerd before using `docker`:

```sh
nohup dockerd-entrypoint.sh >/tmp/dockerd.log 2>&1 &
for i in $(seq 1 30); do docker info >/dev/null 2>&1 && break; sleep 1; done
docker info
```

## Local development

### Build

```bash
docker build -t ci-build-dind:test ./src
```

Override the base tag via `DOCKER_VERSION` (defaults to `dind`):

```bash
docker build --build-arg DOCKER_VERSION=28-dind -t ci-build-dind:28 ./src
```

### Smoke test

```bash
docker run --rm --privileged ci-build-dind:test sh -c \
  'git --version && ssh -V && bash --version && grep git.iwf.io /root/.ssh/known_hosts'
```

### Lint

```bash
brew install hadolint
hadolint src/Dockerfile
```

## CI / release

Pushes to `main` (and `*.*.*` tags) trigger `.github/workflows/ci.yml`, which builds `linux/amd64` + `linux/arm64` and pushes to Docker Hub.

Required repository configuration:
- Variable `DOCKERHUB_USERNAME` — Docker Hub user/org (e.g. `iwfwebsolutions`)
- Variable `IMAGE_NAME` — full image name (e.g. `iwfwebsolutions/ci-build-dind`)
- Secret `DOCKERHUB_TOKEN` — Docker Hub access token with push rights

Base image is unpinned (`docker:dind` → latest). Trigger a manual rebuild via the **Actions → Continuous Integration → Run workflow** button to pick up upstream patches.

## Contributing

Please read [CONTRIBUTING.md][contributing] for details on our code of conduct and the process for submitting pull requests.

## Authors

### Special thanks for all the people who had helped this project so far

- **Manuele** - [D3strukt0r](https://github.com/D3strukt0r)

See also the full list of [contributors][gh-contributors] who participated in this project.

### I would like to join this list. How can I help the project?

We're currently looking for contributions for the following:

- [ ] Bug fixes
- [ ] Translations
- [ ] etc...

For more information, please refer to our [CONTRIBUTING.md][contributing] guide.

## License

This project is licensed under the MIT License - see the [LICENSE.txt](LICENSE.txt) file for details

## Acknowledgments

This project currently uses no third-party libraries or copied code.

[gh-contributors]: https://github.com/iwf-web/ci-build-dind/contributors
[contributing]: https://github.com/iwf-web/.github/blob/main/CONTRIBUTING.md
[code-of-conduct]: https://github.com/iwf-web/.github/blob/main/CODE_OF_CONDUCT.md
