# Based on https://github.com/apache/cassandra-builds/blob/trunk/docker/buster-image.docker
# for building Apache Cassandra from source,
# and https://github.com/docker-library/cassandra/blob/master/4.0/Dockerfile
# for Apache Cassandra image based on the build.
FROM debian:buster as builder

ENV DEB_DIST_DIR=/dist
ENV BUILD_HOME=/home/build
ENV CASSANDRA_DIR=$BUILD_HOME/cassandra
ARG UID_ARG=1000
ARG GID_ARG=1000

LABEL org.cassandra.buildenv=buster

VOLUME ${DEB_DIST_DIR}

# configure apt to retry downloads
RUN echo 'APT::Acquire::Retries "9";' > /etc/apt/apt.conf.d/80-retries

# install deps
RUN apt-get update && apt-get -y install \
    ant \
    build-essential \
    curl \
    devscripts \
    git \
    sudo

RUN echo 'deb http://ftp.debian.org/debian stretch main' >> /etc/apt/sources.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends openjdk-11-jdk \
    && sed -i '$d' /etc/apt/sources.list \
    && apt-get update

# create and change to build user
RUN groupadd --gid ${GID_ARG} --non-unique build \
    && adduser --uid ${UID_ARG} --gid ${GID_ARG} --disabled-login --gecos build build \
    && gpasswd -a build sudo

RUN echo "build ALL=(root) NOPASSWD:ALL" > /etc/sudoers.d/build && \
    chmod 0440 /etc/sudoers.d/build

USER build

# clone Cassandra and cache maven artifacts
ARG CASSANDRA_GIT_URL=https://github.com/yukim/cassandra.git
ARG OTEL_BRANCH=cassandra-4.0-otel
RUN git config --global http.postBuffer 524288000
RUN git clone --depth 1 --branch ${OTEL_BRANCH} --single-branch ${CASSANDRA_GIT_URL} ${CASSANDRA_DIR}
WORKDIR ${CASSANDRA_DIR}

RUN ant -Duse.jdk11=true resolver-retrieve-build
RUN ant -Duse.jdk11=true -Dno-javadoc=true -Dant.gen-doc.skip=true artifacts \
    && sudo cp ${CASSANDRA_DIR}/build/apache-cassandra-*-bin.tar.gz /cassandra-bin.tgz

FROM eclipse-temurin:11-jre-focal

# explicitly set user/group IDs
RUN set -eux; \
    groupadd -r cassandra --gid=999; \
    useradd -r -g cassandra --uid=999 cassandra

RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
    # solves warning: "jemalloc shared library could not be preloaded to speed up memory allocations"
    libjemalloc2 \
    # "free" is used by cassandra-env.sh
    procps \
    # "cqlsh" needs a python interpreter
    python3 \
    # "ip" is not required by Cassandra itself, but is commonly used in scripting Cassandra's configuration (since it is so fixated on explicit IP addresses)
    iproute2 \
    # Cassandra will automatically use numactl if available
    #   https://github.com/apache/cassandra/blob/18bcda2d4c2eba7370a0b21f33eed37cb730bbb3/bin/cassandra#L90-L100
    #   https://github.com/apache/cassandra/commit/604c0e87dc67fa65f6904ef9a98a029c9f2f865a
    numactl \
    ; \
    rm -rf /var/lib/apt/lists/*; \
    # https://issues.apache.org/jira/browse/CASSANDRA-15767 ("bin/cassandra" only looks for "libjemalloc.so" or "libjemalloc.so.1" which doesn't match our "libjemalloc.so.2")
    libjemalloc="$(readlink -e /usr/lib/*/libjemalloc.so.2)"; \
    ln -sT "$libjemalloc" /usr/local/lib/libjemalloc.so; \
    ldconfig

# grab gosu for easy step-down from root
# https://github.com/tianon/gosu/releases
ENV GOSU_VERSION 1.14
RUN set -eux; \
    savedAptMark="$(apt-mark showmanual)"; \
    apt-get update; \
    apt-get install -y --no-install-recommends ca-certificates dirmngr gnupg wget; \
    rm -rf /var/lib/apt/lists/*; \
    dpkgArch="$(dpkg --print-architecture | awk -F- '{ print $NF }')"; \
    wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch"; \
    wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch.asc"; \
    export GNUPGHOME="$(mktemp -d)"; \
    gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4; \
    gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu; \
    gpgconf --kill all; \
    rm -rf "$GNUPGHOME" /usr/local/bin/gosu.asc; \
    apt-mark auto '.*' > /dev/null; \
    [ -z "$savedAptMark" ] || apt-mark manual $savedAptMark > /dev/null; \
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
    chmod +x /usr/local/bin/gosu; \
    gosu --version; \
    gosu nobody true

ENV CASSANDRA_HOME /opt/cassandra
ENV CASSANDRA_CONF /etc/cassandra
ENV PATH $CASSANDRA_HOME/bin:$PATH

COPY --chown=cassandra:cassandra --from=builder /cassandra-bin.tgz .

RUN set -eux; \
    mkdir -p "$CASSANDRA_HOME"; \
    tar --extract --file cassandra-bin.tgz --directory "$CASSANDRA_HOME" --strip-components 1; \
    rm cassandra-bin.tgz*; \
    \
    [ ! -e "$CASSANDRA_CONF" ]; \
    mv "$CASSANDRA_HOME/conf" "$CASSANDRA_CONF"; \
    ln -sT "$CASSANDRA_CONF" "$CASSANDRA_HOME/conf"; \
    \
    dpkgArch="$(dpkg --print-architecture)"; \
    case "$dpkgArch" in \
    ppc64el) \
    # https://issues.apache.org/jira/browse/CASSANDRA-13345
    # "The stack size specified is too small, Specify at least 328k"
    grep -- '^-Xss256k$' "$CASSANDRA_CONF/jvm-server.options"; \
    sed -ri 's/^-Xss256k$/-Xss512k/' "$CASSANDRA_CONF/jvm-server.options"; \
    grep -- '^-Xss512k$' "$CASSANDRA_CONF/jvm-server.options"; \
    ;; \
    esac; \
    \
    mkdir -p "$CASSANDRA_CONF" /var/lib/cassandra /var/log/cassandra; \
    chown -R cassandra:cassandra "$CASSANDRA_CONF" /var/lib/cassandra /var/log/cassandra; \
    chmod 777 "$CASSANDRA_CONF" /var/lib/cassandra /var/log/cassandra; \
    chmod -R a+rwX "$CASSANDRA_CONF"; \
    ln -sT /var/lib/cassandra "$CASSANDRA_HOME/data"; \
    ln -sT /var/log/cassandra "$CASSANDRA_HOME/logs"; \
    \
    # smoke test
    cassandra -v

VOLUME /var/lib/cassandra

COPY docker-entrypoint.sh /usr/local/bin/
ENTRYPOINT ["docker-entrypoint.sh"]

# 7000: intra-node communication
# 7001: TLS intra-node communication
# 7199: JMX
# 9042: CQL
EXPOSE 7000 7001 7199 9042
CMD ["cassandra", "-f"]