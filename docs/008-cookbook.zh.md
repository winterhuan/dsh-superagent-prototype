# 008 实操手册

[English](008-cookbook.md) | 中文

本实操手册（cookbook）介绍如何扩展 dsh-superagent，使用 [001 领域模型](001-domain.zh.md)和 [002 授权](002-authorization.zh.md)中的术语，以及 [007 架构](007-architecture.zh.md)中的组件。

## 添加来源

本例实现名为 `webhook` 的来源类型，通过虚构的 CLI（命令行界面）`hookq` 读取假想消息队列，将消息转换为触发。其他所有接口均属于 dsh-superagent。

实现有四项职责：转换事件、定义接收和授权规则、提供回复及上下文、维护轮询循环。以下编号步骤依次构建这些部分。

示例假定 CLI 遵守以下约定：

- `hookq pull --after <n> --json` 按最早优先打印位置 `n` 之后的消息：`{"me": "<the agent's id on the queue>", "items": […], "next": <position>}`。每项包含 `id`、`conversation`、`thread`（起始消息 id，起始消息自身无此字段）、带 `id` 和 `bot` 的 `sender`、`mentions_me`、`text`、`at` 和 `url`。
- `hookq reply --conversation <id> --to <message id> --key <key> --text <text>` 在消息下回复，每个 key 只发送一次。对已删除对话，在标准错误输出 `{"error": {"code": 410, "message": "…"}}` 并失败。
- `hookq thread --conversation <id> --root <message id> --json` 按最早优先打印一页线程：`{"items": […], "next": "<cursor>"}`，最后一页省略 `next`；后续页使用 `--after <cursor>`。

参照 `src/sources/github.ts`，将类型和插件入口放入 `src/sources/webhook.ts`。来源变复杂时，像飞书来源一样将转换、渠道和轮询代码拆为独立模块。

### 1. 将事件映射到领域

编写代码前选择稳定映射。触发 id、线程 id 和渠道地址会持久化，之后修改可能使已有记录断开联系。

| 概念 | webhook 队列中的对应内容 |
| --- | --- |
| 触发 | 一条消息 |
| 线程 | 同一对话中的一条消息及其回复：`<conversation>:<id of the first message>` |
| 行为者 | 发送者在队列中的 id，与允许名单一致 |
| 渠道地址 | 对话及回复所在的消息 |
| 接收范围 | 一个对话，在准入表中有自己的行 |

### 2. 配置：schema、字段、事实及健康

模块导入后续步骤使用的内容：

```ts
import type { Context } from '@deepseek-ai/cordis'
import { setTimeout as delay } from 'node:timers/promises'
import { z } from 'zod'
import { key, threadKey, type OutboundMessage, type Trigger } from '../core/index.ts'
import { CliError } from '../infra/commands.ts'
import { KV } from '../kv.ts'
import { health, sourceContext, type SourceContext } from './host.ts'
import type { SourceKind } from './kind.ts'
```

```ts
const ConfigSchema = z.object({
  conversations: z.array(z.object({
    id: z.string().regex(/^[a-z0-9-]+$/),
    listen: z.enum(['mentions', 'all']),
  }).strict()).default([]),
  pollSeconds: z.number().int().min(10).max(3600).default(30),
}).strict()
export type WebhookConfig = z.infer<typeof ConfigSchema>

export const webhookKind: SourceKind<WebhookConfig> = {
  kind: 'webhook',
  title: { zh: 'Webhook 队列', en: 'Webhook queue' },
  description: { zh: '接收 webhook 队列里下列对话的消息。', en: 'Receives the messages of these conversations from the webhook queue.' },
  schema: ConfigSchema,
  fields: [
    { key: 'conversations', type: 'rows', label: { zh: '接收的对话', en: 'Conversations' }, columns: [
      { key: 'id', type: 'text', label: { zh: '对话', en: 'Conversation' }, pattern: '[a-z0-9-]+' },
      { key: 'listen', type: 'select', label: { zh: '接收', en: 'Receive' }, options: [
        { value: 'mentions', label: { zh: '仅 @', en: '@ only' } }, { value: 'all', label: { zh: '全部消息', en: 'All' } }] },
    ] },
    { key: 'pollSeconds', type: 'number', advanced: true, min: 10, max: 3600,
      label: { zh: '轮询间隔 / 秒', en: 'Polling interval / seconds' } },
  ],
  health: ['webhook'],
  // find, scope, channel, threadContext, policy and environment follow in the next steps.
}
```

- `kind` 是稳定来源 id，出现在触发 `source`、线程键前缀、`source:webhook` 设置块及来源策略中。
- `schema` 验证存储配置并提供默认值。操作者保存实例前，注册表按默认值创建启用实例。默认值应不接收任何内容；本例默认无对话。
- `fields` 无需专门面板代码即可生成来源设置表单。`rows` 字段编辑以首列为键的条目；`advanced` 隐藏不常用设置。来源可将 id 解析为显示名时，实现 `refreshLabels`，将名称保存在 `KV.sources.labels(kind)` 下；带 `choose` 和 `refresh` 的列随后可以提供这些名称。
- 可选 `facts(deployment)` 显示设置检查派生的值，例如 CLI 对应账号。设置检查目前只检查 `SETUP_FIELDS` 列出的 GitHub 和飞书字段，因此本例不提供事实。
- `health` 声明来源健康项，初始状态为 `starting`。注册认领各名称，拒绝与 `runtime`、`reply`、`worker` 或副作用类型等条目冲突。使用类型名称，并将 `sync:<id>` 留给策略同步。`error` 会使状态脚本退出 2，并让每小时检查通知操作者，因此只在可能需要人工操作时报告。

### 3. 将消息转换为触发

转换前验证 CLI 响应：

```ts
/** One message as `hookq pull` and `hookq thread` print it. */
const MessageSchema = z.looseObject({
  id: z.string().min(1), conversation: z.string().min(1), thread: z.string().optional(),
  sender: z.looseObject({ id: z.string(), bot: z.boolean().default(false) }), mentions_me: z.boolean().default(false),
  text: z.string(), at: z.string(), url: z.string().optional(),
})
type QueueMessage = z.infer<typeof MessageSchema>
/** The answer of `hookq pull`, whose items are read one by one, and of `hookq thread`. */
const PageSchema = z.looseObject({ me: z.string(), items: z.array(z.unknown()), next: z.number().int() })
const ThreadPageSchema = z.looseObject({ items: z.array(MessageSchema), next: z.string().optional() })
```

```ts
/** The Trigger of one queue message; the agent's own messages produce none. */
export function webhookTrigger(message: QueueMessage, me: string): Trigger | undefined {
  if (message.sender.id === me) return undefined
  const thread = `${message.conversation}:${message.thread ?? message.id}`
  return {
    id: key('webhook', message.id),
    source: 'webhook',
    threads: [{ source: 'webhook', id: thread }],
    actor: { source: 'webhook', id: message.sender.id },
    human: !message.sender.bot,
    ...(message.mentions_me ? { addressed: 'mention' as const } : {}),
    at: message.at,
    text: message.text,
    channel: { source: 'webhook', thread: `webhook:${thread}`,
      address: { conversation: message.conversation, message: message.id, ...(message.url ? { url: message.url } : {}) } },
    ...(message.url ? { url: message.url } : {}),
    meta: { conversation: message.conversation, messageId: message.id, root: message.thread ?? message.id },
  }
}
```

- **稳定 id。** 通过 `key` 从队列 id 推导触发 id。`Engine.receive` 对已存 id 返回 `duplicate`，重复接单只创建一个工单，因此重新读取安全。要将编辑视为新事件，在 id 中加入编辑版本，如 GitHub 评论的 `updated_at`。
- **线程。** 首个线程为主线程，拥有评估。其他线程增加关联上下文。来源选择 id；完整键为 `webhook:<id>`。
- **发送者。** `actor` 使用来源原生 id。机器人和 CI 设置 `human: false`。名单中的机器人只能成为推断候选项。CI 等空行为者 id 使用名单外、未直接发出消息的准入单元格。只有消息确实直接向 agent（智能体）发出时才将 `addressed` 设为 `mention` 或 `direct`，这使允许的人的消息具备明确强度。绝不为 agent 自身消息产生触发。
- **回复目的地和元数据。** `channel` 标识回复位置；`url` 链接到来源消息。`channel.thread` 将消息分组，以便有序投递：工单后续消息等待较早消息，直到较早消息失败三次。其他工单不等待。来源专属字段放在 `meta`，例如接收对话及线程根消息。`meta` 和 `text` 均是不可信数据，不是授权。

agent 可以通过 `orchestrator_read trigger/<id>` 和 `thread/<key>` 读取新类型，获得消息文本、去向及相邻触发。新类型不会自动获得内置 skill（技能）、原始服务记录资源或 PR（Pull Request）绑定。

文本搜索无需额外代码。用户使用消息 id、URL 或 `#12` 等来源专属引用时，实现 `find(query)`，返回这些引用标识的线程键和搜索词：

```ts
  find: query => ({ threads: [], terms: [...new Set(query.match(/\bm[0-9]+\b/g) ?? [])] }),
```

这里，查询中的队列消息 id（例如 `m42`）用于搜索对应触发。

### 4. 来源接收什么：scope

```ts
  scope({ meta }, config) {
    const conversation = config.conversations.find(item => item.id === meta.conversation)
    if (!conversation) return { listened: false, reason: 'conversation_not_listed' }
    return { listened: true, observed: conversation.listen === 'all', row: conversation.id }
  },
```

Engine 对每个到达触发通过 `SourceRegistry` 调用 `scope`。范围外消息以已过滤及原因存储，不创建线程。将消息交给 `receive`，不要在轮询循环中静默丢弃，以便面板解释过滤原因。

`observed: true` 表示来源接收该范围中未直接向 agent 发出的消息，允许准入表判断。`row` 标识对应准入行。

重新验证使用授权存储的元数据调用 `scope`。它只能依赖 `meta` 和配置。恢复要求 `listened`，不要求 `observed`。没有 `scope` 时，类型接收自身产生的所有触发，但不观察任何触发。

### 5. 回复：渠道

```ts
  channel: {
    allowed: (address, config) => config.conversations.some(item => item.id === address.conversation),
    send: sendWebhook,
    link: address => ({ url: address.url ?? '',
      label: { zh: `Webhook 对话 ${address.conversation}`, en: `Webhook conversation ${address.conversation}` } }),
  },
```

```ts
export async function sendWebhook(host: Pick<SourceContext, 'commands'>, message: OutboundMessage): Promise<'sent' | 'gone'> {
  const { conversation, message: to } = message.channel.address
  if (!conversation || !to) throw new Error('Missing webhook reply target')
  try {
    await host.commands.run('hookq', ['reply', '--conversation', conversation, '--to', to, '--key', message.id,
      '--text', message.text])
    return 'sent'
  } catch (error) {
    if (error instanceof CliError && error.code === 410) return 'gone'
    throw error
  }
}
```

- **幂等发送。** Outbox 调用 `SourceRegistry.send`，后者调用 `channel.send`。来源插件卸载时消息等待。远程服务已接收后，发送仍可能失败，因此重试必须避免重复。用 `message.id` 作为幂等键：`hookq` 支持 `--key`；GitHub 保存隐藏评论标记；飞书接受幂等键。
- **永久失败与可重试失败。** 成功返回 `sent`，只有永久无法投递时才返回 `gone`，例如本例的 410。gone 消息保留为已丢弃，绝不再次入队，并出现在 `GET sources` 和来源页面。其他失败应抛出异常，让 Outbox 从 5 秒起翻倍重试，最多 300 秒。`CliError.code` 是 CLI 输出报告的代码。
- **允许的目的地。** `allowed(address, config)` 只检查保存的地址和当前配置。所有必需事实放入地址，例如飞书记录是否为私信。不再监听的目的地会丢弃消息；来源禁用时消息等待。没有 `allowed` 时，来源启用期间允许所有地址。
- **链接。** `link(address, links)` 返回面板通过 `SourceRegistry.link` 显示的 URL 和标签。

### 6. 研判的线程上下文

```ts
  threadContext: {
    messageId: trigger => String(trigger.meta.messageId ?? ''),
    // A queue thread is flat: its root, and the message a reply answers, are on its first page, as on GitHub.
    read: async (commands, trigger, _part, cursor) => {
      const page = ThreadPageSchema.parse(JSON.parse(await commands.run('hookq', ['thread', '--conversation',
        String(trigger.meta.conversation), '--root', String(trigger.meta.root), '--json', ...(cursor ? ['--after', cursor] : [])])))
      const messages = page.items.map(item => ({ id: item.id, from: item.sender.id, at: item.at, text: item.text,
        ...(item.sender.bot ? { bot: true as const } : {}), ...(item.url ? { url: item.url } : {}) }))
      return { messages, ...(page.next ? { next: page.next } : {}), body: { messages } }
    },
  },
```

主动研判需要触发周围的对话。`orchestrator_eval` 读取初始窗口；`orchestrator_eval_read` 扩展页面、根消息或父消息。实现可选 `threadContext` 接口以提供读取：

- `read(commands, trigger, part, cursor, config)` 返回 `ContextPart`。其 `messages` 使用 `ContextMessage`：`{id, from, at, text}`，加适用的 `bot`、`deleted`、`reply_to`、`url`、`path` 和 `line` 字段。最后一页省略 `next`。`body` 是显示给研判会话的内容，本例为相同消息。可选 `status` 可以描述飞书 `chat_neighbours` 等特殊结果。`part` 为 `thread`、`root` 或 `parent`。
- `messageId(trigger)` 从中找到触发自身消息。
- `images(text)` 和 `image(…)` 用于消息带图片的类型，例如飞书；本例省略。

没有 `threadContext` 的类型不向研判提供线程上下文；触发自身文本仍可读取。

消息回复较早消息时，添加 `repliesTo(trigger)`，返回对应触发 id，让后续消息通知及 worker 输入在有存储记录时显示被回复消息。例如，`hookq` 提供 `reply_to`，转换时将其保存为 `meta.replyTo`：

```ts
  repliesTo: trigger => typeof trigger.meta.replyTo === 'string' ? key('webhook', trigger.meta.replyTo) : undefined,
```

### 7. 谁可以请求：来源策略部分

```ts
  policy: {
    schema: z.object({ allowed: z.array(z.object({ id: z.string().min(1), name: z.string().trim().max(100).optional() }).strict())
      .default([]) }).strict(),
    fields: [{ key: 'allowed', type: 'rows', label: { zh: '能发起工作的发送者', en: 'Senders who may originate work' }, columns: [
      { key: 'id', type: 'text', label: { zh: '发送者', en: 'Sender' } },
      { key: 'name', type: 'text', label: { zh: '名字', en: 'Name' }, optional: true }] }],
    explicit: { zh: '名单里的人在消息里 @ 智能体。', en: 'Someone on the list mentions the agent in a message.' },
    rows: stored => ConfigSchema.parse(stored).conversations.map(conversation => {
      const all = conversation.listen === 'all'
      return { id: conversation.id, label: conversation.id, note: all ? { zh: '全部消息', en: 'All' } : { zh: '仅 @', en: '@ only' },
        cells: { listedUnaddressed: all, unlistedAddressed: true, unlistedUnaddressed: all } }
    }),
  },
```

- `schema` 验证来源策略，包括 `allowed` 发送者 id。id 比较不区分大小写。准入表单独验证，跨类型共享。策略部分保存在 `policy.sources.webhook`，参与策略版本计算。
- `fields` 在**评估 → 各来源**下生成表单，以 `allowed` 开始。`explicit` 解释如何向 agent 发出消息。可选 `unnamed: {person, bot}` 为飞书 `ou_…` 等不透明 id 提供显示标签；队列 id 是可读名称，因此本例省略。
- `rows(config)` 列出准入范围及可用单元格。每行 id 必须与 `scope` 返回的 `row` 一致。只有来源观察未直接发出消息的范围才提供对应单元格。没有 `rows` 的类型使用默认单元格。
- 可选 `bots` 使用与 `allowed` 相同的条目结构，列出可以成为推断候选项的机器人。没有它时，机器人均不准入。
- 可选 `sync: {label, ready(part), run(host, part, signal)}` 在启动时及点击**立即同步**时刷新成员。飞书使用群成员。失败显示在 `sync:<id>` 下，一分钟后重试，延迟翻倍，最多一小时，直到成功。

### 8. 执行器变量

```ts
  environment: {
    variables: [{ key: 'DSH_SUPERAGENT_WEBHOOK_CONVERSATION', description: {
      zh: '接下工单的 webhook 对话；工单不是从这个来源来的时为空',
      en: 'The webhook conversation that started the ticket; empty when the ticket did not start on this Source' } }],
    values: ({ trigger }) => ({
      DSH_SUPERAGENT_WEBHOOK_CONVERSATION: trigger?.source === 'webhook' ? String(trigger.meta.conversation ?? '') : '' }),
  },
```

即使来源禁用，执行设置也在准备命令旁显示声明的 `variables`。来源启用时，`values` 使用部署、配置及起始触发提供工单专属值。

准备命令读取环境变量；worker 从指令读取值。使用 `DSH_SUPERAGENT_<KIND>_…` 前缀，其他来源的工单返回空值，绝不包含凭据。

### 9. 插件入口及循环

```ts
export const name = 'dsh-superagent-webhook'
export const inject = ['orchestrator']
export function apply(ctx: Context): void {
  ctx.effect(() => ctx.orchestrator.sources.register(webhookKind), 'webhook: source kind')
  ctx.effect(() => {
    const source = sourceContext(ctx.orchestrator, webhookKind)
    const controller = new AbortController()
    // DSH ends its process on an unhandled rejection, so a failure that escapes the loop becomes the Source's health.
    const running = watch(source, controller.signal)
      .catch((error: unknown) => { source.report('webhook', { state: 'error', detail: String(error) }) })
    return async () => { controller.abort(); await running }
  })
}
```

插件注入 `orchestrator`。第一个 effect 注册类型，并在资源释放时注销。第二个使用 `SourceContext` 运行轮询循环。资源释放会中止循环并等待完成，应用停止或重启时也如此。

```ts
async function watch(ctx: SourceContext, signal: AbortSignal): Promise<void> {
  await ctx.ready
  // Setup restarts dsh-superagent once it resolves the accounts; until then the Source reads nothing.
  if (!ctx.configured()) { ctx.report('webhook', health('paused', 'setupPending')); return }
  let failures = 0
  while (!signal.aborted) {
    const config = ctx.config<WebhookConfig>('webhook')
    if (!config) ctx.report('webhook', health('paused', 'sourceOff'))
    else {
      const at = new Date().toISOString()
      try {
        const { read, unreadable } = await pollQueue(ctx, signal)
        failures = 0
        ctx.report('webhook', { state: 'ready', detail: `${read} messages, ${unreadable} unreadable`, at })
      } catch (error) {
        failures += 1
        // One network failure is retried on the next round; three in a row need a person.
        if (!signal.aborted && failures >= 3) {
          ctx.report('webhook', health('error', 'pollFailures', { count: String(failures), error: String(error) }))
        }
      }
    }
    await delay((config?.pollSeconds ?? 30) * 1000, undefined, { signal }).catch(() => {})
  }
}

/** Read the queue after the cursor, take in each message, then move the cursor past them. */
export async function pollQueue(ctx: SourceContext, signal: AbortSignal): Promise<{ read: number; unreadable: number }> {
  const after = ctx.state.get(KV.webhook.cursor) ?? 0
  const output = await ctx.commands.withSignal(signal).run('hookq', ['pull', '--after', String(after), '--json'])
  const page = PageSchema.parse(JSON.parse(output))
  let unreadable = 0
  for (const item of page.items) {
    if (signal.aborted) throw new Error('The webhook Source stopped during the read')
    const message = MessageSchema.safeParse(item)
    if (!message.success) { unreadable++; continue }
    const trigger = webhookTrigger(message.data, page.me)
    if (!trigger) continue
    await ctx.receive(trigger)
    const conversation = { label: { zh: '对话', en: 'Conversation' }, value: message.data.conversation }
    for (const ref of trigger.threads) ctx.threads.describe(threadKey(ref), { facts: { conversation } })
  }
  ctx.state.set(KV.webhook.cursor, page.next)
  return { read: page.items.length, unreadable }
}
```


- **设置和开关。** 等待 `ctx.ready`；`ctx.configured()` 为 false 时不读取任何内容。每轮读取 `ctx.config(kind)`，来源关闭时其值为 undefined。
- **游标。** 使用 `KV`（`src/kv.ts`）中的带类型条目，将读取位置保存到 `ctx.state`，例如 `webhook: {cursor: record<number>('webhook-cursor')}`。`tests/store.spec.ts` 要求键字面量只出现在该注册表中。外部包可以使用导出的 `record` 和 `family` 定义键。设置 `undefined` 移除记录；`ctx.atomic` 将写入组成一个事务。

  只有所有覆盖的触发均已存储后才推进游标。轮次中断时游标不变；稳定 id 使回放安全。时间戳游标使用 `src/sources/poll.ts` 的 `cursorAt(started)`，减去五秒以容纳延迟的远程写入。
- **线程元数据。** 接收触发后调用 `ctx.threads.describe(key, {label, url, facts})` 填充显示信息。`ctx.threads.patch(key, meta)` 委托给 `threads.patchMeta`，设置来源拥有的键，或在值为 `undefined` 时移除。两者均忽略不存在的线程，包括被过滤触发的线程。
- **逐项错误。** 统计格式错误项并继续，不让整批失败。`src/infra/commands.ts` 的 `validItems(item, list, tally)` 支持此模式。独立轮询线程使用 `src/sources/poll.ts` 的 `pollThreads`，将各次失败记录为 `pollError`。其 `polled` 判定保留进行中订阅，并覆盖最后触发与工单结束两者中较晚时间之后的两周。
- **健康。** 只通过 `ctx.report` 写入已声明条目。`health(state, code, values)` 提供 `setupPending`、`sourceOff` 和 `pollFailures` 等固定消息。报告 `error` 前统计连续失败。
- **中止。** 将信号传入 `ctx.commands.withSignal(signal)` 和 `delay`，并在各项之间检查，使停止或重启立即结束循环。

本例不发送来源主动发出的通知。需要通知的来源调用 `ctx.notify`，由 Outbox 像其他回复一样投递。

本例也不将线程绑定到工单完成。只有来源线程生命周期决定工作是否完成时，才使用 `ctx.tickets.bindThread` 和 `threadClosed`，如 GitHub PR。工单何时结束仍由 Engine 决定。

### 10. 注册插件

在 `tsdown.config.ts` 中将模块构建为独立入口：

```ts
entry: { index: 'src/index.ts', 'github-watch': 'src/sources/github.ts', 'feishu-watch': 'src/sources/feishu.ts',
  'webhook-watch': 'src/sources/webhook.ts' },
```

然后在 `package.json` 的 `exports` 中添加 `"./webhook-watch": "./lib/webhook-watch.js"`，导出构建文件；在组合包 `cordis.patch.yml` 的 `insert` 列表添加插件，像飞书一样默认禁用：

```yaml
    - id: dsh-superagent-webhook
      name: dsh-superagent/webhook-watch
      disabled: true
```

在 profile 的 `cordis.patch.yml` 中通过 `id: dsh-superagent-webhook` 和 `disabled: false` 启用。构建并重启，新来源出现在来源设置及**评估 → 各来源**中。

将模块和 spec 添加到 [007 架构](007-architecture.zh.md)的模块及测试表。`tests/docs.spec.ts` 检查两个索引是否完整。

### 11. 测试

按 `tests/sources.spec.ts` 的模式创建 `tests/webhook.spec.ts`：临时目录中的真实 Store、带 `HealthBoard` 的 `SourceRegistry`、使用替身端口的 Engine，以及 `sourceContextOf` 构建的 `SourceContext`。

通过监视 `Commands.run` 来 mock CLI 调用。也监视 `withSignal`，因为它通常返回新的 `Commands` 实例。

```ts
function host(overrides: Partial<SourceContextDeps> = {}) {
  const store = new Store(join(mkdtempSync(join(tmpdir(), 'webhook-')), 'db.sqlite'))
  store.saveSettings(SettingsSchema.parse({
    sources: { webhook: { enabled: true, config: { conversations: [{ id: 'ops', listen: 'mentions' }] } } },
    policy: { sources: { webhook: { allowed: [{ id: 'alice' }] } } },
  }))
  // The registry claims the kind's health entries on the board, where the test reads them.
  const board = new HealthBoard()
  const sources = new SourceRegistry(() => store.settings(), board)
  sources.register(webhookKind)
  const engine = new Engine({ repo: store, scope: sources, policy: () => settingsPolicy(store.settings(), sources),
    assessor: { request: vi.fn() }, executor: { start: vi.fn(), deliver: vi.fn(), stop: vi.fn() },
    outbox: { send: vi.fn(), status: () => undefined, withdraw: vi.fn() } })
  const commands = new Commands(() => testDeployment(), {}, new AbortController().signal)
  const receive = vi.fn((trigger: Trigger) => { const result = engine.receive(trigger); engine.evaluateDue(); return result })
  // The Source's own notices go to the outbox, which keeps their receipts and sends nothing here.
  const deps: SourceContextDeps = { store, engine, outbox: { send: message => { store.receipts.add(message) } }, commands, sources,
    deployment: testDeployment(), ready: Promise.resolve(), configured: () => true, receive, ...overrides }
  return { ...deps, store, engine, commands, receive, health: board.all, ctx: sourceContextOf(deps, webhookKind) }
}
/** The queue as `hookq` answers it. */
function queue(h: ReturnType<typeof host>, answer: () => unknown) {
  vi.spyOn(h.commands, 'withSignal').mockReturnValue(h.commands)
  return vi.spyOn(h.commands, 'run').mockImplementation(async () => JSON.stringify(answer()))
}

it('takes in what the queue holds after the cursor, once, and moves the cursor after it', async () => {
  const h = host()
  queue(h, () => ({ me: 'agent', items: [message('m1'), { id: 'broken' }, message('m2', { conversation: 'elsewhere' })], next: 3 }))
  expect(await pollQueue(h.ctx, new AbortController().signal)).toEqual({ read: 3, unreadable: 1 })
  expect(await h.receive.mock.results[0]!.value).toBe('stored')
  expect(await h.receive.mock.results[1]!.value).toBe('filtered')
  expect(h.store.get(KV.webhook.cursor)).toBe(3)
  // An allowed sender who mentions the agent makes an explicit request.
  expect(h.engine.candidate(key('webhook', 'm1'))?.strength).toBe('explicit')
  await pollQueue(h.ctx, new AbortController().signal)
  expect(await h.receive.mock.results[2]!.value).toBe('duplicate')
  h.store.close()
})
```

`message(id, fields)` 构建队列项，至少覆盖：

- 转换：稳定 id、线程、行为者、`human` 和 `addressed`，以及 agent 自身消息不产生触发；
- 一轮轮询：触发只存储一次，范围外项为 `filtered`，随后推进游标，允许发送者的提及被评估为 `explicit`；
- 中断轮次保留游标；
- `send`：成功为 `sent`，只有 410 返回 `gone`，其他失败抛出异常，CLI 错误由 `commandError` 构建；
- 线程上下文：一页消息及下一页游标，参照 `tests/registry.spec.ts` 通过 `SourceRegistry.threadContext` 读取类型线程；
- 插件：`apply` 注册类型，设置等待时报告 `setupPending`，资源释放回调注销类型，循环外抛的失败成为来源健康状态，不产生未处理的 Promise 拒绝；
- 来源策略部分及准入行，参照 `tests/registry.spec.ts` 和 `tests/policy.spec.ts` 对飞书来源的检查。

### 检查清单

1. 映射触发、线程、行为者、渠道地址及接收范围。
2. 编写类型：`schema`、`fields`、`health`，以及设置检查派生事实时的 `facts`。
3. 用稳定 id 将事件转换为触发，丢弃 agent 自身消息。
4. 只根据 `meta` 在 `scope` 中决定来源接收什么。
5. 提供渠道，`send` 每条消息只发送一次，只有对话永远无法接收时才返回 `gone`。
6. 能读取线程时，为研判提供线程上下文。
7. 提供包含 `allowed` 及准入行的来源策略部分。
8. 注册执行环境变量。
9. 编写插件入口及循环：等待设置、遵循开关、在 `receive` 后推进游标、逐项保留错误、报告健康、中止时停止；在 `KV` 中定义游标。
10. 添加构建入口、导出及 `cordis.patch.yml` 插入项，在 007 表格列出模块及 spec。
11. 像 `tests/sources.spec.ts` 测试其他来源一样测试它。
