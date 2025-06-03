# Build and run Xen Orchestra Community Edition
# This Dockerfile uses multi-stage builds to create a minimal production image

# Use bash for all RUN commands and enable proper error handling
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Base builder image for both XOA and libvhdi compilation
FROM node:20.18-alpine3.21 AS builder
# Install essential build dependencies
RUN apk add --no-cache \
    git python3 g++ make bash \
    fuse fuse3 fuse-dev fuse3-dev \
    curl libc6-compat automake autoconf \
    libtool gettext-dev pkgconf

# Stage 1: Build Xen Orchestra Application (XOA)
FROM builder AS build-xoa
WORKDIR /home/node
# Disable telemetry for Turbo
ENV TURBO_TELEMETRY_DISABLED=1

# Clone and build XO application
# - Install url-loader for webpack
# - Clone XO repository
# - Build core and web interface
# - Link plugins to xo-server
# - Remove development dependencies
RUN --mount=type=cache,target=/root/.npm \
    --mount=type=cache,target=/root/.yarn \
    npm install url-loader --save-dev && \
    git clone -b master --depth 1 https://github.com/vatesfr/xen-orchestra/ && \
    cd xen-orchestra && \
    yarn config set network-timeout 300000 && \
    yarn && \
    yarn build && \
    yarn run turbo run build --filter @xen-orchestra/web && \
    # Install plugins by creating symlinks to xo-server/node_modules
    # Excluding core packages and test plugins
    find ./packages/ -maxdepth 1 -mindepth 1 \
    -not -name "xo-server" \
    -not -name "xo-web" \
    -not -name "xo-server-cloud" \
    -not -name "xo-server-test" \
    -not -name "xo-server-test-plugin" \
    -exec ln -s {} ./packages/xo-server/node_modules \; && \
    # Remove dev dependencies to reduce image size
    npm prune --omit=dev --force

# Stage 2: Build VHDI tools for VHD image support
FROM builder AS build-libvhdi
WORKDIR /home/node
# Clone and compile libvhdi from source
RUN git clone https://github.com/libyal/libvhdi.git && \
    cd libvhdi && \
    ./synclibs.sh && \
    ./autogen.sh && \
    ./configure && \
    make && \
    make install

# Stage 3: Final runtime container
FROM alpine:3.19

# Build arguments for version labeling
ARG VERSION=latest
ARG XOSERVER=latest
ARG XOWEB=latest
LABEL version=$VERSION \
      xo-server=$XOSERVER \
      xo-web=$XOWEB \
      maintainer="Xen Orchestra Community Edition" \
      description="Xen Orchestra provides web UI and management for XenServer/XCP-ng"

# Create non-root user early but switch to it only before running the application
RUN adduser -D -h /home/node node

# Environment configuration
ENV USER=node \
    XOA_PLAN=5 \
    DEBUG=xo:main \
    XO_HTTP_LISTEN_PORT="80" \
    NODE_ENV="production"

# Install runtime dependencies
# - su-exec: for privilege dropping
# - bash: required shell
# - util-linux: for mount operations
# - nfs-utils, cifs-utils: for network filesystems
# - lvm2: for LVM support
# - fuse, fuse3: for filesystem in userspace
# - openssl: for HTTPS support
# - ntfs-3g: for NTFS filesystem support
RUN apk add --no-cache \
    su-exec bash util-linux nfs-utils \
    lvm2 fuse fuse3 gettext cifs-utils \
    openssl ntfs-3g && \
    mkdir -p /home/node /storage /etc/xo-server && \
    chown -R node:node /home/node /storage /etc/xo-server

# Expose the HTTP port
EXPOSE 80

WORKDIR /home/node

# Copy built artifacts from previous stages with proper ownership
# - XO application files
COPY --from=build-xoa --chown=node:node /home/node/xen-orchestra /home/node/xen-orchestra

# - Node.js binary and required libraries
COPY --from=build-xoa /usr/local/bin/node /usr/bin/
COPY --from=build-xoa /usr/lib/libgcc* /usr/lib/libstdc* /usr/lib/

# - VHDI tools and libraries
COPY --from=build-libvhdi /usr/local/bin/vhdimount /usr/local/bin/vhdiinfo /usr/local/bin/
COPY --from=build-libvhdi /usr/local/lib/libvhdi* /usr/local/lib/

# Set working directory for the application
WORKDIR /home/node/xen-orchestra/packages/xo-server/

# Add healthcheck
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:${XO_HTTP_LISTEN_PORT} || exit 1

# Switch to non-root user for security
USER node

# Start the application
CMD ["yarn", "start"]