# Agent × 后端网关 SCI 选题研究包（第一篇）

> **研究方向 1：面向工具调用智能体的风险感知、安全副作用恢复网关**  
> 英文建议题目：**RAGuard: Risk-Aware and Side-Effect-Safe Recovery at the Tool Gateway for LLM Agents**  
> 资料检索截至：**2026-10-20** ｜用途：本科生/小团队立项与 SCI 投稿前评估 ｜状态：**选题论证，不是已验证的创新成果或已完成论文**

## 阅读结论（先看这页）

- **主观点**：Agent 不应该因为一个失败的工具调用而无条件重做任务；恢复必须同时考虑**外部副作用、网络结果不确定性、调用依赖和任务完成率**。
- **主创新点（拟验证）**：在不修改 Agent 推理模型的前提下，在 **HTTP/API/MCP 工具网关边界**构建一个**证据驱动、风险约束、可审计的恢复决策器**。根据工具操作是否可逆、结果是否确认、业务状态是否可回读、失败传播范围，动态选择**重放缓存结果、安全重试、从依赖节点局部恢复、补偿、停止并交人工**等动作。
- **真正的科研问题**：能否在严格限制**重复副作用与不安全恢复**的条件下，相较固定重试、Saga 式补偿、已有 Agent checkpoint/recovery 方案，提高成功率，同时控制 P95 时延、额外 API 调用和成本？
- **非常重要的查重判断**：不能把“失败检测 + 重试 + 日志”称为新方法。**AgentRewind（2026）**已做上下文与环境恢复，**ACRFence（2026）**已识别语义回滚带来的重复副作用，**Crab（2026）**已做非侵入式语义感知检查点。若没有超出这三者的可实验贡献，应**及时换题/缩题**，不宜贸然写 SCI。
- **免费发表的正确理解**：首选 **SCIE 收录、支持 Subscription（订阅发表）** 的期刊；**不选付费 Open Access**，通常可不交文章处理费（APC）。这不等于文章免费向所有读者开放，也**不保证**特殊稿型不存在额外收费。
- **快刊实际情况**：查询到的官方“投稿至录用”统计约 **147–207 天**（视候选刊而异），不是几天录用。最快的期刊不一定最适合网关主题。

---

## 0. 五个方向的主观点和主创新点总览

> 本文件只对方向 1 展开论文级研究方案；方向 2–5 保留作为后续文件的索引。以下全是**待验证的创新设想**，不是“世界首创”结论。

| 方向 | 一句话主观点 | 可验证的核心创新假设 |
|---|---|---|
| **1. Agent 故障感知与安全恢复网关（本篇）** | 恢复 Agent 时，**任务成功**不能以**外部状态损坏**为代价 | 无侵入网关将工具语义、副作用账本、状态证据和风险约束联合决策，减少重复写入/误恢复，同时保留任务成功率 |
| **2. 多模型动态路由与成本优化** | 路由不应只按提问复杂度选模型，而应对每步的失败代价负责 | 将**结果可验证性、重试代价、长尾延迟、任务依赖**纳入任务级动态路由，而不只是最低 Token 价格 |
| **3. MCP 工具调用安全验证网关** | 工具被发现不等于工具值得信任 | 引入**跨调用的工具权限演化图和运行时证据契约**，对危险工具链做细粒度授权与封锁 |
| **4. 多 Agent 调度与拥塞控制** | 多 Agent 的竞争不仅争 GPU，也争工具与外部服务容量 | 联合考虑**Agent 依赖 DAG、限流额度、API 配额、尾延迟**，设计风险/期限感知的调度策略 |
| **5. Agent 调用链根因定位** | 最终报错步骤通常并非最初出错步骤 | 融合**调用拓扑、环境状态差分、概率因果证据**，识别可恢复的最小错误子图 |

---

## 1. 研究问题、边界与论文贡献

### 1.1 场景

一个购物/工单 Agent 依次调用：查询库存 → 创建订单 → 扣除配额 → 发送通知。创建订单已在服务器完成，但 HTTP 响应超时，Agent 以为没成功；自动重试可能生成另一个 request ID，导致重复订单。这时“更聪明地让 LLM 再试一次”反而不安全。

### 1.2 拟研究的四个明确问题（RQ）

- **RQ1（正确性）**：网关是否能显著减少超时、断连、返回体损坏、重复提交、语义改写等情况下的**重复副作用率与业务状态违约率**？
- **RQ2（有效性）**：相比总是停止，风险约束的恢复能否显著提高**任务完成率**与**恢复率**？
- **RQ3（效率）**：与全量重新运行、checkpoint 恢复比较，是否能减小**恢复范围、P95 端到端时延、额外工具调用数**？
- **RQ4（泛化）**：是否能跨**两类 Agent 框架、多个模型、至少三种工具服务、不同故障强度**保持效果？

### 1.3 推荐主创新点：证据约束的恢复动作选择

**假设 H1**：与仅凭 HTTP 状态码或异常字符串的恢复策略相比，结合**操作语义 + 副作用证据 + 局部依赖图**的网关，可在相似故障强度下显著降低误恢复。  
**假设 H2**：与全部故障均交人工相比，对“已被证实可恢复”的子集做自动恢复，可显著提高整体任务完成率，同时将不安全执行维持在预先设定的阈值内。  
**假设 H3**：相较全局 rollback，一种分段/依赖感知的恢复策略可降低无关步骤的重算负担。

**关键区分（审稿人一定会问）**：

| 已有工作 | 已做什么 | 本选题必须额外做什么，否则重复 |
|---|---|---|
| AgentRewind（2026） | 对齐 Agent 语境与受控环境的 checkpoint/restore | 研究**不受 Agent 完全控制的外部 API 状态**、网关独立部署、网络不确定成功 |
| ACRFence（2026） | 研究语义回滚攻击、重复不可逆副作用；提出 replay-or-fork | 不得声称首次发现重复副作用；额外贡献应是**联合故障分类、恢复成本/风险优化、跨服务真实实验** |
| Crab（2026） | 利用 Agent/OS 语义做透明检查点与优化 | 不去重复做 OS 状态快照；重点做**业务/API 外部副作用一致性** |
| SagaLLM（2025） | 在多 Agent 规划中引入事务、校验、补偿 | 证明网关层**异构工具、非协作服务、不确定返回**时的新增价值 |
| AgentDebug/AgentDebugX（2025–2026） | 故障归因与重执行建议 | 方法要能**真实执行和约束恢复**，而非仅诊断/生成提示词 |

### 1.4 对于“能不能发 SCI”的保守判断

**可以立项，不可以现在承诺能发。** 如果只有 1 个 API demo、3 种注入异常和 2 张成功率柱状图，很可能只是工程项目。至少需要：（a）可形式化的安全属性；（b）与紧邻方法公平对照；（c）可复现的状态级任务结果；（d）显著性和消融；（e）明确失败条件。若没有提升，应以负结果调整方法，而不是包装出提高百分比。

---

## 2. 可实现的技术方案

### 2.1 总体结构

```text
User / Task
    │
Agent Framework A or B (原有推理/规划逻辑)
    │ Tool Call (HTTP / MCP / SDK Adapter)
    ▼
┌────────────────────── RAGuard Gateway ──────────────────────┐
│  1. Tool Registry: 调用类型、幂等性、可逆性、依赖、权限      │
│  2. Trace Recorder: request/response、时间戳、父调用、状态   │
│  3. Effect Ledger: 确认成功/确认失败/状态未知的外部副作用   │
│  4. Evidence Resolver: 幂等键、查询回读、状态差分、事件回执 │
│  5. Safety Gate: 禁止未证实安全的高风险自动重试            │
│  6. Recovery Selector: replay / retry / local resume /     │
│                        compensate / stop & escalate          │
└─────────────────────────────────────────────────────────────┘
    │
    ├── Read-only service
    ├── Idempotent write service
    ├── Reversible write service (有有效补偿操作)
    └── Irreversible / opaque external service
```

### 2.2 应记录的最小事实

每次 Tool Call 记录：`task_id, step_id, logical_intent_id, tool_id, operation_class, dependency_ids, stable_idempotency_key, request_fingerprint, response_status, verified_effect_state, evidence_source, cost, timestamp`。

`logical_intent_id` 是经过业务约束定义的**逻辑动作 ID**，不能仅靠 LLM 对两段自然语言算一个相似度就断定它们是同一业务动作。**不能把不同收款人、金额或权限主体的操作合并为同一意图。** 日志应去标识化、加访问控制，不记录明文密钥和无关隐私。

### 2.3 动作候选与安全准入

| 动作 | 允许条件 | 禁止/升级情形 |
|---|---|---|
| **Replay result** 重放结果 | 与本次逻辑调用有可靠匹配且已有验证过的结果 | 请求语义或权限已变化；缓存过期；结果与现态冲突 |
| **Retry** 安全重试 | 可证明只读；或服务端支持稳定幂等键；或能先回读证实原调用未生效 | 已有副作用但结果不明；不可逆调用无幂等保证 |
| **Local resume** 局部恢复 | 依赖图与持久化状态显示部分步骤可复用 | 上游状态已失效或跨服务因果关系不明 |
| **Compensate** 补偿 | 业务定义了可审计的反向操作且前置条件成立 | 例如邮件已发出、款项已结算等无法真正撤销的效果 |
| **Stop / human approval** 停止或人工审核 | 证据不足、风险过高、外部状态不可观察 | 不能为了刷完成率绕过此动作 |

**边界原则**：网关**不能普遍保证 exactly-once 的业务执行**。只有在外部服务支持幂等语义/业务键、可靠回读或事务性协作等明确假设下，才能给出有限的安全保证。对于不透明且不可逆的 API，正确恢复策略可能只能是**停止**。

### 2.4 基础决策形式（可扩展算法论文）

设 `s` 为网关观测到的故障状态，`a` 为恢复动作，`A_safe(s)` 为安全门允许的动作集合：

```text
min_a  E[latency(a) + λ1·API_cost(a) + λ2·recompute(a) + λ3·failure_penalty(a)]
s.t.   a ∈ A_safe(s)
       estimated_harm_risk(a|s) ≤ ε     (仅当风险模型经校准且有支持证据)
```

**先做确定性安全门，再做可学习决策**：

1. **v0**：手工规则＋保守停止策略（可作为论文可靠基线）。
2. **v1**：风险校准的代价敏感选择（如 contextual bandit、受约束决策）。
3. **v2 可选**：轻量时序因果/故障传播图，控制恢复范围。

不可把未验证的模型概率直接当作安全证明；若没有足够风险标签，就只主张**实验风险下降**，不主张形式安全认证。

### 2.5 跨框架最小实现

- **网关**：Go（轻量反向代理 + 并发）或 Python FastAPI（最小科研原型）；状态账本 PostgreSQL/SQLite；结构化日志 JSONL；可选 Redis（缓存，但不能单独作为不可逆事件的唯一持久证据）。
- **Agent**：至少一种 ReAct 风格工具 Agent + 一种不同编排/状态管理实现；可选 LangGraph，但不要依赖特定框架私有 API。
- **服务**：本地容器模拟支付/订单/工单/通知；**不要使用真实支付、真实发信或真实个人信息执行故障试验**。
- **实验**：Docker Compose 固定依赖、种子、版本，脚本一键运行；部署时明确隔离测试 API 和真实系统。

---

## 3. 至少四组可做的交叉学科课题

| 交叉点 | 具体研究题目 | 可以形成的独立贡献 | 代价/风险 |
|---|---|---|---|
| **分布式数据库 × Agent** | 不确定提交（unknown commit）的安全恢复 | 把数据库 WAL/Saga/幂等语义应用到**语义可能变化的 Agent 调用**；建立效果账本 | **首选**，需严谨状态设计 |
| **可靠性工程 × 混沌测试** | Agent 网关故障注入与恢复充分性评测 | 复合故障注入矩阵、故障覆盖指标、故障传播规律；发布开放基准 | **首选**，实验可控 |
| **形式化方法 × Agent 安全** | 对恢复状态机做形式规约与模型检查 | 使用 TLA+/状态机验证“不自动重复不可逆写入”等**条件性不变式** | 成本中，需学习建模 |
| **控制理论/强化学习 × 网关调度** | 具有安全动作集的恢复策略优化 | 在已通过硬性 Gate 的策略中优化时延/成本/成功率 | 成本中高，需防黑箱化 |
| **网络系统 × 拥塞与限流** | 429/超时/链式退避的协同恢复 | 跨 Agent 的重试风暴抑制、P95 尾延迟与成功率折中 | 成本中，适合 JNCA/FGCS |
| **信息安全 × MCP 治理** | 故障驱动的工具权限滥用防护 | 研究在恢复阶段是否扩大工具权限，给出权限收缩策略与可审计日志 | 成本中，注意与 MCPShield 等去重 |
| **软件工程 × 因果定位** | 故障子图与最小恢复切片 | 定位最早导致错误的工具依赖节点，仅重执行必要子图 | 成本中高，与 AgentDebugX 区分 |

**推荐只选一个主干 + 一个强化点**：例如“**分布式事务语义 + 可靠性故障注入**”；不要在一篇本科生论文里同时塞 RL、因果图、形式化证明、密码学验证。

### 值得优先选择的三种论文变体

1. **方案 A（最务实）**：`Semantic-Effect-Aware Agent Recovery Gateway` —— 不确定提交＋外部副作用账本＋局部恢复；目标期刊 JNCA / JSS。
2. **方案 B（偏算法）**：`Risk-Constrained Adaptive Recovery of Tool-Using Agents` —— 有安全门的代价敏感恢复策略；目标期刊 FGCS / ESWA（必须给出真实算法贡献）。
3. **方案 C（偏验证）**：`Failure Injection and Safety Contracts for Agent Gateways` —— 公开故障基准＋协议约束＋条件性安全证明；目标期刊 JSS / IST。

---

## 4. 实验设计：按 SCI 论文标准预注册

### 4.1 实验维度

| 维度 | 最低覆盖建议 |
|---|---|
| 真实任务类型 | 订单/库存、工单/CRM、云资源或文件工作流（至少 3 领域，均为沙箱） |
| Tool 操作性质 | 只读、幂等写、可补偿写、不可逆写、外部状态不可读 |
| 故障类型 | 超时、断联、HTTP 429/5xx、成功但响应丢失、响应篡改/脏读、schema drift、并发重复提交 |
| 故障强度 | 0%、1%、5%、10%、20%（是预设的试验网格，**非预测的实际故障率**） |
| Agent | 至少 2 种不同编排方案；不同 LLM 或模型大小作敏感性分析 |
| 重复 | 固定种子，**每条件至少 30 次**为起点；最终样本量由方差和统计功效决定 |
| 压力 | 1/10/50 并发等可承受档位，记录网关吞吐、CPU/RAM 和长尾延迟 |

### 4.2 必备 Baseline

- B0：无恢复，失败直接结束。
- B1：固定次数重试。
- B2：指数退避＋抖动（带服务端 Retry-After）。
- B3：已知幂等键＋硬规则防重（**强基线，不能故意阉割**）。
- B4：工作流 Saga/补偿规则（适用时）。
- B5：可复现情况下接入 AgentRewind / ACRFence 思路或官方实现；**若实现不可得，公开说明复现近似、缺陷和偏差**。
- B6：本方法（含无风险模型的 Rule Gate 版本）。

### 4.3 指标和严禁的统计错误

| 指标 | 明确定义 |
|---|---|
| Task Success Rate（TSR） | **最终业务数据库状态**满足任务目标的比例，不用 LLM 自评成功代替 |
| Unsafe Side-Effect Rate | 重复创建/重复扣款模拟/权限违规等**业务不变式违反**的任务比例；按异常严重程度报告 |
| Recovery Success Rate | 已注入故障的任务中，最终达到合法目标态的比例；不能与 TSR 混用 |
| Ambiguous-Commit Resolution Rate | 对“服务端已执行、客户端结果未知”事件正确确认或保守阻断的比例 |
| Recovery Overhead | 比无故障同配置多出的耗时、CPU、持久化存储、API 数量和 Token 数量 |
| Tail Latency | 端到端 P50/P95/P99，按完成/失败/人工升级分别报告，防止选择偏差 |
| Escalation Rate | 无法安全自动恢复而被停止/交人工的比例 |
| False-Safe Rate | 被网关判定“可安全重试”但事实上破坏状态约束的事件比例 |

- 对同一个任务和故障种子进行**配对比较**，用 bootstrap 置信区间；条件允许时报告 McNemar 或配对检验与效应量。
- 不要只汇报“提高 80%”一类相对比例；必须同时给**原始分子分母、绝对百分点、95% CI**。
- 不能将拒绝危险操作的“零违规率”隐含解读为系统表现最好；必须联合看 TSR 与升级率。
- 不能将受控假服务上的“条件性不变式成立”当作所有真实 SaaS 都有 exactly-once 保证。

### 4.4 必做消融和压力实验

- Ablation A：去掉 Effect Ledger。
- Ablation B：去掉状态回读，仅凭请求状态码。
- Ablation C：去掉依赖图，退化为全量重跑。
- Ablation D：去掉风险门，仅用最小延迟恢复。
- Ablation E：使用不稳定/被 LLM 重新生成的幂等键，分析退化。
- 交叉测试：先确认提交再断线、先断线再提交、双重 429、并发重复、网络分区恢复、业务补偿失败。

### 4.5 Stop/Go 决策门（研究前设，非当前结果）

| 阶段 | 通关条件 | 不达标时 |
|---|---|---|
| Feasibility | 用 1 个本地工具服务重现“服务端成功、客户端超时导致重复写入” | 暂停，不写论文 |
| Safety | 同条件对照显示网关显著减少危险副作用，且有明确代价说明 | 回到状态/幂等设计 |
| Comparative | 在强 B3/B4 基线之上仍找到净收益（成功率/时延/恢复开销至少一项） | 缩成故障基准研究或换题 |
| Generalization | 至少 3 业务域、2 Agent 实现可复现 | 仅主张特定场景，不做普适结论 |
| Manuscript-ready | 开源脚本、输入任务、原始结果、统计、限制和威胁分析齐全 | 不急于投稿 |

---

## 5. 期刊甄选：只保留可选择**不付 APC 的订阅发表**路线

**时间口径（重点）**：表内数值来自查询时 Elsevier **ScienceDirect 官方 Journal Insights**，是**历史统计，不是审核承诺**。

- **投稿至初次决定**：通常包含编辑初筛/直接拒稿，所以 3–11 天的数值**不能**代表同行评审很快。
- **投稿至外审后决定**：进入审稿之后作出决定的历史统计，不等于接受。
- **投稿至录用**：包含审稿、作者修订等，对排计划更有参考价值；并非“个人预期到账日”。
- **投稿费 ≠ APC**；表内“免费”仅指作者选择 **Subscription** 时**通常无出版费**，需要在提交时核实稿型/额外费用/资助方 OA 规定。

| 期刊（均按官网信息列入 SCIE） | 适配程度 | 初决 | 外审后决定 | 投稿→录用 | 录用→上线 | 订阅发表可免 APC | 主要风险 |
|---|---|---:|---:|---:|---:|---|---|
| **Expert Systems with Applications**（ESWA） | ★★☆：偏智能决策算法 | 5 天 | 62 天 | **147 天** | 7 天 | 是 | 纯代理服务器/重试工程**可能不符 AI 方法要求** |
| **Future Generation Computer Systems**（FGCS） | ★★★★：分布式系统、服务治理 | 3 天 | 72 天 | **183 天** | 10 天 | 是 | 需要规模/可扩展性实验，门槛不低 |
| **Journal of Network and Computer Applications**（JNCA） | ★★★★★：API 网关/网络应用 | 11 天 | 88 天 | **187 天** | 10 天 | 是 | 要突出网络/API 与协议问题，而非泛 Agent Demo |
| **Information and Software Technology**（IST） | ★★★★：软件工程方法/实证 | 7 天 | 85 天 | **202 天** | 10 天 | 是 | 需要严谨研究设计；2026 编辑文章特别强调拒稿风险 |
| **Journal of Systems and Software**（JSS） | ★★★★★：可靠性/软件系统 | 5 天 | 74 天 | **203 天** | 12 天 | 是 | 实证深度高；不能是工具说明文 |
| **Engineering Applications of Artificial Intelligence**（EAAI） | ★★★：有算法＋工程验证时 | 27 天 | 85 天 | **207 天** | 11 天 | 是 | 需要明确 AI 新方法及公共数据可复现验证 |

官方期刊页面及统计：

1. [ESWA — Journal Insights](https://www.sciencedirect.com/journal/expert-systems-with-applications/about/insights)
2. [FGCS — Journal Insights](https://www.sciencedirect.com/journal/future-generation-computer-systems/about/insights)
3. [JNCA — Journal Insights](https://www.sciencedirect.com/journal/journal-of-network-and-computer-applications/about/insights)
4. [IST — Journal Insights](https://www.sciencedirect.com/journal/information-and-software-technology/about/insights)
5. [JSS — Journal Insights](https://www.sciencedirect.com/journal/journal-of-systems-and-software/about/insights)
6. [EAAI — Journal Insights](https://www.sciencedirect.com/journal/engineering-applications-of-artificial-intelligence/about/insights)
7. [Elsevier 出版商政策：混合期刊订阅发表通常无需 APC](https://www.elsevier.com/about/policies-and-standards/pricing)
8. [Clarivate Master Journal List：提交当天再核对 SCIE](https://mjl.clarivate.com/)

> 注：这些是**SCI/SCIE 收录候选**，不是“中科院一区/二区”承诺。中科院分区、JCR 分区及影响因子会随年份更新；本研究包**不引用无法由期刊官方当前数据核对的固定分区**。

### 5.1 按“速度 × 免费 × 适配”的推荐顺序

**若做方案 A（网关事务/故障恢复）**：JNCA → FGCS → JSS → IST；**不能因为 ESWA 的历史录用周期更短就强投 ESWA**。  
**若做方案 B（真正有 AI 算法贡献）**：ESWA → FGCS → EAAI；需能证明决策算法比强规则基线有统计显著收益。  
**若做方案 C（软件实证/基准）**：JSS → IST → JNCA。

**关于免费发表的两次检查**：投稿前看 `Article publishing options → Subscription → No publication fee charged to authors`；接收后签订出版协议时**不要误选付费 OA**。若基金、学校政策要求立即 OA，则可能与“零费用”约束冲突，需提前协调。以上收录、速度和收费均应在真正投递当天复核。

---

## 6. 同类与关联论文清单（40 篇，全部附可追溯入口）

**阅读说明**：按研究关系分层，不意味着每篇都与拟议网关方法直接竞争；**arXiv 预印本/会议论文并不等于 SCI 期刊论文**。作者/年份/摘要/结论以原文最新版本为准。本列表不是声称“领域仅有这 40 篇”。优先级：**P0 必读直接重叠**、P1 重要框架与评测、P2 方法背景。

### A. 恢复、故障、安全与根因定位（16 篇）

| # | 论文 / 年份 / 入口 | 优先 | 与本选题的关系、必须考虑的去重点 |
|---:|---|:---:|---|
| 01 | [AgentRewind: Recoverable Execution for Long-Horizon LLM Agents](https://arxiv.org/abs/2608.14380)（2026） | **P0** | 对齐 Agent 与环境 checkpoint；必须比较恢复代价及跨外部服务的区别 |
| 02 | [ACRFence: Preventing Semantic Rollback Attacks in Agent Checkpoint-Restore](https://arxiv.org/abs/2603.20625)（2026） | **P0** | 重复外部副作用/语义回滚已有人提出；必须明确本工作的新决策对象 |
| 03 | [Crab: A Semantics-Aware Checkpoint/Restore Runtime for Agent Sandboxes](https://arxiv.org/abs/2604.28138)（2026） | **P0** | 非侵入式 OS 侧检查点；网关研究应聚焦 API 业务状态而非复做沙箱快照 |
| 04 | [SagaLLM: Context Management, Validation, and Transaction Guarantees for Multi-Agent LLM Planning](https://arxiv.org/abs/2503.11951)（2025） | **P0** | Agent 事务/补偿、工作流一致性，属于直接先行工作 |
| 05 | [PALADIN: Self-Correcting Language Model Agents to Cure Tool-Failure Cases](https://arxiv.org/abs/2509.25238)（2025） | **P0** | 故障注入＋学习式恢复；区分“改 Agent 能力”与“网关运行时约束” |
| 06 | [Where LLM Agents Fail and How They can Learn From Failures](https://arxiv.org/abs/2509.25370)（2025） | **P0** | AgentErrorTaxonomy / AgentDebug，为故障分类和比较提供依据 |
| 07 | [AgentDebugX: An Open-Source Toolkit for Failure Observability, Attribution, and Recovery in LLM Agents](https://arxiv.org/abs/2607.18754)（2026） | **P0** | 检测—归因—恢复—重测闭环；须做真正 API 安全动作保证 |
| 08 | [CUADebug: Diagnosing and Repairing Computer-Use Agent Failures](https://arxiv.org/abs/2608.02643)（2026） | P1 | 人机界面 Agent 错误定位与重运行，适合作跨模态迁移参考 |
| 09 | [ReliabilityBench: Evaluating LLM Agent Reliability Under Production-Like Stress Conditions](https://arxiv.org/abs/2601.06112)（2026） | **P0** | 可靠性表面、API 故障注入、pass^k；直接参考实验设计 |
| 10 | [When Tools Fail: Benchmarking Dynamic Replanning and Anomaly Recovery in LLM Agents](https://arxiv.org/abs/2606.05806)（2026） | **P0** | ToolMaze 工具故障与动态重规划，必须比较工具故障场景 |
| 11 | [ToolRobustBench: Stage-Wise Perturbation Evaluation and Failure Diagnosis for Tool-Calling Agents](https://arxiv.org/abs/2608.23635)（2026） | **P0** | 分阶段扰动，避免只报一个总成功率 |
| 12 | [MCPShield: A Security Cognition Layer for Adaptive Trust Calibration in Model Context Protocol Agents](https://arxiv.org/abs/2602.14281)（2026） | P1 | MCP 的工具信任、调用前后校验，安全网关近邻工作 |
| 13 | [Attested Tool-Server Admission: A Security Extension to the Model Context Protocol](https://arxiv.org/abs/2605.24248)（2026） | P1 | MCP 工具准入、白名单与可审计授权；与执行后恢复不同 |
| 14 | [LLM Agents as Resilience Engineers for Scientific Applications](https://arxiv.org/abs/2610.11260)（2026） | P1 | LLM 辅助系统 checkpoint/resilience；偏 HPC 工程，跨领域参考 |
| 15 | [When Errors Become Narratives: A Longitudinal Taxonomy of Silent Failures in a Production LLM Agent Runtime](https://arxiv.org/abs/2606.14589)（2026） | P1 | 静默故障与观测性，支持“失败必须可检测且可追责”论据 |
| 16 | [Beyond the Leaderboard: A Synthesis of Tool-Use, Planning, and Reasoning Failures in Large Language Model Agents](https://arxiv.org/abs/2607.05775)（2026） | P1 | 失败模式综合归纳，帮助明确故障不止是超时与异常码 |

### B. 可复现实验与工具调用基准（13 篇）

| # | 论文 / 年份 / 入口 | 优先 | 可用于什么 |
|---:|---|:---:|---|
| 17 | [MCP-Universe: Benchmarking Large Language Models with Real-World Model Context Protocol Servers](https://arxiv.org/abs/2508.14704)（2025） | **P0** | 面向真实 MCP Servers 的任务来源与执行评估思路 |
| 18 | [MCP-Atlas: A Large-Scale Benchmark for Tool-Use Competency with Real MCP Servers](https://arxiv.org/abs/2602.00933)（2026） | P1 | 真实 MCP 服务/多步骤任务，关注工具多样性 |
| 19 | [τ-bench: A Benchmark for Tool-Agent-User Interaction in Real-World Domains](https://arxiv.org/abs/2406.12045)（2024） | **P0** | 以**最终数据库状态**评估任务是否真实完成，强烈推荐 |
| 20 | [τ²-Bench: Evaluating Conversational Agents in a Dual-Control Environment](https://arxiv.org/abs/2506.07982)（2025） | P1 | 双方控制共享状态，评估并发/协调边界 |
| 21 | [ToolSandbox: A Stateful, Conversational, Interactive Evaluation Benchmark for LLM Tool Use Capabilities](https://arxiv.org/abs/2408.04682)（2024） | **P0** | 状态依赖/隐式前置条件/恢复时业务正确性 |
| 22 | [AgentDojo: A Dynamic Environment to Evaluate Prompt Injection Attacks and Defenses for LLM Agents](https://arxiv.org/abs/2406.13352)（2024） | P1 | 安全维度，可扩展为“故障恢复中恶意返回”测试 |
| 23 | [API-Bank: A Comprehensive Benchmark for Tool-Augmented LLMs](https://arxiv.org/abs/2304.08244)（2023） | P2 | API 工具基础基准与调用步骤构造 |
| 24 | [ToolLLM: Facilitating Large Language Models to Master 16000+ Real-world APIs](https://arxiv.org/abs/2307.16789)（2023） | P2 | ToolBench 工具生态；不要把 API 个数当作高质量故障实验 |
| 25 | [Gorilla: Large Language Model Connected with Massive APIs](https://arxiv.org/abs/2305.15334)（2023） | P2 | API 使用参数/版本漂移故障的背景 |
| 26 | [AgentBench: Evaluating LLMs as Agents](https://arxiv.org/abs/2308.03688)（2023） | P1 | 跨 Agent 评测、环境多样性 |
| 27 | [GAIA: a benchmark for General AI Assistants](https://arxiv.org/abs/2311.12983)（2023） | P2 | 真实任务复杂度参考，不适合作唯一状态正确性基准 |
| 28 | [SWE-bench: Can Language Models Resolve Real-World GitHub Issues?](https://arxiv.org/abs/2310.06770)（2023） | P2 | 长链任务与环境部署复杂度；费用与搭建较高 |
| 29 | [WebArena: A Realistic Web Environment for Building Autonomous Agents](https://arxiv.org/abs/2307.13854)（2023） | P1 | 可重复的状态型 Web 任务；环境相对重 |

### C. Agent 行动、自纠错与评估理论（7 篇）

| # | 论文 / 年份 / 入口 | 优先 | 用途 |
|---:|---|:---:|---|
| 30 | [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)（2022） | P1 | 基础工具 Agent 的标准基线机制 |
| 31 | [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366)（2023） | P1 | 自我反思恢复基线，和网关恢复做区分 |
| 32 | [CRITIC: Large Language Models Can Self-Correct with Tool-Interactive Critiquing](https://arxiv.org/abs/2305.11738)（2023） | P1 | Tool feedback 自纠错，对照模型层恢复 |
| 33 | [Self-Refine: Iterative Refinement with Self-Feedback](https://arxiv.org/abs/2303.17651)（2023） | P2 | 提示词自修复背景，不能用作外部业务安全保证 |
| 34 | [Toolformer: Language Models Can Teach Themselves to Use Tools](https://arxiv.org/abs/2302.04761)（2023） | P2 | LLM 工具使用基础发展脉络 |
| 35 | [Evaluation and Benchmarking of LLM Agents: A Survey](https://arxiv.org/abs/2507.21504)（2025） | P1 | 评估目标/过程两维分类，帮助组织实验章节 |
| 36 | [AI for Next Generation Computing: Emerging Trends and Future Directions](https://arxiv.org/abs/2203.04159)（2022） | P2 | 自治资源管理、云边协同与系统研究背景 |

### D. 分布式系统理论基础（3 篇）

| # | 文献 / 入口 | 优先 | 与 Agent 网关的交叉点 |
|---:|---|:---:|---|
| 37 | [Sagas — Garcia-Molina & Salem, SIGMOD '87](https://doi.org/10.1145/38713.38742)（1987） | **P0** | 长事务拆分与补偿，**不能把补偿等同于物理撤销** |
| 38 | [Distributed Snapshots: Determining Global States of Distributed Systems — Chandy & Lamport](https://doi.org/10.1145/214451.214456)（1985） | P1 | 分布式全局状态快照与一致性理论基础 |
| 39 | [Evaluation Report on MCP Servers — MCPBench](https://arxiv.org/abs/2504.11094)（2025） | P1 | MCP Server 基础测试，适合检查工具适配协议与评测口径 |

> **合计 39 篇**：A 类 16、B 类 13、C 类 7、D 类 3。特别说明：#37–38 是**真实学术经典文献**，不是 2025–2026 年新论文；部分 2025–2026 年条目可能仍为**未经同行评审的预印本**。最终论文参考文献要进一步补作者、会议/期刊、页码/DOI/版本，并核对是否存在正式出版版本。

### 6.1 优先阅读顺序（只先读 10 篇，不要一口气 39 篇）

**ACRFence → AgentRewind → Crab → SagaLLM → ReliabilityBench → ToolMaze → ToolRobustBench → AgentDebugX → τ-bench → ToolSandbox**。

每篇读完填 7 个字段：`问题、系统边界、主要方法、创新证据、基线、数据与指标、我能复现/突破的缺口`。只读摘要不够形成可投稿的 Related Work。

---

## 7. 可投稿的论文结构（建议英语全文）

1. **Introduction**：真实 Agent 失败示例；成功提交但响应丢失的双重执行风险；三项贡献。
2. **Background & Threat Model**：工具调用结构、外部服务合作假设、不确定提交模型、攻击者能力、禁止推断的事情。
3. **Problem Definition**：状态、效果账本、合法动作、约束、成本函数。
4. **RAGuard Design**：网关架构、证据检索、语义一致性验证、故障策略、持久性边界。
5. **Safety Analysis**：条件性不变式证明/模型检查；列明不满足假设时的失败场景。
6. **Experimental Setup**：模型、工具域、故障生成、重复/统计、强基线与具体复现脚本。
7. **Results and Ablations**：安全—任务成功率—开销三角权衡、泛化、并发压力、失败案例。
8. **Limitations & Threats to Validity**：未知 API、难以撤回的副作用、模拟服务外推、模型随机性、成本限制。
9. **Related Work / Conclusion**：与 AgentRewind、ACRFence、Crab、SagaLLM、AgentDebugX 严格区分。

### 可用的摘要骨架（**模板，不是已取得的成果**）

> LLM-based agents execute state-changing operations through external tools, where uncertain completion and retry-induced side effects can undermine task correctness. Existing recovery systems primarily target agent-context rollback, sandbox state restoration, or agent-level self-correction. We study a gateway-level recovery problem in which external service state may be only partially observable and tool operations differ in their reversibility. We propose RAGuard, an evidence-driven recovery gateway that combines effect logging, state verification, and a safety-constrained action selector. We evaluate the approach with controlled fault injection on multiple stateful service workflows, comparing strong retry, idempotency, and checkpoint baselines. [Insert **measured** results and uncertainty intervals only after experiments.] We discuss conditions under which safe automation is possible and cases that require escalation.

---

## 8. 八至十二周推进计划（项目管理估计，不是保证时间）

| 周次 | 任务 | 交付与中止标准 |
|---|---|---|
| W1 | 精读 P0 前四篇，做创新冲突表 | 确认不与 ACRFence/AgentRewind/Crab 核心主张重复 |
| W2 | 三类本地状态服务、基线重试、故障注入器 | 可稳定复现 5 类以上可观察故障 |
| W3–4 | Tool Registry / Effect Ledger / State Readback / Safety Gate | 用自动单元和集成测试验证关键不变式 |
| W5 | 实现对照 B0–B4；必要时复现强 Agent 基线 | 有相同故障任务、统一计时和统一判定逻辑 |
| W6–7 | 风险/依赖感知恢复；至少一轮迭代 | 无法优于“强幂等键＋规则”则及时缩题 |
| W8 | 大规模故障注入、统计、消融、压力测试 | 产生原始日志、复现脚本和置信区间 |
| W9 | 理论/状态机与失败案例复核 | 绝不夸大 exactly-once |
| W10 | 初稿、图表、开源和论文查重 | 所有实验表格能由脚本复算 |
| W11–12 | 按 JNCA / FGCS / JSS 稿件格式整改并复核最新官网信息 | 经课题组自审再投稿，不用“投快刊”替代质量 |

**资源预期**：没有训练模型的硬性要求；可以用一台普通开发电脑先复现模拟工具与确定性 Agent。真实 LLM 推理可用小预算 API 或现有可运行模型，但要记录成本。完整参数量/并发规模取决于机器，切勿先宣称某个 GPU 就“足够”。

---

## 9. 给学弟的第一批任务（边做边评估）

1. 读 **ACRFence、AgentRewind、Crab、SagaLLM**；写 1 页“哪些东西别人已做、我们还能否做出独立贡献”的矩阵。
2. 自己写一个**可复现的最小错误场景**：服务端创建订单成功，但客户端超时，于是第二次请求又创建一笔；对照启用稳定幂等键后会发生什么。
3. 给同一个业务操作实现三种观测：**HTTP 状态、服务端状态回读、持久效果账本**，记录它们在丢包和重启时能否判断正确。
4. **先不写 LLM 策略**，实现两个强基线：指数退避、服务端幂等＋业务回读；如果没有新增价值，立即重审选题。
5. 把论文主张写成可证伪问题：**在什么故障场景里，即使基线已经使用幂等键，RAGuard 仍有真实、可量化的增益？**

### 最终判定

**我更推荐：论文主线 =“不确定提交下的外部副作用安全恢复”，交叉增强 =“分布式事务 + 故障注入/可验证状态机”。** 这比“Agent 网关自己修复错误”的泛化宣传更有明确的问题定义和实验切入点。建议优先面向 **JNCA 或 FGCS** 打磨；如果研究更偏软件工程不变式和经验研究，则选 **JSS**。期刊都应在真正投稿日再次确认 **SCIE、Subscription 费用和官方统计时间**。

---

## 10. 检索与准确性备注

- 资料检索时间 **2026-10-20**；研究文献中的数值是对应论文作者自述，不作为本项目预期结果。
- 39 篇均有 DOI、arXiv 或可直接定位的报告入口；但**尚未逐篇全文复现、并未人工确认每篇的后续正式出版版本**。
- ScienceDirect Insights 是**动态统计**，这个文件保留的是检索时官网可见数值；之后若数值变化，以投稿日官网为准。
- **免费 ≠ 开放获取**；此次限制的是**作者无需支付 APC 的订阅发表路线**，而不是让任何读者永久免费阅读。
- 选择期刊前至少复核 [MJL](https://mjl.clarivate.com/) 上的 **SCIE** 收录和学校认可口径；出版社网站/第三方分区数据库可能更新不同步。
- 本研究包是**研究规划和文献线索**，不构成录用承诺，也不代替导师对创新性与伦理的判断。
