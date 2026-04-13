# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**CV Builder** is a commercial SaaS web application for AI-powered resume building and career management. It competes with Resume.io, Enhancv, Rezi, Teal, and similar paid services. The product is built on the Reactive Resume open-source foundation but is being developed into a distinct, monetizable product with features that go well beyond the upstream project.

**Key differentiators we are building:**
- AI Dashboard Assistant — conversational, v0-style resume creation from the main dashboard
- Multi-provider AI (OpenAI, Anthropic, Google, Ollama) with BYOK (Bring Your Own Key)
- MCP server for LLM integration — no other resume builder has this
- Job description analysis and ATS scoring
- Cover letter generation
- Privacy-first, self-hostable architecture

**Tech stack:** TanStack Start (React 19 + Vite 8), ORPC for type-safe RPC APIs, Drizzle ORM with PostgreSQL, Nitro server runtime, Better Auth, Vercel AI SDK v6. PWA with 47 locales and 13 resume templates.

**This is a product, not an open-source contribution.** When building features, think like a product engineer: prioritize user experience, conversion, and retention — not just technical correctness.

## Git & Upstream Sync

The codebase originated as a fork of `amruthpillai/reactive-resume`. We periodically sync bug fixes and community contributions from upstream, but our product diverges significantly.

Remotes:
- `origin` — nhaugaard/cv (our product repo) — **push here only**
- `upstream` — amruthpillai/reactive-resume — **fetch only, NEVER push**

Rules:
- **NEVER** push to upstream. No PRs, no force-pushes, nothing — upstream is read-only.
- To sync: `git fetch upstream && git merge upstream/main`. Resolve conflicts preserving **our changes** — our product features always take priority over upstream.
- New features should be built in separate files/folders where possible to minimize merge conflicts with upstream.

## Documentation and code check

Always use the Devin MCP before implementing new code. Ask Devin to approve every code change you make one at the time. The repo name is: "nhaugaard/cv" and use the ask tool to verify your code. 

## Development Commands

```bash
# Start development server (runs on port 3000)
pnpm dev

# Build for production
pnpm build

# Start production server
pnpm start

# Linting and formatting (uses Biome)
pnpm lint

# Type checking (uses tsgo)
pnpm typecheck

# Database operations
pnpm db:generate    # Generate migration files
pnpm db:migrate     # Run migrations
pnpm db:push        # Push schema changes directly
pnpm db:studio      # Open Drizzle Studio

# Extract i18n strings for translation
pnpm lingui:extract

# Find unused exports / dead code
dotenvx run -- pnpm knip
```

**There is no test framework configured.** No unit, integration, or E2E tests exist in the codebase.

## Local Development Setup

1. Copy `.env.example` to `.env` and configure environment variables
2. Start required services: `docker compose -f compose.dev.yml up -d`
   - PostgreSQL (port 5432)
   - Browserless/Chromium for PDF generation (port 4000)
   - SeaweedFS for S3-compatible storage (port 8333)
   - Mailpit for email testing (ports 1025, 8025)
   - Adminer for DB management (port 8080)
3. Run `pnpm dev`

Database migrations run automatically on server startup via the Nitro plugin at `plugins/1.migrate.ts`.

## Architecture

### Directory Structure

- `src/routes/` - TanStack Router file-based routing
- `src/integrations/` - External service integrations (auth, database, ORPC, AI, email, import)
- `src/integrations/orpc/router/` - oRPC server routers (procedure definitions)
- `src/integrations/orpc/services/` - oRPC server services (business logic)
- `src/integrations/orpc/dto/` - Data transfer objects
- `src/integrations/orpc/context.ts` - Auth and request context setup
- `src/components/` - React components organized by feature
- `src/components/ui/` - Shadcn UI components (Radix + Phosphor icons)
- `src/schema/` - Zod schemas for validation
- `src/hooks/` - Custom React hooks
- `plugins/` - Nitro server plugins (auto-migration on startup)
- `migrations/` - Drizzle database migrations
- `locales/` - i18n translation files (managed by Lingui)
- `docs/` - Documentation (Mintlify)

### Key Integrations (`src/integrations/`)

- **auth/** - Better Auth configuration (session-based + API key via `x-api-key` header)
- **drizzle/** - Database schema and client (PostgreSQL)
- **orpc/** - Type-safe RPC router with procedures for ai, auth, flags, printer, resume, statistics, storage
- **query/** - TanStack Query client configuration
- **ai/** - AI provider integrations (OpenAI, Anthropic, Google Gemini, Ollama)
- **email/** - Nodemailer integration (falls back to console logging if SMTP is not configured)
- **import/** - Resume file parsing/import

### ORPC Procedure Types

Three procedure types exist in `src/integrations/orpc/context.ts`:
- `publicProcedure` - No authentication required
- `protectedProcedure` - Requires authenticated user (session or API key)
- `serverOnlyProcedure` - Server-side calls only

Procedures follow this pattern:
```ts
const handler = protectedProcedure
  .route({ method: "GET", path: "/resumes/{id}", tags: ["Resumes"], ... })
  .input(schema)
  .output(schema)
  .handler(async ({ context, input }) => { ... })
```

### Resume Data Model

The resume schema is defined in `src/schema/resume/data.ts`. Key concepts:
- **ResumeData** - Complete resume data including basics, sections, customSections, metadata
- **Sections** - Built-in sections (profiles, experience, education, skills, etc.)
- **CustomSections** - User-created sections that follow one of the built-in section types
- **Metadata** - Template, layout, typography, design settings, custom CSS

### Resume Templates

13 templates in `src/components/resume/templates/` (Pokemon-themed names):
azurill, bronzor, chikorita, ditgar, ditto, gengar, glalie, kakuna, lapras, leafish, onyx, pikachu, rhyhorn

Shared rendering components live in `src/components/resume/shared/`.

### Database Schema

Defined in `src/integrations/drizzle/schema.ts`:
- `user`, `session`, `account`, `verification`, `twoFactor`, `passkey`, `apikey` - Better Auth tables
- `resume` - Stores Resume Data as JSONB (defined in `src/schema/resume/data.ts`)
- `resumeStatistics` - Views/Download tracking

### Routing

Uses TanStack Router with file-based routing. Key routes:
- `/_home/` - Public landing pages
- `/auth/` - Authentication flows
- `/dashboard/` - User dashboard and resume management
- `/builder/$resumeId/` - Resume editor
- `/printer/$resumeId/` - PDF rendering endpoint
- `/api/` - Public API endpoints
- `/mcp/` - MCP server endpoint for LLM integration

Routes use `createFileRoute()` with `beforeLoad()` for auth guards and `loader()` for server-side data fetching.

### MCP Server

An MCP (Model Context Protocol) server is available at `/mcp/` for LLM-based resume interaction. It requires an `x-api-key` header for authentication. Configuration is in `src/routes/mcp/` with helper modules for resources, prompts, and tools.

### State Management

- **Zustand** - Client-side state (resume editor state in `src/components/resume/store/`)
- **Zundo** - Undo/redo history for resume edits (built on Zustand)
- **TanStack Query** - Server state and caching (configured via ORPC integration)

### Global Providers

Defined in `src/routes/__root.tsx`:
- I18nProvider (Lingui), ThemeProvider, MotionConfig, IconContext (Phosphor Icons)
- ConfirmDialogProvider, PromptDialogProvider, DialogManager, CommandPalette, Toaster

## Code Style

- Uses **Biome** for linting and formatting (`biome.json`)
- Tab indentation, double quotes, 120 character line width
- Imports are auto-organized; unused imports are errors
- a11y rules are disabled
- Path alias: `@/` maps to `src/`
- Tailwind CSS v4 with sorted class names (enforced by Biome's `useSortedClasses`)
- Uses `cn()` utility (from `@/utils/style`) for conditional class names
- Uses `cva()` for component variants
- Shadcn UI components in `src/components/ui/` (Radix UI + Phosphor icons, zinc base color)
- i18n strings use Lingui macros: `<Trans>`, `t`, `msg`
- TypeScript strict mode enabled; `noUnusedLocals` and `noUnusedParameters` enforced

## Environment Variables

Key variables (see `.env.example` for full list):
- `APP_URL` - Application URL
- `DATABASE_URL` - PostgreSQL connection string
- `AUTH_SECRET` - Secret for authentication
- `PRINTER_ENDPOINT` - WebSocket endpoint for PDF printer service
- `PRINTER_APP_URL` - Internal URL for printer to reach the app (important for Docker)
- `S3_*` - S3-compatible storage configuration (falls back to local `/data` filesystem)
- `SMTP_*` - Email configuration (falls back to console logging)
- `GOOGLE_CLIENT_ID/SECRET` - Google OAuth (optional)
- `GITHUB_CLIENT_ID/SECRET` - GitHub OAuth (optional)
- `OAUTH_*` - Custom OAuth provider (optional)
- `FLAG_DEBUG_PRINTER` - Debug PDF printing endpoint
- `FLAG_DISABLE_SIGNUPS` - Block new account registration
- `FLAG_DISABLE_EMAIL_AUTH` - Disable email/password login
- `FLAG_DISABLE_IMAGE_PROCESSING` - Disable image processing

## Build & Deployment

- **Build output**: `.output/` directory (Nitro server bundle)
- **Production start**: `node .output/server/index.mjs`
- **Docker**: Multi-stage Dockerfile with Node 24-slim base
- **Health check**: `GET /api/health`
- **PWA**: Configured via vite-plugin-pwa with auto-update, standalone display, dark theme

### Agent Notes

This section documents common mistakes, confusion points, and surprising behaviors that agents encounter while working in this project.
If you encounter something in this project that surprises you or seems inconsistent, please alert the developer and propose adding a note here to help future agents avoid the same confusion.

<!-- Notes will be added below as agents encounter issues -->

<!-- gitnexus:start -->
# GitNexus — Code Intelligence

This project is indexed by GitNexus as **cv** (1963 symbols, 5276 relationships, 115 execution flows). Use the GitNexus MCP tools to understand code, assess impact, and navigate safely.

> If any GitNexus tool warns the index is stale, run `npx gitnexus analyze` in terminal first.

## Always Do

- **MUST run impact analysis before editing any symbol.** Before modifying a function, class, or method, run `gitnexus_impact({target: "symbolName", direction: "upstream"})` and report the blast radius (direct callers, affected processes, risk level) to the user.
- **MUST run `gitnexus_detect_changes()` before committing** to verify your changes only affect expected symbols and execution flows.
- **MUST warn the user** if impact analysis returns HIGH or CRITICAL risk before proceeding with edits.
- When exploring unfamiliar code, use `gitnexus_query({query: "concept"})` to find execution flows instead of grepping. It returns process-grouped results ranked by relevance.
- When you need full context on a specific symbol — callers, callees, which execution flows it participates in — use `gitnexus_context({name: "symbolName"})`.

## When Debugging

1. `gitnexus_query({query: "<error or symptom>"})` — find execution flows related to the issue
2. `gitnexus_context({name: "<suspect function>"})` — see all callers, callees, and process participation
3. `READ gitnexus://repo/cv/process/{processName}` — trace the full execution flow step by step
4. For regressions: `gitnexus_detect_changes({scope: "compare", base_ref: "main"})` — see what your branch changed

## When Refactoring

- **Renaming**: MUST use `gitnexus_rename({symbol_name: "old", new_name: "new", dry_run: true})` first. Review the preview — graph edits are safe, text_search edits need manual review. Then run with `dry_run: false`.
- **Extracting/Splitting**: MUST run `gitnexus_context({name: "target"})` to see all incoming/outgoing refs, then `gitnexus_impact({target: "target", direction: "upstream"})` to find all external callers before moving code.
- After any refactor: run `gitnexus_detect_changes({scope: "all"})` to verify only expected files changed.

## Never Do

- NEVER edit a function, class, or method without first running `gitnexus_impact` on it.
- NEVER ignore HIGH or CRITICAL risk warnings from impact analysis.
- NEVER rename symbols with find-and-replace — use `gitnexus_rename` which understands the call graph.
- NEVER commit changes without running `gitnexus_detect_changes()` to check affected scope.

## Tools Quick Reference

| Tool | When to use | Command |
|------|-------------|---------|
| `query` | Find code by concept | `gitnexus_query({query: "auth validation"})` |
| `context` | 360-degree view of one symbol | `gitnexus_context({name: "validateUser"})` |
| `impact` | Blast radius before editing | `gitnexus_impact({target: "X", direction: "upstream"})` |
| `detect_changes` | Pre-commit scope check | `gitnexus_detect_changes({scope: "staged"})` |
| `rename` | Safe multi-file rename | `gitnexus_rename({symbol_name: "old", new_name: "new", dry_run: true})` |
| `cypher` | Custom graph queries | `gitnexus_cypher({query: "MATCH ..."})` |

## Impact Risk Levels

| Depth | Meaning | Action |
|-------|---------|--------|
| d=1 | WILL BREAK — direct callers/importers | MUST update these |
| d=2 | LIKELY AFFECTED — indirect deps | Should test |
| d=3 | MAY NEED TESTING — transitive | Test if critical path |

## Resources

| Resource | Use for |
|----------|---------|
| `gitnexus://repo/cv/context` | Codebase overview, check index freshness |
| `gitnexus://repo/cv/clusters` | All functional areas |
| `gitnexus://repo/cv/processes` | All execution flows |
| `gitnexus://repo/cv/process/{name}` | Step-by-step execution trace |

## Self-Check Before Finishing

Before completing any code modification task, verify:
1. `gitnexus_impact` was run for all modified symbols
2. No HIGH/CRITICAL risk warnings were ignored
3. `gitnexus_detect_changes()` confirms changes match expected scope
4. All d=1 (WILL BREAK) dependents were updated

## Keeping the Index Fresh

After committing code changes, the GitNexus index becomes stale. Re-run analyze to update it:

```bash
npx gitnexus analyze
```

If the index previously included embeddings, preserve them by adding `--embeddings`:

```bash
npx gitnexus analyze --embeddings
```

To check whether embeddings exist, inspect `.gitnexus/meta.json` — the `stats.embeddings` field shows the count (0 means no embeddings). **Running analyze without `--embeddings` will delete any previously generated embeddings.**

> Claude Code users: A PostToolUse hook handles this automatically after `git commit` and `git merge`.

## CLI

| Task | Read this skill file |
|------|---------------------|
| Understand architecture / "How does X work?" | `.claude/skills/gitnexus/gitnexus-exploring/SKILL.md` |
| Blast radius / "What breaks if I change X?" | `.claude/skills/gitnexus/gitnexus-impact-analysis/SKILL.md` |
| Trace bugs / "Why is X failing?" | `.claude/skills/gitnexus/gitnexus-debugging/SKILL.md` |
| Rename / extract / split / refactor | `.claude/skills/gitnexus/gitnexus-refactoring/SKILL.md` |
| Tools, resources, schema reference | `.claude/skills/gitnexus/gitnexus-guide/SKILL.md` |
| Index, status, clean, wiki CLI commands | `.claude/skills/gitnexus/gitnexus-cli/SKILL.md` |

<!-- gitnexus:end -->
