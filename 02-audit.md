# 阅读记录：`deepseek-ai/dsh-superagent-prototype`

| 项 | 内容 |
|---|---|
| 对象 | `deepseek-ai/dsh-superagent-prototype`（Research prototype，MIT） |
| 版本 | `6337fc823ce20251c8de8085948165c81aaa701e`（2026-10-31） |
| 规模 | TypeScript 约 15,205 行 / 110 个源文件；测试 59 个文件 / 1197 个用例 |
| 范围 | `src/core`、`src/app`、`src/executor`、`src/sources`、`src/infra`、`docs/000`–`docs/008`、`cordis.patch.yml` |
| 方式 | 静态阅读 + 全量测试（Windows 11 / Node 24.15.0 / pnpm 11.7.0），未做动态测试 |
| 日期 | 2026-10-31 |

---

## 0. 这份记录想做什么

这不是一份缺陷清单。作者在 README、架构文档和源码注释里已经把自己的设计意图、适用范围和已知限制写得相当清楚；我能补充的，主要是把这些线索集中起来，并标出几处**我读了之后仍然不确定的地方**。

记录分三层，请按层阅读：

- **第一层（§1–§2）**：系统做了什么。这部分基本是复述作者的文档，附代码位置。
- **第二层（§3）**：我停下来想过的几处。每条都写「我观察到的」「我可能的理解」「我无法排除的其他解释」，最后是要确认的问题。
- **第三层（§4–§6）**：范围差异、平台验证、以及我没有覆盖的部分。

所有行号对应固定 commit。我读的时间不长，作者写的时间很长——下面的判断若与代码意图不符，以代码为准，以作者的说明为准。

---

## 1. 基线：三个我印象最深的设计

### 1.1 事务性副作用

`src/core/engine.ts` 是唯一写入者；方法全部同步；每条命令一个事务。外部副作用不在事务内执行，而是在同一事务内向 `effect` 表写入，提交后由适配器执行。

> *All Engine methods are synchronous. Each command changes its aggregates in one transaction. ... Ports enqueue external work inside the transaction; adapters perform it after commit.*
> —— `docs/007-architecture.md:48`

**我的理解**：这是 transactional outbox 模式。它让「状态已落库但消息未发出」「消息已发出但状态未落库」这两类不一致不再可能，代价是所有副作用都要先落一行记录。系统的恢复语义（哪些已投递、哪些已读、哪些被打断）能写得这么细，源头在这里。

### 1.2 类型级角色隔离

```
EngineForSources | EngineForMainSession | EngineForPeople | EngineForSettings | EngineForExecutor
```

—— `docs/007-architecture.md:64`、`src/core/engine.ts:269`

**我的理解**：越权在编译期不成立。这一层与模型行为无关，因此不会随模型版本漂移。我认为这是本仓库最容易被忽略、也最值得移植的部分。

### 1.3 授权作为领域对象

`Mandate` / `strength` / `basis` 把「谁基于何种规则批准了该工作」变成可查询、可重校验的数据；策略变更会触发受影响任务的重新校验（`docs/007-architecture.md:94`）。

**我的理解**：这让授权从「对话历史里的语义」变成「数据库里的记录」，也使 `docs/000-philosophy.md` 里那句 *take as little human attention as possible* 有了可执行的形式——因为授权被显式记录，系统才敢在没人看着的时候仍然知道边界在哪。

---

## 2. 作者自己写明的边界

这一节全部来自作者的文档与注释，不是我的推断。列出来是因为它决定了后面的所有讨论。

**可信团队假设。**

> *Provides default-path guardrails rather than an OS-level security boundary, as workers run under the operator's OS user.*
> —— `src/infra/environment.ts:1-4`

**提示词规则不是工具限制。**

> *Avoiding code reads is a prompt rule, not a tool restriction.*
> —— `docs/007-architecture.md:507`

**权限预设。**全部 Session 均使用 `danger-full-access`（`src/app/sessions.ts:152`），与该假设一致。

**范围。**README 明确说明本仓库是适配发布的研究原型，文章中部分设计属于未来方向，未在此实现。

**已知未解问题。**`docs/007-architecture.md` 末尾的 Open questions 第一条即 trust：哪些 Trigger 字段可信，哪些不可信。

**我的理解**：作者没有把「护栏」说成「沙箱」，也没有把「部分实现」说成「完整实现」。这一点在开源项目里不算常见，值得记一笔。

---

## 3. 几处我仍不确定的地方

以下每条都按同一格式：**观察 → 我的理解 → 我无法排除的其他解释 → 想确认的问题**。它们不是结论。

### Q-01 · `policy` 与提示词写入口的校验范围

**观察**

设置系统定义了按来源校验的钩子：

```ts
guard?(current: SettingsValue, candidate: SettingsValue, origin: SettingsOrigin): void
```
—— `src/app/settings.ts:29-31`

实际实现它的只有两个块：`executor`（`src/app/settings.ts:45`，注释写明 *Only the management panel may modify shell execution paths or commands*）与 `source`（`src/app/settings.ts:51`）。`policy` 块只有 `parse` / `merge`（`src/app/settings.ts:33-39`）；提示词保存接收 `source: SettingsOrigin`（`src/app/prompts.ts:202`），但只写入版本记录。

另外：主 Session 持有 `orchestrator_settings_save`、`orchestrator_policy_sync`、`orchestrator_prompts_save`（`docs/007-architecture.md:492-501`），而不可信内容（GitHub 正文、飞书消息、`AGENTS.md`）会经 `orchestrator_eval` 进入主 Session 上下文（`src/app/assessment.ts:183-220`）。

**我的理解**

在「可信团队」前提下，这大概率不构成问题：能写评论的人本来就是被允许的人，模型即使被误导，影响范围也在这个团队之内。

**我无法排除的其他解释**

1. 策略本就应当允许主 Session 调整——否则在没人值守时系统会僵住。
2. 面板是唯一实际入口，模型调用路径只是形式上的存在。
3. 这一处还没有轮到（guard 机制是后加的，先覆盖了其他块）。

三种解释我都没有从代码里排除。

**想确认的问题**

`policy` 与提示词两个写入口默认允许模型修改，是有意的设计（配合可信团队前提），还是尚未覆盖？如果是前者，理由是什么——这对判断「能否扩展到不可信输入场景」很关键。

---

### Q-02 · `gh` 包装脚本的子命令范围

**观察**

```ts
export const WORKER_COMMANDS: readonly string[] = ['gh', 'git', 'lark-cli']
const run = (program: string): string => `${withoutShims}\nexec ${shellQuote(program)} "$@"`
await shim('gh', run(github.cli))
```
—— `src/infra/environment.ts:18`、`#L98-L100`

这三个命令不能加入屏蔽列表（`src/policy.ts:89`），因为 Worker 依赖它们。`"$@"` 全量透传意味着 `gh auth token` 也会正常执行。

环境变量层面已清理 `GH_TOKEN` / `GITHUB_TOKEN`（`src/infra/environment.ts:45-58`）。

**我的理解**

同样地，在可信团队前提下，Agent 读取自己的凭据并不越界。这里的取舍很清楚：为了 Worker 能正常使用 CLI，放弃了子命令粒度的控制。

**我无法排除的其他解释**：子命令白名单需要随 `gh` 版本维护，成本可能高于收益；或者凭据本身已经按团队边界发放，读与不读无差别。

**想确认的问题**：是否有场景下希望限制到子命令粒度？如果有，当前结构留了扩展位吗？

---

### Q-03 · 未送达评估的兜底

**观察**

```ts
} else if (evaluation.delivered && evaluation.deadline <= now) { this.expire(thread); return }
```
—— `src/core/evals.ts:43`

文档表述：

> *The time limit starts when the candidate reaches the main Session. Time spent waiting for delivery does not count, and an undelivered candidate never expires.*
> —— `docs/002-authorization.md:149`

**我的理解**：「没送到就不该算超时」在语义上是对的——一个从未真正开始评估的候选，不应该因为等待而失败。

**我无法排除的其他解释**：送达环节本身有健康检查与重试，因此「永久送不到」在实践中不会静默发生；或者外部 Watchdog 会覆盖这种情况。

**想确认的问题**：如果送达反复失败，除了健康面板之外，有没有自动释放该会话队列的机制？还是预期由人从面板处理？

---

### Q-04 · 有序队列的队头等待

**观察**

有序 handler 每次只取 `seq` 最小的一行；该行处于退避期时同 kind 的其余行不派发。代码注释已声明这一行为：*Ordered kinds block behind their oldest row even when it is not yet due.*（`src/app/effects.ts:108-113`）

重试间隔有上限，`attempts` 只增不减，未见放弃分支（`src/app/effects.ts:141-143`）。

**我的理解**：顺序保证是必须的——比如同一会话的回复不能乱序。用「等最老一行」实现顺序，是最简单也最容易验证的做法。

**我无法排除的其他解释**：错误在实践中都是可自愈的偶发错误（网络、限流），因此无限重试是合理预期；或者运维层面有监控会在持续失败时介入。

**想确认的问题**：对「持久性失败」这一类的预期处理方式是什么——人工介入，还是计划中的自动出口？

---

### Q-05 · 锁失效后的收尾

**观察**

```ts
const compromised = (error: Error) => { this.runtimeHealth.set({ state: 'error', detail: error.message }); this.stopped.abort() }
```
—— `src/app/runtime.ts:92-93`

**我的理解**：上报健康状态并中止主流程，能让运维看到问题，也能阻止大部分后续写入。

**我无法排除的其他解释**：`stopped.abort()` 之后的写入路径可能已经在别处被拦住；真正的双写窗口可能小到可以忽略。

**想确认的问题**：锁失效之后，是否预期该进程完全停止写入？如果要，现有中止路径是否已经覆盖数据库句柄？

---

### Q-06 · `assignmentStarted` 的两次调用

**观察**

```ts
await this.host.sessions.prompt(sessionId, requestId, text, 'queue',
  () => { if (!current()) throw new Error('Launch authorization changed before worker delivery') },
  () => { this.host.engine.assignmentStarted(ticket.id, assignment.id) })
this.host.engine.assignmentStarted(ticket.id, assignment.id)
```
—— `src/executor/dsh.ts:466-469`

`prompt` 的第六个参数是 `accepted` 回调（`src/app/sessions.ts:162-163`）；`Assignments.started` 不幂等，每次都会 `emit('assignment.started')`（`src/core/assignments.ts:34-43`）。

**我的理解**：这更像是兜底——若消息未被接受，`accepted` 不会执行，而 assignment 仍应被标记为运行中。事件目前无消费者（`docs/007-architecture.md:109`），副作用有限。

**我无法排除的其他解释**：初始草稿遗留；或 `accepted` 在某种路径下会重复触发。

**想确认的问题**：这是有意覆盖两条路径，还是可以简化成一次？

---

## 4. 文章与仓库的范围差异

| 文章描述的 | 仓库中的实现 | 位置 |
|---|---|---|
| 双层架构（Router / Worker） | **有**。主 Session 即 Router；Worker 为每 Ticket 一个 Session + 独立目录。隔离在上下文层 | `src/app/runtime.ts:210`、`src/executor/dsh.ts:441` |
| 事件两级过滤 | **有**。确定性 gate 前置，再由主 Session 判断 | `src/core/intake.ts:25-29`、`src/policy.ts:191-211` |
| 会话订阅直通 | **有** | `src/core/intake.ts:34-59` |
| Inline & Spill | **有** | `src/app/assessment.ts:183-220`、`src/executor/dsh.ts:123` |
| 链式召回 | **一跳** | `src/core/intake.ts:37`、`src/app/links.ts:47-53` |
| 投机执行 | 未包含 | — |
| 异步 ReAct Loop | 未包含；以工具级 5 秒上限规避 | `docs/007-architecture.md:513` |
| 计算/状态/环境分离 | 未包含；本地锁状态目录 | `src/app/runtime.ts:93` |
| Watchdog | 未包含；健康面板 + 外部服务脚本 | `src/infra/health.ts`、`scripts/dsh-superagent` |
| §7 实验数据 | 仓库不含实验代码 | — |

**我的理解**：文章描述系统全貌，仓库公开其中可独立运行的一部分。两者的分工是明确的，README 也做了声明。若带着「拿到文章全部能力」的预期来读，会感到落差；若为理解设计而读，则相当充分。

---

## 5. 平台验证（Windows）

| 项 | 结果 |
|---|---|
| `pnpm typecheck` | 0 error |
| `pnpm test`（原始） | 1158 passed / 39 failed |
| `pnpm test`（修改 3 个测试文件后） | 1162 passed / 34 failed / 1 skipped |

剩余失败集中在 `service-script` / `sources` / `environment` / `deployment` / `runtime` / `commands`。成因是这些测试以 POSIX 语义构造替身（`#!/bin/sh`、`chmod 0o755`、`:` 作为 PATH 分隔符、`kill -9 $$`、tmux），以及本机未启用开发者模式导致的 `symlink` EPERM。**这是测试替身的平台假设，不是被测行为的问题。**

一处可移植性观察：`prompts.spec.ts` 以 `relative()` 跨盘符引用临时目录时会返回绝对路径，而 `packagePath` 使用 `path.join()`，其对绝对路径参数是拼接而非替换。改为 `path.resolve()` 可在 POSIX 行为不变的前提下解决。

**一处我读错并推翻的判断（方法记录）**：`src/infra/environment.ts:142` 的 `file:${workerGitConfig(stateDir)}` 曾被我判断为 Windows 必然失配。实测表明 git 原样回显所收到的路径形式，且 `-z` 下不做 C 风格引号转义，原实现正确。据此修改曾导致 200+ 用例失败并已回滚。**教训：MSYS bash 中的 git 与经 Node 子进程调用的 git 输出格式不同，验证须贴近真实调用路径。**

**协作状态**：PR（`POST /pulls` 返回 404）、Issues、Discussions、Wiki 均关闭；发布约三小时内 751 star / 49 fork，PR 数为 0。

---

## 6. 我没有覆盖的部分

1. 未做动态测试，未构造任何 PoC。
2. 未阅读 `src/client`（约 5,000 行 UI 代码）。
3. 未阅读 DSH 宿主实现（`@deepseek-ai/dsh-*`）。若安全属性或超时由宿主提供，本文相关判断需修正。
4. 关于「未找到超时」的表述是检索结果，不是穷尽性结论。
5. 未做负载测试，性能相关表述来自代码路径推断。
6. 未评估模型侧抗注入能力；任何与提示词注入有关的讨论都依赖模型行为，并随版本变化。

---

## 附录 A · 证据索引

| 编号 | 文件:行 |
|---|---|
| A1 | `docs/007-architecture.md:48` — Engine 同步与事务 |
| A2 | `docs/007-architecture.md:64` — 角色 `Pick` |
| A3 | `docs/007-architecture.md:94` — 策略变更通知 |
| A4 | `docs/007-architecture.md:109` — 事件消费者现状 |
| A5 | `docs/007-architecture.md:452` — 单 SQLite |
| A6 | `docs/007-architecture.md:484` — 记录暂不清理 |
| A7 | `docs/007-architecture.md:492-501` — 主 Session 工具表 |
| A8 | `docs/007-architecture.md:507` — 代码读取为提示词规则 |
| A9 | `docs/007-architecture.md:513` — 工具级等待上限 |
| A10 | `docs/002-authorization.md:149` — 未送达候选不过期 |
| A11 | `src/app/settings.ts:29-31` — `guard` 接口 |
| A12 | `src/app/settings.ts:33-39` — `policy` 块 |
| A13 | `src/app/settings.ts:45` / `:51` — executor / source 的 guard |
| A14 | `src/app/prompts.ts:202` — 提示词保存 |
| A15 | `src/app/assessment.ts:183-220` — 评估上下文 |
| A16 | `src/infra/environment.ts:1-4` — 护栏而非安全边界 |
| A17 | `src/infra/environment.ts:18` — `WORKER_COMMANDS` |
| A18 | `src/infra/environment.ts:98-100` — `gh` shim |
| A19 | `src/policy.ts:89` — 阻止屏蔽上述命令 |
| A20 | `src/app/sessions.ts:152` — `danger-full-access` |
| A21 | `src/core/evals.ts:43` — 过期条件 |
| A22 | `src/app/effects.ts:108-113` — 有序队列队头 |
| A23 | `src/app/effects.ts:141-143` — 重试上限 |
| A24 | `src/app/runtime.ts:92-93` — 锁失效处置 |
| A25 | `src/sources/poll.ts:19-31` — 串行轮询 |
| A26 | `src/deployment.ts:46` — 单仓库字段 |
| A27 | `src/executor/dsh.ts:466-469` — 两次调用 |
| A28 | `src/core/assignments.ts:34-43` — `started` 非幂等 |
| A29 | `src/app/sessions.ts:162-163` — `prompt` 签名 |

---

## 附录 B · 复现

```bash
gh api repos/deepseek-ai/dsh-superagent-prototype/tarball > dsh.tar.gz
mkdir dsh-prototype && tar -xzf dsh.tar.gz -C dsh-prototype --strip-components=1
cd dsh-prototype
npx --yes pnpm@11.7.0 install
npx --yes pnpm@11.7.0 typecheck
npx --yes pnpm@11.7.0 test
```

所有行号对应 commit `6337fc823ce20251c8de8085948165c81aaa701e`。
