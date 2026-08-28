# HLD Interview Notes — Real-Time View Counters

## Problem

Design a system to track and display live concurrent viewers during a major sports event such as IPL, handling tens of millions of concurrent viewers.

---

# 1. Requirements Clarification

### Scale

Clarify:
- Is 10M the peak concurrent viewers or total users over the event?
- What is the heartbeat frequency?
- How quickly do users join/leave?
- Are there predictable spikes around match start, wickets, innings breaks, etc.?

Example assumptions:
- 10M peak concurrent viewing sessions
- Heartbeat every 10 seconds
- 30-second inactivity timeout
- Viewer count may be up to ~5 seconds stale

Important distinction:

**10M concurrent users != 10M requests/sec.**

With a heartbeat every 10 seconds:

`10M / 10 = ~1M heartbeat events/sec`

---

# 2. Define "Viewer"

For this system, define:

> One active viewing session/device = one viewer.

If the same human watches on laptop + phone, count them as two concurrent viewing sessions.

Why?
- Concurrent streaming sessions are the natural metric.
- Unique-human counting requires identity-based deduplication and is a different problem.

---

# 3. Consistency Requirement

Prefer eventual consistency.

Example:

> The displayed count can be a few seconds stale, but should converge quickly.

Strict exactness is much harder because we need to define and enforce precise join/leave semantics and handle network failures, delayed events, retries, etc.

For a streaming application, availability and scalability are generally more important than transactional exactness.

---

# 4. Heartbeat Model

Each viewing session periodically sends:

```text
{
  match_id,
  session_id,
  timestamp,
  sequence_number
}
```

Example:
- heartbeat every 10 sec
- session is active if heartbeat received within last 30 sec

A user may disappear without sending a `USER_LEFT` event, so inactivity must be detected through timeout/expiration.

---

# 5. High-Level Architecture

```text
Clients
   |
   | heartbeat every 10 sec
   v
DDoS/WAF/CDN
   |
   v
Load Balancer
   |
   v
Stateless Ingestion Servers
   |
   v
Event Streaming Platform
   |
   | partition by session_id
   v
Stream Consumers
   |
   | session_id -> last_seen
   | expiry structure
   v
Local Active Counts
   |
   v
Aggregation Layer
   |
   v
Match Viewer Count
   |
   v
Pub/Sub / Fanout
   |
   v
SSE Gateway
   |
   v
Clients
```

Video delivery and viewer-count infrastructure can be separate. Reuse an existing long-lived connection only if the streaming platform already supports metadata fanout appropriately.

---

# 6. Why We Need Session State

A heartbeat is not a viewer.

Example:

```text
A heartbeat at t=0
A heartbeat at t=10
A heartbeat at t=20
```

This is:

- 3 heartbeat events
- 1 viewer

Therefore:

> Do not simply count heartbeats.

Maintain:

```text
Map<session_id, last_seen>
```

Example:

```text
A -> 10:10:20
B -> 10:10:25
C -> 10:10:27
```

Active viewer count is the number of sessions whose last heartbeat is within the activity window.

---

# 7. Expiration

A naive implementation would scan all 10M sessions periodically:

```text
for every session:
    if now - last_seen > 30 sec:
        remove
```

This is wasteful.

Instead, use an expiry structure.

## Min-Heap Approach

Maintain:

```text
Map<session_id, last_seen>

MinHeap<(expiry_time, session_id)>
```

When heartbeat arrives:

```text
t = now

map[session_id] = t
heap.push(t + 30 sec, session_id)
```

Every ~5 seconds:

```text
while heap.top.expiry_time <= now:
    (expiry_time, session_id) = heap.pop()

    if map[session_id] == expiry_time - 30 sec:
        delete map[session_id]
```

Why stale entries occur:

```text
A heartbeat at 10:00:00
=> expiry 10:00:30

A heartbeat at 10:00:10
=> expiry 10:00:40
```

Heap contains:

```text
10:00:30 -> A
10:00:40 -> A
```

At 10:00:30, the first entry is stale because:

```text
map[A] = 10:00:10
```

So ignore it.

At 10:00:40, A actually expires.

---

# 8. Timing Wheel Alternative

For huge numbers of timers, a timing wheel can be useful.

Conceptually:

```text
0  1  2  3  4 ... 59
|  |  |  |  |
         ^
       current time
```

Sessions are placed in buckets corresponding to their expiry time.

When a bucket is reached, inspect only sessions scheduled to expire.

This can give approximately O(1) timer scheduling behavior.

For an interview, min-heap is easier to explain unless the interviewer asks about timer scalability.

---

# 9. Why Time Buckets Alone Are Not Enough

A tempting approach is:

> Count heartbeats in the last N buckets and sum them.

This is wrong because the same session can appear in multiple buckets.

Example:

```text
bucket 0  -> A
bucket 10 -> A
bucket 20 -> A
```

Summing gives 3, but there is only 1 active viewer.

Buckets can still be useful for expiration/windowing, but we need per-session state or another distinct-session mechanism.

---

# 10. Partitioning

Do NOT necessarily create one Kafka topic per match.

A common design:

```text
viewer-events topic

Partition 0
Partition 1
Partition 2
...
Partition N
```

Events contain:

```text
match_id
session_id
timestamp
```

Partition by:

```text
hash(session_id)
```

This ensures all heartbeats for one session go to the same partition.

Example:

```text
session A
heartbeat 1 -> P7
heartbeat 2 -> P7
heartbeat 3 -> P7
```

This allows one stream processor to own the complete state for that session.

---

# 11. Why Not Partition Only by Match ID?

If:

```text
hash(match_id)
```

then all 10M viewers of an IPL final may land on one partition.

That creates a hot partition.

Instead:

```text
hash(session_id)
```

distributes a single large match across many partitions.

Each event still contains `match_id`, so workers know which match the session belongs to.

---

# 12. Distributed Local State

Suppose:

```text
C1 -> 2.5M sessions
C2 -> 2.5M sessions
C3 -> 2.5M sessions
C4 -> 2.5M sessions
```

Each consumer maintains:

```text
Map<session_id, last_seen>
```

and an expiry structure.

Each worker can calculate:

```text
local_active_count = map.size()
```

Example:

```text
C1 = 2.4M
C2 = 2.3M
C3 = 2.5M
C4 = 2.4M
```

---

# 13. Aggregation

Do NOT send every session event to a central aggregator.

Instead, workers periodically publish snapshots:

```text
{
  match_id: IPL_FINAL,
  worker_id: C1,
  version: 105,
  active_count: 2400000
}
```

The aggregator maintains the latest count per worker:

```text
C1 -> 2.4M
C2 -> 2.3M
C3 -> 2.5M
C4 -> 2.4M
```

Then:

```text
global_count = SUM(latest worker counts)
```

Important:

Do not add successive snapshots:

```text
2.4M + 2.35M  // wrong
```

Replace the worker's previous value.

Use a worker ID + version/sequence number to handle retries/reordering.

---

# 14. Aggregator HA

Avoid a single central aggregator.

Workers publish snapshots to a durable stream keyed by `match_id`.

Aggregation can itself use a consumer group.

```text
Workers
   |
   v
Aggregation Topic
   |
   +--> Aggregator 1
   +--> Aggregator 2
   +--> Aggregator 3
```

If an aggregator fails:
- partition ownership moves to another instance
- new instance replays durable aggregation events
- it reconstructs latest worker counts
- it resumes publishing global count

Important insight:

> A hot key is only a problem if the workload behind that key is large.

One match may have 10M raw heartbeat events, but the aggregation layer only sees periodic snapshots from perhaps hundreds of workers.

---

# 15. Client Fanout

Do not have the central aggregator maintain 10M client connections.

Use a fanout layer:

```text
Aggregator
    |
    v
Pub/Sub / Fanout
    |
    +--> SSE Node 1
    +--> SSE Node 2
    +--> SSE Node 3
    ...
```

Each SSE server maintains many client connections.

A single match count update is received by the SSE nodes, which broadcast it locally to their connected clients.

Example:

```text
Match A -> 9,642,381
```

Clients receive the new count.

Do not send an update if the count hasn't changed.

---

# 16. SSE Scaling

If 10M clients receive an update every 5 seconds:

```text
10M / 5 = ~2M outbound updates/sec
```

So the fanout layer needs to scale independently.

Different scaling dimensions:

```text
Ingress servers
    -> request/event rate

Stream processors
    -> heartbeat event rate

SSE servers
    -> concurrent connections + outbound messages

Aggregator
    -> worker snapshot rate
```

Video streaming servers/CDN and viewer-count SSE infrastructure should generally be decoupled.

---

# 17. Ingestion Scaling

For predictable events such as IPL:

> Pre-scale before match start using historical traffic patterns.

Do not rely solely on reactive autoscaling.

Keep ingestion servers stateless:

```text
Load Balancer
      |
  +---+---+---+
  |   |   |   |
 I1  I2  I3  I4
```

Any request can go to any ingestion instance.

Use autoscaling as a safety mechanism.

---

# 18. Queue vs Event Stream

A queue/event stream can absorb short bursts and decouple ingestion from processing.

But do not intentionally throttle below sustainable processing capacity.

If 1M heartbeats/sec arrive, eventually the system must process roughly 1M heartbeats/sec.

A queue:

> absorbs bursts

It does NOT:

> eliminate the required processing capacity.

For heartbeats, excessive queue delay is bad because it makes the viewer count stale.

---

# 19. DDoS / Traffic Protection

A reasonable front door:

```text
Internet
   |
CDN / DDoS protection / WAF
   |
Load Balancer
   |
API Gateway / Ingestion
```

Use:
- rate limiting
- WAF
- authentication/session validation
- bot detection
- anomaly detection
- quotas where appropriate

Do not claim a normal API gateway alone solves DDoS.

A legitimate-looking traffic surge is an application-level abuse problem and may require bot/anomaly detection.

---

# 20. Synchronized Heartbeat Problem

If 10M clients all heartbeat at exactly:

```text
0, 10, 20, 30...
```

we get synchronized spikes.

Use client-side jitter:

```text
A -> 8.7 sec
B -> 10.4 sec
C -> 11.2 sec
D -> 9.3 sec
```

This smooths traffic instead of creating huge synchronized bursts.

---

# 21. Duplicate Heartbeats / At-Least-Once Delivery

A stream processor may process an event twice if it crashes before committing its offset.

Example:

```text
heartbeat A
    |
consumer processes
    |
consumer crashes before commit
    |
event redelivered
```

If we did:

```text
active_count++
```

duplicates would be dangerous.

But our model is:

```text
session_id -> last_seen
```

So:

```text
map[A] = timestamp
```

is naturally idempotent for the same heartbeat.

Applying it twice produces the same state.

---

# 22. Out-of-Order Heartbeats

A bigger issue is:

```text
A -> 10:00:20
A -> 10:00:10
```

If the older event arrives second and blindly overwrites the state:

```text
A -> 10:00:10  // wrong
```

Use:

```text
map[A] = max(map[A], incoming_timestamp)
```

or use a client sequence number:

```text
A -> seq 101
A -> seq 102
A -> seq 103
```

Store latest sequence number and ignore older events.

---

# 23. Local State vs Distributed Cache

### Option 1: Distributed Cache

```text
Consumers
    |
    v
Redis / distributed cache

session -> last_seen
```

Advantages:
- shared state
- fast random access
- easy recovery
- consumers can be relatively stateless

Disadvantages:
- ~1M heartbeat writes/sec
- cache becomes another massive distributed dependency
- network overhead
- sharding/replication
- memory capacity
- failure/recovery complexity

### Option 2: Local State + Durable Changelog

```text
Kafka partition
      |
Consumer
      |
Local state
      |
Durable changelog/checkpoint
```

On failure:

```text
partition reassigned
      |
new consumer
      |
restore/replay state
```

This can be preferable because stream processing already has partition ownership and replay semantics.

Key principle:

> Hot state -> local memory is fastest.
>
> Need recovery -> make state durable.
>
> Need shared random access -> distributed cache.
>
> Already have durable event stream -> replay/changelog may remove the need for a cache.

---

# 24. Complete Data Flow

A strong 2–3 minute explanation:

> "I'll define a viewer as an active viewing session, rather than a unique human. Each session sends a heartbeat roughly every 10 seconds, and we'll consider it active if we've seen a heartbeat within 30 seconds. The count can be eventually consistent and around 5 seconds stale.
>
> The client sends heartbeats through a DDoS/WAF layer, load balancer and stateless ingestion servers into a durable event stream. Events are partitioned by session ID so that all heartbeats for a session go to the same stream processor.
>
> Each processor maintains session-to-last-seen state and an expiry structure such as a min-heap. We don't scan all sessions; when an expiry candidate reaches its deadline, we verify the latest last-seen value before removing it. The processor periodically publishes its local active-session count rather than sending individual session counts to a central aggregator.
>
> The aggregation layer consumes these worker snapshots, maintains the latest count per worker, and sums them to get the match-level viewer count. The aggregation stream is durable and partitioned so the aggregator can fail over and reconstruct state.
>
> Finally, the match-level count is published to a fanout layer. SSE servers maintain long-lived client connections and broadcast the latest count to their connected clients. The SSE layer is scaled independently because tens of millions of concurrent connections create a separate scaling problem."

---

# 25. Key Interview Mental Model

Think in layers:

```text
1. Define viewer
       ↓
2. Define activity/timeout
       ↓
3. Ingest heartbeats
       ↓
4. Partition by session
       ↓
5. Maintain per-session state
       ↓
6. Efficient expiration
       ↓
7. Local worker count
       ↓
8. Aggregate worker counts
       ↓
9. Fan out match count
       ↓
10. Handle failures / duplicates / ordering
       ↓
11. Scale each layer independently
```

The most important design principle:

> **Never centralize the 10M-user workload. Distribute the raw heartbeat processing, and only aggregate small summaries.**

---

# 26. Likely Interview Follow-Ups

Be prepared for:

1. Why Kafka/event streaming instead of a normal queue?
2. Why partition by session ID?
3. What happens when one consumer dies?
4. What happens when a partition becomes hot?
5. How do you detect inactive users?
6. Why not simply increment/decrement a counter?
7. How do you handle duplicate heartbeats?
8. How do you handle out-of-order heartbeats?
9. How do you handle 10M clients opening simultaneously?
10. How do you scale SSE to 10M connections?
11. What happens if the aggregator dies?
12. How do you handle a match ending?
13. How would you support historical viewer graphs?
14. How would you support unique viewers rather than concurrent sessions?
15. What consistency guarantees does the system provide?
16. What happens during a regional failure?
17. How do you prevent a single match from creating a hot partition?
18. How much memory is required for 10M sessions?
19. What metrics/alerts would you monitor?
20. What happens if the event stream is temporarily unavailable?

---

# 27. Interview-Level Principles to Remember

### Don't start with technologies

Bad:

> "I'll use Kafka, Redis, Cassandra, Kubernetes..."

Better:

> "We have 1M events/sec, need eventual consistency, and need session-level state. Therefore we need partitioned event processing and recoverable state."

Then choose technologies.

### Separate raw traffic from aggregation traffic

Raw:

```text
~1M heartbeats/sec
```

Aggregation:

```text
hundreds of worker snapshots
```

Never unnecessarily send raw events into the aggregation layer.

### Separate connection scale from event scale

10M concurrent SSE connections is one problem.

1M heartbeat events/sec is another.

Treat them as independently scalable systems.

### Prefer idempotent state transitions

```text
last_seen = max(last_seen, event_timestamp)
```

is much safer than:

```text
counter++
```

### Explain trade-offs

There isn't one universally correct architecture.

The strongest answer is:

> "Given these requirements, I'd choose X because of Y, while the alternative Z trades simplicity/performance/recovery differently."

