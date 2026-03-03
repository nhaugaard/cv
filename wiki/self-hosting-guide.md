# Page: Self-Hosting Guide

# Self-Hosting Guide

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.devcontainer/Dockerfile](.devcontainer/Dockerfile)
- [.devcontainer/devcontainer.json](.devcontainer/devcontainer.json)
- [.devcontainer/docker-compose.yml](.devcontainer/docker-compose.yml)
- [CLAUDE.md](CLAUDE.md)
- [README.md](README.md)
- [compose.dev.yml](compose.dev.yml)
- [compose.yml](compose.yml)
- [crowdin.yml](crowdin.yml)
- [docs/contributing/development.mdx](docs/contributing/development.mdx)
- [docs/getting-started/quickstart.mdx](docs/getting-started/quickstart.mdx)
- [docs/self-hosting/docker.mdx](docs/self-hosting/docker.mdx)
- [docs/self-hosting/examples.mdx](docs/self-hosting/examples.mdx)
- [src/integrations/orpc/router/storage.ts](src/integrations/orpc/router/storage.ts)
- [src/integrations/orpc/services/storage.ts](src/integrations/orpc/services/storage.ts)
- [src/routes/__root.tsx](src/routes/__root.tsx)
- [src/routes/api/health.ts](src/routes/api/health.ts)
- [src/utils/env.ts](src/utils/env.ts)
- [src/vite-env.d.ts](src/vite-env.d.ts)

</details>



This page is a comprehensive operational reference for deploying Reactive Resume on your own infrastructure. It covers the standard Docker Compose setup, environment variable configuration, reverse proxy integration, Docker Swarm deployment, health checks, backup procedures, and common failure modes.

For information about the Docker image build process itself (multi-arch builds, CI/CD pipelines, image signing), see [Docker Deployment](#5.1). For a complete reference of every environment variable and the subsystem it configures, see [Environment Configuration](#5.3).

---

## Prerequisites

| Requirement | Minimum | Recommended |
|---|---|---|
| Docker Engine | 20.10+ | Latest stable |
| Docker Compose | v2.0+ | Latest stable |
| vCPU | 2 | 4 |
| RAM | 2 GB | 4 GB |
| Disk | 10 GB | 20+ GB |

The application is distributed as a single Docker image:

- Docker Hub: `amruthpillai/reactive-resume:latest`
- GHCR: `ghcr.io/amruthpillai/reactive-resume:latest`

---

## Service Topology

The full production stack comprises four services plus one initialization job.

**Diagram: Docker Compose Service Topology**

```mermaid
graph TD
    Browser["Browser\n(port 3000)"] --> App

    subgraph Stack["Docker Compose Stack"]
        App["reactive_resume\n(amruthpillai/reactive-resume:latest)"]
        PG[("postgres\npostgres:latest\nport 5432")]
        Chrome["browserless\nghcr.io/browserless/chromium:latest\nport 3000 (internal)"]
        SW["seaweedfs\nchrislusf/seaweedfs:latest\nS3 API port 8333"]
        SWInit["seaweedfs_create_bucket\nquay.io/minio/mc:latest\none-shot init"]

        Vol1[("postgres_data")]
        Vol2[("seaweedfs_data")]
        Vol3[("reactive_resume_data\n/app/data")]
    end

    App -- "DATABASE_URL\nSQL via Drizzle ORM" --> PG
    App -- "PRINTER_ENDPOINT\nWebSocket / HTTP" --> Chrome
    App -- "S3_ENDPOINT\nS3 API" --> SW

    SWInit -- "creates bucket\nthen exits" --> SW
    SWInit -. "must complete before\napp starts" .-> App

    PG --> Vol1
    SW --> Vol2
    App --> Vol3
```

Sources: [compose.yml:1-116]()

---

## Standard Setup

### Step 1 — Generate `AUTH_SECRET`

```bash
openssl rand -hex 32
```

This value must remain stable after deployment. Changing it invalidates all existing sessions.

### Step 2 — Create `.env`

The minimal required variables:

```bash
TZ=Etc/UTC
APP_URL=http://localhost:3000
PRINTER_ENDPOINT=ws://browserless:3000?token=1234567890
DATABASE_URL=postgresql://postgres:postgres@postgres:5432/postgres
AUTH_SECRET=<your-generated-secret>
```

For S3 storage (optional; falls back to local filesystem if omitted):

```bash
S3_ACCESS_KEY_ID=seaweedfs
S3_SECRET_ACCESS_KEY=seaweedfs
S3_ENDPOINT=http://seaweedfs:8333
S3_BUCKET=reactive-resume
S3_FORCE_PATH_STYLE=true
```

### Step 3 — Start the stack

```bash
docker compose up -d
docker compose ps          # verify all services are healthy
docker compose logs -f reactive_resume
```

The application is available at `http://localhost:3000` once the `reactive_resume` container is healthy.

Sources: [compose.yml:73-116](), [docs/self-hosting/docker.mdx:44-253]()

---

## Environment Variables Reference

The environment schema is validated at startup using `@t3-oss/env-core` and Zod. Invalid or missing required variables cause the server to exit immediately. The authoritative definitions are in [src/utils/env.ts:1-72]().

### Required

| Variable | Format | Purpose |
|---|---|---|
| `APP_URL` | `http(s)://...` | Public canonical URL; used for redirects, auth callbacks, and absolute links |
| `DATABASE_URL` | `postgresql://user:pass@host:port/db` | PostgreSQL connection string |
| `PRINTER_ENDPOINT` | `ws://...` or `http://...` | Endpoint for the headless Chromium service |
| `AUTH_SECRET` | string (min 1 char) | Signs and verifies auth tokens; changing it invalidates all sessions |

### Optional — Server

| Variable | Default | Purpose |
|---|---|---|
| `TZ` | `Etc/UTC` | Container timezone for logs and timestamps |
| `PRINTER_APP_URL` | *(same as `APP_URL`)* | URL the printer service uses to reach the app internally (see [Printer Networking](#printer-networking)) |

### Optional — Social Auth

| Variable | Purpose |
|---|---|
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | Enables Google sign-in |
| `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` | Enables GitHub sign-in |
| `OAUTH_PROVIDER_NAME`, `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET` | Custom OIDC/OAuth2 provider |
| `OAUTH_DISCOVERY_URL` | OIDC discovery (preferred; use instead of manual URLs) |
| `OAUTH_AUTHORIZATION_URL`, `OAUTH_TOKEN_URL`, `OAUTH_USER_INFO_URL` | Manual OAuth endpoint config |
| `OAUTH_SCOPES` | Space-separated; default `openid profile email` |

### Optional — SMTP

| Variable | Default | Purpose |
|---|---|---|
| `SMTP_HOST` | — | If unset, emails are printed to server console instead of sent |
| `SMTP_PORT` | `587` | SMTP port |
| `SMTP_USER` / `SMTP_PASS` | — | SMTP credentials |
| `SMTP_FROM` | — | Sender address, e.g. `Reactive Resume <noreply@example.com>` |
| `SMTP_SECURE` | `false` | `true` for implicit TLS (port 465) |

### Optional — Storage (S3)

| Variable | Default | Purpose |
|---|---|---|
| `S3_ACCESS_KEY_ID` | — | If unset, `LocalStorageService` is used instead |
| `S3_SECRET_ACCESS_KEY` | — | S3 secret |
| `S3_REGION` | `us-east-1` | S3 region |
| `S3_ENDPOINT` | — | Override endpoint URL for S3-compatible services |
| `S3_BUCKET` | — | Bucket name |
| `S3_FORCE_PATH_STYLE` | `false` | Set `true` for MinIO, SeaweedFS, and other self-hosted services |

### Feature Flags

| Variable | Default | Effect when `true` |
|---|---|---|
| `FLAG_DISABLE_SIGNUPS` | `false` | Blocks new account registration (web + server) |
| `FLAG_DISABLE_EMAIL_AUTH` | `false` | Disables email/password login; email verification and password reset are also disabled |
| `FLAG_DEBUG_PRINTER` | `false` | Bypasses printer-only access restriction on `/printer/:resumeId` |
| `FLAG_DISABLE_IMAGE_PROCESSING` | `false` | Skips Sharp-based image resizing/WebP conversion on upload |

Sources: [src/utils/env.ts:1-72](), [src/vite-env.d.ts:9-57](), [docs/self-hosting/docker.mdx:282-368]()

---

## Storage Selection

**Diagram: Storage Service Selection Logic (getStorageService)**

```mermaid
flowchart TD
    Start["getStorageService()"]
    Check{"S3_ACCESS_KEY_ID\nS3_SECRET_ACCESS_KEY\nS3_BUCKET all set?"}
    S3["S3StorageService\n(uses @aws-sdk/client-s3)"]
    Local["LocalStorageService\n(writes to process.cwd()/data)"]
    Volume["Host volume mount\n./data:/app/data"]

    Start --> Check
    Check -- "Yes" --> S3
    Check -- "No" --> Local
    Local -. "requires persistent\nvolume mount" .-> Volume
```

The factory function `createStorageService` in [src/integrations/orpc/services/storage.ts:308-314]() selects the backend at startup and caches it. If `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, and `S3_BUCKET` are all present, `S3StorageService` is used; otherwise `LocalStorageService` stores files under `<cwd>/data` (i.e., `/app/data` inside the container).

When using local storage, **a persistent volume or bind mount is required**. Without one, all uploaded files (profile pictures, PDFs, screenshots) are lost when the container is recreated. The canonical mount is:

```yaml
volumes:
  - ./data:/app/data
  # or a named volume:
  - reactive_resume_data:/app/data
```

Sources: [src/integrations/orpc/services/storage.ts:113-314](), [compose.yml:95-97]()

---

## Printer Networking

The printer service (headless Chromium) renders the resume by fetching the `/printer/:resumeId` route from the app. This creates a two-leg network path:

**Diagram: PDF Generation Network Flow**

```mermaid
sequenceDiagram
    participant Client as "User Browser"
    participant App as "reactive_resume\n(port 3000)"
    participant Printer as "browserless\n(PRINTER_ENDPOINT)"

    Client->>App: "POST /api/printer (export PDF)"
    App->>Printer: "WebSocket/CDP session open"
    Printer->>App: "GET PRINTER_APP_URL/printer/:resumeId"
    App-->>Printer: "HTML resume page"
    Printer-->>App: "PDF bytes"
    App-->>Client: "PDF response"
```

`PRINTER_ENDPOINT` is the URL the **app** uses to reach the printer. `PRINTER_APP_URL` is the URL the **printer** uses to reach the app.

| Scenario | `PRINTER_ENDPOINT` | `PRINTER_APP_URL` |
|---|---|---|
| All in one Compose stack | `ws://browserless:3000?token=...` | `http://reactive_resume:3000` |
| Printer in Docker, app on host | `ws://localhost:4000?token=...` | `http://host.docker.internal:3000` |
| Both on host | `ws://localhost:4000?token=...` | `http://localhost:3000` |

Sources: [compose.yml:83-84](), [docs/self-hosting/docker.mdx:286-305]()

---

## Printer Options

Two headless Chromium images are supported:

| Image | Protocol | `PRINTER_ENDPOINT` value |
|---|---|---|
| `ghcr.io/browserless/chromium:latest` | WebSocket (CDP) | `ws://browserless:3000?token=<token>` |
| `chromedp/headless-shell:latest` | HTTP (Chrome DevTools) | `http://chrome:9222` |

The `browserless` image supports authentication via a `TOKEN` environment variable. Include the token as a query parameter in `PRINTER_ENDPOINT`.

To use `chromedp/headless-shell` instead, replace the `browserless` service in `compose.yml`:

```yaml
chrome:
  image: chromedp/headless-shell:latest
  restart: unless-stopped
  ports:
    - "9222:9222"
```

Then set `PRINTER_ENDPOINT=http://chrome:9222`.

Sources: [compose.yml:35-40](), [docs/self-hosting/docker.mdx:216-227]()

---

## Health Check

The `/api/health` endpoint is implemented in [src/routes/api/health.ts:1-86](). It performs three sub-checks and returns a JSON payload:

```mermaid
flowchart LR
    EP["GET /api/health"]
    DB["checkDatabase()\nSELECT 1 via Drizzle"]
    PR["checkPrinter()\nprinterService.healthcheck()"]
    ST["checkStorage()\ngetStorageService().healthcheck()"]
    RES["HTTP 200 (healthy)\nor HTTP 500 (unhealthy)"]

    EP --> DB
    EP --> PR
    EP --> ST
    DB --> RES
    PR --> RES
    ST --> RES
```

If any sub-check returns `status: "unhealthy"`, the response status is `500`. The Docker Compose health check polls this endpoint:

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:3000/api/health"]
  start_period: 10s
  interval: 30s
  timeout: 10s
  retries: 3
```

Reverse proxies (Traefik in particular) automatically remove unhealthy containers from the routing pool without additional configuration.

Sources: [src/routes/api/health.ts:1-86](), [compose.yml:107-111]()

---

## Startup Sequence and Migrations

On every container start, the server runs pending database migrations automatically before serving traffic. This is implemented as a Nitro plugin at `plugins/1.migrate.ts`. If the database is unreachable or migrations fail, the container exits with an error.

**Diagram: Container Startup Sequence**

```mermaid
sequenceDiagram
    participant PG as "postgres\n(service_healthy)"
    participant Chrome as "browserless\n(service_healthy)"
    participant SWInit as "seaweedfs_create_bucket\n(service_completed_successfully)"
    participant App as "reactive_resume"

    PG-->>App: "healthcheck passes"
    Chrome-->>App: "healthcheck passes"
    SWInit-->>App: "bucket creation complete"
    App->>PG: "run Drizzle migrations\n(plugins/1.migrate.ts)"
    App->>App: "start Nitro server\nport 3000"
```

Sources: [compose.yml:99-106](), [docs/self-hosting/docker.mdx:255-259]()

---

## Reverse Proxy Configurations

### Traefik

Traefik discovers services via Docker labels and handles SSL automatically with Let's Encrypt. Only the `reactive_resume` service gets exposed; `postgres` and `browserless` remain on an internal network.

Key labels for the `reactive_resume` service:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.reactive-resume.rule=Host(`resume.${DOMAIN}`)"
  - "traefik.http.routers.reactive-resume.entrypoints=websecure"
  - "traefik.http.routers.reactive-resume.tls.certresolver=letsencrypt"
  - "traefik.http.services.reactive-resume.loadbalancer.server.port=3000"
```

Required environment variables alongside standard ones:

```bash
DOMAIN=example.com
ACME_EMAIL=admin@example.com
APP_URL=https://resume.${DOMAIN}
PRINTER_APP_URL=http://reactive_resume:3000
```

When using Traefik, set `APP_URL` to the public HTTPS URL. Traefik automatically respects Docker health checks — unhealthy containers are excluded from routing without extra configuration.

Sources: [docs/self-hosting/examples.mdx:18-146]()

### nginx

nginx requires manual SSL certificate provisioning (e.g., via certbot). A proxy configuration for the `reactive_resume` upstream:

```nginx
upstream reactive_resume {
    server reactive_resume:3000;
}

server {
    listen 443 ssl http2;
    server_name resume.example.com;

    location / {
        proxy_pass http://reactive_resume;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Forwarded-Proto $scheme;

        # Required for PDF generation (long-running requests)
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    client_max_body_size 10M;
}
```

The `client_max_body_size 10M` matches the file upload limit enforced server-side in [src/integrations/orpc/router/storage.ts:8](). The upgrade headers are required for any WebSocket connections.

Sources: [docs/self-hosting/examples.mdx:149-295]()

---

## Docker Swarm

For multi-node deployments, use the Swarm compose configuration from [docs/self-hosting/examples.mdx:307-450]().

Key differences from a standard Compose deployment:

| Aspect | Docker Compose | Docker Swarm |
|---|---|---|
| Network driver | `bridge` | `overlay` |
| Service definition | `restart:` | `deploy.mode: replicated` |
| Traefik labels | `labels:` on service | `deploy.labels:` on service |
| Deployment command | `docker compose up -d` | `docker stack deploy -c compose-swarm.yml reactive_resume` |
| Log viewing | `docker compose logs -f` | `docker service logs -f reactive_resume_app` |

The app service can be scaled horizontally because all persistent state lives in PostgreSQL and the configured storage backend (S3 or local volume):

```bash
docker service scale reactive_resume_app=3
```

> **Note:** Scaling to multiple replicas while using `LocalStorageService` is unsupported — each replica would have its own isolated filesystem. Use S3-compatible storage when running multiple replicas.

Sources: [docs/self-hosting/examples.mdx:299-472]()

---

## Updating

```bash
# Pull latest images for all services
docker compose pull

# Recreate containers with new images
docker compose up -d

# Optional: remove dangling images
docker image prune -f
```

Database migrations run automatically on startup. No manual migration step is required.

Sources: [docs/self-hosting/docker.mdx:371-389]()

---

## Backups

### Database

All user accounts, resumes, and application state are stored in PostgreSQL. Use `pg_dump` for periodic backups:

```bash
docker compose exec postgres pg_dump -U postgres postgres > backup-$(date +%F).sql
```

Restore with:

```bash
docker compose exec -T postgres psql -U postgres postgres < backup.sql
```

### File Uploads

| Storage backend | Backup approach |
|---|---|
| `LocalStorageService` (`/app/data`) | Include the `./data` bind-mount directory in regular `rsync` or filesystem snapshot routines |
| `S3StorageService` | Enable bucket versioning; use S3 lifecycle rules and cross-region replication as needed |

Sources: [docs/self-hosting/docker.mdx:391-404]()

---

## Troubleshooting

| Symptom | Common cause | Fix |
|---|---|---|
| App container exits immediately | `DATABASE_URL` is wrong or DB is not ready; migration failed | Check logs with `docker compose logs -f reactive_resume` |
| Sign-in loops / cookies don't persist | `APP_URL` doesn't match the URL the browser is actually using | Set `APP_URL` to the exact public URL (include scheme and domain) and restart |
| PDF export fails or hangs | App cannot reach printer, or printer cannot reach app | Verify `PRINTER_ENDPOINT` and `PRINTER_APP_URL`; check `docker compose logs browserless` |
| Uploads disappear after restart | `/app/data` is not mounted to a persistent volume | Add `./data:/app/data` bind mount and redeploy |
| Emails not delivered | SMTP not configured (expected behavior) | Configure `SMTP_HOST` and related vars; without them, emails are logged to stdout |
| `ENOTFOUND <bucket>.<endpoint>` S3 error | S3 client is using virtual-hosted-style URLs but the server expects path-style | Set `S3_FORCE_PATH_STYLE=true` |

Sources: [docs/self-hosting/docker.mdx:451-489]()

---

