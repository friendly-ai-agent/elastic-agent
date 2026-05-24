# Copyright Elasticsearch B.V. and/or licensed to Elasticsearch B.V. under one
# or more contributor license agreements. Licensed under the Elastic License 2.0;
# you may not use this file except in compliance with the Elastic License 2.0.

# Image used by the "docker" integration-test instance provisioner. It runs systemd
# as PID 1 (so the Elastic Agent's systemd service install works) and sshd (so the
# existing Linux test runner can drive it over SSH exactly like a VM).
ARG UBUNTU_VERSION=24.04
FROM ubuntu:${UBUNTU_VERSION}

ENV DEBIAN_FRONTEND=noninteractive
ENV container=docker

# Base packages: systemd (PID 1), sshd, sudo, and the build toolchain the tests
# need (build-essential + unzip). Baking these in means the runner's Prepare step
# can be skipped entirely (see GO_VERSION below and provisioner Prepared=true).
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        systemd systemd-sysv \
        openssh-server \
        sudo \
        curl ca-certificates gnupg \
        build-essential unzip \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Bake in the exact Go toolchain the tests expect (matches .go-version, passed by
# the provisioner). This is the slow/bandwidth-heavy part of the runner's Prepare
# step; doing it once at image-build time makes every container start ready to go.
ARG GO_VERSION
ARG GO_ARCH=amd64
RUN test -n "${GO_VERSION}" \
    && curl -Ls "https://go.dev/dl/go${GO_VERSION}.linux-${GO_ARCH}.tar.gz" -o /tmp/go.tar.gz \
    && tar -C /usr/local -xzf /tmp/go.tar.gz \
    && rm /tmp/go.tar.gz \
    && ln -sf /usr/local/go/bin/go /usr/bin/go \
    && ln -sf /usr/local/go/bin/gofmt /usr/bin/gofmt

# Pre-install the test build tools (mage + gotestsum) at the versions pinned in the
# repo's go.mod, as the 'ubuntu' user so the binaries and warm Go module/build caches
# land in ubuntu's HOME. The runner's per-run "make mage && mage integration:prepareOnRemote"
# (in linuxCopy) then resolves these same versions from a warm cache instead of
# downloading and compiling them from scratch on every run.
ARG MAGE_VERSION
ARG GOTESTSUM_VERSION
RUN test -n "${MAGE_VERSION}" && test -n "${GOTESTSUM_VERSION}" \
    && su ubuntu -c "export HOME=/home/ubuntu; \
        go install github.com/magefile/mage@${MAGE_VERSION} && \
        go install gotest.tools/gotestsum@${GOTESTSUM_VERSION}"

# Ensure an 'ubuntu' user exists with passwordless sudo (mirrors a cloud image, which
# is what the runner's scripts assume).
RUN if ! id -u ubuntu >/dev/null 2>&1; then useradd --create-home --shell /bin/bash ubuntu; fi \
    && install -d -m 0755 -o ubuntu -g ubuntu /home/ubuntu \
    && echo 'ubuntu ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/90-ubuntu \
    && chmod 0440 /etc/sudoers.d/90-ubuntu

# sshd is socket-activated on modern Ubuntu (ssh.socket listens on :22 and starts
# ssh.service on the first connection); the openssh-server package already enables it.
# Just ensure its runtime directory exists.
RUN mkdir -p /run/sshd

# Mask units that are pointless inside a container and can slow down or break boot.
RUN for unit in \
        systemd-resolved.service \
        systemd-networkd.service \
        systemd-networkd.socket \
        getty.target \
        console-getty.service \
        sys-kernel-config.mount \
    ; do ln -sf /dev/null "/etc/systemd/system/$unit"; done

STOPSIGNAL SIGRTMIN+3
CMD ["/sbin/init"]
