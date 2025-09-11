# Build stage
FROM photon:latest AS builder

ARG KARAPACE_VERSION

# Install Python and other dependencies (without python3-virtualenv to avoid system package)
RUN tdnf -y update && tdnf -y install \
    python3 \
    python3-pip \
    python3-xml \
    expat-devel \
    wget \
    gcc \
    make \
    glibc-devel \
    zlib-devel \
    openssl-devel \
    libffi-devel \
    linux-api-headers \
    binutils \
    && tdnf clean all

# Install virtualenv directly via pip to avoid system package vulnerabilities
RUN pip3 install virtualenv>=20.26.6

# Create, activate, and enforce usage of virtualenv with secure versions
RUN python3 -m venv /venv
ENV PATH="/venv/bin:$PATH"
ENV PIP_REQUIRE_VIRTUALENV=true

# Upgrade setuptools and virtualenv to latest secure versions in venv
RUN pip install --upgrade setuptools>=78.1.1 virtualenv>=20.26.6

# Remove any system-level python packages that might have vulnerabilities
RUN tdnf -y remove python3-setuptools python3-virtualenv || true

# Install Go
ENV GO_VERSION=1.24.6
ENV PATH="/usr/local/go/bin:${PATH}"
RUN ARCH=$(uname -m) \
 && [ "$ARCH" = "aarch64" ] && ARCH="arm64" || true \
 && [ "$ARCH" = "x86_64" ] && ARCH="amd64" || true \
 && wget -q "https://golang.org/dl/go${GO_VERSION}.linux-${ARCH}.tar.gz" -O /tmp/go.tar.gz \
 && tar -C /usr/local -xzf /tmp/go.tar.gz \
 && rm /tmp/go.tar.gz

# COPY requirements.txt before other files to maximize Docker layer cache efficiency
COPY requirements/requirements*.txt /tmp/
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install \
        -r /tmp/requirements.txt \
        -r /tmp/requirements-typing.txt

# Install gunicorn separately to avoid conflicts
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install gunicorn

COPY . /build/karapace-repo
WORKDIR /build/karapace-repo

# Sync vendor directory to fix Go mod inconsistencies
RUN cd go/protopace && go mod tidy && go mod vendor

RUN --mount=type=cache,target=/root/.cache/pip \
    if [ -z "${KARAPACE_VERSION}" ]; then \
        PRETEND_VERSION="$(python -c 'from src.karapace import version; print(version.__version__)')"; \
    else \
        PRETEND_VERSION=$KARAPACE_VERSION; \
    fi; \
    SETUPTOOLS_SCM_PRETEND_VERSION=$PRETEND_VERSION python3 -m pip install --no-deps .

# Karapace image, i.e. production
FROM photon:latest AS karapace

# Install minimal runtime dependencies
RUN tdnf -y update && tdnf -y install \
    python3 \
    python3-pip \
    libffi \
    openssl \
    librdkafka \
    shadow-tools \
    && tdnf clean all

# Copy virtualenv from builder and activate it
COPY --from=builder /venv /venv
ENV PATH="/venv/bin:$PATH"

# Setup user and directories (OpenShift compatible)
# Using gid root for OpenShift compatibility while maintaining security
RUN useradd -m -u 1000 -g root karapace && \
    mkdir -p /opt/karapace /opt/karapace/runtime /var/log/karapace && \
    chgrp -R 0 /opt/karapace /var/log/karapace && \
    chmod -R g+rwX /opt/karapace /var/log/karapace

COPY ./container/start.sh /opt/karapace/
COPY ./container/healthcheck.py /opt/karapace/

RUN chmod 550 /opt/karapace/start.sh && \
    chmod 644 /opt/karapace/healthcheck.py && \
    chgrp 0 /opt/karapace/start.sh /opt/karapace/healthcheck.py && \
    chmod g+r /opt/karapace/start.sh /opt/karapace/healthcheck.py

WORKDIR /opt/karapace
USER karapace

HEALTHCHECK --interval=10s --timeout=30s --retries=3 --start-period=60s \
    CMD python3 healthcheck.py http://localhost:${KARAPACE_PORT:-8081}/_health || exit 1