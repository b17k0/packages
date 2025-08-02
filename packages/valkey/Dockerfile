# Stage 1: Builder (compile and package)
ARG DISTRO
FROM ${DISTRO} AS builder

# Valkey release tag to build (passed from GitHub Actions)
ARG RELEASE_TAG

# Major version (e.g., 7 or 8, parsed in workflow)
ARG MAJOR

# Architecture (e.g., amd64, arm64)
ARG ARCH=amd64

# Install build dependencies required for compilation and packaging
# Adjust based on architecture (e.g., libatomic for arm64)
RUN apt update && \
    apt install -y build-essential curl git autoconf pkg-config libssl-dev libsystemd-dev zlib1g-dev debhelper && \
    if [ "$(dpkg --print-architecture)" = "arm64" ]; then \
        apt install -y libatomic1; \
    fi && \
    # Clean up to reduce image size
    rm -rf /var/lib/apt/lists/*

# Download and extract Valkey source code from GitHub
RUN curl -L https://github.com/valkey-io/valkey/archive/refs/tags/${RELEASE_TAG}.tar.gz -o valkey.tar.gz && \
    tar xzf valkey.tar.gz && \
    cd valkey-${RELEASE_TAG} && \
    # Clean any previous build artifacts
    make distclean

# Compile binaries from sources with version-specific and arch-specific flags
RUN cd valkey-${RELEASE_TAG} && \
    MAKE_FLAGS="SERVER_CFLAGS='-Werror' BUILD_TLS=yes USE_FAST_FLOAT=yes" && \
    if [ "$MAJOR" != "7" ]; then \
        MAKE_FLAGS="$MAKE_FLAGS BUILD_RDMA=yes"; \
    fi && \
    # For arm64, ensure atomic ops are handled (libatomic already installed)
    if [ "$(dpkg --print-architecture)" = "arm64" ]; then \
        MAKE_FLAGS="$MAKE_FLAGS CFLAGS+=-latomic"; \
    fi && \
    make -j$(nproc) $MAKE_FLAGS && \
    # Install compiled binaries to a temporary directory for packaging
    make install DESTDIR=/tmp/valkey-install PREFIX=/usr

# Copy custom debian/ directory from the host repository
COPY packages/valkey/debian/ valkey-${RELEASE_TAG}/debian/

# Build Debian packages using dpkg-buildpackage (now that binaries are compiled)
# Set DEB_BUILD_OPTIONS for parallel build and architecture
RUN cd valkey-${RELEASE_TAG} && \
    DEB_BUILD_OPTIONS="parallel=$(nproc)" dpkg-buildpackage -b -us -uc --host-arch ${ARCH}

# Stage 2: Output (extract only .deb files for clean export)
FROM scratch AS output
COPY --from=builder /*.deb /
