# Page: Architecture Overview

# Architecture Overview

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.env.example](.env.example)
- [CLAUDE.md](CLAUDE.md)
- [README.md](README.md)
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



This page describes the high-level architecture of Reactive Resume, showing how the three-tier system is organized and how major components interact. For detailed information about specific subsystems, see:
- Frontend implementation: [Frontend Architecture](#2.1)
- Backend services: [Backend Services](#2.2)
- Data persistence: [Data Layer](#2.3)
- API design patterns: [API Design](#2.4)

The architecture follows a clean separation between client, service, and data layers, with type-safe communication via ORPC and deployment via Docker containers.

## Three-Tier Architecture

Reactive Resume implements a classic three-tier architecture with modern technologies. The system is composed of distinct layers that communicate through well-defined interfaces.

```mermaid
graph TB
    subgraph "Client Layer (Port 3000)"
        React["React 19<br/>@tanstack/react-router<br/>Vite 8"]
        Zustand["Zustand Store<br/>+ Immer + Zundo"]
        TanStackQuery["@tanstack/react-query<br/>Server State Cache"]
    end
    
    subgraph "API & Service Layer"
        ORPCRouter["ORPC Router<br/>src/integrations/orpc/router/"]
        ORPCServices["ORPC Services<br/>src/integrations/orpc/services/"]
        BetterAuth["Better Auth<br/>src/integrations/auth/"]
        
        ResumeService["resume.ts<br/>CRUD + Patch Operations"]
        PrinterService["printer.ts<br/>PDF Generation"]
        StorageService["storage.ts<br/>S3 or Local FS"]
        AIService["ai.ts<br/>Multi-provider AI"]
    end
    
    subgraph "Data Layer"
        DrizzleORM["Drizzle ORM<br/>src/integrations/drizzle/"]
        PostgreSQL[(PostgreSQL<br/>Database)]
        S3Storage[(S3-Compatible<br/>SeaweedFS or Local FS)]
        Browserless[Headless Chromium<br/>Browserless Service]
    end
    
    React --> Zustand
    React --> TanStackQuery
    TanStackQuery --> ORPCRouter
    Zustand --> ORPCRouter
    
    ORPCRouter --> BetterAuth
    ORPCRouter --> ORPCServices
    
    ORPCServices --> ResumeService
    ORPCServices --> PrinterService
    ORPCServices --> StorageService
    ORPCServices --> AIService
    
    BetterAuth --> DrizzleORM
    ResumeService --> DrizzleORM
    StorageService --> S3Storage
    PrinterService --> Browserless
    PrinterService --> StorageService
    
    DrizzleORM --> PostgreSQL
```

**Sources:** [package.json:33-115](), [src/routes/__root.tsx:1-140](), [compose.yml:1-117]()

## Client Layer

The client layer is built with React 19 and runs in the browser. It uses TanStack Start for server-side rendering and file-based routing, compiled by Vite 8.

### Key Technologies

| Technology | Package | Purpose |
|------------|---------|---------|
| UI Framework | `react@19.2.4` | Component rendering |
| Router | `@tanstack/react-router@1.159.5` | File-based routing + SSR |
| Build Tool | `vite@8.0.0-beta.13` | Development server + bundler |
| State Management | `zustand@5.0.11` | Client-side state |
| Undo/Redo | `zundo@2.3.0` | Time-travel state history |
| Immutability | `immer@11.1.3` | Immutable state updates |
| Server State | `@tanstack/react-query@5.90.20` | API caching + synchronization |
| Styling | `tailwindcss@4.1.18` | Utility-first CSS |
| UI Components | `radix-ui@1.4.3` | Accessible component primitives |

### State Management Architecture

```mermaid
graph LR
    UserAction[User Action] --> Component[React Component]
    Component --> Zustand[Zustand Store<br/>src/components/resume/store/]
    Zustand --> Immer[Immer Producer<br/>Immutable Updates]
    Immer --> Zundo[Zundo Middleware<br/>History Tracking]
    Zundo --> Debounce[Debounce 500ms]
    Debounce --> ORPCClient[@orpc/client]
    ORPCClient --> Server[ORPC Server]
    
    Server --> TanStackQuery[@tanstack/react-query<br/>Cache Layer]
    TanStackQuery --> Component
```

The resume editor implements debounced synchronization: local edits update Zustand immediately for instant UI feedback, then sync to the server after 500ms of inactivity. This provides both responsive editing and automatic persistence.

**Sources:** [package.json:84,114-115,257-259](), [src/routes/__root.tsx:25-32]()

## API & Service Layer

The API layer uses ORPC for type-safe RPC communication. All API endpoints are defined as procedures with Zod schema validation for inputs and outputs.

### ORPC Router Structure

```mermaid
graph TB
    Client[@orpc/client<br/>src/integrations/orpc/client.ts] --> HTTPLayer[HTTP/RPC Layer]
    
    HTTPLayer --> ORPCServer[@orpc/server<br/>src/integrations/orpc/router/index.ts]
    
    ORPCServer --> Procedures[Procedures with Context]
    
    Procedures --> PublicProc[publicProcedure<br/>No Auth Required]
    Procedures --> ProtectedProc[protectedProcedure<br/>Session or API Key]
    Procedures --> ServerOnlyProc[serverOnlyProcedure<br/>Internal Only]
    
    PublicProc --> AuthRouter[auth.ts<br/>Login/Signup]
    ProtectedProc --> ResumeRouter[resume.ts<br/>CRUD Operations]
    ProtectedProc --> StorageRouter[storage.ts<br/>File Upload]
    ProtectedProc --> AIRouter[ai.ts<br/>AI Features]
    ServerOnlyProc --> PrinterRouter[printer.ts<br/>PDF Generation]
    
    ResumeRouter --> ResumeService[resume.ts<br/>services/]
    StorageRouter --> StorageService[storage.ts<br/>services/]
    AIRouter --> AIService[ai.ts<br/>services/]
    PrinterRouter --> PrinterService[printer.ts<br/>services/]
```

### Context and Authentication

ORPC procedures receive a context object that includes authentication state:

```typescript
// src/integrations/orpc/context.ts
type Context = {
  user: User | null;          // Authenticated user
  session: Session | null;    // User session
  apiKey: ApiKey | null;      // API key if using API auth
}
```

Three procedure types enforce different authentication requirements:
- `publicProcedure` - No authentication required
- `protectedProcedure` - Requires authenticated user (session or API key)
- `serverOnlyProcedure` - Only callable from server-side code

**Sources:** [src/integrations/orpc/context.ts](), [package.json:48-53,59-62]()

### Service Layer Components

Each service module implements specific business logic:

| Service | File | Responsibility |
|---------|------|----------------|
| Resume Service | `src/integrations/orpc/services/resume.ts` | Resume CRUD, JSON Patch updates, validation |
| Printer Service | `src/integrations/orpc/services/printer.ts` | PDF generation, screenshot capture, headless browser management |
| Storage Service | `src/integrations/orpc/services/storage.ts` | File upload/download, S3 or local filesystem abstraction |
| AI Service | `src/integrations/orpc/services/ai.ts` | Multi-provider AI (OpenAI, Anthropic, Gemini, Ollama) |
| Auth Service | `src/integrations/auth/` | User authentication via Better Auth |
| Email Service | `src/integrations/email/` | SMTP email via Nodemailer (or console fallback) |

**Sources:** [src/integrations/orpc/services/storage.ts:1-371](), [src/integrations/orpc/router/storage.ts:1-92]()

## Data Layer

The data layer handles persistence through PostgreSQL (via Drizzle ORM) and file storage (via S3-compatible services or local filesystem).

### Database Architecture

```mermaid
graph TB
    DrizzleORM[Drizzle ORM<br/>src/integrations/drizzle/] --> Schema[schema.ts<br/>Table Definitions]
    DrizzleORM --> Client[client.ts<br/>pg Connection Pool]
    
    Client --> PostgreSQL[(PostgreSQL<br/>Port 5432)]
    
    Schema --> UserTables[User Auth Tables]
    Schema --> ResumeTables[Resume Tables]
    
    UserTables --> UserTable[user<br/>session<br/>account<br/>verification<br/>twoFactor<br/>passkey<br/>apikey]
    
    ResumeTables --> ResumeTable[resume<br/>id, userId, title, slug<br/>data JSONB<br/>visibility, locked]
    ResumeTables --> StatsTable[resumeStatistics<br/>views, downloads]
```

The `resume` table stores all resume data as JSONB in the `data` column, which conforms to the `ResumeData` schema defined in `src/schema/resume/data.ts`. This allows flexible schema evolution while maintaining type safety via Zod validation.

### Storage Service Abstraction

The storage service provides a unified interface for file operations, supporting both S3-compatible storage and local filesystem:

```mermaid
graph TB
    StorageRouter[storage.ts Router] --> UploadFile[uploadFile Procedure]
    StorageRouter --> DeleteFile[deleteFile Procedure]
    
    UploadFile --> ProcessImage[processImageForUpload<br/>Resize + WebP Conversion]
    ProcessImage --> StorageService[getStorageService]
    
    StorageService --> CheckEnv{S3 Environment<br/>Variables Set?}
    
    CheckEnv -->|Yes| S3Service[S3StorageService<br/>@aws-sdk/client-s3]
    CheckEnv -->|No| LocalService[LocalStorageService<br/>Node.js fs/promises]
    
    S3Service --> S3Backend[(S3-Compatible<br/>SeaweedFS or AWS S3)]
    LocalService --> LocalFS[(Local Filesystem<br/>/app/data)]
```

**Configuration determines storage backend:**
- If `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, and `S3_BUCKET` are set: uses S3StorageService
- Otherwise: uses LocalStorageService with files stored in `/app/data`

**Sources:** [src/integrations/orpc/services/storage.ts:113-208,210-306,308-323](), [src/utils/env.ts:56-64]()

## Request Flow

This diagram shows how a typical authenticated request flows through the system:

```mermaid
sequenceDiagram
    participant Browser
    participant React as React Component
    participant Zustand as Zustand Store
    participant Query as TanStack Query
    participant ORPC as ORPC Client
    participant Router as ORPC Router
    participant Context as Context Builder
    participant Service as Service Layer
    participant DB as Drizzle ORM
    participant PG as PostgreSQL
    
    Browser->>React: User edits resume
    React->>Zustand: Update local state
    Zustand->>Zustand: Apply via Immer
    Zustand->>Zustand: Record in Zundo history
    
    Note over Zustand: Debounce 500ms
    
    Zustand->>ORPC: orpc.resume.patch(data)
    ORPC->>Router: HTTP POST /api/orpc
    
    Router->>Context: Extract session cookie
    Context->>Context: Validate auth (session or API key)
    Context->>Router: Return context with user
    
    Router->>Router: Validate input via Zod
    Router->>Service: Call service method
    Service->>DB: Execute query via Drizzle
    DB->>PG: SQL transaction
    PG-->>DB: Result
    DB-->>Service: Mapped result
    Service-->>Router: Response
    Router->>Router: Validate output via Zod
    Router-->>ORPC: JSON response
    
    ORPC->>Query: Update cache
    Query->>React: Trigger re-render
    React->>Browser: Update UI
```

**Sources:** [src/integrations/orpc/context.ts](), [src/integrations/orpc/router/index.ts]()

## Deployment Architecture

The application runs as a set of Docker containers orchestrated by Docker Compose.

```mermaid
graph TB
    subgraph "Docker Host"
        subgraph "Application Container"
            App[reactive_resume:latest<br/>Node.js + Vite SSR<br/>Port 3000]
        end
        
        subgraph "Database Container"
            PG[(postgres:latest<br/>Port 5432)]
        end
        
        subgraph "Printer Container"
            Browserless[browserless/chromium:latest<br/>or chromedp/headless-shell<br/>Port 4000 or 9222]
        end
        
        subgraph "Storage Container (Optional)"
            SeaweedFS[(seaweedfs:latest<br/>S3-Compatible<br/>Port 8333)]
        end
        
        subgraph "Development Tools (compose.dev.yml only)"
            Adminer[adminer:latest<br/>Port 8080]
            Mailpit[mailpit:latest<br/>Ports 1025, 8025]
        end
    end
    
    App --> PG
    App --> Browserless
    App --> SeaweedFS
    App -.->|/app/data volume| LocalFS[(Local Filesystem<br/>if S3 not configured)]
    
    Adminer -.-> PG
    Mailpit -.->|SMTP| App
```

### Container Communication

Containers communicate over a Docker network using service names as hostnames:

| Environment Variable | Value | Purpose |
|---------------------|-------|---------|
| `DATABASE_URL` | `postgresql://postgres:postgres@postgres:5432/postgres` | App → PostgreSQL |
| `PRINTER_ENDPOINT` | `ws://browserless:3000?token=1234567890` | App → Browserless |
| `S3_ENDPOINT` | `http://seaweedfs:8333` | App → SeaweedFS |
| `PRINTER_APP_URL` | `http://reactive_resume:3000` | Printer → App (in Compose) |
| `PRINTER_APP_URL` | `http://host.docker.internal:3000` | Printer → App (dev mode) |

The `PRINTER_APP_URL` variable handles the case where the app runs outside Docker (development) while the printer runs inside Docker. The printer needs to access the app to render resumes for PDF generation.

**Sources:** [compose.yml:1-117](), [compose.dev.yml:1-114](), [.env.example:1-78]()

## Health Check System

The application exposes a health check endpoint at `/api/health` that verifies all critical dependencies:

```mermaid
graph LR
    Request[GET /api/health] --> Handler[handler function<br/>src/routes/api/health.ts]
    
    Handler --> CheckDB[checkDatabase<br/>SELECT 1 via Drizzle]
    Handler --> CheckPrinter[checkPrinter<br/>printerService.healthcheck]
    Handler --> CheckStorage[checkStorage<br/>storageService.healthcheck]
    
    CheckDB --> DBResult{healthy?}
    CheckPrinter --> PrinterResult{healthy?}
    CheckStorage --> StorageResult{healthy?}
    
    DBResult -->|All healthy| Return200[HTTP 200<br/>status: healthy]
    PrinterResult -->|All healthy| Return200
    StorageResult -->|All healthy| Return200
    
    DBResult -->|Any unhealthy| Return500[HTTP 500<br/>status: unhealthy]
    PrinterResult -->|Any unhealthy| Return500
    StorageResult -->|Any unhealthy| Return500
```

The health check is used by:
- Docker Compose health checks (every 30 seconds)
- Reverse proxies (Traefik, nginx) for load balancing decisions
- Monitoring systems for uptime tracking

**Sources:** [src/routes/api/health.ts:1-87](), [compose.yml:106-111]()

## Environment Configuration

The application is configured entirely through environment variables, validated at startup via `@t3-oss/env-core` with Zod schemas.

### Required Variables

| Variable | Schema | Purpose |
|----------|--------|---------|
| `APP_URL` | `z.url()` | Public URL of the application |
| `DATABASE_URL` | `z.url({ protocol: /postgres/ })` | PostgreSQL connection string |
| `PRINTER_ENDPOINT` | `z.url({ protocol: /^(wss?\|https?)$/ })` | Printer service URL (WebSocket or HTTP) |
| `AUTH_SECRET` | `z.string().min(1)` | Secret for session encryption |

### Optional Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `PRINTER_APP_URL` | `APP_URL` | Override URL for printer to access app |
| `S3_FORCE_PATH_STYLE` | `false` | Use path-style URLs for S3 (MinIO, SeaweedFS) |
| `FLAG_DISABLE_SIGNUPS` | `false` | Disable new user registration |
| `FLAG_DISABLE_EMAIL_AUTH` | `false` | Disable email/password login (SSO only) |
| `FLAG_DISABLE_IMAGE_PROCESSING` | `false` | Disable image resizing (for low-resource devices) |

All environment variables are validated and typed via the `env` object exported from `src/utils/env.ts`.

**Sources:** [src/utils/env.ts:1-73](), [src/vite-env.d.ts:9-57](), [.env.example:1-78]()

---

