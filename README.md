---
description: "Handle authorized GitHub requests in independent DSH sessions and inspect work through a Web panel."
kind: "package-bundle"
---

# dsh-superagent

English | [中文](README.zh.md)

## Summary

Handle requests for one GitHub repository, with optional Feishu input. Superagent evaluates requests under the team's policy, creates work items for authorized work, executes them in independent DSH sessions and directories, and replies in the source conversation. A Web panel shows work that needs a person. Build this research prototype locally before linking it into a separate profile.

## Table of Contents

- [Use this package](#use-this-package)
- [Understand the implementation](#understand-the-implementation)
- [Further Exploration](#further-exploration)
- [Model Experience](#model-experience)
- [Known Limitations and Deferred Work](#known-limitations-and-deferred-work)
- [Dev Note](#dev-note)

-----

<a id="use-this-package"></a>
## Use this package

Use Node.js 24 and pnpm 11.7.0. This fork keeps the prototype's DSH `0.2.1-alpha.2` development dependencies; verify compatibility with your host before using it. From this repository directory, install its frozen dependencies and build:

```sh
pnpm install --frozen-lockfile
pnpm build
```

The package loads `lib/index.js` and `lib/client.js`; Git does not contain these build outputs. Follow [006 Deployment](docs/006-deployment.md#install) to create a separate profile, then link this checkout with `dsh plugin --profile superagent add "link:$PWD"`. Remove the layer with `dsh plugin --profile superagent remove dsh-superagent`. Configure accounts, models and policy before starting work.

Setup has many steps, so an agent does it with you. Ask your coding agent:

> Read SETUP.md in this repository and set up dsh-superagent with me.

It asks one short question at a time, shows the full plan once, and then installs and configures everything. [SETUP.md](SETUP.md) is what it follows. For a checkout managed by Creative, see the [installation and update guide](https://github.com/winterhuan/dsh-creative/blob/develop/docs/user/superagent-installation.md).

-----

<a id="understand-the-implementation"></a>
## Understand the implementation

<details>
<summary>Runtime, composition and development — click to expand</summary>

[The bundle patch](cordis.patch.yml) composes the runtime and Web client. [The entry point](src/index.ts) starts the installation's sources, orchestration and execution services; [007 Architecture](docs/007-architecture.md) maps the modules and tests. The independent [pnpm workspace](pnpm-workspace.yaml) prevents a parent workspace from absorbing this package's dependency installation.

`pnpm check` runs the typecheck, the tests and the build. Creative's batch commands also run these package scripts. Creative checks the bilingual README, while imported source retains its own tests and style conventions; it is excluded from Creative's UI-copy, unknown-assertion and export-JSDoc checks.

</details>

-----

<a id="further-exploration"></a>
## Further Exploration

Read the numbered documents in order; each builds only on the ones before it.

- [000 Philosophy](docs/000-philosophy.md)
- [001 Domain model](docs/001-domain.md)
- [002 Authorization](docs/002-authorization.md)
- [003 Execution](docs/003-execution.md)
- [004 Prompts](docs/004-prompts.md)
- [005 Panel](docs/005-panel.md)
- [006 Deployment](docs/006-deployment.md): manual setup, configuration, upgrades and troubleshooting
- [007 Architecture](docs/007-architecture.md)
- [008 Cookbook](docs/008-cookbook.md): adding a Source

Superagent aims to take as little human attention as possible and give as much value as possible. It acts when asked or when a team rule allows it, and speaks when someone must act or can use a result. The philosophy document explains the operator's responsibilities. This repository uses the [MIT license](LICENSE).

-----

<a id="model-experience"></a>
## Model Experience

### Role context and work

#### What the model sees

The orchestrator and workers receive role instructions, shared routing rules, operational guides, team policy and source context. [Prompt composition](src/app/prompts.ts) resolves built-in parts such as `runtime/shared` and team overrides, and registers source skills with the `dsh-superagent-` prefix. Workers receive the original request and their ticket context; [004 Prompts](docs/004-prompts.md) owns the detailed composition.

#### Token effect

Admission assessments, orchestration and worker execution add separate model calls. Instructions, source context, ticket history and tool results consume input tokens; replies and work consume output tokens. Configuration bounds context and concurrency, but usage depends on the requests and the selected models.

#### KV Cache effect

Unchanged role instructions can form reusable prefixes. Team overrides, policy revisions, source context and ticket history change subsequent request content. Cache reuse depends on the provider; the plugin makes no cache-hit guarantee.

## Known Limitations and Deferred Work

<a id="known-limitations-and-deferred-work"></a>

> **Research prototype for reference only, not an official DeepSeek product.** This code grew out of our team's day-to-day development work, with much of the engineering done by coding agents. Treat it as research code, not production-ready software. It may still contain bugs, and its coding style and engineering practices may be inconsistent. Review and adapt it before using it in your own environment.
>
> We have adapted it for publication, so it is not identical to our internal version or the implementation described in *Exploring Super-agent Design: Towards a More Autonomous Agent System*. Some designs discussed in the article are directions for future development and are not implemented in this prototype.

The fork keeps package version `0.1.0` and the original DSH dependency series. It has no prebuilt Git release or installation preparation script; use the local build. Host compatibility and real-model operation require separate verification. Account credentials and source permissions must be configured for the chosen deployment; linking the package does not complete setup.

<a id="dev-note"></a>
### Dev Note

<details>
<summary>Working context for maintainers — click to expand</summary>

None.

</details>
