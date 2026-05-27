# OpenAI Codex 源码深度研究

> 这份研究覆盖 OpenAI Codex CLI / Rust 主体 / TS+Python SDK 全貌：**2 篇总纲 + 25 章正文 + 3 个附录**，约 **51 万中文字 / 1.5 MB Markdown** 与 **149 张 Mermaid 架构图**。

## 引言

OpenAI Codex 在 2025–2026 年从一个早期 TS 原型重写为以 Rust 为主体的多入口 harness：

- **codex-cli**（npm）— 跨平台启动器，把请求转交给原生二进制
- **codex-rs**（Rust workspace，~120 个 crate）— 真正的核心：CLI、TUI、Agent 循环、App-Server JSON-RPC、沙箱、Plugin 市场、MCP、Cloud Tasks
- **sdk/typescript** 与 **sdk/python** — 通过 spawn CLI 或 App-Server JSON-RPC 暴露给外部应用

本研究分三部分对其进行系统性源码层面解析：
<br>`Part I 使用方法与原理（8 章）`
<br>`Part II 源码解析（13 章）`
<br>`Part III 对比与延展（4 章）`

生成流程采用三阶段写作：
- **第一稿**：GPT-5.3 Codex High（章节 01-15 + 总纲 + 全网调研）/ Claude Opus 4.7 Thinking High（章节 16-25）
- **review 与优化升级**：Claude Opus 4.7 Thinking High 对 17 篇做源码引用核验 + 七维框架补全 + Mermaid 加固
- **最终质量检测**：GPT-5.5 High 对全 27 篇做 Mermaid 合规、跨章一致性、措辞审慎度收束

## 研究结构

### 总纲

| 文档 | 内容 |
|------|------|
| [总纲 — Codex 技术主线分析](总纲-Codex技术主线分析.md) | 核心机制、设计哲学、火爆原因、整体架构 |
| [全网调研 — 社区认知地图](全网调研-社区认知地图.md) | 中英文社区技术分析索引、观点争议、认知盲区 |

### Part I 使用方法与原理（8 章）

| 序号 | 章节 | 关键源码 |
|------|------|----------|
| 01 | [项目全景与设计哲学](Part%20I%20Principles%20and%20Usage/01-项目全景与设计哲学.md) | `codex-rs/Cargo.toml`、`AGENTS.md` |
| 02 | [多入口与启动分发](Part%20I%20Principles%20and%20Usage/02-多入口与启动分发.md) | `codex-cli/bin/codex.js`、`codex-rs/cli/src/main.rs`、`arg0/` |
| 03 | [配置系统与企业要求](Part%20I%20Principles%20and%20Usage/03-配置系统与企业要求.md) | `core/src/config/mod.rs`、`cloud-requirements/` |
| 04 | [初级使用方法](Part%20I%20Principles%20and%20Usage/04-初级使用方法.md) | `docs/getting-started.md`、`login/` |
| 05 | [高级使用方法](Part%20I%20Principles%20and%20Usage/05-高级使用方法.md) | `cloud-tasks/`、`cli/src/mcp_cmd.rs`、`debug_sandbox.rs` |
| 06 | [Agent 核心循环](Part%20I%20Principles%20and%20Usage/06-Agent核心循环.md) | `core/src/session/mod.rs`、`turn.rs` |
| 07 | [Prompt 组装与 Skill 注入](Part%20I%20Principles%20and%20Usage/07-Prompt组装与Skill注入.md) | `core/gpt_5_codex_prompt.md`、`core-skills/` |
| 08 | [Provider 与 Responses/Realtime API](Part%20I%20Principles%20and%20Usage/08-Provider与API模式.md) | `model-provider/`、`codex-api/`、`realtime-webrtc/` |

### Part II 源码解析（13 章）

| 序号 | 章节 | 关键源码 |
|------|------|----------|
| 09 | [工具系统总览](Part%20II%20Source%20Analysis/09-工具系统总览.md) | `tools/`、`core/src/tools/handlers/` |
| 10 | [命令执行与 unified_exec](Part%20II%20Source%20Analysis/10-命令执行与unified_exec.md) | `core/src/exec.rs`、`shell-command/parse_command.rs` |
| 11 | [apply_patch 工具](Part%20II%20Source%20Analysis/11-apply_patch工具.md) | `apply-patch/` |
| 12 | [macOS Seatbelt 与 Linux Bwrap 沙箱](Part%20II%20Source%20Analysis/12-macOS与Linux沙箱.md) | `sandboxing/`、`linux-sandbox/`、`bwrap/` |
| 13 | [Windows 沙箱与 WFP 防火墙](Part%20II%20Source%20Analysis/13-Windows沙箱与WFP防火墙.md) | `windows-sandbox-rs/` |
| 14 | [执行策略 Starlark execpolicy](Part%20II%20Source%20Analysis/14-执行策略Starlark.md) | `execpolicy/`、`execpolicy-legacy/` |
| 15 | [网络代理与策略](Part%20II%20Source%20Analysis/15-网络代理与策略.md) | `network-proxy/` |
| 16 | [Hook 与生命周期事件](Part%20II%20Source%20Analysis/16-Hook与生命周期事件.md) | `hooks/` |
| 17 | [Plugin 市场系统](Part%20II%20Source%20Analysis/17-Plugin市场系统.md) | `core-plugins/` |
| 18 | [MCP 双向集成](Part%20II%20Source%20Analysis/18-MCP双向集成.md) | `codex-mcp/`、`rmcp-client/`、`mcp-server/` |
| 19 | [会话与轨迹持久化](Part%20II%20Source%20Analysis/19-会话与轨迹持久化.md) | `rollout/`、`rollout-trace/`、`thread-store/`、`state/` |
| 20 | [记忆系统](Part%20II%20Source%20Analysis/20-记忆系统.md) | `memories/{read,write,mcp}/`、`state/runtime/memories.rs` |
| 21 | [App-Server JSON-RPC 协议层](Part%20II%20Source%20Analysis/21-AppServer协议层.md) | `protocol/`、`app-server/`、`app-server-protocol/v2/` |

### Part III 对比与延展（4 章）

| 序号 | 章节 | 关键源码 |
|------|------|----------|
| 22 | [TUI 渲染管线与 Code Mode V8](Part%20III%20Comparative%20Analysis/22-TUI与CodeMode.md) | `tui/`、`code-mode/` |
| 23 | [Cloud Tasks 与外部 Agent 迁移](Part%20III%20Comparative%20Analysis/23-CloudTasks与外部Agent迁移.md) | `cloud-tasks/`、`external-agent-migration/`、`agent-graph-store/` |
| 24 | [Codex vs Claude Code / Opencode 架构对比](Part%20III%20Comparative%20Analysis/24-Codex与同类对比.md) | 横向对比 |
| 25 | [Codex 沙箱与权限模型 vs 同类](Part%20III%20Comparative%20Analysis/25-Codex沙箱与权限对比.md) | 沙箱与权限模型横评 |

### 附录

| 文档 | 内容 |
|------|------|
| [附录 A](Appendix/A-章节配置.yaml) | 章节配置元数据 |
| [附录 B](Appendix/B-关键数据结构索引.md) | 核心 struct/enum/trait 速查表 |
| [附录 C](Appendix/C-参考文献.md) | 参考文献与引用来源 |

---

## 源码基线

| 项目 | 规模 | 语言 |
|------|------|------|
| codex-rs | ~120 crate | Rust |
| codex-cli | npm 启动器 | TypeScript / Node |
| sdk/typescript | App-Server / spawn 客户端 | TypeScript |
| sdk/python | App-Server / spawn 客户端 (含 generated 模型) | Python |
| docs | 16 篇官方文档 | Markdown |

**三大量级集群（行数）**

| Crate | LOC |
|-------|-----|
| `codex-rs/tui` | 193,863 |
| `codex-rs/core` | 151,372 |
| `codex-rs/app-server` | 39,229 |
| `codex-rs/core-plugins` | 21,197 |
| `codex-rs/windows-sandbox-rs` | 15,902 |
| `codex-rs/state` | 15,480 |

## 研究规模

| 维度 | 数量 |
|------|------|
| 总文档数 | 30 (README + 2 总纲 + 25 章 + 3 附录) |
| 总字节数 | ~1.54 MB |
| 中文字数（估计） | ~51 万字 |
| Mermaid 图表 | 149 张 |
| 平均每章字数 | ~17,000 字 |

## 怎么读

- **15 分钟掌握全貌**：先读 [总纲](总纲-Codex技术主线分析.md)
- **快速上手**：Part I 第 4-5 章
- **理解架构**：Part I 第 6-8 章 + Part II 第 21 章（App-Server 协议层）
- **沙箱与权限专题**：Part II 第 12-15 章 + Part III 第 25 章
- **协议演进**：Part II 第 18 章 (MCP) + 第 21 章 (App-Server) + Part III 第 22 章 (Code Mode V8)
- **横向研究**：Part III 第 24-25 章

## 工程化

```bash
# 校验所有 Mermaid 图表语法
npm install
npm run validate:mermaid
```

当前所有 mermaid 块（149 张架构 / 流程 / 时序 / 状态 / ER 图）均通过 `mermaid@11.x` 解析。

## 写作方法论

本研究遵循 `source-deep-research` 7 阶段工作流：

```
阶段 1  通读源码，建立 25 章骨架（chapters.yaml）
阶段 2  全网调研，建立外部认知基线（全网调研.md）
阶段 3  设计七维结构化 Prompt 集
阶段 4  cursor-agent 串行批量生成（GPT-5.3 Codex / Opus 4.7）
阶段 5  Opus 4.7 review + 改进；GPT-5.5 最终质量检测
阶段 6  系统化整理（README / 章节编号 / 文风 / 图表）
阶段 7  开源发布
```

每章遵循"七维分析框架"：本质 → 核心痛点 → 解决思路 → 实现细节 → 易错点 → 竞品对比 → 仍存缺陷。

## 同作者前作

- [Claude Code Source Analysis](https://github.com/xiaonancs/claude-code-source-analysis)
- [Hermes Agent Study](https://github.com/xiaonancs/hermes-agent-study)

## 许可

本研究内容采用 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) 许可证。所分析的源码版权归 OpenAI 所有。
