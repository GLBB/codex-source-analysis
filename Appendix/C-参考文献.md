# 附录 C — 参考文献与引用来源

> 本附录汇总《OpenAI Codex 源码深度研究》引用的全部外部资料。一手资料置顶，第三方分析按主题分组。完整原始链接表见 `全网调研-社区认知地图.md`。

## C.1 一手资料（OpenAI 官方）

- **代码仓库** — https://github.com/openai/codex
- **CLI 官方文档站** — https://developers.openai.com/codex/cli
  - Features — https://developers.openai.com/codex/cli/features
  - Windows — https://developers.openai.com/codex/windows
  - MCP — https://developers.openai.com/codex/mcp
  - App-Server — https://developers.openai.com/codex/app-server
- **OpenAI 主站发布**
  - Introducing Codex — https://openai.com/index/introducing-codex/
  - Unrolling the Codex Agent Loop — https://openai.com/index/unrolling-the-codex-agent-loop/
  - Unlocking the Codex Harness — https://openai.com/index/unlocking-the-codex-harness/
- **官方帮助中心** — https://help.openai.com/en/articles/11369540-codex-in-chatgpt
- **仓内文档** — `docs/getting-started.md`、`docs/install.md`、`docs/authentication.md`、`docs/config.md`、`docs/sandbox.md`、`docs/skills.md`、`docs/slash_commands.md`、`docs/execpolicy.md` 等 16 篇

## C.2 第三方深度分析（英文）

- **Simon Willison**
  - First-look — https://simonwillison.net/2025/Apr/16/openai-codex/
  - How I think about Codex — https://simonwillison.net/2026/Feb/22/how-i-think-about-codex/
- **Pragmatic Engineer — How Codex is built** — https://newsletter.pragmaticengineer.com/p/how-codex-is-built
- **Latent Space — GPT-5 Codex max & training agents** — https://www.latent.space/p/gpt5-codex-max-training-agents-with
- **Hacker News 主线**
  - https://news.ycombinator.com/item?id=43708025
  - https://news.ycombinator.com/item?id=46738288
  - https://news.ycombinator.com/item?id=46737630
  - https://news.ycombinator.com/item?id=44150093
  - https://news.ycombinator.com/item?id=44833858

## C.3 第三方深度分析（中文）

- **InfoQ**
  - Codex CLI 架构分析 — https://www.infoq.cn/article/I4fzvM0XQoWQQYOD6LYT
  - Codex 训练与产品化 — https://www.infoq.cn/article/Ac7pCglOgaK4tEoWg5b7
- **掘金 / CSDN / 博客园 / 少数派 / 知乎**
  - 掘金 Codex 深读 — https://juejin.cn/post/7613658235174387727
  - 博客园 Codex 工具链笔记 — https://www.cnblogs.com/sddai/p/18830867
  - CSDN Codex 入门 — https://blog.csdn.net/qq_31095905/article/details/147887930
  - 少数派 Codex 工作流 — https://sspai.com/post/105621
  - 知乎 Codex 长文 — https://zhuanlan.zhihu.com/p/2038317397019505265
- **火山引擎开发者社区** — https://developer.volcengine.com/articles/7606557839506538506
- **iThome（繁体）** — https://www.ithome.com.tw/news/169341

## C.4 社区生态

- **awesome-codex-cli** — https://github.com/RoggeOhta/awesome-codex-cli
- **核心 Issue 与 PR 讨论**（截至 2026-05）
  - https://github.com/openai/codex/issues/5
  - https://github.com/openai/codex/issues/8925
  - https://github.com/openai/codex/issues/10090
  - https://github.com/openai/codex/issues/13279
  - https://github.com/openai/codex/issues/14913
  - https://github.com/openai/codex/issues/16808
  - https://github.com/openai/codex/issues/17179
  - https://github.com/openai/codex/issues/18829
  - https://github.com/openai/codex/issues/19116
  - https://github.com/openai/codex/issues/19372
  - https://github.com/openai/codex/issues/22335
  - https://github.com/openai/codex/issues/22428
  - https://github.com/openai/codex/issues/23902
  - https://github.com/openai/codex/pull/15276

## C.5 同类项目对照

- Claude Code — https://github.com/anthropics/claude-code
- OpenCode — https://github.com/anomalyco/opencode
- Aider — https://github.com/Aider-AI/aider
- goose — https://github.com/aaif-goose/goose
- Continue — https://github.com/continuedev/continue
- Cline — https://github.com/cline/cline

## C.6 协议与规范

- Model Context Protocol — https://modelcontextprotocol.io
- JSON-RPC 2.0 — https://www.jsonrpc.org/specification
- OAuth 2.1 — https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1

## C.7 沙箱与安全

- macOS Seatbelt（`sandbox-exec`）— Apple 内部文档（部分逆向自 WebKit `Sandbox.sb`）
- Bubblewrap — https://github.com/containers/bubblewrap
- Linux Landlock — https://landlock.io
- Windows Filtering Platform — https://learn.microsoft.com/windows/win32/fwp/

## C.8 本研究方法论与同作者前作

- `source-deep-research` skill 7 阶段工作流
- `mermaid-academic` skill 图表规范
- 同作者前作
  - [Claude Code Source Analysis](https://github.com/xiaonancs/claude-code-source-analysis)
  - [Hermes Agent Study](https://github.com/xiaonancs/hermes-agent-study)

---

> 本附录最后复核于 2026-06-25。Continue 官方仓库已进入只读状态；外部链接与项目归属仍可能变化，引用时应记录访问日期。
