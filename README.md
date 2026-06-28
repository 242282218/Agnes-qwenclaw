# Agnes QwenClaw Skill

一个专门给 QwenClaw/QwenPaw 使用的 Agnes AI 接入 skill，重点覆盖 Agnes 文本模型、图像理解、文生图和图生图。

## 安装

在 QwenClaw/QwenPaw 控制台的工作区技能页，从 GitHub 导入包含 `SKILL.md` 的页面：

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
