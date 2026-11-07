# dsh-superagent 中文文档

这是 dsh-superagent 英文文档的中文译本。内容与仓库根目录的 `README.md`、`SETUP.md` 以及 `docs/` 下的编号文档一一对应。

> **本目录只是译文。** 译文与英文原文不一致时，以英文原文为准。代码、命令、路径、环境变量、API 路由与字段名、程序输出的英文文字都不翻译，保持原样。

## 目录

- [项目说明](#项目说明)：dsh-superagent 是什么
- [安装设置](#安装设置)：由智能体陪你一起完成安装
- 编号文档：000 至 008，见 [文档](#文档)
- [术语](#术语)
- [开发](#开发)与 [许可证](#许可证)

## 项目说明

> **仅供参照的研究原型，不是 DeepSeek 官方产品。** 这份代码来自我们团队的日常开发工作，其中大量工程由编码智能体完成。请把它当作研究代码，而不是可用于生产的软件：它可能仍有缺陷，编码风格与工程实践也可能不够统一。在你自己的环境里使用之前，请先审阅并做相应调整。
>
> 为了对外发布，我们做过适配，因此它与我们的内部版本、以及《Exploring Super-agent Design: Towards a More Autonomous Agent System》中描述的实现并不完全相同。文中讨论的部分设计属于后续方向，本原型尚未实现。

dsh-superagent 是一个面向单个 GitHub 仓库的主动型编码智能体，以 [DeepSeek-Harness](https://www.deepseek.com/en/harness/) 的 bundle 形式发布。它监听 GitHub，也可以选择监听飞书。当被允许的人向它请求工作时，它开出一张工单项，在自己的智能体会话与目录里完成这项工作，并回到原来的对话里答复，例如给出一个 pull request。一个网页面板展示哪些事情需要人来处理。

它同时也是一种与智能体协作的方式：**尽量少占用人的注意力，尽量多地交付价值。** 能为整个团队行动的智能体，也能浪费整个团队的注意力，所以它只在被请求时行动，或只在团队写下的规则允许时行动；它只在必须有人处理、或有人能用上结果时才开口。

## 安装设置

安装步骤不少，所以由智能体陪你完成。对你的编码智能体说：

> 读这个仓库里的 SETUP.md，和我一起安装 dsh-superagent。

它会一次只问一个简短的问题，中间把完整计划给你看一遍，然后安装并配置好全部内容。它遵循的是 [SETUP.md](SETUP.md)（中文版：[安装设置](SETUP.md)）。

## 文档

按顺序阅读编号文档，每篇只依赖它前面的文档。

- [000 设计理念](000-philosophy.md)
- [001 领域模型](001-domain.md)
- [002 授权](002-authorization.md)
- [003 执行](003-execution.md)
- [004 提示词](004-prompts.md)
- [005 面板](005-panel.md)
- [006 部署](006-deployment.md)：手工安装、配置、升级与故障排查
- [007 架构](007-architecture.md)
- [008 操作手册](008-cookbook.md)：新增一个来源

## 术语

术语按 [001 领域模型](001-domain.md) 的术语表统一：

| 英文 | 中文 |
| --- | --- |
| Agent | 智能体 |
| Main Session | 主 Session |
| Executor Session | 执行器 Session |
| Source | 来源 |
| Trigger | 触发 |
| Thread | 线程 |
| Correlation | 关联 |
| Channel | 渠道 |
| Policy | 策略 |
| Policy version | 策略版本 |
| Eval | 评估 |
| Gate | 准入 |
| Assessment | 研判 |
| Decision | 决定 |
| Mandate | 授权 |
| Ticket | 工单 |
| Assignment | 派工 |
| Subscription | 订阅 |
| Fan-out | 扇出 |
| Executor | 执行器 |
| Execution environment | 执行环境 |
| Report | 报告 |
| Attention | 待人工决定 |
| Disposition | 去向 |
| Input | 输入 |
| Scope | 范围 |
| Strength | 强度 |

枚举值保持中英对照，例如 Decision 的 `accept` 接单、`stop` 停止、`ignore` 忽略、`unavailable` 未完成。

## 开发

```bash
pnpm check
```

## 许可证

MIT，见 [LICENSE](../../LICENSE)。

## 阅读顺序

编号文档按顺序阅读，每篇只依赖它前面的文档。英文原文见仓库根目录与 `docs/`。
