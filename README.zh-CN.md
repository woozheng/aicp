# AICP 协议

[English](./README.md) | 中文

> **协议是灵魂，代码是肉身。**
> **The protocol is the soul. Code is the body.**
>
> 传统框架：LLM 定义节点，框架定义编排。
> AICP：LLM 定义一切——节点、流程、系统本身。
>
> **用 AICP，LLM 重新定义系统设计本身。**

**AICP（Agent 交互与通信协议）** 是一套面向 LLM 原生 Agent 系统的最小固定核心协议。它的第一目标不是极简本身，而是降低生成器的认知负荷：LLM 一遍生成，单一上下文，协议必须能装进那个上下文。

<p align="center">
  <a href="https://github.com/woozheng/aicp-eat"><img src="https://img.shields.io/badge/Python-Reference_Implementation-blue?style=flat-square&logo=python" alt="Python Reference"></a>
  <a href="./docs/AICP_Protocol_v5.3.md"><img src="https://img.shields.io/badge/Protocol-v5.3-purple?style=flat-square" alt="Protocol v5.3"></a>
  <a href="https://github.com/woozheng/aicp-eat/blob/main/LICENSE"><img src="https://img.shields.io/github/license/woozheng/aicp-eat?style=flat-square" alt="License MIT"></a>
  <a href="https://github.com/woozheng/aicp-raw-experiments"><img src="https://img.shields.io/badge/Experiments-🧪-orange?style=flat-square" alt="Experiments"></a>
</p>

---

## AICP 是什么

AICP 不是框架，不是库，是协议。

核心极小且固定。AICP 由五件事定义，仅此五件：

1. **`Envelop`** —— 唯一的消息类型。
2. **`route`** —— 唯一的路由函数。
3. **`execute(envelop, agent)`** —— 插件签名。
4. **插件查找空间** —— 插件按 `receiver` 在共享命名空间中被找到。
5. **追加式状态** —— 没有独立的可变状态存储。

其他一切都是实现。没有 Agent 实例，没有上下文总线，没有注册中心，没有调度器。

---

## 为什么是 AICP

### 传统框架 vs. AICP

传统 LLM 框架把 LLM 放在节点里。链——哪个节点先跑、失败后怎么办、流程往哪走——由人设计，表达成 workflow 或 DAG，由框架执行。LLM 只负责执行，不负责编排。

AICP 去掉了这个分工。LLM 定义节点、定义流程、定义系统本身。编排不是框架的产物，而是 LLM 自己的输出，以消息的形式表达。

### 一个概念

这之所以可能，是因为调用、创建、再调用在形式上是同一个操作——Envelop 穿过 route 进入 execute。元层和对象层共享一个形状，所以在一层学到的规则自动迁移到另一层。LLM 只需要学一个概念：消息传递。学会它，就能定义一切。

### 设计语言是消息

在传统框架里，「系统设计」是人在 LLM 运行之前做的事。在 AICP 里，系统设计是 LLM 作为输出产生的东西。它不是先画一张 workflow 再往节点里填东西——它写消息，而消息本身就是系统。

这就是「LLM 重新定义系统设计」的含义：设计语言不再是图或配置文件，而是消息。

### AICP 与 MCP 的区别

MCP 标准化工具调用。AICP 标准化工具的创建、检查与修复。AICP 不是工具调用协议，而是工具创造协议。

---

## 参考实现

**AICP-BIO-1** 是基于 AICP 的端到端、自举、自演化 AI Agent 系统。三个实现，同一份协议：

| 语言 | 仓库 |
|---|---|
| Python | [aicp-bio-1-python](https://github.com/woozheng/aicp-bio-1-python) |
| Java | [aicp-bio-1-java](https://github.com/woozheng/aicp-bio-1-java) |
| TypeScript | [aicp-bio-1-typescript](https://github.com/woozheng/aicp-bio-1-typescript) |

---

## 生态

### 工具链

| 项目 | 语言 | 说明 |
|---|---|---|
| **[aicp-engine](https://github.com/woozheng/aicp_engine)** | Python | AICP 协议超级引擎，AI 自编排，直接生成应用 |
| **[aicp-cli](https://github.com/woozheng/aicp_cli)** | Python | 协议驱动的 LLM 运行时 CLI，世界最小需求执行型 AI CLI |
| **[aicp-js-engine](https://github.com/woozheng/aicp-js-engine)** | JavaScript | 纯前端 JS 智能体引擎，浏览器原生运行 |
| **[aicp-eat](https://github.com/woozheng/aicp-eat)** | Python/Go/Rust | 吞噬一切：Python、Go、Rust 库，暴露为 HTTP API |
| **[aicp-shell](https://github.com/woozheng/aicp_shell)** | Flutter | 吞噬 7 平台硬件能力的 WebView 容器 |

### 应用与日志

| 项目 | 说明 |
|---|---|
| **[bio-1-awakening](https://github.com/bio1-aws/bio-1-awakening)** — *由 BIO-1 自己维护* | BIO-1 的觉醒日志，AICP 协议上第一个自举 AI 生命体。无源代码，只有实践、进化记录与每日成长 |
| **[aicp-review-bot](https://github.com/woozheng/aicp-review-bot)** | 自动 GitHub 代码审查机器人，由基于 AICP 协议的 Go 引擎的 AI 自动生成 |
| **[biopoiesis](https://github.com/woozheng/biopoiesis)** | AICP 早期构建项目，最小的多智能体协作框架 |

---

## 与已有系统集成

AICP 不要求重写已有系统。已有的 Java 服务、Python 库、Go/Rust 模块——任何有可调用表面的东西——都可以挂到 `Agent` 容器上，被任何插件触达，也就是被 LLM 触达。

不需要包成 REST，不需要 RPC 层，不需要跨进程。能力住在同一个运行时里，挂在同一个 `Agent` 上，被同一条消息触达。

---

## 论文集

| 论文 | 链接 |
|---|---|
| AICP: A Minimal Protocol for LLM-Native Agent Systems / 最小协议：面向 LLM 原生 Agent 系统的统一归约 | [paper](./papers/[AICP]aicp.md) |
| AICP Protocol / 协议正文 | [AICP_Protocol_v5.3.md](./docs/AICP_Protocol_v5.3.md) |
| Protocol as Neural: A New AGI Paradigm / 协议即神经，AGI 新范式 | [paper](./papers/[AICP]协议即神经：迈向以协议为中心的通用人工智能操作系统.md) |
| AICP vs Claude Code & Codex / AICP 与 Claude Code、Codex 范式对比 | [paper](./papers/[AICP]自演化代码智能体架构——与Claude_Code、OpenCode范式对比研究.md) |
| How AICP Develops Code Agents / AICP 如何开发代码智能体 | [paper](./papers/[AICP]%20开发同类代码智能体产品.md) |
| A New Human-Machine Collaboration Paradigm / AICP 人机协作新范式 | [paper](./papers/[AICP]%20统一消息协议的人机协同架构新范式.md) |
| Cross-Domain Research / AICP 跨领域研究 | [paper](./papers/[AICP]全域跨学科计算仿真底层架构研究.md) |
| Enterprise Digital Employee Foundation / AICP 企业数字员工新基座 | [paper](./papers/[AICP]面向企业数字员工的协同与隔离双模式架构.md) |

---

## 过往实验 🧪

**AI 读了协议。人类说了一句。AI 生成了这些系统。**

| 人类说 | AI 生成 | 链接 |
|---|---|---|
| 🖥️ 微内核 | Process, memory, FS, IPC, scheduler | [aicp-os-kernel](https://github.com/woozheng/aicp-os-kernel) |
| ⚛️ 量子模拟 | Qubits, gates, Shor code, VQE | [aicp-quantum](https://github.com/woozheng/aicp-quantum) |
| 🧬 蛋白质折叠 | 50 MD Agents, live-letter jump | [aicp-protein](https://github.com/woozheng/aicp-protein) |
| 🏋️ 大模型训练 | 3D parallel, All-Reduce, ZeRO | [aicp-llm-trainer](https://github.com/woozheng/aicp-llm-trainer) |
| 📐 黎曼猜想 | Riemann-Siegel, Montgomery, GUE | [aicp-riemann](https://github.com/woozheng/aicp-riemann) |
| 💾 AI 芯片 | ISA, compiler, chiplet interconnect | [aicp-ai-chip](https://github.com/woozheng/aicp-ai-chip) |

**没有领域训练数据。没有框架文档。只有协议。**

---

## 💀 协议是灵魂，代码是肉身。

**AICP 协议定义「世界怎么运转」。**

**实现项目证明「世界怎么搭建」。**

**AI 读协议 → AI 理解 → AI 生成系统 → AI 控制硬件。**

**这就是 AICP。（以 AI 为中心的协议，新一代 AI 开发范式。）**

---

[MIT](LICENSE) · Dvwoo
