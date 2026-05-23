    # Distributed Mutual Exclusion + Causal Ordering + Shared Memory = The Ordered Collaborative Memory Problem

    ## Three Classic Problems, One Field

    Three stateless agents take turns speaking. After one round, each remembers everything said before them. Behind this lies the superposition of three classic distributed systems problems:

    - **Distributed Mutual Exclusion** — Only one can speak at a time
    - **Causal Ordering** — Bob must follow Alice, Carol must follow Bob
    - **Shared Memory** — Everyone must remember what everyone else said

    ---

    ## The Traditional Approach

    **Scheduling:**
    The Scheduler is the brain. Polls Redis to see "whose turn" → pulls messages from Kafka → checks `round` and `turn` → forwards the message to the current speaker → waits for reply → upon reply, updates `current` and `round` in Redis → triggers the next round. Every step is synchronous. Every step can time out.

    **Distributed Mutual Exclusion:**
    Kafka handles message routing → Redis stores "whose turn it is" → Scheduler polls Redis → Distributed Lock prevents two scheduler instances from sending simultaneously.

    **Causal Ordering:**
    Scheduler manually maintains `round` and `turn` → updates Redis after each utterance → the next speaker must wait for Redis to reflect the update before receiving the message.

    **Shared Memory:**
    PostgreSQL stores every utterance → CDC listens for changes → Notification Service pushes "new message" events → each Agent queries the database upon notification → when history grows long, add a Vector DB for retrieval.

    **Component list:** Scheduler, Kafka, Redis, Distributed Lock, PostgreSQL, CDC, Notification Service, Vector DB. **Eight.**

    ---

    ## AICP: One Field

    ```json
    "meta": {
        "round_robin": {
            "active": true,
            "agents": ["alice", "bob", "carol"],
            "current": 0,
            "max_rounds": 1,
            "round": 0
        }
    }
    ```
    ## How It Solves All Four Problems at Once
    ### Scheduling:
    No scheduler. The group channel grp.{group_id} fans out automatically — the engine delivers the message to all members simultaneously. But only agents[current] is the real receiver; everyone else returns DEAD. The message relays itself: Alice finishes → advances current → sends back to the group → engine fans out again → Bob processes. The engine only delivers. It never decides who should speak. The information relays itself.

    ### Distributed Mutual Exclusion:
    The pointer lives on the message. Whoever holds the message is the sole holder. Non-current speaker receives it → returns DEAD Envelop → silence. No group scatter forms. No lock needed. The message is naturally mutually exclusive.

    ### Causal Ordering:
    current is the order. current=1 must follow current=0. The message itself is the sequencer. No central scheduler required. All memory is naturally synchronized. Speakers complete causal utterances by order alone.

    ### Shared Memory:
    Every utterance continuously fans out through the group channel. Not speaking does not mean not processing memory. Memory is naturally ordered.

    ## In Summary:
    One group channel. One memory plugin — all ordered memory complete. One retrieval plugin — speakers achieve causal order. One speaker plugin controls utterances and advances the pointer after speaking. Three serial function plugins. N stateless agents. Stateless plugin chains. Serial causal utterances.

    Comparison
    | Problem | Traditional Components | AICP |
    |---------|------------------------|------|
    | Scheduling | Scheduler polling Redis, forwarding one by one | Group channel fan-out, information relays itself |
    | Distributed Mutual Exclusion | Kafka + Redis + Scheduler + Lock | `round_robin` pointer |
    | Causal Ordering | Scheduler maintains rounds | `current` field auto-increments |
    | Shared Memory | PostgreSQL + CDC + Notification + Vector DB | History travels with the message |
    | Total Components | **8** | **0 extra components** |
    | Failure Points | Every component can fail | State on the message, nothing to lose |

    ## In one sentence:

    Three classic problems. Eight system-level components vs. one field and three serial function plugins.