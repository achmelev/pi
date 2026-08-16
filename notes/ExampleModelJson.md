# Converting a Continue `config.yml` model to pi's `models.json`

Source: a Continue-style `config.yml` defining the custom model
`LLMaaS Qwen3.6-35B-A3B FULL-THINK` (an OpenAI-compatible LiteLLM proxy in front of
`Qwen/Qwen3.6-35B-A3B-FP8`, with Qwen3's `chat_template_kwargs.enable_thinking` toggle).

## Field mapping

| Continue (`config.yml`) | pi (`models.json`) | Notes |
|---|---|---|
| `provider: openai` + `<<: *defaultApi` | `"api": "openai-completions"` | LiteLLM's OpenAI-compatible endpoint. |
| `apiBase` (env-templated) | `"baseUrl"` | **Must be a literal string.** Unlike `apiKey`/`headers`, `baseUrl` is not passed through pi's `$VAR`/`!cmd` resolver (`resolve-config-value.ts` is only invoked for `apiKey` and `headers` in `model-registry.ts`). Hardcode the actual URL, or use an extension's `registerProvider()` at runtime if it truly needs to be dynamic. |
| `apiKey` (env-templated) | `"apiKey": "$YOUR_ENV_VAR"` | This one *does* support `$VAR` interpolation — just needs to be a real env var name exported in your shell. Rename to whatever you actually export. |
| `requestOptions.headers` | `"headers"` | Direct copy. |
| `requestOptions.verifySsl: true` | *(nothing)* | No SSL-verification toggle exists in `models.json`; `true` is Node's default anyway, so this is a no-op to drop. |
| `extraBodyProperties.chat_template_kwargs.enable_thinking` | `"compat": { "thinkingFormat": "qwen-chat-template" }` | pi has a **built-in** mode for exactly this Qwen3 hybrid-thinking pattern (`packages/ai/src/api/openai-completions.ts:615`). It auto-sends `chat_template_kwargs: { enable_thinking, preserve_thinking: true }`, driven by pi's own `/thinking` level rather than a hardcoded value — see caveat below. |
| `contextLength` | `"contextWindow"` | |
| `defaultCompletionOptions.maxTokens` | `"maxTokens"` | |
| `defaultCompletionOptions.temperature` / `topP` | — | **No config-file equivalent**, but not for the reason I first said. See correction below. |

## Converted `~/.pi/agent/models.json`

```json
{
  "providers": {
    "llmaas": {
      "baseUrl": "https://<your-litellm-host>/v1",
      "apiKey": "$LLMAAS_API_KEY",
      "api": "openai-completions",
      "headers": {
        "User-Agent": "continue-dev",
        "x-litellm-customer-id": "Muster GmbH",
        "x-litellm-tags": "continue,ide,dev"
      },
      "models": [
        {
          "id": "Qwen/Qwen3.6-35B-A3B-FP8",
          "name": "LLMaaS Qwen3.6-35B-A3B FULL-THINK",
          "reasoning": true,
          "contextWindow": 262144,
          "maxTokens": 8192,
          "compat": {
            "thinkingFormat": "qwen-chat-template"
          }
        }
      ]
    }
  }
}
```

## Behavioral difference to be aware of

The original config hardcodes `enable_thinking: true` unconditionally (matching the
"FULL-THINK" name). `qwen-chat-template` instead ties `enable_thinking` to whether pi's
`/thinking` level is off or not — so a user running `/thinking off` would disable it for this
model. If you want it **always** on regardless of pi's thinking toggle (a truer port of
"FULL-THINK"), use the lower-level `chat-template` format with a static value instead:

```json
"compat": {
  "thinkingFormat": "chat-template",
  "chatTemplateKwargs": { "enable_thinking": true, "preserve_thinking": true }
}
```

## Correction: `temperature` is actually supported at the pi-ai layer

I originally wrote that pi "doesn't send `temperature`/`topP`". That's wrong for `temperature`.
`packages/ai/src/types.ts:110` defines `temperature?: number` on `StreamOptions`, and
`packages/ai/src/api/openai-completions.ts:578-579` forwards it into the request params whenever
it's set:

```typescript
if (options?.temperature !== undefined) {
  params.temperature = options.temperature;
}
```

So the *library* (`pi-ai`) fully supports per-request `temperature` — it's a real, working
request parameter, not a stub. `topP`/`top_p`, by contrast, doesn't exist anywhere in `pi-ai` —
that one genuinely has no equivalent at any layer.

The reason `temperature` still can't be set for a custom model via `models.json` is one level up:
`models.json`'s `ModelDefinitionSchema` has no `temperature` field, and `pi-coding-agent`'s
`AgentSession` never populates `options.temperature` when it calls into `pi-ai` (confirmed: no
references to `temperature` anywhere under `packages/coding-agent/src`). There's no CLI flag or
`settings.json` key for it either. So the capability exists in the underlying library, but the
`pi` CLI's own config surface doesn't expose a knob for it — meaning **if you want a fixed
default temperature for a custom model in `pi`, you'd have to add it yourself**, e.g. via an
extension hooking `before_provider_request` (see [`notes/Debug.md`](./Debug.md)) to inject
`temperature` into the outgoing payload, or by using `pi-ai` directly as an SDK where you control
`StreamOptions` yourself.

## Not portable via `models.json`

- The org-identity headers are carried over as plain headers (included above) — nothing special
  needed there.
- The `bteaminit` **rule** (auto-writing `AGENTS.md` from a template on new projects) has no
  per-model equivalent — `models.json` only configures providers/models. Porting that behavior
  would mean writing a pi skill or prompt template, not a model config entry.
