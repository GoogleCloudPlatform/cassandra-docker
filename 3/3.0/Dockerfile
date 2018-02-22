FROM gcr.io/google-appengine/debian9:latest

# explicitly set user/group IDs
RUN groupadd -r cassandra --gid=999 && useradd -r -g cassandra --uid=999 cassandra

# grab gosu for easy step-down from root
ENV GOSU_VERSION 1.10
RUN set -ex; \
        if ! command -v gpg > /dev/null; then \
                apt-get update; \
                apt-get install -y --no-install-recommends \
                        gnupg \
                        dirmngr \
                        procps \
                        ntp \
                ; \
                rm -rf /var/lib/apt/lists/*; \
        fi
RUN set -x \
	&& apt-get update && apt-get install -y --no-install-recommends ca-certificates wget && rm -rf /var/lib/apt/lists/* \
	&& wget -q -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture)" \
	&& wget -q -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture).asc" \
	&& wget -q -O /usr/local/src/gosu.tar.gz "https://github.com/tianon/gosu/archive/$GOSU_VERSION.tar.gz" \
	&& export GNUPGHOME="$(mktemp -d)" \
	&& found='' \
	&& for server in \
		pool.sks-keyservers.net \
		ha.pool.sks-keyservers.net \
		pgp.mit.edu \
		na.pool.sks-keyservers.net \
		eu.pool.sks-keyservers.net \
		oc.pool.sks-keyservers.net \
		ha.pool.sks-keyservers.net \
		hkp://p80.pool.sks-keyservers.net:80 \
		hkp://keyserver.ubuntu.com:80 \
	; do \
		gpg --keyserver $server --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
			&& found="yes" && break; \
	done; test -n "$found" \
	&& gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu \
	&& rm -r "$GNUPGHOME" /usr/local/bin/gosu.asc \
	&& chmod +x /usr/local/bin/gosu \
	&& gosu nobody true \
	&& apt-get purge -y --auto-remove ca-certificates wget

# solves warning: "jemalloc shared library could not be preloaded to speed up memory allocations"
RUN apt-get update && apt-get install -y --no-install-recommends libjemalloc1 && rm -rf /var/lib/apt/lists/*

# https://wiki.apache.org/cassandra/DebianPackaging#Adding_Repository_Keys
ENV GPG_KEYS 514A2AD631A57A16DD0047EC749D6EEC0353B12C A26E528B271F19B9E5D8E19EA278B781FE4B2BDA

RUN set -ex; \
	export GNUPGHOME="$(mktemp -d)"; \
	for key in $GPG_KEYS \
	    ; do \
    	      	 found=''; \
    		 for server in \
        	     pool.sks-keyservers.net \
        	     ha.pool.sks-keyservers.net \
		     pgp.mit.edu \
        	     na.pool.sks-keyservers.net \
        	     eu.pool.sks-keyservers.net \
        	     oc.pool.sks-keyservers.net \
        	     ha.pool.sks-keyservers.net \
        	     hkp://p80.pool.sks-keyservers.net:80 \
        	     hkp://keyserver.ubuntu.com:80 \
        	 ; do \
            	      gpg --keyserver $server --recv-keys "$key" && found="yes" && break; \
   		 done; test -n "$found"; \
	done; \
	gpg --export $GPG_KEYS > /etc/apt/trusted.gpg.d/cassandra.gpg; \
	rm -r "$GNUPGHOME"; \
	apt-key list

RUN echo 3.0.16 | sed 's-\([^.]\+\)\.\([^.]\+\).*$-deb http://www.apache.org/dist/cassandra/debian \1\2x main-' >> /etc/apt/sources.list.d/cassandra.list

ENV CASSANDRA_VERSION 3.0.16

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

