# skills

面向 Codex / Claude Code 的 ticket 编排技能。它们不替代实现技能，而是把已准备好的实现 ticket 组织成可审计的分发、验证和合并流程。

本仓库支持并延伸 [mattpocock/skills](https://github.com/mattpocock/skills) 的 agent 工作流。它默认项目已经采用 `ready-for-agent`、`/implement`、每个 issue/ticket 一个分支和工作树、draft PR/MR handoff、父代理验证、merge gates 等约束。

## 安装

```bash
npx skills add coreylyn/skills
```

## 技能

以下使用 Skill 名称；显式调用语法由 agent runtime 决定。

### `dispatch-tickets`

运行一轮安全分发：

- 刷新仓库、tracker、依赖、文档和工作树状态。
- 为每个选中的 ticket 创建独立分支、`.worktrees/<branch-name>` 工作树和子代理。
- 子代理通过 `implement` Skill 完成实现、验证、提交和 draft PR/MR handoff。
- 父代理等待并验证结果，输出 dispatch ledger。

它不合并 PR/MR，也不循环处理队列。Skill 本身没有固定并发上限；实际并发受 runtime 槽位、ticket 依赖和仓库安全状态约束。

完整流程与就绪规则见 [`skills/dispatch-tickets/SKILL.md`](skills/dispatch-tickets/SKILL.md)。

### `autopilot-tickets`

运行受控的 ticket-draining loop：

- 优先处理当前 loop 或已跟踪 prior round 的 PR/MR。
- 没有可合并 handoff 时，调用 `dispatch-tickets` 分发下一批安全 ticket。
- 父代理验证 draft、应用全部 merge gate，只合并安全的 agent-created PR/MR。
- 每次合并后刷新 tracker、依赖和 blocker 状态。
- 队列清空或安全条件要求停止时，结束并报告。

它复用而不绕过 `dispatch-tickets` 的分发规则。完整循环、合并门控和停止条件见 [`skills/autopilot-tickets/SKILL.md`](skills/autopilot-tickets/SKILL.md)。

## 前置条件

- 项目有可查询的 tracker，以及可创建 draft PR/MR 的 forge 流程。
- ticket 有清晰状态、依赖和验收标准，例如 `ready-for-agent`、`needs-info` 或 `blocked`。
- 单个实现 ticket 能隔离到独立分支和工作树。
- agent runtime 提供原生子代理能力。
- 项目文档足够 AFK 子代理独立执行，例如 `AGENTS.md`、`CLAUDE.md`、`README.md`、`CONTEXT.md` 或相关 docs。
- 环境已提供 `implement` Skill。

## 关键安全边界

- 每个 ticket 一个分支、一个工作树、一个子代理；依赖未完成时不分发后续 ticket。
- ticket 工作树固定在 `<project-root>/.worktrees/<branch-name>`，不复用其他任务的工作树。
- 子代理报告不是完成证明；父代理必须核对提交、diff 范围、测试和 handoff。
- autopilot 只处理可追溯到 dispatch ledger 的 PR/MR，并遵守仓库既有 merge 方法、检查和保护规则。
- 出现歧义、人工决策、新 blocker、冲突、失败检查或未授权高风险变更时停止，而不是绕过 gate。

## 许可证

MIT
