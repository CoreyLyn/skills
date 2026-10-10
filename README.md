# skills

面向 coding agent 的技能集合，包含整份 spec 从开发到合并的自动流程，以及中文解释。

本仓库延伸 [mattpocock/skills](https://github.com/mattpocock/skills) 的工作流：调用一次 `autopilot-spec`，由它先用 `implement-spec` 开发，再完成验收、审查、修复和自动合并。

## 安装

```bash
npx skills add coreylyn/skills
```

## 技能

以下使用 Skill 名称；显式调用语法由 agent runtime 决定。

### `autopilot-spec`

从一份 spec 及其关联 tickets 开始：

1. 调用 `implement-spec` 开发，等待子代理完成并形成集成 PR/MR。
2. 逐项验收整份 spec，审查整体改动，在原集成分支修复问题并复验。
3. 验收、审查、检查、审批和仓库合并条件全部通过后自动合并。
4. 核对 spec/ticket 关闭情况，安全清理工作树并报告结果。

显式调用默认授权该 spec 范围内的开发、修复、提交、推送和条件通过后的合并，无需逐步确认。若要求“只审查，不修改或合并”，则跳过开发，只验收已有成果。需要人工判断的验收项、缺少审批或无法验证的条件会阻止合并。

使用示例：

```text
使用 autopilot-spec 完成 spec #123，从开发到验收、审查和修复，全部条件通过后自动合并。

使用 autopilot-spec 验收 spec #123 和集成 MR !456，只审查，不修改或合并。
```

需要已安装的 `implement-spec` 及其所需能力、spec 和关联 tickets、项目验证环境，以及 tracker/forge 权限。续跑时复用已有集成成果；实现已完成则直接验收。开发编排交给 `implement-spec`，本 skill 只衔接验收和合并，不维护另一套 ticket 调度规则。

完整验收、合并条件和停止规则见 [`skills/autopilot-spec/SKILL.md`](skills/autopilot-spec/SKILL.md)。

### `wait-what-zh`

用自然、清晰的中文重新解释上一段回答，补齐背景和推理，沿用项目术语并保留原意。仅显式调用，无需 subagent。

完整提示词见 [`skills/wait-what-zh/SKILL.md`](skills/wait-what-zh/SKILL.md)。

## 从旧版迁移

`autopilot-spec` 替代了本仓库的 `autopilot-tickets` 和 `dispatch-tickets`。入口改为整份 spec，原来的 ticket 队列分发和逐 ticket 合并流程已移除。

更新安装时，请核对所用 agent 的已安装技能列表，移除旧的两个 skill 副本，避免仍触发旧流程。修改本仓库不会自动同步已有安装副本。

## 许可证

MIT
