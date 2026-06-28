---
name: agnes_qwenclaw
description: Help QwenClaw/QwenPaw connect to Agnes AI OpenAI-compatible text, vision, and image generation APIs, with special care for Agnes Image requests.
metadata:
  requires:
    env: [AGNES_API_KEY]
---

# Agnes for QwenClaw

Use this skill when the user wants QwenClaw or QwenPaw to configure, call, test, or debug Agnes AI models. Prioritize the image workflow, because Agnes image requests have a few non-obvious field-placement rules.

## Activation

Use this skill for:

- Agnes model provider setup in QwenClaw/QwenPaw.
- OpenAI-compatible Agnes text chat, streaming, tool calling, or image understanding.
- Text-to-image, image-to-image, image editing, image URL output, or Base64 image output.
- Debugging Agnes errors such as 401, wrong Base URL, missing image input, inaccessible image URL, timeout, or misplaced `response_format`.

Do not use this skill for generic prompt writing unless the prompt will be sent to an Agnes model.

## Defaults

- Base URL: `https://apihub.agnes-ai.com/v1`
- Chat endpoint: `POST https://apihub.agnes-ai.com/v1/chat/completions`
- Image endpoint: `POST https://apihub.agnes-ai.com/v1/images/generations`
- Chat model: `agnes-2.0-flash`
- Image model: `agnes-image-2.1-flash`
- Required secret: `AGNES_API_KEY`

Never write a real API key into repository files, screenshots, logs, or public examples. Use `YOUR_API_KEY` in examples.

## QwenClaw Setup

Prefer QwenClaw/QwenPaw's custom OpenAI-compatible model provider if Agnes is not built in.

1. Open QwenClaw/QwenPaw Console.
2. Go to Settings -> Models.
3. Add a custom OpenAI-compatible provider:
   - Provider name: `Agnes`
   - Base URL: `https://apihub.agnes-ai.com/v1`
   - API key: from `AGNES_API_KEY`
   - Default model: `agnes-2.0-flash`
4. Save and make it the default LLM if the user wants Agnes as the workspace default.
5. For image generation, use direct HTTP or an OpenAI-compatible images client against `/v1/images/generations`.

If installing this skill manually, put this file at one of these paths and enable it in the workspace:

```text
$QWENPAW_WORKING_DIR/skill_pool/agnes_qwenclaw/SKILL.md
$QWENPAW_WORKING_DIR/workspaces/{agent_id}/skills/agnes_qwenclaw/SKILL.md
```

If importing from GitHub, use the page URL that contains this `SKILL.md`.

## Request Selection

Choose the endpoint by intent:

- Generate or edit an image: use `/v1/images/generations`.
- Understand, describe, OCR, or analyze an existing image: use `/v1/chat/completions` with `image_url` content blocks.
- General agent, coding, or tool workflow: use `/v1/chat/completions`.

Do not send image-generation requests through chat completions.

## Chat And Vision

Basic chat request:

```bash
curl https://apihub.agnes-ai.com/v1/chat/completions \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "agnes-2.0-flash",
    "messages": [
      {"role": "user", "content": "Explain how QwenClaw can use Agnes as an OpenAI-compatible model."}
    ],
    "temperature": 0.3
  }'
```

Image understanding uses chat completions with a public image URL:

```json
{
  "model": "agnes-2.0-flash",
  "messages": [
    {
      "role": "user",
      "content": [
        {"type": "text", "text": "Analyze this screenshot and list UI issues."},
        {"type": "image_url", "image_url": {"url": "https://example.com/screenshot.png"}}
      ]
    }
  ]
}
```

Image URLs must be publicly reachable and should use standard formats such as JPG, PNG, or WebP. If the image is private, upload it somewhere accessible or convert it to a Data URI when the target workflow supports Data URI input.

## Image Generation

Always use:

```text
POST https://apihub.agnes-ai.com/v1/images/generations
```

Use common sizes such as `1024x768`, `1024x1024`, `768x1024`, or `4096x4096`. Prefer dimensions divisible by `16` to avoid parameter errors.

### Text To Image, URL Output

Put `response_format` inside `extra_body`, not at the top level.

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

Read the result from:

```text
data[0].url
```

### Text To Image, Base64 Output

For text-to-image Base64 output, use top-level `return_base64`.

```json
{
  "model": "agnes-image-2.1-flash",
  "prompt": "A luminous floating city above a misty canyon at sunrise, cinematic realism, wide angle",
  "size": "1024x768",
  "return_base64": true
}
```

Read the result from:

```text
data[0].b64_json
```

### Image To Image, URL Output

For image-to-image, provide the source image array in `extra_body.image`. Do not add `tags`.

```json
{
  "model": "agnes-image-2.1-flash",
  "prompt": "Transform the scene into a rain-soaked cyberpunk night with neon reflections while preserving the original composition",
  "size": "1024x768",
  "extra_body": {
    "image": ["https://example.com/input-image.png"],
    "response_format": "url"
  }
}
```

Read the result from `data[0].url`.

### Image To Image, Base64 Output

For image-to-image Base64 output, keep both `image` and `response_format` inside `extra_body`.

```json
{
  "model": "agnes-image-2.1-flash",
  "prompt": "Make the object matte black while preserving the original composition and camera angle",
  "size": "1024x768",
  "extra_body": {
    "image": ["data:image/png;base64,BASE64_HERE"],
    "response_format": "b64_json"
  }
}
```

Read the result from `data[0].b64_json`.

## Image Prompt Pattern

For text-to-image:

```text
[subject] + [scene/environment] + [style] + [lighting] + [composition] + [quality/detail requirements]
```

For image-to-image:

```text
[what to change] + [new style/scene] + [elements to add/remove] + [what must stay unchanged]
```

When editing an existing image, explicitly say what must be preserved, such as original composition, subject layout, camera angle, product shape, brand colors, or text placement.

## Troubleshooting

- `401`: check `Authorization: Bearer YOUR_API_KEY`; confirm the key is present and has no extra spaces.
- `404` or wrong endpoint: check whether the Base URL already includes `/v1`; avoid `https://apihub.agnes-ai.com/v1/v1/...`.
- `response_format` error: move `response_format` into `extra_body`.
- Image-to-image missing input: include `extra_body.image` as an array of public URLs or Data URI strings.
- Image-to-image unexpected error: remove `tags`; Agnes image-to-image does not need `tags: ["img2img"]`.
- Input image cannot be read: use a public HTTPS URL without login, cookies, or hotlink protection; otherwise use Data URI input.
- Timeout: image generation can take seconds to minutes. Use a client timeout between `60s` and `360s`.
- Skill not loaded: confirm the workspace copy exists under `workspaces/{agent_id}/skills/agnes_qwenclaw/SKILL.md` and is enabled in the QwenClaw/QwenPaw skills UI.

## Validation Checklist

Before telling the user setup is complete:

- Confirm the skill is enabled in the target workspace.
- Confirm `AGNES_API_KEY` is configured in the host environment, QwenClaw/QwenPaw model settings, or the skill config.
- Run a small chat request if the user allows a live API test.
- For image tasks, run one minimal text-to-image request with `extra_body.response_format: "url"` when live testing is allowed.
- Report the exact endpoint, model name, and response path used.
