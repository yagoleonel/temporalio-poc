# AGENTS.md — temporal-poc

## Purpose
Docker Compose stack running a self-hosted Temporal cluster with **mTLS**,
backed by PostgreSQL, plus Temporal UI with OIDC authentication.

## Structure
```
docker-compose.yml          # full local stack definition
.env.example                # all required env vars with example values
.env                        # actual env values (gitignored)
README.md                   # quick-start guide
generate-test-certs.sh      # generates CA + cluster + client certificates
cluster-cert.conf           # OpenSSL config for the cluster certificate (SANs)
client-cert.conf            # OpenSSL config for the client certificate
certs/                      # generated certificates — gitignored
doc/                        # project documentation
  TLS_CONFIG.md             # mTLS configuration reference
  OIDC_TEMPORAL_UI.md       # OIDC authentication for the Temporal UI
temporal-doc/               # Temporal platform generic documentation
  ENVIRONMENT_VARIABLES.md  # env var reference for Temporal server images
  MTLS_AUTHENTICATE.md      # mTLS authentication deep-dive
  SELF_HOSTED_GUIDE.md      # self-hosted deployment guide
  TEMPORAL_CLUSTER_CONFIG.md        # cluster topology & configuration
  TEMPORAL_PLATFORM_SECURITY.md     # platform-level security overview
  TEMPORAL_SERVER_OPTIONS_REF.md    # server CLI / config options reference
  TEMPORAL_SERVICE_CONFIG.md        # per-service configuration details
```

## Services
| Service            | Image                            | Notes                                   |
|--------------------|----------------------------------|-----------------------------------------|
| `postgresql`       | `postgres:${POSTGRESQL_VERSION}` | Temporal persistence DB                 |
| `temporal-history` | `temporalio/auto-setup:${TEMPORAL_VERSION}` | DB schema setup + history service |
| `temporal-matching`| `temporalio/server:${TEMPORAL_VERSION}`     | Temporal matching service         |
| `temporal-frontend`| `temporalio/server:${TEMPORAL_VERSION}`     | Temporal frontend (gRPC port 7233)|
| `temporal-worker`  | `temporalio/server:${TEMPORAL_VERSION}`     | Temporal internal worker service  |
| `temporal-ui`      | `temporalio/ui:latest`           | Web UI on port 8080 (OIDC-enabled)      |

## Dependency order
postgresql → temporal-history → temporal-matching → temporal-frontend → temporal-worker
temporal-frontend → temporal-ui

Use `depends_on` with `condition: service_healthy` for postgresql → temporal-history.
Use `depends_on` with `condition: service_started` for all subsequent dependencies.

## Networking
All services share one bridge network: `temporal-network`.

| Port   | Service       | Exposed externally |
|--------|---------------|--------------------|
| `5432` | postgresql    | No                 |
| `7233` | temporal gRPC | Yes                |
| `8080` | temporal-ui   | Yes                |

## mTLS
mTLS is configured entirely via `TEMPORAL_TLS_*` environment variables,
following the official [`tls-simple`](https://github.com/temporalio/samples-server/tree/main/tls/tls-simple) pattern.
Certificates are mounted at `./certs:/etc/temporal/config/certs` on every Temporal service.

- **Frontend server-side TLS** (`CLIENT1_CA_CERT`, `FRONTEND_CERT`, `FRONTEND_KEY`) is set **only** on `temporal-frontend`.
- All four Temporal server services set internode server/client and frontend client vars.
- `temporal-history` (auto-setup) additionally sets `TEMPORAL_TLS_CA/CERT/KEY/SERVER_NAME` for the Temporal CLI health-check.
- See [doc/TLS_CONFIG.md](doc/TLS_CONFIG.md) for the full env-var-to-config mapping and connection examples.

## OIDC Authentication
The Temporal UI authenticates users via OpenID Connect.
OIDC credentials (`TEMPORAL_AUTH_*`) are stored in `.env` — never hardcoded in `docker-compose.yml`.
See [doc/OIDC_TEMPORAL_UI.md](doc/OIDC_TEMPORAL_UI.md) for setup instructions.

## Conventions
- PostgreSQL credentials and OIDC secrets only in `.env` — never hardcoded in `docker-compose.yml`
- All app env vars must reference keys defined in `.env.example`
- Use a named volume for PostgreSQL data: `temporal-db-data`
- Image versions parameterised via `TEMPORAL_VERSION` and `POSTGRESQL_VERSION`

## Documentation
| File                                                              | Topic                            |
|-------------------------------------------------------------------|----------------------------------|
| [doc/TLS_CONFIG.md](doc/TLS_CONFIG.md)                            | mTLS env vars, certs & examples  |
| [doc/OIDC_TEMPORAL_UI.md](doc/OIDC_TEMPORAL_UI.md)                | OIDC authentication for the UI   |
| [temporal-doc/ENVIRONMENT_VARIABLES.md](temporal-doc/ENVIRONMENT_VARIABLES.md) | Server image env var reference |
| [temporal-doc/MTLS_AUTHENTICATE.md](temporal-doc/MTLS_AUTHENTICATE.md)         | mTLS authentication deep-dive |
| [temporal-doc/SELF_HOSTED_GUIDE.md](temporal-doc/SELF_HOSTED_GUIDE.md)         | Self-hosted deployment guide   |
| [temporal-doc/TEMPORAL_CLUSTER_CONFIG.md](temporal-doc/TEMPORAL_CLUSTER_CONFIG.md) | Cluster topology & config   |
| [temporal-doc/TEMPORAL_PLATFORM_SECURITY.md](temporal-doc/TEMPORAL_PLATFORM_SECURITY.md) | Platform security overview |
| [temporal-doc/TEMPORAL_SERVER_OPTIONS_REF.md](temporal-doc/TEMPORAL_SERVER_OPTIONS_REF.md) | Server options reference |
| [temporal-doc/TEMPORAL_SERVICE_CONFIG.md](temporal-doc/TEMPORAL_SERVICE_CONFIG.md) | Per-service configuration    |

## Never touch
- `certs/` contents are generated — regenerate via `bash generate-test-certs.sh`