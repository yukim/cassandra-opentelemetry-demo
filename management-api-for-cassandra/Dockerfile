# Based on https://github.com/k8ssandra/management-api-for-apache-cassandra/blob/master/cassandra-trunk/Dockerfile.ubi8
# Management API jars and CDC agent are taken from the existing 5.1-nightly-latest image.
# (This version does not include Metrics collector for Apache Cassandra.)
#
# Apache Cassadnra with OpenTelemetry instrumentation is taken from `otel` branch of yukim/cassandra repo. 
ARG UBI_MAJOR=8
ARG UBI_BASETAG=latest
ARG CASSANDRA_VERSION=5.1-SNAPSHOT

ARG METRICS_COLLECTOR_VERSION=0.3.4
ARG METRICS_COLLECTOR_BUNDLE=${METRICS_COLLECTOR_VERSION}-4.1-beta1
ARG CASSANDRA_VERSION

FROM k8ssandra/cass-management-api:5.1-nightly-latest as maac

#############################################################

# Build and extract Cassandra
FROM --platform=$BUILDPLATFORM azul/zulu-openjdk-debian:11 as cass-builder
ARG CASSANDRA_VERSION
ARG COMMITSHA="HEAD"
ENV CASSANDRA_PATH /opt/cassandra
ENV CASSANDRA_FILES_PATH /opt/cassandra_files
WORKDIR /build
RUN set -x \
    && rm -fr /etc/apt/sources.list.d/* \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get update \
    && apt-get install -y --no-install-recommends git ant ant-optional make maven \
    && git clone -b otel --single-branch https://github.com/yukim/cassandra.git \
    && cd cassandra \
    && git checkout $(git rev-parse --short ${COMMITSHA}) \
    #&& ant -q -S echo-base-version > /build/cassandra.version \
    # create an empty javadoc archive so we can skip javadoc generation
    && mkdir -p ./build/ \
    && touch ./build/apache-cassandra-${CASSANDRA_VERSION}-javadoc.jar \
    && ant artifacts mvn-install -Duse.jdk11=true -Dno-javadoc=true \
    && mkdir -m 775 ${CASSANDRA_PATH} ${CASSANDRA_FILES_PATH} \
    && tar --directory ${CASSANDRA_PATH} --strip-components 1 --gzip --extract --file /build/cassandra/build/apache-cassandra-${CASSANDRA_VERSION}-bin.tar.gz \
    && rm -rf ${CASSANDRA_PATH}/javadoc ${CASSANDRA_PATH}/doc \
    && chmod -R g+w ${CASSANDRA_PATH}
COPY cassandra/files ${CASSANDRA_FILES_PATH}


FROM registry.access.redhat.com/ubi${UBI_MAJOR}/ubi-minimal:${UBI_BASETAG}
ARG TARGETARCH
ARG CASSANDRA_VERSION
ENV CASSANDRA_VERSION=${CASSANDRA_VERSION}

LABEL maintainer="Yuki Morishita <yukim@apache.org>"
LABEL name="Apache Cassandra"
LABEL release="${CASSANDRA_VERSION}"
LABEL summary="Apache Cassandra is an open source distributed database management system designed to handle large amounts of data across many commodity servers, providing high availability with no single point of failure. Cassandra offers robust support for clusters spanning multiple datacenters, with asynchronous masterless replication allowing low latency operations for all clients."
LABEL description="Apache Cassandra is an open source distributed database management system designed to handle large amounts of data across many commodity servers, providing high availability with no single point of failure. Cassandra offers robust support for clusters spanning multiple datacenters, with asynchronous masterless replication allowing low latency operations for all clients."

ENV LANG='en_US.UTF-8' LANGUAGE='en_US:en' LC_ALL='en_US.UTF-8'

ENV CASSANDRA_PATH /opt/cassandra
ENV MAAC_PATH /opt/management-api
ENV CDC_AGENT_PATH=/opt/cdc_agent

ENV PATH ${CASSANDRA_PATH}/bin:${PATH}
ENV CASSANDRA_HOME ${CASSANDRA_PATH}
ENV CASSANDRA_CONF ${CASSANDRA_PATH}/conf
ENV CASSANDRA_LOG_DIR /var/log/cassandra
ENV CASSANDRA_DATA_DIR /var/lib/cassandra

ENV CASSANDRA_FILES_PATH /opt/cassandra_files

# Update base layer
RUN microdnf update && rm -rf /var/cache/yum \
# Install packages needed during install process
    && microdnf install --nodocs java-11-openjdk-headless tzdata-java python3 zlib findutils which hostname iproute shadow-utils procps util-linux glibc-langpack-en wget \
    && microdnf clean all \
# Add Cassandra user
    && groupadd -r cassandra --gid=999 \
    && useradd -m -d "$CASSANDRA_HOME" -r -g cassandra -G root --uid=999 cassandra

# Copy trimmed installation
COPY --from=cass-builder --chown=cassandra:root ${CASSANDRA_PATH} ${CASSANDRA_PATH}
COPY --from=cass-builder --chown=cassandra:root ${CASSANDRA_PATH}/LICENSE.txt /licenses/
COPY --from=cass-builder --chown=cassandra:root ${CASSANDRA_FILES_PATH} /
COPY --from=maac --chown=cassandra:root ${MAAC_PATH} ${MAAC_PATH}
COPY --from=maac --chown=cassandra:root ${CDC_AGENT_PATH} ${CDC_AGENT_PATH}

# Create directories
RUN (for dir in ${CASSANDRA_DATA_DIR} \
                ${CASSANDRA_LOG_DIR} ; do \
        mkdir -p $dir && chown -R cassandra:root $dir && chmod 775 $dir ; \
    done ) && \
    ln -sT ${CASSANDRA_DATA_DIR} ${CASSANDRA_HOME}/data && \
    ln -sT ${CASSANDRA_LOG_DIR} ${CASSANDRA_HOME}/logs && \
    # setup conf directory
    [ ! -e "/etc/cassandra" ]; \
    mv ${CASSANDRA_CONF} /etc/cassandra && \
    ln -sT /etc/cassandra ${CASSANDRA_CONF} && \
    # change mode of directories
    chmod a+rwX ${MAAC_PATH} ${CASSANDRA_PATH} ${CDC_AGENT_PATH} /etc/cassandra

ENV TINI_VERSION v0.19.0
ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini-${TARGETARCH} /tini
RUN chmod +x /tini

# Set user to run as
USER cassandra

# Expose CASSANDRA folders
VOLUME ["/var/lib/cassandra", "/var/log/cassandra"]

# MANAGEMENT API, MCAC METRICS, NON-MCAC METRICS
EXPOSE 8080 9103 9000

# CASSANDRA PORTS (INTRA-NODE, TLS INTRA-NODE, JMX, CQL, THRIFT, DSEFS INTRA-NODE, INTRA-NODE MESSAGING SERVICE)
EXPOSE 7000 7001 7199 8609 9042 9160

ENTRYPOINT ["/tini", "-g", "--", "/entrypoint.sh"]
CMD ["mgmtapi"]