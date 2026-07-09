# skills

面向 Codex / Claude Code 的 issue-agent 技能集合。它把已经准备好的 issue/ticket 安全分发给子代理，并在需要时持续推进到验证、合并和队列清空。

本仓库支持并延伸 [mattpocock/skills](https://github.com/mattpocock/skills) v1.1.0 的 agent 工作流。它默认项目已经采用 `ready-for-agent`、`/implement`、每个 issue/ticket 一个分支和工作树、draft PR/MR handoff、父代理验证、merge gates 等约束。

这里的技能不替代基础实现技能；它们负责把基础实现流程编排成可重复、可审计的 ticket 分发和 autopilot 处理流程。

## 安装

```bash
npx skills add coreylyn/skills
```

## 技能

### `/dispatch-tickets`

运行一轮安全分发。

- 分析仓库状态、tracker、标签、依赖、文档和工作树。
- 只选择当前 ready、未阻塞、可隔离的实现 ticket。
- 最多并发 3 个子代理。
- 每个 ticket 一个分支、一个 `.worktrees/<branch-name>` 工作树、一个子代理。
- 子代理使用 /implement` 完成实现、测试、提交和 draft PR/MR。
- 父代理等待结果，验证 handoff，输出 ledger。

不负责合并 PR/MR，也不循环处理队列。

### `/autopilot-tickets`

运行受控的 ticket-draining loop。

- 刷新 tracker / forge / git 状态。
- 优先处理已跟踪 PR/MR 的验证和合并门控。
- 只合并通过全部 gate 的 agent-created PR/MR。
- 没有可合并 PR/MR 时，调用 /dispatch-tickets` 分发下一批安全 ticket。
- 每次合并后重新刷新依赖和 blocker 状态。
- 队列清空或遇到停止条件时结束并报告。

不绕过 /dispatch-tickets` 的分发规则。

## 前置条件

- 项目有可查询的 issue tracker：GitHub、GitLab，或本地 Markdown issue 流程。
- issue/ticket 使用清晰状态或标签，例如 `ready-for-agent`、`needs-info`、`needs-triage`、`blocked`。
- 单个实现任务能隔离到一个分支和一个工作树。
- 项目文档足够 AFK 子代理独立执行，例如 `AGENTS.md`、`CLAUDE.md`、`README.md`、`CONTEXT.md` 或相关 docs。
- 子代理实现阶段使用 /implement`；本仓库只负责调度、验证、合并门控和循环。

## 就绪标准

`/dispatch-tickets` 只分发同时满足这些条件的 ticket：

- open、面向实现，有明确验收标准或无歧义结果。
- 不是 blocked、duplicate、stale、closed、design-only、discussion-only、`needs-info`、`needs-triage`、`ready-for-human` 或 `wontfix`。
- 没有未解决的产品/设计决策、冲突评论、缺失上下文或未完成依赖。
- 能隔离到一个分支和一个工作树。
- 不需要 secrets、生产权限、破坏性数据变更或未解决的策略选择。

如果一个基础/schema/API/架构 ticket 阻塞后续工作，只分发这个基础 ticket。

## 合并门控

`/autopilot-tickets` 只合并通过全部 gate 的 PR/MR：

- PR/MR 由当前 loop 或已跟踪的 prior round 创建。
- PR/MR 正确关联 assigned ticket。
- source branch 和 worktree 匹配 dispatch ledger。
- 父代理已验证 `DONE` 状态、提交、diff 范围、PR/MR 描述和测试结果。
- draft PR/MR 已由父代理验证并转为 ready-for-review。
- diff 不超出验收标准。
- 必需检查通过。
- 没有未解决 review comments、merge conflicts、failed checks、新 blocker、人工决策或依赖变化。
- 没混入 human-authored changes，也没触碰未经授权的 secrets、生产控制、破坏性迁移、支付、权限策略或合规文本。

## 停止条件

`/autopilot-tickets` 遇到以下情况会停止并报告：

- 没有 open `ready-for-agent` implementation ticket。
- 当前没有可安全分发的 ticket。
- 基础/schema/API/共享契约 ticket 阻塞后续工作。
- merge gate 失败且无法安全修复。
- forge、tracker、git 或 subagent 工具不可用。
- 测试/检查失败，且不能用 targeted follow-up 解决。
- 仓库出现 loop 未创建的 dirty/conflicting 状态。
- repo policy、branch protection、review requirement 或用户决策需要人工介入。

## 许可证

MIT
