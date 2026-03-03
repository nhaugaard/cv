# Page: Environment Configuration

# Environment Configuration

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.env.example](.env.example)
- [.gitignore](.gitignore)
- [.vscode/settings.json](.vscode/settings.json)
- [CLAUDE.md](CLAUDE.md)
- [compose.dev.yml](compose.dev.yml)
- [compose.yml](compose.yml)
- [docs/changelog/index.mdx](docs/changelog/index.mdx)
- [docs/contributing/development.mdx](docs/contributing/development.mdx)
- [docs/getting-started/quickstart.mdx](docs/getting-started/quickstart.mdx)
- [docs/guides/setting-up-passkeys.mdx](docs/guides/setting-up-passkeys.mdx)
- [docs/self-hosting/docker.mdx](docs/self-hosting/docker.mdx)
- [docs/self-hosting/examples.mdx](docs/self-hosting/examples.mdx)
- [docs/spec.json](docs/spec.json)
- [knip.json](knip.json)
- [package.json](package.json)
- [pnpm-lock.yaml](pnpm-lock.yaml)
- [scripts/fonts/generate.ts](scripts/fonts/generate.ts)
- [scripts/fonts/types.ts](scripts/fonts/types.ts)
- [src/components/resume/preview.module.css](src/components/resume/preview.module.css)
- [src/components/typography/combobox.tsx](src/components/typography/combobox.tsx)
- [src/components/typography/webfontlist.json](src/components/typography/webfontlist.json)
- [src/integrations/auth/client.ts](src/integrations/auth/client.ts)
- [src/integrations/auth/config.ts](src/integrations/auth/config.ts)
- [src/integrations/orpc/router/storage.ts](src/integrations/orpc/router/storage.ts)
- [src/integrations/orpc/services/storage.ts](src/integrations/orpc/services/storage.ts)
- [src/routes/__root.tsx](src/routes/__root.tsx)
- [src/routes/api/health.ts](src/routes/api/health.ts)
- [src/routes/auth/-components/social-auth.tsx](src/routes/auth/-components/social-auth.tsx)
- [src/routes/auth/login.tsx](src/routes/auth/login.tsx)
- [src/routes/auth/register.tsx](src/routes/auth/register.tsx)
- [src/routes/builder/$resumeId/-sidebar/right/sections/typography.tsx](src/routes/builder/$resumeId/-sidebar/right/sections/typography.tsx)
- [src/routes/dashboard/settings/authentication/-components/hooks.tsx](src/routes/dashboard/settings/authentication/-components/hooks.tsx)
- [src/utils/env.ts](src/utils/env.ts)
- [src/vite-env.d.ts](src/vite-env.d.ts)
- [vite.config.ts](vite.config.ts)

</details>



This document details all environment variables used by Reactive Resume, including required and optional configuration, validation rules, and default values. For deployment-specific setup instructions, see [Docker Deployment](#5.1) and [Self-Hosting Guide](#5.2).

## Purpose and Scope

Reactive Resume uses environment variables to configure application behavior across server settings, database connections, authentication providers, storage backends, email delivery, and feature flags. The configuration system validates all variables at startup using Zod schemas, ensuring type safety and preventing runtime errors from misconfiguration.

All environment variables are processed through [`src/utils/env.ts:4-72`](), which provides centralized validation, type coercion, and default values.

---

## Configuration System Architecture

**`env` object flow — from `.env` to typed consumers**

```mermaid
graph TB
    EnvFile[".env file"]
    EnvExample[".env.example (reference)"]

    ProcessEnv["process.env (Node.js)"]

    createEnv["createEnv() — @t3-oss/env-core\nsrc/utils/env.ts"]

    ZodSchemas["Zod schema validation\n(type coercion + defaults)"]

    envObj["export const env\n(type-safe object)"]

    AuthConfig["src/integrations/auth/config.ts\ngetAuthConfig()"]
    StorageSvc["src/integrations/orpc/services/storage.ts\ngetStorageService()"]
    PrinterSvc["src/integrations/orpc/services/printer.ts"]
    EmailSvc["src/integrations/email/service.ts"]
    DrizzleClient["src/integrations/drizzle/client.ts"]

    EnvFile --> ProcessEnv
    ProcessEnv --> createEnv
    createEnv --> ZodSchemas
    ZodSchemas --> envObj

    envObj --> AuthConfig
    envObj --> StorageSvc
    envObj --> PrinterSvc
    envObj --> EmailSvc
    envObj --> DrizzleClient

    EnvExample -.->|"reference"| EnvFile
```

Sources: [src/utils/env.ts:1-72](), [.env.example:1-78]()

The configuration system uses `@t3-oss/env-core`'s `createEnv` function to produce the `env` object from `process.env`. All variables pass through Zod schemas that enforce types, formats, and constraints. Invalid configuration causes the application to fail at startup with descriptive error messages. Empty strings are treated as `undefined` (`emptyStringAsUndefined: true` at [src/utils/env.ts:7]()).

---

## Environment Variable Loading

**Startup validation sequence**

```mermaid
graph LR
    Startup["Application startup\n(node .output/server/index.mjs)"]
    LoadEnv["process.env"]
    Validate["createEnv()\nsrc/utils/env.ts:4"]

    Success["export const env\n(consumed by integrations)"]
    Failure["ZodError thrown\nprocess exits"]

    Startup --> LoadEnv
    LoadEnv --> Validate

    Validate -->|"valid"| Success
    Validate -->|"invalid"| Failure
```

Sources: [src/utils/env.ts:4-7](), [package.json:30]()

The `createEnv` function from `@t3-oss/env-core` validates all variables before the Nitro server begins handling requests. Validation errors are descriptive Zod messages identifying the offending variable and constraint.

---

## Required Variables

The following variables must be set for the application to start:

| Variable | Type | Validation | Description |
|----------|------|------------|-------------|
| `APP_URL` | URL | `http://` or `https://` protocol | Public URL of the application instance |
| `DATABASE_URL` | URL | `postgres://` or `postgresql://` protocol | PostgreSQL connection string |
| `AUTH_SECRET` | String | Minimum 1 character | Secret key for session signing and encryption |
| `PRINTER_ENDPOINT` | URL | `ws://`, `wss://`, `http://`, or `https://` protocol | WebSocket or HTTP endpoint for the printer service |

**Sources:** [src/utils/env.ts:14-18](), [src/vite-env.d.ts:12-23]()

### Generating AUTH_SECRET

Generate a cryptographically secure secret:

```bash
openssl rand -hex 32
```

Changing `AUTH_SECRET` invalidates all existing user sessions.

**Sources:** [docs/self-hosting/docker.mdx:127-145]()

---

## Server Configuration

```mermaid
graph TB
    subgraph "Server Variables"
        TZ["TZ<br/>Default: Etc/UTC"]
        AppURL["APP_URL<br/>Required"]
        PrinterAppURL["PRINTER_APP_URL<br/>Optional"]
    end
    
    subgraph "Usage"
        Logging["Log Timestamps"]
        URLs["Absolute URL Generation"]
        AuthRedirects["OAuth Redirects"]
        PrinterRouting["Printer → App Requests"]
    end
    
    TZ --> Logging
    AppURL --> URLs
    AppURL --> AuthRedirects
    PrinterAppURL --> PrinterRouting
    AppURL -.->|"Fallback"| PrinterRouting
```

**Sources:** [src/utils/env.ts:13-15](), [.env.example:1-10]()

| Variable | Default | Validation | Description |
|----------|---------|------------|-------------|
| `TZ` | `Etc/UTC` | String | Timezone for server logs and timestamps |
| `APP_URL` | **Required** | Valid HTTP(S) URL | Public-facing URL (used for redirects, OAuth callbacks, email links) |
| `PRINTER_APP_URL` | `APP_URL` | Valid HTTP(S) URL | Internal URL for printer to reach the app (useful when printer is in Docker and app URL is external) |

### PRINTER_APP_URL Use Case

When the printer service runs in Docker and needs to access the application that listens on `localhost:3000` on the host, set:

```bash
PRINTER_APP_URL="http://host.docker.internal:3000"
```

This special hostname allows Docker containers to communicate with services on the host machine. In Docker Compose, `PRINTER_APP_URL` is typically set to the app service name (e.g., `http://reactive_resume:3000`) so the printer container can reach it over the internal Docker network.

Sources: [.env.example:6-10](), [compose.yml:81]()

---

## Database Configuration

```mermaid
graph LR
    DatabaseURL["DATABASE_URL"]
    
    Connection["PostgreSQL Connection"]
    
    Drizzle["Drizzle ORM Client<br/>src/integrations/drizzle/client.ts"]
    
    Migrations["Database Migrations<br/>migrations/"]
    
    Schema["Database Schema<br/>src/integrations/drizzle/schema.ts"]
    
    DatabaseURL --> Connection
    Connection --> Drizzle
    Drizzle --> Migrations
    Drizzle --> Schema
```

**Sources:** [src/utils/env.ts:21](), [.env.example:16]()

| Variable | Format | Example |
|----------|--------|---------|
| `DATABASE_URL` | `postgresql://USER:PASSWORD@HOST:PORT/DATABASE` | `postgresql://postgres:postgres@localhost:5432/postgres` |

The connection string must use the `postgres://` or `postgresql://` protocol. In Docker Compose, use the service name (e.g., `postgres`) as the hostname instead of `localhost`.

Database migrations run automatically on application startup via the Nitro plugin at `plugins/1.migrate.ts`. See page [2.3]() for full data layer documentation.

Sources: [src/utils/env.ts:21](), [CLAUDE.md:53]()

---

## Authentication Configuration

### Core Authentication

```mermaid
graph TB
    AuthSecret["AUTH_SECRET<br/>Required"]
    
    BetterAuth["Better Auth Core<br/>src/integrations/auth/"]
    
    Sessions["Session Management<br/>Cookie-based"]
    ApiKeys["API Key Authentication<br/>x-api-key header"]
    EmailAuth["Email/Password Login"]
    
    AuthSecret --> BetterAuth
    BetterAuth --> Sessions
    BetterAuth --> ApiKeys
    BetterAuth --> EmailAuth
```

**Sources:** [src/utils/env.ts:24](), [src/integrations/auth/]()

| Variable | Required | Description |
|----------|----------|-------------|
| `AUTH_SECRET` | Yes | Secret for signing sessions and tokens |

### Social Authentication Providers

```mermaid
graph TB
    subgraph "Google OAuth"
        GoogleID["GOOGLE_CLIENT_ID"]
        GoogleSecret["GOOGLE_CLIENT_SECRET"]
    end
    
    subgraph "GitHub OAuth"
        GitHubID["GITHUB_CLIENT_ID"]
        GitHubSecret["GITHUB_CLIENT_SECRET"]
    end
    
    subgraph "Custom OAuth"
        OAuthName["OAUTH_PROVIDER_NAME"]
        OAuthID["OAUTH_CLIENT_ID"]
        OAuthSecret["OAUTH_CLIENT_SECRET"]
        OAuthDiscovery["OAUTH_DISCOVERY_URL"]
        OAuthManual["OAUTH_AUTHORIZATION_URL<br/>OAUTH_TOKEN_URL<br/>OAUTH_USER_INFO_URL"]
        OAuthScopes["OAUTH_SCOPES"]
    end
    
    BetterAuth["Better Auth Configuration<br/>src/integrations/auth/"]
    
    GoogleID --> BetterAuth
    GoogleSecret --> BetterAuth
    GitHubID --> BetterAuth
    GitHubSecret --> BetterAuth
    
    OAuthName --> BetterAuth
    OAuthID --> BetterAuth
    OAuthSecret --> BetterAuth
    OAuthDiscovery --> BetterAuth
    OAuthManual --> BetterAuth
    OAuthScopes --> BetterAuth
```

**Sources:** [src/utils/env.ts:26-46](), [.env.example:22-36]()

#### Google OAuth

| Variable | Description |
|----------|-------------|
| `GOOGLE_CLIENT_ID` | Google OAuth 2.0 Client ID |
| `GOOGLE_CLIENT_SECRET` | Google OAuth 2.0 Client Secret |

#### GitHub OAuth

| Variable | Description |
|----------|-------------|
| `GITHUB_CLIENT_ID` | GitHub OAuth App Client ID |
| `GITHUB_CLIENT_SECRET` | GitHub OAuth App Client Secret |

#### Custom OAuth Provider

The custom OAuth provider is activated by `isCustomOAuthProviderEnabled()` in [src/integrations/auth/config.ts:14-19](). It returns `true` only when `OAUTH_CLIENT_ID` and `OAUTH_CLIENT_SECRET` are set, and either `OAUTH_DISCOVERY_URL` (Option A) or all three of `OAUTH_AUTHORIZATION_URL`, `OAUTH_TOKEN_URL`, and `OAUTH_USER_INFO_URL` (Option B) are set.

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `OAUTH_PROVIDER_NAME` | No | — | Display name shown in the UI |
| `OAUTH_CLIENT_ID` | Required for custom OAuth | — | OAuth Client ID |
| `OAUTH_CLIENT_SECRET` | Required for custom OAuth | — | OAuth Client Secret |
| `OAUTH_DISCOVERY_URL` | Option A | — | OIDC discovery endpoint (preferred for OIDC-compliant providers) |
| `OAUTH_AUTHORIZATION_URL` | Option B | — | OAuth authorization endpoint (if not using discovery) |
| `OAUTH_TOKEN_URL` | Option B | — | OAuth token endpoint (if not using discovery) |
| `OAUTH_USER_INFO_URL` | Option B | — | OAuth user info endpoint (if not using discovery) |
| `OAUTH_SCOPES` | No | `openid profile email` | Space-separated scopes (parsed to `string[]` at startup) |

**Configuration Methods:**

- **Option A (OIDC Discovery):** Set only `OAUTH_DISCOVERY_URL` to the provider's `.well-known/openid-configuration` endpoint
- **Option B (Manual URLs):** Set all three: `OAUTH_AUTHORIZATION_URL`, `OAUTH_TOKEN_URL`, and `OAUTH_USER_INFO_URL`

Sources: [src/utils/env.ts:35-46](), [src/integrations/auth/config.ts:14-19](), [src/integrations/auth/config.ts:43-54]()

---

## Email Configuration (SMTP)

```mermaid
graph TB
    subgraph "SMTP Variables"
        SmtpHost["SMTP_HOST<br/>Optional"]
        SmtpPort["SMTP_PORT<br/>Default: 587"]
        SmtpUser["SMTP_USER<br/>Optional"]
        SmtpPass["SMTP_PASS<br/>Optional"]
        SmtpFrom["SMTP_FROM<br/>Optional"]
        SmtpSecure["SMTP_SECURE<br/>Default: false"]
    end
    
    EmailService["Email Service<br/>src/integrations/email/"]
    
    Configured{"All SMTP<br/>vars set?"}
    
    SendEmail["Send via SMTP<br/>Nodemailer"]
    LogEmail["Log to Console"]
    
    SmtpHost --> Configured
    SmtpPort --> Configured
    SmtpUser --> Configured
    SmtpPass --> Configured
    SmtpFrom --> Configured
    SmtpSecure --> Configured
    
    Configured -->|"Yes"| SendEmail
    Configured -->|"No"| LogEmail
    
    EmailService --> Configured
```

**Sources:** [src/utils/env.ts:49-54](), [.env.example:37-44]()

| Variable | Default | Validation | Description |
|----------|---------|------------|-------------|
| `SMTP_HOST` | — | Non-empty string | SMTP server hostname |
| `SMTP_PORT` | `587` | Integer 1-65535 | SMTP server port (587 for STARTTLS, 465 for implicit TLS) |
| `SMTP_USER` | — | Non-empty string | SMTP authentication username |
| `SMTP_PASS` | — | Non-empty string | SMTP authentication password |
| `SMTP_FROM` | — | Non-empty string | Default "From" address (e.g., `Reactive Resume <noreply@rxresu.me>`) |
| `SMTP_SECURE` | `false` | Boolean string | Use implicit TLS (`true`) or STARTTLS (`false`) |

If any SMTP variable is unset, the application falls back to logging emails to the console instead of sending them. This is useful for local development.

**Common SMTP Ports:**
- `587`: STARTTLS (explicit TLS upgrade)
- `465`: Implicit TLS/SSL
- `25`: Unencrypted (not recommended)

**Sources:** [src/utils/env.ts:49-54](), [.env.example:37-44]()

---

## Storage Configuration

Reactive Resume supports two storage backends: S3-compatible object storage or local filesystem.

**`getStorageService()` — backend selection**

```mermaid
graph TB
    getStorageService["getStorageService()\nsrc/integrations/orpc/services/storage.ts"]

    Check{"env.S3_ACCESS_KEY_ID\nenv.S3_SECRET_ACCESS_KEY\nenv.S3_BUCKET all set?"}

    S3Svc["S3StorageService\n(@aws-sdk/client-s3)\nuses env.S3_* variables"]
    LocalSvc["LocalStorageService\n(node:fs/promises)\nrootDirectory = cwd()/data"]

    StorageInterface["StorageService interface\nlist / write / read / delete / healthcheck"]

    storageRouter["storageRouter\nsrc/integrations/orpc/router/storage.ts"]
    printerSvc["printerService\n(screenshots, PDFs)"]

    getStorageService --> Check
    Check -->|"yes"| S3Svc
    Check -->|"no"| LocalSvc
    S3Svc --> StorageInterface
    LocalSvc --> StorageInterface
    StorageInterface --> storageRouter
    StorageInterface --> printerSvc
```

Sources: [src/integrations/orpc/services/storage.ts:113-208](), [src/integrations/orpc/services/storage.ts:210-282](), [src/utils/env.ts:57-64]()

### Storage Backend Selection

The storage backend is automatically selected based on whether S3 credentials are configured:

- **S3-compatible storage (`S3StorageService`):** Used if `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, and `S3_BUCKET` are all set. Credentials are checked in the `S3StorageService` constructor at [src/integrations/orpc/services/storage.ts:215-218]().
- **Local filesystem (`LocalStorageService`):** Used as fallback, storing files in `<cwd>/data` (i.e., `/app/data` in Docker).

Sources: [src/integrations/orpc/services/storage.ts:210-229]()

### S3-Compatible Storage Variables

| Variable | Default | Validation | Description |
|----------|---------|------------|-------------|
| `S3_ACCESS_KEY_ID` | — | Non-empty string | S3 access key ID |
| `S3_SECRET_ACCESS_KEY` | — | Non-empty string | S3 secret access key |
| `S3_REGION` | `us-east-1` | String | AWS region or equivalent |
| `S3_ENDPOINT` | — | Valid HTTP(S) URL | S3-compatible endpoint (required for non-AWS providers like MinIO, SeaweedFS) |
| `S3_BUCKET` | — | Non-empty string | Bucket name for storing uploads |
| `S3_FORCE_PATH_STYLE` | `false` | Boolean string | URL addressing style (see below) |

Sources: [src/utils/env.ts:57-64](), [.env.example:46-56]()

### S3_FORCE_PATH_STYLE

This setting controls how the S3 client constructs bucket URLs:

| Value | URL Style | Example | Use With |
|-------|-----------|---------|----------|
| `true` | Path-style | `https://endpoint.com/bucket/key` | MinIO, SeaweedFS, self-hosted S3 |
| `false` | Virtual-hosted-style | `https://bucket.endpoint.com/key` | AWS S3, Cloudflare R2 |

**Common Issue:** If you see `ENOTFOUND mybucket.endpoint.com` errors, set `S3_FORCE_PATH_STYLE="true"`.

Sources: [src/utils/env.ts:62-64](), [.env.example:56]()

### Local Storage Configuration

When S3 is not configured, files are stored in `/app/data`. In Docker deployments, mount this directory to persistent storage:

```yaml
volumes:
  - ./data:/app/data
```

The local storage service creates the directory structure automatically using `buildPictureKey`, `buildScreenshotKey`, and `buildPdfKey` helpers:

```
/app/data/
└── uploads/
    └── {userId}/
        ├── pictures/
        │   └── {timestamp}.webp
        ├── screenshots/
        │   └── {resumeId}/
        │       └── {timestamp}.webp
        └── pdfs/
            └── {resumeId}/
                └── {timestamp}.pdf
```

Sources: [src/integrations/orpc/services/storage.ts:56-68](), [src/integrations/orpc/services/storage.ts:113-208](), [.env.example:48-49]()

---

## Printer Configuration

```mermaid
graph LR
    PrinterEndpoint["PRINTER_ENDPOINT"]
    
    Browserless["Browserless/Chromium<br/>WebSocket connection"]
    ChromeDP["chromedp/headless-shell<br/>HTTP connection"]
    
    PrinterService["Printer Service<br/>src/integrations/orpc/services/printer.ts"]
    
    PDFGeneration["PDF Generation"]
    Screenshots["Screenshot Capture"]
    
    PrinterEndpoint --> Browserless
    PrinterEndpoint --> ChromeDP
    
    Browserless --> PrinterService
    ChromeDP --> PrinterService
    
    PrinterService --> PDFGeneration
    PrinterService --> Screenshots
```

**Sources:** [src/utils/env.ts:18](), [.env.example:12-13]()

| Variable | Format | Examples |
|----------|--------|----------|
| `PRINTER_ENDPOINT` | WebSocket or HTTP URL | `ws://localhost:4000?token=1234567890`<br/>`http://localhost:9222` |

The printer service supports two backends:

1. **Browserless** (recommended): WebSocket connection with optional token authentication
   - Example: `ws://browserless:3000?token=1234567890`
   - Supports concurrent rendering, queuing, and health checks

2. **chromedp/headless-shell**: HTTP connection to Chrome DevTools Protocol
   - Example: `http://chrome:9222`
   - Lightweight alternative with smaller Docker image footprint

**Sources:** [.env.example:12-13](), [docs/self-hosting/docker.mdx:216-227]()

---

## Feature Flags

Feature flags are boolean variables (`z.stringbool()`) defaulting to `false`. They control optional behaviors and debugging features at runtime.

**Feature flags and their code consumers**

```mermaid
graph TB
    subgraph "env flags (src/utils/env.ts:67-70)"
        FDP["FLAG_DEBUG_PRINTER"]
        FDS["FLAG_DISABLE_SIGNUPS"]
        FDEA["FLAG_DISABLE_EMAIL_AUTH"]
        FDIP["FLAG_DISABLE_IMAGE_PROCESSING"]
    end

    subgraph "Consumers"
        PrinterRoute["printer route\n(/printer/:resumeId)\nserver-only guard"]
        AuthConfig["getAuthConfig()\nsrc/integrations/auth/config.ts\ndisableSignUp / enabled flags"]
        ProcessImage["processImageForUpload()\nsrc/integrations/orpc/services/storage.ts:89-111\nskips sharp resize/webp conversion"]
        FlagsRouter["flagsRouter\norpc flags procedure\n(exposed to client)"]
    end

    FDP --> PrinterRoute
    FDS --> AuthConfig
    FDEA --> AuthConfig
    FDIP --> ProcessImage
    FDS --> FlagsRouter
    FDEA --> FlagsRouter
```

Sources: [src/utils/env.ts:67-70](), [src/vite-env.d.ts:51-56](), [src/integrations/auth/config.ts:96-113](), [src/integrations/orpc/services/storage.ts:89-111]()

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `FLAG_DEBUG_PRINTER` | `false` | Boolean | Bypasses server-only restriction on `/printer/{resumeId}` route (useful for debugging PDF rendering) |
| `FLAG_DISABLE_SIGNUPS` | `false` | Boolean | Disables new user registration (existing users can still sign in) |
| `FLAG_DISABLE_EMAIL_AUTH` | `false` | Boolean | Disables email/password authentication; users can only sign in via OAuth providers |
| `FLAG_DISABLE_IMAGE_PROCESSING` | `false` | Boolean | Skips image resize/optimization (useful for low-resource environments like Raspberry Pi) |

### FLAG_DEBUG_PRINTER

The printer route at `/printer/{resumeId}` is normally restricted to server-side requests only. Setting `FLAG_DEBUG_PRINTER="true"` allows browser access for debugging PDF rendering issues.

**⚠ Security Warning:** Never enable this flag in production, as it exposes internal resume rendering endpoints.

**Sources:** [.env.example:59-61](), [src/vite-env.d.ts:52]()

### FLAG_DISABLE_IMAGE_PROCESSING

When `true`, uploaded images are stored in their original format without resize or WebP conversion. This reduces CPU usage but increases storage requirements. Useful for low-resource deployments (e.g., Raspberry Pi).

Normal behavior (when `false`) inside `processImageForUpload()`:
- Images are resized to max 800×800px (`fit: "inside"`)
- Converted to WebP format (`sharp.webp({ preset: "picture" })`)
- Optimized for web delivery

Sources: [src/integrations/orpc/services/storage.ts:89-111](), [.env.example:69-71]()

---

## Environment Variable Validation

### Validation Schema

All environment variables are validated using Zod schemas defined in [src/utils/env.ts:4-72](). The validation occurs at application startup, before any requests are processed.

**Zod schema application per variable group**

```mermaid
graph TB
    createEnv["createEnv() — src/utils/env.ts:4"]

    subgraph "server: schema block"
        URLVars["APP_URL, PRINTER_APP_URL, DATABASE_URL\nz.url({ protocol: /regex/ })"]
        StringVars["AUTH_SECRET, SMTP_HOST, S3_BUCKET, ...\nz.string().min(1).optional()"]
        BoolVars["FLAG_*, SMTP_SECURE, S3_FORCE_PATH_STYLE\nz.stringbool().default(false)"]
        NumVars["SMTP_PORT\nz.coerce.number().int().min(1).max(65535)"]
        ArrayVar["OAUTH_SCOPES\nz.string().transform(v => v.split(' '))\n.default(['openid','profile','email'])"]
    end

    createEnv --> URLVars
    createEnv --> StringVars
    createEnv --> BoolVars
    createEnv --> NumVars
    createEnv --> ArrayVar
```

Sources: [src/utils/env.ts:4-72]()

### Type Coercion Rules

The validation system applies automatic type coercion:

| Type | Zod operator | Input example | Output |
|------|-------------|---------------|--------|
| Boolean strings | `z.stringbool()` | `"true"`, `"1"`, `"false"` | `true` / `false` |
| Numbers | `z.coerce.number()` | `"587"` | `587` |
| URLs | `z.url({ protocol: /.../ })` | `"http://localhost:3000"` | Validated string |
| String arrays | `.transform(v => v.split(" "))` | `"openid profile email"` | `["openid", "profile", "email"]` |

Sources: [src/utils/env.ts:50](), [src/utils/env.ts:42-46]()

### Empty String Handling

By default, empty strings are treated as `undefined` (`emptyStringAsUndefined: true` at [src/utils/env.ts:7]()). This means:

```bash
# These are equivalent — both result in the variable being undefined:
GOOGLE_CLIENT_ID=""
# (not setting GOOGLE_CLIENT_ID at all)
```

This allows optional variables like social OAuth credentials to be left blank in `.env` files without causing validation errors.

Sources: [src/utils/env.ts:7]()

### Script-Only Variables

The `.env.example` file includes `GOOGLE_CLOUD_API_KEY`, which is **not** validated by `createEnv` and is **not** part of the `env` object. It is used solely by the font generation script at `scripts/fonts/generate.ts` to query the Google Fonts Developer API. This variable does not affect the running application.

Sources: [.env.example:74-77](), [scripts/fonts/generate.ts:1-10]()

---

## Configuration Examples

### Minimal Production Setup

```bash
# Required variables only
APP_URL="https://resume.example.com"
DATABASE_URL="postgresql://user:pass@postgres:5432/db"
AUTH_SECRET="abc123..." # openssl rand -hex 32
PRINTER_ENDPOINT="ws://browserless:3000"
```

Sources: [.env.example:1-20]()

### Full Production Setup with OAuth and S3

```bash
# Server
APP_URL="https://resume.example.com"
DATABASE_URL="postgresql://user:pass@postgres:5432/db"
AUTH_SECRET="abc123..."
PRINTER_ENDPOINT="ws://browserless:3000"

# OAuth Providers
GOOGLE_CLIENT_ID="..."
GOOGLE_CLIENT_SECRET="..."
GITHUB_CLIENT_ID="..."
GITHUB_CLIENT_SECRET="..."

# Email
SMTP_HOST="smtp.sendgrid.net"
SMTP_PORT="587"
SMTP_USER="apikey"
SMTP_PASS="SG.xyz..."
SMTP_FROM="Reactive Resume <noreply@example.com>"
SMTP_SECURE="false"

# S3 Storage
S3_ACCESS_KEY_ID="minioadmin"
S3_SECRET_ACCESS_KEY="minioadmin"
S3_ENDPOINT="https://s3.example.com"
S3_BUCKET="resumes"
S3_FORCE_PATH_STYLE="true"
```

Sources: [.env.example:1-78]()

### Development Setup

Uses services from `compose.dev.yml` — Postgres on 5432, Browserless on 4000, SeaweedFS on 8333, Mailpit SMTP on 1025.

```bash
# Server
APP_URL="http://localhost:3000"
PRINTER_APP_URL="http://host.docker.internal:3000"
PRINTER_ENDPOINT="ws://localhost:4000?token=1234567890"

# Database (local Docker)
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/postgres"

# Auth
AUTH_SECRET="development-secret"

# Storage (local SeaweedFS)
S3_ACCESS_KEY_ID="seaweedfs"
S3_SECRET_ACCESS_KEY="seaweedfs"
S3_ENDPOINT="http://localhost:8333"
S3_BUCKET="reactive-resume"
S3_FORCE_PATH_STYLE="true"

# Email (Mailpit)
SMTP_HOST="localhost"
SMTP_PORT="1025"

# Debug flags
FLAG_DEBUG_PRINTER="true"
```

Sources: [compose.dev.yml:1-114](), [.env.example:1-78]()

---

## Environment Variable Reference Table

### Complete Variable Listing

| Variable | Required | Default | Type | Validation |
|----------|----------|---------|------|------------|
| `TZ` | No | `Etc/UTC` | String | — |
| `APP_URL` | Yes | — | URL | HTTP(S) protocol |
| `PRINTER_APP_URL` | No | `APP_URL` | URL | HTTP(S) protocol |
| `PRINTER_ENDPOINT` | Yes | — | URL | WS(S)/HTTP(S) protocol |
| `DATABASE_URL` | Yes | — | URL | PostgreSQL protocol |
| `AUTH_SECRET` | Yes | — | String | Min 1 character |
| `GOOGLE_CLIENT_ID` | No | — | String | Min 1 character |
| `GOOGLE_CLIENT_SECRET` | No | — | String | Min 1 character |
| `GITHUB_CLIENT_ID` | No | — | String | Min 1 character |
| `GITHUB_CLIENT_SECRET` | No | — | String | Min 1 character |
| `OAUTH_PROVIDER_NAME` | No | — | String | Min 1 character |
| `OAUTH_CLIENT_ID` | No | — | String | Min 1 character |
| `OAUTH_CLIENT_SECRET` | No | — | String | Min 1 character |
| `OAUTH_DISCOVERY_URL` | No | — | URL | HTTP(S) protocol |
| `OAUTH_AUTHORIZATION_URL` | No | — | URL | HTTP(S) protocol |
| `OAUTH_TOKEN_URL` | No | — | URL | HTTP(S) protocol |
| `OAUTH_USER_INFO_URL` | No | — | URL | HTTP(S) protocol |
| `OAUTH_SCOPES` | No | `openid profile email` | String[] | Space-separated |
| `SMTP_HOST` | No | — | String | Min 1 character |
| `SMTP_PORT` | No | `587` | Number | 1-65535 |
| `SMTP_USER` | No | — | String | Min 1 character |
| `SMTP_PASS` | No | — | String | Min 1 character |
| `SMTP_FROM` | No | — | String | Min 1 character |
| `SMTP_SECURE` | No | `false` | Boolean | String boolean |
| `S3_ACCESS_KEY_ID` | No | — | String | Min 1 character |
| `S3_SECRET_ACCESS_KEY` | No | — | String | Min 1 character |
| `S3_REGION` | No | `us-east-1` | String | — |
| `S3_ENDPOINT` | No | — | URL | HTTP(S) protocol |
| `S3_BUCKET` | No | — | String | Min 1 character |
| `S3_FORCE_PATH_STYLE` | No | `false` | Boolean | String boolean |
| `FLAG_DEBUG_PRINTER` | No | `false` | Boolean | String boolean |
| `FLAG_DISABLE_SIGNUPS` | No | `false` | Boolean | String boolean |
| `FLAG_DISABLE_EMAIL_AUTH` | No | `false` | Boolean | String boolean |
| `FLAG_DISABLE_IMAGE_PROCESSING` | No | `false` | Boolean | String boolean |

**Sources:** [src/utils/env.ts:11-71](), [src/vite-env.d.ts:10-56](), [.env.example:1-78]()

---

