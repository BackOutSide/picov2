# Pico Evolution v2：以 Skill 为首个纵向切片的证据门控能力演进

> 配套材料：[业界调研与 Pico 当前实现审计](self-evolution-survey.zh-CN.md)。本文件是可直接进入 M0 与详细实现规划的 canonical 方法规范。

## Problem Anchor

- **Bottom-line problem**：把 `pico-harness` 目前仅面向 AppWorld、只会离线搜索少量 Runtime 文件的 Evolver，迭代为一套能够从真实运行经验中产生能力候选，并且可验证、可审批、可灰度、可激活、可回滚的通用自进化系统。
- **Must-solve bottleneck**：当前系统的失败诊断、候选生成、评测、SkillForge、Memory 与 activation 相互割裂；安全沙箱不成立，L1/coverage 没有阻断作用，promotion 统计门槛偏松，activation/rollback 只有账本状态，Skill/Prompt 等候选实际不受支持。
- **Non-goals**：不允许线上 Agent 任意修改自身核心代码；不做在线模型权重训练；不允许候选修改 evaluator、权限、安全策略、审计、依赖或 Evolver 自身；本阶段不处理根目录另一套 `pico==0.1.0` 的合并问题；不以一次性 benchmark 提分替代生产质量。
- **Constraints**：保持 Python 3.12 与现有 `pico-harness` Runtime；复用现有 Git tree、manifest、sealed test、journal 与 AppWorld adapter；Evolver 默认 opt-in；所有高风险改动必须脱离 live checkout；方案要能从小规模离线 replay 起步，不能依赖模型微调或新增 GPU 训练基础设施。
- **Success condition**：每一次能力变化都能回答“从哪些证据来、改了什么、在哪些任务有效、伤害了什么、由谁批准、当前在哪个环境生效、如何恢复”；P0 风险全部关闭；至少打通 Experience/Skill/Prompt/Runtime 四类资产中的前三类；真实 activation 与 rollback 有端到端测试；promotion 在独立验证集上满足统计、回归、安全和成本门槛。

## 1. Technical Gap and Delivery Boundary

当前已有：Git candidate/tree、manifest/G5、screen/confirm、sealed report、activation evidence、Tracing、`SkillRegistry`/BM25，以及 BoxLite `SandboxExecutor`。缺失的不是另一个搜索算法，而是四个可执行合同：

1. **可信结果合同**：谁有资格声明任务成功，哪些运行记录可以驱动学习。
2. **独立评测合同**：候选何时冻结、数据何时暴露、比较单位与停止规则是什么。
3. **可信执行合同**：候选代码在生成、预检、测试和评测的每条路径上拥有什么权限。
4. **环境发布合同**：批准的是哪个完整运行组合，发生并发或崩溃后如何恢复。

首个 release 只交付 instruction-only Skill 的完整闭环。Experience 是 Skill 的来源证据，不作为可直接部署的命令；Prompt 只预留一个结构化 slot，在 Skill 闭环通过后接入；Runtime candidate 保留现有离线搜索，但在可信执行完成前不得部署。

## 2. Method Thesis and Complexity Budget

**Thesis**：Pico Evolution v2 通过共享的能力身份、可信证据、fresh-holdout 评测和环境 Release 合同，把运行经验变为可审计 Skill；候选生成仍可由 LLM 完成，但成功判定、风险计算、评测分配和发布控制全部位于不可自修改的 trust root。

**复用**：

- `pico.evolver` 的 node tree、manifest、evidence、journal 与 sealed runner；
- `pico.sandbox.SandboxExecutor` / BoxLite；
- `SkillRegistry`、`LocalSkillCatalog`、BM25 与 watcher；
- Tracing span/artifact 与 session；
- 已有 atomic I/O、锁与 delete epoch primitive。

**首版新增**：

- `OutcomeReceipt` 与 `EvidenceReadiness`；
- `EvaluationPlan`、fresh-shard allocator 与 exposure ledger；
- Skill candidate adapter；
- `ReleaseSpec`、`DeploymentRecord`、`ActivationOperation` 与本地 reconciler。

**首版明确不做**：模型训练、通用 workflow search、自由代码自改、可复用 adaptive holdout、在线连续显著性检验、Pareto 搜索、bandit/GSME。

## 3. One Vertical Loop

```text
Runtime trace + trusted feedback/verifier
        │
        ▼
OutcomeReceipt ──► EvidenceReadiness
        │                 │
        │                 └─ not ready → collect / incident / reject
        ▼
Experience comparison → Skill candidate + hypothesis
        │
        ▼
Static policy + strict sandbox preflight
        │
        ▼ freeze artifact/control/metric/plan
Fresh promotion shard ──► trusted evaluator ──► PromotionDecision
        │
        ▼
Approval → DeploymentRecord(immutable ReleaseSpec) → CAS ActivationOperation
        │
        ▼
prepare/load/check → shadow → canary → active
        │
        └─ health/regression failure → reconciled rollback
```

候选生成器只读 development evidence；看不到未领取的 promotion/sealed 数据。Promotion 结果可以影响下一个候选，但下一个候选必须领取从未用于任何设计或决策的新 shard。

## 4. Trust and Permission Model

### 4.1 Three identities

| Identity | Can do | Cannot do |
|---|---|---|
| Candidate process | 读取公开任务输入；在限定目录生成 Skill；调用白名单工具 | 读取 grader/答案/宿主凭据；写 promotion ledger；声明 score；修改 R4 |
| Trusted evaluator | 调度两臂 release、读取 oracle、收集结果、签发 measurement | 修改 candidate；放宽 policy；发布 release |
| Trusted controller | 计算风险、验证证据/审批、CAS 发布、reconcile/rollback | 生成候选内容；自行改变 evaluator 结果 |

R4 trust root 包含 evaluator、holdout registry、sandbox policy、权限、secret、audit、risk policy、activation controller 和 Evolver 自身；候选路径规则只是附加防线，不是 R4 的根本隔离。

### 4.2 Risk is computed, not declared

候选只能声明 `requested_risk`；可信 policy 根据内容、可执行附件、依赖、工具权限、scope 和部署环境计算 `effective_risk`。首版 Skill 必须满足：

- 仅 Markdown 指令，无脚本、二进制或新增依赖；
- 不新增或扩大工具/网络/文件权限；
- `owner_id`、`tenant_id`、`workspace_id` 必填；
- managed Skill 永远不能覆盖同名 workspace 用户 Skill；
- 跨 workspace/tenant 默认拒绝。

含可执行附件或权限语义的 Skill 自动升级到 R3，并在首版标记 unsupported。

## 5. Four Core Contracts

### 5.1 CapabilityArtifact

```yaml
schema_version: 1
artifact_id: cap_skill_<sha256>
kind: skill
owner_id: required
tenant_id: required
workspace_id: required
base_release_spec_digest: sha256
parent_artifact_ids: []
content_ref: cas://sha256/...
content_digest: sha256
source_receipt_ids: []
hypothesis:
  failure_mode: ...
  mechanism: ...
  expected_rescue: ...
  possible_harm: ...
requested_risk: R2
effective_risk: R2          # controller writes
permissions_delta: none     # controller verifies
generator_identity: ...
generator_prompt_digest: ...
data_classification: internal
retention_policy_id: ...
source_revision: ...
deletion_epoch: 0
status: draft
```

`parent_artifact_ids` 表示 lineage，不表示运行时兼容性；兼容性由 `base_release_spec_digest` 和完整 evaluated execution spec 决定。

### 5.2 EvaluationPlan

在领取数据前冻结：

```yaml
plan_id: evalplan_<sha256>
campaign_id: ...
candidate_artifact_digest: ...
control_release_spec_digest: ...
challenger_release_spec_digest: ...
control_skill_view_digest: ...
challenger_skill_view_digest: ...
task_family_weights: {...}
attempts_per_task: 1
seed_policy: paired-fixed
reset_policy: fresh-environment-per-arm
arm_order_policy: randomized
primary_metric: macro_task_success_lift
delta_gate: 0.05            # 必须证明显著超过的最小收益
delta_alt: 0.10             # power 计算采用的目标真实收益
alpha_fraction: 1/40         # exact rational from campaign alpha ledger
power_target: 0.80
confidence_method: paired-discordance-cp-bonferroni-v1
planned_tasks: from-pilot
budget_cap_tasks: 300
minimum_tasks_per_family: 20
family_noninferiority_margin: -0.03
cost_cap_ratio: 1.10
latency_p95_cap_ratio: 1.15
missing_policy: fail_task
infra_retry_limit: 1
claim_policy: external-immutable-claim-record
```

M0 pilot 必须用同一 gate 方法估计 discordance 并计算 `delta_alt` 下的 power；`delta_gate` 边界本身不宣称 80% power。若 `planned_tasks > budget_cap_tasks`，campaign 在领取 shard 前即标记 infeasible；执行后样本不足则 verdict 为 `inconclusive`，不能降低门槛。首版 `K=1`，独立单位是 task，不把同一 task 多次 attempt 当独立样本。

### 5.3 EvidenceBundle / PromotionDecision

Bundle 绑定：plan digest、完整两臂 `release_spec_digest`、holdout digest、raw paired measurements、evaluator/build identity、sandbox attestation、runtime/model/tool/config fingerprint、成本、统计方法和 exposure record。

可信 `sandbox_attestation` 至少包含：backend 与实现 digest、image digest、execution nonce、mount 列表、网络策略、环境变量 allowlist、资源上限、开始/结束时间、清理结果和 supervisor identity；candidate 无权写入。

Promotion 条件全部满足才 accepted：

1. fresh shard 成功原子领取且此前未暴露；
2. 主指标的一侧置信下界超过预注册 `delta_gate`；
3. 每个关键 family 满足 `-3pp` 非劣界；
4. 确定性安全不变量零违规；
5. cost 与 p95 latency 不超过绝对 cap；
6. evidence、sandbox、scope、base `ReleaseSpec` 与两臂 `skill_view_digest` 绑定完整。

`critical regression=0` 仅用于确定性不变量；随机质量子组使用非劣检验。安全不是 Pareto 维度，而是硬门。

PromotionDecision 只裁决候选证据是否合格，不等待审批。Approval 在后续 `ActivationEligibility` 中绑定 artifact、evidence、environment 和 policy；两者分离，避免 accepted 与 approval 互相等待。

### 5.4 ReleaseSpec / DeploymentRecord / ActivationOperation

被评测的执行内容与部署证明必须分开，避免“release digest 依赖 evidence，而 evidence 又依赖 release digest”的循环。

`ReleaseSpec` 是评测前即可确定的 immutable 执行规格，一次只改变一个 artifact：

```yaml
schema_version: 1
runtime_git_sha: ...
prompt_profile_digest: ...
managed_skill_set_digest: ...
policy_digest: ...
execution_config_digest: ...
```

`release_spec_digest = sha256(canonical_json(ReleaseSpec))`。Canonical form 固定为 schema v1：UTF-8、字段按字节序排序、无无意义空白、禁止 float/NaN、所有 digest 使用小写十六进制；实现复用项目 canonical JSON primitive，并以 golden vectors 固定跨平台结果。`ReleaseSpec` 不包含 evidence、approval、environment、generation、operation 或 health 状态，所以可在评测前计算，且部署多少次都不会改变被评测身份。

`DeploymentRecord` 才绑定证明与环境：

```yaml
deployment_id: uuid
environment_id: local-dev
generation: 18
release_spec_digest: ...
evidence_bundle_digest: ...
approval_id: ...
approval_policy_digest: ...
previous_release_spec_digest: ...
created_at: ...
```

Operation：

```yaml
operation_id: uuid
environment_id: local-dev
expected_generation: 17
desired_release_spec_digest: ...
observed_release_spec_digest: ...
previous_release_spec_digest: ...
phase: requested | prepared | checked | shadow | canary | switched | healthy | failed | rolled_back
health_evidence_ref: ...
attempt: 1
last_error: null
```

本地首版使用 SQLite 事务和 generation CAS 保存 desired/observed 状态，content-addressed 目录保存 immutable artifact；reconciler 是唯一能产生部署 side effect 的主体。每个 session 在开始时 pin `SessionCapabilitySnapshot`，turn trace 同时记录 `release_spec_digest` 与 `skill_view_digest`。过期 activate/rollback 因 `expected_generation` 不匹配而失败，避免旧 rollback 覆盖新发布。

## 6. Outcome and Data Lifecycle

现有 TraceStore 继续承担可观测性；只有可信 verifier、明确用户反馈接口或环境收据才能生成可驱动学习的 `OutcomeReceipt`。Agent 自评、普通 final 文本和“Skill 被注入后成功”都不能自动成为正/负标签。

Receipt 元数据 append-only：

```yaml
receipt_id: ...
trace_id: ...
release_spec_digest: ...
skill_view_digest: ...
verifier_identity: ...
claim_type: task_success | explicit_preference | correction | safety_violation
verdict: success | failure | inconclusive | infra_error
payload_ref: protected://...
data_classification: ...
retention_policy_id: ...
redaction_version: ...
source_revision: ...
deletion_epoch: 0
```

敏感输入、trajectory 和反馈正文存入按 scope 控制且可删除的 payload store；审计日志只留 digest、分类、时间和 tombstone。来源被删除时：增加 deletion epoch → suspend 所有依赖 artifact/release → 清理 payload 与检索缓存 → 保留不含原文的不可用 tombstone。发送给 LLM 的只能是已去敏视图。

证据优先级按命题定义：任务成功优先确定性 verifier/环境结果；用户偏好只接受经过身份验证的用户明确声明；两类证据不能互相替代。

## 7. Evidence Readiness and Diagnosis

当前 WHY-class 数量仅作为诊断覆盖报告，不再阻断所有候选。每个候选的 readiness 要求：

- 至少一个可复现 failure receipt；
- 至少一个同 family 的成功对照或可信反例；
- hypothesis 引用具体 trace/span；
- verifier 能覆盖预期 rescue 与 possible harm；
- 来源未被删除且 scope 一致；
- 无未恢复的、会污染该时间窗/环境的 infra incident。

Infra incident 必须有 `incident_id`、受影响 environment/time/scope、可信确认、recovery probe 和 resolved_at。候选导致的 timeout、资源耗尽或工具异常属于 candidate failure，不能被重新标为 infra 后排除。L1/infra 命中时只阻断受污染窗口，不消耗搜索 patience；恢复 probe 成功后才能继续。

LLM 仅在确定性归因不足时生成带 evidence span 的语义 hypothesis；它不能签发 receipt 或 gate verdict。

## 8. Skill Candidate and Existing Registry Integration

1. 在 development receipts 中选同 task family 的成功/失败对照。
2. 生成最小 Skill delta、适用条件、反例、预计伤害和开发 fixture。
3. 生成器 fixture 只用于 preflight，不得作为 promotion oracle。
4. 可信 preflight 检查 frontmatter、指令注入模式、secret/URL/权限、最大长度、依赖与 retrieval reachability。
5. Promotion 任务和标签来自冻结的可信 task registry。
6. Accepted Skill 落到 managed content-addressed source，而不是直接写 `workspace/skills`。

`LocalSkillCatalog` 增加 managed evolution source：优先级低于 workspace 与 operator source、高于 builtin。Draft/rejected/revoked 版本永远不进入可用视图。

为了避免 release 17 的 session 在共享 BM25 刷新后读到 release 18，session 启动时创建 immutable `SessionCapabilitySnapshot`：

```yaml
release_spec_digest: ...
skill_view_digest: ...
sources:
  workspace_snapshot_digest: ...
  operator_snapshot_digest: ...
  managed_skill_set_digest: ...
  builtin_snapshot_digest: ...
revocation_epoch: 7
```

检索与读取接口必须接收 `skill_view_digest`，缓存 key 至少为 `(skill_view_digest, query)` 或 `(skill_view_digest, source, name)`；不能只按 source/name。Session start 对 workspace/operator/builtin/managed 的 winner 与 content digest 做 snapshot，之后普通文件 watcher 只使**新 session**生成新 view，不原地改变旧 view。每个 view 使用不可变 catalog/index；有 session pin 时保留，refcount 归零后再 GC。

普通 rollback 只改变新 session 的 desired spec，已有 session 按原 snapshot 完成。强制 revoke（安全事件、来源删除、审批撤销）提高 revocation epoch，并在每次 turn/tool 边界检查；依赖被撤销 digest 的旧 session 必须显式中止或经用户可见的 restart 进入新 snapshot，不能静默混合升级。来源删除会撤销所有包含该 source revision 的 snapshot 与缓存。每次检索、注入、读取分别记录事件；它们只用于可观测性，不自动推断因果贡献。

Evaluation evidence 同时记录两臂的 `skill_view_digest`。生产 session 若因 workspace/operator Skill 变化获得不同 view，只能视为新的运行上下文；原 evidence 仍证明已评测 view 上的结果，不能外推为新组合已被验证。Policy 可允许其运行，但必须在 trace 中显式标记 `context_changed_since_evaluation`。

## 9. Fresh-Data Statistical Protocol

### 9.1 Dataset lifecycle

- `development`：允许查看逐题轨迹，用于诊断和候选生成。
- `promotion reservoir`：预先登记但不暴露；按 `episode_group_id/template_group_id` 分组后分 shard，任何近重复 group 只能属于一个 split；allocator 为冻结的 plan 原子领取一个 fresh shard。
- `sealed audit`：只做最终泛化报告，使用一次后永久 retire；换 run ID 不会恢复独立性。

`exposure_ledger` 跨 run 记录 dataset/shard 的 `unseen → claimed → evaluated → exposed → retired`。Allocator 只从每个独立 episode/template group 抽一个 task，`n_f` 只计这些独立配对；跨 split 与同一 shard 内都不得重复 group。Claim 与 alpha allocation 在一个事务中生成独立 immutable `ClaimRecord(plan_digest, shard_digest, alpha_fraction)`；不得把原 EvaluationPlan 的 `holdout_id` 原地改写。Claim 后无论进程崩溃、部分执行还是 infra error，shard 与该次 alpha 都不退回，只能 retire。任何已暴露 shard 只能降级为 development 数据。

### 9.2 Adaptive campaigns

一个 campaign 每次只提交一个 challenger，默认最多 5 次。第 i 次检验使用新 shard，并由 campaign ledger 用整数 numerator/denominator 精确记录 `alpha_i = 0.05 / (i(i+1))`；因为数据对当前及未来 candidate 均未使用，条件检验才有效。首版不需要 Holm。候选、control、metric、`delta_gate`、`delta_alt`、样本上限和停止规则在领取 shard 前冻结；统计库的 Decimal 转换规则版本化，浮点近似不能改变已分配 alpha。

Control/challenger 以 task 为单位随机顺序、相同模型/权限/预算/seed policy 运行；每个 arm 使用干净环境，避免缓存和顺序污染。“same session”只是一种配对方式，不替代 reset/randomization。

首版主指标是配对二值 task success，使用保守的 `paired-discordance-cp-bonferroni-v1`。对每个 family，令 `b_f` 为 challenger-only success、`c_f` 为 control-only success、`n_f` 为总 task 数；若共有 F 个预注册 family，则 `alpha_family = alpha_i / F`，分别用错误预算 `alpha_family/2` 的单侧 Clopper–Pearson 得到 `p(b_f)/n_f` 的下界与 `p(c_f)/n_f` 的上界，形成 `L_f = lower_CP(b_f) - upper_CP(c_f)`。Bonferroni 保证所有 family 的该组下界以至少 `1-alpha_i` 同时覆盖；零 discordance 仍产生保守而非虚假的确定界。

Allocator 验证所有 `w_f ≥ 0` 且 `Σw_f = 1`。预注册权重下的全局下界为 `L_global = Σ w_f L_f`。Promotion 要求 `L_global > delta_gate`，且每个关键 family 的 `L_f > family_noninferiority_margin`。M0 power simulation 使用完全相同的 interval/gate，并在 `delta_alt`（首个 proposal 为 10pp，而非 5pp 边界）下要求 80% power。分层 paired bootstrap 与 McNemar 仅作探索性报告，不参与晋升。实现测试必须覆盖 `b=0`、`c=0`、全成功和极小 alpha；CP API 接收 `(count, n, tail_budget)`，不能对比例再次除以 n。

Synthetic validation 覆盖 `Δ=0`、`Δ=delta_gate`、`Δ=delta_alt`、稀疏 discordance、family 不均衡、近重复分组、最多 5 次自适应检验、rational alpha 分配、claim 后 crash 和 infra retry。若 family 样本不足以证明非劣，结果必须 inconclusive。

### 9.3 Production validation

首版 shadow 仅 replay、stub 或隔离账户，禁止不可逆副作用。Canary 只进入已列入 allowlist 的可逆/只读 task family；采用预注册固定样本和固定观察窗，不连续窥视显著性。外部邮件、付款等已发生副作用无法被版本 rollback 撤销，必须由业务补偿流程处理。

## 10. Trusted Execution on Existing Sandbox

不新建第二套 executor；为 `pico.sandbox.SandboxExecutor` 增加 `evolution-strict` profile，要求 `is_sandboxed=True`，首版 backend 为 BoxLite，`none` 永远不合格。M0 必须在目标 Windows 开发环境和实际 Linux/KVM runner 上 probe；不支持的平台把评测移到受控 runner，不能 host fallback。

四条路径全部经严格 profile：

1. candidate editing/generation；
2. import/lint/unit/preflight；
3. smoke/replay；
4. promotion/sealed evaluation。

生成环境：只挂载候选 worktree RW，源基线/策略 RO，默认禁网，空环境变量，资源受限。评测环境：候选 release RO、任务输入按例注入；grader、oracle、完整 holdout、ledger、审批凭据不挂载。可信 evaluator 在隔离外收集 supervisor receipt；候选自报 JSON 不可直接成为 measurement。每个 task 使用新 execution nonce 并清理残留进程。

Threat suite 覆盖路径逃逸、宿主/凭据读取、答案探测、grader/result 伪造、网络/DNS 绕过、fork bomb/资源耗尽、残留进程、artifact symlink 与候选修改测试环境。

## 11. Activation, Recovery and Rollback

Reconciler 流程：

1. 验证 artifact、`ReleaseSpec`、evaluation、approval、base generation 与 scope；
2. SQLite `BEGIN IMMEDIATE`，以 expected generation CAS 写 operation intent；
3. materialize desired `ReleaseSpec` 及其 immutable managed catalog/index 到 staging，校验所有 digest；
4. 在无生产流量的 slot load 并执行 health/readiness；
5. 进入 shadow；通过预注册 gate 后生成 canary operation；
6. 路由按 session 固定 `SessionCapabilitySnapshot`，worker 在 catalog/index 就绪后 ACK observed generation；
7. 达到固定样本/观察窗后切 active；
8. 写 observed release 与 health evidence，operation 完成。

进程在任一边界崩溃时，重启后的 reconciler依据 operation phase 幂等继续或恢复 previous spec；未完成 catalog/index build 或 worker ACK 时禁止切流量。Rollback 也是新 operation，必须 CAS 当前 generation；它只保证后续新 session 恢复旧能力版本，不承诺撤销外部业务副作用。安全 revoke 另走 revocation epoch，在下一 turn/tool boundary 阻断仍 pin 被撤销 snapshot 的 session。

状态和 side effect 不再由 `set_activation_state()` 的字符串迁移代表。现有 immutable activation bundle 保留为输入证据；新增 controller/reconciler 承担真实部署。

## 12. Source Mapping

| Existing code | Revision |
|---|---|
| `pico/evolver/candidate_manifest.py` | 先为 instruction-only Skill 接上 fixture/evaluator；risk 由 policy 计算 |
| `pico/evolver/orchestrator/loop.py` | 强制 L1 incident scope、EvidenceReadiness、fresh EvaluationPlan；zero outcome 不耗 patience |
| `pico/evolver/orchestrator/gates/*` | 新增 campaign/exposure ledger、fresh paired analysis；旧 frozen 仅 legacy |
| `pico/sandbox/*` | 复用接口与 BoxLite，实现 strict evolution profile/attestation；禁止 DirectExecutor |
| `pico/evolver/activation/*` | bundle 继续只读证据；新增 release store、operation、reconciler |
| `pico/memory_engine/skill_local/*` | 接入 managed source、release generation、visibility filter 与 cache refresh |
| `pico/tracing/*` | span 继续 best-effort；新增可信 OutcomeReceipt 引用 trace，关键收据写失败则 fail closed |
| `benchmarks/appworld/evolve/*` | 所有 subprocess 路径改走 strict executor；Runtime deployment 延后 |

## 13. Failure Handling

- **Adaptive overfit**：fresh shard、跨 run exposure ledger、one-candidate-at-a-time、sealed retirement。
- **Evaluator/reward hacking**：oracle 与 evaluator 不在候选环境，measurement 由 supervisor 签发。
- **Prompt/Skill permission escalation**：可信 risk policy、instruction-only 首版、permissions delta 必须为 none。
- **Persistent poisoning**：严格 source identity、payload redaction、TTL/deletion epoch、依赖 suspension。
- **Audit loss**：关键 receipt/evaluation/release 写失败立即 blocked；普通 UI trace 可 best-effort。
- **Concurrent publish**：generation CAS；旧审批/rollback 无法覆盖新 release。
- **Cache drift**：registry generation 和 release digest 写入每个 turn；不一致停止新流量并 reconcile。
- **Combination regression**：首版每个 `ReleaseSpec` 只改一个 artifact；base spec 改变必须重评。

## 14. Claim-Driven Validation

### Claim 1：可信执行与发布合同能在定义的 threat/fault matrix 中 fail closed 并恢复

- **Experiment**：对 edit/import/test/eval 四路径运行安全负例；对 activation 每个 phase 注入 crash、重复请求、并发 activate、过期 rollback、worker 不 ACK 和 cache stale。
- **Baseline**：当前 host AppWorld sandbox 与 metadata-only activation。
- **Metric**：未授权资源访问次数、候选自报分被接受次数、错误 release 接流量次数、最终 desired/observed 一致性、恢复时长。
- **Acceptance**：所列攻击集零越界成功、零伪造 measurement 被接受；所有故障在 60 秒内或下一次 reconciler 周期恢复为 previous/desired 的合法状态，且无旧 generation 覆盖新 generation。

### Claim 2：fresh-shard promotion 在自适应候选生成下控制假晋升

- **Experiment**：synthetic generator 根据历次 verdict 主动过拟合；比较 current frozen/lift>0、重复 holdout＋alpha、fresh grouped shard＋conservative paired bound；覆盖零/边界/目标收益、稀疏 discordance 和不均衡 family。
- **Metric**：10,000 campaign Monte Carlo 的 FWER 与 95% Monte Carlo CI、在 `delta_alt` 下的 power、margin coverage、winner's curse、任务/调用成本。
- **Acceptance**：`Δ≤delta_gate` 时完整协议的错误晋升率置信区间上界不超过 campaign alpha 加预注册 Monte Carlo tolerance；`Δ=delta_alt` 时 power 达 0.80。任一条件未满足都先修统计实现或样本计划，不放宽 gate。

### Claim 3：Experience 对照产生的 Skill 在可信 held-out 上提升且可撤销

- **Experiment**：选择一个已有 verifier 的 task family；手工 Skill、failure-only reflection、success/failure contrast Skill 做冻结候选比较；再做 managed-source activate/rollback。
- **Metric**：task-level success lift、family retention、错误检索率、token/cost/latency、release spec 与 session skill view 一致性。
- **Acceptance**：主指标满足 EvaluationPlan；关键 family 非劣；错误检索率不高于预注册 cap；并行 pin 旧/新 snapshot 的 session 不混用 Skill，rollback 后新 session 恢复旧 spec，强制 revoke 后旧 session 在下一边界被阻断。

## 15. Dependency-Ordered Roadmap

### M0 — Reproducible baseline and boundary（3–5 天）

- 独立 repo/解释器/包入口与 Python 3.12 测试环境；
- 在目标 Windows 与受控 Linux runner probe BoxLite；
- 选择一个真实 task family、可信 verifier 和 development/promotion 数据来源；
- 运行 pilot，填写 δ、power、样本量、成本/延迟 cap；
- 冻结 threat model、数据保留和 scope policy。

**Exit**：相同 release 可重复跑；executor 无 host fallback；promotion 数据可分配 fresh shard。

### M1 — Minimal evidence and trusted execution（2 周）

- OutcomeReceipt/EvidenceReadiness；
- strict execution profile 覆盖 edit/import/test/eval；
- campaign/exposure ledger 和 EvaluationPlan；
- 修复 L1 scope、zero-outcome patience、sealed reuse 与关键 audit fail-open。

**Exit**：安全 threat suite 通过；自适应 synthetic FWER study 达标。

### M2 — Skill vertical slice and real release（2–3 周）

- 先允许人工提供 Skill candidate；
- instruction-only manifest/evaluator；
- managed registry source；
- ReleaseSpec/DeploymentRecord、SQLite CAS、reconciler、真实 activate/rollback；
- crash/concurrency matrix。

**Exit**：一个 Skill 从 draft 到 active 再 rollback；状态、registry、turn digest 始终一致。

### M3 — Experience-to-Skill generation, then one Prompt slot（2–3 周）

- 成功/失败对照与引用式 hypothesis；
- 去敏输入、TTL/deletion dependency；
- LLM Skill proposal；
- Skill 稳定后接入一个结构化 Prompt slot，复用同一评测/Release 合同。

**Exit**：真实 task family 产生 held-out 提升；删除来源能 suspend 依赖；Prompt 不新增第二套生命周期。

### M4 — Bounded shadow and canary（2 周）

- replay/stub shadow；
- 只读/可逆 task allowlist canary；
- 固定样本/观察窗、自动 rollback 和 operator runbook。

**Exit**：故障可在预设 60 秒检测目标内停止新流量；rollback 后新 session 使用旧 release。

### Later — Runtime deployment and extra adapters

只有 M0–M4 完成后，才将 Runtime SHA 纳入 production Release，并考虑 Route/Workflow 等 adapter。Runtime candidate 的安全执行修复在 M1 完成，但其生产部署不是 Skill 首版的前置目标。

## 16. Compute, Data and Timeline

- 不训练模型，不需要 GPU；成本来自候选生成、paired replay 和 fresh promotion 数据。
- 需要一个可去敏、可重放、带可信 verifier 的 task family；如果没有，M0 终止并先建设 evaluator。
- 2 名工程师、已有 Linux/KVM runner、task data 与 verifier 就绪时，M0–M4 约 9–11 周；不包含生产流量接入审批、额外数据标注或 Windows 本机不支持 BoxLite 后的 runner 建设。
- 每个 campaign 的评测预算由 M0 pilot 决定；不得以更换更强模型、扩大权限或增加不可比预算计为进化收益。

## Experiment Handoff Inputs

- **Must-prove claims**：全执行路径隔离；fresh data 下假晋升受控；真实 Skill release 可激活、固定、恢复。
- **Must-run ablations**：reused vs fresh holdout；host/path-only vs strict executor；metadata state vs CAS reconciler；failure-only vs success/failure contrast。
- **Critical data/metrics**：安全 attack corpus、crash matrix、自适应 synthetic campaigns、一个有可信 verifier 的真实 task family；FWER、power、task success、retention、cost、latency、RTO。
- **Highest-risk assumptions**：promotion reservoir 可持续提供 fresh tasks；目标平台有可验证的隔离 runner；task verifier 的误判率足够低；本地 SQLite/CAS 足以覆盖首版单机部署。
