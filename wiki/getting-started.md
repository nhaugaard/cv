# Page: Getting Started

# Getting Started

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



This page walks through running Reactive Resume for the first time using Docker Compose. It covers prerequisites, the service stack defined in `compose.yml`, required environment variables, and first-run verification via the `/api/health` endpoint. For comprehensive deployment options, see [Docker Deployment](#5.1) and [Environment Configuration](#5.3). For local development setup, see [Development Setup](#6.1).

## Prerequisites

All deployment methods require:

- **Docker Engine**: Version 20.10 or higher
- **Docker Compose**: Version 2.0 or higher (or Docker Desktop with Compose plugin)

For local development, additionally install:

- **Node.js**: Version 24 or higher
- **pnpm**: Version 10.28.0 or higher
- **Git**: Any recent version

Minimum hardware recommendations:
- 2 vCPU / 2 GB RAM (4 GB recommended if running all services on single host)
- 10-20 GB storage for database and file uploads

Sources: [docs/self-hosting/docker.mdx:31-42](), [docs/contributing/development.mdx:6-12]()

## Deployment Architecture

The following diagram shows the production service topology as defined in `compose.yml`:

**Production service topology (`compose.yml`)**

```mermaid
graph TB
    Browser["Browser :3000"]

    subgraph compose_yml["compose.yml"]
        App["reactive_resume\namruthpillai/reactive-resume:latest\n:3000"]
        Printer["browserless\nghcr.io/browserless/chromium:latest\n:3000 (internal)"]
        DB["postgres\npostgres:latest\n:5432"]
        Storage["seaweedfs\nchrislusf/seaweedfs:latest\n:8333 S3 API"]
        BucketInit["seaweedfs_create_bucket\nquay.io/minio/mc:latest\n(one-shot init)"]
    end

    Browser -- "HTTP" --> App
    App -- "DATABASE_URL\nDrizzle ORM" --> DB
    App -- "S3_ENDPOINT\nS3StorageService" --> Storage
    App -- "PRINTER_ENDPOINT\nws://" --> Printer
    Printer -- "PRINTER_APP_URL\nrender /printer/:resumeId" --> App
    BucketInit -- "mc mb reactive-resume" --> Storage
```

`compose.yml` defines the full production stack. `compose.dev.yml` adds Adminer and Mailpit for local development and exposes all service ports to the host.

Sources: [compose.yml:1-117](), [compose.dev.yml:1-114](), [docs/self-hosting/docker.mdx:180-185]()

## Production Deployment with Docker

This section describes deploying Reactive Resume using `compose.yml`.

### Step 1: Clone Repository

```bash
git clone https://github.com/amruthpillai/reactive-resume.git
cd reactive-resume
```

Sources: [README.md:136-139](), [docs/getting-started/quickstart.mdx:74-78]()

### Step 2: Configure Environment

Create a `.env` file with required variables:

| Variable | Description | Example |
|----------|-------------|---------|
| `APP_URL` | Public URL of the application | `http://localhost:3000` |
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://postgres:postgres@postgres:5432/postgres` |
| `PRINTER_ENDPOINT` | WebSocket or HTTP URL to printer service | `ws://browserless:3000?token=1234567890` |
| `AUTH_SECRET` | Secret for session encryption (32+ chars) | Generate with `openssl rand -hex 32` |

Optional but recommended:

| Variable | Description | Default |
|----------|-------------|---------|
| `PRINTER_APP_URL` | Internal URL for printer to reach app | Uses `APP_URL` |
| `S3_ACCESS_KEY_ID` | S3 access key | Falls back to local filesystem |
| `S3_SECRET_ACCESS_KEY` | S3 secret key | Falls back to local filesystem |
| `S3_ENDPOINT` | S3-compatible endpoint | Required if using S3 |
| `S3_BUCKET` | S3 bucket name | Required if using S3 |
| `S3_FORCE_PATH_STYLE` | Use path-style URLs (`true` for MinIO/SeaweedFS) | `false` |

For a complete environment variable reference, see [Environment Configuration](#5.3).

Sources: [docs/self-hosting/docker.mdx:56-124](), [src/vite-env.d.ts:9-56](), [src/utils/env.ts:1-73]()

### Step 3: Start Services

```bash
docker compose up -d
```

This command starts all services defined in [compose.yml:1-117]():
- `postgres`: PostgreSQL database on port 5432
- `browserless`: Headless Chromium for PDF generation on port 4000
- `seaweedfs`: S3-compatible storage on port 8333
- `seaweedfs_create_bucket`: One-time bucket initialization
- `reactive_resume`: Main application on port 3000

All services include health checks. Database migrations run automatically on application startup via [plugins/1.migrate.ts]().

Sources: [README.md:141-146](), [compose.yml:1-117](), [docs/self-hosting/docker.mdx:234-252]()

### Step 4: Verify Deployment

Check service health:

```bash
docker compose ps
```

All services should show `healthy` status. Access the application at the configured `APP_URL` (default: `http://localhost:3000`).

The application exposes a health check endpoint at `/api/health` that verifies database, printer, and storage connectivity.

Sources: [compose.yml:106-111](), [src/routes/api/health.ts:1-87](), [docs/self-hosting/docker.mdx:406-449]()

## Local Development Setup

For setting up a full local development environment (with `pnpm dev`, hot reload, Adminer, Mailpit, and Drizzle Studio), see [Development Setup](#6.1). The development compose file is `compose.dev.yml`, which exposes all service ports to the host and adds tooling services not present in the production stack.

Sources: [compose.dev.yml:1-114](), [docs/contributing/development.mdx:40-60]()

## Environment Variable Schema

Environment variables are validated using `@t3-oss/env-core` and Zod schemas at startup via `src/utils/env.ts`. Invalid configuration causes the application to exit with validation errors.

### Required Variables

These must be set or the application will not start [src/utils/env.ts:14-24]():

| Variable | Zod Validation | Notes |
|----------|---------------|-------|
| `APP_URL` | `z.url({ protocol: /https?/ })` | Public URL; affects auth redirects |
| `DATABASE_URL` | `z.url({ protocol: /postgres(ql)?/ })` | PostgreSQL connection string |
| `PRINTER_ENDPOINT` | `z.url({ protocol: /^(wss?|https?)$/ })` | WebSocket or HTTP URL to Chromium |
| `AUTH_SECRET` | `z.string().min(1)` | Changing this invalidates all sessions |

### Storage Configuration

The `getStorageService()` function in `src/integrations/orpc/services/storage.ts` selects a storage backend at startup based on environment variables:

**`getStorageService()` storage backend selection**

```mermaid
graph LR
    getStorageService["getStorageService()"]
    EnvCheck{"S3_ACCESS_KEY_ID\nS3_SECRET_ACCESS_KEY\nS3_BUCKET\nall set?"}
    LocalStorage["LocalStorageService\nwrites to /app/data\n(node:fs/promises)"]
    S3Storage["S3StorageService\n@aws-sdk/client-s3\nPutObjectCommand\nGetObjectCommand\nDeleteObjectCommand"]

    getStorageService --> EnvCheck
    EnvCheck -- "No" --> LocalStorage
    EnvCheck -- "Yes" --> S3Storage
```

If S3 variables are not set, files are stored under `/app/data`. This directory must be mounted to persistent storage (e.g. `./data:/app/data` volume) to prevent data loss on container restart.

Sources: [src/integrations/orpc/services/storage.ts:308-323](), [src/utils/env.ts:14-24](), [docs/self-hosting/docker.mdx:349-360]()

### Feature Flags

Boolean flags control optional features:

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `FLAG_DEBUG_PRINTER` | `stringbool` | `false` | Bypasses printer-only access restrictions |
| `FLAG_DISABLE_SIGNUPS` | `stringbool` | `false` | Disables new user registration |
| `FLAG_DISABLE_EMAIL_AUTH` | `stringbool` | `false` | Disables email/password authentication |
| `FLAG_DISABLE_IMAGE_PROCESSING` | `stringbool` | `false` | Disables image resizing (useful for low-resource systems) |

The `stringbool` type accepts both boolean values and string representations (`"true"`, `"false"`).

Sources: [src/utils/env.ts:66-71](), [docs/self-hosting/docker.mdx:362-367]()

## Health Check System

The `GET /api/health` route is defined in `src/routes/api/health.ts` and performs three subsystem checks in parallel. Docker Compose health checks in `compose.yml` poll this endpoint.

**`/api/health` route — `src/routes/api/health.ts`**

```mermaid
graph TB
    HealthRoute["Route /api/health\n(GET handler)"]

    HealthRoute --> checkDatabase["checkDatabase()\nexecutes SELECT 1\nvia db (Drizzle)"]
    HealthRoute --> checkPrinter["checkPrinter()\nprinterService.healthcheck()"]
    HealthRoute --> checkStorage["checkStorage()\ngetStorageService().healthcheck()"]

    checkDatabase --> agg{"any check\nunhealthy?"}
    checkPrinter --> agg
    checkStorage --> agg

    agg -- "No" --> HTTP200["HTTP 200\nstatus: healthy"]
    agg -- "Yes" --> HTTP500["HTTP 500\nstatus: unhealthy"]
```

Each check returns `{ status: "healthy" | "unhealthy", message: string, error?: string }`. The aggregate returns HTTP 500 if any subsystem is unhealthy.

Sources: [src/routes/api/health.ts:17-78](), [compose.yml:106-111]()

## Printer Service Configuration

The printer service generates PDFs and screenshots using headless Chromium. Two options are supported:

### Option 1: Browserless (Recommended)

Full-featured browser automation platform with queue management and resource limits:

```yaml
browserless:
  image: ghcr.io/browserless/chromium:latest
  environment:
    - QUEUED=10
    - CONCURRENT=5
    - TOKEN=1234567890
```

Set `PRINTER_ENDPOINT` to `ws://browserless:3000?token=1234567890` (or `ws://localhost:4000?token=1234567890` for local development).

Sources: [compose.yml:20-33](), [compose.dev.yml:37-52]()

### Option 2: Lightweight Chrome

Minimal headless Chrome with smaller image size:

```yaml
chrome:
  image: chromedp/headless-shell:latest
  ports:
    - "9222:9222"
```

Set `PRINTER_ENDPOINT` to `http://chrome:9222` (or `http://localhost:9222` for local development).

Sources: [compose.yml:37-41](), [compose.dev.yml:56-60]()

The printer service connection is validated by the health check system. The printer communicates back to the application using `PRINTER_APP_URL` (or `APP_URL` if not set) to render resume pages.

Sources: [docs/self-hosting/docker.mdx:289-304](), [src/vite-env.d.ts:14]()

## Common Commands

```bash
# Start the production stack (compose.yml)
docker compose up -d

# Stream application logs
docker compose logs -f reactive_resume

# Check service health status
docker compose ps
curl http://localhost:3000/api/health

# Update to latest image and restart
docker compose pull
docker compose up -d

# Remove unused images after update
docker image prune -f

# Stop the stack
docker compose down
```

Sources: [README.md:141-146](), [docs/self-hosting/docker.mdx:234-252](), [docs/self-hosting/docker.mdx:371-389]()

## Troubleshooting

### Application Exits Immediately

The most common cause is failed database migrations due to invalid `DATABASE_URL`. Check logs:

```bash
docker compose logs -f reactive_resume
```

Migrations run automatically on startup via [plugins/1.migrate.ts](). If the database is unreachable or credentials are invalid, the container exits.

Sources: [docs/self-hosting/docker.mdx:453-460]()

### PDF Generation Fails

Check printer connectivity:
1. Verify `PRINTER_ENDPOINT` is reachable from the application container
2. For local development, ensure `PRINTER_APP_URL=http://host.docker.internal:3000` is set
3. Check printer service health: `docker compose logs -f browserless`

Sources: [docs/self-hosting/docker.mdx:467-472]()

### Uploads Disappear After Restart

If not using S3, the application stores files in `/app/data`. Without a volume mount, this data is lost on container restart:

```yaml
volumes:
  - ./data:/app/data
```

Sources: [docs/self-hosting/docker.mdx:474-477](), [compose.yml:95-96]()

### S3 Connection Error: ENOTFOUND

If the error message shows `ENOTFOUND bucket.endpoint`, the S3 client is using virtual-hosted-style addressing but your storage expects path-style. Set:

```bash
S3_FORCE_PATH_STYLE=true
```

This is required for most self-hosted S3-compatible services (MinIO, SeaweedFS).

Sources: [docs/self-hosting/docker.mdx:484-488](), [src/utils/env.ts:62-64]()

## Next Steps

- For detailed deployment options and reverse proxy configuration, see [Docker Deployment](#5.1)
- For production environment configuration reference, see [Environment Configuration](#5.3)
- For development workflows and contributing guidelines, see [Development Setup](#6.1)
- For system architecture and component interaction, see [Architecture Overview](#1.2)

Sources: [README.md:166-176](), [docs/getting-started/quickstart.mdx:210-227]()

---

