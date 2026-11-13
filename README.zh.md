---
description: "在独立 DSH 会话中处理已授权的 GitHub 请求，并通过 Web 面板查看工作。"
kind: "package-bundle"
---

# dsh-superagent

[English](README.md) | 中文

## 概述

处理一个 GitHub 仓库的请求，可选接入飞书。Superagent 按团队策略评估请求，为已授权工作创建工作项，在独立 DSH 会话和目录中执行，并在来源讨论中回复。Web 面板展示需要人工处理的工作。先在本地构建这个研究原型，再将它链接到独立 profile。

## 目录

- [使用此包](#use-this-package)
- [理解实现](#understand-the-implementation)
- [进一步探索](#further-exploration)
- [模型体验](#model-experience)
- [已知限制与延后工作](#known-limitations-and-deferred-work)
- [开发备注](#dev-note)

-----

<a id="use-this-package"></a>
## 使用此包

使用 Node.js 24 和 pnpm 11.7.0。这个 fork 保留原型的 DSH `0.2.1-alpha.2` 开发依赖；使用前核对与宿主的兼容性。在本仓库目录安装冻结依赖并构建：

```sh
pnpm install --frozen-lockfile
pnpm build
```

包加载 `lib/index.js` 和 `lib/client.js`；Git 中不包含这些构建产物。按 [006 部署](docs/006-deployment.md#install)创建独立 profile，再用 `dsh plugin --profile superagent add "link:$PWD"` 链接此检出目录。用 `dsh plugin --profile superagent remove dsh-superagent` 移除该层。开始工作前配置状态目录、账号、模型和策略。

设置包含多个步骤，由 agent（智能体）与你共同完成。向 coding agent（编程智能体）提出：

> 阅读本仓库的 SETUP.md，与我一起设置 dsh-superagent。

它每次只问一个简短问题，完整展示一次方案，然后安装并配置所有内容。[SETUP.md](SETUP.md) 是它遵循的流程。Creative 管理的检出目录另见[安装与更新指南](https://github.com/winterhuan/dsh-creative/blob/develop/docs/user/superagent-installation.zh.md)。

-----

<a id="understand-the-implementation"></a>
## 理解实现

<details>
<summary>运行时、组合与开发——点击展开</summary>

[组合包补丁](cordis.patch.yml)组合运行时与 Web 客户端。[入口](src/index.ts)启动当前安装的来源、编排与执行服务；[007 架构](docs/007-architecture.md)映射模块与测试。独立的 [pnpm workspace](pnpm-workspace.yaml)避免父级 workspace 接管此包的依赖安装。

`pnpm check` 运行类型检查、测试和构建。Creative 的批量命令也运行这些包脚本。Creative 检查双语 README；导入源码保留自身测试与风格约定，不参与 Creative 的 UI 文案、unknown 类型断言和导出 JSDoc 检查。

</details>

-----

<a id="further-exploration"></a>
## 进一步探索

按顺序阅读编号文档；每篇只依赖它之前的文档。

- [000 理念](docs/000-philosophy.md)
- [001 领域模型](docs/001-domain.md)
- [002 授权](docs/002-authorization.md)
- [003 执行](docs/003-execution.md)
- [004 提示词](docs/004-prompts.md)
- [005 面板](docs/005-panel.md)
- [006 部署](docs/006-deployment.md)：手动设置、配置、升级与排障
- [007 架构](docs/007-architecture.md)
- [008 实操手册](docs/008-cookbook.md)：添加来源

Superagent 的目标是尽量少占用人的注意力，尽量多提供价值。它在收到请求或团队规则允许时行动，在有人必须操作或能使用结果时发言。理念文档说明操作者的责任。本仓库采用 [MIT 许可](LICENSE)。

-----

<a id="model-experience"></a>
## 模型体验

### 角色上下文与工作

#### 模型看到什么

编排器与执行者接收角色指令、共享路由规则、操作指南、团队文本和执行环境设置。编排器还接收团队策略。[提示词组合](src/app/prompts.ts)解析 `runtime/shared` 等内置部分与团队覆盖，并用 `dsh-superagent-` 前缀注册来源 skill（技能）。执行者接收原始请求与工作项上下文；[004 提示词](docs/004-prompts.md)维护详细组合说明。

#### Token 影响

准入评估、编排和执行者工作增加独立模型调用。指令、来源上下文、工作项历史和工具结果消耗输入 token；回复与工作消耗输出 token。配置约束上下文和并发，但用量取决于请求和所选模型。

#### KV Cache 影响

未变的角色指令可形成可复用前缀。团队覆盖、策略版本、来源上下文和工作项历史会改变后续请求内容。缓存复用取决于提供商；插件不保证缓存命中率。

## 已知限制与延后工作

<a id="known-limitations-and-deferred-work"></a>

> **仅供参考的研究原型，并非 DeepSeek 官方产品。** 此代码来自我们团队的日常开发工作，其中大量工程工作由 coding agent 完成。请将其视为研究代码，尚未达到生产可用水平。它可能仍有缺陷，编码风格与工程实践也可能不一致。在自己的环境使用前，请审查并适配。
>
> 我们为公开发布调整了实现，因此它与内部版本或《Exploring Super-agent Design: Towards a More Autonomous Agent System》描述的实现并不相同。文章中讨论的某些设计属于未来方向，尚未在此原型中实现。

- fork 保留包版本 `0.1.0` 和原有 DSH 依赖系列。
- 它没有预构建 Git 发布包或安装准备脚本；使用本地构建。
- 宿主兼容性和真实模型运行需单独验证。
- 必须为选定部署配置账号凭据与来源权限；链接包并不等于完成设置。

<a id="dev-note"></a>
### 开发备注

<details>
<summary>维护者工作上下文——点击展开</summary>

无。

</details>
