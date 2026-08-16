# Logging / debugging what pi sends and receives from the LLM

A few layers, from built-in to fully custom.

## 1. Built-in: `/debug` command (or Shift+Ctrl+D)

Writes `~/.pi/agent/pi-debug.log` containing the current rendered TUI output plus every
`AgentMessage` in the session as JSONL (`packages/coding-agent/src/modes/interactive/interactive-mode.ts:5829`).
This is pi's own message representation (user/assistant/tool messages), not the literal
wire-level JSON sent to the provider — but it's the fastest way to see what was exchanged.

## 2. Session files (always on, no flag needed)

Every turn is persisted to the session JSONL file automatically, so the full conversation
history (including tool calls/results) is inspectable after the fact without turning anything on.

## 3. Wire-level request/response logging — via an extension

For the actual HTTP payload sent to the provider, pi ships a ready-made example extension at
`packages/coding-agent/examples/extensions/provider-payload.ts`:

```typescript
import { appendFileSync } from "node:fs";
import { join } from "node:path";
import { CONFIG_DIR_NAME, type ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  pi.on("before_provider_request", (event, ctx) => {
    const logFile = join(ctx.cwd, CONFIG_DIR_NAME, "provider-payload.log");
    appendFileSync(logFile, `${JSON.stringify(event.payload, null, 2)}\n\n`, "utf8");
    // Optional: return a modified payload to change what's actually sent.
  });

  pi.on("after_provider_response", (event, ctx) => {
    const logFile = join(ctx.cwd, CONFIG_DIR_NAME, "provider-payload.log");
    appendFileSync(logFile, `[${event.status}] ${JSON.stringify(event.headers)}\n\n`, "utf8");
  });
}
```

Drop it into `.pi/extensions/` (project) or `~/.pi/agent/extensions/` (global) and it logs the
exact outgoing JSON body plus the response status/headers to `.pi/provider-payload.log`. Note
`after_provider_response` only exposes status + headers, not the response body — the response
content itself is what shows up via the assistant message stream (and is already persisted to
the session file).

There's also `before_provider_headers` if you just want to inspect/inject request headers.

## 4. If embedding via the SDK directly (not the `pi` CLI)

`pi-ai`'s `stream`/`complete`/`streamSimple`/`completeSimple` all accept an `onPayload` callback
for the same purpose — see `packages/ai/README.md`'s "Debugging Provider Payloads" section.

```typescript
const response = await models.complete(model, context, {
  onPayload: (payload) => {
    console.log('Provider payload:', JSON.stringify(payload, null, 2));
  }
});
```
