# psql + openssl are needed by the bootstrap/setup scripts. The carbonio-sidecar
# base is ubuntu jammy; installing postgresql-client/openssl with apt in the
# final stage runs target-arch maintainer scripts (no QEMU => exec format error
# on arm64).
#
# Instead, cross-download the target-arch .debs on the BUILDPLATFORM (amd64 CI
# builder) from ports.ubuntu.com, extract them with dpkg -x (no maintainer
# scripts execute), and assemble a minimal tree of just the binaries + the libs
# the base does NOT already provide. libssl/libcrypto/libgssapi/libgnutls/
# libtinfo are already in the base, so we ship only: libpq, libldap-2.5,
# liblber-2.5, libsasl2, libreadline. Everything goes under /usr/lib (not /lib):
# ubuntu jammy is usrmerged so /lib is a symlink to /usr/lib, and COPYing into
# /lib/* fails on some buildkit versions ("cannot copy to non-directory").
FROM --platform=$BUILDPLATFORM ubuntu:resolute AS pgclient
ARG TARGETARCH
RUN set -eux; \
    case "$TARGETARCH" in \
      arm64) TRIPLET=aarch64-linux-gnu ;; \
      amd64) TRIPLET=x86_64-linux-gnu ;; \
      *) echo "unsupported TARGETARCH=$TARGETARCH" >&2; exit 1 ;; \
    esac; \
    dpkg --add-architecture "${TARGETARCH}"; \
    if [ "$TARGETARCH" != "amd64" ]; then \
      sed -i 's|http://archive.ubuntu.com/ubuntu/|[arch=amd64] http://archive.ubuntu.com/ubuntu/|; s|http://security.ubuntu.com/ubuntu/|[arch=amd64] http://security.ubuntu.com/ubuntu/|' /etc/apt/sources.list; \
      printf 'deb [arch=arm64] http://ports.ubuntu.com/ubuntu-ports/ jammy main universe\ndeb [arch=arm64] http://ports.ubuntu.com/ubuntu-ports/ jammy-updates main universe\ndeb [arch=arm64] http://ports.ubuntu.com/ubuntu-ports/ jammy-security main universe\n' > /etc/apt/sources.list.d/arm64.list; \
    fi; \
    apt-get update; \
    mkdir -p /dl /x /out/usr/bin "/out/usr/lib/${TRIPLET}"; \
    apt-get install -y --download-only -o Dir::Cache::archives=/dl \
      postgresql-client-14:${TARGETARCH} libpq5:${TARGETARCH} openssl:${TARGETARCH}; \
    for d in /dl/*.deb; do dpkg -x "$d" /x; done; \
    cp -a /x/usr/lib/postgresql/14/bin/psql        /out/usr/bin/psql; \
    cp -a /x/usr/lib/postgresql/14/bin/pg_isready   /out/usr/bin/pg_isready; \
    cp -a /x/usr/bin/openssl                        /out/usr/bin/openssl; \
    cp -a /x/usr/lib/${TRIPLET}/libpq.so.*          "/out/usr/lib/${TRIPLET}/"; \
    cp -a /x/usr/lib/${TRIPLET}/libldap-2.5.so.*    "/out/usr/lib/${TRIPLET}/"; \
    cp -a /x/usr/lib/${TRIPLET}/liblber-2.5.so.*    "/out/usr/lib/${TRIPLET}/"; \
    cp -a /x/usr/lib/${TRIPLET}/libsasl2.so.*       "/out/usr/lib/${TRIPLET}/"; \
    cp -a /x/lib/${TRIPLET}/libreadline.so.*        "/out/usr/lib/${TRIPLET}/"

FROM registry.dev.zextras.com/dev/carbonio-sidecar:devel

# psql + pg_isready + openssl + their not-already-present shared libs
# (arch-matched, cross-downloaded above). No target-arch RUN => QEMU-free.
COPY --from=pgclient /out /

# Service registration
COPY package/carbonio-files-db.hcl /consul/config/

# Setup files at the path the setup script expects
COPY package/service-protocol.json /etc/carbonio/files-db/service-discover/
COPY package/intentions.json       /etc/carbonio/files-db/service-discover/
COPY package/policies.json         /etc/carbonio/files-db/service-discover/

# Setup script (COPY --chmod avoids a target-arch RUN — QEMU-free)
COPY --chmod=755 package/carbonio-files-db /usr/local/bin/carbonio-files-db

# Bootstrap script (creates DB user, database, saves credentials to consul KV)
COPY --chmod=755 package/carbonio-files-db-bootstrap /usr/local/bin/carbonio-files-db-bootstrap

ENV SERVICE_NAME=carbonio-files-db
ENV PGPASSWORD=admin
ENV POSTGRES_USER=admin
ENV POSTGRES_HOST=localhost
ENV POSTGRES_PORT=5432

ENTRYPOINT ["/bin/sh", "-c", "\
  echo '[sidecar] Waiting for consul agent...' && \
  until consul members >/dev/null 2>&1; do sleep 1; done && \
  echo '[sidecar] Consul agent ready.' && \
  echo '[sidecar] Waiting for postgres...' && \
  until pg_isready -h $POSTGRES_HOST -p $POSTGRES_PORT >/dev/null 2>&1; do sleep 1; done && \
  until PGPASSWORD=$PGPASSWORD psql -h $POSTGRES_HOST -p $POSTGRES_PORT -U $POSTGRES_USER -w -c 'SELECT 1' >/dev/null 2>&1; do sleep 1; done && \
  echo '[sidecar] Postgres ready.' && \
  consul services register /consul/config/*.hcl && \
  echo '[sidecar] Running setup...' && \
  (carbonio-files-db setup || true) && \
  echo '[sidecar] Running bootstrap...' && \
  (carbonio-files-db-bootstrap $POSTGRES_USER $POSTGRES_HOST || true) && \
  echo '[sidecar] Starting envoy for $SERVICE_NAME' && \
  exec consul connect envoy \
    -sidecar-for=$SERVICE_NAME \
    -admin-bind=localhost:0 \
"]
