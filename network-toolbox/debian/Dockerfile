FROM debian:bookworm-slim

RUN set -eux; \
# dangit, wireshark-common!  *shakes fist*
	export DEBIAN_FRONTEND='noninteractive'; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		bash-completion \
		ca-certificates \
		curl \
		dnsutils \
		fio \
		gawk \
		gnupg \
		iftop \
		inetutils-ping \
		ipcalc \
		iperf \
		iproute2 \
		iptables \
		jq \
		libfcgi-bin \
		mtr-tiny \
		nbtscan \
		net-tools \
		netcat-openbsd \
		nftables \
		nmap \
		ntpdate \
		openssh-client \
		procps \
		rsync \
		siege \
		socat \
		ssh-audit \
		traceroute \
		tshark \
		wget \
		whois \
		wireguard-tools \
# git is large on Debian (~20MiB), so isn't included
	; \
	rm -rf /var/lib/apt/lists/*

CMD ["bash", "--login", "-i"]
