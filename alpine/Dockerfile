## Use same builder image for XOA & libvhdi
FROM node:20.18-alpine3.21 as builder
RUN apk add --no-cache git python3 g++ make bash fuse fuse-dev fuse3 fuse3-dev curl libc6-compat \
 automake autoconf libtool gettext-dev pkgconf fuse-dev fuse

## XOA Builder
FROM builder as build-xoa

WORKDIR /home/node
#fix error - url loader
RUN npm install url-loader --save-dev
# Clone git and remove .git
RUN git clone -b master --depth 1 https://github.com/vatesfr/xen-orchestra/
# patches
COPY patches /home/node/xen-orchestra/patches
RUN cd /home/node/xen-orchestra \
    && git apply patches/gh_issue_redirect.diff \
    && rm -rf /home/node/xen-orchestra/.git /home/node/xen-orchestra/patches
# build
WORKDIR /home/node/xen-orchestra
RUN yarn config set network-timeout 300000
# Disable Turbo telemetry
RUN yarn turbo telemetry disable
RUN yarn
RUN yarn build
# Add XOv6 UI
RUN yarn run turbo run build --filter @xen-orchestra/web

# Install plugins
RUN find ./packages/ -maxdepth 1 -mindepth 1 -not -name "xo-server" -not -name "xo-web" -not -name "xo-server-cloud" -not -name "xo-server-test" -not -name "xo-server-test-plugin" -exec ln -s {} ./packages/xo-server/node_modules \;

##  ~ Experimental ~
# clean node_modules steps
WORKDIR /home/node/xen-orchestra
# from npm doc remove extraneous modules for production (as before we build with
# the dev dependencies it remove a lot! after this step the size is around 200Mo)
RUN npm prune --omit=dev --force
## 

# VHDIMOUNT support
FROM builder as build-libvhdi

WORKDIR /home/node
RUN git clone https://github.com/libyal/libvhdi.git

WORKDIR /home/node/libvhdi
RUN ./synclibs.sh
RUN ./autogen.sh
RUN ./configure
RUN make && make install

# Runner container
FROM alpine:3.21

ARG VERSION=latest
ARG XOSERVER=latest
ARG XOWEB=latest

LABEL version=$VERSION xo-server=$XOSERVER xo-web=$XOWEB

ENV USER=node
ENV XOA_PLAN=5
ENV DEBUG=xo:main

## Add a user
RUN mkdir -p /home/node

WORKDIR /home/node

RUN apk add --no-cache \
  su-exec \
  bash \
  util-linux \
  nfs-utils \
  lvm2 \
  fuse \
  fuse3 \
  gettext \
  cifs-utils \
  openssl \
  ntfs-3g

RUN mkdir -p /storage /etc/xo-server

# Copy our App from the build container
COPY --from=build-xoa /home/node/xen-orchestra /home/node/xen-orchestra

# Only copy over the node pieces we need from the above image
COPY --from=build-xoa /usr/local/bin/node /usr/bin/
COPY --from=build-xoa /usr/lib/libgcc* /usr/lib/libstdc* /usr/lib/

# Get libvhdi
COPY --from=build-libvhdi /usr/local/bin/vhdimount /usr/local/bin/vhdiinfo  /usr/local/bin/
COPY --from=build-libvhdi /usr/local/lib/libvhdi* /usr/local/lib/

# Environment vars to control config
ENV XO_HTTP_LISTEN_PORT="80"
# List of configurable env vars:
#    XO_HTTP_LISTEN_PORT="80" \
#    XO_HTTP_REDIRECTTOHTTPS="false" \
#    XO_HTTPS_LISTEN_PORT="443" \
#    XO_HTTPS_LISTEN_CERT="./certificate.pem" \
#    XO_HTTPS_LISTEN_KEY="./key.pem" \
#    XO_HTTPS_LISTEN_AUTOCERT="true" \
#    XO_HTTPS_LISTEN_DHPARAM="./dhparam.pem

# Run the App
WORKDIR /home/node/xen-orchestra/packages/xo-server/
CMD ["node", "dist/cli.mjs"]