# Stage 1: Builder (compile and package)
ARG DISTRO=debian:12
FROM ${DISTRO} AS builder

# Valkey release tag to build (passed from GitHub Actions; default for testing)
ARG RELEASE_TAG=8.1.3

# Major version (e.g., 7 or 8, parsed in workflow; default for testing)
ARG MAJOR=8

# Architecture (e.g., amd64, arm64)
ARG ARCH=amd64

# Install build dependencies required for compilation and packaging in one layer
# Adjust based on architecture (e.g., libatomic for arm64)
RUN apt update && \
    apt install -y \
        build-essential \
        curl \
        git \
        tree \
        autoconf \
        pkg-config \
        libssl-dev \
        libsystemd-dev \
        zlib1g-dev \
        debhelper \
        libhiredis-dev \
        libjemalloc-dev \
        librdmacm-dev \
        libibverbs-dev \
        rdma-core \
        fakeroot \
        liblua5.1-0-dev \
        liblzf-dev \
        lua-bitop-dev \
        lua-cjson-dev \
        procps \
        tcl \
        tcl-tls && \
    if [ "$(dpkg --print-architecture)" = "arm64" ]; then \
        apt install -y libatomic1; \
    fi && \
    apt clean && \
    rm -rf /var/lib/apt/lists/*

# Set working directory for source code
WORKDIR /valkey-src

# Download, extract, clean, and compile sources
RUN curl -L https://github.com/valkey-io/valkey/archive/refs/tags/${RELEASE_TAG}.tar.gz -o valkey.tar.gz && \
    tar xzf valkey.tar.gz --strip-components=1 && \
    rm valkey.tar.gz && \
    make distclean

# Compile binaries with version-specific and arch-specific flags
ENV MAKE_FLAGS="SERVER_CFLAGS='-Werror' BUILD_TLS=yes USE_FAST_FLOAT=yes USE_REDIS_SYMLINKS=no"
RUN if [ "$MAJOR" != "7" ]; then \
        export MAKE_FLAGS="$MAKE_FLAGS BUILD_RDMA=yes"; \
    fi && \
    if [ "$(dpkg --print-architecture)" = "arm64" ]; then \
        export MAKE_FLAGS="$MAKE_FLAGS CFLAGS+=-latomic"; \
    fi && \
    make -j$(nproc) $MAKE_FLAGS install DESTDIR=/tmp/valkey-install PREFIX=/usr

# Copy custom debian/ directory from the host repository
COPY packages/valkey/debian/ debian/

RUN pwd
RUN tree /

# Build Debian packages using dpkg-buildpackage
ENV DEB_BUILD_OPTIONS="parallel=$(nproc)"
RUN dpkg-buildpackage -b -us -uc --host-arch ${ARCH} && \
    rm -rf /valkey-src/* # Clean up source after build to reduce layer size

# Stage 2: Output (extract only .deb files for clean export)
FROM scratch AS output
COPY --from=builder /*.deb /
