# How pi assembles the system prompt

This note traces exactly where the text that ends up as `agent.state.systemPrompt` comes
from, for the `pi` CLI (`packages/coding-agent`). There is a second, much simpler
system-prompt mechanism in `packages/agent`'s generic `AgentHarness` used by SDK embedders
that don't use `pi-coding-agent` at all — see [The other, simpler path](#the-other-simpler-path-agentharness)
at the end.

## The short version

`AgentSession` (`packages/coding-agent/src/core/agent-session.ts`) owns a `_baseSystemPrompt`
string. It is (re)computed by `_rebuildSystemPrompt()` (line ~983), which:

1. Collects prompt metadata (`toolSnippets`, `promptGuidelines`) from the currently active tools.
2. Pulls loaded resources — custom system prompt text, append text, skills, `AGENTS.md`/`CLAUDE.md`
   context files — out of a `ResourceLoader`.
3. Passes all of that as a `BuildSystemPromptOptions` object into the pure function
   `buildSystemPrompt()` (`packages/coding-agent/src/core/system-prompt.ts`), which does the actual
   string concatenation.

The result is stored as `this._baseSystemPrompt` and copied onto `this.agent.state.systemPrompt`
(unless a per-turn extension override is active — see [Per-turn overrides](#per-turn-overrides)).

## 1. Resource discovery (`ResourceLoader`)

`DefaultResourceLoader.reload()` (`core/resource-loader.ts:338`) runs once at session startup
(from `createAgentSessionServices`, `core/agent-session-services.ts:151`) and again on `/reload`
or extension-reload. It discovers, in this order:

### Custom system prompt (replaces the whole default prompt)

`discoverSystemPromptFile()` (`resource-loader.ts:965`):

- `--system-prompt <text>` (CLI flag, `cli/args.ts:93`) wins if given.
- Otherwise `.pi/SYSTEM.md` in the project **if the project is trusted** (`settingsManager.isProjectTrusted()`).
- Otherwise `~/.pi/agent/SYSTEM.md` (global, `getAgentDir()`).
- Otherwise: no custom prompt (default prompt is built instead).

`resolvePromptInput()` (`resource-loader.ts:50`) treats the resolved value as a **file path if
it exists on disk**, reading its contents; otherwise the string itself is used literally. This is
why `--system-prompt` accepts either inline text or a path.

### Append text (added to whichever prompt — default or custom — is used)

`discoverAppendSystemPromptFile()` (`resource-loader.ts:979`) follows the same precedence:
`--append-system-prompt <text>` (repeatable CLI flag) > `.pi/APPEND_SYSTEM.md` (trusted project)
> `~/.pi/agent/APPEND_SYSTEM.md` (global). **CLI flags fully replace file discovery** — they are
not merged with the `APPEND_SYSTEM.md` files. Multiple `--append-system-prompt` values (or a
single discovered file) are joined with `\n\n` (`agent-session.ts:1001`).

### Project context files (`AGENTS.md` / `CLAUDE.md`)

`loadProjectContextFiles()` (`resource-loader.ts:85`), disabled by `--no-context-files`/`-nc`:

- Per directory, the first match of `AGENTS.md`, `AGENTS.MD`, `CLAUDE.md`, `CLAUDE.MD` (in that
  order) is used — only one file per directory (`loadContextFileFromDir`, `resource-loader.ts:67`).
- `~/.pi/agent/` (the global agent dir) is checked first and always included if present.
- Then pi walks from `cwd` up to the filesystem root, collecting one file per ancestor directory.
- Final order: **global file first, then ancestors from the root down to `cwd`** (root-most
  first, `cwd`'s own file last — `resource-loader.ts:101-119`, built with `unshift` while walking
  upward). This means the most specific (closest-to-cwd) instructions appear last/most prominent
  in the assembled prompt.

### Skills

`loadSkills()` (`core/skills.ts:387`), disabled by `--no-skills`/`-ns`: scans skill directories
(project `.pi/skills/`, global `~/.pi/agent/skills/`, pi packages, plus any extension- or
CLI-contributed paths) for `SKILL.md` files, parsing YAML frontmatter (`name`, `description`,
`disable-model-invocation`). Respects `.gitignore`/`.ignore`/`.fdignore`.

### Extensions, prompts, themes

Also (re)loaded here but not part of the system prompt text itself — extensions can, however,
register tools (which contribute prompt snippets, see below) and hook `before_agent_start`
(see [Per-turn overrides](#per-turn-overrides)).

All of the above are exposed via `ResourceLoader.getSystemPrompt()`, `.getAppendSystemPrompt()`,
`.getSkills()`, `.getAgentsFiles()`.

## 2. Tool metadata

`AgentSession._refreshToolRegistry()` (`agent-session.ts:2397`) builds, for every currently
active tool, two maps from each tool's `ToolDefinition`:

- `promptSnippet?: string` — a one-line description (`extensions/types.ts:445`). A tool only
  appears in the "Available tools" list if it has one. Built-in examples
  (`core/tools/read.ts:213`, `core/tools/bash.ts:302`):
  - `read`: `"Read file contents"`
  - `bash`: `"Execute bash commands (ls, grep, find, etc.)"`
- `promptGuidelines?: string[]` — extra bullet points appended to the "Guidelines" section
  (e.g. `read.ts:214`: `"Use read to examine files instead of cat or sed."`).

Both are normalized (whitespace-collapsed, deduped) and passed into `buildSystemPrompt()` as
`toolSnippets` / `promptGuidelines`. Extension-registered custom tools participate the same way —
this is the intended mechanism for an extension to add its own line to "Available tools" or its
own guideline bullet, rather than hand-editing the prompt.

`_refreshToolRegistry()` ends by calling `setActiveToolsByName()`, which always calls
`_rebuildSystemPrompt()` — so the prompt is recomputed any time the active tool set changes
(startup, `/tools` changes, extension enabling a tool, `reload()`).

## 3. `buildSystemPrompt()` — the actual string

`packages/coding-agent/src/core/system-prompt.ts:28`. Two branches:

### Custom prompt branch (`customPrompt` set)

```
<customPrompt text>
<appendSystemPrompt, if any>

<project_context> ... </project_context>      (if context files were loaded)
<available_skills> ... </available_skills>    (only if "read" tool is active and skills exist)
Current date: YYYY-MM-DD
Current working directory: <cwd>
```

### Default prompt branch (no custom prompt)

```
You are an expert coding assistant operating inside pi, a coding agent harness. ...

Available tools:
- read: Read file contents
- bash: Execute bash commands (ls, grep, find, etc.)
... (only tools with a promptSnippet are listed; "(none)" if none)

In addition to the tools above, you may have access to other custom tools depending on the project.

Guidelines:
- Use bash for file operations like ls, rg, find      (only if bash is active but grep/find/ls are not)
- <any promptGuidelines contributed by active tools>
- <any extra promptGuidelines passed in>
- Be concise in your responses                        (always)
- Show file paths clearly when working with files      (always)

Pi documentation (read only when the user asks about pi itself, ...):
- Main documentation: <readmePath>
- Additional docs: <docsPath>
- Examples: <examplesPath>
- ... (fixed block of pi-self-documentation pointers, from config.ts's getReadmePath/getDocsPath/getExamplesPath)

<appendSystemPrompt, if any>

<project_context> ... </project_context>      (if context files were loaded)
<available_skills> ... </available_skills>    (only if "read" tool is active and skills exist)
Current date: YYYY-MM-DD
Current working directory: <cwd>
```

Notes:

- The "Pi documentation" block only appears in the default prompt — a custom `--system-prompt`
  skips it entirely (you lose pi's self-help pointers if you fully replace the prompt; use
  append instead if you just want to add to it).
- `<project_context>` wraps each loaded file as
  `<project_instructions path="...">...</project_instructions>`, in the discovery order described
  above (global, then root-to-cwd).
- `<available_skills>` is XML per the [Agent Skills spec](https://agentskills.io/integrate-skills),
  built by `formatSkillsForPrompt()` (`core/skills.ts:335`) — skills with
  `disable-model-invocation: true` in frontmatter are excluded (they're only reachable via
  `/skill:name`). Note `packages/agent/src/harness/system-prompt.ts` has a near-identical
  `formatSkillsForSystemPrompt()` for the generic harness — the two are independent copies with
  slightly different wording.
- Date and cwd are always the last two lines, in both branches.

## Per-turn overrides

Two mechanisms make the system prompt dynamic across a session rather than fixed at startup:

1. **`before_agent_start` extension hook.** Each time the user submits a prompt,
   `AgentSession` calls `_extensionRunner.emitBeforeAgentStart(text, images, this._baseSystemPrompt,
   this._baseSystemPromptOptions)` (`agent-session.ts:1184`). Extensions receive the fully
   assembled base prompt plus the raw `BuildSystemPromptOptions` (so they can inspect what was
   loaded without re-discovering resources) and may return a replacement `systemPrompt` string.
   If they do, it's stored in `_systemPromptOverride` and used for that turn; if they don't,
   `_systemPromptOverride` is cleared and the base prompt is used
   (`agent-session.ts:1204-1212`). The override does **not** persist across user turns — it's
   recomputed (or cleared) on every `before_agent_start`.
2. **`prepareNextTurnWithContext`.** Installed once in the constructor
   (`agent-session.ts:473-494`), this hook re-injects `this._systemPromptOverride ??
   this._baseSystemPrompt` into the LLM call context immediately before *every* turn — including
   follow-up turns after a tool call, not just the first turn of a user prompt. This guarantees
   that a live-edited `AGENTS.md`/skill/tool set change (picked up by a subsequent `reload()`) or
   an extension's per-turn override is always what's actually sent, even mid multi-turn tool loop.

Extensions/commands can also read the *current* effective prompt via `ExtensionContext.getSystemPrompt()`
and the structured build options via `ExtensionCommandContext.getSystemPromptOptions()`
(`extensions/types.ts:334,343`) — used for introspection/debugging rather than construction.

## When the prompt gets rebuilt

`_rebuildSystemPrompt()` runs whenever:

- The session is first constructed (`_buildRuntime()` → `_refreshToolRegistry()` →
  `setActiveToolsByName()`).
- The active tool set changes (`setActiveToolsByName()` called directly, e.g. by `/tools` or an
  extension enabling/disabling a tool).
- `AgentSession.reload()` runs (`/reload`, or extension hot-reload) — this first calls
  `resourceLoader.reload()`, which re-scans `AGENTS.md`/`CLAUDE.md`, skills, and
  `SYSTEM.md`/`APPEND_SYSTEM.md` from disk, then rebuilds tools and the prompt.

## Compaction uses a *different*, hardcoded prompt

Context compaction/summarization (`core/compaction/compaction.ts`,
`packages/agent/src/harness/compaction/*.ts`) does **not** reuse `_baseSystemPrompt`. The
summarization sub-call is made with a separate constant, `SUMMARIZATION_SYSTEM_PROMPT`, unrelated
to anything above.

## CLI flags reference

| Flag | Effect |
|---|---|
| `--system-prompt <text>` | Replace the default prompt (`customPrompt`); `<text>` may be a file path or literal text. |
| `--append-system-prompt <text>` | Append text (repeatable); replaces `APPEND_SYSTEM.md` discovery rather than merging with it. |
| `--no-context-files`, `-nc` | Skip `AGENTS.md`/`CLAUDE.md` discovery entirely. |
| `--no-skills`, `-ns` | Skip skill discovery/loading (so no `<available_skills>` block). |

Project-local `.pi/SYSTEM.md` and `.pi/APPEND_SYSTEM.md` only take effect when the project is
trusted (see project-trust gating in `core/trust-manager.ts` / `core/project-trust.ts`); an
untrusted project falls back to the global `~/.pi/agent/` files.

## The other, simpler path: `AgentHarness`

`packages/agent/src/harness/agent-harness.ts` is a generic, provider-agnostic harness available
to SDK consumers who embed `pi-agent-core` directly without `pi-coding-agent`. Its
`systemPrompt` option (`AgentHarnessOptions.systemPrompt`) is either:

- a plain string, used verbatim, or
- an async function `(ctx) => string`, called fresh each turn (`agent-harness.ts:322-340`),

defaulting to `"You are a helpful assistant."` if omitted entirely. It has no concept of
`AGENTS.md`, skills-as-XML, tool snippets, or pi's own documentation block — all of that richer
assembly is specific to `pi-coding-agent`'s `buildSystemPrompt()`/`ResourceLoader` described above.
The harness does export a standalone `formatSkillsForSystemPrompt()` helper
(`harness/system-prompt.ts`) for consumers who want the same Agent-Skills XML formatting without
adopting the rest of pi-coding-agent's machinery.
