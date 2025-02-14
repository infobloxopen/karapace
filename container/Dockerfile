# Use Photon OS as the base image
FROM photon:latest AS builder

ARG KARAPACE_VERSION

# Install Python and other dependencies
RUN tdnf -y update && tdnf -y install \
    python3 \
    python3-virtualenv \
    python3-pip \
    wget \
    gcc \
    make \
    openssl-devel \
    cyrus-sasl-devel \
    diffutils \
    tar \
    glibc-devel \
    linux-api-headers \
    binutils

# Create, activate, and enforce usage of virtualenv.
RUN python3 -m venv /venv
ENV PATH="/venv/bin:$PATH"
ENV PIP_REQUIRE_VIRTUALENV=true

# Install Go
ENV GO_VERSION=1.23.4
ENV PATH="/usr/local/go/bin:${PATH}"
RUN ARCH=$(uname -m) \
 && [ "$ARCH" = "aarch64" ] && ARCH="arm64" || true \
 && wget "https://go.dev/dl/go${GO_VERSION}.linux-${ARCH}.tar.gz" \
 && tar -C /usr/local -xzf "go${GO_VERSION}.linux-${ARCH}.tar.gz" \
 && rm "go${GO_VERSION}.linux-${ARCH}.tar.gz"

# Install librdkafka from source
RUN wget https://github.com/edenhill/librdkafka/archive/refs/tags/v2.8.0.tar.gz \
 && tar -xzf v2.8.0.tar.gz \
 && cd librdkafka-2.8.0 \
 && ./configure \
 && make \
 && make install \
 && cd .. \
 && rm -rf librdkafka-2.8.0 v2.8.0.tar.gz

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
        PRETEND_VERSION="$(python -c 'from src.karapace import version; print(version.__version__)')"; \
    else \
        PRETEND_VERSION=$KARAPACE_VERSION; \
    fi; \
    SETUPTOOLS_SCM_PRETEND_VERSION=$PRETEND_VERSION python3 -m pip install --no-deps .

# Karapace image, i.e. production.
FROM photon:latest AS karapace

# Install Python 3 and other dependencies
RUN tdnf -y update && tdnf -y install \
    python3 \
    python3-virtualenv \
    python3-pip \
    gcc \
    gdb \
    shadow \
    glibc-devel \
    linux-api-headers \
    binutils \
    && tdnf clean all

RUN ulimit -c unlimited

# Need to be root to install packages.
USER 0

# Setup user and directories.
# https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/images/creating-images#use-uid_create-images
RUN useradd --uid 1000 --gid root karapace \
 && mkdir -p /opt/karapace /opt/karapace/runtime /var/log/karapace \
 && chgrp -R 0 /opt/karapace /opt/karapace/runtime /var/log/karapace \
 && chmod -R g+rwX /opt/karapace

# Install protobuf compiler.
ARG PROTOBUF_COMPILER_VERSION="3.12.4"
RUN tdnf -y update \
 && tdnf -y install \
    protobuf-c \
    gcc \
    && tdnf clean all

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
