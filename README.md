# Agnes Skills

针对不同 Agent 工具的 Agnes AI 接入 skills，重点覆盖 Agnes 文本模型、图像理解、文生图和图生图。

## QwenClaw / QwenPaw

在 QwenClaw/QwenPaw 控制台的工作区技能页，从 GitHub 导入：

```text
https://github.com/242282218/Agnes-qwenclaw/blob/main/agnes-qwenclaw/SKILL.md
```

也可以手动放到工作区：

```text
$QWENPAW_WORKING_DIR/workspaces/{agent_id}/skills/agnes-qwenclaw/SKILL.md
```

启用后，在环境变量或模型配置中设置 `AGNES_API_KEY`，Base URL 使用：

```text
https://apihub.agnes-ai.com/v1
```

## Hermes / HermesAgents

Hermes 专用 skill，重点覆盖 `hermes model`、`hermes config set` 和 `~/.hermes/config.yaml`。

导入地址：

```text
https://github.com/242282218/Agnes-qwenclaw/blob/main/agnes-hermes/SKILL.md
```

## OpenClaw

OpenClaw 专用 skill，重点覆盖 `models.providers` 自定义 provider 和 `agnes/agnes-2.0-flash` 模型引用。

导入地址：

```text
https://github.com/242282218/Agnes-qwenclaw/blob/main/agnes-openclaw/SKILL.md
```

## 参考资料

每个 skill 的参考来源记录在：

```text
docs/参考资料.md
```
