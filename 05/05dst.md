# 05dst.md — 第五篇评估：TraceLite-RCA（预算约束下的根因定位与弃权）

> **评估对象**：`C:\Users\20050224\Desktop\05\`（立项书 `05.md` + 代码包 `05code.zip`）
> **评估方式**：**只读**。已读 49,812 字节立项书 + 全部 14 个 Python 文件 + 6 个 Go 文件 + 131 行测试 + 完整 `results/sample_run.json`（4,768 行）。
> **未执行**：没有跑 `python -m benchmarks.run_eval`。文中所有数字（adaptive 58.3%、ECE 0.36 等）来自项目自带的 `results/sample_run.json` 与 `docs/QUALITY.md`。

---

## 0. 一页速览

| 维度 | 评分（1–5） | 一句话依据 |
|---|:---:|---|
| 选题价值与可证伪性 | **4.0** | "部分可观测 + 采样预算 + 可弃权"是 05 相对 AgentRx/AgentTrace/CORRECT 的真差异化角度 |
| 立项文档质量 | **4.5** | 12 篇直接对手逐个列"不能重复宣称什么"、8 类故障模型、45 篇文献、6 个失败原因自查 |
| 原型实现完成度 | **3.5** | 观测契约把 oracle 严格隔离、预算用真实字节记账、弃权/覆盖率/准确率分离——设计很清醒 |
| 实验证据强度 | **1.5** | **adaptive Top-1 58.3% = first_error 58.3%**；ECE 0.36 却是所有方法里最差的；基准被构造成"找唯一 fail 的 span" |
| 工程与仓库质量 | **3.5** | 隐私默认、模板级留出、配对 bootstrap 按 trace 重采样；扣分在字符串驱动逻辑与 36 个生成页 |
| **总分** | **17 / 25** | 五份里排第 3 |

**一句话结论**：**这是五份里"方法论自觉最强、被自己的 benchmark 骗得最深"的一份**。它的代码反复证明"我没有偷看答案"（而且真的做到了），但**它的数据集把答案写在了题面上**——所以"我比均匀采样更准"这个结论既真又无意义。

**如果只能改三件事**：

1. **换掉 benchmark**：让故障是"所有校验都通过、但语义已错"的（stale cache / 单位混淆），并让多个 span 都有 fail。现在 target 是**唯一**一个 `validation=fail` 的 span，定位任务是假的。
2. **禁止采样器使用 `status_code ≥ 500` 和"异常后代数"做优先级**——这两个特征与 label 高度相关，等于用答案选样本。
3. **把分数变成可校准的量**：现在的 softmax 是对一堆魔数（1.5/2.7/1.05）做归一化，"置信度"没有概率意义，温度缩放救不回来（ECE 0.36 就是证据）。

---

## 1. 项目构成

| 层 | 内容 | 规模 |
|---|---|---|
| 立项书 | `05.md`，与包内 `research/05_original.md` 逐字节相同（49,812 B） | 543 行 |
| 观测契约 | `tracelite/model.py`（`Span / TraceCase / Evidence / Observation / Candidate / Diagnosis`） | 73 行 |
| 采样 | `tracelite/observe.py`（adaptive / uniform / random / full / none，真实字节记账） | 63 行 |
| 诊断 | `tracelite/diagnose.py`（证据分级打分、softmax、弃权、温度拟合） | 103 行 |
| IO | `tracelite/io.py`（JSONL 读写与 trace 分组） | 30 行 |
| 评测 | `benchmarks/workload.py`（9 模板）、`run_eval.py`（9 策略）、`metrics.py`、`matrix.py` | 约 220 行 |
| Go 网关 | `gateway/proxy.go`（单层转发 + 脱敏 + 记录）、`record.go`、`cmd/tracelite`、`cmd/mocktool` | 约 360 行 |
| 测试 | `tests/test_rca.py`（131 行，24 例） | 131 行 |
| 文档 | `README / DESIGN / BENCHMARK / CODE_STANDARDS / QUALITY / DEBUG_INDEX / troubleshooting / GLOSSARY` + **36 个生成页** | 45 个 md |

---

## 2. 优点：它真的做对了什么

### 2.1 oracle 与观测在**类型层面**被分离，并且有测试守住

`model.py:27-36`：

```python
@dataclass(frozen=True)
class TraceCase:
    spans: tuple[Span, ...]
    first_irrecoverable: str | None      # ← 答案在这里
    intervention_roots: tuple[str, ...]

@dataclass(frozen=True)
class Observation:
    spans: tuple[Span, ...]
    evidence: tuple[Evidence, ...]
    used_bytes: int
    ...
```

**`Observation` 里根本没有答案字段。** 采样函数 `collect()` 只能拿到 `TraceCase`，返回 `Observation`。

更值得表扬的是 `tests/test_rca.py:86-97` 两条"防作弊"测试：

```python
def test_adaptive_selection_does_not_inspect_hidden_validation(self):
    case2 = replace(self.case, spans=tuple(replace(s, validation={'altered':'fail'}) ...))
    left  = collect(self.case, 320, seed=4)
    right = collect(case2,    320, seed=4)
    self.assertEqual([e.span_id for e in left.evidence], [e.span_id for e in right.evidence])

def test_adaptive_uses_no_oracle_labels(self):
    changed = replace(self.case, first_irrecoverable='notify', ...)
    ...
```

**"改了答案，采样结果必须不变"** —— 这是一条很硬的约束。绝大多数"自适应采样"论文的原型根本不会写这种测试。

### 2.2 采样预算用**真实 UTF-8 字节**记账

`observe.py:10-13`：

```python
def detail_size(span):
    payload = {'span_id': span.span_id, 'validation': span.validation}
    return len(json.dumps(payload, ensure_ascii=False, separators=(',', ':'), sort_keys=True).encode())
```

配合 `metrics.py:41` 报 `mean_evidence_bytes`。这比"我采集了 N 条 span"要诚实得多——因为**不同 span 的证据长度不同**，按条数记账会让"预算"失去意义。

（不过这个机制本身有个反效果，见 3.6。）

### 2.3 指标口径把"准确率 / 覆盖率 / 弃权"彻底拆开

`metrics.py:33-45`：

```python
'cases', 'fault_cases',
'top1_all',                    # 全故障样本的 Top-1
'top3',
'coverage',                    # 非弃权比例
'answered_accuracy',           # 只在非弃权样本上算
'unsafe_false_report_count',   # 对"健康"样本却给出预测的次数
'mean_evidence_bytes',
'p95_diagnosis_us_offline',
'ece_top1', 'brier_top1'
```

**这套口径直接堵死了"全部弃权拿高分"这条路**（立项书 §6.2 明确警告的作弊方式）。而且 `unsafe_false_report_count` 这个指标很聪明——它统计"本来是健康轨迹却报出根因"的误报。

### 2.4 配对 bootstrap 按 **trace** 重采样，不是按 span

`metrics.py:48-60`：

```python
left  = {r['trace_id']: r for r in a if r['truth'] is not None}
right = {r['trace_id']: r for r in b if r['truth'] is not None}
keys  = sorted(left.keys() & right.keys())     # ← 按 trace 对齐
diffs = [int(left[k]['top1']==left[k]['truth']) - int(right[k]['top1']==right[k]['truth']) for k in keys]
```

立项书 §6.5 的原话是"重复运行不能把同一轨迹重复算成独立样本"。**这条在代码里被强制了。**

### 2.5 留出是"模板级"的，不是随机切分

`run_eval.py:54-57`：

```python
train_templates   = TEMPLATES[:3]     # stale_cache, schema_change, rate_limit
valid_templates   = TEMPLATES[3:5]    # agent_plan, message_order
test_templates    = TEMPLATES[5:]     # permission, duplicate_write, network_timeout, healthy
```

同一模板不会同时出现在验证和测试里。这是**防止"学到注入模板而不是根因"**的正确做法（立项书 §6.4 明确要求）。

### 2.6 隐私默认值到位

`gateway/proxy.go`：

- 只转发到预配置 upstream，调用方**不能指定 host**（`:27-36`）；
- 出站白名单头，**不转发 `Authorization` / `Cookie`**（`:92-95`）；
- 每条 span 固定打上 `observability_mask = ["body_redacted", "reasoning_hidden"]`（`:122-124`）；
- `/healthz` 只暴露"接受/丢弃的 span 数"（`:64-69`）。

### 2.7 `QUALITY.md` 是全系列最诚实的一份

`QUALITY.md:7-13` 直接给表，并写：

> **本方法未超过 First Error 的 Top-1；低覆盖率影响实用性；不能作为论文可发表结论。**

`QUALITY.md:20` 还主动声明"历史成本只包含可选证据 JSON 摘要，**不包含基础元信息**；不是网关全成本"。

**这两句话让它免于被误读。** 但也正因为如此，它把"现在还不能投稿"这个结论摆在了最前面——这是好事，只是需要被正视。

---

## 3. 缺点：问题在哪

### 3.1 最致命：主方法打平了一个 20 行的启发式

`results/sample_run.json:26-120`（种子 17、每模板 8 条、200 字节预算、20% 缺失）：

| 策略 | Top-1 | Top-3 | 覆盖率 | 非弃权准确率 | 证据字节 | ECE |
|---|---:|---:|---:|---:|---:|---:|
| last_error | 4.2% | 70.8% | 91.7% | 4.6% | 0 | — |
| **first_error** | **58.3%** | 70.8% | 91.7% | 63.6% | **0** | — |
| status_only | 58.3% | 70.8% | 91.7% | 63.6% | **0** | — |
| uniform | 41.7% | 70.8% | 33.3% | 75.0% | 74.8 | 0.269 |
| random | 54.2% | 70.8% | 33.3% | 75.0% | 73.8 | 0.329 |
| **adaptive（本方法）** | **58.3%** | 70.8% | **33.3%** | 75.0% | 81.1 | **0.359** |
| no_graph（消融） | 25.0% | 70.8% | 33.3% | 62.5% | 81.1 | **0.049** |
| full_capture | 70.8% | — | 58.3% | 85.7% | 354.0 | — |

三件事必须一起读：

1. **`adaptive` 的 Top-1（58.3%）与 `first_error`（58.3%）完全相同**，而后者**花 0 字节、覆盖率 91.7%**。也就是说：**花了 81 字节、丢了 2/3 的覆盖度，准确率一模一样。**
2. **校准也失败了**：`adaptive` 的 ECE 0.359 是除 `random` 外最差的；而**消融版 `no_graph` 的 ECE 只有 0.049**（好 7 倍）。"校准"这个贡献在实践中反向。
3. `full_capture`（70.8%）说明**信息本身是有用的**——瓶颈在采样策略，不在诊断器。这个结论很重要：它意味着 P0 级改进应该放在采样侧。

**这就是"负结果"的确切形状**：不是方法完全无效，而是**它的两个卖点（省字节、可校准）分别被"打平"和"反向"打掉了**。

### 3.2 benchmark 把答案写在了题面上

这是让 3.1 的数字失去解释力的根本原因。

`benchmarks/workload.py:21-25`：每个模板固定 **6 段线性链**，初始化时**所有 span 的 `validation` 都是 `{'schema':'pass','freshness':'pass'}`**。

`:39-45`：只有**故障那一个 span** 被改成 `{check: 'fail'}`：

```python
old = spans[root_index]
checks = {**old.validation, check: 'fail'}
spans[root_index] = replace(old, status_code=status, ..., validation=checks)
```

而 `diagnose.py:43-50` 给"有 fail 的直接证据"加 **+2.7**，其他证据的贡献是 1.0 / 0.5 / 1.05×3。

**所以：只要采样到了正确 span，Top-1 必然正确；采样没到，必然错误。** 这个 benchmark 实际测的是**采样命中率**，不是"根因定位能力"。

连带后果：

- 图证据（`+1.05 × 异常后代数`，`diagnose.py:57-61`）只在"没采到证据"时才有用——所以 `no_graph` 掉到 25% 这个"图很重要"的结论，其实只是"图在信息缺失时提供了一点排序信号"；
- 证据分级（直接验证 / 有记录支持 / 时间猜测）在数据里从未出现"冲突证据"，所以分级机制没有被真正考验；
- "弃权"只在"没采到证据"时触发，不是在"证据互相矛盾"时触发。

### 3.3 采样优先级用了与 label 强相关的特征

`observe.py:38-40`：

```python
ordered = sorted(available, key=lambda s: (
    -((2 if s.abnormal else 0) + descendants(s) * .9 + (1 if s.status_code >= 500 else 0)) / 160,
    s.time_ns, s.span_id))
```

而 `model.py:22-24`：`abnormal = status_code >= 400 or latency_ms >= 1200`。

再看测试集的三个故障模板（`workload.py:26-35`）：

| 模板 | 故障 span | 状态码 | 是否 `abnormal` |
|---|---|---:|---|
| permission | commit | 403 | ✅ |
| duplicate_write | commit | 409 | ✅ |
| network_timeout | fetch | 503 + 1400ms | ✅ |

**测试集的故障 span 100% 是 `abnormal` 的**，而 `adaptive` 的第一优先级就是"是否 abnormal"。所以"adaptive 能采到答案"是**构造出来的**，不是"证据驱动的"。

更扎心的是：真正难的两个模板 —— `agent_plan`（`interpret`，状态码 **200**，只有 `plan=fail`）和 `message_order`（`notify`，状态码 **200**）—— 被放在了 **validation 集**，而 `stale_cache`（`fetch`，200，只 `freshness=fail`）和 `schema_change`（`interpret`，200，只 `schema=fail`）被放在 **train 集且从未使用**。

**换句话说：立项书 §1.2 讲的那个"库存快照已过期，但所有 HTTP 都是 200"的核心案例（stale_cache），在测试集里根本不存在。**

### 3.4 温度拟合撞在网格上界，而且只在 2 个模板上拟合

`diagnose.py:95`：候选网格是 `(.4, .7, 1., 1.5, 2., 3., 5., 8.)`。

`results/sample_run.json:8`：`"temperature_validation_only": 8.0` —— **取到了网格最大值**，说明最优点在网格外，拟合没收敛。

而验证集是 `make_dataset(seed+1, 8, ('agent_plan','message_order'))` = **16 条样本**，其中还有 healthy 之外的…实际上 `pairs` 只保留 `first_irrecoverable` 非空的，即 16 条。

用 16 条"难模板"样本去拟合温度，然后在"易模板"上测试 → **温度被推得极高（8.0），把 softmax 压平 → 置信度趋近 1/n → 弃权逻辑（阈值 0.25）在 n=6 时会大量触发**。这正好解释了 adaptive 的覆盖率只有 33%。

`test_rca.py:126-128` 那条测试只检查"温度落在那 8 个值里"，**并不检查拟合是否合理**——所以这个 bug 被测试保护着。

### 3.5 弃权判定靠**字符串匹配**

`diagnose.py:78-79`：

```python
has_direct_check = bool(top and any('Tool-reported check failed' in item and 'task' not in item for item in top.evidence))
has_symptom      = bool(top and any('Gateway observed' in item for item in top.evidence))
```

用**人类可读文案**当控制流的谓词。问题：

- 改一句提示语就改行为；
- `'task' not in item` 是子串匹配，任何包含 "task" 的证据文案都会被误伤（例如 "Task-level failure..."）；
- 无法被类型检查、无法被重构工具保护。

正确做法：`Evidence` 里加结构化字段 `kind: Literal['direct_check_failed','symptom','graph_support','missing']`，弃权逻辑只读 `kind`。

### 3.6 预算核算真实但**没对齐**

`observe.py:54-59`：

```python
slots = budget_bytes // 160          # 预算折算成"160 字节券"
for span in ordered[:slots]:
    cost = detail_size(span)
    if cost > 160: raise ValueError(...)
    chosen.append(Evidence(span.span_id, dict(span.validation), cost))
used = sum(item.byte_cost for item in chosen)
```

两个后果：

1. **预算 200 → 1 张券 → 最多采 1 个 span**。所以表里 `mean_evidence_bytes` 只有 74–81，而预算是 200 —— **预算的实际利用率不到 41%**，"同预算比较"变成了"同券数比较"。
2. `full` 策略也被同一个 `slots` 截断（`observe.py:47-48`），所以 `full` 的语义是"券足够多时的全采"，不是"真全采"。

结果：uniform / random / adaptive 三者的字节数（74.8 / 73.8 / 81.1）并不相同，"同预算下更准"这个主张的公平性没有被严格保证。

### 3.7 性能与规模

| # | 问题 | 证据 |
|---|---|---|
| 1 | `descendants()` 每个 span 跑一次 O(n) 迭代，排序键又对每个 span 调一次 → 每次 `collect()` 是 O(n³) 级 | `observe.py:33-40` |
| 2 | `p95_diagnosis_us_offline` 15–24 µs 是"对 6 个 span 的启发式打分"的耗时，和"网关 P95 增量 ≤5%"不是一回事 | `QUALITY.md:20` 已自认 |
| 3 | 轨迹只有 6 个 span、单链、无并发、无跨 Agent 交接；立项书说"跨 Agent/工具因果候选链"，代码是线性链 | `workload.py:17-25` |
| 4 | 网关（Go）与算法（Python）基本解耦，只有 `scripts/demo.py` 把它们接起来；没有"诊断挂在网关异步路径上"的实现 | `gateway/proxy.go` 全文 |
| 5 | 测试集只有 32 条（24 条故障），任何 1 条翻转是 4.2 个百分点 | `run_eval.py:58-62` |

### 3.8 文档层

- **36 个 `docs/files/` 生成页**（全系列最多）；`research/05_original.md` 是 49 KB 逐字节副本。
- `docs/CODE_STANDARDS.md` 同样留着生成痕迹。
- 立项书 543 行、45 篇文献，质量很高；但**代码只有 1,190 行**，文档/代码比是五份里最高的（45 个 md 文件）。

---

## 4. 项目自己已经承认的负结果与红线

1. **"本方法未超过 First Error 的 Top-1"**（`QUALITY.md:13`）。
2. **"低覆盖率影响实用性"**（`QUALITY.md:13`）。
3. **"未忠实复现 AgentRx / CORRECT 等公开方法"**（`README.md:21`）。
4. **"没有真实环境下的因果验证"**（`README.md:21`）。
5. **"历史成本只包含可选证据 JSON，不是网关全成本"**（`QUALITY.md:20`）。
6. **"Go 竞态测试只覆盖当前测试文件的并发路径"**（`QUALITY.md:19`）。

**第 1 条就是它最需要被正视的结论。** 建议把 `QUALITY.md` 这一页当作立项书的"当前状态"附录，避免读者以为主方法已经领先。

---

## 5. 改进清单

### P0 — 不做这些，"根因定位"这个题目不成立

| # | 要做什么 | 为什么 | 验收标准 |
|---|---|---|---|
| P0-1 | **换 benchmark**：让故障是"所有校验都 pass、但结果语义已错"（stale cache / 单位混淆 / 正确但过期的 ID），并让**多个 span 都有 fail**（下游症状 + 可恢复的前置异常）。`stale_cache` 必须进 test | 现在 target 是唯一的 fail span，定位任务是假的（见 3.2） | `first_error` 与 `status_only` 的 Top-1 **掉到 30% 以下**，而证据分级方法保持显著优势（配对 CI 不含 0） |
| P0-2 | **采样器禁用 label 相关特征**：`status_code ≥ 500` 和"异常后代数"必须移出优先级，或明确标注为"cheating 上界"单列一张表 | 现在测试集 100% 异常，等于用答案选样本（见 3.3） | `test_rca.py` 新增一条测试：把 target 的 status_code 改成 200，采样决策**不变** |
| P0-3 | **分数变成可校准的量**：把 `1.5/2.7/1.05` 换成"证据似然"——例如 logistic 回归 / 小 GBM，特征是 `evidence.kind`、异常类型、传播深度、采样完整度；再在验证集做温度 / 保序校准 | 现在 softmax 归一化一堆魔数，ECE 不可能好（见 3.1、3.4） | `ECE < 0.10` 且 Brier 有改善；`no_graph` 不再出现"ECE 更好但准确率更差"的反常 |
| P0-4 | **修温度拟合**：网格上界放到 50（或解析最优），验证集 ≥100 条，并断言"最优温度不在边界" | 现在取到 8.0 上界，等于没拟合（见 3.4） | 单测：拟合结果不在网格端点；若在端点则抛异常 |

### P1 — 决定论文能不能过审

| # | 要做什么 | 验收标准 |
|---|---|---|
| P1-1 | **预算对齐**：改成"按真实字节的贪心背包"（按 utility/byte 排序），uniform / random / adaptive 三者用**相同字节省量**跑满；报"每字节 Top-1" | 三种策略的 `mean_evidence_bytes` 差值 < 5%；主图 X 轴是"证据字节"，不是"券数" |
| P1-2 | **结构化弃权**：`Evidence.kind` 替代字符串匹配；阈值逻辑只读结构化字段 | 改任何证据文案不影响诊断结果（加一条测试） |
| P1-3 | **轨迹规模与拓扑**：6 → 20–60 个 span，加扇出/汇合/跨 Agent 交接；并对 `descendants()` 做一次拓扑预计算（O(n+e)） | 单条轨迹 60 span 时 `collect()` < 1 ms |
| P1-4 | **接真实网关**：把 `diagnose()` 挂到 Go 网关的异步路径，报真实 P95 增量与 QPS | 有一张"网关 P95 增量 vs 采样预算"的曲线，而不是离线 µs |
| P1-5 | **补强基线**：至少一个"单次 LLM 读全日志"（控制 token）和"PageRank/DAG 随机游走" | 立项书 §6.1 的 12 组基线里，至少 6 组真的跑起来 |
| P1-6 | 测试集 32 → ≥200 条，种子 3 → 5，报告每类模板的分解结果 | 每个模板单独一行，不要让"平均"掩盖 permission 模板的特殊性 |

### P2 — 可读性与长期维护

| # | 要做什么 |
|---|---|
| P2-1 | 36 个生成页删掉；`research/05_original.md` 改为指针；删 `CODE_STANDARDS.md` 的生成痕迹 |
| P2-2 | `train_templates` 现在是"定义了但从未使用"，要么真的用它做预训练/预筛，要么在 metadata 里标注"保留未用" |
| P2-3 | `matrix.py` 增加 `budget=0` 时的显式跳过（现在会产出"全弃权"点，混淆 Pareto 曲线） |
| P2-4 | 把 `QUALITY.md:7-13` 那张负结果表扩成 `docs/RESULTS.md`，附上每个模板的分解——这是后续迭代的基线快照 |

---

## 6. 只改五件事的话

1. 换 benchmark（stale cache / 多 fail span）→ 让"定位"成为真任务。
2. 采样器禁用 label 特征 → 让"adaptive 更好"不再循环论证。
3. 分数改成可校准似然 + 修温度 → 让 H3 站得住。
4. 字节预算对齐 → 让"同预算更省"可验证。
5. 结构化弃权 + 接真实网关 → 让"低开销"有实测支撑。

---

## 7. 在全系列里的定位建议

| 关系 | 说明 |
|---|---|
| 与 01（恢复） | 05 的定位结果就是 01 的"故障分类输入"；01 的 `events` 表可以当 05 的最小 trace |
| 与 02（路由） | 05 的"验证失败率"可以作为 02 风险模型的在线特征 |
| 与 03（授权） | 05 的审计轨迹可以消费 03 的哈希链 |
| 与 04（调度） | 05 要区分"真实故障"与"资源拥塞症状"，04 的 429 数据是天然负样本 |

**推荐定位**：**框架与科研素养都很好，但被 benchmark 误导了。** 我的判断是：**这个题目的价值高于它现在的数字**，因为：

- "部分可观测 + 采样预算 + 可弃权"这个角度，立项书自己论证得很清楚（AgentRx / AgentTrace / CORRECT 都假设较完整的轨迹）；
- 代码里已经有两个难得的正确做法（oracle 类型隔离、按 trace 配对重采样），这是可以复用的资产；
- 而它的失败**恰好是可以通过换数据集修好的**——不需要重写算法。

**建议的推进顺序**：
1. 先做 P0-1（换 benchmark，最便宜、收益最大），重跑一次，看 `first_error` 是否掉下来；
2. 若掉下来 → adaptive 的"打平"会变成"领先"，题目立刻活过来；
3. 再做 P0-2、P0-3，让"证据驱动"和"可校准"两个卖点各自成立；
4. 最后才考虑接真实网关。

**风险提示**：立项书 §11.3 列的 6 个投稿失败原因里，第 4 条是"**数据泄漏**：同一生成模板出现在训练/测试，导致模型学到注入位置而不是根因"。当前代码在**模板级留出**这一点上做对了，但在**特征与 label 的相关性**上犯了同一类错误（`abnormal` 既进特征又定 label）——这是"换了形式的泄漏"，必须优先修。

---

## 附录：文件地图

```text
05.md                        ← 立项书（543 行），§2 撞题表、§6 基线设计最值得读
05code/
├─ README.md                 ← 23 行，"未实现"写得比"已实现"长（这是好事）
├─ docs/QUALITY.md           ← 29 行，**负结果表 + "未超过 First Error"**
├─ tracelite/model.py        ← 73 行，**oracle 与 Observation 的类型隔离**（最核心的设计）
├─ tracelite/observe.py      ← 63 行，预算采样（注意 :38-40 的 label 相关性）
├─ tracelite/diagnose.py     ← 103 行，打分 + softmax + 弃权（注意 :78-79 的字符串逻辑）
├─ benchmarks/workload.py    ← 70 行，**9 模板生成器（benchmark 太简单的根源）**
├─ benchmarks/metrics.py     ← 60 行，指标口径（写得很好）
├─ benchmarks/run_eval.py    ← 95 行，模板级留出 + 9 策略
├─ gateway/proxy.go          ← 130 行，脱敏转发（隐私默认值到位）
└─ results/sample_run.json   ← 4,768 行，**所有量化结论的唯一来源**
```

**阅读顺序建议**：`model.py`（看隔离设计）→ `observe.py:29-40`（看采样优先级）→ `workload.py`（看 benchmark 有多简单）→ `results/sample_run.json` 的 `summary` 段（看结果）→ `QUALITY.md`。
