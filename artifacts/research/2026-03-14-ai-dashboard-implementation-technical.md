# AI Dashboard Assistant: Technical Implementation Research

**Date:** 2026-03-14
**Status:** Research Complete
**Scope:** Evaluate three UI approaches for a v0-style AI assistant on the dashboard (/dashboard/resumes)

---

## Strategic Summary

We need a conversational AI assistant on the main dashboard that can build resumes from scratch, improve existing ones, tailor for job descriptions, and provide career coaching. The existing codebase already has a working AI chat in the builder (popover-based, single-resume scope, custom transport over ORPC, localStorage persistence). The dashboard assistant differs in three key ways: (1) it operates across multiple resumes, (2) it needs a larger, more prominent UI surface, and (3) it requires a broader tool set (create, duplicate, list, patch across resumes).

All three approaches are viable. The recommendation is **Approach 1 (prompt-kit)** for its balance of quality, compatibility, and control.

---

## Requirements

| Requirement | Priority | Notes |
|---|---|---|
| Streaming chat with tool calling | Must | Already proven via ORPC + `streamToEventIterator` |
| Multi-resume operations (list, create, duplicate, patch) | Must | Tools exist in MCP server, need ORPC equivalents |
| Tailwind CSS v4 compatibility | Must | Project uses Tailwind v4 |
| React 19 compatibility | Must | Project uses React 19 |
| Shadcn/Radix visual consistency | Must | Must match existing design system |
| Markdown rendering in responses | Should | Career advice responses need formatting |
| Prompt suggestions / quick actions | Should | "Build resume from LinkedIn", "Tailor for job posting" |
| Conversation persistence | Should | localStorage minimum, server-side preferred |
| File upload (job descriptions, PDFs) | Could | For "tailor to job" use case |
| Voice input | Won't | Not needed for v1 |

---

## Approach 1: prompt-kit + New ORPC Procedure

### How It Works

Install prompt-kit components via the shadcn CLI registry. Each component is copied into `src/components/ui/` as source code (not a runtime dependency), giving full control. Create a new `ai.dashboardChat` ORPC procedure with a broader tool set (list_resumes, create_resume, patch_resume, duplicate_resume, etc.). Wire up `useChat` with the same custom transport pattern already proven in `src/components/ai/chat.tsx`.

### Libraries & Tools

| Package | Version | Purpose |
|---|---|---|
| prompt-kit (via shadcn registry) | N/A (source copy) | Chat UI components |
| `@ai-sdk/react` | ^3.0.118 (already installed) | `useChat` hook |
| `ai` | ^6.0.116 (already installed) | Server-side `streamText`, tools |
| `use-stick-to-bottom` | latest | Auto-scroll (prompt-kit dependency) |
| `react-markdown` + `remark-gfm` | latest | Markdown in Message component |

**Installation:**
```bash
# Add prompt-kit components (each downloads source into src/components/ui/)
npx shadcn@latest add "https://prompt-kit.com/c/chat-container.json"
npx shadcn@latest add "https://prompt-kit.com/c/message.json"
npx shadcn@latest add "https://prompt-kit.com/c/prompt-input.json"
npx shadcn@latest add "https://prompt-kit.com/c/prompt-suggestion.json"
npx shadcn@latest add "https://prompt-kit.com/c/markdown.json"
npx shadcn@latest add "https://prompt-kit.com/c/loader.json"
npx shadcn@latest add "https://prompt-kit.com/c/tool.json"
npx shadcn@latest add "https://prompt-kit.com/c/scroll-button.json"
```

### Tailwind v4 Compatibility

**Confirmed compatible.** prompt-kit's own `package.json` uses `tailwindcss: ^4.1.7` and `@tailwindcss/postcss`. Since components are copied as source, any Tailwind class incompatibilities can be fixed in-place. No GitHub issues report Tailwind v4 problems.

### Streaming Integration

Uses the exact same pattern as the existing builder chat:

```ts
// Custom transport wrapping ORPC streaming
transport: {
  async sendMessages(options) {
    return eventIteratorToUnproxiedDataStream(
      await client.ai.dashboardChat(
        { provider, model, apiKey, baseURL, messages: options.messages },
        { signal: options.abortSignal },
      ),
    );
  },
  reconnectToStream() { throw new Error("Unsupported"); },
},
```

No `DefaultChatTransport` needed -- the custom transport pattern already works and is more appropriate for ORPC backends. `DefaultChatTransport` is designed for standard HTTP fetch-based endpoints (e.g., Next.js `/api/chat`), which doesn't match our ORPC streaming architecture.

### Pros

- **Tailwind v4 native** -- developed against v4, no compatibility layer needed
- **Shadcn-native** -- components install into `src/components/ui/`, same pattern as existing components
- **Lightweight** -- only install what you need, no runtime dependency
- **Full control** -- source code is yours, customize icons (Phosphor vs Lucide), styling, behavior
- **Good component coverage** -- ChatContainer, Message, PromptInput, PromptSuggestion, Markdown, Tool, Loader, ScrollButton, Reasoning, ThinkingBar
- **Active project** -- 2.7k GitHub stars, 13 contributors, actively maintained
- **AI SDK integration** -- built with `@ai-sdk/react` in mind, examples use `useChat`

### Cons

- **Lucide icons by default** -- needs manual swap to Phosphor Icons to match project conventions
- **No releases published** -- components are installed via registry URL, not versioned npm package
- **Next.js focused examples** -- examples assume Next.js, need minor adaptation for TanStack Start
- **react-markdown dependency** -- adds react-markdown + remark-gfm if using Markdown component (small bundle cost)
- **Manual integration** -- must wire up useChat, transport, tool rendering yourself

### Best When

You want high-quality starting components with full customization control, and you're comfortable adapting examples from Next.js to TanStack Start.

### Complexity

**Medium.** The hardest parts (ORPC streaming, custom transport, tool calling) are already solved in the existing chat.tsx. The main work is: install components, adapt styling, build the dashboard-specific ORPC procedure with multi-resume tools, and compose the UI.

---

## Approach 2: AI SDK Elements

### How It Works

AI SDK Elements (elements.ai-sdk.dev) is Vercel's official component library for AI applications, built on shadcn/ui. It provides 29+ components across chatbot, code, voice, and workflow categories. Components install via a dedicated CLI or shadcn CLI into your project as source code.

### Libraries & Tools

| Package | Version | Purpose |
|---|---|---|
| AI SDK Elements (via CLI) | N/A (source copy) | Chat UI components |
| `@ai-sdk/react` | ^3.0.118 (already installed) | `useChat` hook |
| `ai` | ^6.0.116 (already installed) | Server-side streaming |

**Installation:**
```bash
# Dedicated CLI (recommended)
npx ai-elements@latest init
npx ai-elements@latest add conversation
npx ai-elements@latest add prompt-input
npx ai-elements@latest add message
npx ai-elements@latest add suggestion
npx ai-elements@latest add tool
npx ai-elements@latest add reasoning
```

### Tailwind v4 Compatibility

**Confirmed compatible.** Documentation explicitly states: "Tailwind CSS 4" is a requirement. Built targeting React 19 (no `forwardRef`).

### Streaming Integration

Deep AI SDK integration means components are designed to work directly with `useChat` message parts, tool invocations, and streaming states. Same custom transport pattern applies for ORPC.

### Pros

- **Official Vercel product** -- tightest possible integration with AI SDK
- **Most comprehensive** -- 29+ components including workflow canvas, voice, code blocks, JSX preview
- **Deep type safety** -- components understand AI SDK message part types natively
- **Tailwind v4 + React 19** -- explicitly targets both
- **Built on shadcn/ui** -- same conventions, theme-compatible

### Cons

- **Next.js focused** -- documentation and examples assume Next.js; integration guidance for other frameworks is sparse
- **Heavier** -- many components we don't need (voice, workflow canvas, code IDE components)
- **Newer / less community validation** -- less community feedback compared to prompt-kit
- **Opinionated structure** -- components install to `@/components/ai-elements/`, different from our `@/components/ui/` convention
- **CLI dependency** -- dedicated `ai-elements` CLI adds another tool to the chain
- **Unknown npm package status** -- no public npm package found; may be registry-only distribution
- **Lucide icons** -- same icon swap issue as prompt-kit

### Best When

You want the most comprehensive AI UI library and plan to use advanced features like workflow canvases, voice input, or code sandboxes in the future.

### Complexity

**Medium-High.** Installation is straightforward, but adapting from Next.js assumptions to TanStack Start, moving components to match our directory structure, and ensuring the dedicated CLI works correctly with our setup adds friction. The component set is larger than needed, which can complicate maintenance.

---

## Approach 3: Custom Build with Shadcn Primitives

### How It Works

Build all chat UI components from scratch using existing shadcn primitives already in the project: `InputGroup`, `InputGroupTextarea`, `ScrollArea`, `Avatar`, `Button`, `Tooltip`, `Popover`/`Sheet`. No external AI-specific component libraries.

### Libraries & Tools

| Package | Version | Purpose |
|---|---|---|
| Existing shadcn components | Already installed | All UI primitives |
| `@ai-sdk/react` | ^3.0.118 (already installed) | `useChat` hook |
| `react-markdown` + `remark-gfm` | latest | Markdown rendering (new dep) |

**Installation:**
```bash
# Only need markdown rendering
pnpm add react-markdown remark-gfm
```

### Existing Primitives Available

Components already in `src/components/ui/`:
- `InputGroup` + `InputGroupTextarea` -- for the prompt input area
- `ScrollArea` -- for the message list (already used in existing chat)
- `Avatar` -- for user/AI message avatars
- `Button` -- send, stop, clear actions
- `Tooltip` -- action tooltips
- `Sheet` or `Popover` -- container options
- `Skeleton` -- loading states
- `Badge` -- tool status indicators

### Streaming Integration

Same custom transport pattern. Reference implementation already exists in `src/components/ai/chat.tsx`.

### Pros

- **Zero new dependencies** -- only react-markdown if needed
- **Perfect design consistency** -- uses exact same components as rest of app
- **Maximum control** -- every pixel, every interaction is custom
- **No adaptation overhead** -- no need to swap icons, restructure directories, or adapt framework assumptions
- **Existing reference** -- `chat.tsx` is already a working custom chat implementation (408 lines)

### Cons

- **Most development time** -- must build ChatContainer with auto-scroll, Message bubbles with markdown, PromptInput with auto-resize, tool result rendering, suggestion chips, loading states from scratch
- **Auto-scroll is non-trivial** -- the "stick to bottom" behavior (scroll when at bottom, don't when user scrolls up) requires careful implementation or `use-stick-to-bottom` library
- **No community reference** -- no examples to follow for best patterns
- **Maintenance burden** -- all chat UI is custom code that must be maintained internally
- **Reinventing the wheel** -- prompt-kit and Elements solve these exact problems

### Best When

You have very specific design requirements that no library satisfies, or the project's UI must be radically different from standard chat patterns.

### Complexity

**High.** Estimated 800-1200 lines of component code for feature parity with what prompt-kit provides out of the box. The existing `chat.tsx` (408 lines) covers only the basics and would need significant expansion for the dashboard assistant use case.

---

## Comparison Table

| Criterion | Approach 1: prompt-kit | Approach 2: AI SDK Elements | Approach 3: Custom Build |
|---|---|---|---|
| **Tailwind v4** | Native (v4.1.7) | Native (v4 required) | N/A (existing) |
| **React 19** | Required (^19.0.0) | Required (no forwardRef) | N/A (existing) |
| **Shadcn compatible** | Yes (same registry) | Yes (shadcn-based) | Yes (IS shadcn) |
| **Component count** | ~15 relevant | 29+ (many irrelevant) | Build from scratch |
| **ORPC streaming** | Custom transport (proven) | Custom transport (proven) | Custom transport (proven) |
| **Installation** | shadcn CLI (familiar) | Dedicated CLI (new) | N/A |
| **Icon system** | Lucide (swap needed) | Lucide (swap needed) | Phosphor (native) |
| **Directory convention** | `src/components/ui/` | `@/components/ai-elements/` | `src/components/ui/` |
| **GitHub stars** | 2.7k | N/A (Vercel product) | N/A |
| **Framework assumptions** | Next.js (minor adaptation) | Next.js (moderate adaptation) | None |
| **Bundle impact** | Small (per-component) | Small (per-component) | Minimal |
| **Dev time estimate** | 2-3 days | 3-4 days | 5-7 days |
| **Maintenance burden** | Low (source owned) | Low (source owned) | High (all custom) |
| **Future extensibility** | Good | Excellent | Good |

---

## Recommendation

**Approach 1: prompt-kit** is the recommended path.

**Rationale:**
1. **Proven Tailwind v4 + React 19 compatibility** -- the library itself runs on these versions
2. **Shadcn-native installation** -- uses the same `npx shadcn` CLI, components land in `src/components/ui/`, consistent with project conventions
3. **Right-sized** -- provides exactly the components needed (ChatContainer, Message, PromptInput, PromptSuggestion, Tool, Markdown) without bloat
4. **Existing pattern reuse** -- the ORPC streaming transport, tool calling, and message handling patterns from `chat.tsx` transfer directly
5. **Full ownership** -- components are source code, not a runtime dependency. Swap Lucide for Phosphor, adjust styling, add i18n -- all straightforward
6. **Active community** -- 2.7k stars, actively maintained, good documentation

**What to watch for:**
- Swap Lucide icon imports to Phosphor equivalents after installation
- The `use-stick-to-bottom` dependency is small but new to the project
- Add `react-markdown` + `remark-gfm` for the Markdown component (or use marked which is already used elsewhere if present)

---

## Implementation Context

### Transport Decision: Custom ORPC Transport (Not DefaultChatTransport)

**Do not use `DefaultChatTransport` or `TextStreamChatTransport`.** These are designed for standard HTTP fetch-based endpoints (Next.js `/api/chat` pattern). Our architecture uses ORPC with `streamToEventIterator` / `eventIteratorToUnproxiedDataStream`, which requires a custom transport object.

The existing pattern in `chat.tsx` is correct and should be reused:

```ts
transport: {
  async sendMessages(options) {
    return eventIteratorToUnproxiedDataStream(
      await client.ai.dashboardChat(payload, { signal: options.abortSignal }),
    );
  },
  reconnectToStream() { throw new Error("Unsupported"); },
}
```

`TextStreamChatTransport` would lose tool calls, usage info, and finish reasons -- all critical for our use case.

### System Prompt Architecture

The dashboard assistant needs a **dual-mode system prompt** that covers both career coaching and resume editing:

**Career Coach mode:** General advice, strategy, interview prep -- no tool calls needed, pure text responses. Triggered by questions about career direction, interview tips, salary negotiation, etc.

**Resume Editor mode:** Multi-resume operations via tools. Triggered by requests to create, modify, improve, or tailor resumes. Needs the full list of available resumes in context.

Recommended structure:
```
1. Role definition (career coach + resume specialist)
2. Available tools and when to use them
3. List of user's resumes (fetched at conversation start via ORPC)
4. Resume data structure reference (same as existing chat-system.md)
5. Career coaching guidelines (no fabrication, evidence-based advice)
6. Behavioral rules (confirm destructive actions, minimal patches, etc.)
```

### Conversation Persistence Strategy

**Phase 1: localStorage (matches existing pattern)**
- Key: `dashboard-chat-messages` (no resume ID scoping needed -- it's a single conversation)
- Same `loadStoredMessages` / `saveStoredMessages` pattern from `chat.tsx`
- Sufficient for MVP

**Phase 2: Server-side persistence (future)**
- Add `chatConversation` table to Drizzle schema (id, userId, messages JSONB, createdAt, updatedAt)
- New ORPC procedures: `ai.saveConversation`, `ai.loadConversation`, `ai.listConversations`
- Enable conversation history across devices
- Use AI SDK's `onFinish` callback pattern for server-side saving

### Tool Set for Dashboard Chat

Reuse tool implementations from the MCP server (`src/routes/mcp/-helpers/tools.ts`), adapted as AI SDK `tool()` definitions:

| Tool | Purpose | Input |
|---|---|---|
| `list_resumes` | Show user's resumes | `{ tags?: string[], sort?: string }` |
| `get_resume` | Fetch full resume data | `{ id: string }` |
| `create_resume` | Create new empty resume | `{ name, slug, tags?, withSampleData? }` |
| `duplicate_resume` | Clone existing resume | `{ id, name, slug, tags? }` |
| `patch_resume` | Apply JSON Patch operations | `{ id, operations[] }` |
| `delete_resume` | Delete a resume | `{ id }` (requires confirmation) |
| `export_resume_pdf` | Generate PDF | `{ id }` |
| `get_resume_screenshot` | Get visual preview URL | `{ id }` |

### Key Files to Create/Modify

```
# New files
src/components/ai/dashboard-chat.tsx          # Main dashboard chat component
src/integrations/ai/prompts/dashboard-system.md  # Dashboard-specific system prompt
src/integrations/ai/tools/dashboard-tools.ts  # Tool definitions for multi-resume ops

# Modified files
src/integrations/orpc/router/ai.ts            # Add dashboardChat procedure
src/integrations/orpc/services/ai.ts          # Add dashboardChat service function
src/routes/dashboard/resumes/index.tsx        # Mount the chat component

# Installed via prompt-kit (new UI components)
src/components/ui/chat-container.tsx
src/components/ui/message.tsx
src/components/ui/prompt-input.tsx
src/components/ui/prompt-suggestion.tsx
src/components/ui/markdown.tsx
src/components/ui/loader.tsx
```

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│  Dashboard Page (/dashboard/resumes)                     │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │  DashboardChat (Sheet/Panel, right side)          │   │
│  │                                                    │   │
│  │  ┌─ ChatContainer (auto-scroll) ──────────────┐  │   │
│  │  │  Message[]                                   │  │   │
│  │  │    - Text (markdown rendered)                │  │   │
│  │  │    - ToolBadge (list/create/patch results)   │  │   │
│  │  │    - Reasoning (thinking indicator)          │  │   │
│  │  └─────────────────────────────────────────────┘  │   │
│  │                                                    │   │
│  │  ┌─ PromptSuggestion[] ────────────────────────┐  │   │
│  │  │  "Build resume from scratch"                 │  │   │
│  │  │  "Tailor resume for job posting"             │  │   │
│  │  │  "Review my resume"                          │  │   │
│  │  └─────────────────────────────────────────────┘  │   │
│  │                                                    │   │
│  │  ┌─ PromptInput ──────────────────────────────┐  │   │
│  │  │  [textarea] [send/stop]                     │  │   │
│  │  └─────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
│  ┌─ useChat() ──────────────────────────────────────┐   │
│  │  transport: custom ORPC transport                 │   │
│  │  → client.ai.dashboardChat()                      │   │
│  │  → eventIteratorToUnproxiedDataStream()            │   │
│  └───────────────────────────────────────────────────┘   │
│                                                          │
└────────────────────────────┬────────────────────────────┘
                             │ ORPC streaming
                             ▼
┌─────────────────────────────────────────────────────────┐
│  Server: ai.dashboardChat procedure                      │
│                                                          │
│  streamText({                                            │
│    model,                                                │
│    system: dashboardSystemPrompt,                        │
│    messages,                                             │
│    tools: {                                              │
│      list_resumes, get_resume, create_resume,            │
│      duplicate_resume, patch_resume, delete_resume,      │
│      export_resume_pdf, get_resume_screenshot            │
│    },                                                    │
│    stopWhen: stepCountIs(5),                             │
│  })                                                      │
│  → streamToEventIterator(result.toUIMessageStream())     │
└─────────────────────────────────────────────────────────┘
```

---

## Sources

| Source | URL | Accessed |
|---|---|---|
| prompt-kit GitHub | https://github.com/ibelick/prompt-kit | 2026-03-14 |
| prompt-kit Docs | https://prompt-kit.com | 2026-03-14 |
| prompt-kit package.json | https://raw.githubusercontent.com/ibelick/prompt-kit/main/package.json | 2026-03-14 |
| AI SDK Elements Docs | https://elements.ai-sdk.dev | 2026-03-14 |
| AI SDK Elements Installation | https://elements.ai-sdk.dev/docs | 2026-03-14 |
| AI SDK useChat Docs | https://ai-sdk.dev/docs/ai-sdk-ui/chatbot | 2026-03-14 |
| AI SDK Tool Usage | https://ai-sdk.dev/docs/ai-sdk-ui/chatbot-tool-usage | 2026-03-14 |
| AI SDK Message Persistence | https://ai-sdk.dev/docs/ai-sdk-ui/chatbot-message-persistence | 2026-03-14 |
| AI SDK useChat Reference | https://ai-sdk.dev/docs/reference/ai-sdk-ui/use-chat | 2026-03-14 |
| Existing chat.tsx | src/components/ai/chat.tsx | 2026-03-14 |
| Existing ai.ts service | src/integrations/orpc/services/ai.ts | 2026-03-14 |
| Existing MCP tools | src/routes/mcp/-helpers/tools.ts | 2026-03-14 |
| Existing AI router | src/integrations/orpc/router/ai.ts | 2026-03-14 |
