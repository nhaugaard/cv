# Page: Development Setup

# Development Setup

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.devcontainer/Dockerfile](.devcontainer/Dockerfile)
- [.devcontainer/devcontainer.json](.devcontainer/devcontainer.json)
- [.devcontainer/docker-compose.yml](.devcontainer/docker-compose.yml)
- [.env.example](.env.example)
- [CLAUDE.md](CLAUDE.md)
- [compose.dev.yml](compose.dev.yml)
- [compose.yml](compose.yml)
- [docs/contributing/development.mdx](docs/contributing/development.mdx)
- [docs/getting-started/quickstart.mdx](docs/getting-started/quickstart.mdx)
- [docs/self-hosting/docker.mdx](docs/self-hosting/docker.mdx)
- [docs/self-hosting/examples.mdx](docs/self-hosting/examples.mdx)
- [package.json](package.json)
- [pnpm-lock.yaml](pnpm-lock.yaml)
- [src/integrations/orpc/router/storage.ts](src/integrations/orpc/router/storage.ts)
- [src/integrations/orpc/services/storage.ts](src/integrations/orpc/services/storage.ts)
- [src/routes/__root.tsx](src/routes/__root.tsx)
- [src/routes/api/health.ts](src/routes/api/health.ts)
- [src/utils/env.ts](src/utils/env.ts)
- [src/vite-env.d.ts](src/vite-env.d.ts)

</details>



This document describes how to set up a local development environment for Reactive Resume. It covers two approaches: using VS Code's devcontainer for a fully containerized development experience, or running the application locally while supporting services run in Docker.

For deployment configurations, see [Docker Deployment](#5.1). For build configuration details, see [Build System](#6.2).

---

## Prerequisites

The following tools must be installed on your development machine:

| Tool | Minimum Version | Purpose |
|------|----------------|---------|
| Node.js | 20.x | Runtime for the application |
| pnpm | 10.28.0 | Package manager (uses workspaces) |
| Docker | 20.10+ | Container runtime |
| Docker Compose | 2.0+ | Multi-container orchestration |
| Git | Any recent version | Version control |

**Sources:** [docs/contributing/development.mdx:6-11](), [CLAUDE.md:1-7]()

---

## Development Environment Options

Reactive Resume supports two development setups:

### Option 1: VS Code Devcontainer (Recommended)

The devcontainer provides a complete, pre-configured development environment inside a Docker container. All dependencies, services, and tooling are automatically configured.

**Configuration Location:** [.devcontainer/devcontainer.json]()

**Container Image:** Based on `mcr.microsoft.com/devcontainers/typescript-node:24` with `corepack` pre-enabled for pnpm support. See [.devcontainer/Dockerfile:1-10]().

**VS Code Extensions:** Automatically installs:
- `biomejs.biome` — Linting and formatting
- `bradlc.vscode-tailwindcss` — Tailwind CSS IntelliSense
- `lokalise.i18n-ally` — Translation management

**Port Forwarding:** The devcontainer automatically forwards these ports to your host machine:

| Port | Service | Auto-Open |
|------|---------|-----------|
| 3000 | Reactive Resume (main app) | Yes (browser) |
| 4000 | Browserless (printer) | No |
| 5432 | PostgreSQL | No |
| 8333 | SeaweedFS (S3) | No |

**Bundled Services:** The devcontainer's own Docker Compose ([.devcontainer/docker-compose.yml]()) starts `postgres`, `browserless`, `seaweedfs`, and `seaweedfs_create_bucket`. Adminer and Mailpit are **not** included — they are only available when using `compose.dev.yml` directly (Option 2).

**Post-Create Command:** After the container is built, `corepack enable && pnpm install` runs automatically to install all Node.js dependencies.

**Sources:** [.devcontainer/devcontainer.json:1-32](), [.devcontainer/Dockerfile:1-10](), [.devcontainer/docker-compose.yml:1-97]()

### Option 2: Local Development with Docker Services

Run the application directly on your host machine while infrastructure services run in Docker containers. This approach gives faster hot reload and easier debugging, and additionally provides Adminer and Mailpit which are not available in the devcontainer setup.

**Services Configuration:** [compose.dev.yml]()

This is the recommended approach when:
- You prefer your native Node.js installation
- You want faster rebuild times during development
- You need the Adminer database browser or Mailpit email inspector

**Sources:** [CLAUDE.md:42-53](), [docs/contributing/development.mdx:40-59]()

---

## Development Services Architecture

The following diagram shows the two modes and which services each provides. Code identifiers in the diagram match service names and environment variable names in the codebase.

**Development Service Topology**

```mermaid
graph TB
    subgraph "Host Machine"
        IDE["VS Code / IDE"]
        Browser["Web Browser :3000"]
    end

    subgraph "compose.dev.yml (Option 2 only)"
        ADM["adminer\nPort: 8080"]
        MP["mailpit\nSMTP: 1025 / Web: 8025"]
    end

    subgraph "Shared Infrastructure (both options)"
        PG["postgres\npostgres:latest\nPort: 5432"]
        BL["browserless\nghcr.io/browserless/chromium\nPort: 4000 / TOKEN=1234567890"]
        Chrome["chrome\nchromedp/headless-shell\nPort: 9222 (alternative)"]
        SW["seaweedfs\nchrislusf/seaweedfs\nPort: 8333"]
        SWInit["seaweedfs_create_bucket\nquay.io/minio/mc (init only)"]
    end

    subgraph "devcontainer only"
        AppContainer["reactive_resume container\nmcr.microsoft.com/devcontainers/typescript-node:24"]
    end

    IDE -.->|"devcontainer"| AppContainer
    Browser -->|"HTTP"| AppContainer
    Browser -->|"HTTP (local mode)"| IDE

    AppContainer -->|"DATABASE_URL"| PG
    AppContainer -->|"PRINTER_ENDPOINT"| BL
    AppContainer -->|"S3_ENDPOINT"| SW

    IDE -->|"local mode: localhost:5432"| PG
    IDE -->|"local mode: ws://localhost:4000"| BL
    IDE -->|"local mode: http://localhost:8333"| SW
    IDE -->|"local mode: localhost:1025"| MP

    BL -.->|"PRINTER_APP_URL\n(renders resume page)"| AppContainer
    Chrome -.->|"alternative PRINTER_ENDPOINT"| AppContainer

    SWInit -->|"creates reactive-resume bucket"| SW
    ADM -->|"web UI"| PG
```

**Key Networking Considerations:**

1. **Devcontainer Mode:** The application runs inside the `reactive_resume` container and accesses other services using their Docker service names (e.g., `postgres`, `browserless`). Environment variables are pre-set in [.devcontainer/docker-compose.yml:18-30]().

2. **Local Mode:** The application runs on the host and accesses services via `localhost` with exposed ports (e.g., `localhost:5432`). Variables are read from the project-root `.env` file.

3. **Printer Service Connection:** When running locally (`pnpm dev` on host), `PRINTER_APP_URL` must be set to `http://host.docker.internal:3000` because the containerized Browserless service needs to reach back to the host machine to fetch the resume page for rendering.

**Sources:** [compose.dev.yml:1-114](), [.devcontainer/docker-compose.yml:1-97]()

---

## Required Services

### PostgreSQL Database

**Image:** `postgres:latest`

**Purpose:** Stores all application data including user accounts, resumes, sessions, and API keys.

**Schema Management:** Uses Drizzle ORM with automatic migrations on startup via [plugins/1.migrate.ts]().

**Database Schema Location:** [src/integrations/drizzle/schema.ts]()

**Connection String Format:**
```
postgresql://postgres:postgres@postgres:5432/postgres
```

**Health Check:** `pg_isready -U postgres -d postgres`

**Sources:** [compose.dev.yml:19-35](), [.devcontainer/docker-compose.yml:32-46]()

### Browserless (Printer Service)

**Image:** `ghcr.io/browserless/chromium:latest`

**Purpose:** Headless Chromium browser for generating PDF exports and resume screenshots.

**Configuration:**
- `CONCURRENT=5` - Maximum concurrent browser sessions
- `QUEUED=10` - Maximum queued requests
- `TOKEN=1234567890` - Authentication token for development

**Alternative:** The `chromedp/headless-shell` image can be used as a lighter alternative. When using this image, set `PRINTER_ENDPOINT` to `http://chrome:9222` instead of the WebSocket endpoint.

**Health Check:** `curl -f http://localhost:3000/pressure?token=1234567890`

**Sources:** [compose.dev.yml:37-52](), [compose.dev.yml:54-60]()

### SeaweedFS (S3-Compatible Storage)

**Image:** `chrislusf/seaweedfs:latest`

**Purpose:** S3-compatible object storage for file uploads (profile pictures, resume screenshots, PDF exports).

**Command:** `server -s3 -filer -dir=/data -ip=0.0.0.0`

**Credentials:**
- Access Key ID: `seaweedfs`
- Secret Access Key: `seaweedfs`

**Bucket Initialization:** The `seaweedfs_create_bucket` service automatically creates the `reactive-resume` bucket on startup using the MinIO client (`mc`).

**Health Check:** `wget -q -O /dev/null http://localhost:8888`

**Sources:** [compose.dev.yml:62-93](), [src/integrations/orpc/services/storage.ts:210-229]()

### Mailpit (Email Testing)

**Image:** `axllent/mailpit:latest`

**Purpose:** Captures all outbound emails during development for testing without sending real emails.

**Ports:**
- SMTP: `1025` - Application sends emails here
- Web UI: `8025` - View captured emails in browser

**Usage:** Access the web interface at [http://localhost:8025](http://localhost:8025) to view all emails sent by the application during development.

**Health Check:** `wget -q -O /dev/null http://localhost:8025/`

**Sources:** [compose.dev.yml:95-108]()

### Adminer (Database Management)

**Image:** `adminer:latest`

**Purpose:** Web-based database administration interface (optional, only in `compose.dev.yml`).

**Port:** `8080`

**Usage:** Access at [http://localhost:8080](http://localhost:8080) to browse database tables, run queries, and manage schema.

**Sources:** [compose.dev.yml:4-17]()

---

## Environment Variable Configuration

The application uses environment variables for all configuration. These are validated at startup using Zod schemas defined in [src/utils/env.ts:4-72]().

### Environment Variable Flow

```mermaid
graph LR
    subgraph "Configuration Sources"
        EnvFile[".env file"]
        ProcessEnv["process.env<br/>(system variables)"]
        DevContainer[".devcontainer/<br/>docker-compose.yml"]
    end
    
    subgraph "Validation Layer"
        CreateEnv["createEnv()<br/>@t3-oss/env-core"]
        ZodSchema["Zod Schemas<br/>src/utils/env.ts"]
    end
    
    subgraph "Application Access"
        EnvObject["env object<br/>(typed & validated)"]
        AppCode["Application Code<br/>imports from @/utils/env"]
    end
    
    EnvFile --> ProcessEnv
    DevContainer --> ProcessEnv
    ProcessEnv --> CreateEnv
    CreateEnv --> ZodSchema
    ZodSchema -->|validation| EnvObject
    EnvObject --> AppCode
    
    ZodSchema -.->|validation error| Exit["Application exits<br/>with error message"]
```

**Sources:** [src/utils/env.ts:1-72](), [src/vite-env.d.ts:9-56]()

### Required Variables

These variables must be set for the application to start:

| Variable | Description | Development Value |
|----------|-------------|-------------------|
| `APP_URL` | Public URL of the application | `http://localhost:3000` |
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://postgres:postgres@localhost:5432/postgres` |
| `AUTH_SECRET` | Secret for session encryption | `development-secret-change-in-production` |
| `PRINTER_ENDPOINT` | Printer service WebSocket URL | `ws://localhost:4000?token=1234567890` |

**Sources:** [docs/contributing/development.mdx:66-89](), [CLAUDE.md:44-51]()

### Local Development Specific Variables

| Variable | Description | Local Value | Why Required |
|----------|-------------|-------------|--------------|
| `PRINTER_APP_URL` | URL for printer to reach app | `http://host.docker.internal:3000` | Docker containers cannot access `localhost` on host |

When the printer service (running in Docker) needs to render a resume page for PDF generation, it must connect back to the application. The `PRINTER_APP_URL` override tells the printer where to find the application when it's running on the host machine.

**Sources:** [docs/contributing/development.mdx:91-93](), [docs/self-hosting/docker.mdx:200-202]()

### Storage Variables (SeaweedFS)

| Variable | Value | Purpose |
|----------|-------|---------|
| `S3_ACCESS_KEY_ID` | `seaweedfs` | S3 credentials |
| `S3_SECRET_ACCESS_KEY` | `seaweedfs` | S3 credentials |
| `S3_ENDPOINT` | `http://localhost:8333` | SeaweedFS S3 endpoint |
| `S3_BUCKET` | `reactive-resume` | Target bucket name |
| `S3_FORCE_PATH_STYLE` | `true` | Required for SeaweedFS and MinIO |

**Fallback Behavior:** If S3 variables are not configured, the storage service falls back to local filesystem storage at `/app/data` (or `./data` in the project root when running locally).

**Sources:** [src/integrations/orpc/services/storage.ts:308-314](), [docs/contributing/development.mdx:79-84]()

### Optional Variables

| Variable | Development Default | Purpose |
|----------|-------------------|---------|
| `SMTP_HOST` | `localhost` | Mailpit SMTP host |
| `SMTP_PORT` | `1025` | Mailpit SMTP port |
| `GOOGLE_CLIENT_ID` | (none) | Google OAuth integration |
| `GITHUB_CLIENT_ID` | (none) | GitHub OAuth integration |
| `FLAG_DEBUG_PRINTER` | `false` | Allow direct access to printer routes |
| `FLAG_DISABLE_SIGNUPS` | `false` | Disable new user registration |
| `FLAG_DISABLE_EMAIL_AUTH` | `false` | Disable email/password authentication |

**Email Behavior:** If SMTP variables are not configured, emails are logged to the console instead of being sent. In development, Mailpit is used to capture emails.

**Sources:** [src/utils/env.ts:48-70](), [docs/contributing/development.mdx:86-88]()

---

## Setup Instructions

### Using VS Code Devcontainer

**Step 1:** Open the project in VS Code.

**Step 2:** When prompted "Reopen in Container", click the button. Alternatively, use Command Palette (Ctrl/Cmd+Shift+P) and select "Dev Containers: Reopen in Container".

**Step 3:** Wait for the container to build and dependencies to install. The `postCreateCommand` automatically runs `corepack enable && pnpm install`.

**Step 4:** The development server starts automatically or run `pnpm dev` in the integrated terminal.

**Environment Variables:** Pre-configured in [.devcontainer/docker-compose.yml:18-30](). No `.env` file needed.

**Sources:** [.devcontainer/devcontainer.json:1-32](), [.devcontainer/docker-compose.yml:1-97]()

### Local Development Setup

**Step 1: Start Infrastructure Services**

```bash
docker compose -f compose.dev.yml up -d
```

This starts PostgreSQL, Browserless, SeaweedFS, Mailpit, and Adminer.

**Step 2: Verify Service Health**

```bash
docker compose -f compose.dev.yml ps
```

All services should show as `healthy` before proceeding.

**Step 3: Create Environment File**

Create a `.env` file in the project root with the following content:

```bash
# Server
APP_URL=http://localhost:3000

# Printer (required for local development)
PRINTER_APP_URL=http://host.docker.internal:3000
PRINTER_ENDPOINT=ws://localhost:4000?token=1234567890

# Database
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/postgres

# Authentication
AUTH_SECRET=development-secret-change-in-production

# Storage (SeaweedFS)
S3_ACCESS_KEY_ID=seaweedfs
S3_SECRET_ACCESS_KEY=seaweedfs
S3_ENDPOINT=http://localhost:8333
S3_BUCKET=reactive-resume
S3_FORCE_PATH_STYLE=true

# Email (Mailpit for local development)
SMTP_HOST=localhost
SMTP_PORT=1025
```

**Step 4: Install Dependencies**

```bash
pnpm install
```

**Step 5: Run Database Migrations**

```bash
pnpm db:migrate
```

Migrations are defined in the `migrations/` directory and executed via Drizzle ORM.

**Step 6: Start Development Server**

```bash
pnpm dev
```

The application will be available at [http://localhost:3000](http://localhost:3000).

**Sources:** [docs/contributing/development.mdx:20-114](), [CLAUDE.md:42-53]()

---

## Development Workflow

### Available Commands

The following npm scripts are defined in `package.json`:

#### Core Development

| Command | Description | Implementation |
|---------|-------------|----------------|
| `pnpm dev` | Start development server with hot reload | TanStack Start dev server on port 3000 |
| `pnpm build` | Build for production | Creates `.output/` directory via Nitro |
| `pnpm start` | Start production build | Runs `node .output/server/index.mjs` |
| `pnpm lint` | Run Biome linter and formatter | Checks and fixes code style issues |
| `pnpm typecheck` | Run TypeScript type checking | Uses `tsgo` for type validation |

#### Database Management

| Command | Description | When to Use |
|---------|-------------|-------------|
| `pnpm db:generate` | Generate migration files from schema changes | After modifying [src/integrations/drizzle/schema.ts]() |
| `pnpm db:migrate` | Apply pending migrations | Before starting development or after pulling changes |
| `pnpm db:push` | Push schema changes directly (no migration file) | Quick prototyping only, not for production |
| `pnpm db:pull` | Introspect existing DB and generate schema | Syncing schema from an existing database |
| `pnpm db:studio` | Open Drizzle Studio | Visual database browser at `https://local.drizzle.studio` |
</thinking>

**Sources:** [package.json:17-31](), [CLAUDE.md:9-38]()

#### Internationalization

| Command | Description | Output |
|---------|-------------|--------|
| `pnpm lingui:extract` | Extract translatable strings from code | Updates `.po` files in `locales/` directory |

**Translation Workflow:** Strings marked with `<Trans>` or `t` macro are extracted to `locales/en-US.po`, then synced to Crowdin for community translation. See page 4.1 for the full pipeline.

#### Other Utilities

| Command | Description |
|---------|-------------|
| `pnpm preview` | Preview the production build locally |
| `dotenvx run -- pnpm knip` | Detect unused exports and dead code (requires `dotenvx` for env loading) |
| `pnpm docs:dev` | Start the Mintlify documentation dev server |

> **Note:** There is no test framework configured in this project. No unit, integration, or end-to-end tests exist in the codebase.

**Sources:** [CLAUDE.md:9-38](), [package.json:17-31](), [docs/contributing/development.mdx:119-151]()

### Hot Reload Behavior

```mermaid
graph TB
    CodeChange["File Change Detected"]
    
    subgraph "Vite Dev Server"
        HMR["Vite HMR<br/>Hot Module Replacement"]
        TSCompile["TypeScript Compilation"]
        RouteGen["TanStack Router<br/>Route Tree Generation"]
    end
    
    subgraph "Browser"
        Update["Browser Updates<br/>(no full reload)"]
        FullReload["Full Page Reload"]
    end
    
    CodeChange --> HMR
    HMR --> TSCompile
    TSCompile --> RouteGen
    
    RouteGen -->|most changes| Update
    RouteGen -->|route structure changes| FullReload
```

**Hot Reload Scope:**
- React component changes: Hot reload without losing state (Vite HMR)
- Route file changes: Full page reload; TanStack Router regenerates the route tree
- Server/API/service changes: Nitro restarts, browser reloads

> **Note:** There is no test framework in the project. Verification during development is done manually through the browser or via `pnpm typecheck` and `pnpm lint`.

**Sources:** [CLAUDE.md:40-53](), [CLAUDE.md:56-59]()

---

## Database Migration Strategy

The application uses Drizzle ORM for database management with automatic migrations on startup.

### Migration Execution

```mermaid
sequenceDiagram
    participant App as Application Startup
    participant Plugin as plugins/1.migrate.ts
    participant Drizzle as Drizzle Migrate
    participant PG as PostgreSQL
    
    App->>Plugin: Server initialization
    Plugin->>Plugin: Check migrations/ directory
    Plugin->>Drizzle: migrate(db, options)
    Drizzle->>PG: SELECT version FROM __drizzle_migrations
    PG-->>Drizzle: Applied migrations list
    Drizzle->>Drizzle: Compare with migrations/ folder
    
    alt New migrations exist
        Drizzle->>PG: BEGIN TRANSACTION
        loop Each pending migration
            Drizzle->>PG: Execute SQL statements
            Drizzle->>PG: INSERT INTO __drizzle_migrations
        end
        Drizzle->>PG: COMMIT TRANSACTION
        Drizzle-->>Plugin: Success
    else No pending migrations
        Drizzle-->>Plugin: Up to date
    end
    
    Plugin-->>App: Continue startup
    
    alt Migration failure
        Drizzle->>PG: ROLLBACK
        Drizzle-->>Plugin: Error
        Plugin-->>App: Exit with error code
    end
```

**Automatic Migration:** Migrations run automatically on application startup via the Nitro plugin at `plugins/1.migrate.ts`. If migrations fail, the application exits immediately with an error.

**Manual Execution:** During development, run `pnpm db:migrate` to apply migrations independently of starting the full application server.

**Migration Files Location:** `migrations/` directory

**Schema Definition:** [src/integrations/drizzle/schema.ts]()

**Sources:** [CLAUDE.md:53](), [docs/self-hosting/docker.mdx:256-259]()

---

## Health Check Endpoints

The application exposes a comprehensive health check endpoint for monitoring service status.

### Health Check Implementation

**Endpoint:** `GET /api/health`

**Route Handler:** [src/routes/api/health.ts:17-41]()

**Checked Services:**

1. **Database:** Executes `SELECT 1` query via [src/routes/api/health.ts:43-52]()
2. **Printer:** Calls `printerService.healthcheck()` via [src/routes/api/health.ts:55-65]()
3. **Storage:** Calls `storageService.healthcheck()` via [src/routes/api/health.ts:68-77]()

**Response Format:**

```json
{
  "version": "1.0.0",
  "status": "healthy",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "uptime": "1234.56s",
  "database": { "status": "healthy" },
  "printer": { "status": "healthy", "endpoint": "ws://..." },
  "storage": { "status": "healthy", "type": "s3" }
}
```

**HTTP Status Codes:**
- `200 OK` - All services healthy
- `500 Internal Server Error` - One or more services unhealthy

**Sources:** [src/routes/api/health.ts:1-87](), [docs/self-hosting/docker.mdx:406-449]()

---

## Service Access Points

### Development Service URLs

| Service | URL | Credentials |
|---------|-----|-------------|
| Reactive Resume | [http://localhost:3000](http://localhost:3000) | Create account via UI |
| Mailpit Web UI | [http://localhost:8025](http://localhost:8025) | No authentication |
| Adminer (DB UI) | [http://localhost:8080](http://localhost:8080) | System: PostgreSQL<br/>Server: postgres<br/>Username: postgres<br/>Password: postgres<br/>Database: postgres |
| Drizzle Studio | [https://local.drizzle.studio](https://local.drizzle.studio) | Run `pnpm db:studio` first |
| Health Check | [http://localhost:3000/api/health](http://localhost:3000/api/health) | No authentication |

**Sources:** [compose.dev.yml:1-114](), [docs/contributing/development.mdx:95-97]()

---

## Troubleshooting

### Common Issues

**Port Conflicts**

If port 3000 is already in use:

```bash
PORT=3001 pnpm dev
```

The application will start on the specified port instead.

**Database Connection Failures**

Verify PostgreSQL is running and healthy:

```bash
docker compose -f compose.dev.yml ps postgres
docker compose -f compose.dev.yml logs postgres
```

Check that the `DATABASE_URL` in `.env` matches the service configuration.

**SeaweedFS Bucket Not Created**

The bucket creation is handled by a one-shot container. Check its status:

```bash
docker compose -f compose.dev.yml logs seaweedfs_create_bucket
```

If it failed, restart it:

```bash
docker compose -f compose.dev.yml up seaweedfs_create_bucket
```

**Printer Service Connection Issues**

When running locally, ensure `PRINTER_APP_URL=http://host.docker.internal:3000` is set. This allows the containerized printer to reach back to your host machine.

Test printer connectivity:

```bash
curl -f http://localhost:4000/pressure?token=1234567890
```

**Type Errors After Pulling Changes**

TanStack Router generates route type definitions. Restart the dev server to regenerate:

```bash
pnpm dev
```

Or run type checking explicitly:

```bash
pnpm typecheck
```

**Sources:** [docs/contributing/development.mdx:276-324]()

---

## Code Quality Tools

### Biome (Linting & Formatting)

**Configuration:** [biome.json]()

**Enabled Rules:**
- Automatic import organization
- Unused imports treated as errors
- 120 character line width
- Tab indentation
- Double quotes
- Sorted Tailwind classes

**IDE Integration:** VS Code extension `biomejs.biome` provides automatic formatting on save when properly configured.

**Sources:** [CLAUDE.md:152-163](), [.devcontainer/devcontainer.json:17-27]()

### TypeScript Configuration

**Strict Mode:** Enabled

**Enforced Rules:**
- `noUnusedLocals` - Error on unused local variables
- `noUnusedParameters` - Error on unused function parameters

**Path Alias:** `@/` maps to `src/` directory.

**Type Definitions:** [src/vite-env.d.ts:1-57]()

**Sources:** [CLAUDE.md:163]()

---

