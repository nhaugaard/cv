# Research Overview: Reactive Resume Fork — Strategic Direction

**Date:** 2026-03-14
**Scope:** 8 research documents covering competitive landscape, technical implementation, feasibility, open-source components, template evolution, and historical lessons for expanding our Reactive Resume fork into an AI-powered career platform.

---

## Executive Summary

Our Reactive Resume fork is exceptionally well-positioned. The existing AI infrastructure (AI SDK v6, multi-provider, MCP server, JSON Patch editing) places us ahead of every competitor in the open-source space and on par with funded SaaS tools technically. The research unanimously points toward one strategic direction: **evolve from a resume builder into a privacy-first, AI-powered career platform** — doing so incrementally, validating each phase with users.

---

## Research Documents Index

| # | Type | File | Key Finding |
|---|------|------|-------------|
| 1 | Deep Dive | `ai-dashboard-assistant-deep-dive.md` | 80% of infrastructure already exists for the AI dashboard; prompt-kit is the ideal UI library |
| 2 | Competitive | `resume-builders-competitive.md` | No competitor offers v0-style agentic resume creation; our MCP server is a technical moat |
| 3 | Landscape | `ai-career-tools-landscape.md` | $660M+ market, 60+ tools mapped; we're the only OSS builder with MCP — first-mover advantage |
| 4 | Technical | `ai-dashboard-implementation-technical.md` | prompt-kit + custom ORPC transport is the recommended stack; 2-3 day implementation |
| 5 | Feasibility | `ai-career-platform-feasibility.md` | Go with conditions — Phase 1-3 feasible, Phase 4-5 needs validation; ~40-66 weeks total |
| 6 | Open-source | `ai-ui-components-open-source.md` | prompt-kit confirmed best fit (shadcn-native, TW v4, React 19); assistant-ui as honorable mention |
| 7 | Options | `template-system-evolution-options.md` | Hybrid React layouts + CSS themes scores highest; 13 templates are ~70% shared structure |
| 8 | History | `ai-resume-builders-history.md` | Resume-only tools churn 12-18%/month; career platforms retain; our architecture matches all success patterns |

---

## Top 10 Strategic Insights

### 1. We Have a Technical Moat
Our MCP server makes us the only resume builder that LLMs can interact with natively. As agentic workflows grow (MCP: 97M+ monthly downloads), this becomes increasingly valuable.

### 2. No One Does Conversational Resume Creation
Every competitor's AI is a sidebar chatbot or form-fill assistant. A v0-style dashboard where you *converse* your resume into existence would be a market first.

### 3. Privacy + Self-Hosting Is a Trust Advantage
Only 26% of candidates trust AI career tools. 9/12 SaaS resume builders retain data post-deletion. Our self-hostable, BYOK (Bring Your Own Key), open-source model addresses this directly.

### 4. Resume-Only Is a Dead End
History shows resume-only tools churn at 12-18%/month. Winners (Teal: 1.5M users, Jobscan) built career platforms. The resume should be one node in a larger workflow.

### 5. The Template System Can Be Dramatically Simplified
Our 13 templates share ~70% structure. Decomposing into 5 React layout shells + CSS themes enables rapid template creation and opens the door to AI-generated themes.

### 6. prompt-kit Is the Right UI Library
Shadcn-native, Tailwind v4 compatible, React 19 required, MIT license, actively maintained. Installs directly into our component structure. AI SDK Elements is the fallback.

### 7. BYOK Keeps AI Costs at Zero for Us
Users supply their own API keys. A moderate user costs ~$0.04/month on Gemini Flash. This model is sustainable and aligns with our open-source ethos.

### 8. Phases 1-3 Are Low-Risk Wins
- Phase 1: AI Dashboard Assistant (existing infra, new UI) — **2-3 weeks**
- Phase 2: Job Description Analysis + ATS Scoring — **4-6 weeks**
- Phase 3: Cover Letter Generation — **4-6 weeks**

### 9. Add Tests Before Phase 2
The feasibility assessment flags no-test-framework as the #1 technical risk. Adding Vitest before Phase 2 prevents regression risk from compounding.

### 10. AI Template Generation Is an Unexploited Differentiator
No resume builder uses AI to generate visual templates. With the hybrid layout+theme system, AI can generate CSS themes within layout constraints — a tractable and unique feature.

---

## Recommended Roadmap

```
Phase 1: AI Dashboard Assistant          [2-3 weeks]
  ├── Install prompt-kit components
  ├── Create ai.dashboardChat ORPC procedure
  ├── Career coach system prompt
  ├── Tools: create_resume, list_resumes, patch_resume
  └── Progressive disclosure UI (prompt → full chat)

Phase 1.5: Template System Refactor      [3-5 weeks]
  ├── Extract 5 layout shells from 13 templates
  ├── Convert visual differences to CSS themes
  └── Foundation for AI theme generation

Phase 2: Job Description Analysis        [4-6 weeks]
  ├── Add Vitest test framework first
  ├── JD parsing and requirement extraction
  ├── ATS keyword matching and scoring
  └── Resume-to-job fit analysis

Phase 3: Cover Letter Generation         [4-6 weeks]
  ├── AI cover letter from resume + JD
  ├── Cover letter templates (reuse layout engine)
  └── PDF export for cover letters

Phase 4: Career Intelligence             [8-12 weeks] [validate first]
  ├── Job application tracker (kanban)
  ├── Interview prep
  └── Skills gap analysis

Phase 5: Multi-document Portfolio        [8-12 weeks] [validate first]
  ├── Shared data model across document types
  ├── Unified export
  └── Portfolio pages
```

---

## Key Risks & Mitigations

| Risk | Severity | Mitigation |
|------|----------|------------|
| Solo developer burnout | High | Phased approach, AI-assisted development, validate before building |
| No test framework | High | Add Vitest before Phase 2 |
| Upstream sync breaks | Medium | Modular additions in separate files/folders, minimize core changes |
| AI hallucinated content | Medium | Strong anti-hallucination prompts (already proven in PDF parser) |
| Scope creep | Medium | Strict phase gates, user validation between phases |

---

## Competitive Position Summary

```
                    AI Depth
                       ↑
                       │
         Teal ●        │        ● Our Target (Phase 3+)
                       │
    Jobscan ●          │   ● Rezi
                       │
                       │        ★ WE ARE HERE
     Resume.io ●       │
                       │   ● Enhancv
    FlowCV ●           │
                       │
    ───────────────────┼────────────────────→ Privacy/Control
    Closed/SaaS        │              Open/Self-hosted
                       │
    Canva ●            │   ● OpenResume
                       │
```

---

## Next Actions

1. **Read the detailed research** — each document has Implementation Context with specific technical guidance
2. **Decide on Phase 1 start** — the AI Dashboard Assistant has the clearest path and highest impact
3. **Consider adding Vitest** — flagged as critical before Phase 2 by the feasibility assessment
4. **Template system refactor** — can run in parallel with Phase 1 as independent work

---

*Generated from 8 parallel research sessions on 2026-03-14. Total research: ~207 KB across competitive analysis, landscape mapping, technical research, feasibility assessment, open-source evaluation, options comparison, and historical analysis.*
