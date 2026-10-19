# review-spec

> 多 Agent 并行方案文档 Review 循环 —— 写码之前，让一组只读子 Agent 拿着你的代码库把 spec 审到量化放行线。

Claude Code 的斜杠命令 / Agent Skill：喂它一份设计文档（spec / 方案文档），它并行派出一组**分维度只读子 Agent** 对照你的真实代码库审查，产出结构化修复建议（patch），自动落盘、自动多轮复审，直到总分达标或熔断转人工。

## 特性

- **分维度并行编队**：A 代码对照（字段/API）· B 流程架构（流程/异常）· C 风险审计（并发/事务/一致性/性能/容量/安全）· D 开源对照（上游组件/K8s CRD，按需）· E 对抗审计（红队挑战设计假设，按需）——同一条消息并行派发，互不等待
- **代码对照（grounded）**：每个 Agent 必须实际读代码验证文档声明，禁止仅凭文档推断
- **patch 并行生成、串行落盘**：修复建议以自包含 patch 输出（定位摘录唯一 + 完整替换文本 + 代码依据），主会话合并后串行 apply，避免并发写冲突
- **量化放行**：各维度 0-10 分按权重（A30/B30/C25/D15/E15，缺席维度归一化）折算百分制，**总分 ≥85 且零 P0** 才放行
- **轮次留痕**：每轮写 `<spec>.review-status.md`（分数趋势/verdict/未决清单），中断可续、历史可溯，通过标 APPROVED
- **省 token 循环**：修复只落在单一维度时只重派该维度复审；跨维度才全编队重跑
- **双重熔断**：5 轮上限；连续 2 轮总分不涨即停转人工
- **设计方向保护阀**：修复若实质是改设计方向（而非修正与代码事实的偏差），停下交给用户裁决——自动循环不替你做产品决策

## 安装

```bash
# 方式一：作为斜杠命令
mkdir -p ~/.claude/commands && cp commands/review-spec.md ~/.claude/commands/

# 方式二：作为 Agent Skill
cp -r skills/review-spec ~/.claude/skills/
```

## 使用

```
/review-spec docs/specs/your-feature-design.md
```

不传路径时默认找 `docs/superpowers/specs/` → `docs/specs/`，都没有会直接询问。多份 spec 可同时各起一套编队并行 review。

## 设计笔记

- **为什么不是一条 prompt**：审查（读，可并行）与修复（写同一文档，必须串行）分离；只读审无共享状态，可以放心 fan-out
- **评分与问题清单绑定**：存在未修复 P0 不可能高分，防止「评分通胀」
- **上下文隔离**：子 Agent 不带会话历史，只传文档路径与项目根目录，防止先入为主

## 参考与致谢

循环机制借鉴/受启发于以下开源项目：

- [obra/superpowers](https://github.com/obra/superpowers) — subagent-driven-development 的 fan-out + 复审循环
- [finna/Finn-loop](https://github.com/finna/Finn-loop) — verdict 状态化与 /loop 重跑
- [dzhng/skills](https://github.com/dzhng/skills) — 镜头间回环（修复重开受影响镜头）
- [yanlinyi101/spec-reviewer-skill](https://github.com/yanlinyi101/spec-reviewer-skill) — 100 分制 spec 质量评分
- [teterouge/spec-auditor](https://github.com/teterouge/spec-auditor) — 对抗式审计人设

## English

A Claude Code slash command / Agent skill for **multi-agent parallel review of design specs against your actual codebase**. Five specialized read-only reviewer agents (code-delta, architecture/flow, risk, upstream, adversarial) fan out in parallel; fix patches are generated in parallel but applied serially by the main session; the loop re-runs automatically until a quantified gate (weighted score ≥85/100 and zero P0) or a circuit breaker trips. Round history is kept in `*.review-status.md` next to the spec.

```bash
# install as slash command
mkdir -p ~/.claude/commands && cp commands/review-spec.md ~/.claude/commands/
# or as an Agent skill
cp -r skills/review-spec ~/.claude/skills/
```

Usage: `/review-spec <path-to-spec>`

## License

MIT
