# dsh-superagent-prototype 阅读笔记

对 DeepSeek 于 2026-10-31 开源的 Super-agent 原型 [`deepseek-ai/dsh-superagent-prototype`](https://github.com/deepseek-ai/dsh-superagent-prototype) 的阅读记录，基于 commit [`6337fc8`](https://github.com/deepseek-ai/dsh-superagent-prototype/tree/6337fc823ce20251c8de8085948165c81aaa701e)（TypeScript 约 15,205 行，59 个测试文件 / 1197 个用例）。

这不是缺陷清单。作者在 README、架构文档和源码注释里已经把自己的设计意图、适用范围和已知限制写得很清楚；这份记录做的事，是把这些线索集中起来，并标出几处我读过之后仍然不确定的地方。

## 两个版本

| 版本 | 适合 | 阅读时间 |
|---|---|---|
| **[阅读笔记（叙事版）](01-plain.md)** | 想理解「它要解决什么问题、怎么解的、边界画在哪」的读者 | 约 25 分钟 |
| **[阅读记录（条目版）](02-audit.md)** | 想按条目看观察、证据和待确认问题的读者 | 约 20 分钟 |

两版内容一致，详略与组织方式不同。

## 我读到的三件事

1. **副作用走事务。** 外部动作先写进同一事务内的 `effect` 表，提交后再执行。这让「状态与外部世界不一致」这一类问题从根上消失，也是它恢复语义能写得那么细的原因。
2. **权限做成类型。** 调用方只拿到按角色裁剪的 `Pick`，越权在编译期不成立。这一层不依赖模型行为，不会随模型版本漂移。
3. **授权做成数据。** `Mandate` / `strength` / `basis` 让「谁基于什么规则批准了这件事」可查、可重校验，而不是留在对话历史里。

另外一点关于边界的：作者明确写了这个原型假设「可信团队」，护栏是防止误用而不是防恶意，安全与隐私在文章里被列为开放问题。**说得清楚，比假装什么场景都能用，更值得参考。**

## 方法

- 完整克隆 → `pnpm install` → `pnpm typecheck` → `pnpm test`（Windows 11 / Node 24.15.0 / pnpm 11.7.0）
- 通读 `src/core`、`src/app`、`src/executor`、`src/sources`、`src/infra` 与 `docs/000`–`docs/008`
- 所有引用均附 `文件:行号`，链接指向固定 commit
- 否定性结论（如「未找到某机制」）均标注为检索结果，不等同于「不存在」
- 未做动态测试，未构造 PoC

## 测试结果

| 项 | 结果 |
|---|---|
| `pnpm typecheck` | 0 error |
| `pnpm test` | 1158 passed / 39 failed |
| 修改 3 个测试文件的可移植性后 | 1162 passed / 34 failed / 1 skipped |

剩余失败源于测试替身对 POSIX 语义的假设（`#!/bin/sh`、`chmod`、`:` 作 PATH 分隔符、`kill -9 $$`、tmux）以及本机未启用开发者模式导致的 `symlink` EPERM，详见条目版第 5 节。

## 说明

- 仅使用公开代码与公开文档，未构造可执行的攻击载荷。
- 该仓库已关闭 PR / Issues / Discussions / Wiki，无法通过仓库渠道反馈；若有不同意见，欢迎开 Issue 或直接联系。
- 文中判断基于有限时间的阅读，若与代码意图不符，以代码和作者的说明为准，欢迎指正。
