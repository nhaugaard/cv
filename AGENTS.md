# AGENTS.md

## Overview

Reactive Resume is a single-package full-stack TypeScript app (not a monorepo) built with [TanStack Start](https://tanstack.com/start/latest/docs/framework/react/overview) (React, Vite, Nitro). It serves both frontend and API on port 3000.

This project uses [Vite+](https://vite.dev/blog/announcing-viteplus), a unified toolchain built on top of Vite, Rolldown, Vitest, tsdown, Oxlint, Oxfmt, and Vite Task. Vite+ wraps runtime management, package management, and frontend tooling in a single global CLI called `vp`. All modules should be imported from the `vite-plus` dependency (e.g., `import { defineConfig } from 'vite-plus'` or `import { expect, test, vi } from 'vite-plus/test'`).

## Key Libraries

| Area                 | Library                                                                  | Docs                               |
| -------------------- | ------------------------------------------------------------------------ | ---------------------------------- |
| Frontend framework   | React                                                                    | https://react.dev                  |
| Full-stack framework | TanStack Start                                                           | https://tanstack.com/start/latest  |
| Router               | TanStack React Router                                                    | https://tanstack.com/router/latest |
| Server state         | TanStack React Query                                                     | https://tanstack.com/query/latest  |
| Client state         | Zustand (+ Zundo for undo/redo, Immer for immutable updates)             | https://zustand.docs.pmnd.rs       |
| Type-safe API        | oRPC                                                                     | https://orpc.unnoq.com             |
| Database ORM         | Drizzle ORM (PostgreSQL)                                                 | https://orm.drizzle.team           |
| Authentication       | Better Auth (+ Drizzle adapter, OAuth provider, API keys, 2FA, Passkeys) | https://www.better-auth.com        |
| Styling              | Tailwind CSS                                                             | https://tailwindcss.com            |
| UI Components        | shadcn/ui (built on Base UI)                                             | https://ui.shadcn.com              |
| Icons                | Phosphor Icons                                                           | https://phosphoricons.com          |
| Forms                | React Hook Form (+ Zod resolvers)                                        | https://react-hook-form.com        |
| Rich text editor     | Tiptap                                                                   | https://tiptap.dev                 |
| Validation           | Zod                                                                      | https://zod.dev                    |
| AI                   | Vercel AI SDK (OpenAI, Anthropic, Google, Ollama providers)              | https://ai-sdk.dev                 |
| MCP                  | Model Context Protocol SDK                                               | https://modelcontextprotocol.io    |
| i18n                 | Lingui                                                                   | https://lingui.dev                 |
| Animations           | Motion (Framer Motion)                                                   | https://motion.dev                 |
| PDF export           | Puppeteer Core (via Browserless)                                         | https://pptr.dev                   |
| Drag and drop        | dnd-kit                                                                  | https://dndkit.com                 |
| Server engine        | Nitro                                                                    | https://nitro.build                |
| PWA                  | Vite PWA Plugin                                                          | https://vite-pwa-org.netlify.app   |
| Unused deps          | Knip                                                                     | https://knip.dev                   |

## Project Structure

```
src/
  components/     UI, resume, layout, animation, theme, locale components
  routes/         File-based routing (TanStack React Router)
  integrations/   Feature modules (auth, drizzle, orpc, ai, email, jobs, mcp, storage)
  schema/         Zod schemas for resume data validation
  utils/          Utility functions (locale, theme, env, resume processing)
  dialogs/        Modal/dialog components
  hooks/          Custom React hooks
  styles/         CSS and Tailwind configuration
  stores/         Zustand stores (resume, AI, dialog, command palette)
migrations/       Drizzle database migrations
locales/          Lingui i18n message catalogs (47+ locales)
```

### Key Config Files

- `vite.config.ts` — Vite + Nitro + TanStack Start + PWA + Tailwind + Lingui
- `drizzle.config.ts` — PostgreSQL dialect, schema at `./src/integrations/drizzle/schema.ts`
- `tsconfig.json` — ES2022, strict mode, path alias `@/*` → `./src/*`
- `lingui.config.ts` — i18n extraction and locale configuration
- `components.json` — shadcn CLI configuration

### API Architecture

- **oRPC API** (`/api/rpc/*`) — Type-safe RPC with routers for: `ai`, `auth`, `resume`, `storage`, `printer`, `jobs`, `statistics`, `flags`. Three procedure types: `publicProcedure`, `protectedProcedure`, `serverOnlyProcedure`.
- **Better Auth API** (`/api/auth/*`) — OAuth, session management, social provider callbacks.
- **MCP Server** (`/mcp/`) — Model Context Protocol with OAuth Bearer tokens and API key auth. Exposes resumes as resources and tools for resume CRUD.

## Infrastructure Services

Before running the dev server, Docker must be running with at least PostgreSQL. Start services via `compose.dev.yml`:

```bash
sudo dockerd &>/var/log/dockerd.log &
sudo docker compose -f compose.dev.yml up -d postgres browserless
```

- **PostgreSQL** (port 5432) — required. The app auto-runs Drizzle migrations on startup via a Nitro plugin.
- **Browserless** (port 4000) — required for PDF export. Maps container port 3000 to host port 4000.

## Environment Variables

Copy `.env.example` to `.env` if not present. Key notes for local dev:

- `APP_URL` — local dev server origin on port 3000.
- `PRINTER_APP_URL` — must use the Docker bridge gateway IP (not localhost) so the Browserless container can reach the app on the host. Get the IP with: `sudo docker network inspect reactive_resume_default --format '{{range .IPAM.Config}}{{.Gateway}}{{end}}'`
- `PRINTER_ENDPOINT` — websocket URL to Browserless on host port 4000 with token `1234567890`.
- `DATABASE_URL` — PostgreSQL connection using `postgres:postgres` credentials on localhost:5432.
- S3/Storage and SMTP vars can be left empty — the app falls back to local filesystem and console-logged emails.

## Common Commands

`vp` is the global CLI for Vite+. Do not use pnpm/npm/yarn directly — Vite+ wraps the underlying package manager.

| Task                       | Command                                                         |
| -------------------------- | --------------------------------------------------------------- |
| Install dependencies       | `vp install`                                                    |
| Dev server (port 3000)     | `vp dev`                                                        |
| Lint (Oxlint, type-aware)  | `vp lint --type-aware`                                          |
| Format (Oxfmt)             | `vp fmt`                                                        |
| Check (lint + fmt + types) | `vp check`                                                      |
| Typecheck                  | `pnpm typecheck` (uses tsgo)                                    |
| Run tests                  | `vp test`                                                       |
| DB migrations              | `pnpm db:generate` / `pnpm db:migrate` (auto-runs on dev start) |
| DB studio                  | `pnpm db:studio`                                                |
| i18n extraction            | `pnpm lingui:extract`                                           |
| Add a dependency           | `vp add <package>`                                              |
| Remove a dependency        | `vp remove <package>`                                           |
| One-off binary             | `vp dlx <package>`                                              |
| Build for production       | `vp build`                                                      |
| Preview production build   | `vp preview`                                                    |
| Start production server    | `pnpm start`                                                    |

## Vite+ Pitfalls

- **Do not use pnpm/npm/yarn directly** for package operations — use `vp add`, `vp remove`, `vp install`, etc.
- **Do not run `vp vitest` or `vp oxlint`** — they don't exist. Use `vp test` and `vp lint`.
- **Do not install Vitest, Oxlint, Oxfmt, or tsdown directly** — Vite+ bundles them.
- **Import from `vite-plus`**, not from `vite` or `vitest` directly (e.g., `import { defineConfig } from 'vite-plus'`).
- **Vite+ commands take precedence** over `package.json` scripts. If there's a naming conflict, use `vp run <script>`.
- **Use `vp dlx`** instead of `npx` or `pnpm dlx`.
- **Type-aware linting** works out of the box with `vp lint --type-aware` — no need to install `oxlint-tsgolint`.

## Gotchas

- The Docker daemon needs `fuse-overlayfs` storage driver and `iptables-legacy` in the cloud VM (nested container environment).
- `pnpm.onlyBuiltDependencies` in `package.json` controls which packages are allowed to run install scripts — no interactive `pnpm approve-builds` needed.
- Email verification is optional in dev — after signup, click "Continue" to skip.
- Vite and Nitro use beta/nightly builds. Occasional upstream issues may occur.

## Review Checklist for Agents

- [ ] Run `vp install` after pulling remote changes and before getting started.
- [ ] Run `vp check` and `vp test` to validate changes.

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
