#
# NOTE: THIS DOCKERFILE IS GENERATED VIA "apply-templates.sh"
#
# PLEASE DO NOT EDIT IT DIRECTLY.
#

#
# HUGE CAVEAT:
#
#   This is nowhere near a supported way to run Docker Desktop.
#   This is a huge ugly hack that barely works for the way I play with it.
#

FROM rust:1.75-slim-bookworm AS virtiofsd

RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		libcap-ng-dev \
		libseccomp-dev \
		patch \
		wget ca-certificates \
	; \
	rm -rf /var/lib/apt/lists/*

WORKDIR /virtiofsd

# see the end of this Dockerfile -- this version needs to match what's shipped with the relevant version of Docker Desktop
ENV VIRTIOFS_VERSION 1.11.1
# https://gitlab.com/virtio-fs/virtiofsd/-/tags

RUN set -eux; \
	wget -O virtiofsd.tar.gz "https://gitlab.com/virtio-fs/virtiofsd/-/archive/v${VIRTIOFS_VERSION}/virtiofsd-v${VIRTIOFS_VERSION}.tar.gz"; \
	tar -xvf virtiofsd.tar.gz --strip-components=1; \
	rm virtiofsd.tar.gz

# it appears that the DD builds apply https://gitlab.com/virtio-fs/virtiofsd/-/merge_requests/237 👀 ("error: unexpected argument '--translate-uid' found")
COPY virtiofsd-237.patch ./
RUN patch --strip=1 --input=virtiofsd-237.patch

COPY virtiofsd.patch ./
RUN patch --strip=1 --input=virtiofsd.patch

RUN cargo build --release

# https://docs.docker.com/desktop/linux/
FROM debian:bookworm-slim

# https://docs.docker.com/desktop/release-notes/
# https://docs.docker.com/desktop/linux/install/
ENV DOCKER_DESKTOP_VERSION 4.35.0
ENV DOCKER_DESKTOP_URL https://desktop.docker.com/linux/main/amd64/172550/docker-desktop-amd64.deb
ENV DOCKER_DESKTOP_SHA256 c93573f1d1c1e83b6460e5666b29283a085a529c2fe8b7e13d1a8955eb0be1ce

RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		ca-certificates \
# https://docs.docker.com/desktop/linux/#credentials-management
		pass \
	; \
	savedAptMark="$(apt-mark showmanual)"; \
	apt-get install -y --no-install-recommends \
		gnupg \
		wget \
	; \
	apt-mark auto '.*' > /dev/null; \
	apt-mark manual $savedAptMark > /dev/null; \
	\
	GNUPGHOME="$(mktemp -d)"; export GNUPGHOME; \
	gpg --keyserver keyserver.ubuntu.com --recv-keys 9DC858229FC7DD38854AE2D88D81803C0EBFCD88; \
	mkdir -p /usr/local/share/keyrings; \
	gpg --export --armor 9DC858229FC7DD38854AE2D88D81803C0EBFCD88 > /usr/local/share/keyrings/docker-archive.gpg.asc; \
	gpgconf --kill all; \
	rm -rf "$GNUPGHOME"; \
	\
	. /etc/os-release; \
	echo "deb [ arch=amd64 signed-by=/usr/local/share/keyrings/docker-archive.gpg.asc ] https://download.docker.com/linux/debian $VERSION_CODENAME stable" > /etc/apt/sources.list.d/docker.list; \
	\
	apt-get update; \
	\
	wget -O docker-desktop.deb --progress=dot:giga "$DOCKER_DESKTOP_URL"; \
	echo "$DOCKER_DESKTOP_SHA256 *docker-desktop.deb" | sha256sum --strict --check -; \
	\
	apt-get install -y --no-install-recommends ./docker-desktop.deb; \
	rm docker-desktop.deb; \
	\
	apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
	rm -rf /var/lib/apt/lists/*
# smoke test
RUN set -eux; \
	find /opt/docker-desktop -type f -executable -exec bash -Eeuo pipefail -xc 'ldd "$@" || :' -- '{}' + > /tmp/ldd.txt; \
	! grep 'not found' /tmp/ldd.txt; \
	rm /tmp/ldd.txt

ENV PATH /opt/docker-desktop/bin:$PATH

# virtiofsd defaults to "--sandbox=namespace" (which doesn't work well in a container) and "--sandbox=chroot" or "--sandbox=none" are only allowed for root, so I provide a patched build of virtiofsd (matching the version that ships with Docker Desktop)
COPY --from=virtiofsd /virtiofsd/target/release/virtiofsd /opt/docker-desktop/bin/virtiofsd.tianon
RUN set -eux; \
	virtiofsd="$(command -v virtiofsd)"; \
	[ "$virtiofsd" = '/opt/docker-desktop/bin/virtiofsd' ]; \
	\
	before="$(virtiofsd --version 2>&1)"; \
	dpkg-divert --add --rename --divert /opt/docker-desktop/bin/virtiofsd.orig /opt/docker-desktop/bin/virtiofsd; \
	ln -svfT virtiofsd.tianon /opt/docker-desktop/bin/virtiofsd; \
	after="$(virtiofsd --version 2>&1)"; \
	[ "$before" = "$after" ]; \
	\
# divert QEMU's "virtiofsd" to make sure it doesn't get used accidentally
	[ -s /usr/lib/qemu/virtiofsd ]; \
	dpkg-divert --add --rename --divert /usr/lib/qemu/virtiofsd.orig /usr/lib/qemu/virtiofsd

RUN set -eux; \
# setting up ID maps for virtiofsd namespace: retrieving subordinate UID range: no range for <USER> found in /etc/subuid
	echo '1000:100000:65536' > /etc/subuid; \
	echo '1000:100000:65536' > /etc/subgid; \
# newuidmap: write to uid_map failed: Operation not permitted
	ln -svfT /bin/true /usr/local/bin/newuidmap; \
	ln -svfT /bin/true /usr/local/bin/newgidmap
# TODO come up with something less ... hacky?

#CMD ["docker-desktop"]
CMD ["com.docker.backend", "--with-frontend"]
