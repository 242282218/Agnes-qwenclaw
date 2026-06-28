---
name: agnes-qwenclaw
description: 帮助 QwenClaw/QwenPaw 接入 Agnes AI 的 OpenAI 兼容文本、视觉理解与图像生成接口，尤其关注 Agnes 图像请求的参数写法。
metadata:
  requires:
    env: [AGNES_API_KEY]
---

# QwenClaw Agnes 接入

当用户需要在 QwenClaw 或 QwenPaw 中配置、调用、测试或排查 Agnes AI 模型时使用本 skill。优先关注图像工作流，因为 Agnes 图像接口有几个容易放错位置的参数。

## 适用场景

使用本 skill 处理：

- 在 QwenClaw/QwenPaw 中配置 Agnes 模型服务商。
- 调用 Agnes 的 OpenAI 兼容文本对话、流式输出、工具调用或图像理解能力。
- 文生图、图生图、图像编辑、图像 URL 输出或 Base64 图像输出。
- 排查 401、Base URL 错误、缺少输入图像、图像 URL 不可访问、超时、`response_format` 位置错误等问题。

如果只是普通提示词润色，且不会发送给 Agnes 模型，不需要使用本 skill。

## 默认配置

- Base URL: `https://apihub.agnes-ai.com/v1`
- Chat endpoint: `POST https://apihub.agnes-ai.com/v1/chat/completions`
- Image endpoint: `POST https://apihub.agnes-ai.com/v1/images/generations`
- Chat model: `agnes-2.0-flash`
- Image model: `agnes-image-2.1-flash`
- 必需密钥：`AGNES_API_KEY`

不要把真实 API Key 写进仓库、截图、日志或公开示例。公开示例统一使用 `YOUR_API_KEY`。

## QwenClaw 接入

如果 QwenClaw/QwenPaw 没有内置 Agnes 服务商，优先使用自定义 OpenAI-compatible provider。

1. 打开 QwenClaw/QwenPaw Console。
2. 进入 Settings -> Models。
3. 添加自定义 OpenAI-compatible provider：
   - Provider name: `Agnes`
   - Base URL: `https://apihub.agnes-ai.com/v1`
   - API key: 来自 `AGNES_API_KEY`
   - Default model: `agnes-2.0-flash`
4. 保存；如果用户希望 Agnes 作为当前工作区默认模型，则设为默认 LLM。
5. 图像生成走直接 HTTP 请求，或使用指向 `/v1/images/generations` 的 OpenAI-compatible images client。

手动安装本 skill 时，把本文件放到以下任一路径，并在工作区启用：

```text
$QWENPAW_WORKING_DIR/skill_pool/agnes-qwenclaw/SKILL.md
$QWENPAW_WORKING_DIR/workspaces/{agent_id}/skills/agnes-qwenclaw/SKILL.md
```

从 GitHub 导入时，使用包含此 `SKILL.md` 的页面 URL。

## 请求选择

按意图选择端点：

- 生成或编辑图像：使用 `/v1/images/generations`。
- 理解、描述、OCR 或分析已有图像：使用 `/v1/chat/completions`，并传入 `image_url` 内容块。
- 通用 Agent、编码或工具调用工作流：使用 `/v1/chat/completions`。

不要把图像生成请求发到 chat completions。

## 文本与图像理解

基础聊天请求：

```bash
curl https://apihub.agnes-ai.com/v1/chat/completions \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "agnes-2.0-flash",
    "messages": [
      {"role": "user", "content": "说明 QwenClaw 如何把 Agnes 作为 OpenAI-compatible 模型接入。"}
    ],
    "temperature": 0.3
  }'
```

图像理解使用 chat completions，并传入公开可访问的图片 URL：

```json
{
  "model": "agnes-2.0-flash",
  "messages": [
    {
      "role": "user",
      "content": [
        {"type": "text", "text": "分析这张截图，并列出 UI 问题。"},
        {"type": "image_url", "image_url": {"url": "https://example.com/screenshot.png"}}
      ]
    }
  ]
}
```

图像 URL 必须能被公网访问，建议使用 JPG、PNG 或 WebP 等标准格式。如果图片是私有资源，应先上传到可访问位置；目标工作流支持 Data URI 时，也可以转为 Data URI。

## 图像生成

始终使用：

```text
POST https://apihub.agnes-ai.com/v1/images/generations
```

常用尺寸包括 `1024x768`、`1024x1024`、`768x1024`、`4096x4096`。优先使用能被 `16` 整除的宽高，减少参数错误。

### 文生图，URL 输出

`response_format` 必须放在 `extra_body` 内，不要放在顶层。

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

结果读取路径：

```text
data[0].url
```

### 文生图，Base64 输出

文生图需要 Base64 输出时，使用顶层 `return_base64`。

```json
{
  "model": "agnes-image-2.1-flash",
  "prompt": "日出时薄雾峡谷上方的发光浮空城市，电影级写实风格，广角构图",
  "size": "1024x768",
  "return_base64": true
}
```

结果读取路径：

```text
data[0].b64_json
```

### 图生图，URL 输出

图生图时，输入图像数组放在 `extra_body.image`。不要添加 `tags`。

```json
{
  "model": "agnes-image-2.1-flash",
  "prompt": "将场景转换为雨夜赛博朋克风格，添加霓虹倒影，同时保留原始构图",
  "size": "1024x768",
  "extra_body": {
    "image": ["https://example.com/input-image.png"],
    "response_format": "url"
  }
}
```

结果读取路径：`data[0].url`。

### 图生图，Base64 输出

图生图需要 Base64 输出时，`image` 和 `response_format` 都放在 `extra_body` 内。

```json
{
  "model": "agnes-image-2.1-flash",
  "prompt": "将物体改为哑光黑色，同时保留原始构图和相机角度",
  "size": "1024x768",
  "extra_body": {
    "image": ["data:image/png;base64,BASE64_HERE"],
    "response_format": "b64_json"
  }
}
```

结果读取路径：`data[0].b64_json`。

## 图像提示词结构

文生图：

```text
[主体] + [场景/环境] + [风格] + [光照] + [构图] + [质量/细节要求]
```

图生图：

```text
[要改变什么] + [新风格/新场景] + [要添加或移除的元素] + [必须保持不变的内容]
```

编辑已有图像时，要明确写出必须保留的内容，例如原始构图、主体布局、相机角度、产品形状、品牌颜色或文字位置。

## 排错

- `401`：检查 `Authorization: Bearer YOUR_API_KEY`；确认 key 存在且没有多余空格。
- `404` 或端点错误：检查 Base URL 是否重复包含 `/v1`；避免 `https://apihub.agnes-ai.com/v1/v1/...`。
- `response_format` 报错：把 `response_format` 移到 `extra_body` 内。
- 图生图缺少输入：传入 `extra_body.image`，值为公开 URL 或 Data URI 字符串数组。
- 图生图异常：移除 `tags`；Agnes 图生图不需要 `tags: ["img2img"]`。
- 输入图像读取失败：使用无需登录、cookie 或防盗链的公开 HTTPS URL；否则改用 Data URI。
- 超时：图像生成可能需要数秒到数分钟。客户端超时建议设为 `60s` 到 `360s`。
- Skill 未加载：确认工作区副本存在于 `workspaces/{agent_id}/skills/agnes-qwenclaw/SKILL.md`，并已在 QwenClaw/QwenPaw 技能界面启用。

## 验证清单

在告诉用户接入完成前，确认：

- 目标工作区已启用该 skill。
- `AGNES_API_KEY` 已在宿主环境变量、QwenClaw/QwenPaw 模型设置或 skill config 中配置。
- 用户允许真实 API 测试时，跑一个小型 chat 请求。
- 图像任务允许真实测试时，跑一个最小文生图请求，并使用 `extra_body.response_format: "url"`。
- 向用户报告实际使用的端点、模型名和响应读取路径。
