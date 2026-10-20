# Paper_Phase_1

**Agent & Gateway Research — Phase I**

本仓库用于整理和推进第一阶段的五项独立科研课题，研究方向集中于 **LLM Agent、后端网关、可靠性、安全控制、任务调度与故障诊断**。

项目采用“研究方案 → 原型实现 → 外部评审 → 实验验证 → 论文完善”的研究流程，探索智能体系统在实际工程环境中的性能、可靠性与安全性问题。

目前五项研究均处于方案设计与原型验证阶段，尚未完成充分的真实环境实验及论文级性能验证。

## 01 — Agent 故障感知与自动恢复网关

**Risk-Aware Fault Recovery for Tool-Using LLM Agents**

研究面向工具调用型 Agent 的故障检测、安全恢复与执行状态一致性问题。传统系统在 API 超时、工具异常或服务中断时，通常采用固定重试或重新执行任务，容易产生重复操作与额外开销。本课题尝试通过网关层的故障分类、执行状态追踪和风险感知恢复决策，实现安全重试、局部恢复及不可逆操作保护，重点评估任务恢复成功率、重复副作用、执行延迟与系统开销。

## 02 — 多模型动态路由与成本优化

**Verification-Cost- and Execution-Risk-Aware Dynamic Routing**

研究多模型协作环境下的动态模型选择和任务执行成本优化。现有路由方法通常关注模型推理价格、响应质量和延迟，但工具调用型 Agent 还存在验证开销、执行失败和任务恢复等隐性成本。本课题尝试构建联合考虑模型能力、验证成本、执行风险与任务整体收益的动态路由机制，以降低任务端到端成本，同时维持任务质量与安全约束。

## 03 — MCP 工具调用安全验证网关

**Security Verification and Access Control for MCP-Based Agents**

研究基于 Model Context Protocol（MCP）的智能体工具调用安全问题。随着 Agent 获得文件操作、数据库访问和外部 API 执行能力，跨工具权限传播、工具描述变化、越权调用与动态撤权成为重要风险。本课题尝试设计独立于 Agent 推理过程的安全验证网关，通过执行前权限检查、上下文约束、工具身份验证和审计机制，降低跨工具调用中的权限滥用与不安全执行风险。

## 04 — 多 Agent 任务调度与拥塞控制

**Dependency-Aware Scheduling and Congestion Control for Multi-Agent Workflows**

研究多个 Agent 共享计算资源、模型服务及 API 配额时的任务调度和资源竞争问题。传统调度方式难以同时兼顾工作流依赖、系统吞吐量、尾部延迟和多租户公平性。本课题尝试结合任务依赖图、动态拥塞反馈、配额管理及公平调度策略，优化高并发环境下的任务执行顺序与资源分配，重点考察任务完成率、P95/P99 延迟、资源利用率和饥饿问题。

## 05 — Agent 调用链异常检测与根因定位

**Evidence-Aware Anomaly Detection and Root-Cause Localization for Tool-Using Agents**

研究多步骤 Agent 工作流中的异常传播、调用链分析和故障根因定位问题。复杂 Agent 系统中的最终报错位置往往不等于真实故障来源，单纯依靠错误日志难以区分初始故障与后续连锁异常。本课题尝试利用网关采集的调用关系、执行状态和异常证据构建故障分析机制，并结合有限观测预算、根因排序和证据可信度评估，提高故障定位准确率，同时减少不必要的日志采集与诊断成本。

---

## Repository Structure

```text
Paper_Phase_1/
│
├── 01/
│   ├── 01.md
│   ├── 01code.zip
│   └── 01dst.md
│
├── 02/
│   ├── 02.md
│   ├── 02code.zip
│   └── 02dst.md
│
├── 03/
│   ├── 03.md
│   ├── 03code.zip
│   └── 03dst.md
│
├── 04/
│   ├── 04.md
│   ├── 04code.zip
│   └── 04dst.md
│
├── 05/
│   ├── 05.md
│   ├── 05code.zip
│   └── 05dst.md
│
├── README.md
└── LICENSE
```

### File Description

| 文件 | 内容 |
|---|---|
| `XX.md` | 论文研究方案、创新点、技术路线、相关文献、实验设计及期刊分析 |
| `XXcode.zip` | 对应研究方向的代码原型、测试程序及相关说明 |
| `XXdst.md` | DeepSeek 提供的独立评审意见、问题分析及改进建议 |

其中 `XX` 对应研究编号 `01–05`。

## Research Workflow

各研究方向按照以下流程独立推进：

1. **Research Design** — 明确研究问题、创新假设及技术路线。
2. **Literature Review** — 调研相关研究，识别创新重叠与潜在改进空间。
3. **Prototype Development** — 构建可运行的最小研究原型。
4. **Independent Review** — 结合外部模型反馈检查理论、架构及实现问题。
5. **Experimental Validation** — 开展基线对照、消融实验、压力测试及统计分析。
6. **Paper Preparation** — 根据实验结果完善方法、讨论局限并准备论文投稿。

## Current Status

**Phase I — Research Prototypes & Validation**

目前五个项目已经建立研究方案和初步代码结构，基础自检已开展，但仍需完成更充分的真实基准、强基线对照、性能优化和跨环境验证。

仓库中的研究命题、算法名称及预期创新均属于当前研究设计，不代表相关算法已获得实验验证，也不代表论文已经投稿或录用。

各项目独立维护，后续将根据实际实验结果决定是否继续深化、调整研究方向或准备投稿。

## License

This repository is distributed under the [MIT License](LICENSE).

研究文献、第三方依赖及相关数据集遵循各自的许可与引用要求。
