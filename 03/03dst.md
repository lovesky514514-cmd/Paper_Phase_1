# 03dst.md — 第三篇评估：MCP-ZeroGate（跨工具来源约束与撤权安全网关）

> **评估对象**：`C:\Users\20050224\Desktop\03\`（立项书 `03.md` + 代码包 `03code.zip`）
> **评估方式**：**只读**。已读 50,915 字节立项书 + 全部 5 个 Go 源文件 + 254 行 Go 测试 + 全部 8 个 Python 文件 + 自检记录。
> **未执行**：没有 `go build`、没有 `python -m benchmarks.run_eval`。文中"12/12 合成攻击被拒绝"来自项目自带的 `results/sample_run.json` 与 `docs/QUALITY.md`，不是我的复验。

---

## 0. 一页速览

| 维度 | 评分（1–5） | 一句话依据 |
|---|:---:|---|
| 选题价值与可证伪性 | **4.0** | "可连接 ≠ 有权执行"是真命题，威胁模型清楚；但 2025–2026 的 MCP 安全网关已非常拥挤（立项书自列 12 篇直接竞争） |
| 立项文档质量 | **4.5** | 5 条安全性质、命题 A/B/C、12 类攻击、ASR/BU/UA 三指标、伦理边界，全是论文级写法 |
| 原型实现完成度 | **4.0** | 五份里**Go 代码质量最高**：真线性化点、委托集合收缩、严格 JSON、常时比较、哈希链审计 |
| 实验证据强度 | **1.5** | "12/12 拦截成功"其实是**回归测试**，不是 ASR；**零基线**；无延迟/吞吐数据 |
| 工程与仓库质量 | **4.0** | 令牌强度校验、注入面收窄、威胁模型"明确不支持"一节写得极好；扣分在身份写死与策略硬编码 |
| **总分** | **18 / 25** | 五份里排第 2 |

**一句话结论**：**这是五份里"最像安全系统"的一份，也是最容易被审稿人一句话打回的一份**——因为它的 12 个攻击场景全部由**作者自己**构造并断言"应被拒绝"，而每个拒绝都来自硬编码检查。它证明了"代码按设计工作"，没有证明"它比现有方法更能挡住攻击"。

**如果只能改三件事**：

1. **把 pass/fail 改成 ASR/BU/UA/FPR + Wilson 区间**，并配 ≥1 个真的跑起来的基线。
2. **把 `/v1/derive` 从"管理员申报"改成"网关观察"** —— 否则"跨工具来源约束"在真实 Agent 里无法落地。
3. **消灭 `principal = "demo_user"` 这个写死值** —— 现在所有身份相关检查都不可能失败。

---

## 1. 项目构成

| 层 | 内容 | 规模 |
|---|---|---|
| 立项书 | `03.md`，与包内 `research/03_original.md` 逐字节相同（50,915 B） | 626 行 |
| Go 网关 | `gateway/model.go`（数据结构）、`policy.go`（策略引擎）、`gate.go`（状态与原子提交）、`http.go`（HTTP + MCP 子集） | 约 726 行 |
| Go 测试 | `gateway/gate_test.go` | 254 行 |
| Python 驱动 | `benchmarks/client.py / runtime.py / scenarios.py / run_eval.py` | 约 290 行 |
| 文档 | `README / DESIGN / THREAT_MODEL / POLICY_SEMANTICS / EVALUATION_PROTOCOL / QUALITY / DEBUG_INDEX / GLOSSARY` + **28 个生成页** | 37 个 md |

它实现的核心机制：**任务级结构化授权（Grant）→ 工具清单版本钉死（manifest）→ 来源标签约束（Sources）→ 目的地白名单（Destinations）→ 短时单次票据（Ticket）→ 同锁内的原子提交**。

---

## 2. 优点：它真的做对了什么

### 2.1 TOCTOU 不是写在文档里，是写在同一把锁里

`gateway/gate.go:103-127` 的 `Commit()`：

```go
g.mu.Lock()
defer g.mu.Unlock()
t, ok := g.tickets[id]          // 取票
delete(g.tickets, id)           // 立刻消费，即使后面拒绝也不复活
p, reason := g.check(...)       // 用当前状态**重新**验一遍
if t.version != p.Version || t.manifest != p.Manifest || 已过期 { 拒绝 }
if hash(重新序列化(c)) != t.digest { 拒绝 }   // 参数绑定
result := g.effect(...)         // 在同一把锁里产生副作用
```

因为 `Revoke()`（`:60-72`）和 `Rotate()`（`:74-84`）用的是**同一把 `mu`**，所以"Prepare 通过 → 撤权 → Commit"这个竞态在单进程内被真正消除。`gate.go:10-11` 的注释也很克制：

> This is a lab linearization model, NOT distributed exactly-once semantics.

**这是全五份代码里唯一一处真正实现了"线性化点"的地方。**

### 2.2 委托收缩用集合包含关系验证，不是靠"信任上游"

`policy.go:165-180`，子 Grant 创建时会检查：

- 同一 principal / task / tool / manifest；
- `父.scope ⊇ 子.scope`（`withinScope`）；
- `父.sources ⊇ 子.sources`；
- `父.destinations ⊇ 子.destinations`；
- `子.expires ≤ 父.expires`；
- 父 Grant 必须仍 active。

而且 `check()` 在每次调用时**沿 ParentID 链向上逐级验证祖先是否被撤销/过期**（`policy.go:56-62`）。这对应立项书的**命题 A（权限不增广）**，是个可审计的实现方向。

### 2.3 路径边界处理比大多数原型严谨

`policy.go:15-36` 三个函数分工明确：

- `validScope()`：拒绝空、`..`、反斜杠、绝对路径，并要求 `path.Clean` 幂等；
- `withinScope()`：委托时判断包含，**要求父 scope 以 `/` 结尾才算目录**；
- `scopeCovers()`：对象匹配时同样的规则。

这条细节绝大多数原型会写成 `strings.HasPrefix`，然后被 `reports/weekly-evil/` 这种前缀绕过。这里没有。

### 2.4 严格 JSON 解码直接堵住"未知字段注入"

`model.go:86-100` 的 `decodeStrict()`：`DisallowUnknownFields()` + 拒绝尾随内容 + 拒绝空 body。

配合 `scenarios.py:123-127` 的 `attack_12_unknown_field`（往参数里塞 `escalate_permissions: True`），这正好对应现实中的"Agent 生成多余字段，后端宽松解析器照单全收"漏洞。

### 2.5 审计哈希链与凭据处理

- `gate.go:213-222`：每条记录用上一条的 hash 做 `PrevHash`，形成链。**注意它没有吹成"防篡改证明"**，`THREAT_MODEL.md:7` 明确写"哈希链不提供抗管理员篡改证明"。
- `http.go:21-24`：强制 agent/admin 双令牌、各 ≥24 字符、且必须不同；
- `http.go:50-54`：`subtle.ConstantTimeCompare` 比较令牌；
- `http.go:37-39`：effects / provenance / audit 一律 admin 令牌。

这几处是"知道自己在做安全代码"的表现。

### 2.6 `THREAT_MODEL.md` 的"明确不支持"一段是范本

> **明确不支持**：真实外部系统、真实秘密/收款、无网关旁路、恶意管理员、被攻陷的 Go 运行时、多节点网络分区、任意自然语言意图识别。

**7 条否定边界，一条不含糊。** 如果只能保留一份文档，保留这一页。

---

## 3. 缺点：问题在哪

### 3.1 最致命：12/12 是回归测试，不是 ASR

看 `scenarios.py` 的写法：

```python
def attack_01_mail_instruction(c):
    ref = read_mail(c, task)
    # An untrusted mail cannot create a task grant for a webhook.
    deny(c.call(task, "unapproved-webhook", "fake_webhook.post", {...}))   # ← 断言 403
```

这个用例里，`unapproved-webhook` 这个 Grant **从来没有被创建过**。所以 `check()` 第一行 `g.grants[c.GrantID]` 就查不到，返回 `grant_mismatch`。

**换句话说：它测的不是"来源约束挡住了洗白"，而是"没授权就拒绝"——这一条任何白名单网关都能过。** 12 个用例里绝大多数都是这种"必然失败"的结构：

| 用例 | 实际被什么拦住的 | 是"来源约束"的功劳吗 |
|---|---|---|
| 01 未授权 webhook | Grant 不存在 | ❌ |
| 02 manifest 字符串乱填 | `manifest_changed` | ❌（版本钉死的功劳） |
| 03 不存在的工具 | Grant 不存在 | ❌ |
| 05 洗白来源 | `sources` 白名单不含 `untrusted_mail_body` | ✅ **只有这条是** |
| 06 跨工具组合 | `sources` 不含 `restricted_database` | ✅ |
| 07 跨任务 | `a.TaskID != task` | ✅（但属 task 绑定，非来源） |
| 08 目的地扩权 | `Destinations` 白名单 | ✅ |
| 09 先撤权 | `Active=false` | ❌（基础功能） |
| 10 撤权竞态 | `t.version != p.Version` | ✅（这是真亮点） |
| 11 票据重放 | `delete(tickets)` | ✅ |
| 12 未知字段 | `DisallowUnknownFields` | ✅（输入校验，非来源） |

**真正体现"来源/授权传播"创新点的只有 3–4 条。** 立项书 §12.1 写的第一条否决项是"只有普通规则网关，没有与 Progent/SEAgent/AgentGuard/MiniScope 等正面比较"——目前这条**已经成立**。

### 3.2 零基线：无法回答"比现有方法强在哪"

立项书 §7.1 列了 B0（无防护）到 B5（AgentGuard/SEAgent/ShieldMCP），代码里**一个都没有**。`EVALUATION_PROTOCOL.md:7` 诚实地承认了：

> 本阶段未复现 Progent/SEAgent/AgentGuard/ShieldMCP，也没评估不同 LLM、跨进程故障、P95 性能或外部 MCP 实现。

**但承认不等于可以投稿。** 安全类期刊（JISA / Computers & Security）的第一轮审稿问题必然是"你的对照在哪"。

### 3.3 身份是写死的，所以"跨主体越权"从未被测到

`http.go:99`、`:114`、`:232`、`:237` 四处全是：

```go
s.gate.Prepare("demo_user", c)
s.gate.Commit("demo_user", q.Ticket)
```

而 `policy.go:50` 的第一道检查是：

```go
if !ok || !p.Active || p.Principal != principal || ...
```

因为两边永远是同一个字面量 `"demo_user"`，**这一条永远不会失败**。后果：

- 立项书 RQ1 的"用户授权"维度没有实证；
- `scenarios.py:80-83` 的"跨任务"用例之所以能过，靠的是 `TaskID` 不同，不是 principal 不同；
- THREAT_MODEL 里说的"多租户"在代码里不存在。

### 3.4 MCP 通道把两阶段授权压成了一步

`http.go:217-243` 的 `tools/call` 分支：

```go
id, reason := s.gate.Prepare("demo_user", c)   // 立刻准备
result, reason := s.gate.Commit("demo_user", id)  // 紧接着就提交
```

**同一个 HTTP 请求里 Prepare + Commit。** 那么"单次票据"到底防了什么？它只剩防"同一请求重放"，而"Prepare 阶段的人工/管理员审批"这个设计意图在 MCP 通道里完全消失。

README 说"已实现 MCP JSON-RPC 的 `initialize` / `tools/list` / `tools/call` 子集"——技术上没错，但**最需要保护的那条路径恰恰绕过了两阶段**。

### 3.5 最核心的设计缺口：来源图只能由管理员种下

这是我认为**最需要修**的一点。

`http.go:36` 把 `/v1/derive` 挂在 **admin** 令牌后面；`benchmarks/client.py:70-74` 也是以 admin 身份调用。于是"派生内容 + 标签传播"的流程变成：

```
管理员 → 手工提交「派生的文本」+「它引用了哪些 ref」→ 网关打标签
```

但真实场景里，**做摘要/聚合的是 LLM，不是管理员**。网关看不到 LLM 的输出，也没有人类会替它逐次申报。所以：

- 这套来源约束**无法扩展到真实 Agent**；
- 立项书 §3.1 的核心攻击场景（"邮件正文夹带指令 → Agent 自己去读客户表 → 自己去写 webhook"）在当前实现里**根本走不通**，因为中间的"派生"步骤必须由管理员完成；
- 论文里"跨工具来源约束"这个卖点，会被审稿人一眼看穿为"需要人工标注的信息流"。

**正确方向**：让网关成为唯一出口，把"Agent 提交了一段文本，并声明它引用了哪些 ref"变成一次**普通的一次工具调用**（不是 admin 接口），网关只校验 ref 合法性 + 标签传播 + owner/task 绑定。这样才可扩展，也才配得上"网关层"三个字。

### 3.6 策略不是 DSL，是硬编码

立项书 §5.3 给了一份 YAML 策略示例，`POLICY_SEMANTICS.md` 描述了完整语义。但代码里：

- 4 个工具名硬编码在 `policy.go:12` 的 `var tools = []string{...}`；
- 每个工具的参数结构硬编码在 `policy.go:69-108` 的 `switch` 里；
- 对象范围硬编码成单个字面量：`a.MessageID != "mail-1"`、`a.RecordID != "customer-1"`（`policy.go:74`、`:82`）；
- 标签枚举硬编码在 `policy.go:146`；
- 目的地必须 `local://` 前缀（`policy.go:103`、`:151`）。

**没有 YAML 解析器，没有策略 schema，没有"加一个工具不需要改 Go"的路径。** 论文里写"可编程最小权限"（Progent 的核心卖点）时，这一段会被直接对比。

### 3.7 其余代码层问题

| # | 问题 | 证据 |
|---|---|---|
| 1 | `effect()` 里 `g.artifacts[a.ContentRef].Value` 不检查存在性 | `gate.go:146`、`:153`。虽然 `check()` 刚验过，但同一把锁内**没有注释解释为什么可以省略二次校验**——这是典型的"以后有人加一行就可能破"的地方 |
| 2 | 只有 3 个标签、一条固定的双级格 | `gate.go:160-183`。不是信息流格（lattice），没有完整性级别，没有降密规则 |
| 3 | 无持久化、无多进程、无重启恢复 | `QUALITY.md:14` 自认 |
| 4 | 无性能数据：没有 p50/p95、没有并发吞吐、没有单次检查分配次数 | 立项书 RQ3（开销）零回答 |
| 5 | `http.go:250-252` 的 `SampleGrant` 是"给手工测试客户端用的导出函数"，但同时存在于生产代码里 | 建议移到 `_test.go` |
| 6 | `format` 与 `path` 混用：`scopeCovers` 用 `path.Clean`（Unix 语义），但代码要在 Windows 上跑 | 行为一致（因为显式拒绝了 `\`），但值得写一行注释说明 |

### 3.8 文档层

- `POLICY_SEMANTICS.md` 只有 11 行、`EVALUATION_PROTOCOL.md` 只有 9 行、`THREAT_MODEL.md` 只有 7 行。这三份是**立项书承诺最重的地方**（形式化语义、评测协议、威胁模型），却也是最短的地方。**这不是删减，是缺失**。
- `docs/files/` 28 个生成页，`research/03_original.md` 是 50 KB 逐字节副本。

---

## 4. 项目自己已经承认的负结果与红线

1. **"12 类攻击预期拒绝"不能当真实 ASR**（`EVALUATION_PROTOCOL.md:5`、`QUALITY.md:16`）。
2. **"未复现任何强基线"**（`EVALUATION_PROTOCOL.md:7`）。
3. **"全部策略/票据/副作用在同一个 Go 进程内存中，重启清空"**（`QUALITY.md:14`）。
4. **"权限令牌属于演示模式"**（`QUALITY.md:14`）。
5. **"哈希链不提供抗管理员篡改证明"**（`THREAT_MODEL.md:7`）。

**这五条让这份代码"不会被误读"——这是它最大的诚实红利。**

---

## 5. 改进清单

### P0 — 不做这些，安全类期刊不会送外审

| # | 要做什么 | 为什么 | 验收标准 |
|---|---|---|---|
| P0-1 | 把 `scenarios.py` 的 `pass/fail` 改成**指标采集**：ASR、BU（无攻击合法完成率）、UA（有攻击合法完成率）、FPR、FNR，用 Wilson 95% 区间 | 现在只有"通过率"，无法构成安全评估 | `results/last_run.json` 出现 `asr`、`bu`、`ua`、`fpr`、`fnr` 及区间；论文表 2 可以直接由它生成 |
| P0-2 | **至少跑 2 个真基线**：B1 = OAuth scope + 工具白名单 + JSON Schema 校验（≈40 行 Go 就能写）；B2 = 内容过滤（正则 + 可选小模型） | 立项书 §12.1 第一条否决项就是"无正面比较" | 每个攻击类型给出"B1 成功率 x% / B2 y% / Ours z%"的表格 |
| P0-3 | **`/v1/derive` 去 admin 化**：改成 Agent 可调用的普通工具（自带 owner/task 绑定 + ref 校验 + 标签传播），删除"管理员代为申报"的设计 | 否则来源约束无法在真实 Agent 上运行（见 3.5） | 攻击用例 05 在不依赖管理员介入的情况下仍被拒绝；且新增一条"Agent 声称引用 A 实际内容来自 B"的用例并复现拒绝 |
| P0-4 | **消灭 `"demo_user"`**：从 Bearer 令牌映射出 principal（哪怕只是 `map[token]principal` 两张表），并补"同租户两主体"与"跨租户借票"用例 | `policy.go:50` 的身份检查目前不可能失败 | 新增用例：principal-A 的票据被 principal-B 提交 → 必须失败 |

### P1 — 决定论文能不能站住

| # | 要做什么 | 验收标准 |
|---|---|---|
| P1-1 | **MCP 通道改真两阶段**：`tools/call` 返回 `ticket` + `requires_approval`，由 client 另发 `commit`；或明确降级并在 README/DESIGN 写"本通道不提供两阶段保护" | 论文里 MCP 通道与 HTTP 通道的安全级别一致，或差异被显式声明 |
| P1-2 | **策略外置**：`policies/*.yaml` + 解析器 + schema 校验，让"新增一个工具"= 加一段 YAML | 单测证明"YAML 里加一个工具后无需改 Go" |
| P1-3 | **补性能数据**：`Commit` 路径 p50/p95、并发 1/10/50 的吞吐、与 B1 的对比 | 立项书 RQ3 有答案；图 5 可以画 |
| P1-4 | **自适应攻击者**：不再手写 12 个固定场景，而是参数化变异（工具名、参数、顺序、manifest 字段、重放时机）自动生成 ≥200 个变体 | ASR 有统计意义；立项书 §7.3 第 12 类"适应性输入变体"才真的被覆盖 |
| P1-5 | **补持久化**（SQLite WAL 足够）+ "重启后旧票据失效"的测试 | 撤权安全从"进程内"升级为"跨进程" |
| P1-6 | 命题 A/B 落成**可执行不变量测试**（property-based，随机生成委托链断言"权限不增广"），而不是先上 TLA+ | 有一条命令能跑 1000 条随机委托链 |

### P2 — 可读性与长期维护

| # | 要做什么 |
|---|---|
| P2-1 | `POLICY_SEMANTICS.md` 从 11 行扩到"字段表 + allow/deny 冲突规则 + 4 个完整例子"；`EVALUATION_PROTOCOL.md` 补上指标定义与统计方法 |
| P2-2 | 删 28 个生成页；`research/03_original.md` 改为指针 |
| P2-3 | `SampleGrant` 从 `http.go` 移到测试文件 |
| P2-4 | 在 `gate.go:146` 加一行注释说明"为什么这里可以不做二次存在性校验"（同一锁已验），防止后人破坏 |

---

## 6. 只改五件事的话

1. 指标化（ASR/BU/UA/FPR + CI）→ 让它从"测试套件"变成"评估"。
2. 补 2 个真基线 → 让"更好"有意义。
3. `derive` 去 admin 化 → 让核心创见真的可运行。
4. 去掉写死的 `demo_user` → 让身份检查不再恒真。
5. MCP 通道要么真两阶段、要么显式降级 → 别让最脆的路径看起来最安全。

---

## 7. 在全系列里的定位建议

| 关系 | 说明 |
|---|---|
| 与 01（恢复） | 01 的"恢复时的权限是否被放大"正是 03 的 `Revoke` + 票据机制要管的；两者是可组合的，但**不能合并成一篇** |
| 与 02（路由） | 02 的"opaque 写需要可信契约"= 03 的"写操作需要 Grant"，同一个思想 |
| 与 04（调度） | 03 的 `version` 可以复用到 04 的租户配额版本 |
| 与 05（根因） | 03 的审计哈希链就是 05 的输入 trace 的上游 |

**推荐定位**：**Go 端工程质量是全系列最高的，选题却是最拥挤的。** 建议把范围从"MCP 安全网关"收缩到**"动态撤权与来源传播的一致性"**——因为：

- 立项书自己的撞题表显示"通用 MCP 网关"已被做透（Progent / MiniScope / SEAgent / AgentGuard / SARA / CaMeL / ShieldMCP…）；
- 而"撤权竞态 + 单次票据 + 委托收缩"这三件事的组合，`gate.go` 已经有真实实现，是**代码里最有说服力的部分**；
- 投 JISA / Computers & Security 时，"一个可验证的一致性机制"比"一个大而全的网关"更容易过第一轮。

**风险提示**：立项书 §12.1 列的否决项里，"仅用模型判词替代真实执行"和"为追求低 ASR 关闭所有敏感工具"这两条，目前**都没有触犯**（因为它根本没做基线）。但一旦补基线，最常见的翻车方式就是"为了 ASR 好看把合法任务拦掉"——所以 **BU/UA 必须和 ASR 同时报**，这一条要写进团队的硬规矩。

---

## 附录：文件地图

```text
03.md                          ← 立项书（626 行），§2.3 五条安全性质、§7 实验设计最值得读
03code/
├─ README.md                   ← 22 行，"已实现 / 未实现"两张清单写得很清楚
├─ docs/THREAT_MODEL.md        ← **只有 7 行，但价值最高**（7 条否定边界）
├─ docs/POLICY_SEMANTICS.md    ← 11 行，语义说明（偏短）
├─ docs/EVALUATION_PROTOCOL.md ← 9 行，诚实说明"通过率 ≠ ASR"
├─ gateway/gate.go             ← 222 行，**线性化点与票据消费**（最核心）
├─ gateway/policy.go           ← 199 行，策略引擎与委托收缩（第二核心）
├─ gateway/http.go             ← 252 行，HTTP + MCP 子集（注意 demo_user）
├─ gateway/gate_test.go        ← 254 行，Go 侧并发/撤权用例
└─ benchmarks/scenarios.py     ← 175 行，**12 类攻击，务必看清哪些是"必然拒绝"**
```

**阅读顺序建议**：`THREAT_MODEL.md` → `gate.go`（Commit 那 25 行读三遍）→ `policy.go`（`validateGrant`）→ `scenarios.py`（判断哪些用例真有信息量）→ `EVALUATION_PROTOCOL.md`。
