# Deep Dive: AI Assistant Dashboard for CV Builder

## Strategic Summary

The project already has a sophisticated AI integration (Vercel AI SDK v6, `useChat`, JSON Patch-based resume editing, 5 providers, MCP server) but the chat UI is buried in the builder dock. By adding an AI assistant to the main dashboard — similar to v0.dev's landing experience — we can let users start building resumes conversationally from the moment they log in. The key building blocks (AI SDK hooks, ORPC procedures, prompt engineering) already exist; what's needed is a new dashboard UI layer using prompt-kit or AI SDK Elements components.

## Key Questions This Research Answers

- What AI features already exist in the project?
- What are AI SDK UI Elements and how do they work?
- What Shadcn-compatible AI components exist?
- How can we build a v0-style AI assistant dashboard?
- What's the recommended component stack?

---

## Current AI Implementation (What We Already Have)

### Features

| Feature | Location | Description |
|---------|----------|-------------|
| AI Chat | `src/components/ai/chat.tsx` (408 lines) | `useChat` hook, JSON Patch tool, localStorage persistence |
| PDF Parsing | ORPC `ai.parsePdf` | Vision-based PDF → ResumeData extraction |
| DOCX Parsing | ORPC `ai.parseDocx` | Word document → ResumeData extraction |
| Connection Test | ORPC `ai.testConnection` | Provider connectivity validation |
| AI Settings | `/dashboard/settings/ai` | Provider/model/API key config with test button |
| MCP Server | `/mcp/` endpoint | Full CRUD + PDF export + prompts for external LLMs |

### Packages Already Installed

```
"ai": "^6.0.116"
"@ai-sdk/react": "^3.0.118"
"@ai-sdk/openai": "^3.0.41"
"@ai-sdk/anthropic": "^3.0.58"
"@ai-sdk/google": "^3.0.43"
"ai-sdk-ollama": "^3.8.0"
"@modelcontextprotocol/sdk": "^1.27.1"
```

### Supported Providers

OpenAI, Anthropic Claude, Google Gemini, Ollama, Vercel AI Gateway — all configurable per-user with local storage persistence.

### AI Chat Architecture (Current)

```
User types in AI Chat (builder dock)
  → useChat hook → client.ai.chat() ORPC call
  → aiService.chat() → streamText() with patch_resume tool
  → LLM responds with JSON Patch operations
  → Client validates & applies patches via applyResumePatches()
  → Resume state updates → preview re-renders
```

### Key Prompts

- `chat-system.md` — Resume expert persona, teaches JSON Patch operations & resume data structure
- `pdf-parser-system.md` — Anti-hallucination rules for PDF extraction
- `docx-parser-system.md` — Word document-specific parsing rules

### MCP Server Tools

`list_resumes`, `get_resume`, `create_resume`, `duplicate_resume`, `patch_resume`, `delete_resume`, `lock/unlock_resume`, `export_resume_pdf`, `get_resume_screenshot`, `get_resume_statistics`

### MCP Prompts

`build_resume`, `improve_resume`, `tailor_resume`, `review_resume` — comprehensive guided workflows.

---

## AI SDK UI Overview

### Core Hooks

| Hook | Purpose | Package |
|------|---------|---------|
| `useChat` | Multi-turn chat with streaming, tools, file attachments | `@ai-sdk/react` |
| `useCompletion` | Single-turn text completion | `@ai-sdk/react` |
| `useObject` | Streaming structured JSON against Zod schema | `@ai-sdk/react` |
| `useAssistant` | OpenAI Assistants API (threads/runs) | `@ai-sdk/react` |

### useChat Transport Pattern (v4+)

```tsx
import { useChat } from "@ai-sdk/react";
import { DefaultChatTransport } from "ai";

const { messages, sendMessage, status, stop, regenerate, error } = useChat({
  transport: new DefaultChatTransport({ api: "/api/chat" }),
  experimental_throttle: 50,
});
```

**Transport modes:**
- `DefaultChatTransport` — Full protocol (tool calls, usage stats, finish reasons)
- `TextStreamChatTransport` — Plain text streams only
- `DirectChatTransport` — No HTTP, runs agent in browser

**Status values:** `ready` | `submitted` | `streaming` | `error`

### Message Parts (Rendering)

```tsx
message.parts.map((part) => {
  if (part.type === "text") return <Markdown>{part.text}</Markdown>;
  if (part.type === "reasoning") return <Reasoning>{part.text}</Reasoning>;
  if (part.type === "file") return <img src={part.url} />;
  if (part.type === "source-url") return <SourceLink url={part.url} />;
});
```

### Server-Side Pattern (TanStack Start)

```ts
import { streamText, convertToModelMessages } from "ai";

export async function POST(req: Request) {
  const { messages } = await req.json();
  const result = streamText({
    model: openai("gpt-4o"),
    system: "You are a helpful assistant.",
    messages: convertToModelMessages(messages),
  });
  return result.toUIMessageStreamResponse();
}
```

---

## AI SDK Elements (`elements.ai-sdk.dev`)

A React component library built on shadcn/ui with deep AI SDK integration. Installed individually via CLI (shadcn-style).

### Component Categories

| Category | Components |
|----------|-----------|
| **Chatbot** | Message, Conversation, PromptInput, Reasoning, Suggestion, ModelSelector, ChainOfThought, Attachments, Sources |
| **Code/IDE** | CodeBlock, Terminal, FileTree, Artifact, Sandbox, StackTrace, TestResults, WebPreview |
| **Voice** | SpeechInput, Transcription, AudioPlayer, MicSelector, VoiceSelector |
| **Workflow** | Canvas, Node, Edge, Connection, Controls (flow-based UIs) |

---

## Shadcn AI Components Ecosystem

### Official shadcn/ui (No AI-Specific Components)

Useful primitives for building AI UIs:
- `InputGroup` + `InputGroupTextarea` — prompt bar base
- `ScrollArea` — message list container
- `Skeleton` / `Spinner` — streaming/loading states
- `Avatar` — user/AI icons
- `Sheet` / `Dialog` — chat panel overlays
- `Command` — slash-command palette in prompt input

### prompt-kit (`prompt-kit.com`) — Best Shadcn-Native AI Library

Installed via: `npx shadcn add "https://prompt-kit.com/c/<component>.json"`

| Component | Purpose |
|-----------|---------|
| `PromptInput` / `PromptInputTextarea` / `PromptInputActions` | Auto-resizing prompt bar with action slots |
| `Message` | Chat message display |
| `ChatContainer` | Full chat layout |
| `CodeBlock` | Syntax-highlighted code |
| `Markdown` | Full markdown renderer |
| `Loader` / `TextShimmer` / `ThinkingBar` | Streaming/thinking states |
| `FileUpload` | File attachment UI |
| `ScrollButton` | Auto-scroll-to-bottom |
| `PromptSuggestion` | Suggested prompt chips |
| `Reasoning` | Chain-of-thought display |
| `Source` | Source citations |
| `Steps` | Step-by-step process |
| `Tool` | Tool/function-call results |
| `FeedbackBar` | Thumbs up/down |

### shadcn-chat (`github.com/jakobhoeg/shadcn-chat`)

Messaging-focused (WhatsApp/Slack style). Components: `Chat`, `ChatHeader`, `ChatMessages`, `ChatToolbar`, `ChatEvent`, `PrimaryMessage`, `AdditionalMessage`, `DateItem`.

---

## Dashboard AI Assistant — Design Concept

### Vision (v0-Style Experience)

The main dashboard (`/dashboard/resumes`) gets a prominent AI assistant area:

```
┌─────────────────────────────────────────────────┐
│  Dashboard Header                               │
├─────────────────────────────────────────────────┤
│                                                 │
│         "What do you want to create?"           │
│                                                 │
│  ┌───────────────────────────────────────────┐  │
│  │  Ask AI to build your resume...           │  │
│  │                                           │  │
│  │  [+ Attach] [Model ▾]        [Send →]    │  │
│  └───────────────────────────────────────────┘  │
│                                                 │
│  ┌─────────┐ ┌─────────┐ ┌──────────────────┐  │
│  │ Build   │ │ Improve │ │ Tailor for job   │  │
│  │ from    │ │ existing│ │ description      │  │
│  │ scratch │ │ resume  │ │                  │  │
│  └─────────┘ └─────────┘ └──────────────────┘  │
│                                                 │
│  ── Your Resumes ────────────────────────────── │
│  [Resume Card] [Resume Card] [+ New Resume]     │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Suggested Flows

1. **"Build from scratch"** — AI asks guided questions (name, role, experience), creates resume via `create_resume` + `patch_resume`
2. **"Improve existing"** — User selects a resume, AI analyzes and suggests improvements
3. **"Tailor for job"** — User pastes job description, AI optimizes resume for ATS matching
4. **Free-form prompt** — Open-ended chat that can create, modify, or analyze resumes

### Implementation Architecture

```
Dashboard AI Component
  ├── PromptInput (prompt-kit or AI SDK Elements)
  ├── PromptSuggestion chips (pre-built flows)
  ├── Chat/Message area (expandable on interaction)
  │   ├── Message rendering (Markdown, CodeBlock)
  │   ├── Tool call results (resume created/modified)
  │   └── Streaming indicators (Loader, ThinkingBar)
  └── Backend
      ├── New ORPC procedure: ai.dashboardChat
      ├── Tools: create_resume, list_resumes, patch_resume, etc.
      └── System prompt: dashboard-assistant context
```

### Key Differences from Current AI Chat

| Aspect | Current (Builder) | New (Dashboard) |
|--------|-------------------|-----------------|
| Context | Single resume being edited | All user's resumes |
| Tools | `patch_resume` only | `create_resume`, `list_resumes`, `patch_resume`, `duplicate_resume` |
| Entry point | Builder dock (small panel) | Dashboard center stage |
| Purpose | Edit current resume | Guide overall CV strategy |
| Prompt | Resume editing expert | Career coach / CV strategist |

---

## Recommended Component Stack

### Option A: prompt-kit (Recommended)

Best fit because:
- Shadcn-native (installs into `src/components/ui/`)
- Matches existing project patterns exactly
- Auto-resizing textarea, action slots, suggestion chips
- Markdown + CodeBlock rendering included
- Streaming state components (Loader, ThinkingBar)

```bash
npx shadcn add "https://prompt-kit.com/c/prompt-input.json"
npx shadcn add "https://prompt-kit.com/c/message.json"
npx shadcn add "https://prompt-kit.com/c/markdown.json"
npx shadcn add "https://prompt-kit.com/c/loader.json"
npx shadcn add "https://prompt-kit.com/c/prompt-suggestion.json"
npx shadcn add "https://prompt-kit.com/c/scroll-button.json"
```

### Option B: AI SDK Elements

More comprehensive but heavier. Better if we want voice input, workflow canvas, or IDE-like features later.

### Option C: Build from scratch with shadcn primitives

`InputGroup` + `InputGroupTextarea` + custom message components. More work but maximum control.

---

## Patterns & Best Practices

- **Throttle streaming renders**: Use `experimental_throttle: 50` in `useChat` to avoid excessive re-renders
- **Persist chat history**: Current implementation uses localStorage per resume — dashboard chat should do the same
- **Tool call tracking**: Use `processedToolCallIds` set to prevent re-applying patches (already implemented)
- **Anti-hallucination**: Current PDF/DOCX parsers have strong guardrails — dashboard prompts should follow same pattern
- **Progressive disclosure**: Start with prompt input + suggestions, expand to full chat on first interaction
- **Error recovery**: Show retry button on streaming errors (`status === "error"` + `regenerate()`)

## Limitations & Edge Cases

- **No test framework**: All changes must be validated via `pnpm lint` + `pnpm typecheck` only
- **AI requires user config**: Users must configure their own API key in settings before AI features work
- **Streaming through ORPC**: Current chat uses `eventIteratorToUnproxiedDataStream` — dashboard chat needs same pattern
- **Token limits**: Long conversations can exceed context windows — implement conversation trimming
- **Tool call ordering**: Multiple concurrent tool calls can conflict — serialize resume mutations
- **Rate limiting**: No rate limiting on AI endpoints currently — consider adding for dashboard usage

## Current State & Trends

- **AI SDK v6** is latest — project is already on it
- **Transport pattern** (DefaultChatTransport) is the new recommended approach over direct API calls
- **prompt-kit** is gaining traction as the shadcn-native AI component library
- **MCP** integration is already implemented — positions the project well for LLM ecosystem interop
- **Agentic patterns** (multi-step tool calling) are the direction — dashboard assistant could be the first agentic feature

## Key Takeaways

1. **The project already has 80% of the infrastructure needed** — AI SDK, ORPC procedures, streaming, tool calling, and MCP are all in place. The dashboard AI assistant is primarily a UI/UX addition.

2. **prompt-kit is the ideal component library** — it's shadcn-native, installs into the existing component structure, and provides exactly the components needed (PromptInput, Message, Markdown, Loader, PromptSuggestion).

3. **The main architectural work is a new ORPC procedure** (`ai.dashboardChat`) with a broader tool set (create/list/patch resumes) and a "career coach" system prompt — distinct from the existing single-resume editor chat.

## Remaining Unknowns

- [ ] How does prompt-kit handle Tailwind CSS v4? (project uses v4, prompt-kit may target v3)
- [ ] Can we reuse the MCP prompts (build_resume, improve_resume, tailor_resume) as dashboard flows?
- [ ] What's the ideal UX for transitioning from dashboard chat to builder? (redirect after creation?)
- [ ] Should the dashboard assistant support file uploads (PDF/DOCX) inline, or keep the import dialog?
- [ ] How do we handle conversation persistence across sessions for dashboard chat?
- [ ] Token usage tracking/budgeting for the dashboard assistant?

## Implementation Context

### Application
- **When to use**: Building the AI assistant dashboard, adding conversational resume creation
- **When not to use**: Simple CRUD operations, non-AI resume editing
- **Prerequisites**: User must have AI provider configured in settings

### Technical
- **Libraries**: `ai`, `@ai-sdk/react` (already installed), prompt-kit components (to install)
- **Patterns**: `useChat` + `DefaultChatTransport` for streaming, ORPC procedures for backend, Zustand for local state
- **Gotchas**: ORPC streaming uses custom `eventIteratorToUnproxiedDataStream` wrapper; prompt-kit may need Tailwind v4 compatibility patches

### Integration
- **Works with**: Existing AI store, ORPC router, Better Auth, MCP tools/prompts
- **Conflicts with**: Nothing — additive feature
- **Alternatives**: Full custom chat UI, AI SDK Elements (heavier), shadcn-chat (messaging-focused)

**Next Action:** Plan the implementation — create the dashboard AI component, new ORPC procedure, and install prompt-kit components.

## Sources

- Project codebase exploration (March 2026)
- AI SDK UI docs: https://ai-sdk.dev/docs/ai-sdk-ui/overview
- AI SDK Elements: https://elements.ai-sdk.dev/
- AI SDK Elements Components: https://elements.ai-sdk.dev/components/
- prompt-kit: https://prompt-kit.com
- shadcn-chat: https://github.com/jakobhoeg/shadcn-chat
- shadcn/ui: https://ui.shadcn.com/docs/components
