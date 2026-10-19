# review-spec

> 多 Agent 并行方案文档 Review 循环 —— 写码之前，让一组只读子 Agent 拿着你的代码库把 spec 审到量化放行线。

Claude Code 的斜杠命令 / 插件：喂它一份设计文档（spec / 方案文档），它并行派出一组**分维度只读子 Agent** 对照你的真实代码库审查，产出结构化修复建议（patch），自动落盘、自动多轮复审，直到总分达标或熔断转人工。

## 安装

**方式一：插件市场（推荐，一行命令）**

在 Claude Code 里执行：

```
/plugin marketplace add yjjnls/review-spec
/plugin install review-spec@review-spec
```

**方式二：手动复制命令文件**

```bash
mkdir -p ~/.claude/commands && cp commands/review-spec.md ~/.claude/commands/
```

## 使用

```
/review-spec docs/specs/your-feature-design.md
```

- 不传路径时默认找 `docs/superpowers/specs/` → `docs/specs/`，都没有会直接询问
- 之后全自动：并行审查 → 汇总 → 自动修复落盘 → 自动复审，直到 **总分 ≥85 且零 P0** 放行，或熔断转人工交还给你
- 每轮结果写在 `<spec>.review-status.md`（与 spec 同目录），中断可续、历史可溯
- 多份 spec 可同时各起一套编队并行 review

## 特性

- **分维度并行编队**：A 代码对照（字段/API）· B 流程架构（流程/异常）· C 风险审计（并发/事务/一致性/性能/容量/安全）· D 开源对照（上游组件/K8s CRD，按需）· E 对抗审计（红队挑战设计假设，按需）——同一条消息并行派发，互不等待
- **代码对照（grounded）**：每个 Agent 必须实际读代码验证文档声明，禁止仅凭文档推断
- **patch 并行生成、串行落盘**：修复建议以自包含 patch 输出（定位摘录唯一 + 完整替换文本 + 代码依据），主会话合并后串行 apply，避免并发写冲突
- **量化放行**：各维度 0-10 分按权重（A30/B30/C25/D15/E15，缺席维度归一化）折算百分制，**总分 ≥85 且零 P0** 才放行
- **轮次留痕**：每轮写 `<spec>.review-status.md`（分数趋势/verdict/未决清单），通过标 APPROVED
- **省 token 循环**：修复只落在单一维度时只重派该维度复审；跨维度才全编队重跑
- **双重熔断**：5 轮上限；连续 2 轮总分不涨即停转人工
- **设计方向保护阀**：修复若实质是改设计方向（而非修正与代码事实的偏差），停下交给用户裁决——自动循环不替你做产品决策

## 与 superpowers 的配合（推荐工作流）

`/review-spec` 的原生槽位在 [superpowers](https://github.com/obra/superpowers) 工作流的「brainstorming 之后、writing-plans 之前」——正好补上 superpowers 缺失的一环：

```
superpowers:brainstorming           产出方案文档 docs/superpowers/specs/*.md
        ↓
/review-spec（本工具）               对照代码库多 Agent 审查 → 自动修复 → 量化放行
        ↓
superpowers:writing-plans           把放行的 spec 拆成实现计划
        ↓
superpowers:subagent-driven-development / executing-plans    实现（每任务自带 review）
        ↓
superpowers:finishing-a-development-branch                  合并收尾
```

**分工边界**（互补不重叠）：

| 环节 | 管什么 |
|------|--------|
| brainstorming | 想清楚（需求 → 方案） |
| **/review-spec** | **想的对不对**（spec ↔ 代码库事实一致性） |
| writing-plans | 怎么拆（spec → 任务） |
| subagent-driven-development | 做得对不对（实现 ↔ plan 一致性，每任务 task-reviewer） |

不装 superpowers 也能用：`/review-spec` 不依赖 superpowers 运行，任何设计文档都能审；装了则在流水线里严丝合缝。

## 使用示例

### 例 1：常规审查（全编队 + 自动收敛）

```
/review-spec docs/superpowers/specs/2026-10-19-portal-route-cache-design.md
```

典型过程（数字为示意）：

```
轮 1（全编队 A-D 并行）
  A 代码对照  8/10   2×P1（字段类型与现有 model 不符）1×P2
  B 流程架构  7/10   1×P0（缓存失效流程与既有 sync 链冲突）
  C 风险审计  6/10   1×P1（并发写缓存无一致性保障）
  D 开源对照  9/10   —
  总分 75/100 → 需修复 → 自动落盘 3 个 patch

轮 2（只重派受影响的 B、C，省 token）
  B 9/10（冲突已消除）  C 8/10 → 总分 ≥85 且零 P0 → APPROVED ✅
```

`2026-10-19-portal-route-cache-design.review-status.md` 留下完整审计链（逐轮分数/verdict/patch 摘要），随 spec 提交。

### 例 2：高风险方案（自动加派对抗审计）

方案触及生产写路径时（数据迁移、缓存切换、限流变更），编队自动加派 E 红队：

```
/review-spec docs/specs/2026-10-20-db-sharding-migration-design.md
# 触及生产写路径 → 编队 = A + B + C + E（对抗审计）
# E 专攻：隐性假设、实现分歧、测试歧义、回滚缺失、治理缺口
```

也可以对任意 spec 主动点名：`/review-spec <路径>` 后追加一句「加红队审」。

### 例 3：熔断转人工（设计分歧不是措辞问题）

```
轮 2 总分 74 → 轮 3 总分 74（修复只在外围打转，分数不动）
→ 无进展熔断：停下并列出决策点
   「方案 A（双写过渡）与方案 B（读侧切换）需二选一，这是方向问题，请裁决」
→ 你拍板后继续循环，或终止回去重写方案
```

自动循环修得好「文档与代码事实的偏差」，修不了「方向选错」——后者熔断交还给你，这是刻意设计。

## 设计笔记

- **为什么不是一条 prompt**：审查（读，可并行）与修复（写同一文档，必须串行）分离；只读审无共享状态，可以放心 fan-out
- **评分与问题清单绑定**：存在未修复 P0 不可能高分，防止「评分通胀」
- **上下文隔离**：子 Agent 不带会话历史，只传文档路径与项目根目录，防止先入为主

## 参考与致谢

循环机制借鉴/受启发于以下开源项目：

- [obra/superpowers](https://github.com/obra/superpowers) — subagent-driven-development 的 fan-out + 复审循环；本仓库的插件分发结构也参考了它
- [finna/Finn-loop](https://github.com/finna/Finn-loop) — verdict 状态化与 /loop 重跑
- [dzhng/skills](https://github.com/dzhng/skills) — 镜头间回环（修复重开受影响镜头）
- [yanlinyi101/spec-reviewer-skill](https://github.com/yanlinyi101/spec-reviewer-skill) — 100 分制 spec 质量评分
- [teterouge/spec-auditor](https://github.com/teterouge/spec-auditor) — 对抗式审计人设

## English

A Claude Code slash command / plugin for **multi-agent parallel review of design specs against your actual codebase**. Five specialized read-only reviewer agents (code-delta, architecture/flow, risk, upstream, adversarial) fan out in parallel; fix patches are generated in parallel but applied serially by the main session; the loop re-runs automatically until a quantified gate (weighted score ≥85/100 and zero P0) or a circuit breaker trips. Round history is kept in `*.review-status.md` next to the spec.

```text
# install via plugin marketplace (recommended)
/plugin marketplace add yjjnls/review-spec
/plugin install review-spec@review-spec

# or manual
cp commands/review-spec.md ~/.claude/commands/
```

Usage: `/review-spec <path-to-spec>`

## License

MIT
