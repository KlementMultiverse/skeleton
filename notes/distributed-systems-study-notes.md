# Distributed Systems & Matrix/Synapse — Level 19

> Curriculum level: Senior AI-Native Backend
> Prerequisites: L18 (async Python, Redis, PostgreSQL), L17 (FastAPI patterns)
> Last updated: 2026-04-14
> Covers: CAP/PACELC, consensus, event-driven patterns, Matrix protocol,
>          Synapse workers, Application Services, AI+Matrix integration

---

## Quick Reference

| Concept | One-line summary |
|---|---|
| CAP theorem | Of Consistency, Availability, Partition-tolerance — pick 2 under partition |
| PACELC | Even without partition: tradeoff between Latency and Consistency |
| Eventual consistency | All replicas converge *eventually* — no guarantee of when |
| Raft | Leader-based consensus: leader election + log replication + term numbers |
| 2PC | Coordinator + participants vote; blocks if coordinator crashes mid-flight |
| Saga | Long-running distributed transaction as a sequence of local txns + compensations |
| Matrix event DAG | JSON events linked by `prev_events` forming a causal DAG, never a chain |
| State resolution v2 | Deterministic merge of forked room state using power-level topological sort |
| Federation | Homeservers sign PDUs with Ed25519, exchange via HTTP PUT transactions |
| Application Service | External server that receives events from Synapse for a registered namespace |
| Outbox pattern | Write event to DB in same txn as business data; separate process publishes |
| CRDT | Merge function that is commutative + associative + idempotent → no coordination needed |

---

## PART 1: DISTRIBUTED SYSTEMS FUNDAMENTALS

### 1.1 CAP Theorem

**The Problem** — A distributed system replicates data across multiple nodes. What happens when a network partition isolates one node?

```
Node A  ─────── Network ─────── Node B
  ↑                               ↑
Write                           Read
```

When the network fails, you must choose:

- **Consistency** (C): Every read receives the most recent write or an error.
- **Availability** (A): Every request gets a non-error response (may be stale).
- **Partition Tolerance** (P): The system continues despite message loss between nodes.

Since partitions happen in real networks (cables fail, GC pauses cause timeouts, VMs migrate), **P is not optional in practice**. The real choice is **C vs A during a partition**.

**CP Systems** — Refuse to answer rather than return stale data:
- PostgreSQL with synchronous replication
- HBase, Zookeeper, etcd, Consul
- Redis in cluster mode (during certain failure scenarios)

Analogy: A bank ATM that goes offline rather than dispense cash it might not have.

**AP Systems** — Stay available but may return stale data:
- DynamoDB, Cassandra, CouchDB
- DNS (cached records serve after TTL)
- Matrix homeservers (federation delivers events eventually)

Analogy: Google Docs offline mode — you can keep editing, conflicts resolved later.

**Practical Rule:** CA (Consistent + Available without partition tolerance) only works on a single machine or within a single datacenter with a reliable network. Real distributed systems must tolerate partitions.

```python
# CP example: PostgreSQL synchronous replication
# Will block writes if replica disconnects
# host1 (primary): synchronous_standby_names = 'replica1'

# AP example: Cassandra with consistency level ONE
# Returns last known value even if replica is down
session.execute(
    SimpleStatement(
        "SELECT * FROM users WHERE id = %s",
        consistency_level=ConsistencyLevel.ONE  # AP: answer even if 2/3 nodes down
    ),
    [user_id]
)

# CP alternative: QUORUM requires majority (2/3 nodes)
session.execute(
    SimpleStatement(
        "SELECT * FROM users WHERE id = %s",
        consistency_level=ConsistencyLevel.QUORUM  # CP: error if majority unavailable
    ),
    [user_id]
)
```

### 1.2 PACELC Theorem

CAP only describes behavior *during partitions*. The PACELC theorem extends it: **even without a partition** (the normal case!), distributed systems face a tradeoff between **Latency** and **Consistency**.

```
P(artition) → A(vailability) vs C(onsistency)
E(lse, normal case) → L(atency) vs C(onsistency)
```

| System | Partition behavior | Normal behavior | Real-world example |
|---|---|---|---|
| DynamoDB | AP | EL (low latency) | Default: eventual consistency for speed |
| Cassandra | AP | EL | Configurable per-query via consistency level |
| PostgreSQL sync replication | CP | EC (wait for replica) | Synchronous commit |
| Zookeeper | CP | EC | Leader serializes all writes |
| MySQL semi-sync | CP | EC | Wait for at least one replica ACK |

**Practical implication:** When designing a system, ask TWO questions:
1. What should happen when nodes can't talk? (CAP)
2. How much latency can we accept for consistency in normal operation? (PACELC)

### 1.3 Eventual Consistency Patterns

"Eventual consistency" is not a single mechanism — it is a property achieved through specific techniques.

#### Read Repair

When a read query hits multiple replicas and detects disagreement, the coordinator writes the latest version back to stale replicas.

```
Client → Coordinator
              ↓ reads from 3 replicas
         [v=5, v=3, v=5]  ← replica 2 is stale
              ↓
         returns v=5 to client
         repairs replica 2 with v=5 in background
```

Used by: Cassandra, DynamoDB

```python
# Pseudocode: read repair in an application layer
async def read_with_repair(key: str, replicas: list[ReplicaClient]) -> Value:
    responses = await asyncio.gather(*[r.get(key) for r in replicas])
    latest = max(responses, key=lambda r: r.timestamp)

    stale = [r for r in responses if r.timestamp < latest.timestamp]
    for stale_replica in stale:
        asyncio.create_task(stale_replica.write(key, latest))  # fire and forget

    return latest.value
```

#### Anti-Entropy

Background process that continuously compares replicas and syncs differences. Uses a **Merkle tree** to efficiently find which data segments differ without comparing every row.

```
Replica A                    Replica B
┌─────────────────┐          ┌─────────────────┐
│ Root hash: abc  │◄── diff ─►│ Root hash: xyz  │
│ Left: aaa       │          │ Left: aaa       │  ← same, skip
│ Right: bbb      │          │ Right: yyy      │  ← different, drill down
└─────────────────┘          └─────────────────┘
                                     ↓
                              Sync rows 1000-2000
```

Used by: Cassandra's `nodetool repair`, S3 replication

#### Vector Clocks

A way to track causality without synchronized clocks. Each node maintains a vector of counters, one per node in the system.

```python
# Each event carries a vector clock
@dataclass
class VectorClock:
    clocks: dict[str, int] = field(default_factory=dict)

    def increment(self, node_id: str) -> "VectorClock":
        new = VectorClock(clocks=dict(self.clocks))
        new.clocks[node_id] = new.clocks.get(node_id, 0) + 1
        return new

    def merge(self, other: "VectorClock") -> "VectorClock":
        """Take element-wise max — used when merging concurrent updates."""
        all_nodes = set(self.clocks) | set(other.clocks)
        return VectorClock(clocks={
            n: max(self.clocks.get(n, 0), other.clocks.get(n, 0))
            for n in all_nodes
        })

    def happens_before(self, other: "VectorClock") -> bool:
        """Returns True if self causally precedes other."""
        return (
            all(self.clocks.get(n, 0) <= other.clocks.get(n, 0) for n in self.clocks)
            and any(self.clocks.get(n, 0) < other.clocks.get(n, 0) for n in self.clocks)
        )

    def concurrent_with(self, other: "VectorClock") -> bool:
        return not self.happens_before(other) and not other.happens_before(self)


# Usage: node A sends message at vc=[A:1, B:0]
# node B receives it, increments B: vc=[A:1, B:1]
# if two messages are concurrent_with each other → conflict, apply CRDT merge or show both
```

**Key insight:** If `vc_a.happens_before(vc_b)`, then event A causally precedes B. If they are concurrent, neither caused the other — this is a conflict.

#### CRDTs (Conflict-Free Replicated Data Types)

CRDTs define merge operations that are **commutative** (order doesn't matter), **associative** (grouping doesn't matter), and **idempotent** (merging same value twice has no effect). This means any two replicas can merge in any order and reach the same result.

**G-Counter (grow-only counter):**
```python
@dataclass
class GCounter:
    """Grow-only counter. Each node only increments its own slot."""
    counts: dict[str, int] = field(default_factory=dict)

    def increment(self, node_id: str, amount: int = 1) -> None:
        self.counts[node_id] = self.counts.get(node_id, 0) + amount

    def value(self) -> int:
        return sum(self.counts.values())

    def merge(self, other: "GCounter") -> "GCounter":
        """Commutative, associative, idempotent — safe to call in any order."""
        all_nodes = set(self.counts) | set(other.counts)
        return GCounter(counts={
            n: max(self.counts.get(n, 0), other.counts.get(n, 0))
            for n in all_nodes
        })
```

**PN-Counter (positive-negative, supports decrement):**
```python
@dataclass
class PNCounter:
    increments: GCounter = field(default_factory=GCounter)
    decrements: GCounter = field(default_factory=GCounter)

    def increment(self, node_id: str) -> None:
        self.increments.increment(node_id)

    def decrement(self, node_id: str) -> None:
        self.decrements.increment(node_id)

    def value(self) -> int:
        return self.increments.value() - self.decrements.value()

    def merge(self, other: "PNCounter") -> "PNCounter":
        return PNCounter(
            increments=self.increments.merge(other.increments),
            decrements=self.decrements.merge(other.decrements),
        )
```

**LWW-Register (Last Write Wins):**
```python
@dataclass
class LWWRegister:
    value: Any
    timestamp: float  # monotonic clock or Hybrid Logical Clock

    def write(self, new_value: Any, ts: float) -> None:
        if ts > self.timestamp:
            self.value = new_value
            self.timestamp = ts

    def merge(self, other: "LWWRegister") -> "LWWRegister":
        if other.timestamp > self.timestamp:
            return other
        return self
```

Real-world CRDT users: Redis (counters), Riak (all data types), Figma (collaborative canvas), Notion (block editor), Matrix (presence aggregation).

---

### 1.4 Consensus Algorithms

Consensus is the problem of getting multiple nodes to agree on a single value, even when some nodes crash. This is harder than it sounds — Paxos is notoriously difficult to reason about.

#### Raft

Raft was designed to be understandable. It decomposes consensus into three problems:

1. **Leader election** — elect exactly one leader per term
2. **Log replication** — leader replicates entries to followers
3. **Safety** — if an entry is committed, all future leaders have it

**The term number is a logical clock.** Every Raft message carries a term. If a node sees a higher term, it immediately becomes a follower.

```
Terms:        [1]    [2]    [3]    [4]
                     ↑leader      ↑leader
                 leader dies      new election
```

**Leader Election:**
```
Initial state: all nodes are Followers

1. Follower times out waiting for heartbeat (150-300ms randomized)
2. Increments its term: term = current + 1
3. Votes for itself
4. Sends RequestVote(term, candidateId, lastLogIndex, lastLogTerm) to all
5. If majority responds with granted=true → becomes Leader
6. Immediately sends heartbeat AppendEntries to assert leadership

Vote is granted only if:
  - Requester's term >= voter's current term
  - Voter hasn't voted in this term yet
  - Requester's log is at least as up-to-date as voter's log
    (lastLogTerm > voter's lastLogTerm, OR equal terms with longer log)
```

**Log Replication:**
```python
# Pseudocode: leader receives client write
async def leader_append(entry: LogEntry) -> bool:
    # 1. Append to own log (uncommitted)
    log.append(entry)

    # 2. Send AppendEntries to all followers in parallel
    acks = await asyncio.gather(*[
        follower.append_entries(
            term=current_term,
            leaderId=node_id,
            prevLogIndex=len(log) - 2,
            prevLogTerm=log[-2].term if len(log) >= 2 else 0,
            entries=[entry],
            leaderCommit=commit_index,
        )
        for follower in followers
    ], return_exceptions=True)

    # 3. Commit once MAJORITY acknowledges (including self = N//2 + 1)
    success_count = sum(1 for a in acks if a is True) + 1  # +1 for self
    if success_count > len(followers) // 2 + 1:
        commit_index = len(log) - 1
        apply_to_state_machine(entry)
        return True

    return False  # not committed, retry or report failure
```

**Key Raft properties:**
- Only leaders accept writes
- A leader only commits entries from its own term (no re-committing old entries)
- `AppendEntries` includes a consistency check (`prevLogIndex` + `prevLogTerm`) — followers reject if they don't match (forces log convergence)
- Split-brain is prevented because only a candidate with a majority of votes can win, and it must have the most up-to-date log

**Used by:** etcd (Kubernetes state), CockroachDB, TiDB, Consul, RabbitMQ Quorum queues

#### Paxos (brief)

Paxos is older and more general, but notoriously difficult to implement correctly. Two phases:

1. **Prepare phase:** Proposer sends `Prepare(n)` to acceptors. Acceptors promise to ignore proposals < n and return the highest proposal they already accepted.
2. **Accept phase:** Proposer sends `Accept(n, value)`. Acceptors accept if they haven't promised a higher n.

The key insight: a value is "chosen" when a majority of acceptors accept the same value in round n. This majority overlap with any future majority guarantees that the value is never lost.

Practical implementations (Multi-Paxos, Fast Paxos, Flexible Paxos) add leader roles to avoid the prepare phase on every round — making them similar to Raft.

---

### 1.5 Two-Phase Commit (2PC)

**The Problem:** Distributed transaction across two databases. How do you ensure both commit or both roll back?

```
Coordinator
   │
   ├─ Phase 1 (Prepare): "Can you commit transaction X?"
   │       ├──→ DB1: "Yes (vote COMMIT)"
   │       └──→ DB2: "Yes (vote COMMIT)"
   │
   └─ Phase 2 (Commit): "Please commit transaction X."
           ├──→ DB1: ACK
           └──→ DB2: ACK
```

**Why 2PC blocks:**

The coordinator holds a lock while waiting for Phase 2 ACKs. If the coordinator crashes after Phase 1 but before Phase 2, participants are blocked indefinitely — they voted to commit but cannot commit without the coordinator's instruction, and they cannot roll back because another participant may have already committed.

```python
# 2PC pseudocode — shows the blocking problem
async def two_phase_commit(participants: list[Participant], txn: Transaction) -> bool:
    # Phase 1: Prepare
    votes = await asyncio.gather(*[p.prepare(txn) for p in participants])

    if all(v == "YES" for v in votes):
        # Phase 2: Commit — if coordinator crashes HERE, participants block forever
        await asyncio.gather(*[p.commit(txn) for p in participants])
        return True
    else:
        await asyncio.gather(*[p.abort(txn) for p in participants])
        return False
```

**When 2PC is acceptable:**
- Short, bounded transactions (milliseconds)
- All participants are in the same datacenter
- Coordinator HA is handled (e.g., XA transactions with a durable coordinator log)

**When to avoid 2PC:**
- Cross-datacenter (latency multiplied, higher partition probability)
- Long-running workflows (seconds to minutes)
- Microservices owned by different teams (you can't impose 2PC on a third-party API)

---

### 1.6 Saga Pattern

The Saga pattern replaces 2PC for long-running distributed transactions. Instead of one atomic transaction, a saga is a sequence of local transactions where each step publishes an event or sends a command that triggers the next step.

If any step fails, **compensating transactions** are executed in reverse to undo completed steps.

**Analogy:** Booking a trip — you book flight, hotel, and car separately. If the car rental fails, you cancel the hotel and flight (compensations), not roll back a single atomic transaction.

#### Choreography-based Saga

Each service listens for events and publishes its own events. No central coordinator.

```python
# Order service publishes OrderCreated
async def create_order(order_data: dict) -> None:
    order = await db.create_order(order_data)
    await message_bus.publish("OrderCreated", {
        "order_id": order.id,
        "items": order.items,
        "total": order.total,
    })

# Payment service listens for OrderCreated, publishes PaymentProcessed or PaymentFailed
async def on_order_created(event: dict) -> None:
    try:
        payment = await payment_gateway.charge(event["total"])
        await message_bus.publish("PaymentProcessed", {
            "order_id": event["order_id"],
            "payment_id": payment.id,
        })
    except PaymentError:
        await message_bus.publish("PaymentFailed", {
            "order_id": event["order_id"],
            "reason": "insufficient_funds",
        })

# Order service listens for PaymentFailed → compensate by cancelling order
async def on_payment_failed(event: dict) -> None:
    await db.cancel_order(event["order_id"])
    await message_bus.publish("OrderCancelled", event)
```

**Pros of Choreography:**
- Decoupled — services don't know about each other directly
- No single point of failure

**Cons:**
- Hard to see the big picture — saga state is spread across all services
- Difficult to debug: "Which saga step failed and why?"
- Cyclic dependencies possible if not careful

#### Orchestration-based Saga

A dedicated orchestrator service tells each participant what to do and tracks state.

```python
class OrderSagaOrchestrator:
    """Central coordinator that knows the full saga flow."""

    async def execute(self, order_id: str) -> SagaResult:
        saga_state = SagaState(order_id=order_id, completed_steps=[])

        try:
            # Step 1: Reserve inventory
            reservation = await inventory_service.reserve(order_id)
            saga_state.completed_steps.append(("reserve_inventory", reservation.id))

            # Step 2: Process payment
            payment = await payment_service.charge(order_id)
            saga_state.completed_steps.append(("process_payment", payment.id))

            # Step 3: Ship order
            shipment = await shipping_service.schedule(order_id)
            saga_state.completed_steps.append(("schedule_shipment", shipment.id))

            return SagaResult.success(saga_state)

        except Exception as e:
            # Compensate in reverse order
            await self._compensate(saga_state, e)
            return SagaResult.failed(saga_state, error=str(e))

    async def _compensate(self, state: SagaState, error: Exception) -> None:
        for step_name, step_id in reversed(state.completed_steps):
            match step_name:
                case "schedule_shipment":
                    await shipping_service.cancel(step_id)
                case "process_payment":
                    await payment_service.refund(step_id)
                case "reserve_inventory":
                    await inventory_service.release(step_id)
```

**Pros of Orchestration:**
- Clear visibility of saga state in one place
- Easy to add steps, timeouts, retries
- Better for complex branching logic

**Cons:**
- Orchestrator is a new service to maintain
- Can become a bottleneck or god class if not bounded

#### Idempotency Keys

Every saga step must be idempotent — safe to retry. Use an idempotency key to deduplicate:

```python
async def process_payment(
    order_id: str,
    amount: Decimal,
    idempotency_key: str,  # must be unique per attempt
) -> Payment:
    # Check if we already processed this
    existing = await db.fetchrow(
        "SELECT * FROM payments WHERE idempotency_key = $1",
        idempotency_key
    )
    if existing:
        return Payment.from_row(existing)  # return cached result, don't charge again

    # Process for real
    payment = await stripe.charge(amount)

    # Store with idempotency key
    await db.execute(
        "INSERT INTO payments (order_id, idempotency_key, stripe_id, amount) "
        "VALUES ($1, $2, $3, $4)",
        order_id, idempotency_key, payment.id, amount
    )
    return payment
```

Pattern: `{resource_type}:{resource_id}:{operation}:{attempt_uuid}`

Example: `payment:order_42:charge:a1b2c3d4`

---

## PART 2: MATRIX PROTOCOL DEEP DIVE

### 2.1 What Is Matrix?

Matrix is an **open standard for federated real-time communication**. Unlike Slack (centralized) or email (federated but with separate inboxes), Matrix is:

- **Federated:** Alice on `matrix.org` can talk to Bob on `synapse.example.com` directly — no central server needed.
- **Persistent:** All messages are stored as an append-only event log.
- **Decentralized:** No single entity controls the network. Anyone can run a homeserver.
- **Verifiable:** All events are cryptographically signed by the originating homeserver.

**Analogy:** Matrix is to chat what email is to messages — federated, open, and anyone can participate — but with real-time delivery, end-to-end encryption, and media support built in.

### 2.2 The Event DAG

Every Matrix room is a **Directed Acyclic Graph (DAG) of JSON events**. This is the core data structure.

```
m.room.create ← m.room.join(alice) ← m.room.join(bob) ← m.room.message(alice, "hello")
                                                      ↑                   ↑
                                                      └───── prev_events ──┘
```

Why a DAG and not a chain? Because federation means two servers can receive messages simultaneously and both publish new events referencing the same `prev_events`. This creates a fork that must be merged via state resolution.

```
Server A timeline:      event_5a ─── event_6a ─┐
                                                ├── event_7 (both prev_events referenced)
Server B timeline:      event_5b ─── event_6b ─┘
```

**Event anatomy — every field matters:**

```json
{
  "event_id": "$abc123:matrix.org",
  "type": "m.room.message",
  "sender": "@alice:matrix.org",
  "room_id": "!roomid:matrix.org",
  "origin_server_ts": 1713100000000,
  "depth": 42,
  "content": {
    "msgtype": "m.text",
    "body": "Hello, World!"
  },
  "prev_events": [
    "$prev1:matrix.org",
    "$prev2:matrix.org"
  ],
  "auth_events": [
    "$create_event:matrix.org",
    "$join_alice:matrix.org",
    "$power_levels:matrix.org"
  ],
  "hashes": {
    "sha256": "base64encodedcontenthashhereXXX"
  },
  "signatures": {
    "matrix.org": {
      "ed25519:key1": "base64sighere..."
    }
  },
  "unsigned": {
    "age": 1234
  }
}
```

**Field explanations:**

| Field | Purpose |
|---|---|
| `event_id` | Globally unique. Computed as SHA-256 hash of canonical JSON, URL-safe base64 encoded with `$` prefix (room version 4+) |
| `type` | Event type namespace. `m.room.*` = state events. `m.room.message` = timeline event. Custom types allowed. |
| `sender` | Full Matrix user ID (`@localpart:server`) |
| `room_id` | Full room ID (`!localpart:server`). Opaque — do not parse the localpart. |
| `origin_server_ts` | Milliseconds since epoch on the *sender's* clock. **NOT trustworthy for ordering** — servers can lie. Use `depth` and DAG position for ordering. |
| `depth` | Integer depth in the DAG. Each event's depth = max(prev_events depths) + 1. Monotonically increasing. |
| `prev_events` | List of event IDs this event directly follows in the DAG. Creates the "timeline" edges. |
| `auth_events` | The specific state events that authorize this event. Immutable reference to the auth chain. |
| `hashes` | Content hash. Used to detect modification in transit. |
| `signatures` | Ed25519 signatures. The originating server signs; other servers may countersign. |
| `unsigned` | Data not covered by signatures. Safe to add in transit. `age` is added by your homeserver. |

### 2.3 Room State vs Timeline Events

**State events** define the current properties of the room. They have a `state_key` field:

```json
{
  "type": "m.room.member",
  "state_key": "@alice:matrix.org",
  "content": {
    "membership": "join",
    "displayname": "Alice"
  }
}
```

The **room state** at any point is the set of `(type, state_key)` pairs, each holding the most recent event with that pair. It is like a key-value store embedded in the DAG.

**State event types:**

| Type | state_key | Purpose |
|---|---|---|
| `m.room.create` | `""` | First event in room; sets `room_version` |
| `m.room.member` | `@user:server` | Join/leave/ban/invite/knock |
| `m.room.name` | `""` | Human-readable room name |
| `m.room.topic` | `""` | Room description |
| `m.room.power_levels` | `""` | Who can do what (user power levels, event type levels) |
| `m.room.join_rules` | `""` | `public`, `invite`, `knock`, `restricted` |
| `m.room.history_visibility` | `""` | Who can see history |
| `m.room.encryption` | `""` | Enables E2EE; sets algorithm |
| `m.room.canonical_alias` | `""` | Primary alias |

**Timeline events** have no `state_key` and only affect the DAG timeline:

- `m.room.message` — text, media, files
- `m.reaction` — emoji reactions (relates to another event)
- `m.room.redaction` — redact (soft-delete) an event
- `m.call.*` — VoIP signalling
- Custom types for application services

### 2.4 State Resolution Algorithm v2

When two homeservers each advance the room DAG simultaneously, a **fork** occurs. Both are valid continuations but lead to different room states. State resolution v2 is the deterministic algorithm all servers run to converge on the same merged state.

**Why determinism matters:** Server A and Server B must run this algorithm and arrive at the *identical* state without consulting each other. If they disagreed on state, they would reject each other's events as unauthorized.

**The algorithm (simplified):**

```
Input: a set of state events from the tips of all DAG forks

Step 1: Split into conflicted and unconflicted sets
  - For each (type, state_key) pair, if all forks agree → unconflicted
  - If forks disagree on a (type, state_key) → conflicted

Step 2: Collect auth difference events
  - Auth events present in some forks but not others
  - These must be resolved first to validate the conflicted events

Step 3: Reverse topological sort of control events (power level events, join rules)
  - Use Kahn's algorithm on the auth DAG
  - Tie-break by: effective power level (DESC), origin_server_ts (ASC), event_id (ASC lexicographic)

Step 4: Apply control events to partial state
  - Authenticate each against the partial state built so far
  - Invalid events are skipped (cannot grant themselves power)

Step 5: Build power level mainline
  - Trace power_level events backward to room creation
  - This creates a "mainline" of power level epochs

Step 6: Sort remaining conflicted state events by mainline order
  - Each event is placed at the closest mainline event it "knows about"
  - Tie-break: origin_server_ts, event_id

Step 7: Apply remaining state events

Step 8: Restore unconflicted state (takes final precedence)
```

**Why power levels first?** If Alice and Bob both try to dethrone each other by changing power levels simultaneously, the algorithm must decide whose power level change wins before evaluating whether their subsequent actions were authorized.

**State Resolution v2.1 (Room Version 12, 2025):**
Project Hydra (MSC4297) fixes an edge case where a malicious server could cause state resets. It changes the starting state and replays more events: not just conflicted events but the entire conflicted subgraph. Room IDs now equal the hash of the creation event (eliminates room ID collisions).

### 2.5 Federation: How Homeservers Communicate

**Every homeserver is an independent peer.** When Alice on `matrix.org` sends a message in a room that also has Bob on `example.com`, `matrix.org` must deliver the event to `example.com`.

```
Alice@matrix.org                            Bob@example.com
       │                                          │
  matrix.org ──── PUT /federation/v1/send ──→ example.com
  (signs PDU with                           (verifies signature,
   ed25519 key)                              applies to DAG,
                                             delivers to Bob)
```

**Key federation concepts:**

**PDU (Persistent Data Unit):** A state or timeline event that is persisted in the DAG. Signed by originating server.

**EDU (Ephemeral Data Unit):** Presence updates, typing notifications, read receipts. Not signed, not stored in DAG.

**Transaction:** The transport envelope. A batch of PDUs and EDUs sent in a single HTTP request.

**Federation transaction endpoint:**
```
PUT /_matrix/federation/v1/send/{txnId}
Authorization: X-Matrix origin="matrix.org", key="ed25519:key1", sig="base64sig"
Content-Type: application/json

{
  "origin": "matrix.org",
  "origin_server_ts": 1713100000000,
  "pdus": [...],
  "edus": [...]
}
```

The `Authorization` header uses a custom `X-Matrix` scheme with the origin server's name, key ID, and signature over the request body. This allows the receiving server to verify the sender's identity by fetching the public key from `/_matrix/key/v2/server` on the origin.

**Fetching missed events:**
If a homeserver receives an event referencing `prev_events` it hasn't seen, it fetches them via:
```
GET /_matrix/federation/v1/event/{eventId}
GET /_matrix/federation/v1/backfill/{roomId}?v=<event_id>&limit=20
```

**Server discovery — `.well-known` delegation:**
```
# matrix.org/.well-known/matrix/server
{
  "m.server": "matrix-federation.example.com:8448"
}
```

This allows `matrix.org` to run its federation endpoint on a different host/port than the base domain.

### 2.6 Client-Server API

Clients communicate with their own homeserver only. The key flows:

**Sending a message:**
```
POST /_matrix/client/v3/rooms/{roomId}/send/{eventType}/{txnId}
Authorization: Bearer {access_token}

{
  "msgtype": "m.text",
  "body": "Hello!"
}
```

Response: `{"event_id": "$abc123:matrix.org"}`

The `txnId` is a client-generated identifier for idempotency. If the client retries, Synapse returns the same `event_id` without creating a duplicate.

**The sync loop:**
```
GET /_matrix/client/v3/sync?since={next_batch}&timeout=30000
Authorization: Bearer {access_token}
```

- `since`: Pagination token from previous sync. Omit for initial sync.
- `timeout`: Long-polling timeout in milliseconds. If no events in 30s, returns empty.
- Response includes: `next_batch` token, `rooms.join`, `rooms.invite`, `rooms.leave`, `presence`, `account_data`

**Incremental sync response structure:**
```json
{
  "next_batch": "s72595_4483_1934",
  "rooms": {
    "join": {
      "!roomid:server": {
        "timeline": {
          "events": [...],
          "limited": false,
          "prev_batch": "t47429-4392820_219380_26003_2265"
        },
        "state": {
          "events": [...]
        },
        "ephemeral": {
          "events": [
            {"type": "m.typing", "content": {"user_ids": ["@bob:server"]}}
          ]
        }
      }
    }
  }
}
```

**Room versions:**
- v1–v3: legacy, deprecated
- v4: event IDs are content hashes (replaces human-readable IDs, prevents forgery)
- v5: stricter enforcement of event signing
- v6: power level integers (not floats)
- v7-v9: knock/restricted join rules
- v10: `m.room.create` uses integer power levels
- v11: `m.room.redaction` uses `redacts` in content (not top-level)
- v12 (2025): state resolution v2.1, room ID = hash of creation event

Always create new rooms with the latest stable version (currently v11 or v12).

### 2.7 Ephemeral Events

These are delivered via sync but never stored in the DAG:

**Typing notifications:**
```
PUT /_matrix/client/v3/rooms/{roomId}/typing/{userId}
{"typing": true, "timeout": 30000}
```

**Read receipts:**
```
POST /_matrix/client/v3/rooms/{roomId}/receipt/m.read/{eventId}
{}
```

**Presence:**
```
PUT /_matrix/client/v3/presence/{userId}/status
{"presence": "online", "status_msg": "Available"}
```

Synapse has `presence_enabled` config (default false in large deployments) — presence is expensive to federate.

---

## PART 3: SYNAPSE — THE REFERENCE HOMESERVER

### 3.1 Architecture Overview

Synapse is written in Python using a hybrid of **asyncio** and **Twisted** (Twisted's reactor drives I/O, asyncio coroutines handle business logic). This is historical — older code is Twisted callbacks/Deferreds, newer code is `async def`.

**Single process (development):**
```
Synapse main process
├── HTTP listeners (client API, federation, admin)
├── Background tasks (room complexity, notifications)
├── Event persistence (writes to PostgreSQL)
└── Federation sender (HTTP to other homeservers)
```

**Workers (production):**
```
┌─────────────────────────────────────────────────────────┐
│                     Redis (pub/sub)                      │
│  replication streams, inter-worker coordination          │
└─────────────────────────────────────────────────────────┘
         ↑           ↑           ↑           ↑
    [main]    [federation   [event      [client
              _reader]      _creator]   _reader x3]
         ↑           ↑           ↑           ↑
┌─────────────────────────────────────────────────────────┐
│                   PostgreSQL                             │
│  (shared read/write, all workers use same DB)            │
└─────────────────────────────────────────────────────────┘
```

### 3.2 Worker Types

| Worker app | Role | Scales? |
|---|---|---|
| `synapse.app.generic_worker` | Client reads, push notifications, media | Yes |
| `synapse.app.generic_worker` (federation_reader) | Incoming federation | Yes |
| `synapse.app.generic_worker` (event_creator) | Create and persist events | Yes |
| `synapse.app.generic_worker` (federation_sender) | Outgoing federation | Single instance |
| `synapse.app.generic_worker` (pusher) | Push gateway | Single instance |
| `synapse.app.media_repository` | Media uploads/downloads | Single instance |
| `synapse.app.user_dir` | User directory search | Single instance |
| `synapse.app.background_worker` | Background tasks (e.g., stats) | Single instance |

**Stream writers** are designated workers that can write specific event streams. Only one process per stream type can be a writer. Examples:
- `events` stream: written by `event_creator` workers
- `to_device` stream: can be written by generic workers
- `account_data` stream: main or designated worker

### 3.3 Homeserver Configuration

Key sections of `homeserver.yaml`:

```yaml
# Identity
server_name: "example.com"
pid_file: /var/run/matrix-synapse.pid

# Public base URL for media links
public_baseurl: "https://matrix.example.com/"

# Listeners
listeners:
  - port: 8008
    tls: false
    bind_addresses: ['::1', '127.0.0.1']
    type: http
    x_forwarded: true
    resources:
      - names: [client, federation]
        compress: false

# Database (SQLite only for dev — PostgreSQL required for workers and production)
database:
  name: psycopg2
  args:
    host: "postgres"
    port: 5432
    database: synapse
    user: synapse
    password: "strongpassword"
    cp_min: 5
    cp_max: 10

# Media storage
media_store_path: /data/media_store

# Logging
log_config: "/data/log.config"

# Workers: Redis required
redis:
  enabled: true
  host: redis
  port: 6379

# Registration
enable_registration: false
registration_requires_token: true

# Federation
federation_domain_whitelist:  # empty = allow all
allow_public_rooms_without_auth: false

# Rate limiting
rc_message:
  per_second: 0.2
  burst_count: 10

rc_login:
  address:
    per_second: 0.003
    burst_count: 5
  account:
    per_second: 0.003
    burst_count: 5

# Room complexity limit (prevent joining enormous rooms)
limit_remote_rooms:
  enabled: true
  complexity: 1.0

# Application services
app_service_config_files:
  - /data/appservice-mybot.yaml

# Security
form_secret: "<long random string>"
macaroon_secret_key: "<long random string>"
signing_key_path: /data/example.com.signing.key
```

### 3.4 Admin API

Synapse provides an admin API (requires `_synapse/admin` on a non-public listener or with admin token):

```bash
# List rooms
GET /_synapse/admin/v1/rooms?limit=20&order_by=size

# Get room details
GET /_synapse/admin/v1/rooms/{roomId}

# Purge old events from a room
POST /_synapse/admin/v1/purge_history/{roomId}
{"purge_up_to_ts": 1713100000000}

# Deactivate user
POST /_synapse/admin/v1/deactivate/{userId}
{"erase": true}

# List users
GET /_synapse/admin/v2/users?guests=false&deactivated=false

# Server statistics
GET /_synapse/admin/v1/server_version
GET /_synapse/admin/v1/statistics/database
```

---

## PART 4: APPLICATION SERVICE (AS) INTEGRATION

### 4.1 What Is an Application Service?

An Application Service is an external server that registers with Synapse and receives events for a **namespace** of users, room aliases, or room IDs. It is the primary extension mechanism for Matrix — used to build:

- **Bridges:** Connect Matrix to Slack, Discord, WhatsApp, IRC, Telegram
- **Bots:** AI assistants, moderation bots, notification services
- **Infrastructure:** Monitoring, logging, analytics

**The AS acts like a privileged Synapse plugin that runs outside the main process.**

Key privileges:
- Receives all events matching its namespaces
- Can masquerade as any user in its namespace without their password
- Bypasses rate limiting (configurable)
- Can create "virtual users" that appear to clients as real Matrix users

### 4.2 Registration File

The AS registers with Synapse via a YAML file on the Synapse server:

```yaml
# /data/appservice-mybot.yaml
id: my-ai-bot
url: "http://localhost:8090"   # Where Synapse sends events to
as_token: "secrettoken1abc"    # AS uses this to authenticate to Synapse
hs_token: "secrettoken2xyz"    # Synapse uses this to authenticate to AS
sender_localpart: my-ai-bot    # Bot's Matrix ID: @my-ai-bot:example.com
rate_limited: false            # Bypass rate limits for virtual users

namespaces:
  users:
    - exclusive: true
      regex: "@ai_.*:example\\.com"   # All users starting with ai_
  aliases:
    - exclusive: false
      regex: "#ai_.*:example\\.com"   # Monitor (not exclusive) ai_ aliases
  rooms: []                           # No room ID namespace
```

**Token security:**
- `as_token`: The AS presents this in `Authorization: Bearer {as_token}` to Synapse
- `hs_token`: Synapse presents this in `Authorization: Bearer {hs_token}` to the AS
- **Never swap them.** Never hardcode them. Rotate if exposed.

**Exclusive namespace:** No other user or server can create `@ai_*` users — only this AS can. Synapse returns 403 if a client tries to register `@ai_alice:example.com` directly.

**Non-exclusive namespace:** The AS receives events from these entities but doesn't own them.

### 4.3 Transaction Endpoint — The Critical Pattern

Synapse delivers events by `PUT /_matrix/app/v1/transactions/{txnId}`:

```json
{
  "events": [
    {
      "event_id": "$abc123:example.com",
      "type": "m.room.message",
      "sender": "@alice:example.com",
      "room_id": "!roomid:example.com",
      "content": {"msgtype": "m.text", "body": "Hey bot!"},
      "origin_server_ts": 1713100000000,
      "unsigned": {"age": 100}
    }
  ]
}
```

**THE GOLDEN RULE: Return 200 immediately. Process asynchronously.**

If the AS takes too long, Synapse times out and **retries with the same `txnId`**. If the AS has already processed it, it must not process it again (idempotency).

```python
from fastapi import FastAPI, Header, HTTPException, BackgroundTasks
from pydantic import BaseModel
import asyncio
import asyncpg
from typing import Any

app = FastAPI()

# In-memory dedup set (use Redis or DB in production)
processed_transactions: set[str] = set()

class ASTransaction(BaseModel):
    events: list[dict[str, Any]]

@app.put("/_matrix/app/v1/transactions/{txn_id}")
async def handle_transaction(
    txn_id: str,
    transaction: ASTransaction,
    background_tasks: BackgroundTasks,
    authorization: str = Header(...),
) -> dict:
    # 1. Verify hs_token
    expected = "Bearer secrettoken2xyz"
    if authorization != expected:
        raise HTTPException(status_code=403, detail="M_FORBIDDEN")

    # 2. Deduplication check (return 200 immediately if seen before)
    if txn_id in processed_transactions:
        return {}  # idempotent — no-op

    processed_transactions.add(txn_id)  # mark as seen BEFORE processing

    # 3. Queue events for async processing — DO NOT AWAIT HERE
    background_tasks.add_task(process_events, txn_id, transaction.events)

    # 4. Return 200 IMMEDIATELY — Synapse must not timeout
    return {}
```

**Why mark as seen before processing, not after?**

If you mark after processing, a crash mid-processing leaves the txnId unseen. Synapse retries. You process again. **Double processing is worse than skipping once** — it sends duplicate messages to users. Mark as seen first; use idempotency keys in your actual processing to handle the edge case where marking succeeded but processing didn't.

```python
async def process_events(txn_id: str, events: list[dict]) -> None:
    for event in events:
        try:
            await route_event(event)
        except Exception as e:
            logger.error(f"Failed to process event {event.get('event_id')}: {e}")
            # Store in dead letter queue, not blocking the rest of the batch

async def route_event(event: dict) -> None:
    event_type = event.get("type")
    sender = event.get("sender", "")

    # Ignore events from our own bot users to prevent echo loops
    if sender.startswith("@ai_"):
        return

    match event_type:
        case "m.room.message":
            await handle_message(event)
        case "m.room.member":
            await handle_membership(event)
        case "m.reaction":
            await handle_reaction(event)
```

### 4.4 Query Endpoints

Synapse calls these when a user tries to interact with an entity that may need to be created:

```python
@app.get("/_matrix/app/v1/users/{user_id}")
async def query_user(
    user_id: str,
    authorization: str = Header(...),
) -> dict:
    """
    Called when a client tries to interact with @ai_alice:example.com
    and she doesn't exist yet.
    Return 200 if we can create her, 404 if not.
    """
    verify_hs_token(authorization)

    if not user_id.startswith("@ai_"):
        raise HTTPException(status_code=404, detail="M_NOT_FOUND")

    # Create the virtual user via Client-Server API
    await create_virtual_user(user_id)
    return {}  # 200 = user now exists

@app.get("/_matrix/app/v1/rooms/{room_alias}")
async def query_room(
    room_alias: str,
    authorization: str = Header(...),
) -> dict:
    """
    Called when a user tries to join #ai_support:example.com
    and the room doesn't exist yet.
    """
    verify_hs_token(authorization)

    if not room_alias.startswith("#ai_"):
        raise HTTPException(status_code=404, detail="M_NOT_FOUND")

    room_id = await create_managed_room(room_alias)
    return {}
```

### 4.5 Virtual Users — Acting as Other Identities

The AS can masquerade as any user in its namespace using the `user_id` query parameter:

```python
import httpx

SYNAPSE_URL = "http://synapse:8008"
AS_TOKEN = "secrettoken1abc"

async def send_as_user(
    room_id: str,
    user_id: str,  # e.g. @ai_alice:example.com
    message: str,
    txn_id: str,
) -> str:
    """Send a message as a virtual user."""
    async with httpx.AsyncClient() as client:
        response = await client.put(
            f"{SYNAPSE_URL}/_matrix/client/v3/rooms/{room_id}/send"
            f"/m.room.message/{txn_id}",
            headers={"Authorization": f"Bearer {AS_TOKEN}"},
            params={"user_id": user_id},  # Masquerade as this user
            json={"msgtype": "m.text", "body": message},
        )
        response.raise_for_status()
        return response.json()["event_id"]

async def create_virtual_user(user_id: str) -> None:
    """Register a user in the AS namespace without a password."""
    localpart = user_id.split(":")[0][1:]  # @ai_alice:server → ai_alice
    async with httpx.AsyncClient() as client:
        # Use the AS register endpoint
        await client.post(
            f"{SYNAPSE_URL}/_matrix/client/v3/register",
            headers={"Authorization": f"Bearer {AS_TOKEN}"},
            json={
                "kind": "user",
                "username": localpart,
                "inhibit_login": False,
            },
        )

async def set_display_name(user_id: str, display_name: str) -> None:
    async with httpx.AsyncClient() as client:
        await client.put(
            f"{SYNAPSE_URL}/_matrix/client/v3/profile/{user_id}/displayname",
            headers={"Authorization": f"Bearer {AS_TOKEN}"},
            params={"user_id": user_id},
            json={"displayname": display_name},
        )
```

### 4.6 Threading and Reply Pattern

Matrix uses `m.relates_to` for threading, quoting, and reactions:

```python
async def reply_to_event(
    room_id: str,
    sender_user_id: str,
    reply_text: str,
    in_reply_to_event_id: str,
    txn_id: str,
) -> str:
    """Send a threaded reply."""
    content = {
        "msgtype": "m.text",
        "body": f"> In reply to message\n\n{reply_text}",
        "m.relates_to": {
            "m.in_reply_to": {
                "event_id": in_reply_to_event_id,
            }
        },
    }

    return await send_as_user(room_id, sender_user_id, None, txn_id, content=content)

async def send_notice(
    room_id: str,
    bot_user_id: str,
    notice_text: str,
    txn_id: str,
) -> str:
    """Send a bot notice (displayed differently from user messages in clients)."""
    content = {
        "msgtype": "m.notice",
        "body": notice_text,
    }
    return await send_as_user(room_id, bot_user_id, None, txn_id, content=content)
```

---

## PART 5: AI AGENT INTEGRATION WITH MATRIX

### 5.1 Matrix as a Multi-Agent Orchestration Layer

Matrix rooms are an excellent container for multi-agent AI systems because:

1. **Persistent context:** The event DAG is the conversation history — no separate storage needed for "what has been said."
2. **Identity per agent:** Each AI agent is a virtual user with its own `@agent_name:server` identity.
3. **Audit trail:** Every action is a signed, timestamped event — perfect for AI audit logs.
4. **Multi-party:** Multiple humans and multiple AI agents can participate in one room.
5. **Replay:** You can reconstruct any conversation state by replaying the DAG.

```
┌──────────────────────────────────────────────────┐
│           !room_id:server (Matrix Room)            │
│                                                    │
│  @alice:server: "Plan a marketing campaign"        │
│  @orchestrator_bot: "I'll coordinate this"         │
│  @research_agent: "Here's the market analysis..."  │
│  @writer_agent: "Draft email copy follows..."      │
│  @review_agent: "Suggested revisions:..."          │
│  @alice:server: 👍 (m.reaction)                    │
└──────────────────────────────────────────────────┘
```

Each agent is implemented as a virtual user in an Application Service.

### 5.2 Per-Room Agent State

Use the `room_id` as the key for agent state/memory. LangGraph checkpointers keyed by room:

```python
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
from langgraph.graph import StateGraph
import asyncpg

# One LangGraph thread per Matrix room
async def get_or_create_agent(room_id: str) -> CompiledGraph:
    pool = await asyncpg.create_pool(DATABASE_URL)
    checkpointer = AsyncPostgresSaver(pool)

    graph = build_agent_graph()  # Your StateGraph definition
    compiled = graph.compile(checkpointer=checkpointer)

    return compiled

async def invoke_agent_for_room(
    room_id: str,
    user_message: str,
    sender_mxid: str,
) -> str:
    agent = await get_or_create_agent(room_id)

    # thread_id = room_id → all messages in the room share one checkpoint
    config = {"configurable": {"thread_id": room_id}}

    result = await agent.ainvoke(
        {"messages": [{"role": "user", "content": user_message, "name": sender_mxid}]},
        config=config,
    )

    return result["messages"][-1].content
```

### 5.3 Per-User Memory Across Rooms

Room state is per-conversation. For persistent user memory across all rooms:

```python
from mem0 import AsyncMemory
import qdrant_client

# Initialize Mem0 with Qdrant backend
memory = AsyncMemory.from_config({
    "vector_store": {
        "provider": "qdrant",
        "config": {
            "host": "localhost",
            "port": 6333,
            "collection_name": "matrix_user_memories",
        }
    },
    "llm": {"provider": "anthropic", "config": {"model": "claude-haiku-4-5"}},
})

async def get_user_context(sender_mxid: str) -> str:
    """Retrieve relevant memories for this user."""
    memories = await memory.search(
        query="user preferences and background",
        user_id=sender_mxid,  # MXID is the stable user identifier
        limit=10,
    )
    return "\n".join(m["memory"] for m in memories)

async def update_user_memory(sender_mxid: str, conversation: list[dict]) -> None:
    """Extract and store new memories from conversation."""
    await memory.add(conversation, user_id=sender_mxid)
```

### 5.4 Full AS + AI Integration Flow

```python
import asyncio
from collections import deque
from typing import Any
import httpx
from fastapi import FastAPI, BackgroundTasks, Header, HTTPException

app = FastAPI()
event_queue: asyncio.Queue = asyncio.Queue()
seen_transactions: set[str] = set()

# ── Transaction endpoint (synchronous, fast) ────────────────────────────────

@app.put("/_matrix/app/v1/transactions/{txn_id}")
async def receive_transaction(
    txn_id: str,
    body: dict,
    background_tasks: BackgroundTasks,
    authorization: str = Header(...),
) -> dict:
    if authorization != f"Bearer {HS_TOKEN}":
        raise HTTPException(status_code=403, detail="M_FORBIDDEN")

    if txn_id in seen_transactions:
        return {}  # deduplicated

    seen_transactions.add(txn_id)  # mark before enqueue, not after

    for event in body.get("events", []):
        await event_queue.put(event)

    return {}  # 200 to Synapse within milliseconds

# ── Background worker (processes events asynchronously) ─────────────────────

@app.on_event("startup")
async def start_worker() -> None:
    asyncio.create_task(event_worker())

async def event_worker() -> None:
    while True:
        event = await event_queue.get()
        try:
            await process_event(event)
        except Exception as e:
            logger.error(f"Event processing failed: {e}", exc_info=True)
            # Send to dead-letter queue for manual inspection
            await dead_letter_queue.put(event)
        finally:
            event_queue.task_done()

async def process_event(event: dict) -> None:
    sender = event.get("sender", "")
    room_id = event.get("room_id")
    event_type = event.get("type")

    # Filter: ignore our own messages to prevent echo loops
    if sender.startswith("@ai_"):
        return

    # Filter: only process direct messages to the bot
    if event_type != "m.room.message":
        return

    content = event.get("content", {})
    if content.get("msgtype") not in ("m.text", "m.emote"):
        return

    body = content.get("body", "")

    # Get room context + user memory
    user_memory = await get_user_context(sender)
    system_prompt = f"User memories:\n{user_memory}" if user_memory else None

    # Send typing indicator while processing
    await send_typing(room_id, BOT_USER_ID, typing=True)

    try:
        # Invoke AI agent
        response_text = await invoke_agent_for_room(room_id, body, sender)

        # Reply with threading
        txn_id = generate_txn_id(event["event_id"])
        await reply_to_event(
            room_id=room_id,
            sender_user_id=BOT_USER_ID,
            reply_text=response_text,
            in_reply_to_event_id=event["event_id"],
            txn_id=txn_id,
        )

        # Update user memory
        asyncio.create_task(
            update_user_memory(sender, [
                {"role": "user", "content": body},
                {"role": "assistant", "content": response_text},
            ])
        )
    finally:
        await send_typing(room_id, BOT_USER_ID, typing=False)

def generate_txn_id(event_id: str) -> str:
    """Deterministic txn_id based on event_id — safe to retry."""
    import hashlib
    return hashlib.sha256(event_id.encode()).hexdigest()[:16]

async def send_typing(room_id: str, user_id: str, typing: bool) -> None:
    async with httpx.AsyncClient() as client:
        await client.put(
            f"{SYNAPSE_URL}/_matrix/client/v3/rooms/{room_id}/typing/{user_id}",
            headers={"Authorization": f"Bearer {AS_TOKEN}"},
            params={"user_id": user_id},
            json={"typing": typing, "timeout": 30000},
        )
```

### 5.5 Handling Reactions for Feedback

```python
async def handle_reaction(event: dict) -> None:
    """User sent a 👍/👎 reaction to a bot message."""
    relates_to = event.get("content", {}).get("m.relates_to", {})
    if relates_to.get("rel_type") != "m.annotation":
        return

    reaction_key = relates_to.get("key", "")  # "👍" or "👎"
    target_event_id = relates_to.get("event_id")
    sender = event.get("sender")

    if reaction_key == "👍":
        await record_positive_feedback(sender, target_event_id)
        # Optionally store in Langfuse as a score
        langfuse.score(
            trace_id=target_event_id,
            name="user_reaction",
            value=1.0,
            data_type="BOOLEAN",
            comment=f"thumbs_up from {sender}",
        )
    elif reaction_key == "👎":
        await record_negative_feedback(sender, target_event_id)
        langfuse.score(
            trace_id=target_event_id,
            name="user_reaction",
            value=0.0,
            data_type="BOOLEAN",
            comment=f"thumbs_down from {sender}",
        )
```

---

## PART 6: EVENT-DRIVEN PATTERNS

### 6.1 PostgreSQL LISTEN/NOTIFY

The simplest pub/sub — no additional infrastructure. Uses PostgreSQL's built-in notification channel.

```python
import asyncpg
import asyncio

async def publisher(pool: asyncpg.Pool, channel: str, payload: str) -> None:
    """Publish a notification from within or outside a transaction."""
    async with pool.acquire() as conn:
        await conn.execute(f"NOTIFY {channel}, $1", payload)

async def subscriber(dsn: str, channel: str) -> None:
    """Listen for notifications. Uses a DEDICATED connection (not pool)."""
    conn = await asyncpg.connect(dsn)

    async def on_notification(conn, pid, channel, payload):
        print(f"Received: {payload} on {channel}")
        await process_notification(payload)

    await conn.add_listener(channel, on_notification)

    # Keep listening
    try:
        await asyncio.Future()  # run forever
    finally:
        await conn.remove_listener(channel, on_notification)
        await conn.close()

# Trigger-based: notify from within a transaction automatically
# CREATE OR REPLACE FUNCTION notify_on_insert() RETURNS trigger AS $$
# BEGIN
#   PERFORM pg_notify('new_events', row_to_json(NEW)::text);
#   RETURN NEW;
# END;
# $$ LANGUAGE plpgsql;
#
# CREATE TRIGGER event_inserted
#   AFTER INSERT ON events
#   FOR EACH ROW EXECUTE FUNCTION notify_on_insert();
```

**Limitations:**
- Payload limited to 8000 bytes
- Notifications are lost if no listener is connected (not durable)
- Not suitable for high-volume (thousands per second) use cases

Use for: cache invalidation, simple fan-out, dev environments, small teams.

### 6.2 Redis Streams vs Redis Pub/Sub

| Aspect | Pub/Sub | Streams |
|---|---|---|
| Durability | Ephemeral — lost if no subscriber | Durable — stored in Redis |
| Consumer groups | No — all subscribers get all messages | Yes — distribute work across consumers |
| Acknowledgment | None | `XACK` — consumer marks message processed |
| Replay | Impossible | Yes — `XREAD` from any offset |
| Backpressure | None | `MAXLEN` cap |
| Use case | Cache invalidation, live presence | Job queue, event log, reliable delivery |

```python
import redis.asyncio as redis

r = redis.Redis(host="localhost", port=6379)

# ── Redis Streams: Producer ──────────────────────────────────────────────────
async def publish_event(stream: str, event: dict) -> str:
    """Publish to a stream. Returns the stream entry ID."""
    entry_id = await r.xadd(
        stream,
        event,
        maxlen=10000,  # keep last 10k entries
        approximate=True,  # ~10k (faster than exact)
    )
    return entry_id.decode()

# ── Redis Streams: Consumer with Consumer Group ──────────────────────────────
async def create_consumer_group(stream: str, group: str) -> None:
    try:
        await r.xgroup_create(stream, group, id="0", mkstream=True)
    except redis.ResponseError as e:
        if "BUSYGROUP" not in str(e):
            raise  # group already exists is fine

async def consume_events(stream: str, group: str, consumer_id: str) -> None:
    await create_consumer_group(stream, group)

    while True:
        # Read pending (unacknowledged) messages first
        pending = await r.xautoclaim(
            stream, group, consumer_id,
            min_idle_time=60000,  # reclaim messages idle >60s (crashed worker)
            start_id="0-0",
            count=10,
        )

        # Read new messages
        messages = await r.xreadgroup(
            groupname=group,
            consumername=consumer_id,
            streams={stream: ">"},  # ">" = only undelivered messages
            count=10,
            block=5000,  # block up to 5 seconds waiting for messages
        )

        if messages:
            for stream_name, entries in messages:
                for entry_id, fields in entries:
                    try:
                        await process_stream_event(fields)
                        await r.xack(stream, group, entry_id)  # ACK after success
                    except Exception as e:
                        logger.error(f"Stream event failed: {e}")
                        # Do NOT ack — message stays pending for retry
```

### 6.3 RabbitMQ

**Exchange types:**
- `direct`: Routes to queues with exact binding key match
- `topic`: Routes by pattern (`order.#` matches `order.created`, `order.created.paid`)
- `fanout`: Broadcasts to all bound queues (ignore routing key)
- `headers`: Routes by message header attributes (rarely used)

```python
import aio_pika
import json

async def setup_rabbitmq(connection_url: str) -> aio_pika.Channel:
    connection = await aio_pika.connect_robust(connection_url)
    channel = await connection.channel()

    # Declare exchange
    exchange = await channel.declare_exchange(
        "events",
        aio_pika.ExchangeType.TOPIC,
        durable=True,
    )

    # Declare queue with dead letter exchange
    queue = await channel.declare_queue(
        "order_processing",
        durable=True,
        arguments={
            "x-dead-letter-exchange": "events_dlx",  # failed messages go here
            "x-dead-letter-routing-key": "failed.order",
            "x-message-ttl": 86400000,  # 24 hours max
        },
    )

    # Bind queue to exchange with routing pattern
    await queue.bind(exchange, routing_key="order.*")

    return channel, exchange, queue

async def publish(exchange, routing_key: str, body: dict) -> None:
    await exchange.publish(
        aio_pika.Message(
            body=json.dumps(body).encode(),
            content_type="application/json",
            delivery_mode=aio_pika.DeliveryMode.PERSISTENT,  # survive broker restart
        ),
        routing_key=routing_key,
    )

async def consume(queue: aio_pika.Queue) -> None:
    async with queue.iterator() as queue_iter:
        async for message in queue_iter:
            async with message.process(requeue=False):  # auto-nack on exception
                try:
                    body = json.loads(message.body)
                    await handle_order_event(body)
                    # message.process() auto-acks on exit
                except Exception as e:
                    logger.error(f"Failed: {e}")
                    # requeue=False → message goes to DLQ instead of infinite retry
```

### 6.4 Kafka

**Core concepts:**
- **Topic:** Named stream of records (like a database table)
- **Partition:** Topic is split into N partitions; each partition is an ordered log
- **Offset:** Integer position within a partition (monotonically increasing)
- **Consumer group:** Group of consumers that collectively consume a topic; each partition is assigned to exactly one consumer in the group
- **Producer ID (PID):** Unique identifier for idempotent producers

```
Topic: "orders"
┌─ Partition 0 ─────────────────────────────┐
│ offset: 0,  1,  2,  3,  4,  5  ...        │
│ value: "o1","o2","o4","o6","o8","o10"...   │
└───────────────────────────────────────────┘
┌─ Partition 1 ─────────────────────────────┐
│ offset: 0,  1,  2,  3,  4,  5  ...        │
│ value: "o3","o5","o7","o9","o11",...       │
└───────────────────────────────────────────┘
```

**Partition key determines which partition receives a message.** Messages with the same key always go to the same partition → order preserved per key.

```python
from aiokafka import AIOKafkaProducer, AIOKafkaConsumer
import json

# Producer with exactly-once semantics
async def create_producer() -> AIOKafkaProducer:
    return AIOKafkaProducer(
        bootstrap_servers=["kafka:9092"],
        enable_idempotence=True,       # idempotent: PID + sequence number per partition
        acks="all",                    # wait for all in-sync replicas
        compression_type="snappy",
        value_serializer=lambda v: json.dumps(v).encode(),
        key_serializer=lambda k: k.encode() if k else None,
    )

async def publish_order(producer: AIOKafkaProducer, order: dict) -> None:
    await producer.send_and_wait(
        topic="orders",
        key=order["customer_id"],  # same customer → same partition → ordered per customer
        value=order,
    )

# Consumer with manual offset commit
async def create_consumer(group_id: str) -> AIOKafkaConsumer:
    return AIOKafkaConsumer(
        "orders",
        bootstrap_servers=["kafka:9092"],
        group_id=group_id,
        auto_offset_reset="earliest",
        enable_auto_commit=False,          # MANUAL commit for exactly-once
        isolation_level="read_committed",  # only read committed transactions
        value_deserializer=lambda v: json.loads(v.decode()),
    )

async def process_messages(consumer: AIOKafkaConsumer) -> None:
    async for message in consumer:
        try:
            await process_order(message.value)
            # Commit AFTER successful processing
            await consumer.commit({
                TopicPartition(message.topic, message.partition): message.offset + 1
            })
        except Exception as e:
            logger.error(f"Failed to process message at offset {message.offset}: {e}")
            # Do NOT commit — message will be redelivered to this consumer group
            # Implement DLQ logic: if N retries, skip and publish to orders_dlq
```

**Consumer group scaling:** Each partition is handled by exactly one consumer in the group. If you have 3 partitions and 5 consumers, 2 consumers are idle. If you have 3 partitions and 2 consumers, one consumer handles 2 partitions. Scale consumer count to match partition count for maximum parallelism.

**NEVER share a consumer group ID across different logical consumers.** Two different services reading from the same topic need different group IDs, or they will steal each other's messages.

### 6.5 The Outbox Pattern

The core problem: you want to publish an event when you write to the database, but:
- If you write to DB and then publish to Kafka, and the app crashes between them → event lost
- If you publish to Kafka first and then write to DB, and the DB fails → event without data

**Outbox pattern:** Make the event publication part of the same database transaction as your business data write. A separate publisher process reads the outbox and publishes.

```sql
-- Schema
CREATE TABLE outbox_events (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type TEXT NOT NULL,      -- e.g. "Order"
    aggregate_id   TEXT NOT NULL,      -- e.g. "order_42"
    event_type     TEXT NOT NULL,      -- e.g. "OrderCreated"
    payload        JSONB NOT NULL,
    created_at     TIMESTAMPTZ DEFAULT now(),
    published_at   TIMESTAMPTZ,        -- NULL = not yet published
    attempts       INT DEFAULT 0
);

CREATE INDEX idx_outbox_unpublished ON outbox_events (created_at)
    WHERE published_at IS NULL;
```

```python
async def create_order_with_outbox(
    conn: asyncpg.Connection,
    order_data: dict,
) -> str:
    """Write order AND outbox event in one atomic transaction."""
    async with conn.transaction():
        # Business data write
        order_id = await conn.fetchval(
            "INSERT INTO orders (customer_id, items, total) VALUES ($1, $2, $3) RETURNING id",
            order_data["customer_id"],
            json.dumps(order_data["items"]),
            order_data["total"],
        )

        # Outbox event write — same transaction, atomic
        await conn.execute(
            """
            INSERT INTO outbox_events (aggregate_type, aggregate_id, event_type, payload)
            VALUES ('Order', $1, 'OrderCreated', $2)
            """,
            str(order_id),
            json.dumps({"order_id": str(order_id), **order_data}),
        )

    return str(order_id)

# Separate publisher process / background task
async def outbox_publisher(pool: asyncpg.Pool, kafka_producer) -> None:
    while True:
        async with pool.acquire() as conn:
            rows = await conn.fetch(
                """
                SELECT id, aggregate_type, aggregate_id, event_type, payload
                FROM outbox_events
                WHERE published_at IS NULL
                ORDER BY created_at
                LIMIT 100
                FOR UPDATE SKIP LOCKED  -- safe for multiple publisher instances
                """
            )

            for row in rows:
                try:
                    await kafka_producer.send_and_wait(
                        topic=f"{row['aggregate_type'].lower()}_events",
                        key=row["aggregate_id"],
                        value={"type": row["event_type"], "data": json.loads(row["payload"])},
                    )

                    await conn.execute(
                        "UPDATE outbox_events SET published_at = now() WHERE id = $1",
                        row["id"],
                    )
                except Exception as e:
                    await conn.execute(
                        "UPDATE outbox_events SET attempts = attempts + 1 WHERE id = $1",
                        row["id"],
                    )
                    logger.error(f"Outbox publish failed for {row['id']}: {e}")

        await asyncio.sleep(0.1)  # poll every 100ms
```

**Alternative: PostgreSQL logical replication (WAL tailing).** Instead of polling the outbox table, use pg_logical to stream WAL changes directly. More complex to set up but eliminates the polling loop.

### 6.6 Event Sourcing and CQRS

**Event Sourcing:** The source of truth is the append-only event log. The current state of any entity is derived by replaying its events.

```python
# Events are immutable facts
@dataclass
class OrderCreated:
    order_id: str
    customer_id: str
    items: list[dict]
    timestamp: float

@dataclass
class OrderPaid:
    order_id: str
    payment_id: str
    amount: Decimal
    timestamp: float

@dataclass
class OrderShipped:
    order_id: str
    tracking_number: str
    timestamp: float

# State is derived by replaying events
def replay_order(events: list) -> dict:
    state = {}
    for event in events:
        match event:
            case OrderCreated() as e:
                state = {
                    "id": e.order_id,
                    "customer_id": e.customer_id,
                    "items": e.items,
                    "status": "created",
                }
            case OrderPaid() as e:
                state["status"] = "paid"
                state["payment_id"] = e.payment_id
            case OrderShipped() as e:
                state["status"] = "shipped"
                state["tracking"] = e.tracking_number
    return state
```

**CQRS (Command Query Responsibility Segregation):**

Write path: User sends Command → validate → append Event → project to Read Model
Read path: User queries Read Model (pre-computed, denormalized view)

```python
# Write side: commands produce events
async def handle_create_order(command: CreateOrderCommand) -> None:
    event = OrderCreated(
        order_id=str(uuid4()),
        customer_id=command.customer_id,
        items=command.items,
        timestamp=time.time(),
    )
    await event_store.append(event)
    await projector.project(event)  # update read model asynchronously

# Read side: pre-computed view
async def get_order_summary(order_id: str) -> dict:
    # Reads from a denormalized table, not the event log
    return await db.fetchrow(
        "SELECT * FROM order_summaries WHERE id = $1", order_id
    )
```

**Exactly-once delivery with idempotency key:**

```python
async def process_event_exactly_once(
    conn: asyncpg.Connection,
    event_id: str,
    handler: Callable,
) -> None:
    """Process an event exactly once using INSERT ON CONFLICT DO NOTHING."""
    inserted = await conn.fetchval(
        """
        INSERT INTO processed_events (event_id, processed_at)
        VALUES ($1, now())
        ON CONFLICT (event_id) DO NOTHING
        RETURNING event_id
        """,
        event_id,
    )

    if inserted is None:
        logger.info(f"Event {event_id} already processed, skipping")
        return  # idempotent skip

    await handler()  # process only if insertion succeeded
```

---

## PART 7: PRACTICAL FASTAPI + MATRIX INTEGRATION — FULL EXAMPLE

### 7.1 Complete Application Service

```python
"""
matrix_as/main.py
Full production-ready Matrix Application Service using FastAPI.
"""

import asyncio
import hashlib
import json
import logging
import os
from contextlib import asynccontextmanager
from typing import Any

import asyncpg
import httpx
from fastapi import BackgroundTasks, FastAPI, Header, HTTPException
from pydantic import BaseModel

logger = logging.getLogger(__name__)

# Configuration (from environment)
SYNAPSE_URL = os.environ["SYNAPSE_URL"]
AS_TOKEN = os.environ["AS_TOKEN"]      # AS uses to authenticate to Synapse
HS_TOKEN = os.environ["HS_TOKEN"]      # Synapse uses to authenticate to AS
BOT_MXID = os.environ["BOT_MXID"]     # e.g. @my-ai-bot:example.com
DATABASE_URL = os.environ["DATABASE_URL"]

# State
db_pool: asyncpg.Pool | None = None
event_queue: asyncio.Queue = asyncio.Queue(maxsize=10000)

# ── Lifespan ─────────────────────────────────────────────────────────────────

@asynccontextmanager
async def lifespan(app: FastAPI):
    global db_pool

    # Initialize DB pool
    db_pool = await asyncpg.create_pool(DATABASE_URL, min_size=2, max_size=10)

    # Create dedup table
    async with db_pool.acquire() as conn:
        await conn.execute("""
            CREATE TABLE IF NOT EXISTS seen_transactions (
                txn_id TEXT PRIMARY KEY,
                received_at TIMESTAMPTZ DEFAULT now()
            )
        """)

    # Start background event processor
    worker_task = asyncio.create_task(event_worker())

    yield

    # Shutdown
    worker_task.cancel()
    await db_pool.close()

app = FastAPI(lifespan=lifespan)

# ── Models ────────────────────────────────────────────────────────────────────

class Transaction(BaseModel):
    events: list[dict[str, Any]] = []

# ── Helpers ───────────────────────────────────────────────────────────────────

def verify_hs_token(authorization: str) -> None:
    if authorization != f"Bearer {HS_TOKEN}":
        raise HTTPException(status_code=403, detail="M_FORBIDDEN")

def make_txn_id(event_id: str) -> str:
    """Deterministic, reproducible transaction ID for replies."""
    return hashlib.sha256(f"reply:{event_id}".encode()).hexdigest()[:16]

async def is_duplicate_txn(txn_id: str) -> bool:
    """Check and register transaction atomically."""
    async with db_pool.acquire() as conn:
        result = await conn.fetchval(
            """
            INSERT INTO seen_transactions (txn_id)
            VALUES ($1)
            ON CONFLICT (txn_id) DO NOTHING
            RETURNING txn_id
            """,
            txn_id,
        )
        return result is None  # None = conflict = duplicate

# ── Transaction endpoint ───────────────────────────────────────────────────────

@app.put("/_matrix/app/v1/transactions/{txn_id}")
async def handle_transaction(
    txn_id: str,
    transaction: Transaction,
    background_tasks: BackgroundTasks,
    authorization: str = Header(...),
) -> dict:
    verify_hs_token(authorization)

    if await is_duplicate_txn(txn_id):
        return {}

    for event in transaction.events:
        try:
            await event_queue.put_nowait(event)
        except asyncio.QueueFull:
            logger.error("Event queue full — dropping event! Increase worker capacity.")

    return {}

# ── Query endpoints ────────────────────────────────────────────────────────────

@app.get("/_matrix/app/v1/users/{user_id}")
async def query_user(
    user_id: str,
    authorization: str = Header(...),
) -> dict:
    verify_hs_token(authorization)
    logger.info(f"Synapse querying user: {user_id}")

    if not user_id.startswith("@ai_"):
        raise HTTPException(status_code=404, detail="M_NOT_FOUND")

    await create_virtual_user(user_id)
    return {}

# ── Background worker ─────────────────────────────────────────────────────────

async def event_worker() -> None:
    while True:
        event = await event_queue.get()
        try:
            await dispatch_event(event)
        except Exception:
            logger.exception(f"Unhandled error in event worker for {event.get('event_id')}")
        finally:
            event_queue.task_done()

async def dispatch_event(event: dict) -> None:
    sender = event.get("sender", "")
    event_type = event.get("type", "")

    # Prevent echo loops
    if "@ai_" in sender or sender == BOT_MXID:
        return

    match event_type:
        case "m.room.message":
            await handle_message(event)
        case "m.reaction":
            await handle_reaction(event)
        case "m.room.member":
            await handle_membership(event)
        case _:
            pass  # Ignore unknown event types

# ── Message handler ────────────────────────────────────────────────────────────

async def handle_message(event: dict) -> None:
    content = event.get("content", {})
    if content.get("msgtype") not in ("m.text",):
        return

    room_id = event["room_id"]
    body = content.get("body", "").strip()

    if not body:
        return

    # Send typing indicator
    await synapse_put(
        f"/_matrix/client/v3/rooms/{room_id}/typing/{BOT_MXID}",
        {"typing": True, "timeout": 30000},
        user_id=BOT_MXID,
    )

    try:
        # AI processing
        reply = await get_ai_reply(room_id, body, event["sender"])

        # Reply with threading
        txn_id = make_txn_id(event["event_id"])
        await synapse_put(
            f"/_matrix/client/v3/rooms/{room_id}/send/m.room.message/{txn_id}",
            {
                "msgtype": "m.text",
                "body": reply,
                "m.relates_to": {
                    "m.in_reply_to": {"event_id": event["event_id"]}
                },
            },
            user_id=BOT_MXID,
        )
    finally:
        await synapse_put(
            f"/_matrix/client/v3/rooms/{room_id}/typing/{BOT_MXID}",
            {"typing": False},
            user_id=BOT_MXID,
        )

# ── Synapse API helpers ────────────────────────────────────────────────────────

async def synapse_put(path: str, body: dict, user_id: str | None = None) -> dict:
    params = {"user_id": user_id} if user_id else {}
    async with httpx.AsyncClient() as client:
        r = await client.put(
            f"{SYNAPSE_URL}{path}",
            headers={"Authorization": f"Bearer {AS_TOKEN}"},
            params=params,
            json=body,
            timeout=30.0,
        )
        r.raise_for_status()
        return r.json()

async def create_virtual_user(user_id: str) -> None:
    localpart = user_id.split(":")[0][1:]
    async with httpx.AsyncClient() as client:
        await client.post(
            f"{SYNAPSE_URL}/_matrix/client/v3/register",
            headers={"Authorization": f"Bearer {AS_TOKEN}"},
            json={"kind": "user", "username": localpart},
        )

# Placeholder: replace with real LLM integration
async def get_ai_reply(room_id: str, message: str, sender: str) -> str:
    return f"Echo from AI: {message}"
```

### 7.2 Error Recovery

**Transaction retry scenario:**

```
Time 0:  Synapse sends txnId=T1 to AS
Time 1:  AS starts processing
Time 5:  AS crashes (power loss)
Time 15: Synapse retries txnId=T1
Time 16: AS starts fresh, receives T1
         → DB check: T1 not in seen_transactions (was written in-memory, not DB)
         → Processes T1 again
         → Deterministic txn_id for reply: make_txn_id(event_id) = same value
         → Synapse deduplicates the reply (same client-side txn_id)
         ✓ User sees exactly one reply
```

Key insight: Use a **database** (not in-memory set) for dedup, so it survives restarts. Use **deterministic** reply txn_ids so Synapse-side dedup catches duplicate replies.

---

## Key Takeaways

1. **CAP is about partition behavior; PACELC is about everyday tradeoffs.** Most of your real decisions are PACELC decisions.

2. **Raft is Paxos made understandable.** Terms are logical clocks. Leaders are elected by majority. Entries commit when a majority acknowledges.

3. **2PC blocks. Sagas compensate.** For anything spanning service boundaries, use Saga with idempotency keys.

4. **Matrix events are a causal DAG, not a chain.** `prev_events` creates edges; `auth_events` proves authorization; `depth` is the ordering integer.

5. **State resolution v2 is deterministic topological sort.** Power level changes are resolved first, then the rest. All servers run the same algorithm and converge.

6. **Federation is signed HTTP.** Servers sign PDUs with Ed25519, verify each other via `/_matrix/key/v2/server`. Never trust an unsigned event.

7. **The AS transaction endpoint must return 200 immediately.** Queue and process asynchronously. Deduplicate by txn_id in a database.

8. **The outbox pattern is the correct solution for dual writes.** Write event to DB in the same transaction as business data. Publish separately.

9. **CRDTs are merge functions that are commutative + associative + idempotent.** They enable conflict-free eventual consistency without coordination.

10. **Matrix rooms are excellent AI orchestration containers.** `room_id` = thread identity. `sender` MXID = user identity for cross-room memory.