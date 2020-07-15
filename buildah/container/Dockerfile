# runc
FROM golang:alpine AS runc
ARG RUNC_VERSION=v1.0.0-rc8
RUN set -eux; \
	apk add --no-cache --virtual .build-deps gcc musl-dev libseccomp-dev make git bash; \
	git clone --branch ${RUNC_VERSION} https://github.com/opencontainers/runc src/github.com/opencontainers/runc; \
	cd src/github.com/opencontainers/runc; \
	make static BUILDTAGS='seccomp selinux ambient'; \
	mv runc /usr/local/bin/runc; \
	rm -rf $GOPATH/src/github.com/opencontainers/runc; \
	apk del --purge .build-deps


# podman build base
FROM golang:alpine AS podmanbuildbase
RUN apk add --update --no-cache git make gcc pkgconf musl-dev btrfs-progs btrfs-progs-dev libassuan-dev lvm2-dev device-mapper glib-static libc-dev gpgme-dev protobuf-dev protobuf-c-dev libseccomp-dev libselinux-dev ostree-dev openssl iptables bash go-md2man


# podman
FROM podmanbuildbase AS podman
ARG PODMAN_VERSION=v1.0.0
RUN git clone --branch ${PODMAN_VERSION} https://github.com/containers/libpod src/github.com/containers/libpod
WORKDIR $GOPATH/src/github.com/containers/libpod
RUN make install.tools
RUN set -eux; \
	make LDFLAGS="-w -extldflags '-static'" BUILDTAGS='seccomp selinux varlink exclude_graphdriver_devicemapper containers_image_ostree_stub containers_image_openpgp'; \
	mv bin/podman /usr/local/bin/podman


# conmon
FROM podmanbuildbase AS conmon
ARG CONMON_VERSION=v0.2.0
WORKDIR $GOPATH
RUN git clone --branch ${CONMON_VERSION} https://github.com/containers/conmon.git /conmon
WORKDIR /conmon
RUN set -eux; \
	rm /usr/lib/libglib-2.0.so* /usr/lib/libintl.so*; \
	make CFLAGS='-std=c99 -Os -Wall -Wextra -Werror -static'; \
	install -D -m 755 bin/conmon /usr/libexec/podman/conmon


# CNI plugins
FROM podmanbuildbase AS cniplugins
ARG CNI_VERSION=0.6.0
RUN set -eux; \
	mkdir -p "${GOPATH}/src/github.com/containernetworking"; \
	wget -O - "https://github.com/containernetworking/plugins/archive/v${CNI_VERSION}.tar.gz" | tar -xzf - -C /tmp; \
	mv "/tmp/plugins-${CNI_VERSION}" "${GOPATH}/src/github.com/containernetworking/plugins"; \
	for TYPE in main ipam meta; do \
		for CNIPLUGIN in `ls ${GOPATH}/src/github.com/containernetworking/plugins/plugins/$TYPE`; do \
			go build -o /usr/libexec/cni/$CNIPLUGIN -ldflags "-extldflags '-static'" github.com/containernetworking/plugins/plugins/$TYPE/$CNIPLUGIN; \
		done \
	done


# slirp4netns
FROM podmanbuildbase AS slirp4netns
RUN apk add --update --no-cache git autoconf automake linux-headers
ARG SLIRP4NETNS_VERSION=v0.3.0
WORKDIR /
RUN git clone --branch $SLIRP4NETNS_VERSION https://github.com/rootless-containers/slirp4netns.git
WORKDIR /slirp4netns
RUN set -eux; \
	./autogen.sh \
	&& LDFLAGS=-static ./configure --prefix=/usr \
	&& make


# fuse-overlay (derived from https://github.com/containers/fuse-overlayfs/blob/master/Dockerfile.static)
FROM podmanbuildbase AS fuse-overlayfs
RUN apk add --update --no-cache automake autoconf meson ninja clang eudev-dev
ARG LIBFUSE_VERSION=fuse-3.5.0
RUN git clone --branch=${LIBFUSE_VERSION} https://github.com/libfuse/libfuse /libfuse
WORKDIR /libfuse
RUN set -eux; \
	mkdir build; \
	cd build; \
	LDFLAGS="-lpthread" meson --prefix /usr -D default_library=static ..; \
	ninja; \
	ninja install
# v0.3 + musl compat fix
ARG FUSEOVERLAYFS_VERSION=2cbd1c4a2d3ab06c4e39276363c74c6a9c62c0fe
RUN set -eux; \
	git clone https://github.com/containers/fuse-overlayfs /fuse-overlayfs; \
	cd /fuse-overlayfs; \
	git checkout "${FUSEOVERLAYFS_VERSION}"; \
	sh autogen.sh; \
	LIBS="-ldl" LDFLAGS="-static" ./configure --prefix /usr; \
	make; \
	make install; \
	fuse-overlayfs --help >/dev/null


# skopeo
FROM podmanbuildbase AS skopeo
ARG SKOPEO_VERSION=v0.1.36
RUN git clone --branch ${SKOPEO_VERSION} https://github.com/containers/skopeo $GOPATH/src/github.com/containers/skopeo
WORKDIR $GOPATH/src/github.com/containers/skopeo
RUN go build -ldflags "-extldflags '-static'" -tags "exclude_graphdriver_devicemapper containers_image_ostree_stub containers_image_openpgp" \
	github.com/containers/skopeo/cmd/skopeo && \
	mv skopeo /usr/local/bin/skopeo


# buildah
FROM podmanbuildbase AS buildah
ARG BUILDAH_VERSION=v1.8.3
RUN git clone --branch ${BUILDAH_VERSION} https://github.com/containers/buildah $GOPATH/src/github.com/containers/buildah
WORKDIR $GOPATH/src/github.com/containers/buildah
RUN make static && mv buildah.static /usr/local/bin/buildah


FROM alpine:3.9
# Add gosu for easy step-down from root
ARG GOSU_VERSION=1.11
RUN set -eux; \
	wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-amd64"; \
	chmod +x /usr/local/bin/gosu; \
	gosu nobody true
# Install iptables & new-uidmap
RUN apk add --no-cache ca-certificates iptables ip6tables shadow-uidmap
# Copy binaries from other images
COPY --from=runc   /usr/local/bin/runc   /usr/local/bin/runc
COPY --from=podman /usr/local/bin/podman /usr/local/bin/podman
COPY --from=conmon /usr/libexec/podman/conmon /usr/libexec/podman/conmon
COPY --from=cniplugins /usr/libexec/cni /usr/libexec/cni
COPY --from=skopeo /usr/local/bin/skopeo /usr/local/bin/skopeo
COPY --from=fuse-overlayfs /usr/bin/fuse-overlayfs /usr/local/bin/fuse-overlayfs
COPY --from=slirp4netns /slirp4netns/slirp4netns /usr/local/bin/slirp4netns
COPY --from=buildah /usr/local/bin/buildah /usr/local/bin/buildah
RUN set -eux; \
	adduser -D podman -h /podman -u 9000; \
	echo 'podman:900000:65536' > /etc/subuid; \
	echo 'podman:900000:65536' > /etc/subgid; \
	ln -s /usr/local/bin/podman /usr/bin/docker; \
	mkdir -pm 775 /etc/containers /podman/.config/containers /etc/cni/net.d /podman/.local/share/containers/storage/libpod; \
	chown -R root:podman /podman; \
	wget -O /etc/containers/registries.conf https://raw.githubusercontent.com/projectatomic/registries/master/registries.fedora; \
	wget -O /etc/containers/policy.json     https://raw.githubusercontent.com/containers/skopeo/master/default-policy.json; \
	wget -O /etc/cni/net.d/99-bridge.conflist https://raw.githubusercontent.com/containers/libpod/master/cni/87-podman-bridge.conflist; \
	runc --help >/dev/null; \
	podman --help >/dev/null; \
	/usr/libexec/podman/conmon --help >/dev/null; \
	slirp4netns --help >/dev/null; \
	fuse-overlayfs --help >/dev/null;
COPY entrypoint.sh /
ENTRYPOINT [ "/entrypoint.sh" ]
VOLUME /podman/.local/share/containers/storage
WORKDIR /podman
ENV HOME=/podman
