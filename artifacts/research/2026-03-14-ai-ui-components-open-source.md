# AI UI Component Libraries for Reactive Resume Fork

**Date:** 2026-03-14
**Status:** Research Complete
**Decision:** Pending

---

## Strategic Summary

We need AI chat UI components for a dashboard assistant in our Reactive Resume fork. The project uses React 19, Shadcn/Radix UI, Tailwind CSS v4, AI SDK v6, and the `useChat` hook from `@ai-sdk/react`. The ideal solution installs into our existing `src/components/ui/` directory following the shadcn pattern (copy-paste ownership, not an npm dependency), supports Tailwind v4 natively, and provides streaming-aware components out of the box.

Three strong contenders emerged: **AI Elements** (Vercel's official library, tightest AI SDK integration), **prompt-kit** (lightweight shadcn-native blocks), and **assistant-ui** (most feature-rich, Y Combinator-backed). The rest are either too heavy (CopilotKit), not AI-focused (chatscope), stale (shadcn-chat), or a reference app rather than a library (vercel/chatbot).

---

## What We Need

| Requirement | Priority | Notes |
|---|---|---|
| Prompt input with auto-resize | Must | Textarea with submit, attachments |
| Message list / thread | Must | User + assistant messages, streaming |
| Markdown rendering | Must | AI responses are markdown |
| Streaming indicator | Must | Shimmer / typing animation |
| Suggestion chips | Should | Quick-action prompts |
| File upload UI | Should | Drag-and-drop, preview |
| Code block with syntax highlighting | Should | AI may return code |
| Reasoning / chain-of-thought display | Nice | Expandable thinking panel |
| Tool call rendering | Nice | Show tool invocations inline |

**Hard constraints:**
- MIT or Apache-2.0 license
- React 19 compatible
- Tailwind CSS v4 compatible (critical -- project uses v4)
- Must work with Vite 8 (not Next.js-only)
- Should follow shadcn copy-paste pattern (own the code)

---

## Options Found

### 1. prompt-kit

| Field | Detail |
|---|---|
| **Repo** | [ibelick/prompt-kit](https://github.com/ibelick/prompt-kit) |
| **Website** | [prompt-kit.com](https://www.prompt-kit.com/) |
| **What it does** | Core building blocks for AI apps -- shadcn-native components for chat interfaces |
| **Stars** | 2,689 |
| **Forks** | 145 |
| **Contributors** | 13 |
| **Open issues** | 7 |
| **Last commit** | 2026-03-12 |
| **Created** | 2025-02-10 |
| **License** | MIT |
| **Language** | TypeScript |
| **React 19** | Yes (requires React 19+) |
| **Tailwind v4** | Yes (shadcn-based, follows shadcn v4 conventions) |
| **Shadcn pattern** | Yes -- installs via `npx shadcn@latest add "https://prompt-kit.com/c/[COMPONENT].json"` into your components dir |
| **AI SDK integration** | Documented examples for Vercel AI SDK, OpenAI SDK |
| **Bundle size** | Zero runtime -- components are copied into your project |

**Components:** Prompt Input, Message, Markdown, Chat Container, Code Block, File Upload, Prompt Suggestion, Reasoning, Chain of Thought, Thinking Bar, Text Shimmer, Loader, Scroll Button, Source, Steps, System Message, Tool, Feedback Bar, Image.

**Fits our need:** Excellent. Covers all "Must" and "Should" requirements. Shadcn-native means it drops into our existing `src/components/ui/` pattern perfectly. Lightweight with no runtime dependency.

**Concerns:**
- Young project (1 year old), small contributor base (13)
- Less battle-tested than assistant-ui
- No thread management or conversation persistence (UI-only)

---

### 2. AI Elements (Vercel)

| Field | Detail |
|---|---|
| **Repo** | [vercel/ai-elements](https://github.com/vercel/ai-elements) |
| **Website** | [elements.ai-sdk.dev](https://elements.ai-sdk.dev/) |
| **What it does** | Official Vercel component library for AI SDK, built on shadcn/ui |
| **Stars** | 1,789 |
| **Forks** | 220 |
| **Contributors** | 54 |
| **Open issues** | 27 |
| **Last commit** | 2026-03-12 |
| **Created** | 2025-08-15 |
| **License** | Apache-2.0 |
| **Language** | TypeScript |
| **React 19** | Yes (built for modern React) |
| **Tailwind v4** | Yes (globals.css imports Tailwind v4, shadcn v4 base styles) |
| **Shadcn pattern** | Yes -- registry-based, installs to `@/components/ai-elements/` |
| **AI SDK integration** | Native -- designed specifically for AI SDK |
| **NPM downloads** | ~24k/week |
| **Bundle size** | ~8.5 kB (registry components are copied, not bundled) |

**Components:** Massive library organized by category:
- **Chat:** Chatbot, Attachments, Chain of Thought, Checkpoint, Confirmation, Context, Conversation, Inline Citation, Message, Model Selector, Plan, Prompt Input, Queue, Reasoning, Shimmer, Sources, Suggestion, Task, Tool
- **Code:** Agent, Artifact, Code Block, Commit, Environment Variables, File Tree, JSX Preview, Sandbox, Schema Display, Snippet, Stack Trace, Terminal, Test Results, Web Preview
- **Voice:** Audio Player, Mic Selector, Persona, Speech Input, Transcription, Voice Selector
- **Workflow:** Canvas, Connection, Controls, Edge, Node, Panel, Toolbar

**Fits our need:** Excellent. The most comprehensive component library. Deepest AI SDK integration since it is Vercel's own. Covers every requirement and more.

**Concerns:**
- Youngest project (7 months old)
- License is Apache-2.0 (compatible but not MIT -- our project uses MIT-compatible deps)
- Primarily designed for Next.js ecosystem; Vite compatibility needs verification
- Large component surface area may include Next.js-specific patterns
- 27 open issues suggests some rough edges

---

### 3. assistant-ui

| Field | Detail |
|---|---|
| **Repo** | [assistant-ui/assistant-ui](https://github.com/assistant-ui/assistant-ui) |
| **Website** | [assistant-ui.com](https://www.assistant-ui.com/) |
| **What it does** | Full-featured AI assistant UI library with composable Radix-style primitives |
| **Stars** | 8,830 |
| **Forks** | 921 |
| **Contributors** | 93 |
| **Open issues** | 81 |
| **Last commit** | 2026-03-14 (today) |
| **Created** | 2023-11-22 |
| **License** | MIT |
| **Language** | TypeScript |
| **React 19** | Yes (confirmed support) |
| **Tailwind v4** | Yes (confirmed, includes tw-shimmer Tailwind v4 plugin) |
| **Shadcn pattern** | Partial -- has shadcn-compatible styled components, but core is an npm package (`@assistant-ui/react`) |
| **AI SDK integration** | Yes, via `@assistant-ui/react-ai-sdk` adapter |
| **NPM downloads** | ~50k+/month (~12k/week) |
| **Latest version** | 0.12.17 |

**Components:** Thread, ThreadList, AssistantModal, AssistantSidebar, Composer (with attachments), Markdown renderer, syntax highlighting, LaTeX, ToolGroup, ToolFallback, generative UI, message branching/editing, speech-to-text, multi-agent rendering, human-in-the-loop approval.

**Fits our need:** Excellent. Most mature and feature-rich option. Radix-style composable primitives give fine-grained control. Y Combinator backed (W25). Largest community.

**Concerns:**
- **Not pure shadcn pattern** -- core is an npm dependency (`@assistant-ui/react`), not copy-paste components. This means version upgrades are dependency-driven, not code-owned.
- Heavier abstraction layer than prompt-kit or AI Elements
- 81 open issues (though proportional to project size)
- Still pre-1.0 (v0.12.x) -- API may change

---

### 4. shadcn-chat

| Field | Detail |
|---|---|
| **Repo** | [jakobhoeg/shadcn-chat](https://github.com/jakobhoeg/shadcn-chat) |
| **Website** | [shadcn-chat.vercel.app](https://shadcn-chat.vercel.app/) |
| **What it does** | CLI for adding chat UI components built on shadcn/ui |
| **Stars** | 1,608 |
| **Forks** | 145 |
| **Contributors** | 8 |
| **Open issues** | 9 |
| **Last commit** | 2025-08-12 (7 months ago) |
| **Created** | 2024-01-21 |
| **License** | MIT |
| **Language** | TypeScript |
| **React 19** | Unknown (no explicit confirmation) |
| **Tailwind v4** | Unknown (last update predates widespread v4 adoption) |
| **Shadcn pattern** | Yes -- CLI-based component installation |

**Components:** Chat container, message bubbles, toolbar, auto-scroll hook.

**Fits our need:** Partial. Messaging-focused (WhatsApp-style) rather than AI-assistant-style. Missing streaming indicators, markdown rendering, suggestion chips, reasoning display.

**Concerns:**
- **Stale** -- last commit 7 months ago, only 8 contributors
- Not AI-focused -- general chat UI
- Missing most AI-specific features (streaming, reasoning, tool calls)
- Tailwind v4 / React 19 compatibility unconfirmed

---

### 5. chatscope/chat-ui-kit-react

| Field | Detail |
|---|---|
| **Repo** | [chatscope/chat-ui-kit-react](https://github.com/chatscope/chat-ui-kit-react) |
| **Website** | [chatscope.io](https://chatscope.io/) |
| **What it does** | General-purpose chat UI toolkit |
| **Stars** | 1,725 |
| **Forks** | 150 |
| **Contributors** | 7 |
| **Open issues** | 59 |
| **Last commit** | 2025-05-15 (10 months ago) |
| **Created** | 2020-08-26 |
| **License** | MIT |
| **Language** | JavaScript |
| **React 19** | Yes (v2.1.1 added React 19 to peer deps) |
| **Tailwind v4** | No -- uses SCSS themes, not Tailwind |
| **Shadcn pattern** | No -- traditional npm package with own styling system |

**Components:** MessageList, Message, MessageInput, ConversationList, Sidebar, Avatar, TypingIndicator, ChatContainer.

**Fits our need:** Poor. General chat UI, not AI-specific. Uses SCSS (conflicts with our Tailwind v4 stack). No streaming support, no markdown rendering, no AI-specific features.

**Concerns:**
- **SCSS-based** -- fundamentally incompatible with our Tailwind-only approach
- 59 open issues with only 7 contributors suggests limited maintenance
- 10 months since last commit
- No AI-specific features (streaming, reasoning, tool calls)
- Written in JavaScript (our project is TypeScript-strict)

---

### 6. CopilotKit

| Field | Detail |
|---|---|
| **Repo** | [CopilotKit/CopilotKit](https://github.com/CopilotKit/CopilotKit) |
| **Website** | [copilotkit.ai](https://www.copilotkit.ai/) |
| **What it does** | Full-stack framework for building AI copilots with React/Angular |
| **Stars** | 29,370 |
| **Forks** | 3,820 |
| **Contributors** | 163 |
| **Open issues** | 581 |
| **Last commit** | 2026-03-14 (today) |
| **Created** | 2023-06-19 |
| **License** | MIT |
| **Language** | TypeScript |
| **React 19** | Yes |
| **Tailwind v4** | Likely (actively maintained) |
| **Shadcn pattern** | No -- full framework with own runtime |
| **NPM downloads** | ~126k/week (`@copilotkit/react-ui`) |
| **Package size** | 1.66 MB |

**Components:** CopilotChat, CopilotPopup, CopilotSidebar, CopilotTextarea, generative UI rendering via tool calls, headless UI primitives.

**Fits our need:** Overkill. CopilotKit is a full-stack agentic framework, not a component library. It includes its own runtime, backend integration layer (AG-UI protocol), and state management. We only need UI components.

**Concerns:**
- **Way too heavy** -- 1.66 MB package, full runtime framework
- Requires CopilotKit runtime/backend, not just AI SDK
- Would introduce a parallel state management system alongside our Zustand/TanStack Query setup
- 581 open issues
- Architectural overkill for a dashboard chat widget

---

### 7. vercel/chatbot (Reference Implementation)

| Field | Detail |
|---|---|
| **Repo** | [vercel/chatbot](https://github.com/vercel/chatbot) |
| **Website** | N/A (template, not library) |
| **What it does** | Production-ready Next.js chatbot template |
| **Stars** | 19,885 |
| **Forks** | 6,394 |
| **Contributors** | 74 |
| **Open issues** | 76 |
| **Last commit** | 2026-03-14 (today) |
| **Created** | 2023-05-19 |
| **License** | Custom (NOASSERTION on GitHub) |
| **Language** | TypeScript |

**Fits our need:** Not directly usable -- it is a Next.js application template, not a component library. However, it demonstrates patterns for chat UI with AI SDK that could inform our implementation. Many of its UI patterns are now formalized in AI Elements.

**Concerns:**
- Not a library -- cannot be installed
- Next.js-specific (RSC, Server Actions)
- License unclear (NOASSERTION)
- Useful only as a reference for patterns

---

## Comparison Table

| Criteria | prompt-kit | AI Elements | assistant-ui | shadcn-chat | chatscope | CopilotKit | vercel/chatbot |
|---|---|---|---|---|---|---|---|
| **Stars** | 2,689 | 1,789 | 8,830 | 1,608 | 1,725 | 29,370 | 19,885 |
| **Contributors** | 13 | 54 | 93 | 8 | 7 | 163 | 74 |
| **Last commit** | 2d ago | 2d ago | Today | 7mo ago | 10mo ago | Today | Today |
| **License** | MIT | Apache-2.0 | MIT | MIT | MIT | MIT | Unclear |
| **React 19** | Yes | Yes | Yes | Unknown | Yes | Yes | Yes |
| **Tailwind v4** | Yes | Yes | Yes | Unknown | No (SCSS) | Likely | N/A |
| **Shadcn pattern** | Yes (full) | Yes (full) | Partial (npm) | Yes | No | No | N/A |
| **Vite compatible** | Yes | Needs check | Yes | Yes | Yes | Yes | No (Next.js) |
| **AI SDK integration** | Documented | Native | Adapter pkg | None | None | Own runtime | Native |
| **Prompt input** | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| **Message list** | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| **Markdown** | Yes | Yes | Yes | No | No | Yes | Yes |
| **Streaming** | Yes | Yes | Yes | No | No | Yes | Yes |
| **Suggestions** | Yes | Yes | No | No | No | No | Yes |
| **File upload** | Yes | Yes | Yes | No | No | No | Yes |
| **Reasoning/CoT** | Yes | Yes | No | No | No | No | Yes |
| **Tool rendering** | Yes | Yes | Yes | No | No | Yes | Yes |
| **Weight** | Zero | Zero | npm dep | Zero | npm dep | Heavy | N/A |

---

## Build vs Use Analysis

### Use: prompt-kit or AI Elements
- Both follow the shadcn copy-paste pattern -- zero runtime dependency
- Components land in our `src/components/ui/` directory, fully owned
- Can be customized to match our existing design system
- Tailwind v4 and React 19 compatible
- AI SDK integration documented/native

### Use: assistant-ui
- Most feature-rich, but introduces an npm dependency (`@assistant-ui/react`)
- Pre-1.0 API means potential breaking changes on upgrades
- Heavier abstraction; less control over internals

### Build from scratch
- Full control, but significant effort (2-3 weeks for a solid chat UI)
- Would need to handle: auto-scroll, streaming text rendering, markdown parsing, textarea auto-resize, keyboard shortcuts
- Not recommended when mature shadcn-native options exist

### Verdict
**Use over Build.** prompt-kit and AI Elements both provide exactly what we need with zero dependency overhead, and they follow our existing shadcn pattern. Building from scratch would duplicate well-solved problems.

---

## Recommendation

### Primary: prompt-kit

**Why:**
1. **Perfect stack alignment** -- shadcn-native, Tailwind v4, React 19, zero runtime dependency
2. **Right-sized** -- covers all our Must/Should requirements without bloat
3. **Ownership model** -- components are copied into our project, fully customizable
4. **AI-focused** -- every component is designed for AI chat UIs specifically
5. **Active maintenance** -- last commit 2 days ago, responsive to issues
6. **MIT license** -- matches our project

**How:** Install individual components via the shadcn CLI:
```bash
npx shadcn@latest add "https://prompt-kit.com/c/prompt-input.json"
npx shadcn@latest add "https://prompt-kit.com/c/message.json"
npx shadcn@latest add "https://prompt-kit.com/c/markdown.json"
npx shadcn@latest add "https://prompt-kit.com/c/text-shimmer.json"
npx shadcn@latest add "https://prompt-kit.com/c/prompt-suggestion.json"
npx shadcn@latest add "https://prompt-kit.com/c/file-upload.json"
npx shadcn@latest add "https://prompt-kit.com/c/code-block.json"
```

### Secondary consideration: AI Elements

If we need more advanced components later (voice, workflow, model selector), AI Elements has the broadest catalog. Since both use the shadcn registry pattern, they can coexist -- we could use prompt-kit for core chat and cherry-pick AI Elements components for specialized needs.

### Not recommended for this use case:
- **assistant-ui** -- excellent but npm-dependency model conflicts with our shadcn ownership pattern
- **shadcn-chat** -- stale, not AI-focused
- **chatscope** -- SCSS-based, not AI-focused
- **CopilotKit** -- massive framework overkill
- **vercel/chatbot** -- reference app, not a library

---

## Implementation Context

```yaml
# claude_context for implementation phase

if_use_prompt_kit:
  install_method: "npx shadcn@latest add https://prompt-kit.com/c/{component}.json"
  components_dir: "src/components/ui/"
  core_components:
    - prompt-input    # Auto-resizing textarea with submit
    - message         # User/assistant message bubbles
    - markdown        # AI response markdown rendering
    - chat-container  # Scroll container with auto-scroll
    - text-shimmer    # Streaming text indicator
    - prompt-suggestion  # Quick-action suggestion chips
    - file-upload     # Drag-and-drop file upload
    - code-block      # Syntax-highlighted code blocks
  optional_components:
    - reasoning       # Expandable CoT panel
    - thinking-bar    # Thinking progress indicator
    - tool            # Tool call rendering
    - feedback-bar    # Thumbs up/down feedback
    - source          # Source citation display
  dependencies_to_check:
    - "react-markdown or similar (for markdown component)"
    - "shiki or prism (for code-block syntax highlighting)"
  integration_pattern: |
    Use with @ai-sdk/react useChat hook:
    - useChat() provides messages, input, handleSubmit, isLoading
    - Map messages to <Message> components
    - Bind input/handleSubmit to <PromptInput>
    - Use isLoading for <TextShimmer> streaming indicator
  styling: "Uses cn() utility, Tailwind v4 classes, inherits shadcn theme vars"

if_use_ai_elements:
  install_method: "npx shadcn@latest add https://elements.ai-sdk.dev/api/registry/{component}.json"
  components_dir: "src/components/ai-elements/"
  note: "Verify Vite 8 compatibility before committing -- primarily tested with Next.js"
  license: "Apache-2.0 (compatible with MIT projects)"

if_build:
  estimated_effort: "2-3 weeks"
  key_challenges:
    - "Auto-scroll with streaming (harder than it looks)"
    - "Markdown rendering with streaming partial content"
    - "Textarea auto-resize across browsers"
    - "Accessible keyboard navigation"
  recommendation: "Not recommended -- prompt-kit solves this well"

integration_with_existing_stack:
  ai_sdk: "AI SDK v6 with @ai-sdk/react useChat hook"
  state: "Chat state via useChat, app state via Zustand"
  routing: "Dashboard route: src/routes/dashboard/"
  ui_pattern: "Shadcn components in src/components/ui/"
  icons: "Phosphor Icons (already in project)"
```

---

## Sources

- [prompt-kit GitHub](https://github.com/ibelick/prompt-kit)
- [prompt-kit Website](https://www.prompt-kit.com/)
- [prompt-kit Installation Docs](https://www.prompt-kit.com/docs/installation)
- [AI Elements GitHub](https://github.com/vercel/ai-elements)
- [AI Elements Website](https://elements.ai-sdk.dev/)
- [Vercel AI Elements Announcement](https://vercel.com/changelog/introducing-ai-elements)
- [AI Elements LogRocket Tutorial](https://blog.logrocket.com/vercel-ai-elements/)
- [assistant-ui GitHub](https://github.com/assistant-ui/assistant-ui)
- [assistant-ui Website](https://www.assistant-ui.com/)
- [assistant-ui npm](https://www.npmjs.com/package/@assistant-ui/react)
- [assistant-ui SaaStr Feature](https://www.saastr.com/ai-app-of-the-week-assistant-ui-the-react-library-thats-eating-the-ai-chat-interface-market/)
- [assistant-ui Y Combinator](https://www.ycombinator.com/companies/assistant-ui)
- [shadcn-chat GitHub](https://github.com/jakobhoeg/shadcn-chat)
- [chatscope GitHub](https://github.com/chatscope/chat-ui-kit-react)
- [chatscope Website](https://chatscope.io/)
- [CopilotKit GitHub](https://github.com/CopilotKit/CopilotKit)
- [CopilotKit npm](https://www.npmjs.com/package/@copilotkit/react-ui)
- [vercel/chatbot GitHub](https://github.com/vercel/chatbot)
- [Vercel AI SDK 6 Blog](https://vercel.com/blog/ai-sdk-6)
- [Generative UI Frameworks Guide 2026](https://medium.com/@akshaychame2/the-complete-guide-to-generative-ui-frameworks-in-2026-fde71c4fa8cc)
- [shadcn/ui Tailwind v4 Docs](https://ui.shadcn.com/docs/tailwind-v4)
