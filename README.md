# Claude Code CLI — Source Architecture

This documents the architecture of the `src/` tree — the actual Claude Code CLI source: a terminal-native, tool-using AI coding agent built on Bun, React, and Ink. Everything below explains the real mechanisms in that code — how the tool-call loop works, why specific tools are built the way they are, how agents talk to each other, how context/memory/skills get loaded into a request, how streaming actually reaches the API, and how the query engine and terminal UI communicate. Every fact and file path was verified directly against `src/`, not copied from prior write-ups.

> `src/` is Anthropic's Claude Code CLI source. It surfaced publicly on 2026-03-31 and is not open-source — see [License & Legal Notice](#license--legal-notice) at the bottom for the exact terms this copy is held under.

## Table of Contents

- [At a Glance](#at-a-glance)
- [`src/` Directory Layout](#src-directory-layout)
- [Startup & Core Pipeline](#startup--core-pipeline)
- [The Query Engine and the Tool-Call Loop](#the-query-engine-and-the-tool-call-loop)
- [The Tool System](#the-tool-system)
- [Permission System](#permission-system)
- [Hooks System](#hooks-system)
- [BashTool: Implementation and Rationale](#bashtool-implementation-and-rationale)
- [FileEditTool: the Diff-Based Approach](#fileedittool-the-diff-based-approach)
- [Context Assembly: System Prompt, CLAUDE.md, Skills, Agents](#context-assembly-system-prompt-claudemd-skills-agents)
- [Extended Thinking and Effort](#extended-thinking-and-effort)
- [Multi-Agent Communication Model](#multi-agent-communication-model)
- [Settings & Session Persistence](#settings--session-persistence)
- [Protocol Inventory](#protocol-inventory)
- [Query Engine ↔ Terminal UI Communication](#query-engine--terminal-ui-communication)
- [UI Layer](#ui-layer)
- [Build System](#build-system)
- [Building `src/`](#building-src)
- [License & Legal Notice](#license--legal-notice)

---

## At a Glance

All figures below were counted directly against `src/` (`find`/`wc -l`/`ls`).

| | |
|---|---|
| **Language** | TypeScript (strict mode, ES modules) |
| **Runtime** | [Bun](https://bun.sh) |
| **Terminal UI** | React + [Ink](https://github.com/vadimdemedes/ink) |
| **CLI parser** | Commander.js (`@commander-js/extra-typings`) |
| **Validation** | Zod v4 |
| **Lint/format** | [Biome](https://biomejs.dev) |
| **Source files** | 1,916 `.ts`/`.tsx` files |
| **Total lines** | ~519,400 |
| **Top-level `src/` subdirectories** | 36 |
| **Agent tools** | 40 tool directories under `src/tools/` (43 entries total — the other 3 are `shared/` helper code, a test-only tool in `testing/`, and a `utils.ts` helper file) |
| **Slash commands** | 102 entries under `src/commands/` (directories + standalone files) |
| **UI components** | 111 top-level files under `src/components/` |
| **React hooks** | 83 under `src/hooks/` |

A note on accuracy: prior write-ups of this codebase state that `src/QueryEngine.ts` is ~46,000 lines, `src/Tool.ts` is ~29,000 lines, and `src/commands.ts` is ~25,000 lines. Those numbers are wrong by roughly 30x. Directly measured: `QueryEngine.ts` is **1,297 lines**, `Tool.ts` is **794 lines**, `commands.ts` is **758 lines**. This document uses only directly-verified numbers.

## `src/` Directory Layout

| Directory | What it holds |
|---|---|
| `tools/` | 40 agent tool directories (`BashTool`, `FileEditTool`, `AgentTool`, etc.), plus a test-only tool in `testing/`, non-tool `shared/` helper code, and a `utils.ts` helper file — see [The Tool System](#the-tool-system) |
| `commands/` | The ~100 slash commands (`/commit`, `/review`, `/mcp`, ...) |
| `components/` | 111 top-level Ink/React UI components |
| `hooks/` | 83 React hooks — permissions, input handling, IDE integration, sessions |
| `services/` | External integrations: `api/` (Anthropic SDK client), `mcp/`, `oauth/`, `lsp/`, `analytics/`, `plugins/`, `voiceStreamSTT.ts` |
| `bridge/` | The IDE/claude.ai bridge protocol (v1 WebSocket, v2 SSE+REST) |
| `state/` | The `AppState` store and its React binding |
| `context/` | React context providers — notifications, stats, FPS, modals, mailbox |
| `tasks/` | Background/parallel work: shell tasks, agent tasks, teammates, workflows |
| `skills/` | Skill discovery/loading (`.claude/skills/*/SKILL.md`) |
| `memdir/` | The agentic memory-directory subsystem (distinct from CLAUDE.md) |
| `coordinator/` | The built-in multi-agent coordinator/worker orchestration |
| `ink/` | The customized Ink terminal renderer this UI layer sits on |
| `screens/` | Full-screen UI modes — `REPL.tsx`, `Doctor.tsx`, `ResumeConversation.tsx` |
| `entrypoints/` | `init.ts`, `cli.tsx`, `mcp.ts`, `sdk/` — see [Startup & Core Pipeline](#startup--core-pipeline) |
| `constants/` | System prompt sections, API/media size limits, feature betas, error IDs, product strings — static config and data, no logic |
| `schemas/` | Currently a single file, `hooks.ts` — hook-config Zod schemas, deliberately extracted from `utils/settings/` to break an import cycle. (The settings/permissions/policy schemas actually live under `utils/settings/` and `utils/permissions/`, not here.) |
| `keybindings/` | A full chord-capable keybinding engine — parsing, context-aware resolution, per-platform default bindings, user customization |
| `migrations/` | One-shot, idempotent settings migrations — mostly rewriting deprecated model-alias IDs to current ones |
| `utils/` | By far the largest directory (564 files) — shell execution, diffing, git, the settings system (`utils/settings/`), the permission engine (`utils/permissions/`), auth, telemetry, and hundreds of cross-cutting helpers |
| `types/` | Shared TypeScript type definitions, broken out to avoid circular imports, plus a `generated/` folder of generated protobuf-style types |
| `query/` | Core query-loop plumbing: `config.ts` (per-call `QueryConfig`), `deps.ts` (dependency injection for `query()`), `tokenBudget.ts`, `stopHooks.ts` (Stop-event hook execution), `transitions.ts` (the loop's terminal/continue state machine) |
| `remote/` | `SessionsWebSocket.ts` and remote session management |
| `server/` | `server/web/pty-server.ts` — the standalone web-terminal server product |
| `upstreamproxy/` | The CONNECT-over-WebSocket egress relay for managed environments |
| `voice/` | Voice-mode feature flag/state |
| `vim/` | Vim-mode motions/operators/text-objects for input |
| `plugins/` | Plugin loader and bundled built-in plugins. A plugin can contribute commands, agents, skills, output-styles, hooks, MCP servers, and LSP servers — see [Hooks System](#hooks-system) for the hooks a plugin can register |
| `outputStyles/` | Output-style discovery/loading, same pattern as skills |
| `shims/` | `bun-bundle.ts`/`macro.ts`/`preload.ts` — the runtime shim layer implementing `bun:bundle` feature flags outside Bun's own bundler |
| `cli/` | `transports/` (`HybridTransport`/`SSETransport`/`ccrClient` — shared streaming infrastructure used by both the bridge and headless/SDK "print mode"), `handlers/` (per-subcommand CLI handlers), stdout/stdin plumbing |
| `native-ts/` | Pure-TypeScript reimplementations of performance-sensitive modules — `color-diff`, `file-index`, `yoga-layout` |
| `buddy/` | A companion/pet UI feature (sprites, rarities, stats) — cosmetic, not core to the agent loop |
| `bootstrap/` | Process-wide startup state — OTel provider handles, hook events, model usage counters |
| `assistant/` | Paginated session-history fetching against the Anthropic API |
| `moreright/` | A stub — the real implementation is internal-only; this file exists so external builds still typecheck |
| `types/generated`, `native-ts/*` | Generated/native-style helper modules referenced above |
| **Top-level files** (`src/*.ts`, `src/*.tsx`) | `main.tsx` (entry), `QueryEngine.ts`, `Tool.ts`, `tools.ts`, `commands.ts`, `context.ts`, `query.ts`, `Task.ts`, `tasks.ts`, `history.ts`, `cost-tracker.ts`, `setup.ts`, `replLauncher.tsx`, `interactiveHelpers.tsx`, `dialogLaunchers.tsx`, `ink.ts`, `costHook.ts`, `projectOnboardingState.ts` |

## Startup & Core Pipeline

```text
src/main.tsx → src/entrypoints/init.ts → src/entrypoints/cli.tsx → src/replLauncher.tsx → REPL
```

- **`src/main.tsx`** — the Commander.js entry point. Before heavy module imports run, it fires parallel prefetch side-effects: MDM (managed device policy) reads, OS Keychain access, and an API connection pre-warm, so the network round-trip for auth/config overlaps with module loading instead of happening serially after it.
- **`src/entrypoints/init.ts`** — config, telemetry, OAuth, and MDM policy initialization.
- **`src/entrypoints/cli.tsx`** — CLI session orchestration, the main path from a parsed command to a running REPL.
- **`src/entrypoints/mcp.ts`** — an alternate entry point that runs Claude Code *as* an MCP server instead of an interactive CLI (see [Protocol Inventory](#protocol-inventory)).
- **`src/entrypoints/sdk/`** — the Agent SDK, a programmatic (non-interactive) API for embedding Claude Code in other programs.
- **`src/replLauncher.tsx`** — launches the interactive REPL screen (`src/screens/REPL.tsx`), the default UI once the CLI is running.

## The Query Engine and the Tool-Call Loop

The core agent loop lives across two cooperating modules, not one giant file:

- **`src/query.ts`** — the actual driving loop: sends a request, streams the response, and when the model requests tool calls, executes them via `runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)` and feeds results back for the next turn.
- **`src/QueryEngine.ts`** — the SDK-facing layer wrapping that loop: exposes an `ask()` async generator, formats messages into the shape consumers expect, and tracks cost/token accounting.

Both are **async generators**, not callback- or event-emitter-based — `for await (const event of query(...))` is the actual consumption pattern used throughout the codebase (see [Query Engine ↔ Terminal UI Communication](#query-engine--terminal-ui-communication)).

### Streaming transport: SSE, not polling

The actual request to the model is made in `src/services/api/claude.ts` via the Anthropic SDK: `anthropic.beta.messages.create({ ...params, stream: true }, { signal, headers })`. This opens an HTTP connection whose response body is a `text/event-stream` (Server-Sent Events); the SDK parses it into discrete events consumed with `for await (const part of stream)`. The codebase's own error handling confirms this is the expected wire format — a bare-JSON, non-streaming response from a misbehaving proxy is explicitly treated as an anomaly (`// proxy returned 200 with non-SSE body`), not the normal case. Each event carries a `content_block_start`/`_delta`/`_stop` shape that the loop uses to incrementally build up text, `thinking`, and `tool_use` content blocks as they arrive — this is what makes token-by-token streaming to the terminal possible instead of waiting for a full response.

### Parallel tool calls

When one assistant turn requests multiple tool calls at once, they are not simply thrown at `Promise.all`. The real logic is in `src/services/tools/toolOrchestration.ts`:

1. **`partitionToolCalls`** walks the `tool_use` blocks from the turn in order. For each, it parses the input against the tool's Zod schema and calls `tool.isConcurrencySafe(parsedInput)`. This check is **fail-closed** — if parsing fails, or the tool doesn't explicitly declare itself safe, it's treated as unsafe.
2. Consecutive concurrency-safe calls are merged into one batch; any unsafe call becomes its own single-call batch. This preserves original ordering between unsafe calls and their neighbors while still letting safe calls run together.
3. A safe batch runs through a custom async-generator combinator, **`all()`** in `src/utils/generators.ts` — not `Promise.all` or `p-map`. It races a set of in-flight generator `.next()` promises up to a concurrency cap (`getMaxToolUseConcurrency()`, default 10, overridable via `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`), pulling in the next queued call as each slot frees up, and yields each tool's result as soon as it's ready.
4. Because results stream back independently, a batch's `tool_result` messages can complete **out of the original call order** — this is fine, because the Anthropic API only requires each `tool_result` to reference the correct `tool_use_id`, not to appear in submission order. Any tool result that would mutate shared conversation context is deferred and applied only after the whole batch finishes, in original block order, so concurrent read-only calls can't race on shared state.

The concrete effect: `BashTool.isConcurrencySafe` is defined as "true only if the command is read-only," so several read-only shell commands, `Read`s, or `Grep`s issued in one turn run in parallel — but `FileEditTool` never overrides `isConcurrencySafe`, so it inherits the fail-closed default and every edit runs strictly one at a time, even inside a batch that also contains safe calls.

### Retry, backoff, and context overflow

Two separate resilience mechanisms sit around the request/response cycle:

- **Retry with backoff** (`src/services/api/withRetry.ts`) wraps every API call as its own async generator, yielding `api_retry` progress messages (`attempt`, `maxRetries`, `retryInMs`) so the UI can show retry status live. `shouldRetry(error)` decides case by case: connection errors and HTTP 408/409 always retry; the response's `x-should-retry` header is honored; a 401 clears the cached API key and retries once; a 429 retries only for non-subscription/non-enterprise accounts (a subscriber 429 usually means an hours-long wait, not worth retrying); HTTP 529 ("overloaded") is retried even when the SDK drops the status code mid-stream and it has to be parsed back out of the raw error text. Backoff is exponential (`BASE_DELAY_MS * 2^(attempt-1)` plus up to 25% jitter), capped at `DEFAULT_MAX_RETRIES = 10` by default (`CLAUDE_CODE_MAX_RETRIES` overrides it), with a separate "persistent retry" mode that retries 429/529 indefinitely for long-running background sessions instead of giving up.
- **Context overflow** is handled two ways. A "prompt too long" (HTTP 413) response first triggers a lighter-weight strip retry (`truncateHeadForPTLRetry`) before falling back to a full compaction. Proactively, `src/services/compact/autoCompact.ts` triggers **auto-compact** once token usage crosses the model's context window minus a fixed buffer (`AUTOCOMPACT_BUFFER_TOKENS = 13,000`), with a circuit breaker (`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`) added after production data showed sessions looping 50+ consecutive failed compaction attempts. `compactConversation()` (`src/services/compact/compact.ts`) strips images/attachments from history, then runs a *sub-query* — thinking disabled, only `FileReadTool`/`ToolSearchTool` available — prompted to summarize the conversation, producing a boundary message the UI later renders collapsed. A lighter, time-based `microCompact.ts` handles smaller trims without a full summarization pass.

## The Tool System

Every capability the model can invoke — file I/O, shell, search, sub-agents, MCP calls — is a **tool**, built through one factory in `src/Tool.ts`.

### `buildTool()` and `TOOL_DEFAULTS`

`buildTool(def)` fills in a `TOOL_DEFAULTS` object with whatever the tool definition provides. The defaults are all deliberately on the *safe/restrictive* side — a tool has to explicitly opt out of caution:

| Field | Default | What it controls |
|---|---|---|
| `isConcurrencySafe()` | `false` | Whether this tool's calls can be batched with others in the same turn (see above) |
| `isReadOnly()` | `false` | Whether the tool can be assumed non-mutating (drives UI collapsing and is often reused as the `isConcurrencySafe` implementation) |
| `isDestructive()` | `false` | Set `true` only for irreversible operations (delete/overwrite/send) |
| `checkPermissions()` | allow, deferring to the general permission system | Whether this specific tool call needs its own permission logic beyond the standard prompt-rule matching |
| `toAutoClassifierInput()` | `''` (skip classifier) | Whether the tool is evaluated by the ML auto-permission classifier |

Fields every tool must supply itself: **`call(args, context, canUseTool, parentMessage, onProgress)`** — the actual execution, returning `{ data, newMessages?, contextModifier? }` (the last only honored for non-concurrency-safe tools); **`inputSchema`/`outputSchema`** (Zod — also used to generate the tool schema sent to the Anthropic API); **`prompt()`/`description()`** — the text describing the tool to the model; **`validateInput()`** — pre-permission validation; **`mapToolResultToToolResultBlockParam()`** — converts a tool's internal result into the actual API `tool_result` content block; a family of `renderToolUseMessage` / `renderToolUseProgressMessage` / `renderToolResultMessage` / `renderToolUseErrorMessage` / `renderToolUseRejectedMessage` React renderers, one per lifecycle stage the terminal UI needs to draw; and **`backfillObservableInput()`** — mutates a *copy* of the input for hooks/transcripts/observers, leaving the original API-bound input (and its prompt-cache key) untouched.

### Registration and filtering (`src/tools.ts`)

The full tool list is assembled in three layers:

1. **`getAllBaseTools()`** — the exhaustive list of every tool that could exist in the current build, gated per-entry by `feature('FLAG_NAME')` (Bun dead-code-elimination flags) or `process.env` checks. For example, `REPLTool` and `ConfigTool` only exist when `process.env.USER_TYPE === 'ant'` (Anthropic-internal builds); `SleepTool` requires `feature('PROACTIVE')` or `feature('KAIROS')`; the cron tools require `feature('AGENT_TRIGGERS')`.
2. **`getTools(permissionContext)`** — filters that list by permission deny-rules (a blanket deny for a tool name removes it from what the model even sees, not just what it's allowed to call) and, when REPL mode is active, hides the raw primitive tools that REPL wraps internally.
3. **`assembleToolPool(permissionContext, mcpTools)`** — merges in tools discovered from connected MCP servers, then sorts *each partition* (built-ins, MCP tools) by name before concatenating them. This isn't cosmetic: the system prompt's cache breakpoint sits right after the built-in tool block, so a flat interleaved sort would shuffle an MCP tool into the middle of the built-ins and invalidate the shared prompt cache for every request that follows.

### Why this tool, not that one

The actual decision rules the model is given, pulled from each tool's real `prompt.ts`:

- **Reading/finding things**: a known file path → `FileReadTool`. Finding a file by name → `GlobTool`. Finding a class/content match inside files → `GrepTool`. An open-ended, multi-step task → `AgentTool`. (Anthropic-internal builds that embed `bfs`/`ugrep` in the binary swap `Glob`/`Grep` for `find`/`grep` via `BashTool` instead, since the dedicated tools become redundant.)
- **Shell**: `BashTool` by default; `PowerShellTool` on Windows when enabled; `REPLTool` only in Anthropic-internal builds.
- **Editing**: `FileEditTool` for targeted, verified string replacement; `FileWriteTool` for a full overwrite or brand-new file; `NotebookEditTool` specifically for Jupyter cells.
- **Web**: `WebFetchTool` for a known URL; `WebSearchTool` for an open-ended lookup.

## Permission System

Source: `src/types/permissions.ts`, `src/utils/permissions/` (`permissions.ts` is 1,487 lines — this is a real, heavily-engineered decision engine, not a side detail of `checkPermissions()`).

- **Modes**: `default` (prompt per potentially-destructive call), `plan` (show the plan, ask once), `acceptEdits`, `bypassPermissions` (auto-approve — dangerous), `dontAsk`, and an Anthropic-internal-only `auto` mode where an ML classifier decides, gated behind its own feature flag.
- **Rule syntax**: strings like `Bash(git *)` are parsed by `permissionRuleValueFromString` into a `{toolName, ruleContent}` pair, with careful escaping since the rule content can itself contain parentheses. Matching supports three shapes — `exact`, a legacy `prefix` form (`npm:*`), and `wildcard` (`git *`, where `*` becomes `.*` in a generated regex, with a special case so a single trailing wildcard also matches the bare command with no arguments).
- **The decision cascade** (`checkRuleBasedPermissions()`): a tool-wide deny rule wins immediately; then a tool-wide ask rule (with a carve-out when the command will run sandboxed anyway); then the tool's own `checkPermissions()` (e.g. `BashTool`'s own subcommand-level logic); then tool-specific deny rules; then content-specific ask rules; and finally a set of **"safety checks"** — paths like `.git/`, `.claude/`, `.vscode/`, and shell config files — that are explicitly **bypass-immune**: they still prompt even under `bypassPermissions` mode or a hook that already approved the call.
- Auto-mode's classifier machinery (`bashClassifier.ts`, `autoModeState.ts`, `denialTracking.ts`) is a real subsystem internally and a no-op stub in external builds.

## Hooks System

Source: `src/schemas/hooks.ts`, `src/utils/hooks.ts`, `src/utils/hooks/`, `src/commands/hooks/` — not to be confused with the React hooks in `src/hooks/`. These are user-configurable lifecycle callbacks, invoked by name at specific points in the agent loop.

- **Events**: `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `Notification`, `UserPromptSubmit`, `SessionStart`, `SessionEnd`, `Stop`, `StopFailure`, `SubagentStart`, `SubagentStop`, `PreCompact`, `PostCompact`, `PermissionRequest`, `PermissionDenied`, `Setup`, `TeammateIdle`, `TaskCreated`, `TaskCompleted`, `Elicitation`, and more.
- **Four hook types**, not just shell scripts: `command` (a shell script, with `timeout` and an `async`/`asyncRewake` mode that runs in the background and can "wake" the model by exiting with code 2), `prompt` (LLM-evaluated, with an `$ARGUMENTS` placeholder and its own configurable model), `http` (POSTs the hook's input JSON to a URL, with an explicit allowlist for which environment variables can be interpolated into headers), and `agent` (a full agentic "verifier" sub-run with its own model/timeout — used, for example, by the plan-verification tool).
- **Scoping**: every hook can carry an `if` field using the exact same permission-rule syntax as above (e.g. `Bash(git *)`), evaluated against the tool name/input before the hook fires — so a hook can be scoped to only trigger for specific tool calls.
- **What a hook can actually do**: its stdout is parsed as JSON. A `decision: "approve"|"block"` or `hookSpecificOutput.permissionDecision: "allow"|"deny"|"ask"` can **override the tool's own permission result**; `updatedInput` can rewrite the tool call's arguments before it runs; `additionalContext` injects text into the conversation. This is why hooks sit logically next to the permission system above — they're a second, user-scriptable layer over the same decision point.
- Skills, agents, and plugins can each declare their own hooks (`registerSkillHooks.ts`, `registerFrontmatterHooks.ts`), and the interactive `/hooks` command (`src/commands/hooks/hooks.tsx`) provides a full TUI for configuring them.

## BashTool: Implementation and Rationale

Source: `src/tools/BashTool/BashTool.tsx`, `src/utils/Shell.ts`, `src/utils/ShellCommand.ts`, `src/utils/bash/`.

- **A fresh shell process per command.** `BashTool` does not keep one persistent interactive shell alive. Each call spawns a new process via `child_process.spawn`, using a `ShellProvider` abstraction that knows how to build a `<shell> -c '<command>'` invocation for bash/zsh (or PowerShell, on a separate provider). This avoids the real complexity of interactive-shell prompt-detection and output-framing that a long-lived pty session requires.
- **Faked working-directory continuity.** Since each command is a fresh process, "cd persists across commands" is simulated: after every command, bash is made to also run `pwd -P` into a temp file, which is read back to seed the *next* spawned shell's working directory. The result feels like a persistent session without actually being one.
- **Streaming output, two ways.** Normal commands run in "file mode" — stdout and stderr are both written by the OS directly into the same append-mode file descriptor, with no per-byte JavaScript involvement, while a poller tails that file to compute progress. A "pipe mode" instead attaches a stream wrapper to `stdout`/`stderr` for callers (like hooks) that need real-time data events.
- **Timeouts and backgrounding.** A configurable timeout (30 minutes by default) either force-kills the process tree (`tree-kill`, `SIGTERM`/`SIGKILL`) or, if auto-backgrounding is enabled, promotes the command to a background task instead of killing it. There are three separate paths into "background": an explicit `run_in_background: true` input, timeout-triggered auto-backgrounding, and a ~15-second "assistant blocking budget" (`ASSISTANT_BLOCKING_BUDGET_MS`) that force-backgrounds a still-running foreground command so the agent loop isn't stalled waiting on it.
- **Permission-aware parsing.** Permission checks use an AST-based command parser (`src/utils/bash/ast.ts`) that splits compound/piped commands (`ls && git push`) into subcommands, so a rule like `Bash(git *)` and any hooks correctly evaluate each piece of a chained command instead of only the first token. Sandboxing (wrapping the command for OS-level isolation) is a separate, composable step layered on top.

**Why this design**: generator-based execution that yields progress lets very large command output stream to the model/UI without holding it all in memory, and lets a command be transparently promoted to background at nearly any point in its lifecycle — both hard to get right with a naive `exec()`-and-wait approach.

## FileEditTool: the Diff-Based Approach

Source: `src/tools/FileEditTool/FileEditTool.ts`, `src/tools/FileEditTool/utils.ts`, `src/utils/diff.ts`.

- **Edits are located by unique substring match, not line numbers.** `old_string` must be found verbatim in the file (`findActualString()`, with a fallback that normalizes curly vs. straight quotes so minor rendering differences don't block a match). If the string matches more than once, the edit is **rejected** unless the caller explicitly sets `replace_all` — a deliberate anti-ambiguity guard rather than a silent "first match wins."
- **Guardrails**: an edit where `old_string === new_string` is rejected outright as a no-op. A file must have been read in the current session before it can be edited, and the edit is rejected if the file changed on disk since that read — preventing the tool from blindly clobbering a file that was modified out from under it.
- **The edit itself is a plain string replace.** `applyEditToFile()` does a straightforward `.replace()`/`.replaceAll()` on the file content. The **diff shown to the user is computed afterward**, as a separate step, by running `structuredPatch()` from the npm `diff` package over the resulting before/after full file contents (`getPatchFromContents` in `src/utils/diff.ts`) — the diff is a display and bookkeeping artifact, decoupled from how the edit was actually located and applied.
- **Why substring match instead of line-number patching**: line numbers are fragile — they shift the instant any earlier edit in the same turn (or any concurrent external change) touches the file, and they require the model to have counted lines perfectly from a prior read, which LLMs are unreliable at. A unique-substring match is self-verifying: at apply time, the tool can directly confirm whether the exact context the model believed it was editing still exists, independent of anything about line positions.

## Context Assembly: System Prompt, CLAUDE.md, Skills, Agents

How the model actually receives instructions, project memory, skills, and subagent definitions on every turn.

### System prompt construction

`src/constants/prompts.ts` (`getSystemPrompt`) builds the system prompt as an array of strings: a fixed static block (tone, output-style rules, tool-usage guidance) followed by a `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` marker, then dynamic sections — memory, environment info, output style, MCP server instructions — each computed by `systemPromptSection(name, computeFn)` in `src/constants/systemPromptSections.ts` and memoized per conversation so they're recomputed only once per session (the cache is cleared on `/clear` and `/compact`). `src/utils/systemPrompt.ts` (`buildEffectiveSystemPrompt`) then picks the final prompt based on context — an explicit override, coordinator mode, a subagent's own system prompt, a custom `--system-prompt` flag, or the default array. **Tool descriptions/schemas are not text-concatenated into this prompt at all** — they're sent to the Anthropic API as a wholly separate `tools` parameter.

### Prompt caching

`splitSysPromptPrefix` (`src/utils/api.ts`) splits the assembled prompt into cache-scoped Anthropic content blocks: static, user-independent content (tone/tool-usage rules) gets a `global` cache scope that can be shared across *different users'* requests on Anthropic's infrastructure, while everything after the dynamic boundary (memory, MCP state) is excluded from that shared cache since it's genuinely per-session. This is why tool registration order and system-prompt section ordering matter — anything that shuffles the static prefix invalidates the shared cache for every request that follows.

### CLAUDE.md loading

`src/utils/claudemd.ts` documents and implements a strict discovery order: **managed** (`/etc/claude-code/CLAUDE.md`) → **user** (`~/.claude/CLAUDE.md`) → **project** (`CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md`, discovered by walking from the working directory up toward the filesystem root) → **local** (`CLAUDE.local.md`, typically gitignored). Files are loaded so that the one closest to the current working directory is effectively given the highest attention weight. Memory files support `@path` / `@~/path` / `@/abs/path` include directives, recursively expanded with cycle detection, and `.claude/rules/*.md` files can carry frontmatter `paths:` globs that scope a rule to only apply when matching files are touched.

Critically, this content is **not** spliced into the system prompt string array — it's delivered as a synthetic **leading user message**, wrapped in a `<system-reminder>` block (`prependUserContext` in `src/utils/api.ts`), sent fresh on every turn. This is a different mechanism from `loadMemoryPrompt()` in `src/memdir/`, which builds a separate "agentic memory directory" (notes the model itself writes and reads back across sessions) that *is* folded in as one of the system-prompt sections above.

### Skill.md loading

`src/skills/loadSkillsDir.ts` discovers skills from managed, user, and project `.claude/skills/*/SKILL.md` directories (plus any `--add-dir` paths and a legacy flat-file loader), parsing each file's frontmatter for `description`, `when_to_use`, `allowed-tools`, `model`, and `paths` (which makes a skill "conditional" — held back until a file matching its glob is actually touched, then dynamically activated). Skills are exposed to the model only as a **compact name + one-line-description listing** inside the `SkillTool`'s own prompt, budget-capped to roughly 1% of the context window (`SKILL_BUDGET_CONTEXT_PERCENT = 0.01`) — the full skill body is fetched as a tool result only when the model actually invokes that skill by name. Unused skills therefore cost the conversation almost nothing in tokens, no matter how many are installed.

### agent.md loading (custom subagents)

`src/tools/AgentTool/loadAgentsDir.ts` parses `.claude/agents/*.md` files into an `AgentDefinition`: `name` (the `agentType`), `description` (`whenToUse` — shown to the parent agent so it can decide when to launch this subagent), `tools`/`disallowedTools`, `model`, `isolation` (`worktree`/`remote`), `permissionMode`, and more. These are unified with JSON-defined agents (from settings) and the codebase's built-in agents (`src/tools/AgentTool/builtInAgents.ts` — `general-purpose`, `Explore`, `Plan`, a statusline-setup agent, a Claude Code guide agent, and a conditionally-enabled verification agent) into one list, keyed by `agentType` with later entries overwriting earlier ones in this exact order: built-in → plugin → user → project → CLI flag → managed/policy settings — so managed settings always win a name collision, and built-ins always lose one.

## Extended Thinking and Effort

Source: `src/utils/thinking.ts`, `src/utils/effort.ts`, `src/services/api/claude.ts`.

Thinking and effort are **two independent request parameters**, easy to conflate but controlling different things:

- **Thinking** is the model's visible reasoning budget. Config is one of `{type:'adaptive'}` (no fixed budget — modern models manage it themselves), `{type:'enabled', budgetTokens}` (an explicit token budget), or `{type:'disabled'}`. The default is chosen by `shouldEnableThinkingByDefault()` unless explicitly overridden, and both the ability to think at all and adaptive mode specifically are gated per-model (`modelSupportsThinking`, `modelSupportsAdaptiveThinking`).
- **Effort** (surfaced via the `/effort` command as `low`/`medium`/`high`/`max`) is a separate `output_config.effort` request field controlling how much overall effort the model puts in — independent of how large its reasoning-token budget is.

On the wire, thinking content arrives as its own SSE content-block type, distinct from normal text: `content_block_start` with `type: 'thinking'` opens a block, and subsequent `thinking_delta` events append to it — kept structurally separate from `text_delta` events the whole time. Each thinking block carries a tamper-evident `signature` that must be echoed back verbatim in later turns of the same tool-call loop, so the API can verify the model's prior reasoning wasn't altered before it continues from it. This separation is also exactly why the terminal UI can render thinking as a distinct, dimmed/collapsible block instead of mixing it into the final answer text.

There's also an **"ultrathink" keyword trigger** (`isUltrathinkEnabled()`, `hasUltrathinkKeyword()` in `src/utils/thinking.ts`): gated behind both a build-time `feature('ULTRATHINK')` flag and a runtime GrowthBook rollout flag, the literal word "ultrathink" typed in a user message is detected and used to bump the thinking budget for that turn — a keyword-driven override layered on top of the default/explicit thinking config described above.

## Multi-Agent Communication Model

How the main session, subagents, and teammates actually talk to each other. Source: `src/tools/AgentTool/`, `src/tools/SendMessageTool/`, `src/tools/TeamCreateTool/`, `src/tasks/types.ts`.

### Spawning agents

`AgentTool` launches work in one of two shapes: a **subagent** with a chosen `subagent_type`, which starts with an entirely fresh context, or — where fork mode is enabled — a **fork** of the caller itself (no `subagent_type`), which inherits the caller's full conversation context and shares its prompt cache. `run_in_background: true` returns immediately with a later completion notification rather than being polled for. `isolation: "worktree"` runs the agent against an isolated git worktree (cleaned up automatically if it made no changes); `isolation: "remote"` (Anthropic-internal) runs it in a remote sandboxed environment.

### The only channel between agents

An agent's plain assistant text output is **invisible to other agents** — the sole way to communicate across agents is calling `SendMessageTool`. Valid targets: a teammate by name, `"*"` to broadcast to the whole team (deliberately expensive — cost is linear in team size, so it's reserved for messages everyone genuinely needs), or, when the UDS inbox subsystem is enabled, cross-session peers addressed as `uds:/path/to.sock` or `bridge:session_id` (discovered via a `ListPeers` tool). The same tool also carries a small legacy structured protocol: `shutdown_request`/`shutdown_response` and `plan_approval_request`/`plan_approval_response` messages, matched by echoing a `request_id`.

### Teams

`TeamCreateTool` creates a team with a strict 1:1 relationship to a task list — a team is, functionally, just a shared task list plus a roster. Creating one writes `~/.claude/teams/{team-name}/config.json` and a matching `~/.claude/tasks/{team-name}/` directory. The intended workflow: create the team → create tasks with `TaskCreateTool` (listed/updated/fetched via `TaskListTool`/`TaskUpdateTool`/`TaskGetTool`) → spawn teammates via `AgentTool` with a `team_name` → each teammate claims an unowned, unblocked task via `TaskUpdate(owner=<name>)`, preferring the lowest task ID (earlier tasks often set up context later ones depend on) → teammates go idle between turns — idle does **not** mean done, it means waiting for input, and a teammate messaging you and then going idle is the normal flow, not an error — → the team lead gracefully shuts teammates down by sending `{type: "shutdown_request"}` via `SendMessage`.

### Task backends

`src/tasks/types.ts` unifies several different execution backends under one `TaskState` type, so the rest of the system (background-task indicators, task lists) doesn't need to know which kind of task it's looking at: `LocalShellTask` (a backgrounded shell command), `LocalAgentTask` (a local subagent), `RemoteAgentTask` (an agent on a remote/CCR sandbox), `InProcessTeammateTask` (a teammate running in-process), `LocalWorkflowTask`, `MonitorMcpTask`, and `DreamTask` (a background "dreaming"/ideation process).

### Coordinator mode

`src/coordinator/coordinatorMode.ts`, gated by `feature('COORDINATOR_MODE')`, is the codebase's own built-in orchestrator pattern — a coordinator agent that manages a pool of worker agents — built entirely on top of the `AgentTool`/`SendMessageTool`/task primitives described above, not a separate mechanism.

## Settings & Session Persistence

### Settings

Source: `src/utils/settings/`. Five sources merge low-to-high priority: `userSettings` → `projectSettings` → `localSettings` → `flagSettings` (CLI-provided) → `policySettings` (managed/MDM). Managed settings have their own internal precedence (a remote-pushed policy beats a local registry/plist value, which beats a `managed-settings.json` file), and file-based managed settings support drop-ins (`managed-settings.d/*.json`, merged alphabetically, later wins) so an organization can ship independent policy fragments instead of one monolithic file. A file watcher (`changeDetector.ts`) fans out change events per source so the running session picks up edits without a restart.

### Session persistence & resume

Source: `src/utils/sessionStorage.ts` (one of the largest files in the codebase), `sessionRestore.ts`, `crossProjectResume.ts`, `listSessionsImpl.ts`. Every conversation is an append-only JSONL transcript, one file per session, stored under a per-project directory keyed by a hash of the project path; a subagent spawned mid-session gets its own sidechain transcript file plus a metadata sidecar. `--resume`/`--continue` rehydrate conversation, tool-permission, and worktree state from that log. `crossProjectResume.ts` specifically handles resuming a session that was logged under a *different* working directory — same-repo-different-worktree is resumed in place, while a genuinely different project instead hands back a `cd <path> && claude --resume <id>` command rather than silently resuming somewhere unexpected.

## Protocol Inventory

Every distinct network/IPC protocol actually used in `src/`, verified against real source rather than assumed:

| Protocol | Files | Purpose |
|---|---|---|
| Anthropic API (HTTPS, SSE streaming) | `src/services/api/client.ts`, `claude.ts` | Model inference calls — `stream: true` over a real `text/event-stream` connection, see [Streaming transport](#streaming-transport-sse-not-polling) |
| MCP client | `src/services/mcp/client.ts` | Connecting to external MCP servers as a client — supports stdio (`StdioClientTransport`), Streamable HTTP, legacy SSE, a custom WebSocket transport, and an in-process linked-pair transport for built-in servers that run in the same process |
| MCP server | `src/entrypoints/mcp.ts` | Running Claude Code itself as an MCP server over stdio (`claude mcp serve`), exposing its own tools to another client |
| OAuth 2.0 | `src/services/oauth/` | Interactive login/token refresh; includes a localhost HTTP server that only captures the browser's redirect `code`/`state` — explicitly not a real OAuth server, just a redirect-capture mechanism |
| Bridge v1 | `src/bridge/replBridgeTransport.ts` | WebSocket reads + HTTP POST writes to "Session-Ingress" — relays a local session to a remote controller (IDE extension, claude.ai web UI) |
| Bridge v2 | `src/bridge/replBridgeTransport.ts` | SSE reads + REST writes via a `CCRClient` — the newer, env-less bridge transport generation |
| JWT decode | `src/bridge/jwtUtils.ts` | Decoding (not cryptographically verifying) session-ingress token expiry, for proactive refresh scheduling |
| Remote session WebSocket | `src/remote/SessionsWebSocket.ts` | Full-duplex SDK-message channel for remote-controlled sessions — a separate protocol from the bridge above, with its own reconnect/backoff logic |
| UDS inbox | wired in `src/setup.ts`, gated by `feature('UDS_INBOX')` | Local peer-to-peer messaging between concurrently running Claude Code processes/swarm teammates on the same machine, over a Unix domain socket |
| File-based teammate mailbox | `src/utils/teammateMailbox.ts` | Lockfile-based JSON inbox files, a lower-tech fallback/complement to the UDS inbox for agent-swarm messaging |
| LSP | `src/services/lsp/LSPClient.ts` | JSON-RPC over stdio to real language servers, powering diagnostics/hover/go-to-definition-style tooling |
| First-party telemetry | `src/services/analytics/` | Anthropic's own HTTPS event logging (`firstPartyEventLogger.ts`) plus Datadog (`datadog.ts`) and GrowthBook feature-flag/A-B-test evaluation (`growthbook.ts`) |
| OpenTelemetry (traces/logs/metrics) | `src/utils/telemetry/instrumentation.ts`, `sessionTracing.ts` | A real OTel SDK setup — `@opentelemetry/sdk-trace-base`/`sdk-logs`/`sdk-metrics` with OTLP (gRPC/HTTP) and Prometheus exporters, all dynamically `import()`ed per configured protocol so the ~1.2MB of exporter code isn't loaded on every startup; session-level spans layered on top in `sessionTracing.ts` |
| Chrome native messaging | `src/utils/claudeInChrome/chromeNativeHost.ts` | Bridges the Chrome extension's native-messaging stdio protocol to an MCP client, over a Unix domain socket |
| Voice streaming WebSocket | `src/services/voiceStreamSTT.ts` | Push-to-talk speech-to-text against an Anthropic streaming endpoint (Anthropic-internal builds only) |
| Upstreamproxy relay | `src/upstreamproxy/relay.ts` | HTTP `CONNECT` traffic tunneled over WebSocket to an org-controlled egress gateway, for managed/sandboxed environments |
| Web PTY server | `src/server/web/pty-server.ts` | A separate, always-on web-terminal product surface (Express + `ws` + `node-pty`) that spawns the `claude` binary itself per browser session |

Telemetry is layered: first-party event logging is the primary usage-analytics path, and OpenTelemetry (row above) is a separate, real tracing/metrics pipeline sitting alongside it — both genuinely present in `src/`.

## Query Engine ↔ Terminal UI Communication

How results from the model/tool loop actually reach what's drawn in the terminal.

- `src/query.ts` / `src/QueryEngine.ts` are **async generators and are UI-agnostic** — nothing in them imports React or touches application state directly. They `yield` a stream of typed messages: `assistant`, `user`, `progress` (tool-call progress), `tool_use_summary`, `tombstone` (retracts a previously shown message), and others.
- `src/screens/REPL.tsx` is what actually drives them: `for await (const event of query(...)) { onQueryEvent(event) }`. `onQueryEvent` delegates to `handleMessageFromStream` (`src/utils/messages.ts`) — a pure function that pattern-matches on the event type and calls a set of injected callbacks. REPL.tsx wires each callback to a plain React `useState` setter. **The visible message list is local component state in `REPL.tsx`**, not part of the global app store.
- The separate, global **`AppState`** store (`src/state/AppStateStore.ts`, `src/state/store.ts`) is a small hand-rolled subscribe/notify store — no external state-management library. Components read it via `useAppState(selector)`, built on React's `useSyncExternalStore`, so a state change only re-renders components whose *selected slice* actually changed. `src/state/onChangeAppState.ts` runs non-rendering side effects on every transition (e.g. notifying the bridge layer when the permission mode changes) — it doesn't itself cause re-renders; `useSyncExternalStore` does that.
- **A single tool call's lifecycle**: `Tool.call(args, context, canUseTool, parentMessage, onProgress)` takes an `onProgress` callback that a tool can invoke repeatedly while it's running — `BashTool`, for example, reports incremental output roughly once per second. Each `onProgress` call becomes a `progress`-typed message flowing through the exact same stream → `handleMessageFromStream` → `setMessages` path as any other message, with the UI special-casing rapid-fire progress entries to overwrite the previous one in place instead of appending, so the message list doesn't grow unbounded during a long-running command.
- **Rendering** is done by the tool's own pure functions — `renderToolUseMessage` (drawn as soon as input has streamed in, even partially), `renderToolUseProgressMessage` (live in-progress UI), and `renderToolResultMessage` (final result) — invoked from `src/components/Message.tsx`. The overall transcript is assembled and grouped (filtering ephemeral progress entries out of the main scroll, handling compact-conversation boundaries) in `src/components/Messages.tsx`, then handed row-by-row to `src/components/MessageRow.tsx`.

## UI Layer

- **Components** (`src/components/`, 111 top-level files) — functional React components built on Ink primitives (`Box`, `Text`, `useInput()`), styled with Chalk, with a design-system layer in `src/components/design-system/`.
- **Screens** (`src/screens/`) — full-screen modes: `REPL.tsx` (the default interactive screen), `Doctor.tsx` (`/doctor` diagnostics), `ResumeConversation.tsx` (`/resume`).
- **Hooks** (`src/hooks/`, 83) — permission hooks (`useCanUseTool`, `src/hooks/toolPermission/`), IDE integration, input handling (`useTextInput`, `useVimInput`), session management, and notification hooks.
- **`src/ink/`** — the vendored/customized Ink renderer wrapper itself (layout, terminal I/O, ANSI handling) that the whole UI layer sits on.

See [Query Engine ↔ Terminal UI Communication](#query-engine--terminal-ui-communication) above for how this layer actually receives its data — this section intentionally doesn't repeat that.

## Build System

- **Bun runtime**: native JSX/TSX execution without a separate transpile step; ES modules with `.js` extensions on `.ts` files (Bun convention).
- **`bun:bundle` feature flags** enable dead-code elimination at build time:

  ```typescript
  import { feature } from 'bun:bundle'

  if (feature('VOICE_MODE')) {
    // Stripped entirely from the build when VOICE_MODE is off
  }
  ```

  This same mechanism gates most of the subsystems described above (`BRIDGE_MODE`, `COORDINATOR_MODE`, `UDS_INBOX`, `AGENT_TRIGGERS`, `PROACTIVE`/`KAIROS`, and more), alongside `process.env.USER_TYPE === 'ant'` checks for Anthropic-internal-only code paths. Outside Bun's own bundler, `src/shims/bun-bundle.ts` provides a runtime implementation of `feature()` so the same code still runs.
- **Biome** handles both linting and formatting (`biome.json`, applies to `src/`).
- **`Dockerfile`** (repo root) is a multi-stage build: a `bun install` + `bun run build:prod` builder stage, then a minimal runtime stage that copies out only the bundled `dist/cli.mjs`.

## Building `src/`

```bash
bun install
bun run typecheck   # tsc --noEmit
bun run lint         # biome check src/
bun run build        # bundle to a runnable file via esbuild
```

```bash
docker build -t claude-code .
docker run --rm -e ANTHROPIC_API_KEY=sk-... claude-code -p "hello"
```

## License & Legal Notice

`src/` is Anthropic's Claude Code CLI source, published here strictly for educational and research purposes. It is **not** open-source — Anthropic has not released it under any permissive or copyleft license, and this repository is marked **UNLICENSED — NOT FOR REDISTRIBUTION**. Use at your own legal risk. See [`LICENSE`](LICENSE).

For the official, supported Claude Code CLI, see [Anthropic's Claude Code documentation](https://docs.anthropic.com/en/docs/claude-code).
