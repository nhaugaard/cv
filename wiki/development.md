# Page: Development

# Development

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.devcontainer/Dockerfile](.devcontainer/Dockerfile)
- [.devcontainer/devcontainer.json](.devcontainer/devcontainer.json)
- [.devcontainer/docker-compose.yml](.devcontainer/docker-compose.yml)
- [.env.example](.env.example)
- [package.json](package.json)
- [pnpm-lock.yaml](pnpm-lock.yaml)

</details>



This page provides an overview of local development workflows for Reactive Resume, including environment setup, build processes, and contribution patterns. For detailed environment setup, see page 6.1 (Development Setup). For build system internals, see page 6.2 (Build System). For dependency management, see page 6.3 (Dependencies). For code quality tools, see page 6.4 (Code Quality).

## Development Environment Overview

Reactive Resume uses a containerized development environment with all required services orchestrated through Docker Compose. The `.devcontainer` directory provides a consistent development experience via VS Code's Remote Containers feature.

**Devcontainer Stack**

```mermaid
graph TB
    subgraph "Host"
        VSCode["VS Code + Remote Containers Extension"]
    end
    subgraph ".devcontainer (Node.js 24 container)"
        App["reactive_resume service"]
        Workspace["/workspace (bind mount to repo root)"]
    end
    subgraph "Sidecar Services (.devcontainer/docker-compose.yml)"
        PG["postgres:latest (port 5432)"]
        BR["ghcr.io/browserless/chromium (port 3000 internal)"]
        SW["chrislusf/seaweedfs (port 8333 S3 API)"]
        SWI["seaweedfs_create_bucket (one-shot init job)"]
    end
    VSCode -->|"Remote Connection"| App
    App --- Workspace
    App -->|"DATABASE_URL postgresql://postgres:5432"| PG
    App -->|"PRINTER_ENDPOINT ws://browserless:3000"| BR
    App -->|"S3_ENDPOINT http://seaweedfs:8333"| SW
    SWI -->|"mc mb seaweedfs/reactive-resume"| SW
    PG -.->|"service_healthy before app"| App
    SWI -.->|"service_completed_successfully before app"| App
```

Sources: [.devcontainer/devcontainer.json:1-32](), [.devcontainer/docker-compose.yml:1-97]()

### Service Architecture

The development environment consists of the following services:

| Service | Image | Host Port | Purpose |
|---------|-------|-----------|---------|
| `reactive_resume` | `.devcontainer/Dockerfile` (Node.js 24) | 3000 | Main application container |
| `postgres` | `postgres:latest` | 5432 | PostgreSQL database |
| `browserless` | `ghcr.io/browserless/chromium` | 4000 (forwarded) | Headless Chromium for PDF generation |
| `seaweedfs` | `chrislusf/seaweedfs` | 8333 | S3-compatible object storage |
| `seaweedfs_create_bucket` | `quay.io/minio/mc` | — | One-shot bucket initialization |

The `reactive_resume` container depends on `postgres` (health check: `pg_isready`), `browserless` (service started), and `seaweedfs_create_bucket` (completed successfully) before it is considered ready.

Sources: [.devcontainer/docker-compose.yml:2-97](), [.devcontainer/Dockerfile:1-10]()

## Quick Start

### Using Devcontainer (Recommended)

1. Open the repository in VS Code with Docker Desktop running.
2. When prompted, click "Reopen in Container" (or use Command Palette: `Remote-Containers: Reopen in Container`).
3. `pnpm install` runs automatically via `postCreateCommand` in [.devcontainer/devcontainer.json:31-31]().
4. Run `pnpm db:push` to push the Drizzle schema to the local database.
5. Run `pnpm dev` to start the Vite development server.

The application will be available at `http://localhost:3000`.

### Manual Setup (Without Devcontainer)

Start only the sidecar services, then run the app directly:

```
docker compose -f .devcontainer/docker-compose.yml up -d postgres browserless seaweedfs seaweedfs_create_bucket
corepack enable && pnpm install
pnpm db:push
pnpm dev
```

Copy [.env.example:1-78]() to `.env` and adjust service endpoints if running outside Docker.

Sources: [.devcontainer/devcontainer.json:1-32](), [.devcontainer/docker-compose.yml:1-97](), [.env.example:1-78]()
</thinking>

## Development Commands

The following npm scripts are available in [package.json:17-32]():
</old_str>
<new_str>
## Development Commands

The following npm scripts are defined in [package.json:17-32]():

## Development Commands

The following npm scripts are available in [package.json:17-32]():

| Category | Command | Underlying Invocation | Purpose |
|----------|---------|----------------------|---------|
| Dev | `pnpm dev` | `vite dev` | Start Vite development server with HMR |
| Build | `pnpm build` | `vite build` | Build production bundle |
| Build | `pnpm preview` | `vite preview` | Preview production build locally |
| Runtime | `pnpm start` | `node .output/server/index.mjs` | Start built production server |
| Types | `pnpm typecheck` | `tsgo --noEmit` | TypeScript type check (no emit) |
| Lint | `pnpm lint` | `biome check --write` | Lint and auto-format source |
| Analysis | `pnpm knip` | `knip` | Find unused files, exports, dependencies |
| i18n | `pnpm lingui:extract` | `lingui extract --clean --overwrite` | Extract translatable strings to `.po` files |
| DB | `pnpm db:generate` | `drizzle-kit generate` | Generate migration files from schema |
| DB | `pnpm db:migrate` | `drizzle-kit migrate` | Apply pending migrations |
| DB | `pnpm db:push` | `drizzle-kit push` | Push schema directly (dev only) |
| DB | `pnpm db:pull` | `drizzle-kit pull` | Pull schema from existing database |
| DB | `pnpm db:studio` | `drizzle-kit studio` | Open Drizzle Studio web UI |

**Toolchain diagram — npm scripts to invoked tools**

```mermaid
graph LR
    subgraph "package.json scripts"
        s_dev["pnpm dev"]
        s_build["pnpm build"]
        s_start["pnpm start"]
        s_tc["pnpm typecheck"]
        s_lint["pnpm lint"]
        s_knip["pnpm knip"]
        s_lingui["pnpm lingui:extract"]
        s_db["pnpm db:generate / push / migrate / studio"]
    end
    subgraph "Tools (devDependencies)"
        t_vite["vite (vite dev / vite build)"]
        t_node["node .output/server/index.mjs"]
        t_tsgo["tsgo --noEmit (@typescript/native-preview)"]
        t_biome["biome check --write (@biomejs/biome)"]
        t_knip["knip (knip)"]
        t_lingui["lingui extract (@lingui/cli)"]
        t_drizzle["drizzle-kit (drizzle-kit)"]
    end
    s_dev --> t_vite
    s_build --> t_vite
    s_start --> t_node
    s_tc --> t_tsgo
    s_lint --> t_biome
    s_knip --> t_knip
    s_lingui --> t_lingui
    s_db --> t_drizzle
```

Sources: [package.json:17-32](), [package.json:116-141]()

## Development Workflow

**Change → verify → commit flow**

```mermaid
flowchart TB
    Edit["Edit source files"]
    Edit --> HMR["Vite HMR (vite dev) updates browser"]
    HMR --> TC{"pnpm typecheck (tsgo --noEmit)"}
    TC -->|"type errors"| Fix["Fix type errors"]
    Fix --> TC
    TC -->|"OK"| Lint{"pnpm lint (biome check --write)"}
    Lint -->|"auto-fixed"| Test["Manual testing at localhost:3000"]
    Test --> Commit["git commit"]
    Commit --> PR["Pull Request -> GitHub Actions CI"]
```

The development server runs on port 3000. `APP_URL` in `.env` controls the base URL; `PRINTER_APP_URL` must point to the internal address reachable by Browserless.

Sources: [package.json:17-32](), [.env.example:3-10]()

## Environment Configuration

Development environment variables are defined in [.env.example:1-78](). Copy to `.env` and adjust values as needed.

### Key Development Variables

| Variable | Devcontainer Default | Description |
|----------|---------------------|-------------|
| `APP_URL` | `http://localhost:3000` | Public-facing URL |
| `PRINTER_APP_URL` | `http://reactive_resume:3000` | Internal URL used by the printer service |
| `PRINTER_ENDPOINT` | `ws://browserless:3000?token=1234567890` | WebSocket endpoint for Browserless/Chromium |
| `DATABASE_URL` | `postgresql://postgres:postgres@postgres:5432/postgres` | PostgreSQL connection string |
| `AUTH_SECRET` | `change-me-to-a-secure-secret-key-in-production` | Session encryption secret |
| `S3_ENDPOINT` | `http://seaweedfs:8333` | SeaweedFS S3-compatible endpoint |

### Feature Flags

| Variable | Default | Purpose |
|----------|---------|---------|
| `FLAG_DEBUG_PRINTER` | `false` | Bypass server-only check on `/printer/{resumeId}` |
| `FLAG_DISABLE_SIGNUPS` | `false` | Disable new user registrations |
| `FLAG_DISABLE_EMAIL_AUTH` | `false` | Disable email/password login |
| `FLAG_DISABLE_IMAGE_PROCESSING` | `false` | Disable `sharp` image processing (useful on low-resource hardware) |

Sources: [.env.example:1-78](), [.devcontainer/docker-compose.yml:19-30]()

## Project Structure

**Key directories and configuration files**

```mermaid
graph TB
    subgraph "Source Code"
        Root["/workspace (repo root)"]
        AppDir["app/ (application code)"]
        Routes["app/routes/ (TanStack Router routes)"]
        Components["app/components/ (React UI components)"]
        Lib["app/lib/ (utilities, hooks, stores)"]
        Server["app/server/ (server-side Nitro code)"]
        DB["app/server/db/ (Drizzle schema + migrations)"]
        RouterDir["app/server/router/ (ORPC procedures)"]
    end
    subgraph "Configuration Files"
        Package["package.json (scripts + dependencies)"]
        Vite["vite.config.ts (Vite + Nitro build)"]
        Tsconfig["tsconfig.json (TypeScript strict config)"]
        Biome["biome.json (Biome linter/formatter)"]
        Drizzle["drizzle.config.ts (DB ORM config)"]
        Lingui["lingui.config.js (i18n extraction)"]
        Knip["knip.config.ts (dead code detection)"]
    end
    subgraph "Localization"
        Locales["locales/*.po (58+ locale files)"]
    end
    Root --> AppDir
    Root --> Package
    Root --> Vite
    Root --> Tsconfig
    Root --> Biome
    Root --> Drizzle
    Root --> Lingui
    Root --> Knip
    AppDir --> Routes
    AppDir --> Components
    AppDir --> Lib
    AppDir --> Server
    Server --> DB
    Server --> RouterDir
    Lingui --> Locales
```

Sources: [package.json:1-32]()

## Technology Stack

| Category | Package | Version in `package.json` | Role |
|----------|---------|--------------------------|------|
| Runtime | Node.js | 24 (devcontainer) | JavaScript runtime |
| Package Manager | `pnpm` | 10.30.2 | Dependency management |
| Build Tool | `vite` | ^8.0.0-beta.15 | Dev server and production bundler |
| Server Bundler | `nitro` (nitro-nightly) | 3.0.1-nightly | Nitro server build |
| Framework | `@tanstack/react-start` | ^1.162.8 | React 19 SSR framework |
| Language | TypeScript | 5.9.3 (via `@typescript/native-preview`) | Type checking |
| UI | `react` | ^19.2.4 | Component model |
| Styling | `tailwindcss` | ^4.2.1 | Utility-first CSS |
| Database ORM | `drizzle-orm` | ^1.0.0-beta.15 | Type-safe PostgreSQL ORM |
| DB Migrations | `drizzle-kit` | ^1.0.0-beta.15 | Schema management CLI |
| API Layer | `@orpc/server` | ^1.13.5 | Type-safe RPC procedures |
| State | `zustand` | ^5.0.11 | Client-side state management |
| Undo/redo | `zundo` | ^2.3.0 | History middleware for Zustand |
| Validation | `zod` | ^4.3.6 | Schema validation |
| Linter/formatter | `@biomejs/biome` | ^2.4.4 | Combined lint + format |
| Dead code | `knip` | ^5.85.0 | Unused exports/dependency detection |
| i18n | `@lingui/cli` | ^5.9.2 | Translation string extraction |
| PWA | `vite-plugin-pwa` | ^1.2.0 | Service worker generation |

The project uses `pnpm` with a single `package.json` at the repo root (not a monorepo workspace). The `pnpm.overrides` field pins `vite` to `^8.0.0-beta.15` across all transitive dependencies.

Sources: [package.json:33-155](), [package.json:142-154]()

## Devcontainer Configuration

The devcontainer is defined in [.devcontainer/devcontainer.json:1-32]() and uses [.devcontainer/Dockerfile:1-10]() (based on `mcr.microsoft.com/devcontainers/typescript-node:24`) with `corepack` pre-enabled.

### VS Code Extensions (auto-installed)

| Extension ID | Purpose |
|--------------|---------|
| `biomejs.biome` | Linting and formatting (`biome.enabled: true`) |
| `bradlc.vscode-tailwindcss` | Tailwind CSS IntelliSense |
| `lokalise.i18n-ally` | i18n translation management |

Sources: [.devcontainer/devcontainer.json:15-29]()

### Port Forwarding

| Port | Label | Auto-Open in Browser |
|------|-------|---------------------|
| 3000 | Reactive Resume | Yes |
| 4000 | Browserless (Printer) | No |
| 5432 | PostgreSQL | No |
| 8333 | SeaweedFS (S3) | No |

Sources: [.devcontainer/devcontainer.json:7-13]()

### Service Startup Dependencies

```mermaid
graph LR
    PG["postgres (pg_isready healthcheck)"]
    BR["browserless/chromium (curl /pressure healthcheck)"]
    SW["seaweedfs (wget :8888 healthcheck)"]
    SWI["seaweedfs_create_bucket (mc mb one-shot)"]
    App["reactive_resume container"]
    PG -->|"service_healthy"| App
    BR -->|"service_started"| App
    SW -->|"service_healthy"| SWI
    SWI -->|"service_completed_successfully"| App
```

Sources: [.devcontainer/docker-compose.yml:9-18](), [.devcontainer/docker-compose.yml:32-93]()

## Database Development

Drizzle ORM manages the schema in TypeScript. The standard development DB workflow:

1. Edit schema files in `app/server/db/schema/`
2. `pnpm db:generate` — generate SQL migration files via `drizzle-kit generate`
3. `pnpm db:migrate` — apply migrations via `drizzle-kit migrate`

For rapid iteration, `pnpm db:push` (`drizzle-kit push`) pushes the schema directly without creating migration files. `pnpm db:studio` opens Drizzle Studio at `https://local.drizzle.studio`.

Development connection string (set in [.devcontainer/docker-compose.yml:23-23]()):
`postgresql://postgres:postgres@postgres:5432/postgres`

See page 2.3 (Data Layer) for full schema documentation.

Sources: [package.json:19-23]()

## Contributing Workflow

Standard fork-and-PR workflow:

1. Fork the repository and create a feature branch.
2. Make changes inside the devcontainer.
3. Run pre-submission checks:
   - `pnpm typecheck` — zero type errors (`tsgo --noEmit`)
   - `pnpm lint` — auto-fix with `biome check --write`
   - `pnpm knip` — no unused exports or dependencies
4. Push to your fork and open a Pull Request.
5. GitHub Actions CI runs typecheck, lint, and build automatically.

Sources: [package.json:17-32]()

## Common Development Tasks

### Adding / Updating Dependencies

```
pnpm add <package>        # production dependency
pnpm add -D <package>     # dev dependency
pnpm install              # regenerate lockfile
```

`npm-check-updates` is listed as a dev dependency for dependency update workflows. The `pnpm.onlyBuiltDependencies` field in [package.json:146-153]() controls which native packages trigger a build step (`bcrypt`, `sharp`, `esbuild`, etc.).

### Adding a Translation String

1. Use Lingui macros in code: `<Trans>Hello World</Trans>` or `t\`Hello World\``
2. Run `pnpm lingui:extract` to update `locales/en-US/messages.po`
3. Crowdin synchronizes translations for other locales automatically.

See page 4.1 (Translation Workflow) for full details.

### Debugging PDF Generation

Set `FLAG_DEBUG_PRINTER=true` in `.env` to bypass the server-only restriction on the `/printer/{resumeId}` route, enabling direct browser access for layout debugging.

Sources: [.env.example:59-61](), [package.json:28-28](), [package.json:137-137]()

## Build Artifacts

| Output | Location | Description |
|--------|----------|-------------|
| Server bundle | `.output/server/index.mjs` | Nitro server entry point |
| Client assets | `.output/public/` | Static JS, CSS, fonts |
| Type declarations | N/A | Not emitted (`tsgo --noEmit`) |

`pnpm start` executes `node .output/server/index.mjs`. See page 6.2 (Build System) for the full Vite + Nitro build pipeline.

Sources: [package.json:30-30]()

## Resource Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| RAM | 4 GB | 8 GB |
| CPU | 2 cores | 4+ cores |
| Disk | 5 GB | 10 GB |
| Docker memory limit | 2 GB | 4 GB |

The Browserless/Chromium container is the primary memory consumer. Ensure ports 3000, 4000, 5432, and 8333 are free on the host before starting the devcontainer.

Sources: [.devcontainer/Dockerfile:1-10](), [.devcontainer/devcontainer.json:7-13]()

## Troubleshooting

### Container Build Failures

If the devcontainer fails to build:

1. Ensure Docker Desktop is running
2. Clear Docker cache: `docker system prune -a`
3. Rebuild container: Command Palette → "Remote-Containers: Rebuild Container"

### Database Connection Issues

If the app cannot connect to PostgreSQL:

1. Check container status: `docker compose ps`
2. Verify health check: `docker compose exec postgres pg_isready -U postgres`
3. Check environment variables in [.devcontainer/docker-compose.yml:23-23]()

### Port Already in Use

If port 3000 is already bound:

1. Find process: `lsof -i :3000` (macOS/Linux) or `netstat -ano | findstr :3000` (Windows)
2. Kill process or change `APP_URL` in `.env`

### Browserless Connection Failures

If PDF generation fails:

1. Verify Browserless is running: `curl http://localhost:4000/pressure?token=1234567890`
2. Check logs: `docker compose logs browserless`
3. Verify `PRINTER_ENDPOINT` in `.env` matches the token

**Sources:** [.env.example:10-13](), [.devcontainer/docker-compose.yml:48-61]()

---

