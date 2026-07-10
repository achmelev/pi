# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Start here: AGENTS.md

**`AGENTS.md` at the repo root is the canonical, binding rule set for agents working in this repo** (conversational style, code quality, git workflow, testing, releasing, changelog format). Read it before making changes — this file only adds architecture context that AGENTS.md doesn't cover. If anything below conflicts with AGENTS.md, AGENTS.md wins.

Key commands (see AGENTS.md for the full list and rationale):

```bash
npm install --ignore-scripts   # install deps
npm run build                  # build all packages (tui -> ai -> agent -> coding-agent -> orchestrator)
npm run check                  # biome + pinned-deps + ts-imports + shrinkwrap + tsgo --noEmit; run after every code change
./test.sh                      # run tests with no API keys/auth present (skips LLM-dependent tests)
./pi-test.sh                   # run pi from source (any cwd); add --no-env to strip provider credentials
```

Run a single test from a package root instead of the full vitest suite (which activates paid e2e tests when credentials are present):

```bash
node ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts
```

## Repo shape

This is an npm workspace monorepo (`packages/*`) published in lockstep (one version across all packages). Dependency direction:

```
pi-tui  ─┐
pi-ai   ─┼─► pi-agent-core ─► pi-coding-agent (CLI: `pi`)
         ┘                        ▲
                          pi-orchestrator (experimental, drives coding-agent via RPC)
```

- **`packages/ai`** (`@earendil-works/pi-ai`) — unified multi-provider LLM API (Anthropic, OpenAI, Google, Bedrock, OpenRouter, ~25 more). No agent/tool-loop logic here, just: model catalogs, auth resolution, streaming, cost/token tracking.
- **`packages/agent`** (`@earendil-works/pi-agent-core`) — provider-agnostic stateful agent runtime: the tool-calling loop, event streaming, and the "harness" (system prompt assembly, compaction, skills, session state) built on top of `pi-ai`. Has no CLI, TUI, or filesystem tools of its own.
- **`packages/coding-agent`** (`@earendil-works/pi-coding-agent`) — the `pi` CLI. Wires `pi-agent-core` + `pi-tui` together with concrete coding tools (`read`, `write`, `edit`, `bash`), session persistence, extensions/skills/prompt-templates/themes, and the interactive/RPC/SDK entry points.
- **`packages/tui`** (`@earendil-works/pi-tui`) — standalone terminal UI framework (differential rendering, components) with no knowledge of agents or LLMs.
- **`packages/orchestrator`** (`@earendil-works/pi-orchestrator`) — experimental; supervises multiple `pi` RPC-mode processes over an IPC protocol. Unstable API, may change or be removed.

## `pi-ai` architecture

- **Provider factories**: each provider lives in `src/providers/<name>.ts` (+ `<name>.models.ts` for the static model catalog). `src/providers/all.ts` registers every built-in provider; apps that care about bundle size import individual provider factories instead of `builtinModels()`.
- **Lazy API modules**: heavy per-provider request/response wiring lives in `src/api/<name>.ts` with a `<name>.lazy.ts` companion (see `src/api/lazy.ts`) so unused providers tree-shake out of bundles.
- **`models.generated.ts` / `image-models.generated.ts` are generated files.** Never hand-edit them — update `packages/ai/scripts/generate-models.ts` and regenerate. A diff in the generated file from an unrelated regeneration is expected and fine to include.
- **Auth resolution**: `src/auth/` + `src/env-api-keys.ts` resolve credentials from the credential store, OAuth (see `src/oauth.ts`, `src/utils/oauth/`), or environment variables, in that order.
- A **faux provider** (`src/providers/faux.ts`) exists for deterministic tests without real API calls.
- Context (`Context`/`AgentMessage[]`) is designed to serialize and hand off between different provider models mid-session.

## `pi-agent-core` architecture

- **`Agent` / `agent-loop.ts`**: the core prompt → LLM stream → tool-call → tool-result → next-turn loop. Tool execution mode is `parallel` (default: preflight sequentially, execute concurrently, but persist `toolResult` messages in assistant source order) or `sequential`, configurable globally or per-tool.
- **`AgentMessage` vs LLM `Message`**: `AgentMessage` is the app-facing type (extensible via declaration merging for custom/UI-only message types). `convertToLlm()` filters/converts it down to the `user`/`assistant`/`toolResult` messages an LLM understands; an optional `transformContext()` runs first to prune/inject context. See `packages/agent/README.md` for the full event sequence (`agent_start` → `turn_start` → `message_start/update/end` → `tool_execution_*` → `turn_end` → `agent_end`).
- **`src/harness/`**: the higher-level assembly on top of the raw loop — system prompt construction, compaction (`harness/compaction/`), skills, session state (`harness/session/`) — that `pi-coding-agent` builds on rather than reimplementing.
- `beforeToolCall`/`afterToolCall` hooks and `terminate: true` tool results are the extension points for gating/short-circuiting the loop.

## `pi-coding-agent` architecture

- **`src/core/`**: the engine — `agent-session.ts`/`agent-session-runtime.ts` (session lifecycle), `session-manager.ts` (persistence, branching), `model-registry.ts`/`model-resolver.ts`, `extensions/` (loader, runner, wrapper, `ExtensionAPI` types), `tools/` (built-in `read`/`write`/`edit`/`bash`/`grep`/`find`/`ls`), `compaction/`, `skills.ts`, `prompt-templates.ts`, `event-bus.ts`.
- **`src/modes/`**: entry points that drive `core/` — `interactive/` (the TUI, built on `pi-tui`, in `interactive-mode.ts` + `components/` + `theme/`) and `rpc/` (stdin/stdout JSONL protocol for non-Node integrations; strict `\n`-only framing, not generic `readline`).
- **Extensions** (`core/extensions/`): TypeScript modules loaded from `~/.pi/agent/extensions/`, `.pi/extensions/`, or a pi package. They get an `ExtensionAPI` to register tools, slash commands, keybindings, event handlers (`pi.on(...)`), and UI components — this is the intended way to add functionality (sub-agents, plan mode, permission gates, custom compaction, etc.) rather than extending the core. Pi deliberately excludes MCP, sub-agents, permission popups, plan mode, and built-in to-dos from core; see the "Philosophy" section of `packages/coding-agent/README.md` before proposing core additions for these.
- **Keybindings**: never hardcode key checks; add defaults to `DEFAULT_EDITOR_KEYBINDINGS`/`DEFAULT_APP_KEYBINDINGS` in `core/keybindings.ts` so they stay user-configurable.
- **Testing**: `test/suite/` uses `test/suite/harness.ts` plus the faux provider from `pi-ai` — no real provider calls, keys, or paid tokens. Issue regressions go in `test/suite/regressions/<issue-number>-<short-slug>.test.ts`.

## `pi-tui`

Standalone differential-rendering terminal UI library (components, overlays, focus, synchronized-output rendering via CSI 2026). No dependency on the agent/LLM packages — treat it as a general-purpose TUI toolkit that `pi-coding-agent`'s interactive mode consumes.

## `pi-orchestrator`

Experimental multi-session supervisor: spawns/manages `pi` in RPC mode as subprocesses (`rpc-process.ts`) and exposes them over its own IPC protocol (`src/ipc/`). API is unstable.
