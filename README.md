# skills

这是一个面向 Codex / Claude Code 的 issue-agent 技能集合，用来把已准备好的实现问题安全分发给子代理，并在需要时持续推进到验证、合并和队列清空。

本仓库依赖并延伸 [mattpocock/skills](https://github.com/mattpocock/skills) 的 agent 工作流，尤其是假设项目里已采用 `ready-for-agent`、`$implement`、每个 issue/ticket 一个分支/工作树、draft PR/MR handoff、父代理验证和 merge gates 等约束。这里的两个 skill 不是替代基础开发技能，而是把 Matt Pocock skills 的实现流程编排成可重复的 ticket 分发和自动处理流程。

## 安装

```bash
npx skills add coreylyn/skills
```

安装后，技能会添加到 Codex / Claude Code 会话中，可以通过斜杠命令或 `$skill-name` 方式调用。

## 前置约定

- 仓库有可查询的 issue tracker 和 forge：GitHub、GitLab，或本地 Markdown issue 流程。
- issue 使用清晰的状态/标签，例如 `ready-for-agent`、`needs-info`、`needs-triage`、`blocked`。
- 单个实现任务能隔离到一个分支和一个工作树。
- 项目有足够的 `AGENTS.md`、`CLAUDE.md`、`README.md`、`CONTEXT.md` 或相关文档，供 AFK 子代理独立执行。
- 子代理实现阶段使用 `$implement`；本仓库负责调度、验证和合并门控。

## 包含的技能

### `/dispatch-tickets`

运行一轮安全分发：分析仓库状态、issue 依赖、标签、文档和工作树，只选择当前未阻塞、可实现、可隔离的问题，然后为每个 issue 启动一个子代理。

**适用场景：**
- 扇出一批当前可做的实现 issue
- 为每个 issue 创建独立分支和 `.worktrees/<branch-name>` 工作树
- 让子代理按 `$implement` 完成实现、测试、提交和 draft PR/MR
- 父代理只做 handoff 验证，不合并、不循环

**核心特性：**
- 只分发 ready、未阻塞、面向实现的问题
- 最多一次并发 3 个子代理
- 每个 issue 一个分支、一个工作树、一个子代理
- 固定 draft PR/MR handoff 合约
- 输出可审计 ledger：issue、分支、工作树、agent id、提交、PR/MR、验证结果和风险

### `/autopilot-tickets`

运行受控的 ticket-draining loop：刷新 tracker/forge 状态，先处理已跟踪 PR/MR 的验证和合并门控；没有可合并 PR/MR 时，再调用 `$dispatch-tickets` 分发下一批安全 ticket。

**适用场景：**
- 自动循环处理 `ready-for-agent` 队列
- 把已验证的 draft PR/MR 转为 ready-for-review
- 只合并通过全部 gate 的 agent-created PR/MR
- 每次合并后刷新 issue 依赖，再决定下一轮

**核心特性：**
- 不绕过 `$dispatch-tickets` 的分发规则
- 持续重建 blocker / dependency 状态
- merge 前验证 dispatch ledger、diff 范围、检查、评论、冲突和新阻塞
- 遇到安全条件或人工决策需求立即停止
- 输出每轮状态、merge gate 结果、merge SHA 或停止原因

## 工作流程

### 基本使用

1. **单次分发：**
   ```
   $dispatch-tickets
   ```
   分析并分发当前就绪的问题，最多 3 个。结束时留下 draft PR/MR 和父代理验证记录。

2. **自动驾驶模式：**
   ```
   $autopilot-tickets
   ```
   循环处理 ready 队列：先合并已安全通过 gate 的 PR/MR，再分发下一批；直到队列清空或遇到停止条件。

### 工作树管理

所有 ticket 工作树都创建在 `<project-root>/.worktrees/<branch-name>` 目录下。`dispatch-tickets` 创建并记录，`autopilot-tickets` 在验证和合并门控时把这个路径当作硬约束。

### 支持的平台

- **GitHub**: 优先使用 GitHub connector 工具，否则使用 `gh` CLI
- **GitLab**: 优先使用 GitLab connector 工具，否则使用 `glab` CLI
- **本地 Markdown 问题**: 支持基于文件的问题管理

## 就绪性标准

`dispatch-tickets` 只分发同时满足以下条件的问题：

- 开放、面向实现，有明确验收标准或无歧义结果。
- 不是 blocked、duplicate、stale、closed、design-only、discussion-only、`needs-info`、`needs-triage`、`ready-for-human` 或 `wontfix`。
- 没有未解决的产品/设计决策、冲突评论、缺失上下文或未完成依赖。
- 能隔离到一个分支和一个工作树。
- 仓库文档足够 AFK 子代理独立执行。
- 不需要 secrets、生产权限、破坏性数据变更或未解决的策略选择。

## 合并门控

`autopilot-tickets` 只合并通过所有 gate 的 PR/MR：

- PR/MR 由当前 loop 或已跟踪的 prior round 创建。
- PR/MR 正确关联到 assigned issue。
- source branch 和 worktree 匹配 dispatch ledger。
- 父代理已验证 `DONE` 状态、提交、diff 范围、PR/MR 描述和测试结果。
- draft PR/MR 由父代理验证后再转为 ready-for-review。
- diff 不超出验收标准。
- 必需检查通过。
- 没有未解决 review comments、merge conflicts、failed checks、新 blocker、人工决策或依赖变化。
- 不混入 human-authored changes，不触碰未经授权的 secrets、生产控制、破坏性迁移、支付、权限策略或合规文本。

## 何时停止

`autopilot-tickets` 会在以下情况停止并报告：

- 没有 open `ready-for-agent` implementation issue。
- 当前没有可安全分发的问题。
- 基础/schema/API/共享契约 issue 阻塞后续工作。
- merge gate 失败且无法安全修复。
- forge、tracker、git 或 subagent 工具不可用。
- 测试/检查失败，且不能用 targeted follow-up 解决。
- 仓库出现 loop 未创建的 dirty/conflicting 状态。
- repo policy、branch protection、review requirement 或用户决策需要人工介入。

## 许可证

MIT
