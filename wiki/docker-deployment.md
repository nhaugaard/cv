# Page: Docker Deployment

# Docker Deployment

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.dockerignore](.dockerignore)
- [.github/workflows/docker-build.yml](.github/workflows/docker-build.yml)
- [CLAUDE.md](CLAUDE.md)
- [Dockerfile](Dockerfile)
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



This page covers the official Docker image for Reactive Resume — its build process, layer structure, image registries, and the production Docker Compose stack including service dependencies, health checks, and volume configuration.

For a step-by-step self-hosting walkthrough and reverse-proxy integration examples, see [Self-Hosting Guide](#5.2). For a full reference of every supported environment variable, see [Environment Configuration](#5.3). For the GitHub Actions CI/CD workflow that builds and publishes the image, see [CI/CD Pipeline](#5.4).

---

## The Docker Image

The official image is published to two registries:

| Registry | Image |
|---|---|
| Docker Hub | `amruthpillai/reactive-resume:latest` |
| GitHub Container Registry | `ghcr.io/amruthpillai/reactive-resume:latest` |

Both registries carry identical content and are updated by the same CI workflow. Images are signed with [Cosign](https://docs.sigstore.dev/cosign/overview/) for supply-chain verification.

Available tags follow [Semantic Versioning](https://semver.org/):

| Tag pattern | Example | Description |
|---|---|---|
| `latest` | `latest` | Most recent release |
| `vMAJOR.MINOR.PATCH` | `v4.3.1` | Exact version |
| `vMAJOR.MINOR` | `v4.3` | Latest patch for minor |
| `vMAJOR` | `v4` | Latest minor for major |
| `sha-<commit>` | `sha-a1b2c3d` | Per-commit build |

The image is built for two platforms in parallel:

- `linux/amd64` (standard cloud VMs, x86 desktops)
- `linux/arm64` (ARM servers, Apple Silicon, Raspberry Pi 4+)

Sources: [README.md:186-194](), [.github/workflows/docker-build.yml:18-24]()

---

## Dockerfile: Multi-Stage Build

The build is defined in [Dockerfile:1-60]() and uses three named stages. Each stage uses `node:24-slim` as its base.

**Build stage diagram:**

```mermaid
flowchart TD
    subgraph "stage: dependencies"
        D1["Copy package.json + pnpm-lock.yaml"]
        D2["pnpm install --frozen-lockfile (dev)"]
        D3["pnpm install --frozen-lockfile --prod"]
        D1 --> D2
        D1 --> D3
    end

    subgraph "stage: builder"
        B1["Copy dev node_modules from dependencies"]
        B2["Copy full source tree"]
        B3["pnpm run build -> .output/"]
        B1 --> B2 --> B3
    end

    subgraph "stage: runtime"
        R1["Copy .output/ from builder"]
        R2["Copy migrations/ from builder"]
        R3["Copy prod node_modules from dependencies"]
        R4["EXPOSE 3000"]
        R5["HEALTHCHECK /api/health"]
        R6["ENTRYPOINT node .output/server/index.mjs"]
        R1 & R2 & R3 --> R4 --> R5 --> R6
    end

    D2 --> B1
    D3 --> R3
    B3 --> R1
    B3 --> R2
```

Sources: [Dockerfile:1-60]()

### Stage Details

**`dependencies` stage** ([Dockerfile:4-16]())

Installs dependencies twice in isolated directories (`/tmp/dev` and `/tmp/prod`) so that the dev dependencies (needed for the build) and production dependencies (needed at runtime) can be copied independently into downstream stages. This avoids shipping devDependencies in the final image.

**`builder` stage** ([Dockerfile:19-30]())

Copies the full source tree plus the dev `node_modules`, then runs `pnpm run build`. The Nitro bundler outputs the server to `.output/server/index.mjs` and the client assets to `.output/public/`.

**`runtime` stage** ([Dockerfile:33-60]())

The final image. It contains:
- `.output/` — compiled server and client bundle
- `migrations/` — Drizzle SQL migration files (applied at startup by `plugins/1.migrate.ts`)
- `node_modules/` — production-only dependencies
- `curl` — installed via `apt-get`, required for the health check

The runtime stage sets `NODE_ENV=production` ([Dockerfile:49]()) and exposes port `3000`.

**Files excluded from the Docker build context** are listed in [.dockerignore:1-13](), including `node_modules`, `.output`, `.env*`, `dist`, `docs`, and local data directories.

Sources: [Dockerfile:1-60](), [.dockerignore:1-13]()

---

## Health Check

The image declares a Docker health check that polls the `/api/health` HTTP endpoint:

```
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:3000/api/health || exit 1
```

[Dockerfile:57-58]()

The `start-period=60s` gives the container time to run database migrations before health checks begin.

The `/api/health` route is implemented in [src/routes/api/health.ts:1-86](). It runs three sub-checks in parallel and returns a JSON response:

| Sub-check | Function | What it verifies |
|---|---|---|
| Database | `checkDatabase` | `SELECT 1` via Drizzle ORM |
| Printer | `checkPrinter` | Calls `printerService.healthcheck()` |
| Storage | `checkStorage` | Calls `storageService.healthcheck()` |

If any sub-check reports `status: "unhealthy"`, the overall response is HTTP 500. A healthy response is HTTP 200.

**Health check data flow diagram:**

```mermaid
flowchart LR
    Docker["Docker daemon\n(HEALTHCHECK)"] -->|"GET /api/health"| HealthRoute["Route: /api/health\nsrc/routes/api/health.ts"]
    HealthRoute --> checkDatabase["checkDatabase()\n-> db.execute(sql SELECT 1)"]
    HealthRoute --> checkPrinter["checkPrinter()\n-> printerService.healthcheck()"]
    HealthRoute --> checkStorage["checkStorage()\n-> storageService.healthcheck()"]
    checkDatabase -->|"healthy / unhealthy"| Response["JSON response\nHTTP 200 or 500"]
    checkPrinter -->|"healthy / unhealthy"| Response
    checkStorage -->|"healthy / unhealthy"| Response
```

Sources: [src/routes/api/health.ts:1-86](), [Dockerfile:57-58]()

---

## Production Docker Compose Stack

The canonical production configuration is in [compose.yml:1-117](). It defines five services under the Compose project name `reactive_resume`.

### Service Overview

| Service | Image | Role |
|---|---|---|
| `postgres` | `postgres:latest` | PostgreSQL database |
| `browserless` | `ghcr.io/browserless/chromium:latest` | Headless Chromium for PDF/screenshot generation |
| `seaweedfs` | `chrislusf/seaweedfs:latest` | S3-compatible object storage |
| `seaweedfs_create_bucket` | `quay.io/minio/mc:latest` | One-shot bucket initialization job |
| `reactive_resume` | `amruthpillai/reactive-resume:latest` | The application |

### Service Dependency Graph

The startup order is enforced with `depends_on` and condition checks:

```mermaid
flowchart TD
    postgres["postgres\n(postgres:latest)"]
    browserless["browserless\n(ghcr.io/browserless/chromium:latest)"]
    seaweedfs["seaweedfs\n(chrislusf/seaweedfs:latest)"]
    seaweedfs_create_bucket["seaweedfs_create_bucket\n(quay.io/minio/mc:latest)"]
    reactive_resume["reactive_resume\n(amruthpillai/reactive-resume:latest)"]

    seaweedfs -->|"service_healthy"| seaweedfs_create_bucket
    postgres -->|"service_healthy"| reactive_resume
    browserless -->|"service_healthy"| reactive_resume
    seaweedfs_create_bucket -->|"service_completed_successfully"| reactive_resume
```

Sources: [compose.yml:59-111]()

### Health Check Configuration

Each service declares its own health check. The `reactive_resume` container will not start until all three dependencies are healthy (or, in the case of `seaweedfs_create_bucket`, have exited successfully).

| Service | Health check command | Interval | Retries |
|---|---|---|---|
| `postgres` | `pg_isready -U postgres -d postgres` | 30s | 3 |
| `browserless` | `curl -f http://localhost:3000/pressure?token=...` | 10s | 10 |
| `seaweedfs` | `wget -q -O /dev/null http://localhost:8888` | 30s | 3 |
| `reactive_resume` | `curl -f http://localhost:3000/api/health` | 30s | 3 |

Sources: [compose.yml:13-17](), [compose.yml:27-33](), [compose.yml:52-57](), [compose.yml:107-111]()

### The `seaweedfs_create_bucket` Service

This is a one-shot job ([compose.yml:59-72]()) that:
1. Waits 5 seconds for SeaweedFS to stabilize.
2. Runs `mc alias set` to register the SeaweedFS S3 endpoint.
3. Creates the `reactive-resume` bucket via `mc mb`.
4. Exits with code `0`.

`restart: on-failure` ensures it retries if the bucket creation fails. The main application container will not start until this job reports `service_completed_successfully`.

### Alternative Printer

A commented-out block in [compose.yml:35-38]() shows how to substitute `chromedp/headless-shell` for `browserless`. When using this alternative, set `PRINTER_ENDPOINT=http://chrome:9222` instead of the WebSocket URL.

Sources: [compose.yml:35-38](), [compose.yml:83-84]()

---

## Volume Configuration

The production Compose stack declares three named volumes:

| Volume | Mounted at | Purpose |
|---|---|---|
| `postgres_data` | `/var/lib/postgresql` (in `postgres`) | Persists all PostgreSQL data |
| `seaweedfs_data` | `/data` (in `seaweedfs`) | Persists object storage files |
| `reactive_resume_data` | `/app/data` (in `reactive_resume`) | Persists local file uploads when S3 is not configured |

`reactive_resume_data` is only used when the `S3_*` environment variables are absent. In that case, the `LocalStorageService` class ([src/integrations/orpc/services/storage.ts:113-208]()) writes files to the `/app/data` directory. Without a persistent volume at this path, uploaded files are lost on container recreation.

If S3 credentials are provided, the `S3StorageService` class ([src/integrations/orpc/services/storage.ts:210-306]()) is used instead, and the local volume is unused. The storage backend is selected at runtime by the `getStorageService()` factory function ([src/integrations/orpc/services/storage.ts:308-323]()).

Sources: [compose.yml:113-116](), [src/integrations/orpc/services/storage.ts:308-323]()

---

## Key Environment Variables

The `reactive_resume` service in `compose.yml` sets these environment variables. All are defined and validated by `src/utils/env.ts` using `@t3-oss/env-core` and Zod.

| Variable | Required | Default in compose.yml | Purpose |
|---|---|---|---|
| `APP_URL` | Yes | `http://localhost:3000` | Canonical public URL of the application |
| `PRINTER_APP_URL` | No | `http://reactive_resume:3000` | URL the printer uses to reach the app internally |
| `PRINTER_ENDPOINT` | Yes | `ws://browserless:3000?token=...` | WebSocket/HTTP URL of the headless Chromium service |
| `DATABASE_URL` | Yes | `postgresql://postgres:postgres@postgres:5432/postgres` | PostgreSQL connection string |
| `AUTH_SECRET` | Yes | `change-me-...` | Secret for session signing — must be changed in production |
| `S3_ACCESS_KEY_ID` | No | `seaweedfs` | S3 credentials (here: SeaweedFS) |
| `S3_SECRET_ACCESS_KEY` | No | `seaweedfs` | S3 credentials |
| `S3_ENDPOINT` | No | `http://seaweedfs:8333` | S3-compatible endpoint |
| `S3_BUCKET` | No | `reactive-resume` | Bucket name |
| `S3_FORCE_PATH_STYLE` | No | `true` | Required for SeaweedFS, MinIO, and other self-hosted S3 |

All accepted variables (including SMTP, OAuth, and feature flags) are validated in [src/utils/env.ts:1-72](). For the full reference, see [Environment Configuration](#5.3).

**Critical production note:** `AUTH_SECRET` must be replaced with a cryptographically random value. Changing it after deployment invalidates all active sessions.

Sources: [compose.yml:76-98](), [src/utils/env.ts:1-72]()

---

## Startup Sequence and Database Migrations

On every container start, the Nitro server plugin at `plugins/1.migrate.ts` runs all pending Drizzle migrations before the HTTP server begins accepting traffic. The migration files are copied into the image from the `migrations/` directory during the `runtime` build stage.

If migrations fail (for example, due to a bad `DATABASE_URL`), the process exits and the container stops. Docker marks it as unhealthy, and dependent services that have not yet started will not proceed.

The `start-period=60s` in the Dockerfile's `HEALTHCHECK` directive ([Dockerfile:57]()) accounts for this migration window.

Sources: [Dockerfile:52](), [Dockerfile:57-58]()

---

## Development vs. Production Compose Files

Two Compose files are present in the repository:

| File | Purpose |
|---|---|
| `compose.yml` | Production stack — includes the `reactive_resume` application container |
| `compose.dev.yml` | Development infrastructure only — no application container |

`compose.dev.yml` exposes service ports to the host (PostgreSQL on `5432`, SeaweedFS on `8333`, browserless on `4000`) and additionally includes `mailpit` for email testing and `adminer` for database inspection. The application itself runs directly on the host via `pnpm dev` during development.

For details on the development environment setup, see [Development Setup](#6.1).

Sources: [compose.dev.yml:1-114](), [compose.yml:1-117]()

---

