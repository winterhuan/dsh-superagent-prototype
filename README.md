---
description: "Handle authorized GitHub requests in independent DSH sessions and inspect work through a Web panel."
kind: "package-bundle"
---

# dsh-superagent

English | [中文](README.zh.md)

## Summary

Handle requests for one GitHub repository, with optional Feishu input. Superagent evaluates requests under the team's policy, creates work items for authorized work, executes them in independent DSH sessions and directories, and replies in the source conversation. A Web panel shows work that needs a person. Install the prebuilt fork into a separate profile; source development uses a local build.

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

Use Node.js 24 and pnpm 11.7.0. This fork requires DSH `0.2.2-rc.5`; its development and peer dependencies use the same exact version as the host. Follow [006 Deployment](docs/006-deployment.md#install) to create a separate profile, then install the prebuilt Git release:

```sh
dsh plugin --profile superagent add "github:winterhuan/dsh-superagent-prototype#v0.1.2"
```

For source development, install the frozen dependencies and rebuild from this repository directory:

```sh
pnpm install --frozen-lockfile
pnpm build
```

The Git release includes `lib/index.js` and `lib/client.js`, so installation does not run a build. To load a development build, link this checkout with `dsh plugin --profile superagent add "link:$PWD"`. Remove the layer with `dsh plugin --profile superagent remove dsh-superagent`. Configure accounts, models and policy before starting work.

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

- DSH dependencies must match the host version exactly; other host versions require a compatibility update.
- Runtime changes require rebuilding and publishing a new immutable Git ref.
- Host compatibility and real-model operation require separate verification.
- Account credentials and source permissions must be configured for the chosen deployment; installing the package does not complete setup.

<a id="dev-note"></a>
### Dev Note

<details>
<summary>Working context for maintainers — click to expand</summary>

None.

</details>
