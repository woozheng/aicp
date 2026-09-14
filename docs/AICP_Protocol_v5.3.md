# AICP Protocol v5.4

Agent Interaction & Communication Protocol

Three atomic units. One Envelop. Infinite systems.

## 1. Philosophy

Every AI application is information flow. The protocol defines how information flows. The engine only routes. Plugins do everything else.

- No scheduler
- No message queue
- No state machine
- The Envelop carries its own control information.

**v5.3 进化：** Agent 从退化路由器升级为能力容器。插件自注册成为约定。

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

### meta — Reserved Keys

The engine reads a small set of reserved keys from `meta`. Plugins **MUST NOT** use these for other purposes. All other keys are free for plugin use.

| Reserved Key | Set By | Purpose |
|--------------|--------|---------|
| callback_receiver | Caller | If set, engine executes the plugin asynchronously and delivers the result to this receiver |
| is_callback | Engine | Marks an Envelop as a callback delivery |
| callback_session_id | Caller / Engine | Session context carried through async callbacks |

#### Rules

- Engine **MUST** ignore unknown meta keys
- Plugins **MUST NOT** use reserved keys for custom purposes
- Plugins **MAY** add any other key to `meta`

## 3. Plugin — Processor

Plugin is the only place where intelligence lives. Engine routes. Plugin processes.

### Plugin Signature

```
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
| Read envelop.payload |
| Process the data |
| Write envelop.payload |
| Write envelop.meta (control info) |
| Set envelop.receiver to route to next plugin |
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

```
Registry.register(receiver, plugin_fn)
```

**v5.3 约定：** 插件自注册，无需手动集中注册。

### Plugin Example (Language Agnostic)

**Plugin:** camera  
**Receiver:** `device/camera`

**Input Envelop:**

```json
{
  "receiver": "device/camera",
  "payload": { "action": "take", "quality": 0.9 }
}
```

**Processing:**

1. Check `payload["action"] == "take"`
2. Call camera hardware
3. Write result to payload

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

## 4. Agent — Capability Container (v5.3)

| Version | Description |
|---------|-------------|
| v3.0 | Agent = Degenerate Router (only routes, no intelligence) |
| v5.3 | Agent = Capability Container (injected with capabilities) |

Agent is a container. It carries capabilities.

**Engine injects:**

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
| agent.config | System config dict |
| agent.log | Logger object |
| agent.data_dir | Path to data directory |
| agent.base_url | Base URL of engine |

## 5. Registry — Plugin Address Book

```
map[receiver] = plugin_fn
```

**v5.3 约定：** 插件自注册

```
Plugins register themselves to the Registry.
No manual central registration.

Registry.register(receiver, plugin_fn)
```

**Why?** Because plugin knows its own receiver. Central registration is extra work.

### receiver — Naming

`receiver` is a logical address. Its format is implementation-defined.

**Recommended conventions:**

- Use `/` as hierarchy separator
- Use lowercase with underscores

**Example:** `os/file_utils_api`, `builtins/tools/aicp_chat`

The engine does not enforce any format. The Registry simply maps the string to a plugin function.

## 6. Route — Engine Router

Route is the only behavior the engine defines. Everything else is plugin responsibility.

### Route Steps

```
1.  Receive Envelop
2.  If receiver is empty → discard
3.  If ttl <= 0 → DEAD
4.  ttl -= 1
5.  If receiver == sender → discard (loop protection)
6.  Look up receiver in Registry
7.  Not found → DEAD
8.  If meta.callback_receiver is set → async mode
    - Execute plugin in background
    - Return immediately with status "processing"
9.  Else → sync mode
    - Execute plugin with timeout
10. Plugin returns Envelop or None
11. If Envelop with receiver set → continue routing (go to step 1)
12. If Envelop with receiver empty → stop
13. If None → DEAD
```

### Notes

- **Loop protection:** `receiver == sender` is discarded to prevent self-loops
- **TTL:** each hop decrements `ttl` by 1; when `ttl` reaches 0, the message dies
- **Timeout:** sync mode enforces a timeout; on timeout, the Envelop is returned with an error in `payload`
- **Async mode:** when `callback_receiver` is set, the engine executes the plugin in the background and delivers the result via a new callback Envelop

### Engine Reference Implementation (~100 lines)

```python
# AICP Engine — reference implementation
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

## 7. DEAD — Natural Termination

```python
# Plugin returns None → chain terminates
return None

# Plugin sets unresolvable receiver → message dies
envelop.receiver = "DEAD_xxx"
return envelop
```

No special intent. No error code. Just a receiver that doesn't exist.

## 8. Scope — What the Protocol Does Not Define

This protocol defines message flow only.

It does **not** define:

- **Sandboxing** — plugins run with the same privileges as the engine
- **Permission control** — no access control is defined at the protocol level
- **Resource limits** — CPU, memory, and I/O limits are engine concerns
- **Security policies** — authentication, authorization, and encryption are engine concerns
- **Persistence** — no state is defined; plugins handle their own storage

These are engine-level concerns. A conforming engine **MAY** implement any of them. The protocol stays out of the way.

The protocol also does **not** define:

- **Return payload structure** — plugins may return any shape. Conventions such as `{"ok": true, "data": {...}}` are recommended but not required
- **Error classification** — the protocol only specifies that errors are placed in `payload`. How errors are categorized is implementation-defined
- **Inter-plugin call semantics** — plugins call each other via `agent.system.call()`, but the exact conventions (sender, trace inheritance) are plugin-level decisions

## 9. v5.3 — What Changed

| Concept | v3.0 | v5.3 |
|---------|------|------|
| Envelop | ✅ | ✅ (unchanged) |
| Plugin | ✅ | ✅ (signature unchanged) |
| Agent | Degenerate Router | Capability Container |
| Plugin Registration | Manual central registry | Self-registration (convention) |
| Engine | 80 lines | 80 lines (unchanged) |
| Philosophy | Information flow | Information flow (unchanged) |

## 10. Language Agnostic

Any language can implement this protocol:

| Language | Envelop | Plugin | Agent | Registry | Route |
|----------|---------|--------|-------|----------|-------|
| Python | class | async def | class | dict | async def |
| Flutter/Dart | class | Future | class | Map | Future |
| Go | struct | func | struct | map | func |
| Rust | struct | async fn | struct | HashMap | async fn |
| JavaScript | class | async | class | Map | async |

## 11. Why So Simple

Because the protocol doesn't do anything. It only routes.

- **Scheduling?** → Plugin writes `meta`
- **Orchestration?** → Plugin calls other plugins
- **Memory?** → Plugin writes to a file
- **Termination?** → Plugin returns `None`
- **UI?** → HTML + JS

> The Envelop carries control. The plugin does the work. The engine stays out of the way.

That's why it's 80 lines. That's why any language can implement it. That's why AI can understand it.

## 12. Protocol vs Implementation

```
Protocol = Pure definition (this document)
Engine   = Implementation of the protocol
Shell    = Engine + Hardware APIs + 7 platforms
```

AICP Shell is one implementation. Any language can implement this protocol.

## 13. Version History

| Version | Date | Change |
|---------|------|--------|
| v3.0 | 2026-05 | Initial protocol definition |
| v5.3 | 2026-07 | Agent = Capability Container, Self-registration, Engine reference |
| v5.4 | 2026-08 | Route steps expanded (loop protection, async mode, timeout), Scope section added, meta reserved keys defined |

---

*The protocol defines soul. Code implements flesh.*

**AICP-Dvwoo&AI. v5.4**



