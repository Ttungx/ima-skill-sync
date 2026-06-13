# ima-skill-sync

自动同步腾讯 ima 官方 [ima-skill](https://ima.qq.com/agent-interface)

将此仓库添加到 [cc-switch]([farion1231/cc-switch: A cross-platform desktop All-in-One assistant for Claude Code, Codex, OpenCode, OpenClaw, Gemini CLI & Hermes Agent. Only official website: ccswitch.io](https://github.com/farion1231/cc-switch)) 的技能仓库配置中，即可自动获取最新版本的 `ima-skill`，无需手动下载。

## 工作原理

- GitHub Actions 每 2 天自动执行一次同步工作流（也支持 `workflow_dispatch` 手动触发）
- 下载官方 zip → 校验 SHA256 → 如有变更则更新仓库并发布 GitHub Release
- 仓库中的 `ima-skill/` 目录即 opencode 实际加载的技能载荷
