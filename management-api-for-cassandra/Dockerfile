# Based on https://github.com/k8ssandra/management-api-for-apache-cassandra/blob/master/Dockerfile-4_0
# this version does not include Metrics collector for Apache Cassandra.
FROM k8ssandra/cass-management-api:4.0 as maac

FROM ghcr.io/yukim/cassandra:4.0-otel

ARG TARGETARCH=amd64

ENV CASSANDRA_PATH /opt/cassandra
ENV MAAC_PATH /opt/management-api
ENV CDC_AGENT_PATH=/opt/cdc_agent
ENV USER_HOME_PATH /home/cassandra

ENV CASSANDRA_HOME ${CASSANDRA_PATH}
ENV CASSANDRA_CONF ${CASSANDRA_PATH}/conf
ENV CASSANDRA_LOGS ${CASSANDRA_PATH}/logs
# Log directory for Management API startup logs to avoid issues:
# https://datastax.jira.com/browse/DB-4627
# https://issues.apache.org/jira/browse/CASSANDRA-16027
ENV MGMT_API_LOG_DIR /var/log/cassandra

COPY --from=maac --chown=cassandra:root ${MAAC_PATH} ${MAAC_PATH}
COPY --from=maac --chown=cassandra:root ${USER_HOME_PATH} ${USER_HOME_PATH}
COPY --from=maac --chown=cassandra:root ${CDC_AGENT_PATH} ${CDC_AGENT_PATH}

# Setup user and fixup permissions
RUN chown -R cassandra:root ${CASSANDRA_PATH} && chmod -R g+w ${CASSANDRA_PATH}

ENV TINI_VERSION v0.19.0
ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini-${TARGETARCH} /tini
RUN chmod +x /tini

RUN set -eux; \
    rm -fr /etc/apt/sources.list.d/*; \
    rm -rf /var/lib/apt/lists/*; \
    apt-get update; \
    apt-get install -y --no-install-recommends wget iproute2; \
    rm -rf /var/lib/apt/lists/*

# backwards compat with upstream ENTRYPOINT
COPY scripts/docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh && \
    ln -sf /usr/local/bin/docker-entrypoint.sh /docker-entrypoint.sh && \
    # fix for the missing mtab in the containerd
    ln -sf /proc/mounts /etc/mtab

EXPOSE 9103
EXPOSE 8080

USER cassandra

ENTRYPOINT ["/tini", "-g", "--", "/docker-entrypoint.sh"]
CMD ["mgmtapi"]