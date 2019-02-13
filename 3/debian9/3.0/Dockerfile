FROM gcr.io/google-appengine/debian9:latest

# explicitly set user/group IDs
RUN groupadd -r cassandra --gid=999 && useradd -r -g cassandra --uid=999 cassandra

RUN set -ex; \
	apt-get update; \
	if ! command -v gpg > /dev/null; then \
		apt-get install -y --no-install-recommends \
			gnupg \
			dirmngr \
		; \
	fi ; \
	if ! command -v free > /dev/null; then \
		apt-get install -y --no-install-recommends \
			procps \
		; \
	fi ; \
	rm -rf /var/lib/apt/lists/*;

# grab gosu for easy step-down from root
ENV GOSU_VERSION 1.10
ENV GOSU_GPG B42F6819007F00F88E364FD4036A9C25BF357DD4

RUN set -x \
	&& apt-get update && apt-get install -y --no-install-recommends ca-certificates wget && rm -rf /var/lib/apt/lists/* \
	&& wget -q -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture)" \
	&& wget -q -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture).asc" \
	&& wget -q -O /usr/local/src/gosu.tar.gz "https://github.com/tianon/gosu/archive/$GOSU_VERSION.tar.gz" \
	&& export GNUPGHOME="$(mktemp -d)" \
	&& found='' && \
	for server in \
		pool.sks-keyservers.net \
		na.pool.sks-keyservers.net \
		eu.pool.sks-keyservers.net \
		oc.pool.sks-keyservers.net \
		ha.pool.sks-keyservers.net \
		hkp://p80.pool.sks-keyservers.net:80 \
		hkp://keyserver.ubuntu.com:80 \
		pgp.mit.edu \
	; do \
		gpg --no-tty --keyserver $server --recv-keys $GOSU_GPG \
			&& found=yes && break; \
	done; \
	test -n "$found" \
	&& gpg --no-tty --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu \
	&& rm -rf "$GNUPGHOME" /usr/local/bin/gosu.asc \
	&& chmod +x /usr/local/bin/gosu \
	&& gosu nobody true \
	&& apt-get purge -y --auto-remove ca-certificates wget

# solves warning: "jemalloc shared library could not be preloaded to speed up memory allocations"
RUN apt-get update && apt-get install -y --no-install-recommends libjemalloc1 && rm -rf /var/lib/apt/lists/*

# https://wiki.apache.org/cassandra/DebianPackaging#Adding_Repository_Keys
ENV GPG_KEYS 514A2AD631A57A16DD0047EC749D6EEC0353B12C A26E528B271F19B9E5D8E19EA278B781FE4B2BDA

RUN set -ex; \
	export GNUPGHOME="$(mktemp -d)"; \
	for key in $GPG_KEYS; do \
		found='' && \
	for server in \
		pool.sks-keyservers.net \
		na.pool.sks-keyservers.net \
		eu.pool.sks-keyservers.net \
		oc.pool.sks-keyservers.net \
		ha.pool.sks-keyservers.net \
		hkp://p80.pool.sks-keyservers.net:80 \
		hkp://keyserver.ubuntu.com:80 \
		pgp.mit.edu \
	; do \
		gpg --no-tty --keyserver $server --recv-keys "${key}" \
			&& found=yes && break; \
	done; \
	test -n "$found"; \
	done; \
	gpg --no-tty --export $GPG_KEYS > /etc/apt/trusted.gpg.d/cassandra.gpg; \
	rm -rf "$GNUPGHOME"; \
	apt-key list

RUN echo 3.0.18 | sed 's-\([^.]\+\)\.\([^.]\+\).*$-deb http://www.apache.org/dist/cassandra/debian \1\2x main-' >> /etc/apt/sources.list.d/cassandra.list

ENV CASSANDRA_VERSION 3.0.18

RUN set -x; \
	if [ "${CASSANDRA_VERSION%%.*}" = "3" ]; then \
		echo deb http://httpredir.debian.org/debian jessie-backports main > /etc/apt/sources.list.d/backports.list ; \
		apt-get update && apt-get upgrade; \
		apt-get -t jessie-backports install -y openjdk-8-jre-headless; \
		apt-get install -y ntp; \
	fi

RUN apt-get update \
	&& apt-get install -y cassandra="$CASSANDRA_VERSION" \
	&& rm -rf /var/lib/apt/lists/*

# https://issues.apache.org/jira/browse/CASSANDRA-11661
RUN sed -ri 's/^(JVM_PATCH_VERSION)=.*/\1=25/' /etc/cassandra/cassandra-env.sh

ENV CASSANDRA_CONFIG /etc/cassandra

COPY docker-entrypoint.sh /docker-entrypoint.sh
ENTRYPOINT ["/docker-entrypoint.sh"]

RUN mkdir -p /var/lib/cassandra "$CASSANDRA_CONFIG" \
	&& chown -R cassandra:cassandra /var/lib/cassandra "$CASSANDRA_CONFIG" \
	&& chmod 777 /var/lib/cassandra "$CASSANDRA_CONFIG"
VOLUME /var/lib/cassandra

# 7000: intra-node communication
# 7001: TLS intra-node communication
# 7199: JMX
# 9042: CQL
# 9160: thrift service
EXPOSE 7000 7001 7199 9042 9160
CMD ["cassandra", "-f"]

