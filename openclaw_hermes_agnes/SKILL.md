---
name: agnes_openclaw_hermes
description: 帮助 OpenClaw 与 Hermes Agent/HermesAgents 用户接入 Agnes AI 的 OpenAI 兼容模型，覆盖配置、验证和排错，重点照顾 Hermes。
metadata:
  requires:
    env: [AGNES_API_KEY]
---

# OpenClaw 与 Hermes 的 Agnes 接入

当用户需要把 Agnes AI 模型接入 OpenClaw、Hermes Agent 或 HermesAgents 时使用本 skill。用户写成 "agens" 时，除非明确指向其他服务商，否则按 "Agnes" 处理。

本 skill 保持窄范围：通过 OpenAI-compatible `/v1/chat/completions` 配置 Agnes 文本、工具调用和视觉理解模型；当宿主工具没有原生图像生成路由时，直接调用 Agnes 图像生成端点。

## 默认配置

- API base URL: `https://apihub.agnes-ai.com/v1`
- Chat endpoint: `https://apihub.agnes-ai.com/v1/chat/completions`
- Image endpoint: `https://apihub.agnes-ai.com/v1/images/generations`
- 主聊天/Agent 模型：`agnes-2.0-flash`
- 图像生成模型：`agnes-image-2.1-flash`
- 密钥环境变量：`AGNES_API_KEY`
- Chat API 模式：OpenAI Chat Completions

不要把真实 API Key 写入仓库、截图、issue、共享配置片段或公开 skill。公开示例统一使用 `YOUR_API_KEY`。

## 选择接入路径

优先使用官方推荐的最简单路径：

- Hermes Agent / HermesAgents：优先使用 `hermes model` 交互式配置；需要可复用配置时，再写 `~/.hermes/config.yaml`。
- OpenClaw：显式添加 `models.providers.agnes` provider，然后把主模型设为 `agnes/agnes-2.0-flash`。
- 图像生成：除非当前 OpenClaw 版本已经配置好图像生成路由，否则直接调用 Agnes `/v1/images/generations`。

不要把 Base URL 写成 `/v1/chat/completions`。Hermes 和 OpenClaw 在调用聊天模型时会自行拼接 chat-completions 路径。

## Hermes Agent / HermesAgents

Hermes 是本 skill 的优先目标。

### 推荐交互式配置

运行：

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

除非 Hermes 明确要求填写完整 Authorization header，否则 API key 字段里不要加 `Bearer` 前缀。

### HermesAgents 命令式配置

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

### 推荐 `config.yaml`

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

### Hermes 验证

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

## OpenClaw

OpenClaw 接入 Agnes 时，应使用显式 custom provider 配置。Provider 条目必须定义模型；只设置 `agents.defaults.model.primary` 不足以注册自定义运行时模型。

### OpenClaw 配置片段

把下面配置加入 OpenClaw 配置文件或对应 agent 的 `models.json`；具体文件位置按用户的 OpenClaw 安装方式调整。

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

如果用户的 OpenClaw 版本要求严格 JSON，而不是 JSON5，需要移除注释、尾逗号和未加引号的对象键。

### OpenClaw CLI 验证

修改配置后，如有需要先重启或重新加载 OpenClaw Gateway，再在完整 Agent 运行前测试模型传输：

```bash
openclaw infer model run --gateway --model agnes/agnes-2.0-flash --prompt "Reply with exactly: pong" --json
openclaw models set agnes/agnes-2.0-flash
```

如果小型模型测试成功，但完整 agent 回合失败，只缩小失败面：

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

只有在小请求已通过、工具密集型 agent 回合仍失败时，才使用 `compat.supportsTools: false`。Agnes 2.0 Flash 本身面向 Agent 和工具工作流，禁用工具只是兜底方案，不是默认建议。

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

Hermes 的模型 provider 配置主要面向 chat-completions 风格的 Agent 模型。Agnes 图像生成建议直接调用图像端点，除非宿主工具明确支持图像生成模型路由。

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

- `401`：key 缺失、无效、复制时带空格，或前缀写错。直接 HTTP 使用 `Authorization: Bearer YOUR_API_KEY`；Hermes/OpenClaw 的 API key 输入框里不要写 `Bearer`，除非工具明确要求。
- `404`：Base URL 可能包含了完整路由。只使用 `https://apihub.agnes-ai.com/v1`。
- Hermes unknown provider：裸配置用 `provider: custom`；命名 provider 用 `provider: custom:agnes`。
- Hermes 不读取环境变量：custom provider 使用 `model.api_key` 或 `custom_providers[].key_env`；泛用 `OPENAI_BASE_URL` 只适用于 OpenAI API provider 路径。
- Hermes 图像附件被预处理而不是发给 Agnes：给模型设置 `supports_vision: true`。
- OpenClaw 模型不可选：确认 `models.providers.agnes.models[]` 包含 `id: "agnes-2.0-flash"`，且模型引用为 `agnes/agnes-2.0-flash`。
- OpenClaw 直接 prompt 成功但 agent 运行失败：先测试 Gateway；再针对失败模型收窄工具设置，必要时才尝试 `compat.supportsTools: false`。
- 图像生成 400：把 `response_format` 移到 `extra_body`；图生图时把 `image` 放到 `extra_body.image`，并移除 `tags`。
- 输入图像不可读：使用公开 HTTPS URL 或 Data URI 输入。
- 超时：图像生成可能需要数秒到数分钟，客户端超时建议设为 `60s` 到 `360s`。

## 完成清单

在告诉用户接入完成前，确认：

- 明确目标工具是 Hermes 还是 OpenClaw，并使用对应接入路径。
- Base URL 精确为 `https://apihub.agnes-ai.com/v1`。
- 聊天/Agent 模型精确为 `agnes-2.0-flash`。
- `AGNES_API_KEY` 或对应 key 字段存在，且没有暴露实际值。
- 用户允许真实 API 测试时，运行一个小型 `pong` 冒烟测试。
- 图像任务需要说明：当前是直接调用 Agnes 图像 HTTP，还是使用宿主工具的原生图像路由。
