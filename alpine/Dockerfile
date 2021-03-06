FROM node:14.5.0-alpine3.12 as build

WORKDIR /home/node

RUN apk add --no-cache git python2 g++ make bash

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
RUN cd /home/node/xen-orchestra &&\
    yarn config set network-timeout 300000 &&\
    yarn &&\
    yarn build

# plugins
COPY link_plugins.sh /home/node/xen-orchestra/packages/xo-server/link_plugins.sh
RUN /home/node/xen-orchestra/packages/xo-server/link_plugins.sh

# VHDIMOUNT support
FROM alpine:3.12 as build-libvhdi

WORKDIR /home/node
RUN apk add --no-cache git g++ make bash automake autoconf libtool gettext-dev pkgconf fuse-dev fuse

RUN git clone https://github.com/libyal/libvhdi.git

RUN cd libvhdi && ./synclibs.sh && \
    ./autogen.sh && \
    ./configure && \
    make && \
    make install

# Runner container
FROM alpine:3.12

ARG VERSION=latest
ARG XOSERVER=latest
ARG XOWEB=latest

LABEL version=$VERSION xo-server=$XOSERVER xo-web=$XOWEB

ENV USER=node \
    XOA_PLAN=5 \
    DEBUG=xo:main

## Add a user
RUN mkdir -p /home/node

WORKDIR /home/node

RUN apk add --no-cache \
  tini \
  su-exec \
  bash \
  util-linux \
  nfs-utils \
  lvm2 \
  fuse \
  gettext \
  cifs-utils \
  openssl

RUN mkdir -p /storage

# configurations
COPY config.toml /etc/xo-server/config.toml

# Copy our App from the build container
COPY --from=build /home/node/xen-orchestra /home/node/xen-orchestra

# Only copy over the node pieces we need from the above image
#fix node - new path - version 14.5
COPY --from=build /usr/local/bin/node /usr/bin/
COPY --from=build /usr/lib/libgcc* /usr/lib/libstdc* /usr/lib/

# Get libvhdi
COPY --from=build-libvhdi /usr/local/bin/vhdimount /usr/local/bin/vhdiinfo  /usr/local/bin/
COPY --from=build-libvhdi /usr/local/lib/libvhdi* /usr/local/lib/

# Healthcheck
COPY healthcheck.js /usr/local/share/healthcheck.js
HEALTHCHECK --interval=1m --timeout=12s --start-period=30s \  
 CMD /usr/bin/node /usr/local/share/healthcheck.js

# Run the App
COPY entrypoint.sh /entrypoint.sh
EXPOSE 8000
WORKDIR /home/node/xen-orchestra/packages/xo-server/
ENTRYPOINT ["/sbin/tini", "--", "/entrypoint.sh" ]
CMD ["./bin/xo-server"]
