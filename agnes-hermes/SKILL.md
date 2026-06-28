---
name: agnes-hermes
description: 帮助 Hermes Agent/HermesAgents 接入 Agnes AI 的 OpenAI 兼容模型，覆盖 hermes model、hermes config、config.yaml、视觉能力、验证和排错。
metadata:
  requires:
    env: [AGNES_API_KEY]
---

# Hermes Agnes 接入

当用户需要把 Agnes AI 模型接入 Hermes Agent 或 HermesAgents 时使用本 skill。用户写成 "agens" 时，除非明确指向其他服务商，否则按 "Agnes" 处理。

本 skill 专注 Hermes：通过 OpenAI-compatible `/v1/chat/completions` 接入 Agnes 文本、工具调用和视觉理解模型；图像生成直接调用 Agnes 图像端点，除非 Hermes 当前版本已有原生图像生成路由。

## 默认配置

- API base URL: `https://apihub.agnes-ai.com/v1`
- Chat endpoint: `https://apihub.agnes-ai.com/v1/chat/completions`
- Image endpoint: `https://apihub.agnes-ai.com/v1/images/generations`
- 主聊天/Agent 模型：`agnes-2.0-flash`
- 图像生成模型：`agnes-image-2.1-flash`
- 密钥环境变量：`AGNES_API_KEY`
- Chat API 模式：OpenAI Chat Completions

不要把真实 API Key 写入仓库、截图、issue、共享配置片段或公开 skill。公开示例统一使用 `YOUR_API_KEY`。

## 推荐交互式配置

优先运行官方交互式配置：

```bash
hermes model
```

按提示选择或填写：

```text
Provider: Custom endpoint / self-hosted / OpenAI-compatible
API base URL: https://apihub.agnes-ai.com/v1
API key: YOUR_API_KEY
Model name: agnes-2.0-flash
API mode, if prompted: chat_completions
Context length, if prompted: 512000
```

API key 字段里不要加 `Bearer` 前缀，除非 Hermes 明确要求填写完整 Authorization header。

## 命令式配置

当用户按 Agnes 的 HermesAgents 命令式文档接入时，可使用：

```bash
hermes config set model.provider custom
hermes config set model.base_url https://apihub.agnes-ai.com/v1
hermes config set model.api_key YOUR_API_KEY
hermes config set model.default agnes-2.0-flash
```

如果用户已经用环境变量管理密钥，尽量避免把密钥写进 shell history 或配置文件：

```bash
hermes config set OPENAI_API_KEY YOUR_API_KEY
```

对于自定义 Agnes provider，优先使用 `model.api_key`，或使用带 `key_env` 的命名 provider；不要依赖泛用的旧 OpenAI 环境变量路径。

## 推荐 `config.yaml`

需要命名 provider 和更安全的密钥管理时使用：

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

把密钥放到：

```bash
# ~/.hermes/.env
AGNES_API_KEY=YOUR_API_KEY
```

在 Hermes 聊天会话中切换模型：

```text
/model custom:agnes:agnes-2.0-flash
```

如果用户希望 Hermes 将附件图像或图像 URL 原样传给 Agnes，而不是先交给其他视觉辅助流程处理，请设置 `supports_vision: true`。

## 验证

运行：

```bash
hermes doctor
hermes chat -q "Reply with exactly: pong"
```

如果用户使用的是裸 `provider: custom`，而不是命名的 `custom:agnes` provider，也可以直接单次测试：

```bash
hermes chat --provider custom --model agnes-2.0-flash -q "Reply with exactly: pong"
```

如果返回 `pong`，说明自定义端点、模型名和 API key 已接通。若 `hermes doctor` 报 unknown provider，检查 `custom_providers[].name`、`model.provider` 和 YAML 缩进。

## Chat 与视觉理解请求

直接冒烟测试：

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

视觉/图像理解使用 OpenAI 风格的 content parts：

```json
{
  "model": "agnes-2.0-flash",
  "messages": [
    {
      "role": "user",
      "content": [
        {"type": "text", "text": "描述这张图片，并提取可见文字。"},
        {"type": "image_url", "image_url": {"url": "https://example.com/image.png"}}
      ]
    }
  ]
}
```

图像 URL 必须公开可读，不能依赖 cookie、登录态或私有请求头。

## 图像生成

Hermes 的模型 provider 配置主要面向 chat-completions 风格的 Agent 模型。Agnes 图像生成建议直接调用图像端点，除非 Hermes 明确支持图像生成模型路由。

端点：

```text
POST https://apihub.agnes-ai.com/v1/images/generations
```

文生图 URL 输出：

```json
{
  "model": "agnes-image-2.1-flash",
  "prompt": "白色摄影棚中的半透明玻璃立方体产品图，柔和阴影，高细节，干净商业摄影风格",
  "size": "1024x768",
  "extra_body": {
    "response_format": "url"
  }
}
```

图生图 URL 输出：

```json
{
  "model": "agnes-image-2.1-flash",
  "prompt": "将场景转换为霓虹赛博朋克夜景，同时保留原始构图和相机角度",
  "size": "1024x768",
  "extra_body": {
    "image": ["https://example.com/input.png"],
    "response_format": "url"
  }
}
```

规则：

- `response_format` 放在 `extra_body` 内，不要放在顶层。
- 图生图时，输入图像数组放在 `extra_body.image`。
- 不要发送 `tags: ["img2img"]`。
- 文生图 Base64 输出使用顶层 `return_base64: true`。
- 图生图 Base64 输出使用 `extra_body.response_format: "b64_json"`。
- 优先使用能被 `16` 整除的尺寸，例如 `1024x768`、`1024x1024`、`768x1024`、`4096x4096`。

## 排错

- `401`：key 缺失、无效、复制时带空格，或前缀写错。直接 HTTP 使用 `Authorization: Bearer YOUR_API_KEY`；Hermes API key 输入框里不要写 `Bearer`，除非工具明确要求。
- `404`：Base URL 可能包含了完整路由。只使用 `https://apihub.agnes-ai.com/v1`。
- Hermes unknown provider：裸配置用 `provider: custom`；命名 provider 用 `provider: custom:agnes`。
- Hermes 不读取环境变量：custom provider 使用 `model.api_key` 或 `custom_providers[].key_env`；泛用 `OPENAI_BASE_URL` 只适用于 OpenAI API provider 路径。
- Hermes 图像附件被预处理而不是发给 Agnes：给模型设置 `supports_vision: true`。
- 图像生成 400：把 `response_format` 移到 `extra_body`；图生图时把 `image` 放到 `extra_body.image`，并移除 `tags`。
- 输入图像不可读：使用公开 HTTPS URL 或 Data URI 输入。
- 超时：图像生成可能需要数秒到数分钟，客户端超时建议设为 `60s` 到 `360s`。

## 完成清单

在告诉用户接入完成前，确认：

- Base URL 精确为 `https://apihub.agnes-ai.com/v1`。
- 聊天/Agent 模型精确为 `agnes-2.0-flash`。
- `AGNES_API_KEY` 或对应 key 字段存在，且没有暴露实际值。
- 用户允许真实 API 测试时，运行一个小型 `pong` 冒烟测试。
- 图像任务需要说明：当前是直接调用 Agnes 图像 HTTP，还是使用 Hermes 的原生图像路由。
