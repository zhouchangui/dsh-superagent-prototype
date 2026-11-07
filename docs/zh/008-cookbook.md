# 008 手册

这本手册讲怎么扩展 dsh-superagent。它用 [001 领域模型](001-domain.md) 和 [002 授权](002-authorization.md) 里的术语，以及 [007 架构](007-architecture.md) 里的组件。

## 新增一个来源

这道菜实现一个叫 `webhook` 的来源 kind。它通过一个虚构的 CLI `hookq` 读一个想象中的消息队列，把消息转成触发。其他接口都属 dsh-superagent 已有。

实现有四份职责：翻译事件、定义接收与授权规则、提供回复与上下文、维护一个轮询循环。下面的编号步骤按顺序搭出这些部分。

示例假设这样的 CLI 约定：

- `hookq pull --after <n> --json` 打印位置 `n` 之后的消息，最旧的在前：`{"me": "<the agent's id on the queue>", "items": […], "next": <position>}`。每个条目有 `id`、`conversation`、`thread`（开启它所在线程的那条消息的 id，在那条消息上缺省）、`sender`（含 `id` 和 `bot`）、`mentions_me`、`text`、`at`、`url`。
- `hookq reply --conversation <id> --to <message id> --key <key> --text <text>` 在某条消息下回复，每个 key 一次。会话已被删除时，它在标准错误上以 `{"error": {"code": 410, "message": "…"}}` 失败。
- `hookq thread --conversation <id> --root <message id> --json`（后续分页加 `--after <cursor>`）打印线程的一页，最旧的在前：`{"items": […], "next": "<cursor>"}`，最后一页没有 `next`。

把 kind 及其插件入口放在 `src/sources/webhook.ts`，照 `src/sources/github.ts` 的样子写。来源变大时，像飞书来源那样把翻译、渠道和轮询代码拆成独立模块。

### 1. 把事件映射到领域

写代码之前先定好稳定的映射。触发 id、线程 id 和渠道地址会持久化，之后改动会断开已有记录。

| 概念 | 在这个 webhook 队列里 |
| --- | --- |
| 触发 | 一条消息 |
| 线程 | 一条消息及同一会话里对它的回复：`<conversation>:<第一条消息的 id>` |
| 行动者 | 发送者在队列上的 id，与允许名单的写法一致 |
| 渠道地址 | 回复所属的会话和消息 |
| 接收范围 | 一个会话，在准入表格里有自己的一行 |

### 2. 配置：schema、字段、事实与健康

模块导入下面各步用到的东西：

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

- `kind` 是稳定的来源 id。它出现在触发的 `source`、线程键前缀、`source:webhook` 设置块和来源策略里。
- `schema` 校验已存储的配置并提供默认值。操作者保存某个实例之前，注册表会用这些默认值创建一个启用的实例。让默认值什么都不接收；这个示例默认没有对话。
- `fields` 无需面板自定义代码就能生成来源设置表单。`rows` 字段编辑以其第一列为键的条目；`advanced` 隐藏不常用的设置。来源能把 id 解析成显示名时，实现 `refreshLabels`。把这些名字存在 `KV.sources.labels(kind)` 下。带 `choose` 和 `refresh` 的列随后可以提供它们。
- 可选的 `facts(deployment)` 显示 setup 推导出的值，例如某个 CLI 背后的账号。setup 目前只检查 `SETUP_FIELDS` 里的 GitHub 和飞书字段，所以这个示例不提供事实。
- `health` 声明来源的健康条目，初始为 `starting`。注册会认领每个名字，并拒绝与 `runtime`、`reply`、`worker` 或效应 kind 之类的条目冲突。用 kind 的名字，并把 `sync:<id>` 留给策略同步。出现 `error` 会让状态脚本以 2 退出、每小时检查通知操作者，所以只在可能需要人处理时才报。

### 3. 把消息翻译成触发

转换之前先校验 CLI 的应答：

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

- **稳定的 id。** 用 `key` 从队列 id 推导触发 id。已存储的 id 让 `Engine.receive` 返回 `duplicate`，重复接单只会创建一张工单。因此重读消息是安全的。若要把编辑视作新事件，在 id 里带上编辑版本，像 GitHub 用评论的 `updated_at` 那样。
- **线程。** 第一个线程是主线程，拥有评估。额外的线程补充关联上下文。id 由来源选，它的完整键是 `webhook:<id>`。
- **发送者。** `actor` 里用来源原生的 id。机器人或 CI 设 `human: false`。名单上的机器人只能成为推断候选。行动者 id 为空时（如 CI），走准入表格里"未列入、未 @"那一格。只有消息确实 @ 了智能体时才设 `addressed` 为 `mention` 或 `direct`；这让允许名单上的人的消息具有明确强度。绝不为智能体自己的消息产生触发。
- **回复目的地与元数据。** `channel` 指明回复去处；`url` 链接到来源消息。`channel.thread` 把消息分组以保证按序投递：工单较晚的消息排在它较早的消息之后，直到较早的那条失败三次。其他工单不等待。来源特有的字段放进 `meta`，例如接收的会话和线程根。`meta` 和 `text` 都不可信，不是授权。

智能体可以通过 `orchestrator_read trigger/<id>` 和 `thread/<key>` 读新的 kind。它们暴露消息文本、去向和相邻触发。新 kind 不会自动获得内置技能、读取原始服务记录的资源，或 pull request 绑定。

文本搜索无需额外代码即可工作。当人们使用来源特有的引用（如消息 id、URL 或 `#12`）时，实现 `find(query)`。返回这些引用指向的线程键和搜索词：

```ts
  find: query => ({ threads: [], terms: [...new Set(query.match(/\bm[0-9]+\b/g) ?? [])] }),
```

这里查询里的队列消息 id（例如 `m42`）会按其搜索触发。

### 4. 来源接收什么：scope

```ts
  scope({ meta }, config) {
    const conversation = config.conversations.find(item => item.id === meta.conversation)
    if (!conversation) return { listened: false, reason: 'conversation_not_listed' }
    return { listened: true, observed: conversation.listen === 'all', row: conversation.id }
  },
```

对每个进来的触发，Engine 都通过 `SourceRegistry` 调 `scope`。范围外的消息被存为 filtered 并记录原因，不创建线程。把消息交给 `receive`，而不要在轮询循环里悄悄丢掉，这样面板能解释过滤。

`observed: true` 表示来源接收该范围内未被 @ 的消息，从而允许针对它们的准入表格判定。`row` 指明对应的准入行。

重新校验会带着授权里存的元数据调用 `scope`。让它只依赖 `meta` 和配置。恢复需要 `listened`，不需要 `observed`。没有 `scope` 的 kind 接收它产生的每个触发，但一个都不观察。

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

- **幂等发送。** Outbox 调 `SourceRegistry.send`，它再调用 `channel.send`。来源插件未加载时消息等待。远端服务可能已经收到、发送却仍失败，所以重试不能重复消息。用 `message.id` 作幂等键：`hookq` 支持 `--key`；GitHub 存一个隐藏的评论标记；飞书接受幂等键。
- **永久失败与可重试失败。** 成功返回 `sent`，只有投递永久不可能时才返回 `gone`，例如示例里的 410。gone 的消息按丢弃保留，永不再排队；它出现在 `GET sources` 和来源页面上。其他失败请抛出，好让 Outbox 在 5 秒后重试，翻倍到 300 秒。`CliError.code` 是 CLI 输出报告的代码。
- **允许的目的地。** `allowed(address, config)` 只检查保存的地址和当前配置。把必需的事实都放进地址里；例如飞书记录该会话是否为私聊。不再被监听的目的地会被丢弃，而停用的来源让消息继续等待。没有 `allowed` 时，来源启用期间所有地址都允许。
- **链接。** `link(address, links)` 返回面板通过 `SourceRegistry.link` 显示的 URL 和标签。

### 6. 供研判用的线程上下文

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

主动研判需要触发周围的对话。`orchestrator_eval` 读取初始窗口；`orchestrator_eval_read` 展开分页、根消息或父消息。实现可选的 `threadContext` 接口来提供这些读取：

- `read(commands, trigger, part, cursor, config)` 返回一个 `ContextPart`。它的 `messages` 用 `ContextMessage`：`{id, from, at, text}`，以及适用的 `bot`、`deleted`、`reply_to`、`url`、`path`、`line` 字段。最后一页省略 `next`。`body` 是展示给研判 Session 的内容，这里是同样的消息。可选的 `status` 可以描述特殊结果，例如飞书的 `chat_neighbours`。`part` 是 `thread`、`root` 或 `parent`。
- `messageId(trigger)` 在它们当中找出触发自己的那条消息。
- `images(text)` 和 `image(…)` 面向消息带图片的 kind，例如飞书；本示例不做。

没有 `threadContext` 的 kind 不给研判提供线程上下文；触发自身的文本仍可读。

如果消息会回复更早的消息，加上 `repliesTo(trigger)`。它返回对应的触发 id，让跟进通知和工作节点输入在已存储时显示被回复的消息。例如 `hookq` 提供 `reply_to`、翻译把它存为 `meta.replyTo`：

```ts
  repliesTo: trigger => typeof trigger.meta.replyTo === 'string' ? key('webhook', trigger.meta.replyTo) : undefined,
```

### 7. 谁能请求：来源策略部分

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

- `schema` 校验来源策略，其中包括 `allowed` 的发送者 id。id 比较不区分大小写。准入表格单独校验，跨 kind 共用。这份策略部分保存在 `policy.sources.webhook`，并计入策略版本。
- `fields` 在 **Eval → On each Source**（评估 → 每个来源上）下生成表单，以 `allowed` 开头。`explicit` 说明如何 @ 智能体。可选的 `unnamed: {person, bot}` 为不透明的 id（如飞书的 `ou_…`）提供显示标签；本示例省略它，因为队列 id 本身可读。
- `rows(config)` 列出准入范围及其可用表格单元。每行 id 必须与 `scope` 返回的 `row` 一致。未 @ 消息的单元只在来源观察这些消息时可用。没有 `rows` 的 kind 用默认单元。
- 可选的 `bots` 列出可成为推断候选的机器人，条目形状与 `allowed` 相同。没有它就受理任何机器人。
- 可选的 `sync: {label, ready(part), run(host, part, signal)}` 在启动时和点 **Sync now**（立即同步）时刷新成员关系。飞书用它同步会话成员。失败显示在 `sync:<id>` 下，一分钟后重试，翻倍到一小时，直到成功。

### 8. 供执行器使用的变量

```ts
  environment: {
    variables: [{ key: 'DSH_SUPERAGENT_WEBHOOK_CONVERSATION', description: {
      zh: '接下工单的 webhook 对话；工单不是从这个来源来的时为空',
      en: 'The webhook conversation that started the ticket; empty when the ticket did not start on this Source' } }],
    values: ({ trigger }) => ({
      DSH_SUPERAGENT_WEBHOOK_CONVERSATION: trigger?.source === 'webhook' ? String(trigger.meta.conversation ?? '') : '' }),
  },
```

即使来源停用，执行设置也会在 prepare 命令旁显示声明的 `variables`。启用时，`values` 用部署、配置和来源触发给出工单特有的值。

prepare 命令读环境变量；工作节点在其指令里读到这些值。用 `DSH_SUPERAGENT_<KIND>_…` 前缀，对来自其他来源的工单返回空值，绝不放凭据。

### 9. 插件入口与它的循环

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

插件注入 `orchestrator`。第一个 effect 注册 kind，并在销毁时取消注册。第二个带着 `SourceContext` 跑轮询循环。销毁时中止循环并等待其结束，应用停止或重启时也一样。

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

- **setup 与开关。** 等 `ctx.ready`，在 `ctx.configured()` 为假时什么都不读。每轮都读 `ctx.config(kind)`：来源关闭时它是 undefined。
- **游标。** 把读取位置存在 `ctx.state` 里，用 `KV`（`src/kv.ts`）里的带类型条目，例如 `webhook: {cursor: record<number>('webhook-cursor')}`。`tests/store.spec.ts` 要求键字面量只出现在该注册表里。外部包可以用导出的 `record` 和 `family` 定义键。设为 `undefined` 会删掉一条记录；`ctx.atomic` 把写入归到一个事务里。

  只在覆盖到的每个触发都已存储之后推进游标。被打断的一轮不改变游标；稳定的 id 让重放安全。对时间戳游标，用 `src/sources/poll.ts` 的 `cursorAt(started)`，它减去五秒以容纳远端的延迟写入。
- **线程元数据。** 收到触发之后调用 `ctx.threads.describe(key, {label, url, facts})` 填充它的显示信息。`ctx.threads.patch(key, meta)` 转调 `threads.patchMeta`，设置来源拥有的键，或在值为 `undefined` 时删除它们。两者都忽略不存在的线程，包括被过滤触发的线程。
- **逐条错误。** 统计格式错误的条目并继续，而不是让整批失败。`src/infra/commands.ts` 的 `validItems(item, list, tally)` 支持这种写法。对各自独立轮询的线程，用 `src/sources/poll.ts` 的 `pollThreads`；它把每处失败记为 `pollError`。它的 `polled` 判定保留在线订阅，以及最后一个触发与工单结束中较晚者之后两周的窗口。
- **健康。** 只通过 `ctx.report` 写已声明的条目。`health(state, code, values)` 提供 `setupPending`、`sourceOff`、`pollFailures` 之类的固定文案。报 `error` 之前先数连续失败次数。
- **中止。** 把 signal 传给 `ctx.commands.withSignal(signal)` 和 `delay`，并在条目之间检查，这样停止或重启能立刻结束循环。

本示例不发送来源发起的通知。需要它们的来源调用 `ctx.notify`；Outbox 像其他回复一样投递它们。

本示例也不把线程绑定到工单完成。只有来源的线程生命周期决定工作是否完成时（如 GitHub 的 pull request），才用 `ctx.tickets.bindThread` 和 `threadClosed`。工单何时结束仍由 Engine 决定。

### 10. 注册插件

在 `tsdown.config.ts` 里把这个模块建为自己的入口：

```ts
entry: { index: 'src/index.ts', 'github-watch': 'src/sources/github.ts', 'feishu-watch': 'src/sources/feishu.ts',
  'webhook-watch': 'src/sources/webhook.ts' },
```

然后在 `package.json` 的 `exports` 里加上 `"./webhook-watch": "./lib/webhook-watch.js"` 导出构建产物，并把插件以停用状态加进 bundle 的 `cordis.patch.yml` 的 `insert` 列表，像飞书那样：

```yaml
    - id: dsh-superagent-webhook
      name: dsh-superagent/webhook-watch
      disabled: true
```

在某个 profile 的 `cordis.patch.yml` 里用 `id: dsh-superagent-webhook` 和 `disabled: false` 启用它。构建并重启。新的来源出现在来源设置里，以及 **Eval → On each Source** 下。

把它的模块和 spec 加到 [007 架构](007-architecture.md) 的模块表和测试表里。`tests/docs.spec.ts` 会检查两个索引是否完整。

### 11. 测试

照 `tests/sources.spec.ts` 的写法建 `tests/webhook.spec.ts`：临时目录里一个真实的 Store、一个带 `HealthBoard` 的 `SourceRegistry`、一个带替身端口的 Engine，以及一个由 `sourceContextOf` 得到的 `SourceContext`。

通过监视 `Commands.run` 来模拟 CLI 调用。也监视 `withSignal`，因为它通常返回一个新的 `Commands` 实例。

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

`message(id, fields)` 构造一个队列条目。至少覆盖：

- 翻译：稳定的 id、线程、行动者、`human` 与 `addressed`，以及智能体自己的消息不产生触发；
- 一轮：触发只存一次、范围外的一条为 `filtered`、游标在它们之后推进、允许发送者的 @ 被评估为 `explicit`；
- 被提前中断的一轮保留其游标；
- `send`：`sent`、只在 410 时 `gone`、其他失败抛出，CLI 错误由 `commandError` 构造；
- 线程上下文：一页的消息和下一页的游标，读法与 `tests/registry.spec.ts` 通过 `SourceRegistry.threadContext` 读取 kind 的线程相同；
- 插件：`apply` 注册 kind 并在 setup 等待时报告 `setupPending`，其销毁函数注销该 kind，逃出循环的失败变成来源的健康信息而不是未处理的拒绝；
- 来源的策略部分及其准入行，检查方式与 `tests/registry.spec.ts`、`tests/policy.spec.ts` 检查飞书来源相同。

### 清单

1. 映射触发、线程、行动者、渠道地址和接收范围。
2. 写 kind：`schema`、`fields`、`health`，以及 setup 有推导值时的 `facts`。
3. 把事件翻译成触发，用稳定的 id，并丢掉智能体自己的消息。
4. 在 `scope` 里仅凭 `meta` 决定来源接收什么。
5. 给它一个渠道：`send` 每条消息只投递一次，只有在会话永远无法接收时才回答 `gone`。
6. 如果来源能读线程，就给研判提供线程上下文。
7. 给它一份带 `allowed` 和准入行的策略部分。
8. 注册它的执行环境变量。
9. 写插件入口和循环：等 setup、跟随开关、在 `receive` 之后推进游标、逐条保留错误、上报健康、中止时停止；游标键写在 `KV` 里。
10. 加上构建入口、导出和 `cordis.patch.yml` 的插入项，并在 007 的表格里列出该模块及其 spec。
11. 像 `tests/sources.spec.ts` 测试其他来源那样测试它。
