# AICP: A Minimal Protocol for LLM-Native Agent Systems

**A Unified Reduction of Actor, Message Bus, Event Sourcing, and Microkernel Paradigms**

**Dvwoo**  
GitHub: [@woozheng](https://github.com/woozheng/aicp)

---

## Abstract

Contemporary agent frameworks accumulate complexity: state machines, multi-agent schedulers, tool registries, context buses, and lifecycle managers. Each solves a real problem, yet their combination imposes heavy cognitive load on both developers and large language models (LLMs).

We present AICP (Agent Interaction & Communication Protocol), a minimal protocol that reduces the canonical paradigms of distributed computing — Actor model, message bus, event sourcing, microkernel, plugin architecture, and dependency injection — into a single uniform structure: messages flow, plugins react, state accumulates as information flow.

AICP is defined by roughly 200 lines of core specification. It has no agent instances, no context bus, no registry, and no scheduler. Plugins share a single signature (`async def execute(envelop, agent)`), communicate exclusively through a single message type (`Envelop`), and persist state solely through an append-only information flow (`_flow`).

We argue that this reduction is not merely aesthetic. It lowers the cognitive burden on LLMs, enabling one-shot code generation with minimal context; it enables cross-language replication (Python and TypeScript implementations are structurally identical); and it establishes a foundation on which self-bootstrapping agent systems — capable of creating, inspecting, and repairing their own plugins — can be built.

We further discuss the role of a "constitution" layer: a set of non-negotiable constraints enforced at the framework level, which grants the LLM maximum freedom of intent while bounding the space of expression. Finally, we position AICP relative to MCP and traditional agent architectures, and report cross-domain experiments in which a single protocol description enabled an LLM to generate systems in domains ranging from microkernel operating systems to quantum simulation.

---

## 1. Introduction

### 1.1 The Accumulation Problem

Modern agent frameworks tend to grow by addition. Each new requirement — tool calling, multi-agent coordination, memory, streaming, async callbacks — is met with a new abstraction: a scheduler, a context bus, a memory store, a tool registry, a lifecycle manager. Over time, the framework becomes a layered stack of mechanisms, each locally justified, collectively opaque.

This accumulation has three costs:

- **Cognitive cost for developers.** Understanding the system requires understanding every layer and its interactions.
- **Cognitive cost for LLMs.** Generating correct code requires holding many conventions simultaneously — signatures, return shapes, registration rules, lifecycle expectations.
- **Portability cost.** Each abstraction tends to be language- and runtime-specific, making cross-language replication a rewrite rather than a translation.

### 1.2 The Reduction Hypothesis

We hypothesize that most of these abstractions are not independent primitives but variants of a single underlying structure: a message-oriented system in which independent units react to messages, and state is the accumulated record of those reactions.

If this is true, then a protocol built on this single structure — without the additional abstractions — should be able to express everything the layered frameworks express, while imposing far less cognitive load.

AICP is an attempt to test this hypothesis.

### 1.3 Contributions

We reduce six canonical paradigms (Actor, message bus, event sourcing, microkernel, plugin architecture, dependency injection) to a single protocol of roughly 200 lines.

We show that this protocol enables one-shot LLM code generation with minimal context.

We demonstrate cross-language structural identity: a Python implementation and a TypeScript implementation share the same architecture, differing only in syntax.

We show that the protocol supports self-bootstrapping: an agent can create, inspect, and repair its own plugins.

We introduce the notion of a constitution layer: framework-enforced constraints that bound LLM expression without bounding LLM intent.

We report cross-domain experiments in which one protocol description enabled an LLM to generate systems across six unrelated domains.

---

## 2. Related Work

### 2.1 The Actor Model

The Actor model (Hewitt, 1973; Agha, 1986) introduced the idea of independent computational entities communicating solely via messages, with no shared state. Erlang (Armstrong, 1986) demonstrated its practicality at scale.

AICP preserves the message-passing core of Actor systems but removes the Actor as an entity. There is no actor object, no mailbox, no PID. A "plugin" is a function; a "session" is an identifier; state lives in an append-only flow.

### 2.2 Message Buses and RPC

Enterprise message buses and RPC systems (CORBA, gRPC) provide a central dispatch layer with registered endpoints. AICP retains dispatch but eliminates the bus as an object: the dispatcher is a dictionary lookup followed by a function call.

### 2.3 Event Sourcing

Event sourcing (Fowler, 2005) treats state as the fold of an event log. AICP adopts this directly: the information flow (`_flow`) is the sole state store. Unlike typical event sourcing, AICP does not separate "events" from "state projections" — the flow is read directly by the LLM, with per-entry retention hints controlling what is surfaced.

### 2.4 Microkernels and Plugin Architectures

Microkernels (Liedtke, 1995) minimize the kernel and push functionality into user-space servers. Plugin architectures (e.g., Eclipse, VS Code) register extensions against a host. AICP combines both: the "kernel" is a routing function; "plugins" are files on disk. There is no registry — the filesystem is the registry.

### 2.5 Dependency Injection

DI frameworks inject capabilities into components. AICP injects a single `agent` object holding all capabilities (`llm`, `system`, `config`, `data_dir`, `log`). Plugins never import capabilities; they receive them.

### 2.6 MCP and Tool Protocols

The Model Context Protocol (MCP) standardizes how LLMs call predefined tools. AICP differs fundamentally: tools (plugins) are not predefined — they are generated, inspected, and repaired at runtime by the LLM itself. AICP is not a tool-calling protocol; it is a tool-creating protocol.

---

## 3. The AICP Protocol

### 3.1 Design Principles

AICP is governed by four principles:

1. **One shape.** Every plugin has the same signature: `async def execute(envelop, agent)`.
2. **One message.** All communication is a single type: `Envelop`.
3. **One state.** All state is an append-only information flow.
4. **No privileges.** No plugin is special; the main agent is itself a plugin.

### 3.2 Envelop

`Envelop` is the sole message type. It carries:

- `sender`, `receiver` — routing identity
- `intent`, `payload` — business content
- `trace_id`, `message_id` — tracing
- `ttl` — lifecycle
- `meta` — out-of-band control (callbacks, sessions, task IDs)

### 3.3 Route

Routing is a single function:

```python
async def route(envelop, agent):
    plugin = plugins.get(envelop.receiver)
    return await plugin(envelop, agent)
```

Three modes are supported, distinguished by `meta`:

- **Synchronous** — direct invocation
- **Asynchronous with callback** — background execution, results delivered to `callback_receiver`
- **Callback acknowledgement** — immediate ACK, background delivery

### 3.4 Plugins

A plugin is a Python (or TypeScript) function:

```python
async def execute(envelop, agent):
    ...
    envelop.payload = {"ok": True, "data": {...}}
    return envelop
```

There is no class, no instance, no lifecycle. A plugin "exists" iff its file exists.

### 3.5 Information Flow

`InformationFlow` is an append-only list of entries, persisted as JSON. Entries have a `from` field (`user` / `ai` / `system`), a timestamp, and a retain hint controlling how many subsequent turns the entry remains visible to the LLM.

The flow is both the agent's memory and its state. There is no separate state store.

### 3.6 Capability Injection

The `agent` object is the sole dependency container. It exposes:

- `agent.llm` — LLM interface (`chat`, `chat_json`, `chat_stream`)
- `agent.system` — inter-plugin call interface
- `agent.config` — configuration
- `agent.data_dir` — data root
- `agent.log` — logger

Plugins never import capabilities; they receive `agent`.

### 3.7 The Absence of a Context Bus

Traditional agent systems maintain a context bus: a central object holding registered agents, session state, and global context.

AICP has no such object. "Context" is the `_flow`. "Sessions" are identifiers. "Agents" are functions. The bus is replaced by two primitives: `route` and `plugins`.

We argue this absence is not a simplification but a dissolution: the concept of a context bus is a consequence of treating agents as instances. Once agents are functions, the bus has nothing to hold.

---

## 4. The Constitution Layer

### 4.1 Freedom and Constraint

Giving an LLM maximum freedom of intent while preventing arbitrary expression is a design tension. Too little freedom reduces the LLM to a dispatcher; too much freedom makes the system unpredictable.

AICP resolves this with a constitution layer: a set of framework-enforced constraints that bound expression without bounding intent.

### 4.2 The Constitution

The constitution includes:

- **Plugin shape.** Only `async def execute(envelop, agent)` is accepted; AST validation rejects top-level code, `if __name__`, and non-standard signatures.
- **Output format.** Each turn produces exactly one JSON object or one plain-text reply. Multiple JSON objects trigger a circuit breaker.
- **Large-text handling.** Content exceeding a sentence is emitted in a `@@CONTENT@@` block, not embedded in JSON.
- **Reasoning brevity.** The `think` field is bounded; oversized reasoning triggers a retry.
- **Sandbox rules.** `print`, `time.sleep`, `while True`, `subprocess.Popen`, and shell metacharacters are blocked at the sandbox level, not merely documented.
- **Self-invocation rules.** The main agent cannot be directly called via `use_tool`; sub-agents are created only via `task_manager`.
- **Contract-first discovery.** Unknown plugins must be inspected via `contract_agent` before use.

### 4.3 Enforcement, Not Documentation

Each constitution rule is enforced mechanically:

| Rule | Enforcement |
|------|-------------|
| Plugin shape | AST pre-check + runtime check |
| Output format | FastValidator + streaming circuit breaker |
| Large-text handling | Parser-level requirement |
| Reasoning brevity | Streaming circuit breaker |
| Sandbox rules | Replaced builtins in the execution namespace |
| Self-invocation | FastValidator interception |
| Contract-first | Prompt guidance + contract caching |

The LLM does not "follow" the constitution; it cannot violate it. This distinction matters: rules the LLM can violate are rules the developer must defend against; rules the LLM cannot violate are rules the developer can forget.

---

## 5. Cognitive Load Reduction

### 5.1 The Hypothesis

We hypothesize that protocol simplicity directly reduces LLM cognitive load, which in turn:

- Increases one-shot correctness
- Reduces required context
- Enables cross-language replication

### 5.2 Mechanism

A minimal protocol removes decisions from the LLM's generation process. Instead of choosing among N signatures, return shapes, and registration conventions, the LLM chooses among one. Its attention is freed for business logic.

This is analogous to how high-level languages reduce programmer cognitive load by removing concerns (registers, memory layout) irrelevant to the task.

### 5.3 Evidence

- **One-shot correctness.** In practice, plugins generated against AICP are accepted on first attempt far more often than against layered frameworks.
- **Minimal context.** Generating a plugin requires the protocol description (~a few hundred tokens), not framework documentation.
- **Cross-language replication.** Two independent implementations (Python and TypeScript) were produced by feeding the protocol and the reference implementation to an LLM; both are structurally identical to the original.

We do not claim these as formal measurements; we present them as design observations motivating further study.

---

## 6. Self-Bootstrapping

### 6.1 Creation, Inspection, Repair

AICP supports three operations the LLM can invoke on its own plugin ecosystem:

- `create_tool` — generate a new plugin from a description
- `contract_agent` — read a plugin's source, extract its contract
- `fix_tool` — modify an existing plugin

These compose into a loop: create → inspect → use → evaluate → repair. The loop is closed entirely within the agent; no human intervention is required.

### 6.2 Self-Inspection

`contract_agent` is itself a plugin. It can therefore inspect its own contract. This is not a special case but a consequence of the absence of privileges: a plugin that inspects plugins can inspect itself.

We take this self-inspection to be a minimal criterion for a self-bootstrapping agent system.

### 6.3 Sub-Agents Without Sub-Agent Machinery

Parallelism is achieved not by spawning agent instances but by invoking the main agent with a different `session_id`. `task_manager` packages the required metadata (callback receiver, task ID, session ID) so the LLM does not need to.

"Sub-agents" are thus a naming convention, not a mechanism. This is consistent with the broader principle: mechanisms are replaced by conventions.

---

## 7. The Meta-Model: Everything Is Message Flow

AICP reduces six paradigms to a single meta-model:

| Paradigm | AICP Expression |
|----------|------------------|
| Actor | Plugin reacting to Envelop |
| Message bus | `route(envelop, agent)` |
| Event sourcing | `_flow` append-only list |
| Microkernel | Minimal core + plugin files |
| Plugin architecture | Files on disk; no registry |
| Dependency injection | Single agent object |

The meta-model is: **messages flow; plugins react; state accumulates.**

No paradigm-specific machinery is retained. What remains is the intersection of the paradigms — the smallest structure that expresses all of them.

---

## 8. Cross-Domain Evidence

A single protocol description was provided to an LLM, which then generated systems in six unrelated domains:

| Domain | System |
|--------|--------|
| Operating systems | Microkernel with processes, memory, FS, IPC, scheduler |
| Quantum computing | Simulator with qubits, gates, Shor code, VQE |
| Computational biology | Protein folding with multi-agent molecular dynamics |
| Machine learning | 3D-parallel LLM trainer with All-Reduce and ZeRO |
| Number theory | Riemann Hypothesis exploration (Riemann–Siegel, Montgomery, GUE) |
| Hardware design | AI chip with ISA, compiler, chiplet interconnect |

No domain-specific training data and no framework documentation were provided. The protocol was the sole specification.

We do not claim these systems are production-grade. We claim the protocol is sufficient to convey the domain-independent structure an LLM needs to begin generating in a new domain.

---

## 9. Positioning

### 9.1 Relation to MCP

MCP standardizes tool invocation. AICP standardizes tool creation, inspection, and repair. The two are complementary: MCP could be exposed as a set of AICP plugins; AICP plugins could be wrapped as MCP tools.

### 9.2 Relation to Traditional Agent Frameworks

Traditional frameworks accumulate mechanisms. AICP removes them. The claim is not that AICP does more; it is that AICP does the same with less.

### 9.3 Relation to Serverless and HTTP

AICP shares with serverless the absence of instance management and with HTTP the absence of session state at the protocol level. It extends both by treating the agent itself as a serverless, stateless unit.

---

## 10. Discussion

### 10.1 Why "Nothing" Looks Like Nothing

A system with no kernel object, no context bus, no registry, and no scheduler looks empty. This appearance is not a defect: it is the consequence of removing every abstraction whose necessity was assumed rather than demonstrated.

### 10.2 The Cost of Reduction

Reduction has costs. There is no node-level optimization, no cross-node state synchronization, no long-lived agent identity. We regard these as acceptable: they are features of systems that require node-level concerns, which AICP does not address.

### 10.3 The Role of LLMs

AICP's minimalism is a precondition for LLM-nativeness. The protocol is simple enough that an LLM can internalize it from a few hundred tokens and generate correct code without framework documentation. Reduction and LLM-nativeness are not independent properties; the former enables the latter.

---

## 11. Conclusion

We have presented AICP, a protocol that reduces six canonical paradigms of distributed computing to a single meta-model: messages flow, plugins react, state accumulates.

The protocol has no agent instances, no context bus, no registry, and no scheduler. It is defined by roughly 200 lines of core specification. It supports cross-language replication, one-shot LLM code generation, self-inspection, self-bootstrapping, and cross-domain generation.

We propose that the protocol's minimalism is not a limitation but a design stance: the smallest structure capable of expressing what larger frameworks express, thereby imposing the least cognitive load on the LLMs that will increasingly be the primary authors of code.

> The protocol is the soul; code is the body.

---

## Appendix A: AICP in 30 Lines

```python
class Envelop:
    def __init__(self, sender="", receiver="", payload=None, meta=None):
        self.sender = sender
        self.receiver = receiver
        self.payload = payload or {}
        self.meta = meta or {}

plugins = {}

async def route(envelop, agent):
    plugin = plugins.get(envelop.receiver)
    return await plugin(envelop, agent)

async def execute(envelop, agent):
    envelop.payload = {"ok": True, "data": {...}}
    return envelop
```

---

## Appendix B: The Constitution (Summary)

- One plugin shape: `async def execute(envelop, agent)`
- One output per turn: JSON or plain text
- Large text via `@@CONTENT@@`
- Reasoning bounded by `think` length
- Sandbox replaces dangerous builtins
- Self-invocation routed through `task_manager`
- Unknown plugins inspected via `contract_agent`

---

## References

- Agha, G. (1986). *Actors: A Model of Concurrent Computation in Distributed Systems*.
- Armstrong, J. (2007). *Programming Erlang*.
- Fowler, M. (2005). *Event Sourcing*.
- Hewitt, C. (1973). *A Universal Modular Actor Formalism for Artificial Intelligence*.
- Liedtke, J. (1995). *On µ-Kernel Construction*.
- MCP Specification (Anthropic).
- AICP Protocol Specification v5.3.
- AICP Reference Implementations (Python, TypeScript).



