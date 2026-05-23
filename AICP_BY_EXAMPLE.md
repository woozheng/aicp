# AICP by Example: Content Moderation Without Middleware

**How one Envelop replaces Kafka, Redis, a state machine, and a retry queue.**

---

## The Problem

A UGC platform needs to moderate user-generated content with these rules:

1. AI reviews all content first (toxicity, NSFW, sensitive keywords)
2. Borderline content (confidence 60-90%) goes to human review
3. Human reviewer can approve → publish, or reject → escalate to senior reviewer
4. Senior reviewer can approve → publish, or reject → ban with reason
5. High-risk content (>90%) gets banned immediately
6. Safe content (<60%) gets published immediately
7. Every step must be logged to an audit trail
8. Unreviewed content auto-escalates after 30 minutes
9. Batch uploads supported, each item tracked independently

**The traditional approach requires:**
Kafka — message queue
Redis — state storage
State machine — workflow logic
Cron job — timeout checking
Retry queue — escalation handling
Admin UI — human review interface
Audit system — compliance logging
Notification service — escalation alerts


That's a lot of infrastructure for what's essentially a flowchart.

---

## The AICP Approach: One Envelop, Four Plugins

### Step 1: Define the Workflow

```yaml
# config/workflows/content_audit.yaml
steps:
  - plugin: "ai_moderator"
  - plugin: "audit_router"
  - plugin: "audit_logger"
```
### Step 2: AI Review

```python
# plugins/ai_moderator.py
async def execute(envelop, agent):
    content = envelop.payload.get("content", "")
    
    result = await agent.system.llm.chat_json([
        {"role": "system", "content": "Rate content risk from 0-100. Return risk_score and risk_type."},
        {"role": "user", "content": content}
    ])
    
    # State lives on the Envelop, not in Redis
    envelop.meta["audit_state"] = {
        "stage": "ai_reviewed",
        "risk_score": result.get("risk_score", 0),
        "risk_type": result.get("risk_type", ""),
        "timestamp": time.time()
    }
    envelop.payload["audit_result"] = result
    return envelop
```    
### Step 3: Route Decision
```python
# plugins/audit_router.py
async def execute(envelop, agent):
    risk = envelop.meta["audit_state"]["risk_score"]
    
    if risk < 60:
        # Safe → publish immediately
        envelop.receiver = "publisher"
        envelop.intent = "PUBLISH"
        
    elif risk > 90:
        # Banned → ban immediately
        envelop.receiver = "ban_executor"
        envelop.intent = "BAN"
        
    else:
        # Borderline → route to human reviewer
        reviewer = await get_available_reviewer(agent.system, "level_1")
        envelop.receiver = reviewer
        envelop.intent = "HUMAN_REVIEW"
        envelop.meta["audit_state"]["deadline"] = time.time() + 1800  # 30 min timeout
        
    return envelop
```
### Step 4: Human Review
```python
# plugins/human_review_handler.py
async def execute(envelop, agent):
    decision = envelop.payload.get("human_decision", "")
    
    if decision == "approve":
        envelop.receiver = "publisher"
        envelop.intent = "PUBLISH"
    elif decision == "reject":
        # Escalate to senior reviewer
        senior = await get_available_reviewer(agent.system, "level_2")
        envelop.receiver = senior
        envelop.intent = "HUMAN_REVIEW"
        
    return envelop
```    
### Step 5: Audit Trail
```python
# plugins/audit_logger.py
async def execute(envelop, agent):
    state = envelop.meta.get("audit_state", {})
    
    audit = {
        "content_id": state.get("content_id"),
        "stage": state.get("stage"),
        "risk_score": state.get("risk_score"),
        "timestamp": time.time(),
        "decision": envelop.intent
    }
    
    with open("data/audit_log.jsonl", "a") as f:
        f.write(json.dumps(audit) + "\n")
    
    return envelop
```
### Step 6: Batch Upload (Automatic Parallelism)
```python
# plugins/batch_handler.py
async def execute(envelop, agent):
    contents = envelop.payload.get("contents", [])
    
    # One Envelop per item → return the list
    # Engine handles parallel delivery and result collection automatically
    return [
        AICPEnvelop(
            sender=agent.agent_id,
            receiver=agent.agent_id,
            intent="AUDIT",
            payload={"content": c, "content_id": gen_id()},
            meta={"audit_state": {"stage": "submitted"}}
        )
        for c in contents
    ]
```
### Architecture Comparison
| Component | Traditional Architecture | AICP |
|-----------|-------------------------|------|
| Message Delivery | Kafka | `bus.publish()` |
| State Storage | Redis | `envelop.meta` |
| Workflow Logic | Separate state machine | `audit_state.stage` on the Envelop |
| Timeout Handling | Cron job + retry queue | `timeout_checker` plugin |
| Human Review | Separate review system | Human Agent, same Envelop |
| Audit Logging | Log collection system | `audit_logger` plugin |
| Batch Processing | Batch consumer + concurrency control | Return list, engine parallelizes |
| Escalation | Notification service | Change receiver, Envelop keeps flowing |

## Why This Works
Traditional architectures keep state external. Kafka delivers the message, Redis stores the state, a separate state machine manages transitions, a cron job checks timeouts. Each boundary is a failure point.

AICP keeps state on the Envelop. The letter carries its own state. Where the Envelop goes, the state follows. No external coordination needed.

The plugin chain is the state machine

Returning a list is batch parallelism

Changing receiver is workflow routing

A timeout checker plugin is the cron job

The audit logger plugin is the compliance system

One rule. One Envelop. No middleware.