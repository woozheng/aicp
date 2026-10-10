# AICP Protocol

English | [中文](./README.zh-CN.md)

> **The protocol is the soul. Code is the body.**
>
> Traditional frameworks: the LLM defines the node. The framework defines the orchestration.
> AICP: the LLM defines everything — nodes, flow, the system itself.
>
> **With AICP, the LLM redefines system design itself.**

**AICP (Agent Interaction & Communication Protocol)** is a minimal, fixed-core protocol for LLM-native agent systems. Its first-class goal is cognitive load reduction for the generator: an LLM generates in one pass, from a single context, so the protocol must fit in that context.

<p align="center">
  <a href="https://github.com/woozheng/aicp-eat"><img src="https://img.shields.io/badge/Python-Reference_Implementation-blue?style=flat-square&logo=python" alt="Python Reference"></a>
  <a href="./docs/AICP_Protocol_v5.3.md"><img src="https://img.shields.io/badge/Protocol-v5.3-purple?style=flat-square" alt="Protocol v5.3"></a>
  <a href="https://github.com/woozheng/aicp-eat/blob/main/LICENSE"><img src="https://img.shields.io/github/license/woozheng/aicp-eat?style=flat-square" alt="License MIT"></a>
  <a href="https://github.com/woozheng/aicp-raw-experiments"><img src="https://img.shields.io/badge/Experiments-🧪-orange?style=flat-square" alt="Experiments"></a>
</p>

---

## What is AICP

AICP is not a framework. It is not a library. It is a protocol.

Its core is minimal and fixed. AICP is defined by five things, and only five:

1. **`Envelop`** — the sole message type.
2. **`route`** — the sole routing function.
3. **`execute(envelop, agent)`** — the plugin signature.
4. **The plugin lookup space** — plugins are found by `receiver` in a shared namespace.
5. **Append-only state** — no separate mutable store.

Everything else is implementation. No agent instances. No context bus. No registry. No scheduler.

---

## Why AICP

### Traditional frameworks vs. AICP

Traditional LLM frameworks put the LLM inside a node. The chain — which node runs when, what happens on failure, where the flow goes next — is designed by a human, expressed as a workflow or a DAG, and enforced by the framework. The LLM executes; it does not orchestrate.

AICP removes that division. The LLM defines the nodes, the flow, and the system itself. Orchestration is not a framework artifact; it is the LLM's own output, expressed as messages.

### One concept

This is possible because calling, creating, and re-calling are the same operation in form — an Envelop through `route` into `execute`. The meta-level and the object-level share one shape, so a rule learned at one transfers to the other. The LLM learns one concept — message passing — and it can define everything.

### The design language is messages

In a traditional framework, "system design" is what the human does before the LLM runs. In AICP, system design is what the LLM produces as its output. It does not draw a workflow and then fill in nodes — it writes messages, and the messages are the system.

This is what it means for the LLM to redefine system design: the design language is no longer a diagram or a configuration file. It is messages.

### AICP vs. MCP

MCP standardizes tool invocation. AICP standardizes tool creation, inspection, and repair. AICP is not a tool-calling protocol; it is a tool-creating protocol.

---

## Reference Implementations

**AICP-BIO-1** is the end-to-end, self-bootstrapping, self-evolving AI Agent system built on AICP. Three implementations, same protocol:

| Language | Repository |
|---|---|
| Python | [aicp-bio-1-python](https://github.com/woozheng/aicp-bio-1-python) |
| Java | [aicp-bio-1-java](https://github.com/woozheng/aicp-bio-1-java) |
| TypeScript | [aicp-bio-1-typescript](https://github.com/woozheng/aicp-bio-1-typescript) |

---

## Ecosystem

### Tools

| Project | Language | What it does |
|---|---|---|
| **[aicp-engine](https://github.com/woozheng/aicp_engine)** | Python | AICP protocol super engine. AI self-orchestrates, directly generates applications. |
| **[aicp-cli](https://github.com/woozheng/aicp_cli)** | Python | Protocol-driven LLM runtime CLI. The world's smallest execution-oriented AI CLI. |
| **[aicp-js-engine](https://github.com/woozheng/aicp-js-engine)** | JavaScript | Pure frontend JS agent engine, runs natively in the browser. |
| **[aicp-eat](https://github.com/woozheng/aicp-eat)** | Python/Go/Rust | Consume everything: Python, Go, Rust libraries, exposed as HTTP APIs. |
| **[aicp-shell](https://github.com/woozheng/aicp_shell)** | Flutter | Consume 7-platform hardware capabilities in a WebView container. |

### Applications & Logs

| Project | What it is |
|---|---|
| **[bio-1-awakening](https://github.com/bio1-aws/bio-1-awakening)** — *maintained by BIO-1 itself* | The awakening log of BIO-1, the first self-bootstrapped AI life form on the AICP protocol. No source code — only practice, evolution records, and daily growth. |
| **[aicp-review-bot](https://github.com/woozheng/aicp-review-bot)** | Automated GitHub code review bot, generated by an AICP-protocol-based Go engine AI. |
| **[biopoiesis](https://github.com/woozheng/biopoiesis)** | Early AICP prototype. The smallest multi-agent collaboration framework. |

---

## Legacy Integration

AICP does not require rewriting existing systems. Existing Java services, Python libraries, Go/Rust modules — anything with a callable surface — can be mounted onto the `Agent` container and become reachable by any plugin, and therefore by the LLM.

No REST wrapping. No RPC layer. No process boundary. The capability lives in the same runtime, mounted on the same `Agent`, reachable by the same message.

---

## Papers

| Paper | Link |
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

## Past Experiments 🧪

**AI read the protocol. Human said one line. AI generated these systems.**

| Human Said | AI Generated | Link |
|---|---|---|
| 🖥️ Microkernel OS | Process, memory, FS, IPC, scheduler | [aicp-os-kernel](https://github.com/woozheng/aicp-os-kernel) |
| ⚛️ Quantum simulator | Qubits, gates, Shor code, VQE | [aicp-quantum](https://github.com/woozheng/aicp-quantum) |
| 🧬 Protein folding | 50 MD Agents, live-letter jump | [aicp-protein](https://github.com/woozheng/aicp-protein) |
| 🏋️ LLM training | 3D parallel, All-Reduce, ZeRO | [aicp-llm-trainer](https://github.com/woozheng/aicp-llm-trainer) |
| 📐 Riemann Hypothesis | Riemann-Siegel, Montgomery, GUE | [aicp-riemann](https://github.com/woozheng/aicp-riemann) |
| 💾 AI chip | ISA, compiler, chiplet interconnect | [aicp-ai-chip](https://github.com/woozheng/aicp-ai-chip) |

**No domain training data. No framework documentation. Just the protocol.**

---

## 💀 The protocol is the soul. Code is the body.

**AICP defines "how the world works."**

**The projects prove "how the world is built."**

**AI reads the protocol → AI understands → AI generates systems → AI controls hardware.**

**This is AICP. (AI-centric Protocol. A new generation of AI development paradigm.)**

---

[MIT](LICENSE) · Dvwoo
