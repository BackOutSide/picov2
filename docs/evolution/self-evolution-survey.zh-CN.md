# Pico 自进化：业界调研与当前实现审计

> 调研日期：2026-10-04
>
> 审计范围：`pico-harness`（现已独立上传至 `BackOutSide/picov2`）
>
> 目的：回答“成熟系统实际怎样持续改进”“Pico 现在真正做到哪一步”，并为 [Evolution v2 迭代方法](self-evolution-v2.zh-CN.md) 提供依据。

## 1. 结论先行

截至当前，市面上并不存在一个已经证明能在开放生产环境中长期、安全、无回归地自主修改全部 Agent 能力的通用系统。成熟的是若干局部机制：

1. 任务内的 `执行 → 测试/批评 → 重试`；
2. 跨任务持久记忆和 Skill/规则加载；
3. 基于 dataset、grader 和 trace 的离线 Prompt/Skill 优化；
4. 人工审批、版本化发布、线上监控和回滚。

自动搜索 workflow、修改 harness 代码或在线更新权重仍主要属于研究或高风险实验。Pico 因此不应把“多反思几轮”“写入 memory”直接称为完整自进化，而应将自进化定义为：

> 从可信运行证据产生版本化能力候选，经独立数据和安全约束验证，再通过可恢复发布进入后续任务。

Pico 当前的 Evolver 已有不错的离线搜索骨架，但真实边界是：**只支持 AppWorld、只允许修改两个 Runtime 文件、没有有效 OS 隔离、没有真实 activation/rollback；Skill Forge 只有检索注入，没有技能学习闭环。**

## 2. 先统一术语

| 层级 | 改变的对象 | 是否跨任务保留 | 例子 | 是否应称长期自进化 |
|---|---|---:|---|---:|
| 单次反思 | 当前回答/轨迹 | 否 | critic/refiner、测试失败后重试 | 否 |
| Memory | 偏好、事实、经验文本 | 是 | auto memory、episodic memory | 部分 |
| Prompt/Skill | 长期指令、示例、Skill | 是 | prompt optimizer、Skill 版本 | 是，若有独立 gate |
| Workflow | 节点、角色、路由、工具组合 | 是 | ADAS、AFlow | 是，风险较高 |
| Harness code | Agent 实现代码 | 是 | DGM、SICA、Gödel Agent | 是，研究级高风险 |
| Model weights | 模型参数 | 是 | fine-tuning、RFT/RL | 是，但应是独立离线训练流程 |

判断一个系统是否形成真正闭环，至少检查：

- 是否有可持久化、可版本化的能力载体；
- 是否有环境、测试、用户或独立 evaluator 的反馈；
- 是否存在候选、对照评测、选择和发布；
- 改进是否作用于之后的任务；
- 是否在未用于生成候选的数据上证明收益；
- 是否可撤销、回滚并审计。

## 3. 市面上相对成熟的做法

### 3.1 成熟度矩阵

| 系统 | 真正做了什么 | 没有做什么 | 成熟度 | Pico 应借鉴 |
|---|---|---|---|---|
| OpenAI Agents / Evals | Trace、grader、dataset、eval run、Prompt 优化；任务内工具反馈循环 | 不会自动安全上线所有候选；权重更新是独立流程 | 运行/评测高 | trace grading、可重复 dataset、grader 优先 |
| Claude Code | Hooks、Skills、跨 session auto memory、Skill baseline/eval | memory 是 context，不是强制 policy；Skill 主要由人维护 | 高 | 软经验与强制 hook/policy 分离 |
| Google ADK / Agent Engine | 确定性 LoopAgent、session/memory、CLI/pytest eval、sandbox code execution | 没有内建通用 prompt/workflow 自优化器 | 高 | 固定控制流、CI eval、沙箱执行 |
| LangGraph / LangSmith / LangMem | Durable execution、offline/online eval、生产失败回流 dataset、memory/prompt optimizer | 应用仍需自建 holdout、发布、权限和回滚 | 编排/评测高，优化中 | 生产轨迹到回归集的飞轮、最小 Prompt 修改 |
| DSPy / GEPA | 根据 metric、轨迹和文本反馈离线搜索 Prompt/程序 | 不负责生产权限、canary、rollback | 离线优化中高 | 候选档案、独立 validation、预算化搜索 |
| CrewAI | 自动 memory；把人工反馈蒸馏成未来 task prompt suggestions | 所谓 Training 不改权重，也不修改 Python/YAML | 中 | 明确 human feedback 的作用域和来源 |
| OpenHands / Aider | 代码执行、测试/工具反馈、事件流、repo Skill/规则 | 没有公开的自动改 Skill 并安全推广闭环 | Coding loop 高 | sandbox、event stream、benchmark harness |
| EvoAgentX / ADAS / DGM | Prompt/workflow/code 候选搜索与 archive | 仍依赖 benchmark，执行生成代码风险高 | 研究/早期框架 | 只在完全隔离的研究面借鉴 archive/search |

### 3.2 代表机制与一手证据

**OpenAI** 的成熟部分是 trace、grader、dataset 与重复 eval。官方 Agent eval 指南把 trace grading 用来定位工具、handoff、guardrail 和 routing 问题，再把问题固化到 dataset 中比较版本；这是一条人工监督的质量飞轮，而不是 Agent 自行宣告进步。[Agent evals](https://developers.openai.com/api/docs/guides/agent-evals)、[Trace grading](https://developers.openai.com/api/docs/guides/trace-grading)

OpenAI 的 dataset-backed Prompt Optimizer 能读取 grader 和人工 annotation 产生候选 Prompt，但官方已宣布 Evals 于 2026-10-31 只读、2026-11-30 下线。因此它适合参考机制，不适合作为 Pico 的核心依赖。[Prompt Optimizer](https://developers.openai.com/api/docs/guides/prompt-optimizer)

**Claude Code** 将三类能力明确分开：Hooks 是确定性生命周期控制；Skills 是按需加载的说明与资源；auto memory 是 Claude 可更新的跨 session context。这个划分很重要：必须执行的安全和测试规则不能只写进可被模型改写的 memory。[Hooks](https://code.claude.com/docs/en/hooks)、[Skills](https://code.claude.com/docs/en/skills)、[Memory](https://code.claude.com/docs/en/memory)

**Google ADK** 的 LoopAgent 是固定控制流，适合 draft→critic→refiner；evaluation 支持 CLI、pytest、response/trajectory matching；MemoryService 需要显式写入，不是部署后自动学习。这代表成熟产品的共同选择：运行和评测基础设施优先，自动优化保持受控。[LoopAgent](https://adk.dev/agents/workflow-agents/loop-agents/)、[ADK evaluation](https://adk.dev/evaluate/)、[ADK memory](https://adk.dev/sessions/memory/)

**LangSmith/LangMem** 分别覆盖 offline/online eval 与从 trajectory/feedback 修改 Prompt。LangSmith 建议把生产失败加入离线 dataset；LangMem 提供 gradient、metaprompt 和 prompt-memory optimizer。但版本晋升、独立 holdout、权限与回滚仍由应用负责。[LangSmith evaluation](https://docs.langchain.com/langsmith/evaluation-types)、[LangMem optimizer](https://github.com/langchain-ai/langmem/blob/main/src/langmem/prompts/optimization.py)

**DSPy** 把优化明确表示为 `program + metric + training inputs`。MIPROv2 搜索 instructions/demos，GEPA 根据轨迹与文本反馈反思并维护候选；官方同样建议独立 validation 防止过拟合。这是 Pico Prompt/Skill candidate generator 可参考的形态，但不能替代 promotion 和发布控制。[DSPy optimizers](https://dspy.ai/3.1.0/learn/optimization/optimizers/)、[GEPA tutorial](https://dspy.ai/3.0.1/tutorials/gepa_ai_program/)

**Letta** 展示了 Agent 自主管理长期 context 的较完整产品形态：Git-backed memory、显式 `/remember`、后台 dreaming 和二次 Agent review。但它主要解决“记什么、如何组织”，不是用 held-out task 证明策略提升。[Letta memory](https://github.com/letta-ai/letta-docs-md/blob/main/configuration/memory/index.md)

**EvoAgentX、ADAS 与 Darwin Gödel Machine** 说明 Prompt、workflow 乃至代码搜索在技术上可行；同时也暴露了 benchmark-directed hill climbing、成本、生成代码执行和 reward hacking 风险。它们应作为隔离研究面，而不是 Pico 近期生产架构。[EvoAgentX](https://github.com/ANative-Lab/EvoAgentX)、[ADAS](https://github.com/ShengranHu/ADAS)、[DGM](https://github.com/jennyzzt/dgm)

### 3.3 学术路线提供的关键经验

- **Reflexion**：轨迹→evaluator→文字经验→重试，证明外部反馈比纯自省可靠；但多为任务内经验。[论文](https://papers.nips.cc/paper_files/paper/2023/hash/1b44b878bb782e6954cd888628510e90-Abstract-Conference.html)
- **ExpeL**：比较多条成功/失败轨迹，抽取跨任务 insight，再按新任务检索；比“只总结失败”更适合 Pico。[论文](https://ojs.aaai.org/index.php/AAAI/article/view/29936)
- **Voyager**：可执行 Skill、环境反馈、自验证、技能库检索与组合；说明 Skill 必须可测试、可复用，而不是一段聊天摘要。[论文](https://arxiv.org/abs/2305.16291)
- **OPRO / Promptbreeder / GEPA**：把 Prompt 视为黑盒候选，外部 metric 选择；主要风险是固定 benchmark 过拟合与调用成本。
- **ADAS / Gödel Agent / DGM**：把 workflow 或 harness code 纳入搜索；必须有不可变 evaluator、安全边界、版本树和回滚。[ADAS 论文](https://proceedings.iclr.cc/paper_files/paper/2025/file/36b7acf6f6010652b3f2a433774a66fe-Paper-Conference.pdf)、[Gödel Agent](https://aclanthology.org/2025.acl-long.1354/)
- 没有可靠外部反馈时，LLM 自我纠错可能无效甚至降级；因此 evaluator 是可信边界，不是附属模块。[ICLR 2024 反例](https://openreview.net/forum?id=IkmD3fKBPQ)

## 4. Pico 当前的自进化到底如何工作

### 4.1 当前 Evolver 调用链

```text
pico evolve
  → pico/cli/evolve_commands.py
  → pico/evolver/cli.py
  → pico/evolver/launch/runner.py
  → AppWorld BenchBundle
  → cold start
  → EvolutionOrchestrator 多轮诊断/设计/应用/评测
  → sealed evaluation
  → retention report + activation evidence bundle
```

它是**离线、benchmark-gated 的 Git 候选搜索**，不是运行中的 Agent 在线修改自己：

- 候选在 detached Git worktree 生成并保存为 child commit；
- 当前 registry 只注册 AppWorld；
- `CandidateManifest` 虽定义 Skill、Prompt、Policy、Runtime、Model Profile、Route，但除 Runtime 外均 `supported=False`；
- Runtime 只允许修改 `benchmarks/appworld/agent_cli.py` 和 `benchmarks/appworld/tool.py`。

源码证据：

- `pico/evolver/candidate_manifest.py:78-163`
- `pico/evolver/launch/registry.py:15-17`
- `pico/evolver/orchestrator/loop.py:391-646`
- `benchmarks/appworld/evolve/entry.py:121-225,257-350`
- `benchmarks/appworld/evolve/eval.py:198-213`

### 4.2 已有价值

当前实现并非从零开始，已经有值得保留的部分：

- train/test overlap 拒绝；
- Gate0、fixed denominator、inconclusive fail-closed；
- immutable path 与 G5 manifest/digest/Git binding；
- sealed evaluation 只在搜索终止后读取；
- node tree、archive、round journal、candidate history 和 evidence bundle；
- `pico.sandbox.SandboxExecutor` 与 BoxLite 后端；
- `SkillRegistry`、多来源优先级、BM25、watcher 与缓存刷新；
- Tracing、Session、usage、tool span 与 skill read span。

这意味着 v2 应优先接通现有组件，不应再建设一套平行的 sandbox、registry 或 tracing。

### 4.3 当前 P0/P1 缺陷

| Priority | 缺陷 | 真实影响 | 证据 |
|---|---|---|---|
| P0 | L1 与 `coverage_satisfied` 未被主循环消费 | infra 污染仍继续设计候选；空 outcome 还会消耗 patience | `failure_map_builder.py:78-160`；`loop.py:431-468,629-642` |
| P0 | Activation/rollback 只有账本状态 | `activated` 不会应用 patch；rollback JSON 不是可执行恢复 | `activation/artifacts.py:8-15,1131-1146` |
| P0 | AppWorld “Sandbox” 直接调用宿主 `bash` | 候选可访问宿主环境、网络、凭据和工作树外路径 | `benchmarks/appworld/evolve/sandbox.py:85-114` |
| P0 | `--force` 解封后没有真实 invalid 标记 | 看过 sealed 数据后仍可继续搜索并覆盖 retention | `launch/runner.py:202-207`；`launch/state.py:62-115` |
| P1 | 默认 frozen baseline、`min_confirm_lift=0` | 漂移、多候选/多轮选择造成假晋升 | `benchmarks/appworld/evolve/entry.py:19-27,302-312` |
| P1 | 2σ/Fisher 多为报告标签 | candidate mean 只要略高就可能晋升 | `gates/strategies.py:167-184,290-328` |
| P1 | 关键审计写入有 best-effort 吞错 | 控制流可恢复但学习证据可能残缺 | `orchestrator/loop.py:648-720` |

### 4.4 Skill Forge 不是 Skill learning

当前链路是：

```text
Workspace/Builtin/Configured SKILL.md
  → SkillRegistry
  → LocalPool BM25
  → SkillForgeRouter
  → Resolver
  → Context Segment
  → inline body/reference/skill_read
```

它能够发现并注入现有 Skill，但没有 candidate generation、成功/失败归因、自动修订、独立 benchmark gate、版本晋升或 rollback。`SkillEvolver` 已明确删除；Agent turn 主要记录 injected IDs，并没有把 outcome 反馈给 Skill 学习器。

另外存在两类风险：

- workspace `SKILL.md` 可由 Agent 写入，watcher 会让新文件随后进入索引，正文又直接进入 system context；
- `SkillForgeRouterConfig.enabled=False` 当前没有真正 bypass factory，配置语义与运行行为不一致。

证据：

- `pico/memory_engine/skill_local/__init__.py:3-11`
- `pico/memory_engine/skill_local/registry.py:41-49,166-178`
- `pico/memory_engine/skill_forge/catalog.py:30-114`
- `pico/context_engine/factory.py:96-163`
- `pico/context_engine/segments/skills.py:28-87`
- `pico/agent/loop/main.py:1920-2011`

### 4.5 Memory、Curator、Personalizer 的正确定位

- MemoryConsolidator/Layered Memory 在积累状态和摘要，不会生成并晋升策略候选；
- Curator 在选择、归档和裁剪上下文，不修改 Agent policy；
- Personalizer 默认关闭，启用后由单模型提取偏好写 `user.md`，没有二次 judge、用户确认或 rollback；
- `MemoryBackend.feedback()` 当前并不由 active host dispatch。

这些能力是 Evolution 的证据源和部署对象之一，但不能直接等同于 Evolution 闭环。

### 4.6 测试现状

已有单元/集成覆盖 manifest、gate arithmetic、Git ops、activation artifact、launch/resume、Skill registry/BM25/router、Memory/Curator/Personalizer 等。关键空白是：

- L1 pause 与局部 evidence readiness；
- 真实 apply/activate/rollback；
- candidate edit/import/test/eval 的全路径隔离；
- sealed 数据跨 run exposure；
- Skill 的独立效果归因与 promotion；
- 真实 AppWorld 端到端。

当前机器默认 Python 为 3.10.9，而项目要求 `>=3.12,<3.13`，且默认环境缺部分依赖；因此本次审计没有声称 Harness 全量测试通过。M0 必须先建立可重复的 Python 3.12 环境。

## 5. 从调研到 Pico v2 的直接映射

| 成熟经验 | Pico 当前状态 | v2 选择 |
|---|---|---|
| Trace + grader + dataset 是质量飞轮核心 | 有 trace，缺可信 outcome receipt | 新增 typed `OutcomeReceipt`，Agent 自评不能签发 |
| Memory/Skill 是外部能力资产，不是权重学习 | 有 Memory 和只读 Skill retrieval | 首个闭环选择 instruction-only Skill |
| Candidate generator 与 evaluator 分离 | Judge/候选/评测边界仍有泄漏 | Candidate、trusted evaluator、controller 三身份 |
| 独立 validation 防过拟合 | sealed 单次有设计，但同 run/跨 run 暴露治理不足 | fresh grouped shard + exposure ledger + retire |
| 安全规则不能由学习器修改 | path guard 有效但不是 OS 隔离 | R4 trust root + existing BoxLite strict profile |
| 发布必须版本化、可观测、可恢复 | activation 只有 metadata | `ReleaseSpec` + `DeploymentRecord` + CAS reconciler |
| Context 版本必须固定 | shared registry/cache 可能混版本 | `SessionCapabilitySnapshot` 与 digest-keyed catalog |
| 先做小范围可验证对象 | 当前想扩到多类 candidate | Skill 纵向切片；Prompt 后接；Runtime 部署后移 |

## 6. 决策

1. Pico Evolution v2 的首个产品闭环定为 `Experience → Skill → fresh promotion → Release → rollback`。
2. LLM 负责对比轨迹、生成 hypothesis 和最小 Skill delta；不负责签发 success、risk、promotion 或 activation。
3. 所有候选执行路径接入现有 BoxLite executor；没有合格 backend 就拒绝运行。
4. Promotion 只使用从未参与生成或先前决策的 grouped fresh data；sealed 数据一经暴露永久退休。
5. 首版只支持 instruction-only Skill；Prompt 作为第二 adapter；Runtime 候选先完成隔离，生产部署后置。
6. Memory、workspace Skill 和 operator Skill 的变化纳入 session snapshot；普通 rollback 与强制 revoke 分开。
7. 在线权重更新、自由 workflow/code 自修改和自动权限扩张继续保持 out of scope。

详细 schema、统计 gate、发布事务、验收实验和 9–11 周里程碑见 [Evolution v2 迭代方法](self-evolution-v2.zh-CN.md)。
