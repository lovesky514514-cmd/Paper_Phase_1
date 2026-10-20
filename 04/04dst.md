# 04dst.md — 第四篇评估：DAG-FlowGate（多 Agent 工作流调度与拥塞控制）

> **评估对象**：`C:\Users\20050224\Desktop\04\`（立项书 `04.md` + 代码包 `04code.zip`）
> **评估方式**：**只读**。已读 51,881 字节立项书 + 全部 10 个 Python 文件 + 6 个 Go 文件 + 测试 + 完整 `results/sample_run.json`（1,051 行）。
> **未执行**：没有跑 `python -m benchmarks.run_eval`。文中所有数字（FCFS 0.428、FlowGate 0.317 等）来自项目自带的 `results/sample_run.json` 与 `docs/QUALITY.md`。

---

## 0. 一页速览

| 维度 | 评分（1–5） | 一句话依据 |
|---|:---:|---|
| 选题价值与可证伪性 | **3.5** | "共享外部配额下的多租户 DAG 拥塞"是真问题；但 Autellix/Kairos/Maestro/H-MAS 等直接对手密集，立项书自己也承认 |
| 立项文档质量 | **4.0** | 12 周路线、9 组基线、8 条消融、8 轴负载矩阵都写了；比第三、五篇略粗，但可用 |
| 原型实现完成度 | **3.5** | **2×2 因子消融（公平 × 反馈）是全系列设计得最漂亮的一处**；但基线与控制器有明显缺陷 |
| 实验证据强度 | **1.0** | **自家数据：FlowGate 完成率 0.317，FCFS 0.428**——主方法显著更差，且负载未标定，结论不可用 |
| 工程与仓库质量 | **3.0** | 可复现性好、有配对 bootstrap、区分"仅成功 p95"与"全量 p95"；扣分在 Go/Python 双实现不一致、32 个生成页 |
| **总分** | **15 / 25** | 五份里排第 5 |

**一句话结论**：**这是五份里"框架搭得最像论文、结论最难看"的一份**。它的负结果不是调参不到位，而是**控制器被写成了只会刹车不会加速的不对称 AIMD**，再叠加**从未标定的过载负载**，导致"拥塞控制"必然变成"自我饿死"。

**如果只能改三件事**：

1. **重新标定负载**（加 `--utilization`，从 0.3 到 1.5），现在实验全程在 1.1–1.6 倍持续过载里跑。
2. **修控制器**（对称增、去掉 `window≤3` 的硬上限、用"429 比例 + 排队延迟"做主信号），现在它只会衰减。
3. **把 goodput 的分母从"最大截止时间"改成真实观测窗口**，否则跨实验不可比。

---

## 1. 项目构成

| 层 | 内容 | 规模 |
|---|---|---|
| 立项书 | `04.md`，与包内 `research/04_original.md` 逐字节相同（51,881 B） | 675 行 |
| 仿真器 | `sim/engine.py`（离散事件循环）、`sim/model.py`、`sim/policy.py`（9 种排序）、`sim/workload.py`（DAG 生成） | 约 412 行 |
| 评测 | `benchmarks/run_eval.py`、`benchmarks/stats.py`（配对 bootstrap） | 约 93 行 |
| Go 网关 | `gateway/scheduler.go`（325 行）、`gateway/http.go`、`gateway/model.go`、`cmd/flowgate/main.go` | 约 528 行 |
| 测试 | `tests/test_sim.py`（117 行）+ `gateway/*_test.go`（119 行） | 约 236 行 |
| 文档 | `README / DESIGN / BENCHMARK / CODE_STANDARDS / QUALITY / DEBUG_INDEX / troubleshooting / GLOSSARY` + **32 个生成页** | 41 个 md |

**两条验证路径**：Python 离散事件仿真（产出所有量化结果）+ Go 单进程 HTTP 调度器（只跑一条 3 节点 demo）。

---

## 2. 优点：它真的做对了什么

### 2.1 2×2 因子消融设计得干净利落

`sim/policy.py:29-47` 的 `rank()` 里：

```python
if policy == "critical" or policy == "dag_feedback":
    return (slack, arrival, job, node)        # 只用关键路径松弛
...
return (debt_order, slack, arrival, job, node)  # dag_fair 和 flowgate 都落到这里
```

而 `sim/engine.py:192-197`、`:237-241` 只在 `policy in {"flowgate", "dag_feedback"}` 时启用反馈。

于是形成一个**干净的 2×2**：

|  | 无反馈 | 有反馈 |
|---|---|---|
| **无公平** | `critical` | `dag_feedback` |
| **有公平** | `dag_fair` | `flowgate` |

这是一个**正确的因子设计**——能回答"增益来自公平还是来自反馈"。绝大多数原型只会做"我的方法 vs FCFS"。

### 2.2 "已发出的调用不可撤回"被真的建模了

`engine.py:274-283`：仿真结束后，仍然在飞的调用会以 `late_finish` / `late_unknown` 事件被记录，**但不会把已失败的 job 改回成功**。

这对应立项书 §2.2 那条边界（"不能神奇地中断第三方正在运行的黑箱模型"），**在仿真里是有成本的**：任务失败后副作用可能仍在发生。这一点很少有人愿意建模，因为它会让结果变难看。

### 2.3 指标口径比同类原型严谨

`engine.py:88-107` 同时报：

- `p95_ms`（**仅成功任务**）与 `p95_all_jobs_ms`（**含失败与超时**）——正好避开立项书 §6.3 警告的"只统计被放行请求"的生存者偏差；
- `retry_amplification = attempts / logical_calls`（`:100`）——立项书 §6.3 要求的主指标之一；
- `fairness_jain_service`、`worst_tenant_completion`、`p99_ready_to_start_or_stop_ms`（`:101-105`）。

**这是一份"审稿人会想要什么"的清单，不是"代码方便算什么"。**

### 2.4 统计方法到位

`benchmarks/stats.py:6-17` 的 `paired_bootstrap_ci()`：

- 用**同一个 trace 种子**做配对差值再 bootstrap，不是独立均值比较；
- `run_eval.py:40-43` 明确注释"comparisons use identical trace seeds, not independent means"。

配对比较是立项书 §6.5 的硬要求，这里落实了。

### 2.5 Go 端有一条正确的"不盲重试"门槛

`gateway/scheduler.go:300-320`：

```go
if status == 429 {
    q.window = max(1, q.window/2)
    q.cooldownUntil = time.Now().Add(retryAfter)
    if !n.RetrySafe { t.status = "failed"; t.reason = "unsafe_retry"; return }
    if t.attempts[n.ID] > 2 { t.status="failed"; t.reason="retry_budget"; return }
    ...
}
if status != 200 {
    // Timeout/500 can be ambiguous for non-idempotent tools. Never blind retry.
    t.status = "failed"; t.reason = "unverified_tool_result"
}
```

**在调度器里就把"不可逆节点失败"和"可重试节点失败"分开**——这恰好是第一篇（01）的核心论点，在这里被当成了硬约束。

### 2.6 `QUALITY.md` 主动贴出负结果

`QUALITY.md:8-17` 直接给表，并写：

> **观察：** 拥塞反馈降低 429，但过于保守，造成 Goodput/完成率退化；下一阶段要调参、改进策略并和强基线公平比较。不要以节省错误码次数替代业务效果。

**"不要以节省错误码次数替代业务效果"这句话，恰好点出了这份工作的全部问题。** 作者知道，只是还没解决。

---

## 3. 缺点：问题在哪

### 3.1 最致命：主方法在全系列里是唯一"显著更差"的

`results/sample_run.json:14-67`（60 工作流 × 3 种子 × 默认参数）：

| 策略 | goodput/秒 | 完成率 | 429 次数 | 最差租户完成率 |
|---|---:|---:|---:|---:|
| **fcfs** | **34.46** | **0.428** | 49.0 | 0.333 |
| srpt | 33.68 | 0.417 | 50.3 | 0.350 |
| edf | 32.57 | 0.406 | 53.7 | 0.283 |
| critical | 31.75 | 0.394 | 54.3 | **0.350** |
| dag_fair | 31.39 | 0.389 | 52.0 | 0.250 |
| dag_feedback | 23.02 | 0.289 | 6.0 | 0.167 |
| **flowgate** | **25.37** | **0.317** | **5.3** | 0.217 |

三件事必须一起读：

1. **FlowGate 比 FCFS 差 26%（完成率）**，比 `dag_fair` 也差——即"加上反馈"这一步是**净负收益**；
2. 它唯一的改善是把 429 从 49 降到 5.3 —— 而这恰恰是 `QUALITY.md:17` 自己说的"不要以节省错误码次数替代业务效果"；
3. `critical`（只用关键路径）的**最差租户完成率反而最高**（0.350），说明"公平"这一维在本实验里也没有兑现。

**这是五份里唯一一份"主方法排在倒数第一"的。**

### 3.2 病因可以在代码里定位：控制器只会刹车

`engine.py:237-241`（本地 429 路径）：

```python
if self.policy in {"flowgate", "dag_feedback"}:
    window[r] = max(1, window[r] // 2)        # 窗口减半
    cooldown[r] = t + s.cooldown_ms           # 冷却 28ms
    recovered[r] = 0
    adaptive_rate[r] = max(.04, adaptive_rate[r] * .65)   # 速率乘 0.65
```

`engine.py:192-197`（恢复路径）：

```python
adaptive_rate[r] = min(s.local_rate, adaptive_rate[r] + .004 * k)   # 加性 +0.004
if recovered[r] >= 5 and window[r] < s.slots:                      # 要连续 5 次
    window[r] += 1                                                  # 才 +1
```

对比一下量级：

- 一次 429：速率 **×0.65**（−35%），窗口 **÷2**；
- 恢复一次窗口：需要 **5 次成功**，且上限就是 `slots=3`；
- 恢复一次速率：每次 +0.004，从 0.04 回到 0.34 需要 **75 次**。

**这是极端不对称的 AIMD。** 在持续过载下它会锁死在 `max(1, ...)` 和 `0.04` 附近——这就解释了为什么 429 少了 9 倍，但完成率也掉了 26%：**它不是"控制拥塞"，是"主动降速到几乎不接单"**。

而且 `window` 的上限就是初始值 3（`Settings.slots=3`），**从来没有"加速"的可能**。

### 3.3 负载从未标定，实验全程在过载区

这是让 3.1 的结果无法解释为"方法好坏"的根本原因。

- `workload.py:19`：平均到达间隔 `expovariate(1/11)` ≈ **11 ms**；
- `workload.py:23-27`：每节点 `units = randint(1,3)`（均值 2），节点时长 tool 5–18ms / llm 9–28ms；
- mixed 模式平均：每 job **2.33 个 tool 节点 + 1.67 个 llm 节点** → tool 需 4.67 units、llm 需 3.33 units；
- `engine.py:12-15`：`external_rate = 0.27 units/ms`（每资源独立），`slots = 3`。

**稳态需求 vs 容量**：

| 资源 | 需求（units/ms） | 名义容量 | 过载倍数 |
|---|---:|---:|---:|
| tool | 4.67 / 11 ≈ 0.425 | 0.27 | **≈1.57×** |
| llm | 3.33 / 11 ≈ 0.303 | 0.27 | ≈1.12× |

再加上 `slots=3` 的并发上限（每个资源同时在飞 ≤3），系统**从第 0 秒起就处于排队发散状态**。经验证据：连 FCFS 也只完成 42.8%，`worst_tenant_completion` 全线 0.17–0.35。

**所以这张表测的不是"deadline 感知调度好不好"，而是"在 1.5 倍永久过载下，哪个策略把哪批任务丢掉"。** 在这种工况下，"先接单再超时"的 FCFS 天然占优，任何"降速省配额"的策略必然吃亏——结果是被构造出来的，不是被发现的。

立项书 §6.4 要求 8 轴负载矩阵，第一轴就是"负载强度 0.3/0.6/0.9/1.1/1.5 × 标定系统容量"。**代码里既没有容量标定，也没有负载强度参数。**

### 3.4 goodput 的分母是"最大截止时间"，不是观测窗口

`engine.py:133`：`horizon = max(job.deadline_ms for job in trace.jobs)`
`engine.py:91`：`goodput_per_second = on_time / (horizon_ms / 1000)`

而 `workload.py:41`：`deadline = t + sum(节点时长) × U(1.7, 3.4)`

于是：

- horizon ≈ 最后一个到达时刻 + 该 job 的 deadline 偏移；
- **把 deadline 倍数从 2.55 调到 5.0，同一个成功数会让 goodput 几乎腰斩**；
- 不同 pattern / 不同 seed 的 horizon 不同 → **goodput 跨实验不可比**。

正确做法：分母用 `last_completion − first_arrival`，或固定 warmup + 测量窗。现在这个定义让"吞吐类指标"混进了"截止时间参数"，读者无法分辨性能变化来自哪一边。

### 3.5 两个基线名不副实

**(1) `srpt` 不是 SRPT，是 SPT。**
`policy.py:39`：

```python
return (sum(n.estimate() for n in job.nodes if n.id not in path or path[n.id] > 0), ...)
```

`path` 是"每个节点的剩余最长后缀"，只要节点没完成它就 >0，所以筛选条件**近似恒真** → 它等于"整个 job 的估计总时长"，**进度信息被完全丢弃**。SRPT（最短剩余处理时间）的核心恰恰是用进度。这条基线在任何论文里都不成立。

**(2) `edf` 退化成 FCFS。**
`workload.py:41` 把 deadline 定义为 `arrival + 自身时长 × U(1.7,3.4)`，于是绝对 deadline 与到达顺序基本同序。数据也印证：EDF 0.406 vs FCFS 0.428，**EDF 更差**——这在真正的 EDF 下几乎不可能。

结果：论文里"我们比 EDF/SRPT 强"这句话，现在测的是一个假 EDF 和一个假 SRPT。

### 3.6 关键路径在 FlowGate 里几乎没起作用

`policy.py:47`：`return (debt_order, slack, arrival, job, node)`

**第一排序键是"租户已服务量/权重"**（公平），slack 只是第二键。所以：

- "依赖感知/关键路径"这个卖点，在 FlowGate 的排序里被公平性彻底压住；
- 数据印证：`dag_fair`（同样的排序键、无反馈）完成率 0.389 **高于** flowgate 的 0.317 → 反馈是纯粹负贡献；
- 而 `critical`（纯 slack）完成率 0.394，也比 dag_fair 高 → **公平性也是负贡献**。

也就是说：**在这个负载配置下，"公平 + 关键路径 + 反馈"三个部件没有一个带来正收益**，而论文的三条创新声明正好就是这三个部件。这不是"没调好参"，是**实验工况与主张不匹配**。

### 3.7 Go 与 Python 是两套实现，量化结果只来自 Python

`README.md:22` 自认：

> Go 和 Python 是两条独立的验证路径，**算法细节不完全一致**。

具体差异（对照阅读后）：

| 维度 | Python `sim` | Go `scheduler` |
|---|---|---|
| 排序键 | `(debt_order, slack, ...)` | `(service/weight, slack, ...)` |
| 配额恢复 | 加性 `+0.004`、窗口 +1/5 次成功 | 加性 `+0.34*elapsed`、窗口 +1/5 次成功但上限 3 |
| 429 衰减 | `rate ×0.65`、`window //2` | `window //2`（无速率衰减） |
| 退避 | `max(retry_after, 2^n × 5) + jitter` | `2^attempts × max(retryAfter, 10ms)` |
| 时钟 | 离散事件，`t` 为整数毫秒 | 真实 `time.Now()`，3ms ticker |

而 `scripts/demo.py` 只提交**一条 3 节点链**做冒烟测试。**所以"系统"和"结果"是两件互不验证的事。** 对 FGCS / Computer Communications 这类系统刊，这是硬伤——审稿人会要求"仿真结论在你的实际系统上复现"。

### 3.8 其他问题

| # | 问题 | 证据 |
|---|---|---|
| 1 | `retry_safe` 由节点名硬编码推导（`name != "write"`），不是从工具契约来 | `workload.py:27` |
| 2 | `unsafe_retry` 只把 job 标失败，**不检查外部状态是否已被破坏**——"不安全"没有可观测的后果 | `engine.py:172-174`、`:242-243` |
| 3 | `lost_response` 与 `http_5xx` 被同等对待（都进重试/失败分支），而前者语义上正是"可能已提交" | `engine.py:167-180` vs `:262-266` |
| 4 | `deadline` 被当作**绝对时刻**（`engine.py:199` 的 `t >= job.deadline_ms`），但 `model.py` 里没有写清这个约定 | `engine.py:199` |
| 5 | `Settings.shift_ms = 350`、`shifted_fraction = 0.55`（配额突降）是硬编码，无法命令行配置——立项书 §6.4 的"配额突变"轴做不了扫描 | `engine.py:16-17` |
| 6 | 只有 3 个种子；立项书要求 ≥10 | `run_eval.py:58` |
| 7 | `docs/CODE_STANDARDS.md:3` 留着"参考用户提供的 Skill：`codebase-design`、`domain-modeling`、`tdd`、`code-review`"——**生成过程的痕迹不该进交付物** | `CODE_STANDARDS.md:3` |

---

## 4. 项目自己已经承认的负结果与红线

1. **主方法更差**：`QUALITY.md:8-17`，FlowGate 0.317 vs FCFS 0.428。
2. **"不要以节省错误码次数替代业务效果"**：`QUALITY.md:17`。
3. **"WFQ 是轻量代理，不是 DRF 的忠实复现"**：`run_eval.py:49`、`BENCHMARK.md:10`。
4. **"Go 与 Python 算法细节不完全一致"**：`README.md:22`。
5. **"合成虚拟延迟不能代表线上 P95"**：`QUALITY.md:19`。
6. **"尚缺 Autellix / Kairos / 真实 DRF 的复现"**：`BENCHMARK.md:20-27`。

**这份自检报告的质量高于它自己的实验结果。** 建议保留原文，它本身就是 Threats to Validity 一节的草稿。

---

## 5. 改进清单

### P0 — 不做这些，实验结论无意义

| # | 要做什么 | 为什么 | 验收标准 |
|---|---|---|---|
| P0-1 | **负载标定**：加 `--utilization 0.3/0.6/0.9/1.1/1.5`，按 `external_rate` 与平均节点需求反推到达率；先写一个"容量=1.0 的定义与测量脚本" | 现在全程 1.1–1.6 倍过载，所有策略都在弃流 | 在 0.6 负载下所有策略 completion > 0.9；在 1.1 负载下才允许出现差异；论文主图用 0.9 / 1.1 两档 |
| P0-2 | **修控制器**：① 主信号换成"429 比例 + p95 排队延迟"而不是"外部配额差额"；② AIMD 改成对称（或 MIMD）；③ **去掉 `window ≤ 3` 的硬上限**；④ 冷却结束后快速恢复（类似 slow-start） | 现在只会刹车（见 3.2） | 在 0.9 与 1.1 负载 + 配额突降 30%/60% 下，FlowGate 的 completion **不低于** FCFS，且 429 至少低 3 倍 |
| P0-3 | **goodput 分母改成真实观测窗口**：`last_completion − first_arrival`，或固定 warmup + 测量窗；同时**单独**报 throughput 与 completion | 现在分母是 `max(deadline)`，跨实验不可比（见 3.4） | 同一 trace 上把 deadline 倍数从 2.55 改到 5.0，goodput 不应整体缩放 |
| P0-4 | **让关键路径进第一排序键**：先按"能否赶上 deadline"分级，级内再按 debt；并新增一个 `deadline_first` 消融 | 现在 `slack` 是第二键，"依赖感知"在结果里看不出来（见 3.6） | 至少一档负载下 `critical` 或新消融显著优于 `dag_fair`，否则删掉"关键路径"这条创新声明 |

### P1 — 决定论文能不能进系统刊

| # | 要做什么 | 验收标准 |
|---|---|---|
| P1-1 | **修基线**：`srpt` 必须跟踪剩余工作量；`edf` 的 deadline 改成与自身时长无关的相对松弛（如 `arrival + U(500,1500)`）；补一个真 DRF 近似 | 修完后 EDF 在低负载下应优于 FCFS；否则说明 workload 设计还有问题 |
| P1-2 | **合并两条验证路径**：Python 只做 trace 生成 + 指标统计，调度决策由 Go 通过 HTTP 暴露（或反过来，Go 只做重放）。二者择一，不要并存 | 论文里"系统"与"结果"是同一个实现；任何一张图都能由 Go 端复现 |
| P1-3 | 把 `shift_ms / shifted_fraction / slots / bucket` 全部提到命令行，扫描"配额突变"轴 | 图 4（并发 × P95 / SLA 违反）可以自动生成 |
| P1-4 | 种子 3 → ≥10；报告效应量与配对 CI 的**具体数值**表（不只是区间） | 每个结论后面能贴一个 p 值或 CI |
| P1-5 | `unsafe_retry` 要有可观测后果：记录"外部状态是否被破坏"（重复写/丢写），并纳入 unsafe 指标 | 从"任务失败"升级为"任务失败 + 状态是否安全"，才配得上和第一篇对话 |

### P2 — 可读性与可信度

| # | 要做什么 |
|---|---|
| P2-1 | 删 `CODE_STANDARDS.md` 里的生成过程痕迹（"参考用户提供的 Skill…"） |
| P2-2 | 32 个 `docs/files/` 生成页删掉；`research/04_original.md` 改为指针 |
| P2-3 | `model.py` 里给 `deadline_ms` 加一行明确注释："绝对时刻（= 到达时刻 + 相对松弛）" |
| P2-4 | 区分 `lost_response` 与 `http_5xx` 的处理分支（前者应进入"不确定提交"语义），哪怕只在注释里说明为什么现在合并 |

---

## 6. 只改五件事的话

1. 标定负载（`--utilization`）→ 让"谁更好"这个问题有意义。
2. 修控制器（对称 AIMD + 去上限）→ 让 FlowGate 至少不输给 FCFS。
3. 修 goodput 定义 → 让数字可比。
4. 修 srpt / edf 两个基线 → 让"我们更强"有对手。
5. 合并 Go 与 Python → 让"系统"和"结果"对得上。

---

## 7. 在全系列里的定位建议

| 关系 | 说明 |
|---|---|
| 与 01（恢复） | 04 里 `retry_safe` 的静态布尔，正是 01 想用"效果账本"替换掉的东西；两篇可以互相引用但不要合并 |
| 与 02（路由） | 02 的 `Cost` 结构应该被 04 复用（现在 04 只有 `units`，没有美元） |
| 与 03（授权） | 无关度高，只共享"版本化配额"的思想 |
| 与 05（根因） | 04 的"429 重试放大"是 05 的一类标准故障模式，可以作为 05 的数据源 |

**推荐定位**：**五份里最应该"暂停、先重做实验"的一份。** 它的代码框架、消融设计和自检报告都不错，但**结论是负的，而且负得没有解释力**（因为负载未标定）。我的建议顺序是：

1. 先做 P0-1 + P0-3（半天到一天的工作量），看结果是否反转；
2. 若反转 → 继续 P0-2、P0-4，这个题目仍有希望投 FGCS / Computer Communications；
3. 若不反转 → **诚实地把它写成一篇"负结果 + 方法学"论文**（"为什么在过载 agent 工作流上，公平 + 拥塞反馈会降低 goodput"），这对社区是有价值的，而且立项书 §7 的 Stop 条件本来就这么写的。

**风险提示**：立项书 §12.2 的风险表里，最高级的两条是"直接竞争论文密集"和"只做模拟、系统实验不足"。**这两条在当前代码里都成立**：Go 端没有任何量化结果，Go/Python 还不一致。如果决定继续做这个题目，**"把 Go 端变成唯一结果来源"应该比"再调一版控制器"优先**。

---

## 附录：文件地图

```text
04.md                            ← 立项书（675 行），§3 撞题表、§6 实验方案最值得读
04code/
├─ README.md                     ← 24 行，限制写明"Go/Python 算法不完全一致"
├─ docs/QUALITY.md               ← 19 行，**负结果表 + "不要以 429 替代业务效果"**
├─ docs/BENCHMARK.md             ← 27 行，实验协议与"尚缺实验"
├─ sim/engine.py                 ← 284 行，**离散事件循环 + 反馈控制**（病因所在）
├─ sim/policy.py                 ← 47 行，**9 种策略的排序键**（2×2 消融在此）
├─ sim/workload.py               ← 46 行，DAG 生成（到达率、deadline、retry_safe）
├─ benchmarks/run_eval.py        ← 84 行，配对 bootstrap 与 CLI
├─ gateway/scheduler.go          ← 341 行，Go 单进程调度器（注意与 sim 的差异）
└─ results/sample_run.json       ← 1,051 行，**所有量化结论的唯一来源**
```

**阅读顺序建议**：`results/sample_run.json` 的 `aggregated` 段（先看结论有多难看）→ `sim/policy.py`（看 2×2 设计）→ `sim/engine.py` 的 `:192-197` 与 `:237-241`（看病因）→ `sim/workload.py`（看负载量级）→ `QUALITY.md`。
