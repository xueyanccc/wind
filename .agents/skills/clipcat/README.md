# Clipcat skill

此目录提供仓库级 Clipcat skill，供 Codex 及支持 Agent Skills 的工具读取。

- 上游仓库：https://github.com/Clipcat-ai/clipcat-skill
- 来源文件：`skills/clipcat/SKILL.md`（原样复制）
- 来源提交：`7888594dfb5fea29e40a50077d51a7dade97f66f`
- 中文指南：https://github.com/Clipcat-ai/clipcat-skill/blob/7888594dfb5fea29e40a50077d51a7dade97f66f/README_ZH.md

## 使用准备

在仓库目录中启动新的 Codex 会话，使其发现 `.agents/skills/clipcat/SKILL.md`。

按照上游安装说明安装 Clipcat CLI：

```bash
curl -fsSL https://clipcat.ai/cli | bash
clipcat --version
clipcat -h
```

在 https://clipcat.ai/workspace?modal=settings&tab=apikeys 获取 API Key，并按上游说明在本机配置。不要将密钥提交到仓库或发送到聊天中。

然后可以让 agent 使用 Clipcat skill 研究 TikTok 商品、分析视频或生成带货视频。实际 API 调用需要有效密钥，部分操作会消耗积分；生成和发布流程须遵循 SKILL.md 中的确认步骤。

本仓库仅包含技能及说明，不包含 CLI 二进制或 API Key。
