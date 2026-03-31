# Claude Code vs Zenpower: Architectural Comparison

A three-way comparison between Claude Code (leaked CLI source), zenpower-mono (the product platform), and zenpower-monorepo (the ops/infrastructure layer).

---

## What Each System Is

**Claude Code** — A TypeScript CLI agent (React/Ink terminal app) that runs locally on a developer's machine. Sends messages to the Anthropic API, executes 40+ tools (bash, file edits, web search), manages permissions with risk classification, and handles multi-turn agentic conversations with auto-compaction and memory. Single-purpose: AI-assisted coding.

**zenpower-mono** (v0.23.1) — A Python platform of 7 layered packages: `zenpower-ai` (multi-provider LLM gateway + Knowledge Brain with 7 retrieval strategies), `zenpower-mcp` (449 tools across 11 domains), `zenpower-agents` (29 agents with P2P trust, 6 safety hooks, mixture-of-agents routing), `zenpower-repricing` (6 pricing strategies, e-commerce connectors), `zenpower-sec` (Guardian identity system, 28 security modules, OPA policies), `zenpower-ui` (web dashboard + Textual TUI + CLI), and `zenpower-skynet` (autonomous 4-tier agent topology with circuit breakers). 3,468 tests, 668 Python files.

**zenpower-monorepo** — The operational backbone: FastAPI API (116 routers), RQ worker queue, 58 MCP workers (498+ tools), Docker Compose with 30+ profiles, 24 systemd units, Astro/Next.js frontends, Odoo 19 ERP (10 custom modules), branding-driven multi-tenant deployment. 837+ tests, 11,869 Python files. Production infrastructure at zenpower.at.

---

## Similarities

### MCP as Plugin Bus
All three use MCP as the extensibility mechanism. Claude Code consumes MCP servers (`services/mcp/`). zenpower-mono produces MCP tools (449 tools, 11 domains with progressive loading under 3KB token budget). zenpower-monorepo runs a full MCP gateway (`apps/mcp/gateway.py`) with 58 workers and 498+ tools. All treat MCP as the universal tool interface.

### Multi-Agent Architecture
Claude Code has coordinator mode (research → synthesis → implementation → verification phases), agent swarms, and sub-agent spawning. zenpower-mono has 29 agents with personas, mixture-of-agents routing, P2P delegation (depth limit 3), and a 4-tier topology (strategic → tactical → operational → feedback) in Skynet. zenpower-monorepo has agent lifecycle management (run/stop/watch), orchestrator with DAG-based execution plans, and an agents dispatcher pipeline.

### Tiered Authorization
Claude Code classifies tools as LOW/MEDIUM/HIGH risk with permission modes (default, auto, bypass). zenpower-mono has TrustTier gating (sandbox 0.0-0.3 → trusted 0.8-1.0) filtering tool access per agent, plus MCP AuthLevel hierarchy (PUBLIC → API_KEY → TENANT → ADMIN). zenpower-monorepo has OPA policies with sandbox tiers 0-3, surface constraints (phone=0, pc=2, server=3), and agent type ceilings. All go beyond simple RBAC.

### Memory Systems
Claude Code has `~/.claude/projects/*/memory/` with typed memories (user, feedback, project, reference) and a "dream" consolidation engine. zenpower-mono has a 4-tier memory model (working, episodic, semantic, archival) in the Knowledge Brain with 7 retrieval strategies. zenpower-monorepo has the Brain MCP worker with `brain.memory` tool (recall/insert/update/forget) and a `brain-consolidate` systemd timer.

### Async-First Design
Claude Code's conversation loop is an async generator streaming events. zenpower-mono uses FastAPI + async SQLAlchemy throughout, with SSE streaming for LLM responses. zenpower-monorepo's API is fully async with `AsyncSession` and no blocking calls in hot paths. All three avoid synchronous I/O.

### Feature Gating
Claude Code uses compile-time flags (`feature()` from `bun:bundle`) and runtime GrowthBook (`tengu_*` prefix). zenpower-mono uses Pydantic `BaseSettings` with env prefixes (`ZPAI_*`, `ZPMCP_*`, etc.). zenpower-monorepo uses environment variables and config files. All gate features progressively, though with varying sophistication.

---

## Differences

| Dimension | Claude Code | zenpower-mono | zenpower-monorepo |
|-----------|------------|---------------|-------------------|
| **Language** | TypeScript (Bun) | Python 3.12+ | Python 3.11+ |
| **Purpose** | AI coding CLI | AI business platform | Ops/infrastructure layer |
| **Scope** | Single-purpose client | 7-package product suite | Multi-app monorepo |
| **UI** | React/Ink terminal | Web + Textual TUI + CLI (`zpui`) | Astro/Next.js + ZenControl TUI (curses) |
| **State** | Zustand (200+ fields, in-process) | PostgreSQL + Redis + pgvector | PostgreSQL + Redis |
| **LLM Provider** | Anthropic only | 9 providers, 25+ models | Multi-provider (OpenAI primary, fallback chain) |
| **Agent Model** | Single AI (Claude) | 29 agents, mixture-of-experts | Agent dispatcher with 9 persona types |
| **Tools** | 40+ built-in tools | 449 MCP tools, 11 domains | 498+ tools via 58 MCP workers |
| **Auth** | Claude AI OAuth | Multi-tier (API key, JWT, Guardian, OPA) | Multi-method (email, wallet, API key, staff) |
| **Testing** | None visible | 3,468 tests, 80% coverage req | 837+ tests, 48 markers |
| **Deployment** | npm package (user's machine) | Docker Compose (6 services) | Docker Compose (30+ profiles) + systemd |
| **Security** | Permission dialogs, ML classifier | Guardian (Ed25519 + hash ledger), OPA, 6 hooks | OPA policies, SOPS/age (in progress) |
| **Knowledge** | CLAUDE.md + conversation memory | Knowledge Brain (7 retrieval, P2P federation) | Brain MCP worker + knowledge ingestion |
| **Job Queue** | In-process async generators | Redis Streams (Skynet event bus) | RQ + Redis (fire-and-forget) |
| **Build** | Bun bundler (single .js output) | Hatchling + per-package Dockerfiles | Makefile + Docker Compose |
| **MCP Role** | Consumer | Producer (protocol implementation) | Producer (gateway + workers) |

---

## Pros and Cons

### Claude Code

| Pros | Cons |
|------|------|
| Deeply optimized conversation loop with streaming, auto-compaction, and reactive recovery | Single-provider lock-in (Anthropic API only) |
| Sophisticated tool permission system with ML-based auto-approval | Monolithic — 785KB main.tsx, circular dependency issues |
| Compile-time dead-code elimination keeps external builds lean | No server-side component — everything runs on user's machine |
| Tool concurrency partitioning (read-only parallel, write serial) | Complex state (200+ Zustand fields) hard to reason about |
| System prompt caching (static/dynamic split) reduces API costs | No visible test suite |
| Dream memory consolidation is a novel approach to long-term learning | Tight coupling to Bun runtime and React/Ink |
| Interactive permission dialogs give users real-time control | Feature flag sprawl (dozens of `tengu_` gates) |

### zenpower-mono

| Pros | Cons |
|------|------|
| 7 clean packages with layered dependencies | No context window management or auto-compaction |
| 9 LLM providers with cost-optimized routing | Memory tier implementation delegated to external service |
| 449 MCP tools with progressive loading (<3KB token budget) | No skill hot-reload (code-based, requires restart) |
| Guardian identity (Ed25519 + hash-chained ledger) for non-repudiation | No WebSocket (SSE only, no bidirectional streaming) |
| 3,468 tests with 80% coverage requirement | No true multi-agent parallelism (single agent per task) |
| P2P brain federation with vector sync protocol | Agent orchestration limited to depth-3 delegation |
| TrustTier-based tool filtering per agent | No job cancellation mechanism |
| Mixture-of-agents routing with learning from outcomes | No input/output token tracking (only cost estimates) |
| Textual TUI + web dashboard + CLI (3 interfaces) | No prompt caching or system prompt composition strategy |
| Circuit breaker + RLHF feedback loops (Skynet) | Some subsystems are research-phase (P2P, federation) |

### zenpower-monorepo

| Pros | Cons |
|------|------|
| Production-proven infrastructure (zenpower.at running) | No agentic conversation loop (jobs are fire-and-forget) |
| 116 API routers with clean service layer separation | No streaming from within agent execution |
| 58 MCP workers with massive tool surface | No context window management |
| Branding-driven multi-tenant deployment | No tool concurrency optimization |
| 24 systemd units for ops automation | Basic RQ queue (no progress reporting, no cancellation) |
| Comprehensive OPA policies (tier/surface/auth/budget) | No prompt caching |
| 10 Odoo custom modules for ERP integration | Some apps incomplete (Shopware, Bitcoin fork) |
| Excellent documentation discipline (20MB, 153 anti-patterns) | Feature toggling is basic (env vars only) |
| Strong deployment pipeline (GO script, health checks, rollback) | No interactive permission UI for agents |

---

## What's Missing in zenpower-mono

### 1. Agentic Conversation Loop with Streaming

**What Claude Code has**: An `async generator` in `query.ts` that streams API responses, detects tool calls, executes them inline, appends results, and loops until the model stops. The UI receives events in real-time. This is the core pattern that makes Claude Code feel like an autonomous agent.

**What zenpower-mono has**: `AgentRuntime.execute()` calls the Claude API with native `tool_use`, loops through tool calls, but returns a single `ExecutionResult` at the end. No streaming to the caller during execution. No event yielding.

**Gap**: The runtime has the loop but not the streaming. Callers can't see what the agent is doing in real-time. Adding `async generator` semantics to `AgentRuntime` (yielding events per tool call) would close this gap. The Skynet event bus (Redis Streams) could be the transport.

### 2. Context Window Management and Auto-Compaction

**What Claude Code has**: Token estimation for fast local counting. Auto-compaction when approaching the context limit (13K buffer). Reactive compaction on `prompt_too_long` errors (compact and retry). Tool result budgeting (large outputs get truncated). A `max_output_tokens` recovery loop (retry up to 3 times).

**What zenpower-mono has**: `estimated_tokens = len(content) // 4` (rough approximation). Cost tracking per persona/action. No context window sizing, no compaction, no retry-with-recovery. Budget enforcement is global (24-hour period), not per-conversation.

**Gap**: Critical for long-running agents. Need a token tracking layer around LLM calls, auto-summarization of older messages when approaching limits, and graceful handling of `prompt_too_long` errors. The Brain's summarization capabilities could power the compaction.

### 3. Tool Concurrency Partitioning

**What Claude Code has**: `toolOrchestration.ts` partitions tool calls into batches — consecutive read-only tools run concurrently (up to 10 in parallel), write tools run serially. Each tool declares `isConcurrencySafe()`.

**What zenpower-mono has**: The MCP gateway has `parallel.py` for batch execution with progress callbacks, but it's used for explicit batch requests — not automatic partitioning of tool calls within an agent turn.

**Gap**: Add a `concurrency_safe` flag to MCP `ToolSchema`. When the agent runtime receives multiple tool calls from the model, partition them: run read-only tools concurrently, write tools serially. The parallel execution infrastructure already exists in the gateway.

### 4. System Prompt Composition with Cache Boundaries

**What Claude Code has**: The system prompt is assembled from modular sections in `constants/`, split into static (cacheable across orgs) and dynamic (per user/session) by a `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` marker. This lets the API cache the expensive static prefix. There's even a `DANGEROUS_uncachedSystemPromptSection()` for volatile content.

**What zenpower-mono has**: No prompt composition strategy. Agent prompts are assembled ad-hoc. No caching, no static/dynamic split, no cost optimization for repeated prompt prefixes.

**Gap**: With 9 LLM providers and cost-optimized routing, this is a significant cost saving opportunity. Build a prompt composition system that marks cache boundaries, reuses static portions across requests, and tracks which sections changed.

### 5. Reactive Error Recovery

**What Claude Code has**: When the API returns `max_output_tokens`, retry up to 3 times. When `prompt_too_long`, compact and retry. Model fallback chains (`fallbackModel`) activate on persistent errors. Withheld error messages prevent premature termination by SDK callers.

**What zenpower-mono has**: The AI gateway has fallback routing (primary → fallback provider), and the circuit breaker (Skynet) handles fault tolerance with exponential backoff. But these are at the provider level, not the conversation level. No compact-and-retry within an agent turn.

**Gap**: Add conversation-level error recovery to `AgentRuntime`: detect recoverable errors (context too long, output truncated), apply fixes (compact history, reduce tool results), and retry. The circuit breaker pattern from Skynet could be adapted for this.

### 6. Per-Tool Risk Classification with Interactive Approval

**What Claude Code has**: Every tool action is classified as LOW/MEDIUM/HIGH risk. Protected files are guarded. Path traversal attacks are detected. An ML classifier can auto-approve in "auto" mode. Users see interactive permission dialogs in the terminal.

**What zenpower-mono has**: OPA policies enforce tier-based access and TrustTier filters tools per agent. The approval hook (`hooks/approval.py`) gates high-tier actions. But there's no per-tool risk metadata, no ML auto-approval, and no interactive UI for real-time permission decisions.

**Gap**: Add risk level metadata to `ToolSchema` (LOW/MEDIUM/HIGH). Combine with existing TrustTier: a SANDBOX agent hitting a HIGH-risk tool triggers the approval hook. The Textual TUI could render approval dialogs. The router learning system could eventually learn to auto-approve safe patterns (analogous to Claude Code's ML classifier).

### 7. Dream-Like Memory Consolidation

**What Claude Code has**: A background sub-agent (`services/autoDream/`) that runs when three gates pass (24h elapsed, 5+ sessions, consolidation lock). It reads recent memories, synthesizes them into durable topic files, prunes stale entries, and keeps the index under 200 lines. The sub-agent gets read-only bash.

**What zenpower-mono has**: The Brain has 4-tier memory and the monorepo has a `brain-consolidate` systemd timer. But consolidation is a scheduled job — not an intelligent pass that reads recent context, detects patterns, resolves contradictions, and prunes outdated facts.

**Gap**: Build a consolidation agent (could be a Skynet operational agent) that periodically reviews recent episodic memory, promotes patterns to semantic memory, resolves contradictions, and prunes stale entries. Gate it with Claude Code's three-gate pattern to prevent over-consolidation. The Brain's retrieval strategies (HyDE, RAPTOR) could power the synthesis.

### 8. Multi-Agent Parallel Orchestration

**What Claude Code has**: Coordinator mode transforms a single agent into an orchestrator managing parallel workers in phases. Workers communicate via `<task-notification>` XML messages. There's a shared scratchpad directory for cross-worker state. Agent swarms with color-coded panes and team memory synchronization.

**What zenpower-mono has**: The orchestrator (`apps/orchestrator/service.py`) executes DAG-based plans with steps, but each step is single-agent. Delegation allows nesting (depth 3) but not true parallelism. Skynet has a 4-tier topology but agents don't coordinate within a single task.

**Gap**: Add a coordinator agent type that decomposes a complex task, dispatches subtasks to multiple agents in parallel (using the existing mixture-of-agents routing), collects results, and synthesizes. The Skynet event bus (Redis Streams) is the right transport. The DAG orchestrator already handles step dependencies — extend it to support parallel step execution.

### 9. Typed Feature Flag System with Analytics

**What Claude Code has**: Compile-time flags via `feature()` that eliminate dead code. Runtime flags via GrowthBook with typed access, aggressive caching (`getFeatureValue_CACHED_MAY_BE_STALE()`), and analytics integration.

**What zenpower-mono has**: Pydantic `BaseSettings` with env prefixes. Clean and typed, but no runtime toggling, no analytics on flag evaluation, no gradual rollout, no A/B testing.

**Gap**: Add a feature flag service (could be as simple as a Redis-backed config store — Skynet's `ConfigStore` already exists). Track flag evaluations for analytics. Enable gradual rollout and per-tenant overrides.

### 10. Conversation-Level Token Budget (Not Just Daily)

**What Claude Code has**: Per-turn token budget tracking, `task_budget` with total/remaining computed per iteration, and `getCurrentTurnTokenBudget()` for within-conversation limits.

**What zenpower-mono has**: `CostTracker` with rolling 24-hour window, `budget_remaining()` and `is_over_budget()`. But this is daily/global — not per-conversation or per-turn.

**Gap**: Add per-conversation token accounting to `AgentRuntime`. Track input/output tokens from actual API responses (not just cost estimates). Enforce per-turn limits so a single runaway conversation can't exhaust the daily budget.

---

## What zenpower-mono Has That Claude Code Doesn't

It's not a one-way comparison. zenpower-mono has significant capabilities Claude Code lacks:

| Capability | zenpower-mono | Claude Code |
|-----------|---------------|-------------|
| **Multi-provider routing** | 9 providers, cost-optimized fallback | Anthropic only |
| **Knowledge Brain** | 7 retrieval strategies, P2P federation, entity graph | Flat memory files |
| **P2P Trust** | Guardian identity (Ed25519), hash-chained ledger, delegation | OAuth tokens |
| **Business vertical** | Repricing engine (6 strategies), e-commerce connectors | None |
| **Learning systems** | Router learning, approval learning, RLHF feedback loops | Static rules |
| **Security framework** | 28 modules, self-healing, forensics, honeypots, killswitch | Permission dialogs |
| **Agent personas** | 29 specialized agents with personality and skill profiles | Single generic agent |
| **Circuit breaker** | Exponential backoff, fault isolation per provider | Simple retry |
| **Safeguards** | 20+ disaster prevention checks (disk, DB, SSL, backups) | None |
| **Multi-interface** | Web + Textual TUI + CLI, same backend | Terminal only |
| **Token budget awareness** | Progressive tool loading (<3KB), schema caching | Tool schemas sent in full |

---

## Priority Ranking: What to Adopt from Claude Code

| Priority | Pattern | Impact | Effort | Notes |
|----------|---------|--------|--------|-------|
| **1** | Streaming agent execution (async generators) | Critical | Medium | AgentRuntime already loops — add `yield` semantics |
| **2** | Context window management + auto-compaction | Critical | Medium | Brain summarization can power it |
| **3** | Tool concurrency partitioning | High | Low | `parallel.py` exists, add `concurrency_safe` flag |
| **4** | Reactive error recovery (compact-and-retry) | High | Low | Extend circuit breaker to conversation level |
| **5** | Per-conversation token budget | High | Low | Extend existing CostTracker |
| **6** | System prompt caching | Medium | Low | Static/dynamic split, mark cache boundaries |
| **7** | Per-tool risk classification | Medium | Medium | Extend ToolSchema, connect to approval hook |
| **8** | Intelligent memory consolidation | Medium | Medium | Skynet agent + Brain retrieval |
| **9** | Multi-agent parallel orchestration | Medium | High | Extend DAG orchestrator + Skynet event bus |
| **10** | Typed feature flags with analytics | Low | Low | Extend Skynet ConfigStore |
