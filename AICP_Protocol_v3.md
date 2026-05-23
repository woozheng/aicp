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

## 3. Bus — Scatterer
```python
bus.publish(channel, envelop)
```
channel	Behavior
| `channel` | Behavior |
|-----------|----------|
| `"agent_id"` | Unicast |
| `"grp.xxx"` | Scatter to all subscribers |
| `""` | Discard |

Bus does not process content. It only scatters.

## 4. Router — Blind Router (Degenerate Agent)
```text
1. Receive Envelop
2. If ttl <= 0 → discard
3. ttl -= 1
4. Look up route entry by receiver
5. Not found → DEAD (natural termination)
6. sender == receiver → skip
7. Clear receiver
8. Run Workflow
9. After Workflow:
   - receiver set → publish
   - receiver empty → backtrack
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

## 6. Workflow — Plugin Chain
```yaml
steps:
  - plugin_a
  - plugin_b
  - plugin_c
```  

Sequential execution. Same Envelop passes through all steps. If any returns None, chain terminates.

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
```
No special intent. No error code. Just a receiver that doesn't exist.

## 10. Engine
```python
# Reference: 80 lines of Python
class Envelop: ...
class Bus: ...
async def engine_route(envelop): ...
async def workflow_run(envelop, agent): ...
```


## 11. Why So Simple
Because the protocol doesn't do anything. It only routes.

Scheduling? → Plugin writes meta.

Orchestration? → Plugin calls other plugins via HTTP.

Memory? → Plugin writes to a file.

Termination? → Plugin returns None or sets DEAD_xxx.

The Envelop carries control. The plugin does the work. The engine stays out of the way.

That's why it's 80 lines. That's why any language can implement it. That's why AI can understand it.

See [`How a Traditional Middleware-Based Content Moderation System Is Deconstructed by AICP`](AICP_BY_EXAMPLE.md)

See [`Distributed Mutual Exclusion + Causal Ordering + Shared Memory — Three Classic Problems, Deconstructed by One Field in AICP`](AICP_One.md)

## 12. Implement in Any Language

Read this spec

Read the 80-line Python reference

Implement in your language

Same Envelop format → cross-engine communication

Go, Rust, TypeScript, Zig — any language can join the information field.


*Still curious? [The philosophy and paradigm behind AICP →](AICP_PHILOSOPHY.md)*

AICP-Dvwoo&AI.