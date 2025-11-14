# Use Python 3.12 to address multiple CVEs including CVE-2023-36632, CVE-2025-1795
FROM python:3.12-bookworm AS builder

ARG KARAPACE_VERSION

# Create, activate, and enforce usage of virtualenv.
RUN python3 -m venv /venv
ENV PATH="/venv/bin:$PATH"
ENV PIP_REQUIRE_VIRTUALENV=true

# Update pip and setuptools to address multiple vulnerabilities
RUN python3 -m pip install --upgrade \
    pip>=23.3 \
    setuptools>=78.1.1

# Install golang needed by extensions (updated to address CVE-2025-61723, CVE-2025-61725, CVE-2025-22871, CVE-2025-58187)
ENV GO_VERSION=1.24.9
ENV PATH="/usr/local/go/bin:${PATH}"
RUN wget --progress=dot:giga "https://go.dev/dl/go${GO_VERSION}.linux-$(dpkg --print-architecture).tar.gz" \
 && tar -C /usr/local -xzf "go${GO_VERSION}.linux-$(dpkg --print-architecture).tar.gz" \
 && rm "go${GO_VERSION}.linux-$(dpkg --print-architecture).tar.gz"

# Copy the requirements.txt and install dependencies in venv. Using a separate
# command to use layer caching.
#
# Note: the requirements.txt is pinned, if any of the dependencies is updated
# the cache will be invalidated and the image regenerated, which is the
# intended behavior.
COPY ./requirements/requirements.txt /build/
RUN --mount=type=cache,target=/root/.cache/pip \
    python3 -m pip install -r /build/requirements.txt

COPY . /build/karapace-repo
WORKDIR /build/karapace-repo
RUN --mount=type=cache,target=/root/.cache/pip \
    if [ -z "${KARAPACE_VERSION}" ]; then \
        PRETEND_VERSION="3.12.0"; \
    else \
        PRETEND_VERSION=$KARAPACE_VERSION; \
    fi; \
    SETUPTOOLS_SCM_PRETEND_VERSION=$PRETEND_VERSION python3 -m pip install --no-deps .

# Karapace image, i.e. production.
FROM python:3.12-slim-bookworm AS karapace

# Setup user and directories.
# https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/images/creating-images#use-uid_create-images
RUN useradd --system --gid 0 karapace \
 && mkdir /opt/karapace /opt/karapace/runtime /var/log/karapace \
 && chgrp -R 0  /opt/karapace /opt/karapace/runtime /var/log/karapace \
 && chmod -R g+rwX /opt/karapace

# Update system packages to get security patches and install protobuf compiler
# Update packages to address critical CVEs: libexpat1, curl, protobuf, etc.
RUN apt-get update \
 && apt-get upgrade -y \
 && apt-get install --assume-yes --no-install-recommends \
    curl \
    wget \
    ca-certificates \
    gnupg \
    gpgv \
    libexpat1 \
    protobuf-compiler \
    libprotobuf-dev \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/* \
 && rm -rf /tmp/* /var/tmp/*

# Install newer Go version to address Go stdlib vulnerabilities (CVE-2025-61723, CVE-2025-61725, CVE-2025-58187)
RUN ARCH=$(dpkg --print-architecture) \
 && curl -L "https://go.dev/dl/go1.24.9.linux-${ARCH}.tar.gz" | tar -C /usr/local -xz \
 && ln -sf /usr/local/go/bin/go /usr/bin/go \
 && ln -sf /usr/local/go/bin/gofmt /usr/bin/gofmt

# Copy virtualenv from builder and activate it.
COPY --from=builder /venv /venv
ENV PATH="/venv/bin:$PATH"

COPY ./container/start.sh /opt/karapace
RUN chmod 550 /opt/karapace/start.sh \
 && chgrp -R 0  /opt/karapace/start.sh

COPY ./container/healthcheck.py /opt/karapace

WORKDIR /opt/karapace
USER karapace

HEALTHCHECK --interval=10s --timeout=30s --retries=3 --start-period=60s \
    CMD python3 healthcheck.py http://localhost:$KARAPACE_PORT/_health || exit 1
