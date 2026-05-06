ARG DOCKER_VERSION=dind

FROM docker:${DOCKER_VERSION}

RUN <<EOT
    set -eu
    apk add --no-cache \
        bash \
        git \
        openssh-client \
        ca-certificates
    mkdir -p /root/.ssh
    ssh-keyscan -H git.iwf.io >> /root/.ssh/known_hosts 2>/dev/null
    chmod 600 /root/.ssh/known_hosts
EOT
