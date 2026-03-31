# Claude Code Architecture

## Background: What This Is and How It Was Leaked

The public `anthropics/claude-code` repo on GitHub is **not the source code** — it's an issue tracker, documentation hub, and changelog. The actual TypeScript source was never open-sourced.

What gets published to npm (`@anthropic-ai/claude-code`) is a **bundled, minified single JavaScript file** — not human-readable. You can run it, but you can't meaningfully understand the architecture from it.

The leak happened because Anthropic's build process (using Bun's bundler) generated a **sourcemap file** (`.map`) and accidentally included it in the npm package. Sourcemaps contain the full original source code embedded as strings in a JSON structure — they exist so debuggers can map minified code back to the original. Someone ran `npm pack`, found the `.map` file, and extracted everything.

This repository contains that extracted source. It is the **client-side CLI tool** that runs on your machine when you type `claude` in your terminal. There is no server code here — the server side is just the Anthropic API (`api.anthropic.com`), which this client calls via the `@anthropic-ai/sdk`.

What makes this leak significant is what was **never meant to be public**:

- Internal system prompts and safety instructions (with named owners on the Safeguards team)
- Internal codenames (Tengu for Claude Code, Fennec for Opus, "Penguin Mode" for fast mode)
- Unreleased features: KAIROS (always-on assistant), Buddy (Tamagotchi pet), ULTRAPLAN (30-min remote planning), agent swarms, the Dream memory consolidation system
- The Undercover Mode system that hides the fact Anthropic employees use Claude Code to contribute to open-source repos
- Compile-time feature flags showing what's gated for internal vs external builds
- Unreleased API beta headers (`redact-thinking`, `afk-mode`, `advisor-tool`)

The irony is that they built an entire subsystem (Undercover Mode) to prevent their AI from leaking internal information in git commits — and then shipped all the source in a `.map` file.

---

How all the pieces of Claude Code fit together, from boot to tool execution.

## 1. Boot Sequence

Everything starts in **`entrypoints/cli.tsx`**. It has several fast-paths that short-circuit before loading anything heavy:

- `--version` — prints version, exits (zero imports)
- `--dump-system-prompt` — renders system prompt, exits
- `--daemon-worker` — launches a daemon worker subprocess
- `remote-control` — starts bridge mode

For the normal interactive case, it calls into **`main.tsx`** — the 785KB entry point. Before any imports even evaluate, three side-effects fire in parallel to hide latency:

1. **MDM settings prefetch** (`startMdmRawRead`) — reads managed device config via subprocess
2. **Keychain prefetch** (`startKeychainPrefetch`) — reads OAuth tokens from macOS keychain
3. **Startup profiler** checkpoint

`main.tsx` then uses Commander.js to parse CLI args (model, permission mode, flags, etc.), runs `init()` from `entrypoints/init.ts` (trust dialog, auth validation, telemetry), loads GrowthBook feature flags, loads plugins/skills, builds the tool and command registries, and finally calls `launchRepl()`.

## 2. The React Terminal UI

Claude Code is a **React app running in the terminal** via [Ink](https://github.com/vadimdemedes/ink). `launchRepl()` in **`replLauncher.tsx`** mounts:

```
<App>              ← AppStateProvider (Zustand store), MailboxProvider, VoiceProvider
  <REPL />         ← The main interactive screen
</App>
```

The **`REPL`** component in `screens/REPL.tsx` is the core UI. It manages:

- The text input area (with vim mode support)
- The message transcript display
- Permission request dialogs
- Tool output rendering
- Agent/task panels
- MCP server status

State flows through a **Zustand store** defined in `state/AppStateStore.ts` with 200+ fields. React components read state via `useAppState(selector)` and write via `useSetAppState()`. The store holds everything: messages, tool permissions, MCP connections, agent tracking, UI mode, tasks, etc.

## 3. The Conversation Loop

When you type a message and press Enter, the REPL calls into **`QueryEngine.ts`**, which orchestrates a full conversation turn:

```
User Input
  → QueryEngine.processUserInput()
    → Builds system prompt (modular, cached sections from constants/)
    → Loads CLAUDE.md files and memory files
    → Injects system context (git status, date) and user context
    → Calls query() in query.ts
```

**`query.ts`** is the agentic loop — it runs in an `async generator` that `yield`s stream events back to the UI. The core loop:

```
while (true) {
  1. Send messages to Anthropic API (via SDK in services/api/)
  2. Stream back response tokens → yield to UI for display
  3. If response contains tool_use blocks → execute tools
  4. If tools were executed → append results, continue loop
  5. If no tool calls → break (turn complete)

  // Also handles:
  - Auto-compaction when context gets too long
  - max_output_tokens recovery (retry up to 3 times)
  - Token budget tracking
  - Stop hooks
  - Command queue processing
}
```

## 4. Tool Execution

When the model responds with tool calls, **`services/tools/toolOrchestration.ts`** takes over. It **partitions tool calls into batches**:

- **Read-only tools** (Glob, Grep, FileRead) — run concurrently (up to 10 in parallel)
- **Write tools** (Bash, FileEdit, FileWrite) — run serially

Each tool call goes through the **permission system** before execution:

```
Tool call from model
  → useCanUseTool hook
    → hasPermissionsToUseTool() checks:
      1. Permission mode (default, auto, bypass)
      2. Allow/deny rules from settings
      3. Risk classification (LOW/MEDIUM/HIGH)
      4. Protected file checks
      5. Path traversal prevention
    → If "ask": show permission dialog in UI, wait for user
    → If "allow": proceed
    → If "deny": return error to model
  → tool.call() executes
  → Result appended to messages
```

In **auto mode**, a transcript classifier (ML-based) decides whether to auto-approve tool calls without prompting.

## 5. System Prompt Architecture

The system prompt isn't a single string — it's assembled from **modular sections** in `constants/`:

```
[Static sections - cached across orgs]
  - Base instructions (tone, tool usage rules, safety)
  - Cyber risk instruction (owned by Safeguards team)
  - Tool schemas

[SYSTEM_PROMPT_DYNAMIC_BOUNDARY marker]

[Dynamic sections - per user/session]
  - CLAUDE.md content (project instructions)
  - Memory files
  - Git status
  - Current date
  - Coordinator mode context (if enabled)
  - Undercover mode (if in public repo as ant user)
```

The boundary marker lets the API cache the static prefix and only reprocess the dynamic suffix when it changes.

## 6. Context Management

As conversations grow, Claude Code manages context through several mechanisms:

- **Auto-compaction** (`services/compact/autoCompact.ts`): When token count approaches the context window limit (minus a 13K buffer), it compacts the conversation into a summary and continues.
- **Reactive compaction**: When the API returns a `prompt_too_long` error, it compacts immediately and retries.
- **Token estimation** (`services/tokenEstimation.ts`): Fast local estimation to avoid API round-trips just for counting.
- **Tool result budgeting** (`utils/toolResultStorage.ts`): Large tool outputs get truncated/summarized.

## 7. Plugin and Skill System

**Plugins** (`plugins/`) are bundles of skills, hooks, and MCP servers. They're loaded at startup and can be enabled/disabled per user:

```
Plugin
  ├── skills (PromptCommands injected into conversation)
  ├── hooks (PreToolUse / PostToolUse / Stop event handlers)
  └── mcpServers (Model Context Protocol server configs)
```

**Skills** are slash commands that expand into conversation prompts. When you type `/commit`, it invokes the `SkillTool`, which loads the skill definition and injects its prompt into the conversation.

**Hooks** run at three points: before a tool executes (can block it), after a tool executes (can modify results), and when the model wants to stop (can force continuation).

## 8. MCP (Model Context Protocol)

`services/mcp/` manages connections to MCP servers — external processes that provide additional tools, resources, and prompts. At startup, Claude Code:

1. Reads MCP config from `.mcp.json` files
2. Connects to configured servers
3. Discovers their tools/resources
4. Merges MCP tools into the tool registry
5. MCP tool calls are proxied through `MCPTool`

## 9. Multi-Agent Systems

There are two levels of multi-agent orchestration:

**Sub-agents** (`tools/AgentTool/`): The model can spawn child agents via the `Agent` tool. Each sub-agent is a forked process with its own conversation context, tool set, and working directory. Results flow back to the parent.

**Coordinator mode** (`coordinator/`): When `CLAUDE_CODE_COORDINATOR_MODE=1`, Claude transforms into an orchestrator that manages multiple workers in phases: research → synthesis → implementation → verification. Workers communicate via `<task-notification>` XML messages and can share state through a scratchpad directory.

**Agent swarms** (feature-gated): Full team-based agents with color assignments, tmux/iTerm2 pane management, and team memory synchronization.

## 10. Memory and Persistence

- **Session storage** (`utils/sessionStorage.ts`): Conversations are persisted to JSONL files so they can be resumed with `/resume`.
- **Memory system** (`memdir/`): Persistent user/project memories stored in `~/.claude/projects/*/memory/`.
- **Dream system** (`services/autoDream/`): Background memory consolidation that runs as a forked sub-agent when three gates pass (24h since last dream + 5 sessions + lock acquired).
- **File history** (`utils/fileHistory.ts`): Snapshots of file states for undo capability.

## Data Flow Summary

```
User types message
  ↓
REPL component captures input
  ↓
QueryEngine builds context (system prompt + CLAUDE.md + memory + git status)
  ↓
query() sends to Anthropic API via SDK
  ↓
Stream response back → render in terminal via Ink
  ↓
If tool_use → permission check → execute (concurrent if read-only) → append result
  ↓
Loop back to API with tool results
  ↓
If context too long → auto-compact → continue
  ↓
Model responds with text (no tools) → turn complete
  ↓
Session persisted to disk, memory updated
```

## Key Architectural Patterns

- **Compile-time feature flags**: `feature('FLAG_NAME')` from `bun:bundle` enables dead-code elimination. Gated code is absent from external builds.
- **Runtime feature flags**: GrowthBook with `tengu_` prefix. Uses `getFeatureValue_CACHED_MAY_BE_STALE()` to avoid blocking the main loop.
- **Ant-only gating**: `process.env.USER_TYPE === 'ant'` gates Anthropic-internal features (staging API, debug tools, undercover mode).
- **Lazy imports**: Used to break circular dependencies (team tools, commands) and for conditional loading.
- **Memoization**: `lodash-es/memoize` used extensively for expensive operations (git status, context building), cached per conversation lifetime.
- **Parallel prefetching**: MDM settings, keychain reads, memory prefetch, and MCP connections all start before they're needed.
- **Async generators**: The conversation loop (`query.ts`) and tool execution use `async function*` generators to stream results back to the UI incrementally.
