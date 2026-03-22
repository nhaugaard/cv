# Open Source Research: Obsidian-based Agent Context Systems

## Strategic Summary

Several projects exist that solve the same problem we're tackling: persistent memory for Claude Code agents using Obsidian vaults. The two most interesting are **phelps-sg/claude-code-obsidian-skills** (skill-based vault manager with tag taxonomy) and **chennurivarun/infinite-context** (memory hierarchy architecture with parallel agent coordination). Neither is a drop-in replacement — they're more like pattern libraries to learn from. Our setup is already more advanced in several ways (QMD semantic search, structured workspace per agent, Obsidian CLI integration), but both projects have ideas worth adopting.

## What we need

A persistent context system for Claude Code subagents that:
- Survives session boundaries (agent picks up where last session stopped)
- Loads relevant context fast at startup (not 6-minute queries)
- Supports structured knowledge (wiki, learnings, decisions, tasks)
- Works with Obsidian vault as storage
- Integrates with QMD for search
- Keeps token cost low (don't load everything every message)

## Options Found

### Option 1: phelps-sg/claude-code-obsidian-skills

- **Repo:** https://github.com/phelps-sg/claude-code-obsidian-skills
- **What it does:** 4 Claude Code skills that turn Obsidian into a shared knowledge base. Core `/pkm` skill defines vault conventions, instructs agent to write notes when discovering architecture, and orient at session start. Also has vault auditor, external sync, and WIP dashboard skills.
- **Stars/Downloads:** 10 stars, 0 forks
- **Last commit:** 2026-03-02
- **Contributors:** 1 (Steve Phelps)
- **License:** MIT (stated in README, no LICENSE file)
- **Architecture:**
  - **Flat vault — no folders.** All notes at root, organized by tags and wikilinks only
  - Tag taxonomy: `#project/`, `#service/`, `#lib/`, `#architecture`, `#domain/`, `#decision`, `#pr`, `#sketch`, `#methodology`, `#backlog`
  - Daily notes for backlog tracking
  - Uses only file tools (Read, Write, Glob, Grep) — no Obsidian CLI or MCP
  - Session startup: read recent daily note, grep `^#` for tag overview, check backlog
- **Key insight:** "Before exploring code for any architectural question, ALWAYS search the vault first"
- **Fits our need:** Partial — good conventions, but no search beyond grep, no agent workspace separation
- **What we can steal:**
  - `/pkm` startup routine (read daily note + tag grep) is simple and effective
  - Vault auditor concept (check for contradictions, broken links, latent insights)
  - "Code is always source of truth, notes capture understanding at a point in time" disclaimer
  - Backlog in daily notes pattern
- **Concerns:** Single-person project, flat vault won't scale for multi-agent, no semantic search

### Option 2: chennurivarun/infinite-context

- **Repo:** https://github.com/chennurivarun/infinite-context
- **What it does:** Obsidian as persistent memory layer with 3-tier hierarchy (CLAUDE.md → MEMORY.md → Obsidian vault). Focused on token efficiency and parallel agent coordination.
- **Stars/Downloads:** 3 stars, 0 forks
- **Last commit:** 2026-03-05
- **Contributors:** 1
- **License:** MIT
- **Architecture:**
  - 3-layer memory hierarchy (borrowed from CPU cache analogy):
    - CLAUDE.md (~50 lines, auto-loaded every msg) = registers
    - MEMORY.md (~30 lines, pointers to vault) = L1 cache
    - Obsidian vault (unlimited, on-demand) = hard drive
  - Vault template: Architecture.md, Design System.md, Gotchas.md, Patterns.md, Decisions.md, Build/ (agent scopes + results)
  - Parallel agent rules: agents own FLOWS not FILES, non-overlapping file ownership, read scope from Obsidian, write results to Obsidian
  - Wave pattern: Foundation → Features (parallel) → Integration → Audit → Fixes
- **Key insight:** "MEMORY.md is an index, not storage. It points to Obsidian docs. Claude reads them only when needed."
- **Fits our need:** Partial — excellent mental model and agent coordination patterns, but no actual implementation (it's a template repo, not code)
- **What we can steal:**
  - Memory hierarchy framing (what's always loaded vs on-demand)
  - MEMORY.md as pure pointer index (we're already close to this)
  - Agent coordination patterns (scope files, results files, wave dispatching)
  - Token budgeting thinking (3 tokens/line × messages = cost per line per session)
- **Concerns:** Template only, no search, no tooling, no Obsidian integration code

### Option 3: SizzleTheWizzle/obsidian-claude-code

- **Repo:** https://github.com/SizzleTheWizzle/obsidian-claude-code
- **What it does:** Pre-built Obsidian vault template with slash commands for knowledge management
- **Stars/Downloads:** 2 stars, 0 forks
- **Last commit:** 2026-03-14
- **License:** Not specified
- **Fits our need:** No — it's a generic vault template, nothing agent-specific beyond slash commands
- **Concerns:** Low quality README (looks auto-generated), zip download only

### Option 4: Aura-Intel/business-vault-os

- **Repo:** https://github.com/Aura-Intel/business-vault-os
- **What it does:** Knowledge graph architecture for Obsidian + Claude Code
- **Stars/Downloads:** 5 stars, 0 forks
- **Last commit:** 2025-12-14 (3 months stale)
- **Fits our need:** No — business-focused, not agent context
- **Concerns:** Abandoned

## Comparison

| Aspect | phelps-sg/pkm | infinite-context | Our current setup |
|--------|--------------|-----------------|-------------------|
| Vault structure | Flat (tags only) | Folder per project | Folder per agent (workspace) |
| Search | Grep only | None (manual) | QMD BM25 + vector + Obsidian CLI |
| Session startup | Daily note + tag grep | Read MEMORY.md pointers | Memory.md + qmd search + learnings scan |
| Token efficiency | Good (grep is cheap) | Excellent (pointer-based) | Good but room for improvement |
| Agent coordination | None | Excellent (scope/results files, wave pattern) | Basic (subagent definition only) |
| Knowledge capture | Proactive (writes on discovery) | Manual | Continuous (~5 min habit) |
| Audit/maintenance | Yes (vault-insights skill) | No | No |
| Code as source of truth | Explicit rule | Implicit | Not stated |
| Tooling | File tools only | Obsidian MCP | Obsidian CLI + QMD CLI |

## Build vs. Use Analysis

**Use existing:** None of these are drop-in solutions. They're all pattern libraries / templates.

**Build custom (what we're doing):** Our setup is already the most complete:
- Only one with actual search (QMD)
- Only one with structured agent workspaces
- Only one with continuous saving habit
- Only one with Obsidian CLI (vs MCP or file tools)

**Recommendation:** Keep building custom, but adopt these patterns from the research:

### Patterns to adopt

1. **From infinite-context — pointer-based MEMORY.md:**
   Our Memory.md is growing (70+ lines). Should be trimmed to pure pointers:
   ```
   - Architecture → wiki/architecture-overview.md
   - Current tasks → Tasks/session-handoff-tasks.md
   - Gotchas → Learnings/
   ```

2. **From infinite-context — token budgeting:**
   Every line in agent definition costs ~3 tokens × every API call. Our agent def is 193 lines. At 80 messages that's ~46K tokens just for the agent definition. Consider moving vault_access docs to a skill that loads on-demand.

3. **From phelps-sg — "vault before code" rule:**
   Add to CLAUDE.md: "Before exploring code for any question, search the vault first."

4. **From phelps-sg — vault auditor:**
   Periodic skill that checks for stale notes, contradictions, broken links.

5. **From infinite-context — agent scope/results pattern:**
   For parallel agent work: write scope to vault, agent reads it, writes results back.

6. **From phelps-sg — code is source of truth disclaimer:**
   Notes capture understanding at a point in time. If note is older than recent commits, verify against code first.

## Sources

- phelps-sg/claude-code-obsidian-skills: https://github.com/phelps-sg/claude-code-obsidian-skills (accessed 2026-03-15)
- chennurivarun/infinite-context: https://github.com/chennurivarun/infinite-context (accessed 2026-03-15)
- SizzleTheWizzle/obsidian-claude-code: https://github.com/SizzleTheWizzle/obsidian-claude-code (accessed 2026-03-15)
- Aura-Intel/business-vault-os: https://github.com/Aura-Intel/business-vault-os (accessed 2026-03-15)
- Blog post: https://sphelps.substack.com/p/a-shared-memory-for-claude-code (referenced, not fetched)
