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
ENV USER=node

# Clone and build XO application
# - Install url-loader for webpack
# - Clone XO repository
# - Build core and web interface
# - Link plugins to xo-server
# - Remove development dependencies
RUN npm install url-loader --save-dev && \
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
LABEL version=$VERSION xo-server=$XOSERVER xo-web=$XOWEB

# Environment configuration
ENV USER=node \
    XOA_PLAN=5 \
    DEBUG=xo:main \
    XO_HTTP_LISTEN_PORT="80"

# Install runtime dependencies and setup user
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
    adduser -D -h /home/node node && \
    chown -R node:node /home/node

WORKDIR /home/node

# Copy built artifacts from previous stages
# - XO application files
# - Node.js binary and required libraries
# - VHDI tools and libraries
COPY --from=build-xoa /home/node/xen-orchestra /home/node/xen-orchestra
COPY --from=build-xoa /usr/local/bin/node /usr/bin/
COPY --from=build-xoa /usr/lib/libgcc* /usr/lib/libstdc* /usr/lib/
COPY --from=build-libvhdi /usr/local/bin/vhdimount /usr/local/bin/vhdiinfo /usr/local/bin/
COPY --from=build-libvhdi /usr/local/lib/libvhdi* /usr/local/lib/

# Set working directory and user for running the application
WORKDIR /home/node/xen-orchestra/packages/xo-server/

# Rename sample config to config.toml if not already present
RUN mv sample.config.toml config.toml

USER node
CMD ["yarn", "start"]