# Pico Evolution Local-First 改造计划

> 状态：实施前设计稿（2026-10-04）。本文件取代 `self-evolution-v2.zh-CN.md` 作为当前 canonical 计划；原 v2 保留为生产化方向的历史设计参考。

## 0. 决策摘要

Pico 当前没有生产环境，近期目标不是建设灰度发布平台，而是把现有 AppWorld Runtime 补丁搜索器改造成一套完整、可信、可复现的本地自进化实验系统：

```text
Development 轨迹
  → 失败/成功对照与候选生成
  → 类型化预检
  → Development 筛选
  → 冻结候选与 EvaluationPlan
  → Fresh Validation 晋升
  → 类型化审批
  → 采用为下一版本地实验基线
  → 最终 Sealed Test 审计
```

本轮明确删除或后置：生产 DeploymentRecord、分布式 CAS、reconciler、shadow/canary、worker ACK、多租户、生产 Session 快照和紧急流量回滚。

本轮保留轻量的本地采用与回退：通过评测的候选可以成为下一轮 `base_sha` 或本地能力 profile；回退依赖 Git commit 和版本化资产指针，不修改线上流量。

## 1. 要解决的问题

当前 Evolver 的核心链路可以复用，但有五个直接阻碍：

1. 只有 Runtime label 真正具备 fixture/evaluator；Skill 和 Prompt 仍为 unsupported。
2. `coverage_satisfied`、L1 告警和空 outcome 没有正确控制搜索循环。
3. Development/train 数据同时承担生成和晋升，候选容易适应固定题集；sealed test 只有最终报告作用。
4. Skill、Prompt、Runtime 的影响面不同，却只有 `gated` 和 `human_review` 两档审批语义。
5. AppWorld 编辑器仍在宿主机执行 `bash -c`；这会污染本地实验，也可能访问工作树外文件和凭据。

本计划不追求在线模型训练、自由修改 Evolver/evaluator、安全策略、依赖和测试，也不允许一个候选同时修改 Skill、Prompt 与 Runtime。

## 2. 复用与新增边界

继续复用：

- Git node/tree、worktree、journal、manifest digest 和 sealed runner；
- focused screen、sentinel、full confirm 的两级漏斗；
- `SkillRegistry`、`LocalSkillCatalog`、BM25 与 `local_dirs`；
- `SandboxExecutor` / BoxLite；
- AppWorld scorer 和 task-level measurement。

新增四个小而明确的层：

- `CandidateEnvelope`：三类候选共享身份、来源、假设和 digest，payload 保持类型化；
- `EvaluationPlan`：候选冻结后固定对照、数据、指标、预算和停止规则；
- `PromotionDecision`：只判断独立证据是否支持改进；
- `LocalAdoptionDecision`：根据副作用、人工复核和 evidence 决定是否成为下一版本地基线。

同时必须迁移旧状态语义。当前 `promoted_to_baseline` 把 Development 通过、独立证据通过和成为父版本混在一起，不能在新路径中继续作为 `accepted` 的同义词。新状态至少拆为：

```text
draft
  → dev_selected
  → validation_accepted | validation_rejected | validation_failed | validation_inconclusive
  → adoption_pending | adoption_approved | adoption_rejected
  → baseline_adopted | baseline_reverted
```

旧 evidence/artifact 统一标记 `legacy_train_only`，只能用于审计和 Development 对照，不能自动迁移成 fresh Validation evidence。

## 3. 统一候选合同

所有候选共享以下最小字段：

```yaml
schema_version: 1
candidate_id: cand_<sha256>
label: skill | prompt | runtime
base_sha: <git sha>
baseline_profile_digest: <prompt + skill catalog + tools + context config>
source_evidence_ids: []
hypothesis:
  failure_mode: ...
  mechanism: ...
  expected_rescue: ...
  possible_harm: ...
affected_task_families: []
change_class: C1 | C2 | C3
content_digest: <sha256>
policy_version: evolver_policy_v1
generator_identity: ...
generator_prompt_digest: ...
evaluator_profile: ...
status: draft
```

`candidate_id` 由 canonical envelope（不含自身 ID）和 typed payload 共同求 digest，不接受生成器自由命名。Skill/Prompt 候选不能只绑定 `base_sha`，因为相同代码配不同 Prompt、Skill catalog 或 tool/context config 并不是同一个基线。

Payload 按类型分开：

- Skill：完整 `SKILL.md`、检索描述、适用条件和反例；
- Prompt：预声明 slot、完整渲染前内容和渲染后 prompt digest；
- Runtime：`AppliedPatch`、before/after digest、changed paths 和 child commit。

一个候选只能有一个 label、一个主要机制和一个 base SHA。混合候选无法做效果归因，首版硬拒绝。

建议把可演化文本资产放在仓库内独立数据面，而不是让候选修改核心代码：

```text
evolution_assets/skills/<skill-name>/SKILL.md
evolution_assets/prompts/<slot-name>.md
```

Skill 评测时将候选 worktree 的 `evolution_assets/skills` 作为现有 `LocalSkillCatalog.local_dirs` 输入；Prompt 只通过一个由人工实现、固定代码读取的结构化 slot 注入。Runtime 继续使用 Git patch。这样三类候选都能沿用 Git tree，同时用户的 `workspace/skills` 不会被覆盖。

正式实验不能读取实时变化的用户 workspace。运行前将完整 Skill catalog 快照并求 digest，control/challenger 共用该冻结快照；candidate Skill 使用独立 source namespace，并在 trace 中记录 `candidate_id`、content digest、physical source 和 rendered digest。同名资产除非明确替换 exact baseline asset，否则 fixture 直接拒绝。

三个控制概念必须分开：

- `promotion_control`：候选所基于的 exact parent composite profile，决定它是否可以成为下一基线；
- `paper_reference_c0`：冻结的 vanilla，仅用于长期报告；
- `method_reference`：例如 current Runtime-only Evolver，用于方法比较，不自动成为每个候选的晋升对照。

正式主实验还要在任何 method × seed 生成前冻结唯一 `formal_common_parent_profile_digest`。M2–M4 的 Skill/Prompt/Runtime 工程验收采用独立 `engineering/<adapter>` namespace，测试后回退，不能改变 formal base；否则不同方法将不再拥有共同 promotion control，无法做 shared-panel 直接比较。

### 3.1 三个机器合同

`EvaluationPlan` 在领取 Validation 前冻结，至少绑定：campaign/evaluation ID、全部候选与 exact control 的 composite digest、runner/evaluator/verifier 版本、dataset/split/group/lineage manifest、primary/secondary contrasts、family 权重、CI 与多重校正规则、K/retry/timeout、预算、arm-order/bootstrap seeds、resolved model/provider revision、tool schema、Prompt/Skill digest，以及 sandbox image/network/mount/env policy digest。

`PromotionDecision` 只能引用一个冻结 plan，包含 plan/candidate/control digest、exposure record、measurement digest、每道 gate 的输入数值与 `accepted|rejected|failed|inconclusive` 终态。

`LocalAdoptionDecision` 只能引用 `accepted` 的 promotion，包含 promotion/evidence/policy digest、risk class、审查过的 diff/资产 digest、actor、timestamp、`approved|rejected` 和理由。任一绑定对象变化，决定自动失效。

## 4. 副作用与审批模型

### 4.1 两道门

审批必须拆成两道互不替代的门：

1. **证据门**：`PromotionDecision` 回答“独立实验是否证明它更好”。
2. **采用门**：`LocalAdoptionDecision` 回答“考虑影响面后，是否把它设为下一版本地基线”。

规则：

- `accepted` 不等于 `approved`；
- 人工可以拒绝已经通过实验的候选；
- 人工不能把 `failed`、`inconclusive` 或出现硬违规的候选强行批准；
- approval 必须绑定 candidate digest、base SHA、evidence digest 和 policy version；任一变化后原审批失效；
- 候选生成模型不能签发自己的 risk、score、promotion 或 approval。

### 4.2 审批设计如何对应调研

这里不是给三类候选套同一个“人工看一眼”流程，而是把[业界调研与当前实现审计](self-evolution-survey.zh-CN.md)中已经反复出现的边界落成代码规则：

| 调研经验 | 对 Pico 的含义 | 本计划的审批落点 |
|---|---|---|
| OpenAI Evals、DSPy/GEPA 都把候选生成与 dataset/grader/validation 分开 | 生成器不能证明自己更好，训练题提分也不能直接晋升 | 三类候选先过独立 `PromotionDecision`，再进入采用门；生成模型不能签发 verdict |
| Claude Skills/Memory 与 Hooks 分离，Voyager 的 Skill 需要可检索、可执行验证 | Skill 是可版本化文本资产，通常不直接执行代码，但错误触发仍会污染上下文和工具选择 | 允许 C1 Skill 自动实验；用正负检索、ITT、token 和冲突证据控制副作用；首版仍默认人工采用 |
| Prompt Optimizer、DSPy/GEPA 依赖独立验证，但 Prompt 往往影响整段行为 | 即使只改文本，Prompt 的作用面也可能跨所有任务和工具 | 只开放结构化 slot，必须跑全 family 回归，并始终由人决定是否采用 |
| Google ADK/OpenHands 强调确定性控制流、测试和隔离；DGM/ADAS 的代码自改仍属高风险研究 | Runtime 不是“另一种文本”，而是可执行控制面，可能访问文件、网络、凭据或评测器 | 只在 strict sandbox 运行；执行前做范围许可，晋升后仍需完整 diff 与人工合并；信任根改动硬拒绝 |

所以审批强度由**可执行性、作用面和最坏副作用**决定，而不是由文件后缀决定。这里借鉴的是成熟系统共同的“外部评测、确定性边界、分离权限”，没有照搬当前不需要的生产灰度发布。

### 4.3 变更分级

分级由作用域、权限、可执行性、路径和 diff 规模中的最高风险决定。LOC/token 只是升级人工审查的触发器，不能覆盖语义判断。

| 等级 | 定义 | 处理 |
|---|---|---|
| C1 bounded | 单一资产、单一机制、局部作用域、无权限或依赖变化 | 允许自动生成和自动实验 |
| C2 broad | 跨 family、全局上下文、多个文件或核心控制流影响 | 人工允许采用；Runtime 还需在执行前人工许可 |
| C3 prohibited | 触碰信任根、扩大权限、混合资产或无法归因 | 自动进化路径硬拒绝，转普通人工开发 |

### 4.4 三类候选审批矩阵

| 类型 | 主要副作用 | C1 边界 | 必须提供的额外证据 | 默认采用审批 |
|---|---|---|---|---|
| Skill | 错误检索、过度触发、与现有指令冲突、上下文膨胀、误导工具选择或诱导危险操作 | 单个纯 Markdown Skill；允许创建/修改该 Skill 的 bounded name/description/body；单一 family；不含附件、requirements、`always`、权限、依赖和宽泛触发；建议变更不超过 300 tokens | relevant retrieval recall、irrelevant false-activation、注入/读取 trace、命令诱导/凭据/路径/外传负例、目标任务提升、非目标 sentinel、token 增量 | 证据通过后进入 `evidence_passed_waiting_adoption`；首版默认人工一键采用，可通过显式实验配置开启自动采用 |
| Prompt | 全局行为漂移、指令优先级冲突、工具/格式变化、token 与稳定性变化 | 只改一个预声明 `evolvable_slot`；建议不超过 200 tokens；不改 role、工具 schema、安全指令 | before/after rendered prompt、冲突/注入测试、全 family 回归、tool/format 合规、多 seed 稳定性、token/latency | 始终人工采用；全局 system prompt 不进入 C1 |
| Runtime | crash、死循环、状态污染、API 破坏、资源消耗、权限绕过和评测操纵 | 单个 allowlist 文件、单一机制、无新依赖/API/权限；建议不超过 80 changed LOC | 完整 Git diff、lint/type/unit/integration/smoke、调用路径与触发 trace、crash/timeout/resource、安全负例、paired benchmark | C1 可在 strict sandbox 自动实验，但始终人工合并；C2 执行前和合并前各一次人工确认 |

Skill C2 仅包括多个 Skill 或显著扩大既有 Skill 的适用 family/触发范围；带 requirements、附件或权限变化直接进入 C3。Prompt 首版只有单 slot C1，多 slot、role/system/security/tool schema 和全局 Prompt 都是 C3/unsupported。Runtime C2 只允许 2–3 个明确 allowlist 的 benchmark-local agent/tool 文件；Pico core loop、context assembler、tool registry、Evolver、sandbox 与 evaluator 永久是 C3。C2 的规模上限初始建议为 Skill 1000 tokens、Runtime 300 changed LOC；超过上限进入 C3，由普通开发流程处理。

change class 必须由 host 根据 normalized paths、diff、AST/metadata 和 capability delta 计算，不能相信候选自报；canonical policy table 与 policy version digest 同时写入 manifest、EvaluationPlan、promotion 和 adoption。

### 4.5 硬拒绝条件

以下条件不可通过人工 override：

- 修改 evaluator、grader、答案、validation/sealed ledger、Evolver、sandbox/security、审批 policy、测试、依赖或 CI；
- candidate 能读取 validation/sealed 内容、宿主凭据或审批记录；
- Skill/Prompt 带可执行附件、新依赖或权限扩张；
- Runtime 新增未允许的网络、subprocess 或文件系统能力；
- 一个候选混合修改多个资产类型；
- base SHA、内容 digest、证据或 split provenance 不完整；
- validation 已向生成器暴露或存在近重复数据泄漏；
- 确定性安全不变量出现一次违规；
- 通过修改测试或 scorer 使自己得分更高。

## 5. 三类候选的具体实现改动

### 5.1 Skill

生成器输入同 family 的失败轨迹、成功轨迹和当前可用 Skill，输出最小 Skill delta、适用条件、反例和预计伤害。

需要实现：

1. `skill_v1` fixture：验证 frontmatter、UTF-8、纯文本、最大长度、无脚本/二进制/secret/危险 URL、单一逻辑 Skill；
2. candidate worktree 的 evolved Skill source；
3. retrieval fixture：正例应检索，负例不应误检索；
4. 冻结 experimental catalog 和独立 candidate namespace；Skill 注入/读取 trace 绑定 physical source 与 content digest，区分“没被检索”“被检索但未读取”“被读取但无收益”；
5. primary evaluation 使用全部预注册任务的 intent-to-treat 分数，不能只对实际命中 Skill 的任务算提升；
6. control/challenger 分别使用独立 HOME、TMP、XDG/cache、session、watcher/index；
7. approval 后把 candidate commit 记录为下一实验基线，不直接覆盖用户 workspace Skill。

### 5.2 Prompt

首版不允许改写完整 `SOUL.md`、`AGENTS.md` 或 evaluator prompt。由人工先在 Context 组装中增加一个稳定的 `evolvable_slot`，候选只能修改该 slot 的文本资产。

需要实现：

1. `prompt_slot_v1` fixture：验证 slot identity、最大 token、无 role/schema/security 变化；
2. 固定渲染器，证据同时保存模板 diff 和最终 rendered prompt；
3. 冲突检测：与 bootstrap、Skill、tool description 的相反指令；
4. prompt injection、输出格式、tool selection/argument 合规回归集；
5. 全 family validation；Prompt 不能只在目标失败子集上晋升；
6. approval 后仅更新本地实验 profile 指向的 prompt digest。

### 5.3 Runtime

保留现有 detached worktree、Git child commit、Manifest/G5 和 AppWorld runner，但收紧执行与晋升。

需要实现：

1. 所有 edit/import/test/eval 路径走 `SandboxExecutor` 的冻结 `evolver_strict_v1` profile，`DirectExecutor` 不可用于自动 Runtime 候选；
2. allowlist 逐项扩展，不允许候选修改 `pico/evolver`、评测、sandbox、安全、测试和依赖；
3. 增加 lint/type/unit/integration/smoke 和安全负例；
4. 主指标按全部预注册任务计算，activation beacon 只做机制分析，不缩小晋升分母；
5. 人工看到完整 diff、测试矩阵和资源变化后，才能把 child commit 设为下一基线；
6. 回退使用 previous Git commit 或 revert commit，禁止通过 `reset --hard` 实现产品功能。

`evolver_strict_v1` 不是一个名字检查，而是具体安全合同：`allow_net=false`、禁止 extra volumes、环境变量显式 allowlist、不给宿主凭据；生成阶段仅有 candidate overlay 可写；hard-reject 在第一次 candidate import/exec 前完成；import/test/eval 从 child commit 建立全新 checkout，trust root、grader 和 verifier 只读；每个 arm/task 使用独立 VM、HOME、cache、workdir 和 session；verifier 在 candidate VM 外运行。实际 backend、immutable image digest、mount table、network/env policy 与资源上限全部进入 EvaluationPlan。没有满足这些条件的 backend，Runtime 自动实验直接 `failed`。

## 6. 实验评测协议

### 6.1 一次 campaign 的不可变顺序

```text
1. 注册数据分组、模型、工具、预算和指标
2. 在 Development 上运行 exact parent `promotion_control`；另冻结 C0 仅作报告参照
3. 每个 method × search-seed 生成多个候选，只在 Development 上自适应筛选
4. 每个 method × search-seed 仅按 Development 选出一个 champion
5. 只用 Development 指标按预注册 selector 选定一个 `preselected_final_artifact`，并冻结 selector inputs/decision/digest
6. 所有待比较 champion、formal common control、primary/secondary contrasts 一起冻结进 EvaluationPlan
7. 原子领取一个从未暴露的 shared Validation panel
8. 所有 frozen arms 在同一 panel、同 seed policy、同预算下做 shared-control 配对运行
9. 生成每个候选的 PromotionDecision 和预注册的方法比较；无论结果如何，本次 evaluation 结束
10. accepted 候选进入类型化采用门；不得按 Validation 选择“最佳 seed”或替换预选 final artifact
11. 只有预选 final artifact 自身通过 PromotionDecision 时，才与同一 formal common control 运行一次 Sealed Test；否则保留 sealed 未暴露并结束研究
```

这里把 **search campaign**（一个 method × generator seed 在 Development 上产出一个 champion）与 **fresh evaluation**（多个已冻结 arm 在同一 panel 上比较）分开。唯一 primary contrast 是 `Skill-first vs current Runtime-only Evolver`；Prompt、hardened Runtime、C0 及消融属于 secondary contrasts，并使用 Holm 校正。search seed 表示候选实现随机性，不是额外 task 样本，不能按 Validation 挑最好的 seed。

首版 confirmatory estimand 明确收缩为：**这 3 个预注册 Skill artifacts 的固定平均表现，与这 3 个预注册 current Runtime-only artifacts 的固定平均表现之差，在新 task groups 上是否成立**。置信区间只外推 task/group，不声称覆盖未来 generator seeds。若要声称“pipeline 对候选生成随机性也普遍更好”，必须另开扩展实验，先由 power simulation 冻结更多 seed 数，并使用 seed × lineage-group crossed/hierarchical bootstrap；3 seeds 不足以支撑该广义 claim。

Validation 失败后不能根据结果修改候选再重试。若开启新研究，只能使用新的 Validation panel；上一轮 Validation 的 task ID、shard identity、轨迹、逐题分数、grader 输出永不回流 generator。人工若看到聚合 verdict 后继续设计，必须标记为 sequential adaptation，并重新冻结 multiplicity 与 claim。这样避免反复刷同一 holdout。

### 6.2 数据划分

- **Development**：允许查看任务、轨迹和逐题分数；用于诊断、生成、focused screen 和 full-dev confirm。
- **Validation reservoir**：按 group/lineage 分组；candidate freeze 前不可见；每次 fresh evaluation 只领取一个 shared panel。
- **Sealed Test**：方法、阈值和候选类型选择全部冻结后使用一次；只报告最终泛化，不再根据结果换候选。

AppWorld 首版从现有 train IDs 内按 group 划出 Development 和 Validation reservoir，现有 test IDs 保持 sealed。group assignment 必须在 task、template、episode 和近重复 lineage 的传递闭包上全局排他，并覆盖历史 campaign。具体比例和每个 family 的最小样本数由 M0 pilot 决定；power pilot 只能使用 Development、合成数据或单独的 pilot-only groups，不能打开正式 reservoir。若 fresh group 不足，只能报告 `inconclusive` 或扩大数据，不能复用已暴露 panel。

Exposure ledger 使用不可逆状态：

```text
reserved → claimed → running → completed | failed → exposed → retired
```

原子 `claimed` 后即永久消耗，进程 crash 也不得放回池。记录 dataset digest、group/lineage IDs、evaluation/plan digest、claim nonce、可见角色、暴露级别和时间。

### 6.3 配对运行与独立单位

- 独立统计单位是 task/template group，不是同一 task 的多次 attempt；
- control 与 challenger 使用相同模型、工具、权限、token/time budget 和 seed policy；
- 每个 arm 使用干净 workspace/state；arm order 按 group × repetition 阻断随机化，由 EvaluationPlan seed 固定，并记录 queue/start/end/provider request ID；
- Development 可沿用 `K=3` 估计模型随机性；正式 Validation 首版默认 `K=1`，保持 task-level binary outcome。若 pilot 证明必须使用 `K>1`，须在计划中预先定义聚合规则，且重复 attempt 仍不能当作独立样本；
- 方法主比较中，每个 task 先对该方法的 3 个 frozen search-seed champions 求固定平均，再与另一方法形成 task-level paired difference；该 estimand 只代表这 3 个预注册 artifacts，不把 seed 当成 3 倍样本量，也不外推 generator-seed 分布；
- candidate generation 的调用和 token budget在比较方案间保持一致；
- `agent_budget_timeout` 算 task failure/0；provider transport timeout、sandbox/runner crash 算 infra；未启动或缺记录为 `inconclusive`；
- retry 以完整 paired block 对称重跑，两臂采用同一个有效 block attempt；旧 attempt 只留作审计，不进入统计分母。

### 6.4 指标

共同主指标：

- `macro_task_success_lift`：按预注册 family 权重聚合的 paired task success 提升。

共同次指标：

- 目标 failure family 的 rescue rate；
- 非目标 family 的 regression rate；
- task success、tool error、invalid action；
- input/output token、调用次数、墙钟时间；
- candidate 生成和评测总成本；
- infra failure 与 inconclusive 比例。

成本同时报告 `marginal_evaluation_cost` 和包含生成/筛选的 `end_to_end_search_cost`，避免把没有 generator 成本的 C0 与搜索方法混成一个比率。Latency 从 task 被 runner 接受至终态 artifact 落盘；agent budget timeout 按 cap 值计入 task latency，infra timeout 不进入 task p95、单独进入 infra rate。p95 的最小样本数和估计方法由 M0 冻结；样本不足则该 gate 为 `inconclusive`。

类型特有指标：

| 类型 | 特有指标 |
|---|---|
| Skill | relevant retrieval recall、irrelevant false activation、read/injection rate、注入 token |
| Prompt | rendered prompt token、格式遵循、tool selection/argument 合规、跨 seed 方差 |
| Runtime | unit/integration 通过率、crash/timeout、资源峰值、机制覆盖、安全负例 |

### 6.5 晋升判定

M0 pilot 先估计 variance、discordance 和可承受成本，再将阈值写入 EvaluationPlan。第一版建议以以下值作为 pilot 起点，而不是永久常量：

- 最小有意义提升 `delta_min = 5pp`；
- 关键 family 非劣界 `-3pp`；
- token/cost 增长上限 `10%`；
- p95 latency 增长上限 `15%`。

Promotion 必须同时满足：

1. 两臂 measurement 完整，无未解决 infra 污染；
2. 按 task group 聚类的 95% 单侧置信下界大于冻结的 `delta_min`；
3. paired point lift 与完整区间均报告；
4. 每个关键 family 的下界高于预注册非劣界；
5. 类型特有门槛通过，确定性安全不变量零违规；
6. token、成本、延迟和资源不超过 cap；
7. split、artifact、base、model、tool 和 config digest 完整。

正式统计顺序固定为：attempt 先聚合到 task；artifact 与 formal common control 形成 task-level paired difference；方法比较先在 task 内对 3 个预注册 artifacts 求固定平均；在每个 family 内按 immutable group/lineage 聚类重采样；最后按冻结 family 权重汇总。主报告使用至少 10,000 次、seed 预注册的 95% 单侧 stratified group bootstrap lower bound；最小 group 数由 M0 power pilot 冻结。该区间条件于冻结的 3 artifacts，不包含 generator-seed 不确定性。`K=1` 的 exact McNemar 仅作描述性辅助，不能忽略 group 相关性充当正式 gate；`K>1` 使用预注册 paired permutation/bootstrap。唯一 primary contrast 不校正，secondary contrasts 和关键 family 多重判断使用 Holm。若 group 太少或区间无法计算，结果为 `inconclusive`。所有阈值、quantile 方法和停止规则必须在领取 Validation panel 前冻结。

token/cost/resource cap 首版作为 EvaluationPlan 中的确定性预算硬门；latency cap 只有达到预注册最小样本时才判定，否则为 `inconclusive`。如果 M0 显示 `lower_bound > 5pp` 在可用 group 数下没有可行 power，应在正式运行前调整 claim 或扩充数据，不能在看过 Validation 后放宽成 `point ≥ 5pp 且 lower > 0`。

### 6.6 Sealed Test 规则

- 显式改写当前 sealed runner/unseal 路径，只评 Development 阶段预选且通过 Validation 的 final artifact 与 formal common control；旧的 vanilla + 每轮 deliverable curve 仅作为 legacy Development 报告；
- 不向候选生成器返回逐题轨迹；
- 结果暴露后数据永久记为 used；
- test 结果无论好坏都必须报告，不能再选择另一个候选；
- 任何根据 sealed 结果进行的修改都属于新实验，必须使用新的 sealed 数据或明确降级为 development 观察。

Sealed 只回答“这个预选 artifact 相对 exact base 是否泛化”，不作为 `Skill-first vs current Runtime-only` 方法级 primary claim 的第二次检验；方法级结论来自 shared Validation panel。

## 7. Claim-Driven 实验块

### C1：预注册 Skill-first artifacts 在不增加回归的前提下优于预注册 current Runtime-only artifacts

唯一 primary contrast：相同 formal common base、模型、任务和候选预算下，3 个预注册 Skill-first artifacts 的固定平均在同一 fresh Validation panel 上优于 3 个预注册 current Runtime-only artifacts 的固定平均，并满足关键 family 非劣、成本和 Skill 特有门槛。C0、Prompt 与 hardened Runtime 为 secondary contrasts，不以“至少一个类型成功”替代 primary claim。此结论外推新 task groups，不外推未来 generator seeds。

### C2：成功/失败对照和类型化 gate 是收益来源，不只是更多模型调用或更大的搜索空间

最小可信证据：在相同候选数量与 token budget 下，success/failure contrast 优于 failure-only；去掉 fresh validation 或类型特有 gate 会增加假晋升、误检索或回归。

需要排除的解释：收益仅来自更多 token、更多候选、测试泄漏，或 Runtime 改动拥有更大表达空间。

必须执行的实验块：

1. **完整性与统计 sanity**：split/group 泄漏测试、metric golden cases、synthetic null/positive campaigns。
2. **主结果**：C0、current Runtime-only Evolver、Skill-first、Prompt slot、hardened Runtime 的所有预注册 seed champion 先一起冻结，再在同一 shared Validation panel 以共同 control、等候选预算比较；Skill-first vs current Runtime-only 为唯一 primary，其余用 Holm 校正。
3. **生成机制消融**：failure-only、success/failure contrast、手工 Skill upper-bound。
4. **门禁消融**：score-only、共同 gate、类型化 gate；reused validation 仅用于展示过拟合风险，不参与正式晋升。
5. **失败与成本分析**：未检索 Skill、Prompt 全局回归、Runtime crash/资源问题及每次 accepted improvement 的总成本。

主结果使用 3 个预注册 search seeds 描述候选生成随机性；每个 seed champion 都必须报告并进入冻结计划，不能把 `seed × task` 当独立样本，也不能按 Validation 选择最佳 seed。pilot 和昂贵的 Runtime 安全矩阵可以先用 1 seed 验证管线，再决定是否进入正式运行。

## 8. 代码改动清单

| 模块 | 计划改动 |
|---|---|
| `pico/evolver/candidate_manifest.py` | 引入 common envelope、typed payload、change class、scope、permissions/executable delta；Skill/Prompt/Runtime 分别绑定 fixture/evaluator/approval policy |
| `pico/evolver/candidate_evidence.py` | 从 Runtime-only 扩展为 typed evidence；Development screen 与 fresh Validation promotion 分离；ITT 分母不可由 trigger 动态缩小 |
| `pico/evolver/candidates/`（新增） | `base.py` protocol，`skill.py`、`prompt.py`、`runtime.py` 三个 adapter；各自负责生成、materialize、preflight 和 mechanism trace |
| `pico/evolver/evaluation/`（新增） | `plan.py`、`splits.py`、`exposure.py`、`decision.py`；冻结计划、group 分配、shared fresh panel 账本和 PromotionDecision |
| `pico/evolver/approval/`（新增） | `policy.py` 计算 C1/C2/C3 与审批路线；`artifacts.py` 保存 LocalAdoptionDecision；`baseline.py` 保存 current/previous local baseline |
| `pico/evolver/tree/node.py`、journal readers/writers | 用新状态机替换 `promoted_to_baseline` 多义语义；旧记录显式迁移为 `legacy_train_only`，不能伪造 Validation 状态 |
| `pico/evolver/orchestrator/loop.py` | 消费 L1/readiness；空 outcome 不耗 patience；每个 method × seed 只在 Development 选一个 champion，交由独立 multi-arm fresh evaluation；分离 dev selection、promotion 与 adoption |
| `pico/evolver/orchestrator/production.py`、`pico/evolver/activation/summary.py` | evidence hook 从 Development gate 后移到 fresh Validation verdict 后；重命名为 assembly/adoption 或保留 deprecated facade；summary 不再把 dev selection 算 accepted |
| `pico/evolver/orchestrator/gates/*` | 保留 measurement validity；新增 effect-size、clustered interval、family noninferiority、成本和类型特有 gate；旧 mean>0 仅 legacy |
| `pico/evolver/launch/config.py` | 增加 development/validation/sealed、group file、candidate label、approval mode、budget 和 metric 配置；全部进入 config fingerprint |
| `benchmarks/appworld/evolve/entry.py` | 加载三份 split 和 group metadata；拒绝 group overlap；为三类 adapter 构建对照与 challenger 环境 |
| `benchmarks/appworld/evolve/eval.py` | 将现有基于路径猜测 runtime label 的逻辑委托给 typed adapter；三类候选都从冻结 envelope materialize |
| `pico/evolver/orchestrator/sealed/runner.py`、launch unseal | 从“vanilla + 每轮 deliverable”改为只接受一个 frozen final challenger 与 exact frozen control；exposure 后不可继续有效搜索 |
| `benchmarks/picobench/plan.py`、`records.py`、`statistics.py` | 扩展 campaign/group/lineage/digest 与 blocked arm-order 字段；实现 family-stratified group paired bootstrap，不能原样复用当前 task bootstrap |
| `pico/memory_engine/skill_local/*` | 复用 `local_dirs` 挂载冻结 experimental catalog 与 candidate source；trace 加 source/digest；正式评测不读取实时 workspace，拒绝未声明同名覆盖 |
| `pico/context_engine/factory.py`、`pico/context_engine/assembler.py`、`pico/context_engine/segments/` | 在真实 `ContextAssembler` 请求路径人工新增固定的 `evolution_prompt` slot；候选只能提供 slot 内容，不能改 Context 组装代码 |
| `pico/evolver/applier/path_guard.py` | 将 evaluator、Evolver、sandbox/security、测试、依赖和账本固定为 trust root；只允许类型 adapter 写入其声明的数据面 |
| `benchmarks/appworld/evolve/sandbox.py`、`pico/sandbox/config.py`、`boxlite_executor.py` | 移除 host `bash/subprocess` 候选路径；实现并验证 `evolver_strict_v1` 的 image、只读 mount、禁网、env/resource/isolation 合同；无合格 backend 时 fail closed |
| `pico/evolver/cli.py` | 增加 `inspect`、`review`、`adopt`、`rollback-local` 和 `report`；命令操作 evidence/baseline，不做生产部署 |

当前 `activation/artifacts.py` 的不可变 evidence bundle 可以复用，但 `activated` 命名改为本地 adoption 语义，或者由新的 approval 模块替代。不要继续让一个字符串状态同时表示实验通过、人工同意和实际采用。

`evaluation/` 应复用 PicoBench 的 canonical plan digest、comparison-block 对称 retry 和 artifact discipline；不能原样搬用当前只按 task 重采样的 `statistics.py`。需先扩展 `plan.py`、`records.py`、`environment.py`、`isolation.py` 和 `verifier.py` 的 group/lineage 与环境身份，再提取通用实现。Prompt slot 必须接入真实请求路径 `ContextAssembler`/`context_engine`，旧 `ContextBuilder.build_system_prompt()` 只用于估算和兼容，不能成为唯一实现。

## 9. 测试计划

必须新增：

- 三类 candidate fixture/evaluator 的 unit test；
- C1/C2/C3 policy table 与 hard-reject 参数化测试；
- failed/inconclusive 不能被人工 override；candidate/control/profile/evidence/policy 任一 digest 改变使 approval 失效；
- 旧 `promoted_to_baseline` journal 不得变成 validation accepted；新状态迁移与 summary 分类测试；
- Skill 正负检索集、冻结 catalog、同名冲突、source digest trace 和 cache/HOME 隔离测试；
- Prompt rendered snapshot、冲突、注入、tool/format 回归测试；
- Runtime 无 strict sandbox 不执行、越界访问和 grader 篡改负例；
- development/validation/sealed group/lineage overlap、claim 后 crash 永久 retired、generator 无法读取 Validation identity/trace 的测试；
- attempt→task→paired difference→family-stratified group bootstrap 的 null/positive/小样本 golden test；
- `draft → dev_selected → validation_accepted → adoption_approved → baseline_adopted → baseline_reverted` 本地端到端测试；
- 三类候选均能恢复 previous exact digest/commit 的回退测试。

## 10. 依赖顺序与里程碑

### M0 — 基线与数据协议（3–5 天）

- 固定 Python 3.12 环境并跑现有 Evolver/Skill/Sandbox 测试；
- 冻结 AppWorld Development/Validation/Sealed 的 group/lineage split；power 只用 Dev/合成/pilot-only 数据；
- 重现 C0 和 current Runtime-only baseline；
- pilot 估计样本量、variance、discordance、成本与阈值；验证 exposure ledger 和 `evolver_strict_v1`。

**退出条件**：相同 base/config 可复现；group 无泄漏；EvaluationPlan 可冻结。

### M1 — 公共合同和控制流修复（1 周）

- CandidateEnvelope、EvaluationPlan、PromotionDecision、LocalAdoptionDecision；
- 迁移 `promoted_to_baseline`、summary 和 journals，拆开 dev/validation/adoption/baseline 状态；
- 修复 L1、coverage/readiness、zero-outcome patience、sealed reuse；
- exposure ledger、typed evidence 和新 gate skeleton。

**退出条件**：synthetic null 不假晋升；failed/inconclusive 无法进入审批。

### M2 — Skill 纵向切片（1–2 周）

- evolved Skill data surface、冻结 experimental catalog、Skill adapter、retrieval 正负 fixture；
- success/failure contrast generator；
- Skill-specific evaluation、review、adopt 和 rollback-local。

**退出条件**：一个纯文本 Skill 从轨迹生成到成为下一实验基线，并可精确回退。

### M3 — Prompt slot（1 周）

- 固定 evolvable slot、rendered digest、Prompt adapter；
- conflict/tool/format/global regression 评测；
- human approval 与本地 profile 切换。

**退出条件**：Prompt 不能越过 slot，且完整回归门通过后才能采用。

### M4 — Runtime 加固（1–2 周）

- strict sandbox 全路径；
- 静态/单元/集成/安全矩阵；
- ITT promotion 和双人工门；
- Git baseline/revert 工作流。

**退出条件**：Runtime 候选不能接触 trust root，所有 accepted commit 可重放、审查和回退。

### M5 — 主实验与报告（1 周）

- 先冻结唯一 formal common parent/profile；M2–M4 工程采用不得污染它；
- 冻结所有方法 × 3 search-seed champions，并用 Development-only deterministic selector 预选 final Skill artifact；
- 在同一 shared panel 完成 multi-arm 主比较、生成/门禁消融、成本与失败分析；
- 预选 final artifact 通过 Validation 后执行一次 Sealed Test；若未通过则不解封；
- 输出 candidate tree、逐类副作用、所有 promotion verdict 和最终报告。

在两名工程师、AppWorld 数据可用且 BoxLite runner 就绪的前提下，总计约 5–7 周；若只有一名工程师或需要建设隔离 runner，应相应延长。

## 11. 完成定义

计划完成不是指某个候选分数变高，而是同时满足：

1. Skill、Prompt、Runtime 都有可执行 adapter、fixture、evaluator 和审批策略；
2. 任何候选都不能修改 trust root 或给自己打分；
3. Development、Validation、Sealed 的数据生命周期可审计且无 group 泄漏；
4. 晋升由 fresh Validation 和冻结规则决定，证据不足明确为 `inconclusive`；
5. Skill/Prompt/Runtime 的副作用分别被测量，不能共用一个“总分高就过”的门；
6. 每个 accepted 候选都有完整 diff/资产、实验、审批和 previous baseline；
7. 本地采用与回退可重放，但系统不包含生产部署假设。
