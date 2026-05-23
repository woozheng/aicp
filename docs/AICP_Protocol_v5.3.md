# AICP Protocol v5.3

**Three atomic units. One Envelop. Infinite systems.**

---

## 1. Philosophy

Every AI application is information flow. The protocol defines how information flows. The engine only routes. Plugins do everything else.

- No scheduler
- No message queue
- No state machine

The Envelop carries its own control information.

> **v5.3 进化**：Agent 从退化路由器升级为能力容器。插件自注册成为约定。

---

## 2. Envelop — The Only Data Carrier

```json
{
  "sender": "",
  "receiver": "",
  "intent": "",
  "payload": {},
  "trace_id": "",
  "message_id": "",
  "channel_id": "",
  "ttl": 10,
  "meta": {}
}
```

| Field | Type | Who Sets | Purpose |
|-------|------|----------|---------|
| sender | string | Engine / Plugin | Who sent it |
| receiver | string | Engine / Plugin | Where it goes |
| intent | string | Plugin | What it means (optional hint) |
| payload | map | Plugin | What it carries |
| trace_id | string | Engine | Trace across hops |
| message_id | string | Engine | Unique message ID |
| channel_id | string | Engine | Channel for routing |
| ttl | integer | Engine | Hop limit (decremented each hop) |
| meta | map | Plugin | Control info for plugins |

### Rules

- Plugins may only read and write `payload` and `meta`
- Setting `receiver` routes to the next plugin
- Return `None` to terminate the flow

---

## 3. Plugin — Processor

Plugin is the only place where intelligence lives. Engine routes. Plugin processes.

### Plugin Signature

```text
fn(envelop, agent) -> envelop | None
```

**Input:**

- `envelop` — The Envelop to process
- `agent` — Capability container injected by engine

**Output:**

- `Envelop` — Continue routing (if receiver is set)
- `None` — DEAD, terminate the flow

### Plugin Responsibilities

| Responsibilities |
|------------------|
| Read `envelop.payload` |
| Process the data |
| Write `envelop.payload` |
| Write `envelop.meta` (control info) |
| Set `envelop.receiver` to route to next plugin |
| Return Envelop or None |

### Plugin Constraints

| Can Do | Cannot Do |
|--------|-----------|
| Modify payload | Modify sender |
| Modify meta | Modify intent |
| Return None (DEAD) | Modify trace_id |
| Call external services | Modify message_id |
| Call other plugins via `agent.system.call()` | Modify channel_id |
| Set receiver to route | Set sender |

### Plugin Registration (v5.3)

```text
Plugin registers itself to the Registry.

Registry.register(receiver, plugin_fn)
```

> **v5.3 约定**：插件自注册，无需手动集中注册。

### Plugin Example (Language Agnostic)

**Plugin:** camera
**Receiver:** device/camera

**Input Envelop:**

```json
{
  "receiver": "device/camera",
  "payload": { "action": "take", "quality": 0.9 }
}
```

**Processing:**

- Check `payload["action"] == "take"`
- Call camera hardware
- Write result to payload

**Output Envelop:**

```json
{
  "payload": { "ok": true, "path": "/photos/photo.jpg" }
}
```

### Plugin Execution

```
Engine:
1. Look up receiver in Registry
2. Get plugin_fn
3. Call plugin_fn(envelop, agent)
4. Plugin returns Envelop or None
5. If Envelop:
   - Check receiver → continue routing or stop
   - If receiver empty → stop
   - If receiver set → continue
6. If None → DEAD (natural termination)
```

---

## 4. Agent — Capability Container (v5.3)

- **v3.0**: Agent = Degenerate Router (only routes, no intelligence)
- **v5.3**: Agent = Capability Container (injected with capabilities)

> Agent is a container. It carries capabilities.

Engine injects:

- `llm` → Call LLM
- `system` → Cross-plugin communication
- `scheduler` → Timers
- `config` → System config
- `log` → Logger
- `data_dir` → Data directory
- `base_url` → Engine base URL

Plugins can mount new capabilities onto Agent. Because intelligence lives in plugins. Agent holds the tools.

### Agent Capabilities Reference

| Tool | Returns | Description |
|------|---------|-------------|
| `agent.llm.chat(messages)` | str | Call LLM |
| `agent.llm.chat_json(messages)` | dict | Call LLM, return parsed JSON |
| `agent.llm.chat_stream(messages)` | AsyncIterator[str] | Streaming LLM call |
| `agent.system.call(envelop)` | Envelop | Cross-plugin communication |
| `agent.scheduler.create_timer(seconds, callback)` | timer_id | Create timer |

### Base Properties

| Property | Description |
|----------|-------------|
| `agent.config` | System config dict |
| `agent.log` | Logger object |
| `agent.data_dir` | Path to data directory |
| `agent.base_url` | Base URL of engine |

---

## 5. Registry — Plugin Address Book

```text
map[receiver] = plugin_fn
```

> **v5.3 约定**：插件自注册

```text
Plugins register themselves to the Registry.
No manual central registration.

Registry.register(receiver, plugin_fn)
```

Why? Because plugin knows its own receiver. Central registration is extra work.

---

## 6. Route — Engine Router

```
1. Receive Envelop
2. If ttl <= 0 → discard
3. ttl -= 1
4. Look up receiver in Registry
5. Not found → DEAD (natural termination)
6. receiver == sender → skip (infinite loop protection)
7. Execute Plugin
8. Plugin returns:
   - Envelop with receiver set → continue routing
   - Envelop with receiver empty → stop
   - None → DEAD
```

### Engine Reference Implementation (80 lines)

```python
# AICP Engine — 80 lines
# Any language can implement this.

class Envelop:
    def __init__(self, sender="", receiver="", intent="", payload=None, ...):
        self.sender = sender
        self.receiver = receiver
        self.intent = intent
        self.payload = payload if payload is not None else {}
        self.trace_id = f"tr_{uuid.uuid4().hex[:8]}"
        self.message_id = f"msg_{uuid.uuid4().hex[:6]}"
        self.channel_id = channel_id
        self.ttl = ttl
        self.meta = meta if meta is not None else {}

class Agent:
    def __init__(self, **kwargs):
        for key, value in kwargs.items():
            setattr(self, key, value)
    def get(self, key, default=None):
        return getattr(self, key, default)

plugins: Dict[str, Callable] = {}

async def route(envelop: Envelop, agent: Agent = None, timeout: float = 300.0) -> Optional[Envelop]:
    if not envelop.receiver:
        envelop.payload = {"error": "Missing receiver"}
        return envelop
    if envelop.ttl <= 0:
        envelop.payload = {"error": "TTL expired"}
        return envelop
    envelop.ttl -= 1
    plugin = plugins.get(envelop.receiver)
    if not plugin:
        envelop.payload = {"error": f"Plugin not found: {envelop.receiver}"}
        return envelop
    if agent is None:
        agent = Agent()
    try:
        result = await asyncio.wait_for(plugin(envelop, agent), timeout=timeout)
        return result
    except asyncio.TimeoutError:
        envelop.payload = {"error": f"Plugin timeout after {timeout}s"}
        return envelop
    except Exception as e:
        envelop.payload = {"error": f"Plugin execution error: {str(e)[:200]}"}
        return envelop
```

---

## 7. DEAD — Natural Termination

```python
# Plugin returns None → chain terminates
return None

# Plugin sets unresolvable receiver → message dies
envelop.receiver = "DEAD_xxx"
return envelop
```

No special intent. No error code. Just a receiver that doesn't exist.

---

## 8. v5.3 — What Changed

| Concept | v3.0 | v5.3 |
|---------|------|------|
| Envelop | ✅ | ✅ (unchanged) |
| Plugin | ✅ | ✅ (signature unchanged) |
| Agent | Degenerate Router | Capability Container |
| Plugin Registration | Manual central registry | Self-registration (convention) |
| Engine | 80 lines | 80 lines (unchanged) |
| Philosophy | Information flow | Information flow (unchanged) |

---

## 9. Language Agnostic

Any language can implement this protocol:

| Language | Envelop | Plugin | Agent | Registry | Route |
|----------|---------|--------|-------|----------|-------|
| Python | class | async def | class | dict | async def |
| Flutter/Dart | class | Future | class | Map | Future |
| Go | struct | func | struct | map | func |
| Rust | struct | async fn | struct | HashMap | async fn |
| JavaScript | class | async | class | Map | async |

---

## 10. Why So Simple

Because the protocol doesn't do anything. It only routes.

- **Scheduling?** → Plugin writes meta
- **Orchestration?** → Plugin calls other plugins
- **Memory?** → Plugin writes to a file
- **Termination?** → Plugin returns None
- **UI?** → HTML + JS

The Envelop carries control. The plugin does the work. The engine stays out of the way.

That's why it's 80 lines. That's why any language can implement it. That's why AI can understand it.

---

## 11. Protocol vs Implementation

```
Protocol = Pure definition (this document)
Engine   = Implementation of the protocol
Shell    = Engine + Hardware APIs + 7 platforms
```

AICP Shell is one implementation. Any language can implement this protocol.

---

## 12. Version History

| Version | Date | Change |
|---------|------|--------|
| v3.0 | 2026-05 | Initial protocol definition |
| v5.3 | 2026-07 | Agent = Capability Container, Self-registration, Engine reference |

---

*The protocol defines soul. Code implements flesh.*

**AICP-Dvwoo&AI. v5.3**



