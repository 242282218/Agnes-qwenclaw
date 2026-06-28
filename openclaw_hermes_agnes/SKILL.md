---
name: agnes_openclaw_hermes
description: Guide OpenClaw and Hermes Agent/HermesAgents users through Agnes AI OpenAI-compatible model setup, verification, and troubleshooting, with Hermes as the primary path.
metadata:
  requires:
    env: [AGNES_API_KEY]
---

# Agnes for OpenClaw and Hermes

Use this skill when the user wants to connect Agnes AI models to OpenClaw, Hermes Agent, or HermesAgents. Treat "agens" as a likely typo for "Agnes" unless the user clearly means a different provider.

This skill is intentionally narrow: configure Agnes text, tool, and vision-capable chat models through OpenAI-compatible `/v1/chat/completions`; use Agnes image generation endpoints directly when the host tool does not expose image-generation model routing.

## Defaults

- API base URL: `https://apihub.agnes-ai.com/v1`
- Chat endpoint: `https://apihub.agnes-ai.com/v1/chat/completions`
- Image endpoint: `https://apihub.agnes-ai.com/v1/images/generations`
- Primary chat/agent model: `agnes-2.0-flash`
- Image generation model: `agnes-image-2.1-flash`
- Secret env var: `AGNES_API_KEY`
- Chat API mode: OpenAI Chat Completions

Never store a real API key in a repo, screenshot, issue, shared config snippet, or public skill. Use `YOUR_API_KEY` in public examples.

## Choose The Integration Path

Prefer the simplest official path:

- Hermes Agent / HermesAgents: use `hermes model` first. If repeatable config is needed, use `~/.hermes/config.yaml`.
- OpenClaw: add an explicit `models.providers.agnes` provider entry, then set the primary model to `agnes/agnes-2.0-flash`.
- Image generation: call Agnes `/v1/images/generations` directly unless OpenClaw image-generation routing is already configured for the user's version.

Do not set the base URL to `/v1/chat/completions`. Hermes and OpenClaw append the chat-completions route themselves for chat models.

## Hermes Agent / HermesAgents

Hermes is the primary target for this skill.

### Recommended Interactive Setup

Run:

```bash
hermes model
```

Then choose:

```text
Provider: Custom endpoint / self-hosted / OpenAI-compatible
API base URL: https://apihub.agnes-ai.com/v1
API key: YOUR_API_KEY
Model name: agnes-2.0-flash
API mode, if prompted: chat_completions
Context length, if prompted: 512000
```

Do not prefix the key with `Bearer` unless Hermes explicitly asks for a full Authorization header.

### Manual HermesAgents Commands

Use these when the user is following Agnes' HermesAgents command-style docs:

```bash
hermes config set model.provider custom
hermes config set model.base_url https://apihub.agnes-ai.com/v1
hermes config set model.api_key YOUR_API_KEY
hermes config set model.default agnes-2.0-flash
```

If the user already stores secrets in environment variables, keep the key out of shell history and config where possible:

```bash
hermes config set OPENAI_API_KEY YOUR_API_KEY
```

For a custom Agnes provider, prefer `model.api_key` or a named provider with `key_env` over generic legacy OpenAI variables.

### Recommended `config.yaml`

Use this when the user wants a reusable named provider and safer secret handling.

```yaml
# ~/.hermes/config.yaml
custom_providers:
  - name: agnes
    base_url: https://apihub.agnes-ai.com/v1
    key_env: AGNES_API_KEY
    api_mode: chat_completions
    models:
      agnes-2.0-flash:
        context_length: 512000
        supports_vision: true

model:
  provider: custom:agnes
  default: agnes-2.0-flash
```

Put the secret in:

```bash
# ~/.hermes/.env
AGNES_API_KEY=YOUR_API_KEY
```

Switch during a Hermes chat session:

```text
/model custom:agnes:agnes-2.0-flash
```

Use `supports_vision: true` when the user wants Hermes to pass attached images or image URLs natively to Agnes instead of pre-processing them through a separate vision helper.

### Hermes Verification

Run:

```bash
hermes doctor
hermes chat -q "Reply with exactly: pong"
```

If the user is using bare `provider: custom` instead of the named `custom:agnes` provider, a direct one-shot is also acceptable:

```bash
hermes chat --provider custom --model agnes-2.0-flash -q "Reply with exactly: pong"
```

Success means the custom endpoint, model name, and API key are wired correctly. If `hermes doctor` reports an unknown provider for named config, check `custom_providers[].name`, `model.provider`, and YAML indentation.

## OpenClaw

Use OpenClaw's explicit custom provider config for Agnes. The provider entry must define the model; setting only `agents.defaults.model.primary` is not enough to register a custom runtime model.

### OpenClaw Config Snippet

Add this to the OpenClaw config file or the relevant per-agent `models.json`, adjusting the file location to the user's OpenClaw install.

```json5
{
  env: {
    AGNES_API_KEY: "YOUR_API_KEY",
  },
  models: {
    providers: {
      agnes: {
        baseUrl: "https://apihub.agnes-ai.com/v1",
        apiKey: "${AGNES_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "agnes-2.0-flash",
            name: "Agnes 2.0 Flash",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 512000,
            maxTokens: 65500,
          },
        ],
      },
    },
  },
  agents: {
    defaults: {
      model: {
        primary: "agnes/agnes-2.0-flash",
      },
      models: {
        "agnes/agnes-2.0-flash": {},
      },
    },
  },
}
```

If the user's OpenClaw version expects strict JSON rather than JSON5, remove comments, trailing commas, and unquoted object keys.

### OpenClaw CLI Verification

After changing config, restart or reload the OpenClaw Gateway if required, then test the transport before a full agent run:

```bash
openclaw infer model run --gateway --model agnes/agnes-2.0-flash --prompt "Reply with exactly: pong" --json
openclaw models set agnes/agnes-2.0-flash
```

If the tiny model test works but full agent turns fail, reduce only the failing surface:

```json5
{
  models: {
    providers: {
      agnes: {
        models: [
          {
            id: "agnes-2.0-flash",
            compat: {
              supportsTools: false
            }
          }
        ]
      }
    }
  }
}
```

Use `compat.supportsTools: false` only after proving small calls work and tool-heavy agent turns are the failure point. Agnes 2.0 Flash is intended for agent and tool workflows, so disabling tools is a fallback, not the default.

## Chat And Vision Request Shape

For direct smoke tests:

```bash
curl https://apihub.agnes-ai.com/v1/chat/completions \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "agnes-2.0-flash",
    "messages": [
      {"role": "user", "content": "Reply with exactly: pong"}
    ]
  }'
```

Vision/image understanding uses OpenAI-style content parts:

```json
{
  "model": "agnes-2.0-flash",
  "messages": [
    {
      "role": "user",
      "content": [
        {"type": "text", "text": "Describe the image and extract visible text."},
        {"type": "image_url", "image_url": {"url": "https://example.com/image.png"}}
      ]
    }
  ]
}
```

The image URL must be public and readable without cookies, login, or private headers.

## Image Generation

Hermes' model provider setup is for chat-completions style agent models. For Agnes image generation, call the image endpoint directly unless the host explicitly supports image-generation model routing.

Endpoint:

```text
POST https://apihub.agnes-ai.com/v1/images/generations
```

Text-to-image URL output:

```json
{
  "model": "agnes-image-2.1-flash",
  "prompt": "A clean product photo of a translucent glass cube on a white studio background, soft shadows, high detail",
  "size": "1024x768",
  "extra_body": {
    "response_format": "url"
  }
}
```

Image-to-image URL output:

```json
{
  "model": "agnes-image-2.1-flash",
  "prompt": "Transform the scene into a neon cyberpunk night while preserving the original composition and camera angle",
  "size": "1024x768",
  "extra_body": {
    "image": ["https://example.com/input.png"],
    "response_format": "url"
  }
}
```

Rules:

- Put `response_format` inside `extra_body`, not at top level.
- For image-to-image, put the source array in `extra_body.image`.
- Do not send `tags: ["img2img"]`.
- For text-to-image Base64 output, use top-level `return_base64: true`.
- For image-to-image Base64 output, use `extra_body.response_format: "b64_json"`.
- Prefer dimensions divisible by `16`, such as `1024x768`, `1024x1024`, `768x1024`, or `4096x4096`.

## Troubleshooting

- `401`: key missing, invalid, copied with spaces, or incorrectly prefixed. Use `Authorization: Bearer YOUR_API_KEY` for direct HTTP; do not type `Bearer` into Hermes/OpenClaw API-key fields unless asked.
- `404`: base URL likely includes the full route. Use only `https://apihub.agnes-ai.com/v1`.
- Hermes unknown provider: use `provider: custom` for bare config or `provider: custom:agnes` when using `custom_providers`.
- Hermes ignores env var: for custom providers, set `model.api_key` or `custom_providers[].key_env`; generic `OPENAI_BASE_URL` only applies to the OpenAI API provider path.
- Hermes image attachments are pre-processed instead of sent to Agnes: set `supports_vision: true` for the model.
- OpenClaw model not selectable: ensure `models.providers.agnes.models[]` contains `id: "agnes-2.0-flash"` and the model ref is `agnes/agnes-2.0-flash`.
- OpenClaw direct prompt works but agent run fails: test Gateway first; then try narrower tool settings or `compat.supportsTools: false` only for the failing model.
- Image generation 400: move `response_format` into `extra_body`; for image-to-image, move `image` into `extra_body.image` and remove `tags`.
- Image input cannot be read: use a public HTTPS URL or Data URI input.
- Timeout: image generation may take seconds to minutes; use `60s` to `360s` client timeouts.

## Completion Checklist

Before saying the integration is finished:

- Confirm the tool is Hermes or OpenClaw and use the matching setup path.
- Confirm the base URL is exactly `https://apihub.agnes-ai.com/v1`.
- Confirm the model is exactly `agnes-2.0-flash` for chat/agent work.
- Confirm `AGNES_API_KEY` or the configured key field exists without exposing the value.
- Run a tiny `pong` smoke test when the user allows live API calls.
- For image tasks, state whether the host is using direct Agnes image HTTP or native host image-routing support.
