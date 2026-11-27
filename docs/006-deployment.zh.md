# 006 部署

[English](006-deployment.md) | 中文

本指南介绍 dsh-superagent 的安装、配置和运行，涵盖账号、启动检查、升级及排障，以 [001 领域模型](001-domain.zh.md)至 [005 面板](005-panel.zh.md)为基础。

## 运行组成

- **独立的 DeepSeek Harness profile**，本指南称为 `superagent`。从 Web profile 模板创建，加载组合包并在 `127.0.0.1` 提供面板。与日常工作的 profile 分开：插件会改变进程全局的 `PATH` 和 Git 设置，其进程需要独立重启及升级。
- **agent（智能体）专用 GitHub 账号。** agent 通过只登录该账号的 GitHub CLI（命令行界面），以该账号评论、推送分支和创建 PR（Pull Request）。
- **可选：** 用于聊天请求的飞书机器人。

插件配置保存状态目录和机器限制。其他设置使用面板：来源配置仓库及账号，策略配置允许名单，执行配置 worker 目录及命令。

### 权限与隔离

组合包为执行器定义 `dsh-superagent-worker`，为主会话定义 `dsh-superagent-router`。它们出现在 DeepSeek Harness 预设列表中，但只供 dsh-superagent 使用。不要为个人会话选择它们，也不要禁用其配置项。应用需要它们来创建和恢复会话。

在 Web UI 中打开空白的托管会话，可能会选中 profile 默认预设。dsh-superagent 在首个轮次前恢复角色专属预设。

两个预设均以操作者的系统用户身份使用 `danger-full-access` 运行，并读取订阅线程的消息。账号包装层默认选择 agent 凭据，但不隔离进程。worker 可以通过路径执行程序，也能读取操作者主目录。需要隔离时，使用只包含 agent 凭据的独立系统账号。

## 前提条件

- Node.js 22.19+ 或 24+、pnpm、git 2.32+、Python 3.8+ 和 tmux
- GitHub CLI（`gh`）2.48 或更高版本。来源 CLI 字段必须指向 `gh` 或 `lark-cli` 包装脚本；宿主依赖它们的子命令和输出格式。
- github.com 上的仓库；不支持 GitHub Enterprise Server。要为每个工单创建检出目录，保留一份本地克隆，让准备命令从中添加 worktree。仓库使用 Git LFS 时，安装 `git-lfs`。
- 为 DeepSeek Harness 配置模型提供商。新建 dsh-superagent 会话使用 profile 默认模型，除非评估或执行设置指定其他模型；恢复的会话保留已保存的模型选择。
- 使用飞书时：`lark-cli`（已用 1.0.90 测试）及启用机器人的飞书或 Lark 应用

## 账号

### agent GitHub 账号

1. 为 agent 创建 GitHub 用户账号，并授予仓库写权限。GitHub App 不能作为 agent，因为 dsh-superagent 读取账号通知。
2. 为它提供只使用自身凭据的 GitHub CLI：

   ```sh
   mkdir -p ~/.local/bin
   cat > ~/.local/bin/gh-agent <<'EOF'
   #!/bin/sh
   GH_CONFIG_DIR="$HOME/.config/gh-agent" exec gh "$@"
   EOF
   chmod +x ~/.local/bin/gh-agent
   gh-agent auth login
   gh-agent api user --jq .login
   ```

   登录 agent 账号；最后一条命令打印其登录名。dsh-superagent 以相同方式读取登录名。

### 飞书机器人（可选）

1. 创建具有机器人能力的自建应用。授予接收消息事件、读取和发送聊天消息、读取群成员的权限，然后发布应用版本。
2. 为机器人提供只允许机器人身份的独立 `lark-cli` 配置及包装脚本：

   ```sh
   LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" lark-cli config init --app-id <cli_…> --app-secret-stdin
   LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" lark-cli config strict-mode bot --global
   cat > ~/.local/bin/lark-bot <<'EOF'
   #!/bin/sh
   export LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent"
   exec lark-cli "$@"
   EOF
   chmod +x ~/.local/bin/lark-bot
   ```

   第一条命令从标准输入读取应用密钥。使用独立配置目录：`lark-cli` 每个 app id 只允许一个 profile，而个人配置可能已包含该应用的用户登录。

   宿主拒绝能以用户身份行动的 CLI。从机器人配置读取 app id、飞书／Lark 品牌及机器人 `open_id`。

   两份配置共享应用密钥的钥匙串条目。只移除机器人配置时，删除其目录；不要运行 `lark-cli config remove`，该命令会影响共享密钥。

<a id="install"></a>

## 安装

在尚无 `superagent` profile 的机器上，将以下命令块运行一次：

```sh
npm install -g @deepseek-ai/dsh@0.2.1-rc.3
git clone <this repository> dsh-superagent && cd dsh-superagent
pnpm install && pnpm build
dsh superagent --from-default-profile web --dump-config > /dev/null
dsh plugin --profile superagent add "link:$PWD"
```

上述固定版本已公开；其他版本必须满足 `package.json` 中的 `engines.dsh`。

第四条命令从 DeepSeek Harness Web 模板创建 `~/.dsh/profiles/superagent`。第五条安装此组合包。设置 `DSH_HOME` 可迁移默认 `~/.dsh` 目录。

只在新安装时运行 profile 创建命令。profile 已存在时，该命令会替换它并删除其 `cordis.patch.yml`，包括机器配置。不要用它升级。

对已有 profile，此 fork 还提供预构建 Git 发布包。它替换本地链接或 tarball 依赖，不改变 profile 配置或状态目录：

```sh
dsh plugin --profile superagent add "github:winterhuan/dsh-superagent-prototype#v0.1.1"
scripts/dsh-superagent --profile superagent restart
```

发布包包含 Host 和 Client 构建产物。插件管理器中的来源为 `github:winterhuan/dsh-superagent-prototype`，带有固定 Git 修订。安装新发布的 ref 并重启即可更新；重新构建另一检出目录不会更新此安装。

## 机器配置

将 [examples/profile.cordis.patch.yml](../examples/profile.cordis.patch.yml)复制到 `~/.dsh/profiles/superagent/cordis.patch.yml`，替换该文件，并替换所有 `<placeholder>`。使用飞书时，还需在其中启用 `dsh-superagent-feishu` 配置项。`webserver` 配置项设置面板端口。

| 键 | 必填 | 含义 |
| --- | --- | --- |
| `stateDir` | 是 | 绝对路径：存放数据库、锁、`worker-bin` 中的命令包装脚本、agent 的 Git 配置，以及默认位于 `work` 中的工单目录。升级前备份 |
| `promptMaxChars`, `admissionContextMaxPages`, `admissionContextTimeoutMs` | 否 | 已保存提示词配置的最大大小，以及一次研判在规定时间内读取的线程页数 |

首次启动后，在**设置 → 部署**或 DeepSeek Harness 插件管理器中编辑机器限制，无需重启即生效。`stateDir` 不能在那里修改。无效值或未知配置键会阻止启动；日志指出对应键。

## 启动和停止

```sh
scripts/dsh-superagent start
scripts/dsh-superagent status
```

`start` 在 tmux 会话 `dsh-superagent` 中运行 `dsh --profile superagent`；其他 profile 使用 `dsh-<profile>`。它最多等待一分钟，直到运行时就绪或报告设置要求。首次运行打印缺失设置，拒绝重复实例，包括未通过本脚本启动的实例。

在 macOS 上，脚本会在服务运行期间阻止系统因空闲而休眠，以便及时接收事件。防休眠保护随服务进程退出而释放；手动让电脑休眠仍会中断连接。

`stop` 和 `restart` 请求正常关闭，并最多等待执行器五分钟。它们绝不强制终止进程，也不影响其他 tmux 会话。

`status` 打印进程 id、健康及设置问题。其他 profile 或端口使用 `--profile` 和 `--url`，或设置 `DSH_SUPERAGENT_PROFILE` 和 `DSH_SUPERAGENT_URL`。

`status`、`start` 和 `restart` 以四种代码之一退出：

| 代码 | 含义 |
| --- | --- |
| 0 | 服务正常运行 |
| 1 | 服务未运行、重复运行、未响应，或运行时报告错误；重启可能解决 |
| 2 | 服务运行，但运行时以外的部分（例如来源或策略同步 `sync:<source id>`）报告错误；重启无法解决 |
| 3 | 服务运行并等待设置；重启无法解决 |

仍在启动的来源算正常运行。设置过程中无法检查的飞书机器人也算正常。

`status` 在 `errors` 下返回失败部分及详情，在 `setup` 下返回设置要求。`start` 和 `restart` 逐行打印这些问题；服务始终未启动时退出 1。只有主动执行 `stop` 才暂停自动恢复。

每个 profile 将日志和控制文件存储在 `.artifacts/` 下：

- 启动日志：`dsh-superagent-<profile>-<date>-<time>-<pid>.log`，其中 pid 属于脚本。保留最新 20 份。
- 控制文件：`control/profiles/<profile>/`，包含停止标记、锁和进程 id。

打开日志打印的 URL，然后在侧栏选择 **Superagent**。

### 每小时检查

在日常使用的 DeepSeek Harness profile 中创建每小时检查，不要放在 `superagent` 中。agent 服务停止时，检查必须仍能运行。让该 profile 中的 agent 使用以下提示词创建每小时计划，并填入检出路径：

```text
Check dsh-superagent: run `scripts/dsh-superagent --profile superagent status` in <checkout>. Exit 0: do nothing and say nothing. Exit 2: do not restart; tell me each part under `errors` with its detail, unless you already told me about the same ones. Exit 3: do not restart; tell me what `setup` says setup needs, unless you already told me. If it reports manually_stopped: true, leave it stopped. Otherwise (exit 1) run `scripts/dsh-superagent --profile superagent --respect-stop restart`, which exits the same way: 0 means it recovered, so say nothing; 2 and 3 are reported as above; if it exits 1 or fails otherwise, read the newest .artifacts/dsh-superagent-superagent-*.log and the Troubleshooting section of docs/006-deployment.md, fix what you can, and tell me what was wrong and what you did.
```

`--respect-stop` 保留主动停止。只有 `stop` 写入标记；`start` 清除标记。`restart` 不写标记，因此检查可以恢复关闭超时的重启。

## 设置

轮询或工作开始前，设置必须成功。概览列出缺失要求并链接到对应设置。修改设置会重新检查。修复应用外的问题，例如 CLI 认证后，选择**重新检查**。

1. **来源。** 设置 GitHub 仓库（`owner/name`）及 agent CLI，例如 `/home/me/.local/bin/gh-agent`。请求通常通过 agent 账号通知到达。新 Issue 或评论提及账号后，在下次轮询到达；通过编辑已有 Issue 正文添加提及则不会到达，应发布新评论。使用飞书时，设置机器人 CLI、群、各群的全部接收或仅提及模式，以及是否接收私信。
2. **策略。** 两条示例规则可以编辑或移除。在**各来源**下添加可以请求工作的 GitHub 账号和飞书用户；名单无人时，请求不能成为工单。飞书可以选择一个群，在启动时及点击**立即同步**时从该群同步名单。
3. **执行。** 设置目录根路径及准备命令。要给每个工单一个仓库检出目录：

   ```sh
   git -C /home/me/src/widgets fetch --no-tags "https://github.com/$DSH_SUPERAGENT_GITHUB_REPOSITORY.git" "${DSH_SUPERAGENT_GITHUB_PR_HEAD:-HEAD}:refs/dsh-superagent/$DSH_SUPERAGENT_TICKET" &&
   git -C /home/me/src/widgets worktree add -b "agent/$DSH_SUPERAGENT_NAME" "$DSH_SUPERAGENT_DIR" "refs/dsh-superagent/$DSH_SUPERAGENT_TICKET"
   ```

   将登录个人账号的 CLI 等个人 CLI 列为禁止命令。
4. **提示词。** 设置团队部分：GitHub 披露行、评审机器人、提交作者和自动合并条件。

### Media Desk 技能

将 Media Desk 安装到它的独立 profile，再把[技能提供器配置层](../examples/media-desk.cordis.patch.yml)追加到 Superagent profile 的 `cordis.patch.yml`。将包目录占位符替换为已安装 Media Desk 包的绝对路径。该层把同一份 `media-desk`、`image-generation` 文件、引用资料和脚本提供给主会话及执行器。修改 profile 配置层后重启 Superagent。

在提示词中添加名为 `media-desk` 的团队 skill，会话将其暴露为 `dsh-superagent-media-desk`。指引需明确绑定 Media Desk 包、数据 home、Wiki 和服务 URL。在范围部分将编辑创作请求指向该 skill，在交付部分要求作品 ID、当前 Notes 路径和接收回执。更新任一插件时，这些配置继续保存在 profile 和提示词存储中。

共用 npm 命令及技能目录中的脚本只提供读取和校验。作品登记、成稿接收、排期和媒体关联使用包根目录下的执行适配：

```sh
node /absolute/media-desk-package/scripts/media-desk-read.mjs read --data-root /absolute/media-home/media-desk --wiki /absolute/wiki --kind works
node /absolute/media-desk-package/scripts/media-desk.mjs read --mode plugin --url http://127.0.0.1:3085 --home /absolute/media-home --wiki /absolute/wiki --login-log /absolute/live.log
```

执行业务动作时将 `read` 换成动作名，并添加 `--input /absolute/input.json`；输入格式见包内 `skills/media-desk/references/plugin-adapter.md`。保留 `--mode plugin` 和明确的 Media Desk 路径，使服务缺失时直接失败，不打开其他 profile 的数据。适配器从实时登录日志私下读取认证；凭据不进入提示词或报告。

新成稿按 `begin` → 策划与写作 → `commit` 执行。接收前将候选稿保存在工单目录：`begin` 后修改已登记 Notes 文件会使冻结的内容版本失效。提交前逐条检查冻结的写作规则。已有成稿使用原生文件工具修改后校验。请求图片时加载 `image-generation`，检查保存的结果，再用新读取的内容版本通过 `asset` 关联。报告完成前核对成稿和媒体接收回执；排期只记录计划，不向平台发布。

### 管理 API

面板使用本地 HTTP API，agent 也可以调用。`SETUP.md` 用它保存设置，服务脚本读取 `GET state`。

向 `http://127.0.0.1:<port>/orchestrator/api/` 发送请求，并携带 `x-orchestrator-request: 1`。host 必须是 `127.0.0.1:<port>` 或 `localhost:<port>`。缺少请求头或 `Origin` 来自其他站点时返回 403。

API 保存具有与面板相同的权限。与主会话工具保存不同，它可以改变执行环境及设置中的账号字段。

agent 可以依赖以下路由和字段。`src/app/api.ts` 中的其他路由供面板使用，随面板变化。

| 请求 | 响应 |
| --- | --- |
| `GET state` | 各页面读取的状态：`setup` 包含 `problem` 和 `feishuProblem`，各自为 `null` 或含 `place`、`code`、`values` 和 `detail` 的设置问题，以及账号解析后的 `accounts`；`health` 为下表各部分的条目，包含 `state`（`starting`、`ready`、`paused` 或 `error`）、`detail` 和 `at`；`load` 包含执行器会话 `running`、工单 `queued` 及 `maxWorkers`；另有 `paused` 和 `masterId` |
| `GET settings` | `blocks`：`policy`、`eval` 和 `executor`，各含 `value` 和 `revision`；`sources`：来源实例，各含 `id`、`kind`、`installed`、配置块的 `value`（`{"enabled": …, "config": {…}}`）和 `revision`、类型表单及存在时的设置 `problem`；另有 `policyVersion` 和 `environment`，后者包含执行环境的 `root` 和 `variables` |
| `GET sources` | 各来源状态：`state`（`missing`、`off`、`error`、`setup`、`starting`、`paused` 或 `ready`）、各部分的 `health`、设置 `problem`、最近一天的 `stats`、当天 `filtered` 的触发，以及当天 `dropped` 的消息。消息按最新丢弃优先排序，每项包含 `channel`、`preview`、所属工单的 `ticket`（`id` 和 `title`，有工单时）、存储中存在时的 `thread`，以及 `dropped.at` 和 `dropped.reason`（`channelGone` 或 `channelIgnored`） |
| `PUT settings/<block>`，携带 `{"value": …, "revision": n}` | 保存一个配置块：`saved`、块的新 `revision`、`policy_changed` 和 `restarting`。`value` 是 `GET settings` 返回的整个块；缺失键采用默认值 |
| `POST executor/pause`，携带 `{"paused": true}` 或 `false` | 暂停或恢复处理：`paused` |
| `POST setup/check` | 重启 dsh-superagent，重新运行设置检查：`restarting: true`。设置无待修复项时拒绝 |
| `POST policy-sync/<source id>` | 立即同步该来源的允许名单：`policy`，即保存的来源策略部分 |
| `GET prompts`、`PUT prompts`，携带 `{"config", "baseRevision"}` | 读取提示词配置：`current` 包含 `revision` 和 `config`，另有 `defaults`、按会话读取顺序排列的团队 `sections`、团队 skill 名称模式 `skillName`、团队 skill 禁用名称 `builtin`，以及会话加载 skill 名称所用的 `skillPrefix`。保存时返回新 `revision` |
| `POST tickets/<id>/retry`、`POST tickets/<id>/stop` | 与面板相同，重试或停止工单 |

健康项按职责分组：

| 条目 | 报告内容 |
| --- | --- |
| `runtime` | 应用自身 |
| `models` | 等待模型预算重试的会话 |
| `evals` | 最近一小时内未完成、且无后续判断或人工忽略处理的评估 |
| `assessment`, `launch`, `delivery`, `relay`, `notice`, `reply` | [003 执行](003-execution.zh.md)描述的存储工作；delivery 统计等待工作，reply 涵盖渠道消息 |
| `worker` | 执行器会话事件处理；首次失败后出现 |
| `github`, `feishu`, `history` | 来源轮询、事件及历史恢复 |
| `sync:<source id>` | 允许名单同步 |

拒绝响应包含状态码及 `error`，后者为英文文本：

- 400：无效值（`issues`，每项包含 `path`）、账号检查失败（`setup`）、固定拒绝原因（`code` 和 `values`），或请求体非 JSON 或过大。
- 404：对象不存在。
- 409：陈旧修订。
- 500：其他情况。

配置块：

- `source:github`、`source:feishu`：`{"enabled": true, "config": {…}}`。GitHub 配置包含 `repository`、`cli`，以及默认关闭的 `observeIssues`、`observePulls`、`observeComments` 和 `observeCi` 开关，用于接收仓库中的所有新 Issue、新 PR、评论及默认分支 CI；飞书配置包含 `cli`、格式为 `[{"chatId": "oc_…", "name": "…", "listen": "mentions"}]`（或 `"all"`）的 `chats`、`directMessages`，以及值为 `"none"` 或 `"mentions"` 的 `otherChats`。
- `policy`：`{"rules": […], "sources": {"github": {"allowed": [{"id": "<login>"}], "bots": [{"id": "dependabot[bot]"}]}, "feishu": {"allowed": [{"id": "ou_…", "name": "…"}], "syncChat": "oc_…", "admission": {"oc_…": {"unlistedAddressed": "explicit"}}}}}`。`bots` 列出其消息可成为推断候选项的机器人。`admission` 行在飞书中以 chat id、`other` 或 `direct` 为键，在 GitHub 中以 `issues`、`pulls`、`comments` 或 `ci` 为键；未设置的单元格取默认值（见 [002 授权](002-authorization.zh.md)）。
- `eval`：`mode`、`timeoutSeconds` 和 `model`。
- `executor`：`model`、`maxWorkers`、`directory`、`prepare` 和 `blockedCommands`。处理暂停是独立设置 `paused`，由 `POST executor/pause` 设置。

修改仓库或 CLI 时先运行对应设置检查。检查失败返回 400，详情在 `setup` 中。账号修改成功，或设置待完成期间的任何保存，返回 `restarting: true`。应用重启期间 API 暂时不可用。

```sh
api=http://127.0.0.1:3082/orchestrator/api
rev=$(curl -s -H 'x-orchestrator-request: 1' $api/settings | jq '.sources[] | select(.id == "github") | .revision')
curl -s -X PUT -H 'x-orchestrator-request: 1' -H 'content-type: application/json' $api/settings/source:github \
  -d "{\"value\": {\"enabled\": true, \"config\": {\"repository\": \"acme/widgets\", \"cli\": \"$HOME/.local/bin/gh-agent\"}}, \"revision\": $rev}"
```

## 设置检查

设置检查与 worker shell 一样使用 DeepSeek Harness 子进程服务，移除名称包含 KEY、PASSWORD、SECRET 或 TOKEN 的环境变量。

设置检查验证 GitHub 仓库及 agent 推送权限，写入 worker 环境，并检查 Git 对 github.com 只使用 agent 凭据。飞书设置失败只禁用飞书来源。

设置检查写入的进程环境：

- **命令包装脚本**位于 `<stateDir>/worker-bin`，排在 `PATH` 首位：`gh` 运行 agent CLI；`lark-cli` 运行机器人 CLI；禁止命令失败。包装脚本运行目标前，先从 `PATH` 移除自身目录。
- **Git 配置**位于 `<stateDir>/worker-gitconfig`：替代操作者全局及系统 Git 配置，选择 agent CLI 作为 github.com 凭据助手，将 GitHub SSH 地址改写为 HTTPS，并定义 Git LFS 过滤器。HTTP 代理等个人 Git 设置不生效。此处不设置提交身份，Git 绝不提示输入凭据，且移除 `GH_TOKEN` 和 `GITHUB_TOKEN`。

`worker-bin` 或 `worker-gitconfig` 中出现非 dsh-superagent 写入的文件会阻止设置检查。工单目录位于 Git 仓库内，且仓库自身配置添加凭据时，工单失败。

## 试用

从允许的 GitHub 账号创建 Issue 或发布包含 `@<agent login> <request>` 的评论。一个轮询间隔内（默认一分钟），概览应显示工单。agent 确认请求，在自身会话和目录中工作，并在原始对话中交付。飞书中，在已配置群内提及机器人。

## 升级

1. 等待所有执行器均不处于轮次中途。
2. 备份 `<stateDir>/orchestrator.sqlite`。
3. 运行 `git pull && pnpm install && pnpm build`。
4. 运行 `scripts/dsh-superagent restart` 并查看状态。

重启结束运行中的派工，并启动带有恢复说明的替代派工。团队文本及覆盖的提示词部分保留；未覆盖的部分跟随新默认值。升级后检查标记为**默认内容已更新**的部分。

新版本迁移的数据库不能由旧版本打开。因此回滚需要升级前备份。升级 DeepSeek Harness 本身后，重启运行实例，让面板加载匹配产物。

### agent 预设

worker 和 router 预设在 shell 命令运行 5 秒后将其转为后台。会话缺少预设时无法加载。会话仍在使用时，保留 `cordis.patch.yml` 中的 `dsh-superagent-worker` 和 `dsh-superagent-router` 配置项，否则恢复会报 `agent-preset/not-found`。

## 排障

设置问题以面板语言出现在概览上，并链接到修复位置；日志和 API 显示下表中的英文文本。

| 症状 | 原因与解决方法 |
| --- | --- |
| `The agent GitHub CLI is not set.` | 在 GitHub 卡片上设置 agent CLI |
| `The agent GitHub CLI was not found.` | 设置包装脚本的绝对路径，或 `PATH` 中的命令名 |
| `The agent GitHub CLI cannot read its account` | 用 `auth login` 登录包装脚本 |
| `@… cannot read …` 或 `@… cannot push to …` | 授予 agent 账号仓库写权限 |
| `The GitHub Source names no repository.` | 在 GitHub 卡片上设置仓库 `owner/name` |
| 工单报 `The prepare command failed: …` | 输出说明原因，例如克隆路径错误；修复命令后重试工单 |
| `git does not read the github.com credential configuration dsh-superagent writes` | 安装 git 2.32 或更高版本 |
| 工单报 `The git configuration of the repository in … sets …` | 从克隆的 `.git/config` 移除指定凭据或 `http.*.extraheader` 设置 |
| `… was not written by dsh-superagent` | 将指定文件移出 `<stateDir>/worker-bin` 或 `<stateDir>/worker-gitconfig`，或选择其他 `stateDir` |
| `The bot lark-cli allows user identities` | 按上文飞书机器人部分，将飞书来源指向仅机器人配置 |
| `The bot lark-cli does not work.` 或 `The bot identity of app … is …` | 检查配置的 app id 及密钥 |
| `The Feishu bot is not a member of the chat.` | 在群设置中添加机器人；下次尝试成功 |
| `status` 退出 2 | 某部分报告错误，重启无法解决：`errors` 按管理 API 中的列表列出各部分及详情；各部分恢复正常后自行清除条目 |
| `status` 退出 3 | 设置等待中：`setup` 说明缺失内容，概览链接到可解决问题的设置 |
| `status` 退出 1 | 服务未运行、重复运行、未响应或运行时报告错误；输出和最新日志说明具体情况 |
| `… timed out after 90 s`、`… was stopped by …`、`… exited with code …` 或 `… could not run: …` | CLI 失败但未打印原因；以操作者用户身份手动运行指定程序，查看其消息 |
| 面板显示 dsh-superagent 未响应 | 应用正在重启，或因配置未能启动；DeepSeek Harness 日志说明原因 |
| `dsh-superagent configuration has keys it does not use`（日志） | 移除指定键；消息说明各值现在所属位置 |
| 日志显示插件与运行中的 dsh 不兼容 | 安装满足 `engines.dsh` 范围的 DeepSeek Harness 版本 |
| DeepSeek Harness 升级后面板显示「Failed to load plugins」 | 重启运行实例 |
| 提及后没有反应 | 检查作者在来源允许名单中、来源已开启且处理未暂停。GitHub 不接收通过编辑 Issue 正文添加的提及；在评论中提及 agent |
| 已接收的触发反复超时，主会话报 `NO_ADAPTER` | 会话保存的 provider 已不存在。在主会话及受影响的执行器会话中重新选择可用模型，并为新会话设置评估与执行模型。然后对触发选择**重新评估** |
| 模型请求报 HTTP 503 | 所选模型服务不可用。将受影响的会话及角色默认模型切换到可用的 provider 和模型。验证一次实际模型回复；`status` 成功或模型列表加载成功不能验证模型调用 |
