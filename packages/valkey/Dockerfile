# Stage 1: Builder (compile and package)
ARG DISTRO=debian:12
FROM ${DISTRO} AS builder


ARG RELEASE_TAG=8.1.3
ARG MAJOR=8
ARG ARCH=amd64

WORKDIR /valkey-src

# Install build dependencies required for compilation and packaging
# Adjust based on architecture (e.g., libatomic for arm64)
RUN apt update && \
    apt install -y \
        build-essential \
        curl \
        git \
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
        rdma-core && \
    if [ "$(dpkg --print-architecture)" = "arm64" ]; then \
        apt install -y libatomic1; \
    fi && \
    apt clean && \
    rm -rf /var/lib/apt/lists/*

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
    make -j$(nproc) $MAKE_FLAGS && \
    make install DESTDIR=/tmp/valkey-install PREFIX=/usr

# Copy custom debian/ directory from the host repository
COPY packages/valkey/debian/ debian/

# Build Debian packages using dpkg-buildpackage
ENV DEB_BUILD_OPTIONS="parallel=$(nproc)"
RUN dpkg-buildpackage -b -us -uc --host-arch ${ARCH}

# Stage 2: Output (extract only .deb files for clean export)
FROM scratch AS output
COPY --from=builder /valkey-src/*.deb /
