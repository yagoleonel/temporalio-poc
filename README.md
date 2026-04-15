# temporal-poc

Self-hosted Temporal cluster with **mTLS**, backed by PostgreSQL.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/install/) (v2)
- OpenSSL (for certificate generation)
- [grpcurl](https://github.com/fullstorydev/grpcurl) (optional — for verifying the gRPC endpoint)

## Quick start

### 1. Create the `.env` file

Copy the example and adjust values as needed:

```bash
cp .env.example .env
```

The defaults in `.env.example` are ready for local development:

| Variable             | Example value          |
|----------------------|------------------------|
| `POSTGRES_USER`      | `temporal`             |
| `POSTGRES_PASSWORD`  | `temporal`             |
| `POSTGRES_DB`        | `temporal`             |
| `POSTGRES_PORT`      | `5432`                 |
| `POSTGRESQL_VERSION` | `16`                   |
| `TEMPORAL_VERSION`   | `1.29.1`               |

For OIDC authentication on the Temporal UI, also configure:

| Variable                     | Example value                                                  |
|------------------------------|----------------------------------------------------------------|
| `TEMPORAL_AUTH_ENABLED`      | `true`                                                         |
| `TEMPORAL_AUTH_PROVIDER_URL` | `https://your-idp.example.com/.well-known/openid-configuration`|
| `TEMPORAL_AUTH_CLIENT_ID`    | `your-client-id`                                               |
| `TEMPORAL_AUTH_CLIENT_SECRET`| `your-client-secret`                                           |
| `TEMPORAL_AUTH_CALLBACK_URL` | `http://localhost:8080/auth/sso/callback`                      |
| `TEMPORAL_AUTH_SCOPES`       | `openid profile email`                                         |

See [doc/OIDC_TEMPORAL_UI.md](doc/OIDC_TEMPORAL_UI.md) for details.

### 2. Generate TLS certificates

```bash
bash generate-test-certs.sh
```

This creates a self-signed CA and issues a cluster certificate and a client
certificate under `certs/`. The SANs are defined in `cluster-cert.conf` and
`client-cert.conf`.

> **Note:** These certificates are for development/testing only.
> In production, use certificates issued by a proper CA.

### 3. Start the cluster

```bash
docker compose up -d
```

The first start takes a bit longer because `temporal-history` (the `auto-setup`
image) creates the database schema and registers the default namespace.

### 4. Verify

Check that all six services are running:

```bash
docker compose ps
```

Confirm there are no TLS errors in the logs:

```bash
docker compose logs 2>&1 | grep -iE "x509|handshake failed|tls error"
```

Test the gRPC endpoint with mTLS using `grpcurl`:

```bash
grpcurl \
  -cacert ./certs/ca.cert \
  -cert   ./certs/client.pem \
  -key    ./certs/client.key \
  -servername temporal-cluster \
  localhost:7233 \
  temporal.api.workflowservice.v1.WorkflowService/GetSystemInfo
```

### 5. Access the UI

Open <http://localhost:8080> in your browser.

## Stopping / restarting

```bash
docker compose down          # stop and remove containers
docker compose up -d         # start again (data persists in the named volume)
```

## Regenerating certificates

If you change the OpenSSL configs or the certs expire, regenerate and restart:

```bash
bash generate-test-certs.sh
docker compose down && docker compose up -d
```

## Further reading

- [doc/TLS_CONFIG.md](doc/TLS_CONFIG.md) — full mTLS env-var mapping, certificate details, and connection examples
- [doc/OIDC_TEMPORAL_UI.md](doc/OIDC_TEMPORAL_UI.md) — OIDC authentication for the Temporal UI
- [temporal-doc/](temporal-doc/) — Temporal platform documentation:
  - [ENVIRONMENT_VARIABLES.md](temporal-doc/ENVIRONMENT_VARIABLES.md) — server image env var reference
  - [MTLS_AUTHENTICATE.md](temporal-doc/MTLS_AUTHENTICATE.md) — mTLS authentication deep-dive
  - [SELF_HOSTED_GUIDE.md](temporal-doc/SELF_HOSTED_GUIDE.md) — self-hosted deployment guide
  - [TEMPORAL_CLUSTER_CONFIG.md](temporal-doc/TEMPORAL_CLUSTER_CONFIG.md) — cluster topology & configuration
  - [TEMPORAL_PLATFORM_SECURITY.md](temporal-doc/TEMPORAL_PLATFORM_SECURITY.md) — platform security overview
  - [TEMPORAL_SERVER_OPTIONS_REF.md](temporal-doc/TEMPORAL_SERVER_OPTIONS_REF.md) — server CLI / config options reference
  - [TEMPORAL_SERVICE_CONFIG.md](temporal-doc/TEMPORAL_SERVICE_CONFIG.md) — per-service configuration
