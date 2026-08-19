# Mastra Durable Agent 深入教程

> **目标读者**：准备面试、需要能在白板上把 Mastra Durable Agent 整条链路画出来并解释设计取舍的人。
>
> **阅读方式**：每节按「问题 → 机制 → 源码 → 面试话术」组织。

## 版本说明（重要，先读）

Durable Agent 是 **`@mastra/core@1.45.0` 起提供的特性，目前标注 beta**（API 可能在不发大版本的情况下破坏性变更）。

**本仓库当前 checkout 停在 `3d3366f`（2026-01-06），早于该特性**，所以你在本地 `packages/core/src/agent/` 下找不到 `durable/` 目录。本文的源码行号基于**上游 `mastra-ai/mastra` 的 `6bc37fd`（2026-08-19）**。

两棵树的关系是**演进而非替代**：

| | 本地 checkout（2026-01） | 上游（2026-08） |
|---|---|---|
| Agent loop 是 workflow | ✅ `packages/core/src/loop/workflows/` | ✅ `packages/core/src/agent/durable/workflows/` |
| suspend / resume / 快照 | ✅ | ✅（同一套 workflow 引擎） |
| 工具审批 human-in-the-loop | ✅ `agent.approveToolCall()` | ✅ `durableAgent.resume()` |
| **可恢复的流**（断线重连） | ❌ | ✅ PubSub + Cache + `observe()` |
| **崩溃恢复** | ❌ | ✅ `recoverActiveRuns()` |
| **公开 API** | 无独立类型 | `DurableAgent` / `createDurableAgent()` |

所以本地这棵树是 Durable Agent 的**执行基座**（第 8、9 章），上游多出来的是**流层和恢复层**（第 5、6、7 章）。面试时两部分都要能讲。

---

## 0. 先建立一句话认知

官方博客里的原话是：

> A long-running agent is simply a durable record that short-lived processes take turns with.
> （长时运行的 agent 无非是一条持久化记录，一批短命进程轮流接手它。）

拆成工程语言就是三句：

> 1. **Durable agent 把 agentic loop 包进 workflow**，于是 loop 的控制流从调用栈搬到了存储里。
> 2. **流式输出走 PubSub 而不是 HTTP 连接**，于是 stream 的生命周期和 HTTP 请求解耦了。
> 3. **PubSub 上挂一层带序号的 cache**，于是断线重连能补齐漏掉的 chunk。

三句话对应官方文档里的「三层」：**Workflow execution / PubSub streaming / Cache layer**。

一个重要的心智提醒：**"durable" 在这里是两个独立的属性，别混**。

- **执行持久（durable execution）**：loop 本身能跨进程续跑 —— 靠 workflow 快照。
- **流持久（resumable stream）**：客户端能断线重连不丢 chunk —— 靠 PubSub + cache。

一个 agent 可以只有前者（老版本），也可以两者都有（`DurableAgent`）。面试里能主动区分这两层，基本就赢一半了。

---

## 1. 问题：为什么普通 Agent 不够

### 1.1 流被绑死在 HTTP 连接上

官方描述得很直接：

> Before Durable Agents, an agent stream was tied to the HTTP connection that started it. But if the connection dropped, there was no way of resuming from the point it disconnected. You had to go back to the beginning of the turn or step.

普通 `agent.stream()` 返回一个 `ReadableStream`，它的生产者是**这次函数调用本身**。连接断了：

- 生产端（loop）可能还在跑，但没人接了 → 白烧 token
- 或者生产端跟着请求一起被 kill → 前面几轮全废
- 换个客户端想接上 → 没有任何寻址方式

移动端弱网、长研究任务、多端同步，三个场景都踩这个坑。

### 1.2 循环寿命长于进程寿命

一个 agentic loop：`LLM → tools → LLM → tools → ...`。朴素实现是一个 `while` 循环跑在一次函数调用里。三种场景直接崩：

1. **Human-in-the-loop**：工具是"删生产库"/"给客户退款"，必须人确认。人 20 分钟后在**另一个 HTTP 请求**里点确认。你不能把调用栈挂 20 分钟。
2. **超时**：serverless 15 分钟就砍。工具要等 3 小时批处理任务。
3. **崩溃**：第 7 轮 OOM。前 6 轮 token 钱花了、邮件发了、款扣了。重跑 = 重复副作用 + 重复付费。

### 1.3 本质

调用栈是**易失的、不可寻址的**；状态机是**可序列化的、可寻址的（runId）**。

Durable execution 的全部内容就是把前者变成后者。而 durable agent 额外做的一步是：**把流也变成可寻址的**（topic = `agent.stream.${runId}`）。

---

## 2. 三个执行变体

官方给了三个工厂函数，产出的都是 durable agent，区别只在 workflow 怎么跑：

| 工厂 | 包 | workflow 执行方式 | 适用 |
|---|---|---|---|
| `createDurableAgent()` | `@mastra/core` | 同进程 `run.start()`，`stream()` 可直接 await | 本地开发、单进程服务 |
| `createEventedAgent()` | `@mastra/core` | `startAsync()` 后台起，不阻塞调用方 | fire-and-forget 后台任务 |
| `createInngestAgent()` | `@mastra/inngest` | Inngest 平台，step 记账 + 重试 + 面板 | 生产 |

```ts
import { Agent } from '@mastra/core/agent'
import { createDurableAgent } from '@mastra/core/agent/durable'

const agent = new Agent({
  id: 'researcher',
  instructions: 'You research topics thoroughly.',
  model: 'openai/gpt-4o',
})

export const durableResearcher = createDurableAgent({ agent })
```

三者都注册进 `Mastra` 的方式和普通 agent 一样。前两个返回 `extends Agent` 的类实例；`createInngestAgent()` 返回一个 **Proxy**，把 `Agent` 的方法转发给内部 agent。

还有一个更省事的写法（推荐）—— 在 `AgentConfig` 上打 `durable` 标记，注册到 `Mastra` 时自动包装：

```ts
const agent = new Agent({ id: 'helper', instructions: '...', model: '...', durable: true })
const mastra = new Mastra({ agents: { helper: agent } })
```

`durable` 也接受 `{ maxSteps, cleanupTimeoutMs }`。通过 API 创建的 stored agent 用同一个字段，服务端 hydrate 时自动包 —— **不需要重新部署代码**。

**面试点**：这是一个很干净的「包装器 + 工厂」组合。`DurableAgent extends Agent`，把 `getModel` / `getTools` / `getMemory` 这些全部 `override` 成转发给被包的 agent（`durable-agent.ts:1131-1420` 有一长串 override）。只有 `stream` / `generate` / `resume` 被真正改写。这样：
- 类型上仍然是 `Agent`，能注册进 `Mastra`、能被 server handler 统一处理
- 语义上是装饰器：能力叠加，不改变被装饰对象

---

## 3. 三层架构

```
调用方
  │  stream(messages, opts)                      DurableAgent.stream()
  │                                              durable-agent.ts:1621
  ├─① prepareForDurableExecution()               preparation.ts
  │    ├─ 把 messages / tools / model / options 序列化成 workflowInput
  │    └─ 把不可序列化的东西（execute 函数、SaveQueueManager、model 实例、
  │       span、stopWhen 闭包）塞进 RunRegistry，用 runId 索引
  │
  ├─② createDurableAgentStream()                 stream-adapter.ts:145
  │    订阅 topic `agent.stream.${runId}`，把 PubSub 事件转成
  │    ReadableStream，喂给 MastraModelOutput
  │
  ├─ await ready   ← 关键：先订阅成功，再启动 workflow
  │
  └─③ executeWorkflow(runId, workflowInput)
        durable-agentic-loop (Workflow)           create-durable-agentic-workflow.ts
          └─ dowhile(durable-agentic-execution)
               ├─ map: map-to-llm-input
               ├─ durable-llm-execution   (Step)  ← 调模型，chunk 发 PubSub
               ├─ map: extract-tool-calls
               ├─ foreach(durable-tool-call)(Step)← 执行工具，可 suspend
               ├─ map: collect-tool-results
               ├─ durable-llm-mapping     (Step)
               ├─ background-task-check   (Step)
               ├─ signal-drain            (Step)
               ├─ map: update-iteration-state
               ├─ durable-is-task-complete(Step)
               └─ durable-goal            (Step)
```

三层各自解决一件事：

1. **Workflow 层** —— 控制流可持久化。每个 step 可被 memoize、replay。
2. **PubSub 层** —— 输出与调用方解耦。chunk 发到以 runId 命名的 topic，谁订阅谁收到。
3. **Cache 层** —— 迟到的订阅者能补历史。每个事件带一个自增 index。

**`await ready` 那行是个容易被追问的细节**（`durable-agent.ts:1766`）：

```ts
const workflowExecution = ready
  .then(async () => {
    await emitChunkEvent(this.pubsub, runId, { type: 'start', ... });
    return await this.executeWorkflow(runId, workflowInput);
  })
  .catch(error => { void this.emitError(runId, error); });
```

必须先等订阅建立再启动 workflow，否则第一批 chunk 发出去时还没人订阅。虽然有 cache 兜底，但**订阅先于生产**是更强的保证，也避免依赖 cache 是否开启。

---

## 4. 序列化边界：把数据挤出 agentic loop

这是整个设计里最核心、也最能体现工程功力的一块。

### 4.1 问题

Workflow 的 step 输入输出必须能过 JSON（Inngest 要跨进程传、快照要落盘）。但 agent loop 里全是不可序列化的东西：

- 工具的 `execute` 函数
- `MastraLanguageModel` 实例
- `SaveQueueManager`（带定时器和队列）
- `AbortController`
- 用户传的 `stopWhen` / `onIterationComplete` 闭包
- observability 的 `Span` 对象
- `MessageList` 实例

### 4.2 答案：分成两条通道

**通道 A：可序列化状态 → workflow input**

`utils/serialize-state.ts` 负责把活对象降解成元数据：

```ts
// serialize-state.ts:25
export function serializeToolMetadata(name: string, tool: CoreTool): SerializableToolMetadata {
  return {
    id: ..., name,
    description: tool.description,
    inputSchema,                      // Zod → JSON Schema
    requireApproval: tool.requireApproval,
    hasSuspendSchema: tool.hasSuspendSchema,
  };
  // 注意：execute 没了
}

// serialize-state.ts:66
export function serializeModelConfig(model: MastraLanguageModel): SerializableModelConfig {
  return {
    provider: model.provider,
    modelId: model.modelId,
    specificationVersion: model.specificationVersion,
    originalConfig: `${model.provider}/${model.modelId}`,   // 供运行时重新解析
  };
}
```

于是 workflow input 长这样（`create-durable-agentic-workflow.ts:52`）：

```ts
const durableAgenticInputSchema = z.object({
  __workflowKind: z.literal('durable-agent'),   // 快照身份标记
  runId: z.string(),
  agentId: z.string(),
  messageListState: z.any(),        // MessageList.serialize()
  toolsMetadata: z.array(z.any()),  // 只有元数据，没有 execute
  modelConfig: modelConfigSchema,   // 只有 id/provider
  modelList: z.array(modelListEntrySchema).optional(),
  options: z.any(),
  state: z.any(),
  messageId: z.string(),
  agentSpanData: z.any().optional(),      // 导出的 span，保证同一个 traceId
  modelSpanData: z.any().optional(),
  requestContextEntries: z.record(z.string(), z.any()).optional(),
});
```

**通道 B：不可序列化状态 → RunRegistry**

`run-registry.ts` 是一个按 runId 索引的进程内注册表：

```ts
export const globalRunRegistry = new TTLCache<string, RunRegistryEntry>({
  max: 1000,
  ttl: 10 * 60 * 1000,
  updateAgeOnGet: true,
  dispose: entry => { entry?.cleanup?.(); },
  noDisposeOnSet: true,
});
```

注意这几个参数是有讲究的：

- `max: 1000` + `ttl: 10min` —— **防止内存无界增长**。durable run 可能永远不被 resume（用户关了页面就不管了），没有 TTL 就是泄漏。
- `updateAgeOnGet` —— 活跃的 run 不会被误驱逐。
- `dispose` 里调 `cleanup()` —— 驱逐时释放订阅、定时器。

step 执行时通过 `RUN_REGISTRY_SYMBOL` 拿到 registry，用 runId 取回真身。

### 4.3 registry 落空怎么办 —— `resolve-runtime.ts`

这是最关键的一环。跨进程恢复时（Inngest 换了 worker，或者进程重启），registry 里**什么都没有**。

`utils/resolve-runtime.ts` 的职责就是：**从 workflow input 的元数据 + Mastra 实例，重建运行时依赖**。

```ts
export interface ResolvedRuntimeDependencies {
  _internal: StreamInternal;
  tools: Record<string, CoreTool>;        // 从 mastra.getAgentById(agentId).getTools() 重建
  model: MastraLanguageModel;             // 从 originalConfig 重新 resolveModelConfig()
  modelList?: RegistryModelListEntry[];
  messageList: MessageList;               // MessageList.deserialize(messageListState)
  memory?: MastraMemory;
  saveQueueManager?: SaveQueueManager;    // 重新 new
  workspace?: Workspace;
  inputProcessors?: ...;
  outputProcessors?: ...;
  processorStates?: ...;
}
```

**这才是"跨进程"能成立的原因**：workflow input 里存的不是对象，是**重建对象所需的最小信息**（agentId + modelId + toolName）。真身永远从 `Mastra` 这个 DI 容器里现取。

**面试话术**：
> "Durable agent 的序列化边界设计是：**能重建的东西不序列化，只序列化重建它所需的标识符**。工具只存 name + JSON Schema + 审批标记，`execute` 从 Mastra 容器按 agentId 现取；模型只存 `provider/modelId` 字符串，用 `resolveModelConfig` 现解析。真正必须跨进程搬运的只有会话状态 —— `MessageList.serialize()` 和迭代累积量。同进程时走 `RunRegistry` 直接拿真身省掉重建开销，跨进程时 registry 落空就走 `resolve-runtime` 兜底。这是一个 fast path / slow path 的经典分层。"

### 4.4 一个防御性细节

`preparation.ts:49` 的 `snapshotRequestContextEntries`：

```ts
for (const [key, value] of requestContext.entries()) {
  if (key === MASTRA_INHERITED_MEMORY_KEY) continue;   // 会 stringify 出巨大的无方法空壳
  const json = boundedStringify(value);                // 有预算上限的序列化
  if (json === undefined) continue;                    // 序列化不了就跳过，不是报错
  out[key] = JSON.parse(json);
}
```

三个点都值得说：
1. **显式排除 memory 实例** —— 它会序列化成一个几 MB 的、方法全丢的壳子。恢复后的 agent 从自己的配置里解析 memory。
2. **`boundedStringify`** —— 共享引用图会让 `JSON.stringify` 指数膨胀，把事件循环卡死。有预算的一次性序列化。
3. **失败即跳过而非抛错** —— 一个不可序列化的 context 项不该让整个 run 挂掉。

---

## 5. 可恢复的流：CachingPubSub

这是 Durable Agent 相对老架构**最大的增量**，也是最值得在面试里深挖的一块。

### 5.1 事件与 topic

`constants.ts` 定义了两个方向的 topic：

```ts
export const AGENT_STREAM_TOPIC  = (runId: string) => `agent.stream.${runId}`;   // worker → consumer
export const AGENT_CONTROL_TOPIC = (runId: string) => `agent.control.${runId}`;  // caller → worker
```

注释解释了为什么要分开：

> Separate from `AGENT_STREAM_TOPIC` because control messages travel the opposite direction... Keeping them apart means stream consumers never see control traffic and vice versa.

control topic 目前只跑一种消息：`ABORT_REQUEST`。**为什么需要它**：`abort()` 可能在 A 进程调用，而 run 实际在 B 进程执行。本地的 `AbortController` 只能停到本进程的 step，所以还要通过 pubsub 广播一次（`durable-agent.ts` 的 `requestRemoteAbort`）。

stream topic 上的事件类型（`AgentStreamEventTypes`）：`CHUNK` / `STEP_START` / `STEP_FINISH` / `FINISH` / `ERROR` / `SUSPENDED` / `ABORT` / `ITERATION_COMPLETE`。

### 5.2 写入端：给每个事件编号

`events/caching-pubsub.ts` 的 `publish`：

```ts
async publish(topic, event, options?) {
  if (options?.localOnly || this.shouldCache?.(topic) === false) {
    // 不进 cache，直接透传
    await this.inner.publish(topic, { ...event, id: crypto.randomUUID(), createdAt: new Date() }, options);
    return;
  }

  let index: number | undefined;
  let indexFailed = false;
  try {
    index = (await this.cache.increment(counterKey)) - 1;   // 原子自增，0-based
  } catch (error) {
    this.logError(...);
    indexFailed = true;
  }

  const fullEvent: Event = { ...event, id: crypto.randomUUID(), createdAt: new Date(),
                             ...(index !== undefined ? { index } : {}) };

  if (!indexFailed) {
    try {
      await this.cache.listPush(cacheKey, fullEvent);   // ← 先入 cache
    } catch (error) { this.logError(...); }
  }

  await this.inner.publish(topic, fullEvent, options);    // ← 再 live 发布
}
```

四个可以被追问的决策：

**① 为什么先写 cache 再 live publish？**
注释写了：`Cache BEFORE live publish so late-joining observers never miss events`。如果反过来，一个刚好在两步之间订阅的 observer 既收不到 live（还没订上）也读不到 cache（还没写入）—— 事件永久丢失。

**② counter 失败为什么不 fallback 到 0？**

```ts
// On counter failure leave `index` undefined rather than defaulting to 0:
// downstream consumers that key off `index` would otherwise see colliding indices.
```
默认 0 会造成 index 碰撞，replay-from-offset 直接错乱。宁可没有 index（这个事件不进 cache、只 live 送达），也不要错的 index。

**③ cache 失败为什么不阻塞 live？**
`Always publish to inner PubSub — cache failure must not block live delivery`。降级策略：cache 挂了，实时流照常，只是失去可恢复性。

**④ 为什么 `localOnly` 事件不缓存？**
注释说得很具体：`workflow.events.v2.*` 的 watch 事件携带累积的 step results，**单条可能好几 MB**。它们不跨实例中继，缓存下来没有读者，只会把 cache 撑爆。

### 5.3 读取端：四阶段 `subscribeFromOffset`

这是全篇最值得背下来的一段。**这是"订阅 + 读历史"这个经典竞态问题的标准解法**。

```ts
async subscribeFromOffset(topic: string, offset: number, cb: EventCallback): Promise<void> {
  // ── Phase 1: 先订阅 live，bootstrap 期间全部进 buffer ──
  let bootstrapping = true;
  const buffer = [];
  let lastDelivered = -1;

  const wrappedCb: EventCallback = (event, ack, nack) => {
    if (typeof event.index === 'number' && event.index < offset) {
      return ack?.();          // 早于请求 offset 的直接 ack 掉，别让它滞留在后端
    }
    if (bootstrapping) { buffer.push({ event, ack, nack }); return; }

    // 稳态去重：index 水位线
    const isRetry = typeof event.deliveryAttempt === 'number' && event.deliveryAttempt > 1;
    if (typeof event.index === 'number' && event.index <= lastDelivered && !isRetry) {
      return ack?.();
    }
    if (typeof event.index === 'number' && event.index > lastDelivered) lastDelivered = event.index;
    return cb(event, ack, nack);
  };
  await this.inner.subscribe(topic, wrappedCb);

  try {
    // ── Phase 2: 拉历史，按序投递 ──
    const seen = new Set<string>();
    const history = await this.getHistory(topic, offset);
    for (const event of history) {
      seen.add(this.dedupKey(event));
      if (typeof event.index === 'number') lastDelivered = event.index;
      await cb(event);         // await 保证顺序
    }

    // ── Phase 3: 排空 buffer，跳过历史已覆盖的 ──
    for (const { event, ack, nack } of buffer) {
      const key = this.dedupKey(event);
      if (seen.has(key)) continue;
      seen.add(key);
      if (typeof event.index === 'number') lastDelivered = event.index;
      try { await cb(event, ack, nack); } catch { await nack?.(); }
    }

    // ── Phase 4: 切换到透传 ──
    bootstrapping = false;
    buffer.length = 0;
  } catch (error) {
    // 回滚：别让 wrappedCb 永远卡在 bootstrap 模式
    this.callbackMap.delete(cb);
    await this.inner.unsubscribe(topic, wrappedCb).catch(() => {});
    throw error;
  }
}
```

**为什么必须是这个顺序**：

- 先读历史再订阅 → 两者之间产生的事件**丢失**（gap）
- 先订阅再读历史 → 两者之间产生的事件**重复**（duplicate）

重复是可以修的（去重），丢失是不可修的。所以选**先订阅、缓冲、去重**。这是分布式系统里的通用取舍：**宁可 at-least-once 再去重，也不要 at-most-once**。

**去重键为什么不用 `event.id`** —— 这个注释太值得引用了：

```
We cannot dedup on `event.id`: `CachingPubSub.publish` assigns the id and caches
the event with it, but inner PubSub implementations regenerate `id` inside their
own `publish`, so the cached copy and the live copy of the SAME publish carry
different ids. The sequential `index` is assigned here and is preserved by every
inner implementation, so it matches across both paths.
```

同一次 publish 的「cache 副本」和「live 副本」`id` 不一样（内层实现会重新生成），但 `index` 是这一层分配的、内层会原样保留。所以 `dedupKey` 是：

```ts
private dedupKey(event: Event): string {
  return event.index !== undefined ? `i:${event.index}` : `id:${event.id}`;
}
```

**`isRetry` 那个例外**也要能解释：nack 重投的消息 index 相同但 `deliveryAttempt > 1`，必须放行，否则消费者永远看不到重试。

**面试话术**：
> "`subscribeFromOffset` 是四阶段：先订阅并缓冲、再拉历史按序投递、然后排空缓冲并去重、最后切透传并用 index 水位线做稳态去重。核心决策是**订阅先于读历史**——中间窗口宁可重复也不能丢，重复可以去、丢了没法补。去重键用的是 CachingPubSub 自己分配的自增 index 而不是 event.id，因为内层 pubsub 会重新生成 id，导致同一次 publish 的缓存副本和实时副本 id 不一致。"

### 5.4 从事件到 `MastraModelOutput`

`stream-adapter.ts:145` 的 `createDurableAgentStream()` 把 PubSub 事件转成 `ReadableStream`，再包成 `MastraModelOutput` —— **和普通 `agent.stream()` 返回的是同一个类型**。

这是 API 兼容性的关键：调用方拿到的 `output.textStream` / `output.fullStream` / `await output.text` 用法完全一致，感知不到底下是 pubsub 还是直连。

它还接受两个防悬挂参数：

```ts
idleTimeoutMs?: number;
isAlive?: () => boolean | Promise<boolean>;
```

注释解释得很清楚：

> A durable run whose driving process crashed stops emitting but never publishes a terminal event, so `observe()` would otherwise hang forever on a producerless topic.

驱动进程崩了，topic 上再也不会有事件，也不会有终止事件 —— `observe()` 会**永远挂着**。所以：空闲超时触发时查一次 `isAlive()`（心跳探针），活着就继续等，死了就结束流。

而且 `isAlive` 抛异常按"活着"处理 —— 依赖抖动不该杀掉一个正常的流。**这种"故障时倾向保守"的默认值选择，是面试里的加分细节。**

---

## 6. `observe()`：另一个客户端接上来

```ts
const { output, cleanup } = await durableResearcher.observe(runId)
for await (const chunk of output.fullStream) { /* 包含断线期间漏掉的 */ }
cleanup()
```

实现（`durable-agent.ts:3031`）本质就是「不启动 workflow 的 stream()」：

```ts
const stream = createDurableAgentStream<TOutput>({
  pubsub: this.pubsub,
  runId,
  messageId: crypto.randomUUID(),
  model: { modelId: undefined, provider: undefined, version: 'v3' },  // observe 不知道模型
  offset: options?.offset,            // ← 从这里续
  idleTimeoutMs: options?.idleTimeoutMs,
  isAlive: options?.isAlive,
  messageList: globalRunRegistry.get(runId)?.messageList ?? this.#runRegistry.getMessageList(runId),
  ...
});
await ready;
```

不传 `offset` 就是从 0 全量 replay（整个对话重放一遍）；传了就从那个位置续。客户端记住自己收到的最后一个 index，重连时传 `offset = last + 1`，就是精确续传。

**`observe()` 不拥有 run 的生命周期**，但返回的 `abort()` 仍然能停掉它 —— 先翻本进程的 controller（如果有），再 `requestRemoteAbort(runId)` 走 control topic。注释点明了：

> the common case for observe(), which exists precisely to watch runs this process did not start.

### 6.1 `resume()` 里的 offset 妙用

`durable-agent.ts:2023`：

```ts
// Skip events already broadcast by the original run (e.g. the SUSPENDED
// chunk that paused it). Without this, a resume that closes on suspend
// (resumeGenerate) would immediately close on the replayed SUSPENDED.
const resumeOffset = await this.#getPubsubOffset(runId);
```

`#getPubsubOffset` 就是读 `getHistory(topic).length`。

**这个 bug 很典型，值得记住**：`resumeGenerate()` 设了 `closeOnSuspend: true`（否则 `getFullOutput()` 会永远挂着）。如果 resume 时从 offset 0 订阅，第一个重放到的就是把 run 暂停掉的那条 `SUSPENDED` 事件 —— 流立刻关闭，resume 等于没做。所以必须从「已发布事件数」这个水位线开始订阅。

---

## 7. 工具审批（Human-in-the-loop）

### 7.1 API

```ts
const { output, runId, cleanup } = await durableAgent.stream('删掉旧记录', {
  requireToolApproval: true,
  onSuspended: ({ toolCallId, toolName, args }) => notifyUser(...),
})

// 人点了确认之后，可以是另一个进程、另一个请求：
await durableAgent.resume(runId, { approved: true })
```

三种触发暂停的方式（和老架构一致）：

| | 谁决定 | 时机 | resume 数据 | 语义 |
|---|---|---|---|---|
| `requireToolApproval: true`（请求级） | 框架 | `execute` **之前** | `{ approved: boolean }` | 授权闸门 |
| `requireApproval: true`（工具级） | 框架 | `execute` **之前** | `{ approved: boolean }` | 授权闸门 |
| `needsApprovalFn(args)` | 框架 | `execute` **之前**，按参数判断 | `{ approved: boolean }` | 动态授权 |
| 工具内调 `suspend()` | 工具自己 | `execute` **内部** | 工具自定义 `resumeSchema` | 缺信息 / 等外部事件 |

`durable/workflows/steps/tool-call.ts` 的注释总结了整个流程：

```
2. Checks if approval is required (global or per-tool)
3. If approval required, emits suspended event, persists messages, and suspends
   - Tool approval: step suspends with approval payload
   - In-execution suspension: tool calls suspend() callback, step suspends with suspension payload
```

和老架构的唯一区别：老版本 `controller.enqueue({ type: 'tool-call-approval' })`，durable 版本 `emitChunkEvent(pubsub, runId, ...)`。**接口一样，传输层换了。**

### 7.2 suspend 的四步（durable 版本）

1. **发事件**：`tool-call-approval` / `tool-call-suspended` chunk 进 pubsub，前端立刻能渲染审批 UI
2. **写元数据**：把 `pendingToolApprovals` / `suspendedTools` 挂到最后一条 assistant 消息的 `content.metadata`，**并且带上 `runId`**
3. **强制 flush**：消息同步落盘（正常路径是 `SaveQueueManager` 异步批量的）
4. **真正挂起**：`suspend(payload, { resumeLabel: toolCallId })`

第 2 步的 `runId` 是 memory 层和 workflow 层的桥：**用户刷新页面后前端只有 threadId，从消息历史里能读回 runId**。

第 3 步的必要性：suspend 之后当前进程可能永远不再执行，异步队列里的消息就丢了。`tool-call.ts:378` 有一条注释直接点名了这个 bug：

> `addToolMetadata()` is never persisted — a reloading client then sees no pending approval

第 4 步的 `resumeLabel` 见下节。

### 7.3 `resumeLabel` —— 精确寻址挂起点

`workflows/handlers/step.ts:385`：

```ts
if (suspendOptions?.resumeLabel) {
  for (const label of resumeLabel) {
    const labelData = { stepId: step.id, foreachIndex: executionContext.foreachIndex };
    contextMutations.resumeLabels[label] = labelData;
    executionContext.resumeLabels[label] = labelData;
  }
}
```

快照里因此有一张表：`toolCallId → { stepId: 'durable-tool-call', foreachIndex: 2 }`。

**为什么需要**：`foreach(toolCallStep)` 里所有 item 的 `step.id` 都一样。光靠 stepId 定位不到"是第几个工具调用挂起了"。`resumeLabel = toolCallId` 给了一个业务层天然唯一的地址 —— 调用方只需要说 "resume 这个 toolCallId"，不用理解 workflow 的执行路径。

### 7.4 拒绝不是异常

```ts
if (!resumeData.approved) {
  return { result: 'Tool call was not approved by the user', ...inputData };
}
```

**拒绝返回一个字符串作为 tool result 塞回 LLM**，不抛异常。这样 LLM 能看到"用户拒绝了"并自然组织回复，而不是整个 run 失败。

配套的还有一处：纯审批的 `{ approved: true }` **不会**透传给工具 —— 工具不需要知道它被审批过，保持实现干净。

---

## 8. 执行基座：Workflow 引擎（这部分本地 checkout 就有）

上面几章讲的是 durable agent 特有的层。往下一层，是它依赖的通用 workflow 引擎。**这一层在本地 checkout 里就完整存在**，也是最容易被深挖的部分。

### 8.1 快照数据结构

`packages/core/src/workflows/types.ts:383`：

```ts
export interface WorkflowRunState {
  runId: string;
  status: WorkflowRunStatus;
  result?: Record<string, any>;
  error?: SerializedError;
  requestContext?: Record<string, any>;
  value: Record<string, string>;                    // workflow state
  context: { input?: ... } & Record<string, SerializedStepResult<...>>;  // 每个 step 的结果
  serializedStepGraph: SerializedStepFlowEntry[];   // 执行图本身
  activePaths: Array<number>;
  activeStepsPath: Record<string, number[]>;
  suspendedPaths: Record<string, number[]>;         // stepId → 执行路径坐标
  resumeLabels: Record<string, { stepId: string; foreachIndex?: number }>;
  waitingPaths: Record<string, number[]>;
  timestamp: number;
  tripwire?: StepTripwireInfo;
}
```

三个字段是恢复的核心：

- **`context`** —— 已完成 step 的结果。恢复时当 memoization cache 用，不重跑。
- **`suspendedPaths`** —— stepId → 执行图坐标，用来算恢复入口。
- **`resumeLabels`** —— 业务地址（toolCallId）→ 内部坐标。

### 8.2 三种"跳过重跑"的机制

**这是最容易被追问的地方**。Mastra 的 replay 不是 event-sourcing 式从头重放，而是**基于 stepResults 的 memoization + 执行路径快进**，三种粒度：

#### 机制 A：顶层快进 —— `resumePath`

`workflows/default.ts:819`：

```ts
let startIdx = 0;
if (timeTravel)      { startIdx = timeTravel.executionPath[0]!; timeTravel.executionPath.shift(); }
else if (restart)    { startIdx = restart.activePaths[0]!;      restart.activePaths.shift(); }
else if (resume?.resumePath) { startIdx = resume.resumePath[0]!; resume.resumePath.shift(); }

const stepResults = timeTravel?.stepResults || restart?.stepResults || resume?.stepResults || { input };

for (let i = startIdx; i < steps.length; i++) { ... }
```

主循环直接从 `startIdx` 开始，前面的顶层 entry 一个都不碰。`shift()` 是在逐层下钻 —— 每进一层嵌套（parallel / 嵌套 workflow）就消耗掉路径的一段。

#### 机制 B：循环续接 —— `iterationCount`

`workflows/handlers/control-flow.ts:679`：

```ts
const prevIterationCount = stepResults[stepId]?.metadata?.iterationCount;
let iteration = prevIterationCount ? prevIterationCount - 1 : 0;
```

`dowhile` 恢复时不从第 0 轮重来，从快照里的轮次续上。

同一个函数里还有一处**幂等防护**：

```ts
// Clear resume for next iteration only if the step has completed resuming
if (currentResume && result.status !== 'suspended') {
  currentResume = undefined;
}
```

**resume payload 只消费一次**。否则 `{ approved: true }` 会在后续每一轮里被重复当成 resume 数据注入。

#### 机制 C：foreach 逐项 memoization —— 工具不重复执行的真正保证

`workflows/handlers/control-flow.ts:992, 1168`：

```ts
const prevForeachOutput = prevPayload?.suspendPayload?.__workflow_meta?.foreachOutput || [];

// 遍历每个 item：
const prevItemResult = prevForeachOutput[k];
if (prevItemResult?.status === 'success' ||
    (prevItemResult?.status === 'suspended' && resume?.forEachIndex !== k && resume?.forEachIndex !== undefined)) {
  return prevItemResult;      // ← 直接返回缓存，不执行
}

let resumeToUse = undefined;
if (resume?.forEachIndex !== undefined) {
  resumeToUse = resume.forEachIndex === k ? resume : undefined;   // 只给目标 item 注入
}
```

场景：LLM 一轮返回 3 个工具调用，第 2 个需要审批。
- item 0、1 已成功 → `prevForeachOutput` 有结果 → **直接返回，不重新调用工具**
- item 2 → `resume.forEachIndex === 2` → 注入 `{ approved: true }` 继续执行

`prevForeachOutput` 存在 suspend payload 的 `__workflow_meta` 里，跟着快照一起走。

### 8.3 并发度为什么会被强制降到 1

`create-durable-agentic-workflow.ts:216`：

```ts
.foreach(toolCallStep, {
  concurrency: ({ inputData, getInitData }) => {
    const state = getInitData() as IterationState | undefined;
    return resolveDurableToolCallConcurrency({
      options: state?.options,
      toolsMetadata: state?.toolsMetadata,
      toolCalls: inputData as DurableToolCallInput[],
    });
  },
})
```

`resolveDurableToolCallConcurrency`：只要有任何工具带 `requireApproval` 或 `hasSuspendSchema`，或请求级开了 `requireToolApproval`，这一轮就 **concurrency = 1**；否则用 run 的 `toolCallConcurrency`（默认 10）。

**为什么**：`foreach` 的 suspend 语义是"记录第 k 个 item 挂起了，恢复时只重跑第 k 个"。并发跑 10 个工具、其中 3 个同时 suspend，恢复语义就变成多点恢复 —— `Run.resume()` 会明确抛 `Multiple suspended steps found`。降到 1 保证任意时刻最多一个挂起点。

这是典型的**用并发度换语义简单性**。

注意注释里还强调了一句：

> The workflow graph is shared across runs, so this must be a resolver — never a mutated shared options object.

workflow 对象是启动时创建、所有 run 共用的。并发度必须是**每次求值的函数**，不能是被改写的共享配置 —— 否则会串 run。**这是个很好的"共享可变状态"陷阱例子。**

### 8.4 快照持久化策略（和老架构的重要差异）

普通 workflow 默认 `shouldPersistSnapshot: () => true`。老的 agentic-loop 收紧成只在 `suspended` 时写。

**Durable agent 又放宽了**（`create-durable-agentic-workflow.ts:139`）：

```ts
shouldPersistSnapshot: params => {
  // 需要持久化记录来同时支持：
  //  - suspend 之后的 resumeStream()（pending / paused / suspended）
  //  - 进程重启后对孤儿 RUNNING run 的启动时恢复，via recoverActiveRuns()
  //    —— 这要求 loop 在飞行途中就把行标记成 running（issue #19056）
  return params.workflowStatus === 'pending'
      || params.workflowStatus === 'paused'
      || params.workflowStatus === 'suspended'
      || params.workflowStatus === 'running';
},
pruneSnapshot: pruneAgentLoopSnapshot,
```

**为什么要放宽**：崩溃恢复需要能查到"哪些 run 还在 running"。只在 suspended 时写快照的话，进程崩了根本查不到它。这是 durable agent 相对老架构的功能性代价。

**代价怎么补 —— `pruneSnapshot`**：`loop/workflows/prune-snapshot.ts` 的注释把问题说得非常清楚：

> Without pruning, every persisted snapshot re-serializes the conversation several times over (step payload/prevOutput message arrays, AI SDK `output.steps` request/response history, and a stale `__streamState` retained on completed steps after each resume), so snapshot size scales with **thread length × number of historical suspensions**.

裁剪规则：
- **终态 step**（success/failed/skipped/bailed/canceled）永远不会再被 resume → 丢掉 `suspendPayload` / `suspendOutput` / `resumePayload`，剥掉 payload/output 里的重型字段
- **非终态 step**（suspended/waiting/paused/running）保留 `suspendPayload` **完整** —— 那是 resume 状态（`__streamState`、审批信息、嵌套 run id）；但 payload 仍然剥掉重型字段，因为 resume 会从 `__streamState.messageList` 重建消息
- **foreach 聚合项**同样逐项处理，仍挂起的并行工具调用保留各自 resume 状态
- **引擎路由状态**（`suspendedPaths` / `waitingPaths` / `activePaths` / `resumeLabels` / `serializedStepGraph`）**一律不动**

并且注释明确限定：

> This must only be registered on the internal agent workflows. User-authored workflows keep full suspend/resume history in their run record.

**面试话术**：
> "快照策略在 durable agent 上是个明显的取舍点。老的 agentic-loop 只在 suspended 时写快照，省 IO，但代价是进程崩了查不到孤儿 run。Durable agent 为了支持崩溃恢复，把 running 也纳入持久化，然后用 `pruneSnapshot` 补性能：终态 step 丢掉 resume 状态、剥离消息副本，非终态保留完整 suspendPayload，引擎路由状态一律不动。本质是**把"存全量"换成"只存 resume 真正会读的字段"**。"

### 8.5 `__streamState`：流式 Agent 特有的问题

普通 durable workflow 只需要恢复控制流状态。流式 Agent 多一层：**用户可见的增量输出是有状态的累加器**。

假设 LLM 已经吐了 "我来帮你查天气，"，然后发起工具调用并挂起。恢复时创建的是一个**全新的 `MastraModelOutput`**，`bufferedText` 是空的 —— `await output.text` 只会拿到恢复后的那半截。

解法：`MastraModelOutput.serializeState()` 把整个累加器打包进 suspend payload 的 `__streamState` 键：

```ts
serializeState() {
  return {
    status, bufferedSteps, bufferedReasoningDetails, bufferedByStep,
    bufferedText, bufferedTextChunks, bufferedSources, bufferedReasoning,
    bufferedFiles, toolCallArgsDeltas, toolCallDeltaIdNameMap,
    toolCalls, toolResults, warnings, finishReason, request, usageCount, tripwire,
    messageList: this.messageList.serialize(),
  };
}
```

恢复时作为 `initialState` 注入新的 output 对象。`__streamState.messageList.memoryInfo` 还被复用来恢复 `threadId` / `resourceId` —— 所以 `resume(runId, data)` 甚至不需要重新传 threadId。

### 8.6 可插拔执行引擎

`workflows/execution-engine.ts` 定义抽象类，`DefaultExecutionEngine` 是内存实现，**同时把所有需要持久化语义的地方开成 hook**：

| Hook | Default | 语义 |
|---|---|---|
| `wrapDurableOperation(id, fn)` | 直接 `fn()` | 把操作变成可 memoize 的原子单元 |
| `executeSleepDuration(ms, ...)` | `setTimeout` | 睡眠 |
| `executeStepWithRetry(...)` | 循环重试 | 重试策略 |
| `evaluateCondition(...)` | 包一层 `wrapDurableOperation` | 条件求值也要 memoize |
| `onStepExecutionStart(...)` | 发事件 | 事件发布也要 memoize |
| `getEngineContext()` | `{}` | 给 step 注入引擎原语 |
| `requiresDurableContextSerialization()` | `false` | context 是否需要序列化 |
| `isNestedWorkflowStep(step)` | `instanceof Workflow` | 嵌套 workflow 识别 |

核心是 `wrapDurableOperation`（`default.ts:190`）：

```ts
async wrapDurableOperation<T>(_operationId: string, operationFn: () => Promise<T>): Promise<T> {
  return operationFn();      // Default：恒等函数，零开销
}
```

Inngest 覆盖（`workflows/inngest/src/execution-engine.ts:211`）：

```ts
async wrapDurableOperation<T>(operationId: string, operationFn: () => Promise<T>): Promise<T> {
  return this.inngestStep.run(operationId, async () => {
    try { return await operationFn(); } catch (e) { throw e; }
  }) as Promise<T>;
}
```

一行之差，语义完全不同：`step.run(id, fn)` 会**记账**。函数被重新调用（replay）时，同一个 `id` 直接返回上次的结果。

### 8.7 `requiresDurableContextSerialization` —— 高分题

`execution-engine.ts` 注释：

> Inngest requires requestContext serialization for memoization. When steps are replayed, the original function doesn't re-execute, so requestContext modifications must be captured and restored.

**问题**：Default 引擎下 `requestContext` 是共享对象引用。step A 里 `set('x', 1)`，step B 里 `get('x')` 拿得到。

但 Inngest 下 step A 被 memoize 了，**函数体根本不会再执行**，那次 `set` 也就不会发生。replay 到 step B 时 `x` 就丢了。

**解法**：让 step 的返回值携带 context 变更。因为返回值被 memoize，变更也就被 memoize 了：

```ts
return {
  result: stepResult,
  stepResults: {...},
  mutableContext: engine.buildMutableContext(executionContext),  // state / suspendedPaths / resumeLabels
  requestContext: engine.serializeRequestContext(requestContext),
};
```

`suspend()` 里同样写两份：

```ts
contextMutations.suspendedPaths[step.id] = executionContext.executionPath;  // 给 Inngest replay
executionContext.suspendedPaths[step.id] = executionContext.executionPath;  // 给 Default 直接用
```

**面试话术**：
> "这是 durable execution 最经典的坑：**副作用必须在 memoize 边界内，或者变成返回值的一部分**。Mastra 把跨 step 的可变状态（suspendedPaths / resumeLabels / requestContext）显式抽成 MutableContext 和序列化的 requestContext，随 step 结果一起返回。Default 引擎共享内存不需要这层，所以用 `requiresDurableContextSerialization()` 做开关，避免为不需要的场景付序列化成本。"

### 8.8 生命周期回调恰好一次

Inngest 引擎把 `invokeLifecycleCallbacks` 覆盖成 **no-op**，改在一个 `step.run('...finalize')` 里调 `invokeLifecycleCallbacksInternal`：

```ts
await step.run(`workflow.${this.id}.finalize`, async () => {
  if (result.status !== 'paused') await engine.invokeLifecycleCallbacksInternal(result);
  if (result.status === 'failed') throw new NonRetriableError(`Workflow failed`, { cause: result });
  return result;
});
```

因为 `step.run` 是 memoized 的，replay 时不重复执行。**这就是 `onFinish` / `onError` "恰好一次"的实现方式。**

另外 function 级 `retries: 0` —— **重试在 step 级做**，这样重试不会把已完成的 step 也拖进重跑语义。

---

## 9. 崩溃恢复

### 9.1 问题

进程崩了，run 在 storage 里还是 `running` 状态，**没有任何自动重试**。

### 9.2 自动恢复

```ts
export const mastra = new Mastra({
  agents: { myAgent: durableAgent },
  storage: new PostgresStore({ connectionString: process.env.DATABASE_URL! }),
  recovery: { durableAgents: 'auto' },
})
```

deployer 在启动时调 `recoverAllDurableAgents()`，紧接在重启活跃 workflow run 之后。它扫描所有注册的 durable agent，找出卡在 `running` 的 run，从最后一个持久化快照重新驱动。

这正是 §8.4 里 `shouldPersistSnapshot` 必须包含 `'running'` 的原因 —— **没有 running 快照就发现不了孤儿 run**。

### 9.3 手动恢复

```ts
const result = await mastra.recoverAllDurableAgents()
const agentResult = await durableAgent.recoverActiveRuns()
await durableAgent.recoverActiveRuns({ runId: 'run-abc-123' })
```

配套的还有 `listActiveRuns()`（`durable-agent.ts:2816`），支持按 thread / resource / 时间范围过滤分页。

### 9.4 两个必须主动说出来的限制

**① 幂等性要求**（官方文档的 warning）：

> Recovery re-runs the agentic loop from the last snapshot, which **re-issues LLM calls (real cost) and re-executes tool calls**. Make sure your tools are idempotent before enabling automatic recovery.

恢复是从**最后一个快照**重跑，不是精确到崩溃那一刻。已经执行但还没落进快照的工具调用会**再跑一次**。开自动恢复的前提是工具幂等。

**② 多副本竞争**：

> Mastra doesn't provide a distributed lease or lock yet. In multi-replica deployments, every replica that starts with `recovery.durableAgents: 'auto'` will race to recover the same runs.

没有分布式锁，多副本会抢同一批 run。要么自己做 leader election，要么只在单个副本上跑。

不过代码里能看到正在往这个方向走 —— `durable-agent.ts:559` 有 `#acquireRecoveryLease()`、`#raceRecoveryLease()`、`#createRecoveryFencedPubSub()`，`CachingPubSub.getLeaseProvider()` 会把内层（比如 Redis）的租约能力透出来：

```ts
// Leasing is a capability of the underlying backend (e.g. Redis), not of the
// caching decorator itself — so rather than unconditionally declaring lease
// methods (which would make isLeaseProvider report true even when the inner
// can't coordinate a lock), we surface the inner's capability directly.
```

**这段注释本身就是很好的接口设计范例**：装饰器不该假装拥有它没有的能力。

**面试话术**：
> "崩溃恢复是 durable agent 里最需要提前跟业务方对齐的部分。恢复是 at-least-once 而不是 exactly-once —— 从最后一个快照重跑，快照之后已执行的工具会重复执行，LLM 也会重新调用产生真实费用。所以框架把幂等性要求显式写进了文档 warning 而不是偷偷保证。多副本目前没有分布式租约，需要自己做 leader election，不过代码里 `#acquireRecoveryLease` 和 `CachingPubSub.getLeaseProvider` 已经在铺路了。"

---

## 10. 资源清理

每个 `stream()` / `observe()` 都返回 `cleanup()`。不调也有兜底定时器（默认 30s，`cleanupTimeoutMs: 0` 可关）。

`cleanup()` 做三件事：
1. `streamCleanup()` —— 退订 pubsub
2. `runRegistry.cleanup(runId)` + `globalRunRegistry.delete(runId)` —— 释放不可序列化状态
3. `#clearPubsubTopic(runId)` —— 清 cache 历史和 counter

`#clearPubsubTopic` 的注释解释了为什么要清两个 topic：

```ts
void this.pubsub.clearTopic(AGENT_STREAM_TOPIC(runId));
void this.pubsub.clearTopic(`workflow.events.v2.${runId}`);
```

> The durable agentic loop runs on the default workflow engine, so the evented engine's terminal topic cleanup never runs for these runs — without this, CachingPubSub permanently orphans a no-TTL counter key per completed run.

**每个完成的 run 泄漏一个没有 TTL 的 counter key** —— 在 Redis 上跑久了就是灾难。

还有一段说明"为什么不需要防重入保护"，逻辑链很完整：

> cleanup timers arm only on terminal outcomes (FINISH/ERROR/ABORT — never SUSPENDED), `resume()` rejects runs whose snapshot isn't `suspended`, `untilIdle` continuations mint a fresh runId per segment, and cross-process `recover()` can't race a dead process's timer.

**这种"穷举所有可能重入路径并逐个排除"的注释，是面试里可以拿来展示代码阅读深度的好素材。**

---

## 11. 常见面试追问与答法

**Q: Durable Agent 和普通 Agent 的区别？**

A: 三层增量。①agentic loop 跑在 workflow 里，控制流可持久化、可跨进程续跑；②流走 PubSub 而不是 HTTP 连接，生命周期与请求解耦；③PubSub 上挂带序号的 cache，断线重连能补齐。API 上多了 `observe(runId)` 和 `resume(runId, data)`，`stream()` 多返回 `runId` 和 `cleanup`。

**Q: 断线重连怎么做到不丢、不重？**

A: `CachingPubSub` 给每个事件分配自增 index，先写 cache 再 live publish。订阅端 `subscribeFromOffset` 四阶段：先订阅并缓冲 → 拉历史按序投递 → 排空缓冲并去重 → 切透传用 index 水位线稳态去重。核心决策是**订阅先于读历史**，宁可重复也不能丢。去重键用 index 而不是 event.id，因为内层 pubsub 会重新生成 id。

**Q: 不可序列化的东西（工具的 execute 函数、模型实例）怎么办？**

A: 双通道。可序列化的降解成元数据进 workflow input（工具只留 name + JSON Schema + 审批标记，模型只留 `provider/modelId` 字符串）；不可序列化的进 `RunRegistry`（TTLCache，10 分钟过期、上限 1000）按 runId 索引。跨进程时 registry 落空，走 `resolve-runtime.ts` 从 Mastra 容器 + 元数据重建真身。fast path / slow path 分层。

**Q: 工具会被重复执行吗？**

A: 正常 suspend/resume 路径不会 —— foreach 的 `prevForeachOutput` 缓存已完成子项，resume payload 只消费一次，Inngest 下每个 step 还有 `step.run` memoization。但**崩溃恢复路径会** —— 从最后一个快照重跑，快照之后已执行的工具会重复。所以文档明确要求工具幂等。这个区分要主动说，它体现你读过 warning 而不只是 happy path。

**Q: 流式响应跨恢复怎么保持完整？**

A: `MastraModelOutput.serializeState()` 把所有 buffer（bufferedText、toolCalls、usage、messageList…）打包进 suspend payload 的 `__streamState` 键，恢复时作为新 output 对象的 `initialState` 注入。普通 durable workflow 没有这个问题，因为它没有"用户可见的增量累加器"。

**Q: 怎么定位到具体哪个工具调用挂起了？**

A: `suspend(payload, { resumeLabel: toolCallId })`。快照里存成 `resumeLabels: { [toolCallId]: { stepId, foreachIndex } }`。因为 foreach 里所有 item 的 stepId 都一样，光靠 stepId 定位不到第几个。

**Q: 用户刷新页面后 runId 从哪来？**

A: suspend 时把 runId 写进了 assistant 消息的 `content.metadata.pendingToolApprovals[toolName].runId`，并且强制同步 flush。前端只需要 threadId，从消息历史里就能读到 runId。

**Q: 为什么工具审批时并发度会掉到 1？**

A: foreach 的 suspend 语义是单挂起点 —— 记录第 k 个 item 挂起、恢复时只重跑第 k 个。并发跑多个工具、多个同时 suspend，就变成多点恢复，`Run.resume()` 会抛 `Multiple suspended steps found`。用并发度换语义简单性。而且并发度必须是每次求值的 resolver 而不是共享配置对象，因为 workflow graph 是所有 run 共用的。

**Q: `requestContext` 在 Inngest 下为什么要序列化？**

A: step 被 memoize 后函数体不再执行，对共享对象的 mutation 就丢了。所以要把 mutation 变成返回值的一部分跟着 memoize 走。Default 引擎共享内存不需要，用 `requiresDurableContextSerialization()` 开关避免无谓开销。

**Q: 生命周期回调怎么保证只调一次？**

A: Inngest 引擎把 `invokeLifecycleCallbacks` 覆盖成 no-op，改在一个 `step.run('...finalize')` 里调内部实现。`step.run` memoized，replay 不重复。

**Q: 快照会无限膨胀吗？**

A: 会，所以有 `pruneSnapshot`。不裁剪的话每次持久化都把对话重复序列化好几遍（step payload、AI SDK 的 `output.steps` 请求响应历史、已完成 step 上残留的 `__streamState`），大小随 **对话长度 × 历史挂起次数** 增长。`pruneAgentLoopSnapshot` 的规则是：终态 step 丢掉所有 resume 状态并剥离重型字段，非终态保留完整 suspendPayload，引擎路由状态一律不动。而且只注册在内部 agent workflow 上，用户自己写的 workflow 保留完整历史。

**Q: 这个设计的缺点是什么？**（一定要准备）

A: 四点：
1. **恢复是 at-least-once**。崩溃恢复重跑 LLM（真实费用）和工具。要求业务方保证工具幂等，这是把复杂度推给了使用者。
2. **多副本无分布式锁**。`recovery: 'auto'` 在多副本下会互相抢 run，目前只能靠外部 leader election。代码里 lease 相关的东西还在铺路阶段。
3. **审批场景并发退化到 1**。只要工具集里有任何一个可挂起的工具，整轮串行。可能的改进是按工具分组：可挂起的串行、其余并行。
4. **`running` 也持久化带来写放大**。为了崩溃恢复不得不这么做，`pruneSnapshot` 只是缓解不是根治。长对话 + 多轮工具调用下，Postgres 单行写入压力仍然实在。
5. **beta**：API 可能不发大版本就破坏性变更。上生产要锁版本。

---

## 12. 30 秒 / 3 分钟版本

**30 秒**：

> Durable Agent 把 agentic loop 包进 workflow，把流从 HTTP 连接搬到 PubSub，再在 PubSub 上挂一层带自增序号的 cache。于是三件事同时成立：loop 能跨进程续跑、客户端断线重连能补齐漏掉的 chunk、另一个客户端能用 runId `observe()` 接上同一个流。工具需要人工审批时 workflow suspend、状态落快照，人在任意进程调 `resume(runId, {approved:true})` 续上。执行引擎是可插拔的：开发用内存版，生产换 Inngest 就获得 step 记账、重试、限流和监控面板 —— agent 代码一行不改。

**3 分钟**：按官方三层展开，每层给一个源码锚点 + 一个设计取舍：

1. **Workflow 层** → `create-durable-agentic-workflow.ts` 的 `dowhile` + `foreach`
   取舍：并发度在有可挂起工具时强制降到 1，用并发换单挂起点语义
2. **PubSub 层** → `constants.ts` 的双向 topic 设计
   取舍：stream / control 分离，因为方向相反、消费者不同
3. **Cache 层** → `caching-pubsub.ts:subscribeFromOffset` 四阶段
   取舍：订阅先于读历史，宁可重复也不丢

再补一个跨层的取舍（`shouldPersistSnapshot` 包含 `running` 换崩溃恢复能力，用 `pruneSnapshot` 补性能）和一个缺点（恢复是 at-least-once，要求工具幂等），就是一个完整、有深度、有批判性的回答。

---

## 附 A：源码地图（上游 `mastra-ai/mastra`）

### Durable Agent 层

| 关注点 | 文件 |
|---|---|
| 工厂函数、`durable: true` 说明 | `packages/core/src/agent/durable/create-durable-agent.ts` |
| **DurableAgent 主体**（stream/resume/observe/recover） | `packages/core/src/agent/durable/durable-agent.ts` |
| topic 命名、事件类型、step id、默认值 | `packages/core/src/agent/durable/constants.ts` |
| 序列化边界（活对象 → workflow input） | `packages/core/src/agent/durable/preparation.ts` |
| 元数据降解 | `packages/core/src/agent/durable/utils/serialize-state.ts` |
| **跨进程重建运行时依赖** | `packages/core/src/agent/durable/utils/resolve-runtime.ts` |
| 不可序列化状态注册表（TTLCache） | `packages/core/src/agent/durable/run-registry.ts` |
| **PubSub → MastraModelOutput 适配** | `packages/core/src/agent/durable/stream-adapter.ts` |
| workflow 图（dowhile + foreach + 9 个 step） | `packages/core/src/agent/durable/workflows/create-durable-agentic-workflow.ts` |
| 模型调用 step | `packages/core/src/agent/durable/workflows/steps/llm-execution.ts` |
| **工具 step、审批、suspend** | `packages/core/src/agent/durable/workflows/steps/tool-call.ts` |
| 并发度解析 | `packages/core/src/agent/durable/workflows/shared/tool-call-concurrency.ts` |
| fire-and-forget 变体 | `packages/core/src/agent/durable/create-evented-agent.ts` |
| Inngest 变体 | `workflows/inngest/src/durable-agent/create-inngest-agent.ts` |

### 流与缓存层

| 关注点 | 文件 |
|---|---|
| **带序号缓存 + 四阶段 replay** | `packages/core/src/events/caching-pubsub.ts` |
| PubSub 抽象（`subscribeWithReplay` / `subscribeFromOffset`） | `packages/core/src/events/pubsub.ts` |
| 缓存后端接口 | `packages/core/src/cache/base.ts` |

### Workflow 引擎基座（本地 checkout 也有）

| 关注点 | 文件 |
|---|---|
| 快照类型 | `packages/core/src/workflows/types.ts:383` |
| 引擎抽象 + hook 定义 | `packages/core/src/workflows/execution-engine.ts` |
| 默认引擎、`wrapDurableOperation` | `packages/core/src/workflows/default.ts:190` |
| step 执行、`suspend()` 实现、`resumeLabel` | `packages/core/src/workflows/handlers/step.ts:385` |
| 快照写入 | `packages/core/src/workflows/handlers/entry.ts:163` |
| **loop / foreach 的 replay** | `packages/core/src/workflows/handlers/control-flow.ts:679, 992, 1168` |
| `Run.resume` 定位逻辑 | `packages/core/src/workflows/workflow.ts` |
| **快照裁剪** | `packages/core/src/loop/workflows/prune-snapshot.ts` |
| 流式状态序列化 | `packages/core/src/stream/base/output.ts` |
| Inngest 引擎覆盖 | `workflows/inngest/src/execution-engine.ts:105, 211` |

### 文档

- `docs/src/content/en/docs/harness/durable-agents.mdx`
- `docs/src/content/en/reference/agents/durable-agent.mdx`
- `docs/src/content/en/reference/agents/inngest-agent.mdx`
- 官方博客：[What are durable AI agents?](https://mastra.ai/blog/what-are-durable-ai-agents)、[Introducing Durable Agents](https://mastra.ai/blog/introducing-durable-agents)

## 附 B：本地 checkout（2026-01）对应位置

如果你要在**当前仓库**里对照阅读第 8 章的基座部分：

| 关注点 | 文件 |
|---|---|
| loop 入口、`__streamState` 还原 | `packages/core/src/loop/loop.ts` |
| ReadableStream 驱动、start vs resume | `packages/core/src/loop/workflows/stream.ts:139` |
| 外层 dowhile、stopWhen | `packages/core/src/loop/workflows/agentic-loop/index.ts:72` |
| 单轮编排、并发决策 | `packages/core/src/loop/workflows/agentic-execution/index.ts:44, 103` |
| **工具 step、审批、suspend** | `packages/core/src/loop/workflows/agentic-execution/tool-call-step.ts:277, 318` |
| Agent 用户 API | `packages/core/src/agent/agent.ts:3175`（resumeStream）`:3355`（approveToolCall） |
| 官方文档 | `docs/src/content/en/docs/agents/agent-approval.mdx` |

注意本地这版**没有** PubSub 流层、cache 层、崩溃恢复和 `pruneSnapshot`，`shouldPersistSnapshot` 也还是只在 `suspended` 时写。
