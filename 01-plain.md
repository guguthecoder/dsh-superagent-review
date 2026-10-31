# 阅读笔记：DeepSeek 的 Super-agent 原型是怎么设计它的核心问题的

2026 年 10 月 31 日，DeepSeek 发布了一篇架构文章《Super-agent 设计探索》，同时开源了 `dsh-superagent-prototype`。我把仓库完整读了一遍，跑了类型检查和全部 1197 个测试，想把自己读到的设计思路整理出来。

这篇笔记不试图评价它"好不好"，只想回答三个问题：**它要解决的核心问题是什么，它怎么解的，以及它把边界画在了哪里。**

先给一个整体印象：这是一个在状态一致性和授权建模上想得很清楚的系统，作者对它的适用范围也有明确认识，并且写在自己的文档里。下面尽量按作者的思路来讲。

---

## 一、它要解决的核心问题：副作用和状态对不上

所有 Agent 系统都会遇到同一件麻烦事。

假设 Agent 收到一个任务，它要做两件事：在数据库里记一笔"我开始干活了"，然后往群里发一条"收到，这就去做"。这两件事几乎不可能同时成功——要么先记数据库再发消息（发消息失败，数据库里留下一条永远对不上的记录），要么先发消息再记数据库（程序崩在中间，群里说"我做了"，数据库里查无此事）。

人写业务代码时这个问题有成熟解法，叫 **transactional outbox**：把"要发的消息"先写进数据库的同一张表、同一个事务里，事务提交之后，再由一个独立的投递器把消息真正发出去。这样账本永远是准的，消息发不出去就重试。

Agent 系统里不太常见这种写法，因为 Agent 的循环天然是"边想边做"，副作用到处都是。这个原型做了。

它的 `src/core/engine.ts` 是全库唯一能修改任务状态的地方。所有方法都是同步的，每条命令一个事务。需要调用外部世界的地方（发消息、调 GitHub、启动一个 Worker），不在事务里执行，而是在事务内往 `effect` 表里写一行，等事务提交之后再由适配器执行。

作者在架构文档里写得很直白：

> All Engine methods are synchronous. Each command changes its aggregates in one transaction. ... Ports enqueue external work inside the transaction; adapters perform it after commit.

（[docs/007-architecture.md#L48](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/docs/007-architecture.md#L48)）

这一条是这个系统很多其他性质的前提。比如它的崩溃恢复能写得那么细——哪些消息已经投递、哪些已经被读过、哪些任务中途被打断——是因为账本本身从不含糊。

## 二、它怎么划权限：把角色做进类型

Agent 系统里常见的做法，是在系统提示词里写一句"你只负责调度，不要亲自执行任务"。

这个原型换了个位置。它给不同的调用方发不同的"权限切片"：

```
EngineForSources | EngineForMainSession | EngineForPeople | EngineForSettings | EngineForExecutor
```

（[docs/007-architecture.md#L64](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/docs/007-architecture.md#L64)）

主 Session 拿到的方法里有 `decide`（决定接不接）、`attach`（把消息挂到某个任务上）；Executor 拿到的是 `assignmentStarted`、`report`、`turnEnded` 这类"汇报进度"的方法。**Executor 想越权接新任务，它在类型上根本没有这个方法。**

这件事的价值在于：提示词是"劝"，类型是"不存在"。模型可以忽略提示词，但没法调用一个不存在的函数。

这是我认为最值得借鉴的一条设计，而且它不依赖任何模型能力。

## 三、它怎么定义授权：把"谁批准了"变成数据

大多数 Agent 产品的授权是这样的：用户说了一句"帮我修一下这个 bug"，这句话留在对话历史里，之后所有动作的合法性都靠"我相信模型记得"。

这个原型把授权抽成了 `Mandate`（授权书）、`strength`（强度：显式请求，还是推断出来的）、`basis`（依据：来自哪个来源、哪个群、哪个白名单用户、哪一版策略）。这些是可以查询、可以重新校验的字段，不是埋在一长串对话里的语义。

策略一改，所有受影响的任务会收到通知并被重新校验（[docs/007-architecture.md#L94](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/docs/007-architecture.md#L94)）。

配套的 `docs/000-philosophy.md` 也值得一读。它把整个项目的设计原则收敛成一句话：

> Take as little human attention as possible, and give as much value as possible.

然后从这个原则往下推：只在被要求时行动、只在有人必须做决定时才说话、按影响而不是按工作量停下。这份文档解释了很多看起来"保守"的设计选择——比如主动工作默认只记录不执行。

## 四、它把边界画在哪里

读这份代码时，我觉得最有意思的一点是：**作者很清楚自己假设了什么，并且把这些假设写下来了。**

### 4.1 它假设的是"可信团队"

源码开头有一段注释：

> Provides default-path guardrails rather than an OS-level security boundary, as workers run under the operator's OS user.

（[src/infra/environment.ts#L1-L4](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/infra/environment.ts#L1-L4)）

架构文档里也有一句同样坦率的说明：

> Avoiding code reads is a prompt rule, not a tool restriction.

（[docs/007-architecture.md#L507](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/docs/007-architecture.md#L507)）

也就是说，那些命令包装和路径护栏，作用是"防止误用"，不是"防止恶意"。作者没有假装它是沙箱。文档末尾的开放问题里，第一条就是 trust：哪些字段可信、哪些不可信。这说明这个问题在他们的视野里，只是还没给机制。

理解这一点之后，后面很多设计就顺了：所有 Session 都用 `danger-full-access`（[src/app/sessions.ts#L152](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/app/sessions.ts#L152)），因为它本来就是给一个团队内部用的工具，不是给不可信用户跑的多租户服务。

### 4.2 跨出这个前提时，需要补什么

如果你的场景不再是"可信团队"，有几处需要留意。我说的是**需要留意**，不是"它有 bug"——在它设定的前提下，这些行为可能都是合理的。

**（1）外部内容进入的是主 Session。**

GitHub 评论正文、飞书消息、仓库 `AGENTS.md`、PR diff，都会进入主 Session 的上下文（[src/app/assessment.ts#L183-L220](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/app/assessment.ts#L183-L220)）。主 Session 同时持有三个配置类工具：`orchestrator_settings_save`、`orchestrator_policy_sync`、`orchestrator_prompts_save`（[docs/007-architecture.md#L492-L501](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/docs/007-architecture.md#L492-L501)）。

设置系统里本来有一个按来源校验的机制：

```ts
guard?(current: SettingsValue, candidate: SettingsValue, origin: SettingsOrigin): void
```

（[src/app/settings.ts#L29-L31](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/app/settings.ts#L29-L31)）

它目前用在 `executor`（[#L45](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/app/settings.ts#L45)）和 `source`（[#L51](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/app/settings.ts#L51)）两个块上。`policy` 块只有 `parse` 和 `merge`（[#L33-L39](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/app/settings.ts#L33-L39)），提示词的保存虽然接收了 `source` 参数（[src/app/prompts.ts#L202](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/app/prompts.ts#L202)），但只记录、不校验。

**我无法从代码判断这是有意还是无意。** 可能的解释有几种：也许策略本来就该允许主 Session 调整（否则没有人看着的时候它就僵住了），也许面板是唯一入口这个假设已经足够，也许只是还没轮到。这三种解释我都没法从代码里排除。

如果你要用在不可信环境里，这一处值得先想清楚：外部内容能到达的那个会话，手里有改策略的工具。

**（2）`gh` 包装脚本会透传任意子命令。**

```ts
export const WORKER_COMMANDS: readonly string[] = ['gh', 'git', 'lark-cli']
const run = (program: string): string => `${withoutShims}\nexec ${shellQuote(program)} "$@"`
await shim('gh', run(github.cli))
```

（[src/infra/environment.ts#L18](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/infra/environment.ts#L18)、[#L98-L100](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/infra/environment.ts#L98-L100)）

这三个命令不能被屏蔽（[src/policy.ts#L89](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/policy.ts#L89)），因为 Worker 要靠它们干活。代价是 `gh auth token` 这类子命令也会正常执行，输出会进入工具结果。

他们确实做了环境变量层面的清理——`GH_TOKEN`、`GITHUB_TOKEN` 都被剥掉了（[src/infra/environment.ts#L45-L58](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/infra/environment.ts#L45-L58)）。在"团队可信"的前提下，读自己的 token 不算问题；跨出这个前提时，它就是一个需要额外处理的地方。

### 4.3 一句话总结这一节

**它的安全模型是"护栏 + 可信团队"，不是"隔离 + 不可信输入"。** 这不是它做得不好，而是它选择解决的问题范围。文章第 8.4 节把安全与隐私列为开放问题，代码的状态和文章的表述是一致的。

## 五、可靠性的几个边界

这一节记三个我在阅读时停下来想过的地方。它们都是"在什么条件下会出问题"，不是"一定会出问题"。

**未送达的评估不会过期。** 超时只在候选送进主 Session 之后才开始算：

```ts
} else if (evaluation.delivered && evaluation.deadline <= now) { this.expire(thread); return }
```

（[src/core/evals.ts#L43](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/core/evals.ts#L43)）

文档里也写明了这一点："an undelivered candidate never expires"（[docs/002-authorization.md#L149](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/docs/002-authorization.md#L149)）。这是有道理的——没送到就不该算它超时；但它同时意味着"送达"这一步本身没有独立的兜底。

**有序队列会等在队头。** 有序的后台任务每次只看最老的一行，而重试次数没有上限：

（[src/app/effects.ts#L108-L113](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/app/effects.ts#L108-L113)、[#L141-L143](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/app/effects.ts#L141-L143)）

代码注释里明说了 "Ordered kinds block behind their oldest row even when it is not yet due"——顺序是刻意保证的。代价是：如果某一条永远发不出去，它后面的同类任务会一直等。这在"错误是偶发"的假设下没问题；如果错误是持久的，就需要一个放弃或转人工的出口。

**锁失效后的收尾比较简单。** `proper-lockfile` 用来防止启动第二个实例，锁失效时执行的是：

```ts
const compromised = (error: Error) => { this.runtimeHealth.set({ state: 'error', detail: error.message }); this.stopped.abort() }
```

（[src/app/runtime.ts#L92-L93](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/app/runtime.ts#L92-L93)）

记录健康状态并中止主流程。这在正常运维下够用；如果两个进程同时写的场景真的出现，收尾可以更彻底一些。

## 六、它的规模边界

这部分写给考虑"能不能用在更大场景"的读者。

- 一个实例管一个仓库：`repository` 是一个 `owner/name` 字符串字段（[src/deployment.ts#L46](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/deployment.ts#L46)）
- 单进程 + 单个 SQLite 文件（[docs/007-architecture.md#L452](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/docs/007-architecture.md#L452)）
- 旧记录暂不清理，作者写明了："Nothing deletes old records yet, so these grow without bound"（[docs/007-architecture.md#L484](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/docs/007-architecture.md#L484)）
- 轮询按会话串行执行（[src/sources/poll.ts#L19-L31](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/sources/poll.ts#L19-L31)），每个会话需要若干次 `gh` 调用
- 有若干处按全量集合扫描的查询（[src/core/replies.ts#L68](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/core/replies.ts#L68)、[src/executor/dsh.ts#L229](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/executor/dsh.ts#L229)、[src/sources/github-pull.ts#L45](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/sources/github-pull.ts#L45)、[src/app/store-read.ts#L22-L26](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/app/store-read.ts#L22-L26)）

这些放在一起，画出的是一条清晰的线：**一个团队、一个仓库、十几个并发任务，它工作得很好；再往上就需要换架构，而不是调参数。** 这条线作者没有明说，但从代码结构看是清楚的。

## 七、它和文章的关系：范围不同

这一节可能是最实用的。

| 文章描述的 | 仓库里的实现 |
|---|---|
| 双层架构（Router + Worker） | **有**。主 Session 就是 Router（[src/app/runtime.ts#L210](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/app/runtime.ts#L210)），Worker 是每个任务一个 Session 加独立目录（[src/executor/dsh.ts#L441](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/executor/dsh.ts#L441)）。隔离在上下文层面，不在进程或用户层面 |
| 事件两级过滤 | **有**。确定性 gate 在前（[src/core/intake.ts#L25-L29](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/core/intake.ts#L25-L29)、[src/policy.ts#L191-L211](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/policy.ts#L191-L211)），然后由主 Session 判断 |
| 会话订阅直通 | **有**（[src/core/intake.ts#L34-L59](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/core/intake.ts#L34-L59)） |
| Inline & Spill | **有**（[src/app/assessment.ts#L183-L220](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/app/assessment.ts#L183-L220)、[src/executor/dsh.ts#L123](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/executor/dsh.ts#L123)） |
| 链式召回 | **有一跳**（[src/core/intake.ts#L37](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/core/intake.ts#L37)、[src/app/links.ts#L47-L53](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/app/links.ts#L47-L53)） |
| 投机执行、异步 ReAct Loop、存算分离、独立 Watchdog | **尚未包含** |
| 文章第 7 节的实验数据 | 仓库不含实验代码 |

这件事 README 已经说明了：

> We have adapted it for publication, so it is not identical to our internal version or the implementation described in *Exploring Super-agent Design*. Some designs discussed in the article are directions for future development and are not implemented in this prototype.

所以更准确的说法不是"文章吹了"，而是：**文章描述的是一个系统的全貌，这个仓库公开的是其中一个可以独立运行的部分。** 两篇文章要讲的也确实是两件事：文章讲设计思路，仓库给可运行的样例。

如果你是为了理解设计而读它，完全够；如果你是想直接拿到文章里那套能力，会失望——但那个失望来自范围理解，不来自它做得不好。

## 八、一处我自己读错又纠正的地方

`src/executor/dsh.ts` 里有一处调用：

```ts
await this.host.sessions.prompt(sessionId, requestId, text, 'queue',
  () => { if (!current()) throw new Error('Launch authorization changed before worker delivery') },
  () => { this.host.engine.assignmentStarted(ticket.id, assignment.id) })
this.host.engine.assignmentStarted(ticket.id, assignment.id)
```

（[src/executor/dsh.ts#L466-L469](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/executor/dsh.ts#L466-L469)）

我一开始以为这是重复调用。核对之后发现：`Assignments.started`（[src/core/assignments.ts#L34-L43](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/src/core/assignments.ts#L34-L43)）确实不幂等，但它发出的事件目前没有消费者（[docs/007-architecture.md#L109](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/6337fc823ce20251c8de8085948165c81aaa701e/docs/007-architecture.md#L109)），投递也按未投递的消息去重。

更可能的情况是：这行代码是为了覆盖"回调没被触发"的分支——如果消息没被接受，`accepted` 不会执行，而 assignment 仍应被标记为运行中。**也就是它可能是有意的兜底，不是笔误。**

我把它留在这里，是因为"我怀疑了什么、又为什么收回"比我下什么结论更有参考价值。

## 九、附带的平台验证

顺手在 Windows 11 + Node 24.15 上跑了一遍完整测试。

| 项 | 结果 |
|---|---|
| `pnpm typecheck` | 0 error |
| `pnpm test` | 1158 通过 / 39 失败 |
| 修掉 3 个测试文件的可移植性问题后 | **1162 通过 / 34 失败** |

剩下 34 条都在 5 个文件里，原因是这些测试用 POSIX 语义构造替身：`#!/bin/sh` 假程序、`chmod 0o755`、`:` 作 PATH 分隔符、`kill -9 $$`、tmux 守护脚本。另有一类是本机没开开发者模式，创建符号链接报 EPERM。

测试里有一处跨盘符的问题：`prompts.spec.ts` 用 `relative()` 从仓库（D: 盘）指向临时目录（C: 盘），跨盘符时返回绝对路径，而 `packagePath` 用的是 `path.join()`——`join` 遇到绝对路径参数是拼接而不是替换。改成 `resolve()` 即可，POSIX 上无差异。

排查过程中我一度认为 `src/infra/environment.ts:142` 的 `file:${workerGitConfig(stateDir)}` 在 Windows 上会失配，并据此改了代码——**结果 200 多个测试连带失败，判断是错的**。git 会原样回显它收到的路径形式，配合 `-z` 也不做引号转义，所以原实现是对的。

这件事说明一个方法问题，值得记下来：**MSYS bash 里的 git，和经由 Node 子进程调用的 git，输出格式不一样。实验环境要贴近真实调用路径。**

另外补一个事实：这个仓库关闭了 PR、Issues、Discussions、Wiki。发布三小时内 751 star、49 fork，零 PR。看起来是把开源当作公开发布，而不是一个接收外部协作的项目——这个选择本身没什么可指摘的，只是读者需要知道。

## 十、读完之后我记住的三件事

1. **副作用要赶出事务**，用 outbox 把状态和对外动作串起来。这一条能省掉大量恢复逻辑的复杂度。
2. **权限做成类型，而不是提示词。** 提示词是劝，类型是不存在。
3. **把授权做成可查询的数据**，而不是对话历史里的语义。

以及一条关于边界的：**一个系统说自己"假设可信团队"，比它假装自己什么场景都能用，要可信得多。** 这个原型的文档在这一点上是诚实的，值得学。

## 复现

```bash
gh api repos/deepseek-ai/dsh-superagent-prototype/tarball > dsh.tar.gz
mkdir dsh-prototype && tar -xzf dsh.tar.gz -C dsh-prototype --strip-components=1
cd dsh-prototype
npx --yes pnpm@11.7.0 install
npx --yes pnpm@11.7.0 typecheck
npx --yes pnpm@11.7.0 test
```

文中代码链接均指向 commit `6337fc8`。代码若有更新，行号会漂移，但结论应可通过相同路径复核。
