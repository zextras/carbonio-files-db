FROM registry.dev.zextras.com/dev/carbonio-sidecar:latest

# Service registration
COPY package/carbonio-files-db.hcl /consul/config/

# Setup files at the path the setup script expects
COPY package/service-protocol.json /etc/carbonio/files-db/service-discover/
COPY package/intentions.json       /etc/carbonio/files-db/service-discover/
COPY package/policies.json         /etc/carbonio/files-db/service-discover/

# Setup script
COPY package/carbonio-files-db /usr/local/bin/carbonio-files-db
RUN chmod +x /usr/local/bin/carbonio-files-db

ENV SERVICE_NAME=carbonio-files-db

ENTRYPOINT ["/bin/sh", "-c", "\
  echo '[sidecar] Waiting for consul agent...' && \
  until consul members >/dev/null 2>&1; do sleep 1; done && \
  echo '[sidecar] Consul agent ready.' && \
  consul services register /consul/config/*.hcl && \
  (carbonio-files-db setup || true) && \
  echo '[sidecar] Starting envoy for $SERVICE_NAME' && \
  exec consul connect envoy \
    -sidecar-for=$SERVICE_NAME \
    -admin-bind=localhost:0 \
"]
