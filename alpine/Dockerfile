## Use same builder image for XOA & libvhdi
FROM node:20.18-alpine3.21 AS builder
RUN apk add --no-cache \
    git python3 g++ make bash \
    fuse fuse3 fuse-dev fuse3-dev \
    curl libc6-compat automake autoconf \
    libtool gettext-dev pkgconf

## XOA Builder
FROM builder AS build-xoa
WORKDIR /home/node

# Build XO application
RUN npm install url-loader --save-dev && \
    git clone -b master --depth 1 https://github.com/vatesfr/xen-orchestra/ && \
    cd xen-orchestra && \
    yarn config set network-timeout 300000 && \
    yarn && \
    yarn build && \
    yarn run turbo run build --filter @xen-orchestra/web && \
    # Install plugins
    find ./packages/ -maxdepth 1 -mindepth 1 \
    -not -name "xo-server" \
    -not -name "xo-web" \
    -not -name "xo-server-cloud" \
    -not -name "xo-server-test" \
    -not -name "xo-server-test-plugin" \
    -exec ln -s {} ./packages/xo-server/node_modules \; && \
    # Clean up
    npm prune --omit=dev --force

# VHDIMOUNT support
FROM builder AS build-libvhdi
WORKDIR /home/node
RUN git clone https://github.com/libyal/libvhdi.git && \
    cd libvhdi && \
    ./synclibs.sh && \
    ./autogen.sh && \
    ./configure && \
    make && \
    make install

# Runner container
FROM alpine:3.19

ARG VERSION=latest
ARG XOSERVER=latest
ARG XOWEB=latest
LABEL version=$VERSION xo-server=$XOSERVER xo-web=$XOWEB

ENV USER=node \
    XOA_PLAN=5 \
    DEBUG=xo:main \
    XO_HTTP_LISTEN_PORT="80"

# Add user and install dependencies
RUN apk add --no-cache \
    su-exec bash util-linux nfs-utils \
    lvm2 fuse fuse3 gettext cifs-utils \
    openssl ntfs-3g && \
    mkdir -p /home/node /storage /etc/xo-server && \
    adduser -D -h /home/node node && \
    chown -R node:node /home/node

WORKDIR /home/node

# Copy artifacts from build stages
COPY --from=build-xoa /home/node/xen-orchestra /home/node/xen-orchestra
COPY --from=build-xoa /usr/local/bin/node /usr/bin/
COPY --from=build-xoa /usr/lib/libgcc* /usr/lib/libstdc* /usr/lib/
COPY --from=build-libvhdi /usr/local/bin/vhdimount /usr/local/bin/vhdiinfo /usr/local/bin/
COPY --from=build-libvhdi /usr/local/lib/libvhdi* /usr/local/lib/

WORKDIR /home/node/xen-orchestra/packages/xo-server/
USER node
CMD ["yarn", "start"]