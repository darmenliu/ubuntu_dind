FROM ubuntu:18.04

RUN  if [ "x$http_proxy" != "x" ]; then \ 
         echo "Acquire::http::Proxy \"$http_proxy\";" >> /etc/apt/apt.conf; \
     fi; \
     if [ "x$https_proxy" != "x" ]; then \
         echo "Acquire::http::Proxy \"$https_proxy\";" >> /etc/apt/apt.conf; \
     fi

RUN apt update && apt-get install ca-certificates openssh-client wget -y && apt-get clean;

RUN set -eux \
    && apt-get install \
        btrfs-progs \
        e2fsprogs \
        iptables \
        openssl \
        uidmap \
        xfsprogs \
        xz-utils \
        pigz \
        kmod \
        iproute2 \
        net-tools \
        zfsutils-linux -y && apt-get clean;

# set up subuid/subgid so that "--userns-remap=default" works out-of-the-box
RUN set -x \
    && addgroup --system dockremap \
    && adduser --system dockremap \
    && adduser dockremap dockremap \
    && echo 'dockremap:165536:65536' >> /etc/subuid \
    && echo 'dockremap:165536:65536' >> /etc/subgid;

# set up nsswitch.conf for Go's "netgo" implementation (which Docker explicitly uses)
# - https://github.com/docker/docker-ce/blob/v17.09.0-ce/components/engine/hack/make.sh#L149
# - https://github.com/golang/go/blob/go1.9.1/src/net/conf.go#L194-L275
# - docker run --rm debian:stretch grep '^hosts:' /etc/nsswitch.conf
RUN echo 'hosts: files dns' >> /etc/nsswitch.conf

ENV DOCKER_CHANNEL stable
ENV DOCKER_VERSION 19.03.8
# TODO ENV DOCKER_SHA256
# https://github.com/docker/docker-ce/blob/5b073ee2cf564edee5adca05eee574142f7627bb/components/packaging/static/hash_files !!
# (no SHA file artifacts on download.docker.com yet as of 2017-06-07 though)

RUN set -eux; \
	\
# this "case" statement is generated via "update.sh"
	arch="$(arch)"; \
	case "$arch" in \
# amd64
		x86_64) dockerArch='x86_64' ;; \
# arm32v6
		armhf) dockerArch='armel' ;; \
# arm32v7
		armv7) dockerArch='armhf' ;; \
# arm64v8
		aarch64) dockerArch='aarch64' ;; \
		*) echo >&2 "error: unsupported architecture ($arch)"; exit 1 ;;\
	esac; \
	\
	if ! wget -O docker.tgz "https://download.docker.com/linux/static/${DOCKER_CHANNEL}/${dockerArch}/docker-${DOCKER_VERSION}.tgz"; then \
		echo >&2 "error: failed to download 'docker-${DOCKER_VERSION}' from '${DOCKER_CHANNEL}' for '${dockerArch}'"; \
		exit 1; \
	fi; \
	\
	tar --extract \
		--file docker.tgz \
		--strip-components 1 \
		--directory /usr/local/bin/ \
	; \
	rm docker.tgz; \
	\
	dockerd --version; \
	docker --version

# https://github.com/docker/docker/tree/master/hack/dind
ENV DIND_COMMIT 37498f009d8bf25fbb6199e8ccd34bed84f2874b

RUN set -eux; \
    wget -O /usr/local/bin/dind "https://raw.githubusercontent.com/docker/docker/${DIND_COMMIT}/hack/dind"; \
    chmod +x /usr/local/bin/dind

COPY dockerd-entrypoint.sh /usr/local/bin/
COPY modprobe.sh /usr/local/bin/modprobe
COPY docker-entrypoint.sh /usr/local/bin/
RUN  chmod 755 /usr/local/bin/dockerd-entrypoint.sh && \
     chmod 755 /usr/local/bin/modprobe && \
     chmod 755 /usr/local/bin/docker-entrypoint.sh

# https://github.com/docker-library/docker/pull/166
#   dockerd-entrypoint.sh uses DOCKER_TLS_CERTDIR for auto-generating TLS certificates
#   docker-entrypoint.sh uses DOCKER_TLS_CERTDIR for auto-setting DOCKER_TLS_VERIFY and DOCKER_CERT_PATH
# (For this to work, at least the "client" subdirectory of this path needs to be shared between the client and server containers via a volume, "docker cp", or other means of data sharing.)
ENV DOCKER_TLS_CERTDIR=/certs
# also, ensure the directory pre-exists and has wide enough permissions for "dockerd-entrypoint.sh" to create subdirectories, even when run in "rootless" mode
RUN mkdir /certs /certs/client && chmod 1777 /certs /certs/client
# (doing both /certs and /certs/client so that if Docker does a "copy-up" into a volume defined on /certs/client, it will "do the right thing" by default in a way that still works for rootless users)

VOLUME /var/lib/docker
EXPOSE 2375 2376

ENTRYPOINT ["/usr/local/bin/dockerd-entrypoint.sh"]
CMD []
