# Archive: markdown.engineering/learn-claude-code

> **Archived on:** 2026-04-01  
> **Source URL:** https://www.markdown.engineering/learn-claude-code  
> **Note:** The site returns HTTP 403 to automated fetchers. All content below was retrieved via search engine caches, indexed snippets, and companion GitHub repositories.

---

## Main Landing Page

**URL:** https://www.markdown.engineering/learn-claude-code

**Description (from search index):**  
A way to learn markdown engineering through a terminal-based learning game, covering primitives, models, tools, MCPs, skills, and agents. The site offers a curriculum-style breakdown of how Claude Code works internally — a source-code deep dive series.

---

## Lesson 01 — Claude Code Boot Sequence — Source Deep Dive

**URL:** https://www.markdown.engineering/learn-claude-code/01-boot-sequence  
**Title:** "From claude keystroke to interactive REPL — a deep dive into every phase of startup"

### Overview

When you type `claude` in your terminal, a sophisticated multi-phase boot pipeline runs before you see the first prompt. Understanding this pipeline helps you:
- Reason about startup latency
- Debug weird first-launch behaviors
- Appreciate the careful parallelism engineers built in to keep time-to-interactive low

### Source Files Covered

```
entrypoints/cli.tsx → main.tsx → setup.ts → bootstrap/state.ts → replLauncher.tsx → ink.ts
```

### Three Nested Boot Layers

At the highest level, boot happens in three nested layers:

- **Layer 1** · `cli.tsx` — zero-cost fast paths, environment prep, argv dispatch
- **Layer 2** · `main.tsx` — Commander parsing, init, migrations, permission checks
- **Layer 3** · `setup.ts` + `replLauncher.tsx` — session wiring, Ink render

### Key Design Decisions

**Thin Bootstrap / Dynamic Imports:**  
`cli.tsx` is a deliberate thin bootstrap — all imports are dynamic so the "zero module" fast paths (like `--version`, `--daemon-worker`, `--claude-in-chrome-mcp`) return without loading any heavy CLI surface.  
→ Significant UX win: `claude --version` returns in milliseconds.

**Bare Mode (`--bare` / `CLAUDE_CODE_SIMPLE`):**  
Used for scripted/SDK calls (`claude -p "..."` style). In bare mode, several startup steps are skipped entirely.  
Design principle: *"Bare mode is latency-sensitive — every millisecond saved matters when calling Claude from a CI pipeline hundreds of times a day."*

**Migrations:**  
Migrations run on every process start but are gated by `migrationVersion`.  
⚠️ If you downgrade Claude Code, the migration version is already advanced and migrations won't re-run — this can cause subtle config inconsistencies.

**Sticky-on Latches in `state.ts`:**  
Once AFK mode, fast mode, or cache-editing mode is first activated, the corresponding API header stays on for the rest of the session, preventing server-side prompt cache busts during mid-session toggles.  
If the header toggled with each settings change, "the server's cached prompt prefix would be busted, causing expensive cache misses on the Anthropic side (~50–70K tokens re-processed)."

---

## Companion Resource: shareAI-lab/learn-claude-code (GitHub)

**URL:** https://github.com/shareAI-lab/learn-claude-code  
**Title:** "Learn Claude Code — Harness Engineering for Real Agents"  
**License:** MIT

### Core Philosophy

> "An agent is a model. Not a framework. Not a prompt chain."

The model makes decisions; the harness enables execution.  
**"The model is the driver. The harness is the vehicle."**

**What is a Harness?**  
`Tools + Knowledge + Observation + Action Interfaces + Permissions`

**Key teaching insight:**  
> "The best agent products are built by engineers who understand that their job is harness, not intelligence. You are not writing the intelligence — you are building the world the intelligence inhabits. The quality of that world — how clearly the agent can perceive, how precisely it can act, how rich its available knowledge is — directly determines how effectively the intelligence can express itself."

### Minimal Agent Loop Pattern

```
User → messages[] → LLM → response
                    ↓
              stop_reason check
              ↙                ↘
        tool_use             end
          ↓
    execute tools
    append results
    loop back
```

### 12-Session Curriculum (Four Phases)

**Phase 1: The Loop**
- `s01` — Basic agent loop with tool dispatch
- `s02` — Tool use mechanism

**Phase 2: Planning & Knowledge**
- `s03` — Task planning (TodoWrite)
- `s04` — Subagent spawning
- `s05` — On-demand skill loading
- `s06` — Context compression

**Phase 3: Persistence**
- `s07` — File-based task systems with dependency graphs
- `s08` — Background task execution

**Phase 4: Teams**
- `s09–s12` — Multi-agent coordination, autonomous claiming, worktree isolation

### Resources
- Python reference implementations for each session
- Multi-language documentation (English, Chinese, Japanese)
- Interactive web platform (Next.js)
- Companion tools: Kode Agent CLI, Kode Agent SDK
- Sister repo: `claw0` — teaches proactive harness mechanisms (heartbeat, cron, IM routing, memory systems)

---

## Companion Resource: Ringmast4r/674019130-learn-real-claude-code (GitHub)

**URL:** https://github.com/Ringmast4r/674019130-learn-real-claude-code  
**Title:** "Learn Real Claude Code — Source Archaeology on a 512K-Line Agent"  
**License:** MIT

### Background

On March 31, 2026, a security researcher discovered Anthropic accidentally shipped the complete TypeScript source of Claude Code in an npm package — 512,000+ lines, 1,902 files, a 60MB sourcemap.

**Core distinction:**  
> "Claude Code is not an agent. Claude Code is a harness."  
The agent is Claude (the neural network). Claude Code is the shell providing tools, context management, permissions, UI, and streaming infrastructure.

### Actual Agent Loop (from `query.ts` source)

```
User input
    ↓
[ Fast Path? ]──yes──> exit(0) before any imports load
    ↓ no
[ Slash Command? ]──yes──> commandRegistry.dispatch()
    ↓ no
Build messages[] ◄──────────────────────────────────┐
    ↓                                                 │
Assemble system prompt                               │
(10+ sources, cache boundary)                        │
    ↓                                                 │
Stream API call ─────────────────────────────────────┼──> telemetry
    ↓ (mid-stream)                                   │
StreamingToolExecutor fires tools                    │
(concurrent if isConcurrencySafe())                  │
    ↓                                                 │
Inject tool results                                  │
    ↓                                                 │
[ tokenBudget > 95%? ]──yes──> compressContext()     │
    ↓                                                 │
[ More tool calls? ]──yes────────────────────────────┘
    ↓ no
Render response (Ink/React → terminal)
```

**What's missing from textbooks:**
- Mid-stream tool execution
- Concurrent tool dispatch
- Automatic context compression
- Permission racing
- Cost tracking on every API call

### Four Production Agent Patterns (Hidden in Textbooks)

1. **Mid-stream tool execution** — Tools fire at delimiter boundaries while response streams, not after completion
2. **Context as finite resource** — Seven compression strategies fire in priority order
3. **Permission racing** — Shell hooks, AI classifier, and user confirmation run concurrently; first safe signal wins
4. **Multi-agent as file I/O** — Subprocesses inherit parent's prompt cache, write to temp files; coordinator reads those files

### 15 Chapters + 3 Plus Chapters

| Chapter | Topic |
|---------|-------|
| `c01` | Fast paths and lazy module evaluation (804KB TypeScript startup in 500ms) |
| `c02` | StreamingToolExecutor and mid-stream tool firing |
| `c03` | Concurrent tool execution with `isConcurrencySafe()` |
| `c04` | Commands vs. tools distinction (100+ slash commands) |
| `c05` | `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` cache economics |
| `c06` | Seven context compression strategies |
| `c07` | Auto-Dream phases (orient→gather→consolidate→prune) with CLAUDE.md persistence |
| `c08` | Permission racing: hooks, classifier, user confirmation run concurrently |
| `c09` | Hook lifecycle: PreToolUse, PostToolUse, three backends |
| `c10` | Fork subagents sharing parent's prompt cache (Coordinator Mode) |
| `c11` | 18 deferred tools via ToolSearchTool (200K token base prompt) |
| `c12` | React in terminal: three interning pools, Yoga layout, double-buffered screen |
| `c13` | Dual feature gating: compile-time DCE + runtime flags (108 missing modules) |
| `c14` | Circuit breaker (max 3 failures), observability, cost tracking, 11-step error recovery |
| `c15` | BUDDY, KAIROS, Undercover, speculative execution — unshipped features |
| `p01` (PLUS) | Context Engineering as 2026 paradigm shift |
| `p02` (PLUS) | Agent scaffolding matters more than prompts |
| `p03` (PLUS) | Industrial agents: 20% LLM calls, 80% resource management |

### Unshipped Features Discovered in Source

- **BUDDY** — Virtual companion system using Mulberry32 PRNG seeded by userId, 0.01% chance of shiny legendary variant
- **KAIROS** — Always-on background agent continuously monitoring codebase
- **Undercover Mode** — System allowing Claude Code to present as a different AI assistant
- **Speculative Execution** — Pre-runs likely next actions during user reading time

### Source

Claude Code source extracted from npm package `@anthropic-ai/claude-code` version 2.1.88 via `cli.js.map` sourcemap.

Source mirrors:
- https://github.com/674019130/claude-code
- https://github.com/Kuberwastaken/claude-code

### Closing Statement

> "The source code doesn't lie. 512,000 lines say: production agents are mostly plumbing. Read the plumbing. Build better agents."

---

## What Could NOT Be Retrieved

The following URLs returned HTTP 403 (Forbidden) to automated fetchers:
- `https://markdown.engineering/learn-claude-code/`
- `https://www.markdown.engineering/learn-claude-code`
- `https://www.markdown.engineering/learn-claude-code/01-boot-sequence`

The Wayback Machine (`web.archive.org`) was also inaccessible from this environment.

Only Lesson 01 (`01-boot-sequence`) appeared in search engine indexes. No indexed content was found for lessons `02` onward from `markdown.engineering`. The site likely has more lessons but they are either not yet crawled or behind a login/paywall.

---

## Sources

- [Claude Code Source Deep Dive — Markdown Engineering](https://www.markdown.engineering/learn-claude-code)
- [mdENG — Lesson 01 — Claude Code Boot Sequence — Source Deep Dive](https://www.markdown.engineering/learn-claude-code/01-boot-sequence)
- [GitHub — shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code)
- [GitHub — Ringmast4r/674019130-learn-real-claude-code](https://github.com/Ringmast4r/674019130-learn-real-claude-code)
