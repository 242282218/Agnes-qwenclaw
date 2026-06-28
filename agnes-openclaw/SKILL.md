---
name: agnes-openclaw
description: 帮助 OpenClaw 接入 Agnes AI 的 OpenAI 兼容模型，覆盖 models.providers 自定义 provider、默认模型、Gateway 验证、视觉能力和排错。
metadata:
  requires:
    env: [AGNES_API_KEY]
---

# OpenClaw Agnes 接入

当用户需要把 Agnes AI 模型接入 OpenClaw 时使用本 skill。用户写成 "agens" 时，除非明确指向其他服务商，否则按 "Agnes" 处理。

本 skill 专注 OpenClaw：通过 OpenAI-compatible `/v1/chat/completions` 接入 Agnes 文本、工具调用和视觉理解模型；图像生成直接调用 Agnes 图像端点，除非当前 OpenClaw 版本已有原生图像生成路由。

## 默认配置

- API base URL: `https://apihub.agnes-ai.com/v1`
- Chat endpoint: `https://apihub.agnes-ai.com/v1/chat/completions`
- Image endpoint: `https://apihub.agnes-ai.com/v1/images/generations`
- 主聊天/Agent 模型：`agnes-2.0-flash`
- 图像生成模型：`agnes-image-2.1-flash`
- 密钥环境变量：`AGNES_API_KEY`
- OpenClaw provider id: `agnes`
- OpenClaw model ref: `agnes/agnes-2.0-flash`

不要把真实 API Key 写入仓库、截图、issue、共享配置片段或公开 skill。公开示例统一使用 `YOUR_API_KEY`。

## 配置原则

OpenClaw 接入 Agnes 时，应使用显式 custom provider 配置。Provider 条目必须定义模型；只设置 `agents.defaults.model.primary` 不足以注册自定义运行时模型。

不要把 Base URL 写成 `/v1/chat/completions`。OpenClaw 在调用聊天模型时会自行拼接 chat-completions 路径。

## OpenClaw 配置片段

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

## CLI 验证

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

OpenClaw 的聊天 provider 配置主要面向 chat-completions 风格的 Agent 模型。Agnes 图像生成建议直接调用图像端点，除非 OpenClaw 已明确配置图像生成模型路由。

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

- `401`：key 缺失、无效、复制时带空格，或前缀写错。直接 HTTP 使用 `Authorization: Bearer YOUR_API_KEY`；OpenClaw API key 输入框里不要写 `Bearer`，除非工具明确要求。
- `404`：Base URL 可能包含了完整路由。只使用 `https://apihub.agnes-ai.com/v1`。
- OpenClaw 模型不可选：确认 `models.providers.agnes.models[]` 包含 `id: "agnes-2.0-flash"`，且模型引用为 `agnes/agnes-2.0-flash`。
- OpenClaw 直接 prompt 成功但 agent 运行失败：先测试 Gateway；再针对失败模型收窄工具设置，必要时才尝试 `compat.supportsTools: false`。
- 图像生成 400：把 `response_format` 移到 `extra_body`；图生图时把 `image` 放到 `extra_body.image`，并移除 `tags`。
- 输入图像不可读：使用公开 HTTPS URL 或 Data URI 输入。
- 超时：图像生成可能需要数秒到数分钟，客户端超时建议设为 `60s` 到 `360s`。

## 完成清单

在告诉用户接入完成前，确认：

- Base URL 精确为 `https://apihub.agnes-ai.com/v1`。
- provider id 为 `agnes`，模型引用为 `agnes/agnes-2.0-flash`。
- `models.providers.agnes.models[]` 已显式声明 `agnes-2.0-flash`。
- `AGNES_API_KEY` 或对应 key 字段存在，且没有暴露实际值。
- 用户允许真实 API 测试时，运行一个小型 `pong` 冒烟测试。
- 图像任务需要说明：当前是直接调用 Agnes 图像 HTTP，还是使用 OpenClaw 的原生图像路由。
