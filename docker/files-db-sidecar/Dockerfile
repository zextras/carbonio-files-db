FROM registry.dev.zextras.com/dev/carbonio-sidecar:latest

# psql is needed by the bootstrap script to create the database
RUN apt-get update && apt-get install -y postgresql-client openssl && rm -rf /var/lib/apt/lists/*

# Service registration
COPY package/carbonio-files-db.hcl /consul/config/

# Setup files at the path the setup script expects
COPY package/service-protocol.json /etc/carbonio/files-db/service-discover/
COPY package/intentions.json       /etc/carbonio/files-db/service-discover/
COPY package/policies.json         /etc/carbonio/files-db/service-discover/

# Setup script
COPY package/carbonio-files-db /usr/local/bin/carbonio-files-db
RUN chmod +x /usr/local/bin/carbonio-files-db

# Bootstrap script (creates DB user, database, saves credentials to consul KV)
COPY package/carbonio-files-db-bootstrap /usr/local/bin/carbonio-files-db-bootstrap
RUN chmod +x /usr/local/bin/carbonio-files-db-bootstrap

ENV SERVICE_NAME=carbonio-files-db
ENV PGPASSWORD=admin
ENV POSTGRES_USER=admin
ENV POSTGRES_HOST=localhost
ENV POSTGRES_PORT=5432

ENTRYPOINT ["/bin/sh", "-c", "\
  echo '[sidecar] Waiting for consul agent...' && \
  until consul members >/dev/null 2>&1; do sleep 1; done && \
  echo '[sidecar] Consul agent ready.' && \
  consul services register /consul/config/*.hcl && \
  (carbonio-files-db setup || true) && \
  echo '[sidecar] Waiting for postgres...' && \
  until pg_isready -h $POSTGRES_HOST -p $POSTGRES_PORT >/dev/null 2>&1; do sleep 1; done && \
  echo '[sidecar] Postgres ready, running bootstrap...' && \
  (carbonio-files-db-bootstrap $POSTGRES_USER $POSTGRES_HOST $POSTGRES_PORT || true) && \
  echo '[sidecar] Starting envoy for $SERVICE_NAME' && \
  exec consul connect envoy \
    -sidecar-for=$SERVICE_NAME \
    -admin-bind=localhost:0 \
"]
