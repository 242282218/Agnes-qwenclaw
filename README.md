# Agnes Skills

针对不同 Agent 工具的 Agnes AI 接入 skills，重点覆盖 Agnes 文本模型、图像理解、文生图和图生图。

## QwenClaw / QwenPaw

在 QwenClaw/QwenPaw 控制台的工作区技能页，从 GitHub 导入：

```text
https://github.com/242282218/Agnes-qwenclaw/blob/main/SKILL.md
```

也可以手动放到工作区：

```text
$QWENPAW_WORKING_DIR/workspaces/{agent_id}/skills/agnes_qwenclaw/SKILL.md
```

启用后，在环境变量或模型配置中设置 `AGNES_API_KEY`，Base URL 使用：

```text
https://apihub.agnes-ai.com/v1
```

## OpenClaw / Hermes

通用版 skill，重点照顾 Hermes / HermesAgents 的自定义 OpenAI-compatible provider 接入，也包含 OpenClaw 的 `models.providers` 配置模板。

导入地址：

```text
https://github.com/242282218/Agnes-qwenclaw/blob/main/openclaw_hermes_agnes/SKILL.md
```
