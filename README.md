# dsh-superagent-prototype 审读

对 DeepSeek 于 2026-10-31 开源的 Super-agent 原型 [`deepseek-ai/dsh-superagent-prototype`](https://github.com/deepseek-ai/dsh-superagent-prototype) 的独立审读，基于 commit [`6337fc8`](https://github.com/deepseek-ai/dsh-superagent-prototype/tree/6337fc823ce20251c8de8085948165c81aaa701e)（TypeScript 约 15,205 行，59 个测试文件 / 1197 个用例）。

## 两个版本

| 版本 | 适合 | 阅读时间 |
|---|---|---|
| **[平实版](01-plain.md)** | 想理解「它好在哪、危险在哪、和文章差在哪」的读者；不要求工程背景 | 约 25 分钟 |
| **[审读报告](02-audit.md)** | 想直接看风险分级、证据索引、复现步骤的工程读者 | 约 20 分钟 |

两版结论一致，详略与语气不同。

## 三句话结论

1. 它的工程纪律显著高于同类开源项目：副作用走事务性 outbox，角色权限做进类型，授权落成可审计的领域对象。
2. 它的安全边界实际上是提示词：主 Session 持有改 `policy` 与提示词的入口，而这两个入口恰好是整套设置系统里仅有的、能自我扩权却没有 origin 校验的地方。
3. 它是单仓库、单进程、可信团队内的原型，不是文章描述的多任务、跨设备自治系统——Watchdog、投机执行、存算分离在代码里都不存在。

## 方法

- 完整克隆 → `pnpm install` → `pnpm typecheck` → `pnpm test`（Windows 11 / Node 24.15.0 / pnpm 11.7.0）
- 通读 `src/core`、`src/app`、`src/executor`、`src/sources`、`src/infra` 与 `docs/000`–`docs/008`
- 所有结论均附 `文件:行号`，链接指向固定 commit
- 否定性结论（如「未发现某机制」）均标注为「未穷尽」，不等同于「不存在」

## 测试结果

| 项 | 结果 |
|---|---|
| `pnpm typecheck` | 0 error |
| `pnpm test` | 1158 passed / 39 failed |
| 修复 3 个文件的可移植性问题后 | 1162 passed / 34 failed / 1 skipped |

剩余失败全部源于 POSIX 进程语义或本机未启用开发者模式，详见审读报告第 4 节。

## 披露

- 仅使用公开代码与公开文档，未构造可执行的攻击载荷。
- 该仓库已关闭 PR / Issues / Discussions / Wiki，无法通过仓库渠道反馈。
