# AICP Protocol v3.0
Three atomic units. One Envelop. Infinite systems.

## 1. Philosophy
Every AI application is information flow. The protocol defines how information flows. The engine only routes. Plugins do everything else.

No scheduler. No message queue. No state machine. The Envelop carries its own control information.

## 2. Envelop
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
| Field | Who Sets | Purpose |
|-------|----------|---------|
| `sender` | Engine | Who sent it |
| `receiver` | Engine / Plugin | Where it goes |
| `intent` | Plugin | What it means |
| `payload` | Plugin | What it carries |
| `ttl` | Engine | Hop limit |
| `meta` | Plugin | Control info |

Lifecycle: Created → Routed → Processed → DEAD. Save Envelop to restore state. Envelop is the single source of truth.

## 3. Bus — Scatterer
```python
bus.publish(channel, envelop)
```
channel Behavior

| `channel` | Behavior |
|-----------|----------|
| `"agent_id"` | Unicast |
| `"grp.xxx"` | Scatter to all subscribers |
| `""` | Discard |
Bus does not process content. It only scatters.

## 4. Router — Blind Router (Degenerate Agent)
```text
0. Save original_receiver (Engine needs it, Plugin doesn't see it)
1. Receive Envelop
2. If ttl <= 0 → discard
3. ttl -= 1
4. Look up route entry by original_receiver
5. Not found → DEAD (natural termination)
6. sender == receiver → skip
7. Clear receiver
8. Run Workflow
9. After Workflow:
   - receiver set → publish to receiver
   - receiver empty + meta.backtrack_to exists → publish to backtrack_to
   - receiver empty → DEAD (natural termination, Envelop can be saved for later recovery)
10. Set sender = route_entry.id
```
A route entry is a mailbox with an identity:

```json
{
  "id": "agent_id",
  "workflow": ["memory", "agent_loop"],
  "prompts": "...",
  "brain": "model_id"
}
```
It does not think. It does not remember. It does not decide. It does one thing: receive the letter, run the process, send it out.

Intelligence lives in plugins. The Agent degenerates into a blind router — nothing but an identity and a processing chain.

A degenerate agent is a free agent.

## 5. Plugin — Processor
```python
async def execute(envelop, agent) -> Envelop | None:
    # Read payload
    # Process
    # Write payload
    # Return envelop (or None for DEAD)
```    
| Can Do | Cannot Do |
|--------|-----------|
| Modify `payload` | Modify `sender` |
| Modify `meta` | Modify `receiver` (except DEAD) |
| Return `None` (DEAD) | Modify `intent` |
| Call external services | Modify `channel_id` |

### Termination semantics:

return None → chain aborts immediately, no backtrack. Use for: validation failure, unrecoverable error.

return envelop with receiver="" → DEAD, natural pause. Use for: step completed, waiting for external event.

return envelop with receiver="DEAD_{uuid}" → explicit permanent death.

Two Plugin collaboration modes:

Orchestration: Plugin sets receiver, Engine routes to next Agent.

HTTP Call: Plugin calls another Plugin directly via HTTP, gets result synchronously.

## 6. Workflow — Plugin Chain
```yaml
steps:
  - plugin_a
  - plugin_b
  - plugin_c
Sequential execution. Same Envelop passes through all steps. If any returns None, chain terminates.
```
## 7. Groups
```python
bus.subscribe(agent_id, "grp.group_id")
```
Group messages scatter to all subscribers. Unsubscribe to leave. No central registry.

## 8. Round Robin
```json
"meta": {
  "round_robin": {
    "active": true,
    "agents": ["a", "b", "c"],
    "current": 0,
    "round": 0,
    "max_rounds": 2
  }
}
```
State lives on Envelop. Plugin updates the pointer. Engine maintains no scheduling state.

## 9. DEAD — Natural Termination
```python
# Plugin returns None → chain terminates
return None

# Plugin sets unresolvable receiver → message dies
envelop.receiver = f"DEAD_{uuid}"
return envelop

# Plugin returns with empty receiver → natural pause (can be revived)
envelop.receiver = ""
return envelop

```
No special intent. No error code. Just a receiver that doesn't exist.

## 10. Engine — Reference Implementation
```python
class Envelop:
    def __init__(self):
        self.sender = ""
        self.receiver = ""
        self.intent = ""
        self.payload = {}
        self.trace_id = ""
        self.message_id = ""
        self.channel_id = ""
        self.ttl = 10
        self.meta = {}

class Bus:
    def __init__(self):
        self._subscribers = {}  # agent_id -> [callback]
        self._groups = {}       # "grp.xxx" -> [agent_id]
    
    def subscribe(self, agent_id, channel, callback):
        if channel.startswith("grp."):
            self._groups.setdefault(channel, []).append(agent_id)
        else:
            self._subscribers.setdefault(agent_id, []).append(callback)
    
    def publish(self, channel, envelop):
        if not channel: return
        if channel.startswith("grp."):
            targets = self._groups.get(channel, [])
        else:
            targets = [channel] if channel in self._subscribers else []
        for tid in targets:
            for cb in self._subscribers.get(tid, []):
                cb(envelop.clone())

class Engine:
    def __init__(self, bus):
        self.bus = bus
        self.route_table = {}
    
    def register(self, agent_id, workflow_plugins):
        self.route_table[agent_id] = workflow_plugins
    
    async def route(self, envelop):
        if envelop.ttl <= 0: return
        envelop.ttl -= 1
        
        original_receiver = envelop.receiver
        if original_receiver not in self.route_table: return  # DEAD
        if envelop.sender == original_receiver: return        # skip
        
        envelop.receiver = ""
        for plugin in self.route_table[original_receiver]:
            result = await plugin.execute(envelop, None)
            if result is None: return  # chain terminated
            envelop = result
        
        envelop.sender = original_receiver
        if envelop.receiver:
            self.bus.publish(envelop.receiver, envelop)
        elif envelop.meta.get("backtrack_to"):
            self.bus.publish(envelop.meta["backtrack_to"], envelop)
        # else: DEAD (natural pause)
```        
80 lines. No scheduler. No queue. No state machine.

## 11. Envelop Lifecycle
```text
External Trigger (HTTP/Cron/MQ)
  │
  ▼
Envelop() ──→ Engine.route()
  │               │
  │          TTL>0? Route exists? sender≠receiver?
  │               │
  │          Run Workflow (plugins in sequence)
  │            │         │
  │        return None  return Envelop
  │            │         │
  │        chain dies   receiver set?
  │                     │      │
  │                    Yes     No
  │                     │      │
  │                publish   DEAD
  │                     │    (save Envelop
  │                     │     to restore later)
  └─────────────────────┘
  (new External Trigger with revived Envelop)
```
Envelop is the single source of truth. Save it → restore it → replay it. No external database needed for state.

## 12. meta Convention (Best Practice)
```json
{
  "meta": {
    "action": "confirm | retry | skip | rollback",
    "workflow_steps": ["step_a", "step_b"],
    "current_step_index": 0,
    "step_status": {"step_a": "awaiting_confirm | completed | failed | skipped"},
    "rollback_stack": ["step_a"],
    "backtrack_to": "agent_id",
    "retry_count": 1,
    "max_retries": 3,
    "round_robin": {"active": true, "agents": [], "current": 0, "round": 0, "max_rounds": 2}
  }
}
```
Not part of protocol spec. Convention only. Implementors choose their own keys.

## 13. Named Patterns (Reference Only)
| Pattern | Mechanism | Example |
|---------|-----------|---------|
| **Ping-Pong** | Set receiver → Engine routes → next Agent responds | Two agents chatting |
| **Chain-Step** | Each step returns empty receiver → DEAD → wait for external trigger | Onboarding with manual approval |
| **Scatter-Gather** | Publish to `grp.xxx` → collect results in payload | Voting, parallel processing |
| **Round-Robin** | Use `meta.round_robin` → Plugin updates pointer | Load balancing |
| **Fire-Forget** | Return None after side effect | Logging, notification |
| **Backtrack** | Set `meta.backtrack_to` before DEAD | Rollback, retry |
```python
class EchoPlugin:
    async def execute(self, envelop, agent):
        envelop.payload["echo"] = f"Hello from {envelop.sender}"
        return envelop  # receiver already set, Engine will route

bus = Bus()
engine = Engine(bus)

# Agent A: echoes and routes to B
engine.register("agent_a", [EchoPlugin()])

# Start
env = Envelop()
env.sender = "human"
env.receiver = "agent_a"
env.trace_id = "hello-001"
env.ttl = 5
await engine.route(env)
# Agent A processes → receiver remains "agent_a" → Engine publishes → no subscriber for "agent_a" → DEAD
```
## 15. Why So Simple
Because the protocol doesn't do anything. It only routes.

Scheduling? → Plugin writes meta.
Orchestration? → Plugin calls other plugins via HTTP.
Memory? → Plugin writes to a file.
Termination? → Plugin returns None or sets DEAD_xxx.
Pause/Resume? → DEAD + save Envelop + re-route later.

The Envelop carries control. The plugin does the work. The engine stays out of the way.

That's why it's 80 lines. That's why any language can implement it. That's why AI can understand it.

## 16. Implement in Any Language
Read this spec → Read the 80-line reference → Implement in your language → Same Envelop format → cross-engine communication.

Go, Rust, TypeScript, Zig — any language can join the information field.