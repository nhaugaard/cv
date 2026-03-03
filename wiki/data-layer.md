# Page: Data Layer

# Data Layer

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.env.example](.env.example)
- [.gitignore](.gitignore)
- [.vscode/settings.json](.vscode/settings.json)
- [docs/changelog/index.mdx](docs/changelog/index.mdx)
- [docs/community/spotlight.mdx](docs/community/spotlight.mdx)
- [docs/docs.json](docs/docs.json)
- [docs/guides/setting-up-passkeys.mdx](docs/guides/setting-up-passkeys.mdx)
- [docs/guides/using-the-patch-api.mdx](docs/guides/using-the-patch-api.mdx)
- [docs/self-hosting/sso.mdx](docs/self-hosting/sso.mdx)
- [docs/spec.json](docs/spec.json)
- [knip.json](knip.json)
- [package.json](package.json)
- [pnpm-lock.yaml](pnpm-lock.yaml)
- [scripts/fonts/generate.ts](scripts/fonts/generate.ts)
- [scripts/fonts/types.ts](scripts/fonts/types.ts)
- [src/components/resume/preview.module.css](src/components/resume/preview.module.css)
- [src/components/resume/store/resume.ts](src/components/resume/store/resume.ts)
- [src/components/typography/combobox.tsx](src/components/typography/combobox.tsx)
- [src/components/typography/webfontlist.json](src/components/typography/webfontlist.json)
- [src/integrations/auth/client.ts](src/integrations/auth/client.ts)
- [src/integrations/auth/config.ts](src/integrations/auth/config.ts)
- [src/integrations/orpc/dto/resume.ts](src/integrations/orpc/dto/resume.ts)
- [src/integrations/orpc/router/printer.ts](src/integrations/orpc/router/printer.ts)
- [src/integrations/orpc/router/resume.ts](src/integrations/orpc/router/resume.ts)
- [src/integrations/orpc/services/ai.ts](src/integrations/orpc/services/ai.ts)
- [src/integrations/orpc/services/printer.ts](src/integrations/orpc/services/printer.ts)
- [src/integrations/orpc/services/resume.ts](src/integrations/orpc/services/resume.ts)
- [src/routes/auth/-components/social-auth.tsx](src/routes/auth/-components/social-auth.tsx)
- [src/routes/auth/login.tsx](src/routes/auth/login.tsx)
- [src/routes/auth/register.tsx](src/routes/auth/register.tsx)
- [src/routes/builder/$resumeId/-sidebar/right/sections/typography.tsx](src/routes/builder/$resumeId/-sidebar/right/sections/typography.tsx)
- [src/routes/dashboard/settings/authentication/-components/hooks.tsx](src/routes/dashboard/settings/authentication/-components/hooks.tsx)
- [src/utils/resume/move-item.ts](src/utils/resume/move-item.ts)
- [src/utils/resume/patch.ts](src/utils/resume/patch.ts)
- [src/utils/string.ts](src/utils/string.ts)
- [vite.config.ts](vite.config.ts)

</details>



This document describes the data persistence layer in Reactive Resume, including the PostgreSQL database with Drizzle ORM integration and the dual-mode storage system that supports both S3-compatible object storage and local filesystem storage.

For API-level data access patterns and validation, see [API Design](#2.4). For backend service implementations that use the data layer, see [Backend Services](#2.2).

---

## Architecture Overview

The data layer consists of two primary subsystems: a PostgreSQL relational database for structured application data, and a flexible storage system for file uploads (images, PDFs, screenshots).

### Data Layer Components

```mermaid
graph TB
    subgraph "Application Layer"
        Services["ORPC Services<br/>(resume, auth, storage)"]
    end
    
    subgraph "Data Access Layer"
        DrizzleClient["Drizzle Client<br/>db instance"]
        StorageService["Storage Service<br/>getStorageService()"]
    end
    
    subgraph "Storage Implementations"
        LocalStorage["LocalStorageService<br/>/data directory"]
        S3Storage["S3StorageService<br/>S3Client"]
    end
    
    subgraph "Data Persistence"
        PostgreSQL[("PostgreSQL<br/>port 5432")]
        LocalFS[("Local Filesystem<br/>/app/data")]
        S3[("S3-Compatible Storage<br/>SeaweedFS/MinIO/AWS S3")]
    end
    
    Services --> DrizzleClient
    Services --> StorageService
    
    DrizzleClient --> PostgreSQL
    
    StorageService -.->|S3 configured| S3Storage
    StorageService -.->|S3 not configured| LocalStorage
    
    S3Storage --> S3
    LocalStorage --> LocalFS
    
    style DrizzleClient fill:#e1f5ff
    style StorageService fill:#fff4e1
```

**Sources:** [src/integrations/orpc/services/storage.ts:308-323](), [src/utils/env.ts:20-64](), [CLAUDE.md:74-82]()

---

## PostgreSQL Database

The application uses PostgreSQL as its primary relational database, accessed via Drizzle ORM.

### Database Configuration

The database connection is configured through a single environment variable:

| Variable | Format | Example |
|----------|--------|---------|
| `DATABASE_URL` | `postgresql://user:pass@host:port/database` | `postgresql://postgres:postgres@localhost:5432/postgres` |

**Sources:** [src/utils/env.ts:20-21](), [.env.example:15-16]()

### Connection Management

The Drizzle client is initialized in `src/integrations/drizzle/client.ts` and provides a singleton `db` instance used throughout the application. The PostgreSQL driver is the Node.js `pg` package.

**Sources:** [package.json:90](), [CLAUDE.md:115-120]()

### Database Schema

The database schema is defined using Drizzle ORM's table definitions in `src/integrations/drizzle/schema.ts`:

| Table | Purpose |
|-------|---------|
| `user` | User accounts (email, name, credentials) |
| `session` | Active user sessions (Better Auth) |
| `account` | OAuth provider accounts linked to users |
| `verification` | Email verification tokens |
| `twoFactor` | Two-factor authentication settings (TOTP) |
| `passkey` | WebAuthn passkey credentials |
| `apikey` | API key authentication tokens |
| `resume` | Resume data stored as JSONB |
| `resumeStatistics` | View/download tracking metrics |

The `resume` table stores the complete resume data structure as a JSONB column, defined by the `ResumeData` schema in `src/schema/resume/data.ts`. This allows flexible document storage while maintaining relational integrity for user accounts and authentication.

**Sources:** [CLAUDE.md:115-120]()

### Database Schema Relationships

```mermaid
erDiagram
    user ||--o{ resume : "owns"
    user ||--o{ session : "has"
    user ||--o{ account : "linked_to"
    user ||--o{ apikey : "has"
    user ||--o{ passkey : "has"
    user ||--o{ verification : "has"
    user ||--o{ twoFactor : "has"
    resume ||--o| resumeStatistics : "tracks"
    
    user {
        string id PK
        string email
        string name
        datetime createdAt
    }
    
    resume {
        string id PK
        string userId FK
        jsonb data
        datetime createdAt
        datetime updatedAt
    }
    
    session {
        string id PK
        string userId FK
        datetime expiresAt
    }
    
    apikey {
        string id PK
        string userId FK
        string hashedKey
        datetime expiresAt
    }
```

**Sources:** [CLAUDE.md:115-120]()

---

## Drizzle ORM

Drizzle ORM provides type-safe database access with a minimal abstraction layer over PostgreSQL.

### ORM Client Initialization

The Drizzle client is instantiated with the `pg` driver:

```typescript
// Conceptual structure from src/integrations/drizzle/client.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';

const pool = new Pool({ connectionString: env.DATABASE_URL });
export const db = drizzle(pool);
```

**Sources:** [package.json:77](), [package.json:90](), [src/utils/env.ts:20-21]()

### Schema Definition

Schema tables are defined using Drizzle's table builder functions in `src/integrations/drizzle/schema.ts`. The schema integrates with Zod for validation through `drizzle-zod`.

**Sources:** [package.json:78](), [CLAUDE.md:115-120]()

### Database Migrations

The migration system uses Drizzle Kit for schema management:

| Command | Purpose |
|---------|---------|
| `pnpm db:generate` | Generate migration files from schema changes |
| `pnpm db:migrate` | Apply pending migrations to database |
| `pnpm db:push` | Push schema changes directly (development) |
| `pnpm db:studio` | Open Drizzle Studio GUI |

Migrations are stored in the `migrations/` directory and run automatically on application startup via the Nitro plugin at `plugins/1.migrate.ts`.

**Sources:** [package.json:19-23](), [CLAUDE.md:53](), [docs/self-hosting/docker.mdx:256-259]()

### Migration Flow

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Schema as schema.ts
    participant CLI as drizzle-kit
    participant MigDir as migrations/
    participant Plugin as plugins/1.migrate.ts
    participant DB as PostgreSQL
    
    Dev->>Schema: Edit schema definition
    Dev->>CLI: pnpm db:generate
    CLI->>Schema: Read current schema
    CLI->>DB: Read existing schema
    CLI->>MigDir: Generate SQL migration file
    
    Note over Plugin,DB: On app startup
    Plugin->>MigDir: Read pending migrations
    Plugin->>DB: Apply migrations in order
    DB-->>Plugin: Migration complete
    Plugin->>Plugin: Continue startup
```

**Sources:** [package.json:19-23](), [CLAUDE.md:53](), [docs/self-hosting/docker.mdx:256-259]()

---

## Storage Layer

The storage layer provides a unified interface for file uploads with two backend implementations: local filesystem and S3-compatible object storage.

### Storage Service Abstraction

The `StorageService` interface defines the contract for all storage implementations:

```typescript
interface StorageService {
  list(prefix: string): Promise<string[]>;
  write(input: StorageWriteInput): Promise<void>;
  read(key: string): Promise<StorageReadResult | null>;
  delete(key: string): Promise<boolean>;
  healthcheck(): Promise<StorageHealthResult>;
}
```

**Sources:** [src/integrations/orpc/services/storage.ts:26-32]()

### Storage Selection Logic

```mermaid
flowchart TD
    Start["getStorageService()"] --> CheckCache{"cachedService<br/>exists?"}
    CheckCache -->|Yes| ReturnCache["Return cached service"]
    CheckCache -->|No| CheckS3{"S3_ACCESS_KEY_ID,<br/>S3_SECRET_ACCESS_KEY,<br/>S3_BUCKET all set?"}
    CheckS3 -->|Yes| CreateS3["new S3StorageService()"]
    CheckS3 -->|No| CreateLocal["new LocalStorageService()"]
    CreateS3 --> Cache["cachedService = service"]
    CreateLocal --> Cache
    Cache --> Return["Return service"]
```

**Sources:** [src/integrations/orpc/services/storage.ts:308-323]()

### Local Filesystem Storage

The `LocalStorageService` class stores files in the `/data` directory relative to the application root (maps to `/app/data` in Docker containers).

| Operation | Implementation |
|-----------|----------------|
| Root directory | `join(process.cwd(), "data")` |
| Path resolution | Normalizes keys, prevents directory traversal |
| Write | Creates parent directories, writes file |
| Read | Returns file buffer with metadata |
| Delete | Removes file or directory recursively |
| List | Recursively lists files under prefix |

The local storage implementation requires the `/app/data` directory to be mounted to persistent storage in production deployments.

**Sources:** [src/integrations/orpc/services/storage.ts:113-208](), [docs/self-hosting/docker.mdx:350]()

### S3-Compatible Storage

The `S3StorageService` class uses the AWS SDK S3 client to interact with any S3-compatible object storage service:

| Service | Configuration |
|---------|---------------|
| AWS S3 | Standard endpoint, virtual-hosted style |
| SeaweedFS | Custom endpoint, path-style URLs |
| MinIO | Custom endpoint, path-style URLs |
| Cloudflare R2 | Custom endpoint, virtual-hosted style |

The `S3_FORCE_PATH_STYLE` environment variable controls URL addressing:
- `"true"`: Path-style URLs (`https://endpoint/bucket/key`) - required for SeaweedFS, MinIO
- `"false"`: Virtual-hosted style URLs (`https://bucket.endpoint/key`) - used by AWS S3, R2

**Sources:** [src/integrations/orpc/services/storage.ts:210-306](), [src/utils/env.ts:56-64](), [.env.example:46-56]()

### S3 Client Configuration

```mermaid
graph LR
    EnvVars["Environment Variables"] --> S3Client["S3Client"]
    
    EnvVars --> AccessKey["S3_ACCESS_KEY_ID"]
    EnvVars --> SecretKey["S3_SECRET_ACCESS_KEY"]
    EnvVars --> Region["S3_REGION<br/>(default: us-east-1)"]
    EnvVars --> Endpoint["S3_ENDPOINT"]
    EnvVars --> Bucket["S3_BUCKET"]
    EnvVars --> ForcePathStyle["S3_FORCE_PATH_STYLE<br/>(default: false)"]
    
    AccessKey --> Credentials
    SecretKey --> Credentials
    Credentials["credentials"] --> S3Client
    Region --> S3Client
    Endpoint --> S3Client
    ForcePathStyle --> S3Client
    
    S3Client --> Operations["PutObjectCommand<br/>GetObjectCommand<br/>DeleteObjectCommand<br/>ListObjectsV2Command"]
```

**Sources:** [src/integrations/orpc/services/storage.ts:210-229](), [src/utils/env.ts:56-64]()

### Upload Types and Key Generation

The storage service supports three upload types with different key generation strategies:

| Type | Key Pattern | Function |
|------|-------------|----------|
| `picture` | `uploads/{userId}/pictures/{timestamp}.webp` | `buildPictureKey(userId)` |
| `screenshot` | `uploads/{userId}/screenshots/{resumeId}/{timestamp}.webp` | `buildScreenshotKey(userId, resumeId)` |
| `pdf` | `uploads/{userId}/pdfs/{resumeId}/{timestamp}.pdf` | `buildPdfKey(userId, resumeId)` |

Keys are constructed to organize files by user and purpose, with timestamps ensuring uniqueness.

**Sources:** [src/integrations/orpc/services/storage.ts:55-69](), [src/integrations/orpc/services/storage.ts:341-370]()

### Image Processing Pipeline

Images uploaded to the storage service are automatically processed unless `FLAG_DISABLE_IMAGE_PROCESSING` is enabled:

```mermaid
sequenceDiagram
    participant Client
    participant API as uploadFile handler
    participant Proc as processImageForUpload()
    participant Sharp
    participant Storage as StorageService
    
    Client->>API: POST file (image/jpeg)
    API->>API: Check isImageFile(mimeType)
    
    alt FLAG_DISABLE_IMAGE_PROCESSING = true
        API->>Storage: Upload original file
    else Image processing enabled
        API->>Proc: processImageForUpload(file)
        Proc->>Sharp: sharp(buffer)
        Sharp->>Sharp: resize(800, 800, fit: inside)
        Sharp->>Sharp: webp(preset: picture)
        Sharp-->>Proc: Processed buffer
        Proc-->>API: { data, contentType: "image/webp" }
        API->>Storage: Upload processed WebP
    end
    
    Storage-->>API: { url, key }
    API-->>Client: Upload result
```

The image processing:
1. Resizes to max 800x800px (maintains aspect ratio)
2. Converts to WebP format
3. Uses "picture" preset for optimal compression

**Sources:** [src/integrations/orpc/services/storage.ts:89-111](), [src/utils/env.ts:70]()

### Content Type Inference

Content types are determined by file extension:

```typescript
const CONTENT_TYPE_MAP: Record<string, string> = {
  ".webp": "image/webp",
  ".jpg": "image/jpeg",
  ".jpeg": "image/jpeg",
  ".png": "image/png",
  ".gif": "image/gif",
  ".svg": "image/svg+xml",
  ".pdf": "application/pdf",
};
```

**Sources:** [src/integrations/orpc/services/storage.ts:41-49]()

---

## Data Access Patterns

### Resume Data Persistence

Resume documents are stored as JSONB in the `resume` table. The application uses Drizzle ORM to query and update resume records, with the JSON structure validated by Zod schemas defined in `src/schema/resume/data.ts`.

**Sources:** [CLAUDE.md:100-106](), [CLAUDE.md:115-120]()

### Atomic Updates with JSON Patch

Resume updates use RFC 6902 JSON Patch for atomic, partial updates. This is handled by the `fast-json-patch` library and integrated through the resume service's patch API.

For details on the patching system, see [JSON Patch API](#3.1.4).

**Sources:** [package.json:82]()

### File Upload Flow

```mermaid
sequenceDiagram
    participant Client
    participant StorageRouter as storage.uploadFile
    participant ImageProc as processImageForUpload()
    participant UploadFunc as uploadFile()
    participant StorageService
    
    Client->>StorageRouter: POST /storage/upload (File)
    StorageRouter->>StorageRouter: Validate file size (max 10MB)
    StorageRouter->>StorageRouter: Check isImageFile(file.type)
    
    alt Image file
        StorageRouter->>ImageProc: Process image
        ImageProc-->>StorageRouter: { data, contentType }
    else Non-image file
        StorageRouter->>StorageRouter: Use original buffer
    end
    
    StorageRouter->>UploadFunc: uploadFile({ userId, data, contentType, type })
    UploadFunc->>UploadFunc: Generate key (buildPictureKey)
    UploadFunc->>StorageService: write({ key, data, contentType })
    StorageService->>StorageService: Store file (local or S3)
    StorageService-->>UploadFunc: Success
    UploadFunc->>UploadFunc: buildPublicUrl(key)
    UploadFunc-->>StorageRouter: { url, key }
    StorageRouter-->>Client: { url, path, contentType }
```

**Sources:** [src/integrations/orpc/router/storage.ts:32-61](), [src/integrations/orpc/services/storage.ts:341-370]()

---

## Health Checks

The data layer includes health checks for both the database and storage systems, exposed via the `/api/health` endpoint.

### Database Health Check

The database health check executes a simple query to verify connectivity:

```typescript
async function checkDatabase() {
  try {
    await db.execute(sql`SELECT 1`);
    return { status: "healthy" };
  } catch (error) {
    return {
      status: "unhealthy",
      error: error.message
    };
  }
}
```

**Sources:** [src/routes/api/health.ts:43-53]()

### Storage Health Check

Each storage implementation provides a `healthcheck()` method:

| Implementation | Test Method |
|----------------|-------------|
| `LocalStorageService` | Verifies read/write permissions on `/data` directory |
| `S3StorageService` | Writes and deletes a test object to S3 bucket |

The health check response includes the storage type (`"local"` or `"s3"`) and detailed error messages if unhealthy.

**Sources:** [src/integrations/orpc/services/storage.ts:178-196](), [src/integrations/orpc/services/storage.ts:284-305](), [src/routes/api/health.ts:68-78]()

### Health Check Response

```mermaid
graph TD
    HealthEndpoint["/api/health"] --> RunChecks["Run health checks"]
    
    RunChecks --> DBCheck["checkDatabase()"]
    RunChecks --> StorageCheck["checkStorage()"]
    RunChecks --> PrinterCheck["checkPrinter()"]
    
    DBCheck --> DBResult["{ status: 'healthy/unhealthy' }"]
    StorageCheck --> StorageResult["{ status, type, message, error? }"]
    PrinterCheck --> PrinterResult["{ status }"]
    
    DBResult --> Aggregate["Aggregate results"]
    StorageResult --> Aggregate
    PrinterResult --> Aggregate
    
    Aggregate --> IsUnhealthy{"Any component<br/>unhealthy?"}
    IsUnhealthy -->|Yes| Return500["Return 500<br/>status: 'unhealthy'"]
    IsUnhealthy -->|No| Return200["Return 200<br/>status: 'healthy'"]
```

**Sources:** [src/routes/api/health.ts:17-41]()

---

## Environment Configuration

### Required Variables

| Variable | Description |
|----------|-------------|
| `DATABASE_URL` | PostgreSQL connection string |

### Optional Storage Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `S3_ACCESS_KEY_ID` | S3 access key (required for S3 mode) | — |
| `S3_SECRET_ACCESS_KEY` | S3 secret key (required for S3 mode) | — |
| `S3_REGION` | S3 region | `"us-east-1"` |
| `S3_ENDPOINT` | S3 endpoint URL (for non-AWS providers) | — |
| `S3_BUCKET` | S3 bucket name | — |
| `S3_FORCE_PATH_STYLE` | Use path-style URLs | `false` |

If S3 credentials are not provided, the application automatically falls back to local filesystem storage in the `/data` directory.

**Sources:** [src/utils/env.ts:20-64](), [.env.example:15-56]()

### Docker Deployment

In Docker deployments, the data layer services are configured via `compose.yml`:

```yaml
postgres:
  image: postgres:latest
  environment:
    POSTGRES_DB: postgres
    POSTGRES_USER: postgres
    POSTGRES_PASSWORD: postgres
  volumes:
    - postgres_data:/var/lib/postgresql

seaweedfs:
  image: chrislusf/seaweedfs:latest
  command: server -s3 -filer -dir=/data -ip=0.0.0.0
  environment:
    - AWS_ACCESS_KEY_ID=seaweedfs
    - AWS_SECRET_ACCESS_KEY=seaweedfs
  volumes:
    - seaweedfs_data:/data
```

**Sources:** [compose.yml:4-57](), [docs/self-hosting/docker.mdx:154-211]()

---

