# HLD Interview Notes — Watch History & Resume Video

## Problem

Design a Watch History & Resume Video system with cross-device synchronization.

A user may watch the same video across phone, laptop, TV, etc. When they switch devices, the new device should resume approximately where they left off.

---

# 1. Requirements

### Assumptions

* Cross-device synchronization is **user/account based**, not device based.
* Resume does not need exact second-level accuracy.
* Target roughly **5–10 seconds of staleness**.
* Eventual consistency is acceptable.
* 100M registered users.
* 10M DAU.
* ~2M concurrent viewers.
* Playhead update roughly every 10 seconds.
* Peak ~200K playhead events/sec.

### Features

1. Resume a specific video.
2. Continue Watching — recent unfinished videos.
3. Cross-device resume.
4. High availability.
5. High scalability.

---

# 2. Resume vs Continue Watching

### Resume

> Where should I start this specific video?

Example:

```
Movie A → 35:20
```

### Continue Watching

> Which videos are unfinished and recently watched?

Example:

```
Movie A → 35:20
Movie B → 12:10
Movie C → 55:00
```

Therefore:

```
Resume
    ↓
Specific (user, video)

Continue Watching
    ↓
List of recent unfinished videos
```

---

# 3. Start With Access Patterns

Don't start with:

> Which database should I use?

Start with:

> What queries do I need to support?

### Access Pattern 1 — Resume

```
(user_id, video_id)
        ↓
latest playback state
```

Needs an efficient point lookup.

### Access Pattern 2 — Continue Watching

```
user_id
   ↓
recent unfinished videos
   ↓
ORDER BY updated_at DESC
LIMIT 10–20
```

Needs efficient per-user ordered retrieval.

---

# 4. Data Model

Logical record:

```
user_id
video_id
playhead_seconds
updated_at
completed
```

Potential additional fields:

```
session_id
device_id
event_time
version / ordering metadata
expires_at
```

We do NOT need to permanently store every 10-second playhead update.

We mainly need the **latest state** for Resume/Continue Watching.

---

# 5. DynamoDB Data Model

## Base Table

```
PK = user_id
SK = video_id
```

Attributes:

```
playhead_seconds
updated_at
completed
expires_at
```

Example:

```
user123
 ├── MovieA → 35:20
 ├── MovieB → 12:10
 └── MovieC → 55:00
```

This supports:

```
GetItem(user_id, video_id)
```

for Resume.

---

# 6. GSI for Continue Watching

The base table is optimized for:

```
(user_id, video_id)
```

But Continue Watching needs:

```
user_id
   ↓
recent videos
   ↓
ordered by updated_at
```

So create:

```
GSI PK = user_id
GSI SK = updated_at
```

Query:

```
PK = user_id
ScanIndexForward = false
Limit = 20
```

This gives recent videos first.

### Composite GSI sort key

We could also use:

```
updated_at#video_id
```

The `#` is just an application-defined delimiter.

It is NOT a DynamoDB operator.

---

# 7. PK + SK Uniqueness

### Base Table

If:

```
PK = user_id
SK = video_id
```

then:

```
(user_id, video_id)
```

must uniquely identify an item.

Valid:

```
user123 | MovieA
user123 | MovieB
```

Invalid as two separate items:

```
user123 | MovieA
user123 | MovieA
```

### GSI

A GSI does NOT require unique PK + SK combinations.

For example:

```
user123 | 21:30 | MovieA
user123 | 21:30 | MovieB
```

is allowed.

---

# 8. Why DynamoDB?

Don't say:

> DynamoDB because it scales.

Build the argument from requirements.

### Predictable Access Patterns

We have:

```
(user_id, video_id)
```

and:

```
user_id → recent updated videos
```

DynamoDB keys and GSI map directly to these access patterns.

### High Write Throughput

At ~2M concurrent viewers:

```
2M / 10 sec
≈ 200K events/sec
```

This is a high-volume write workload.

We don't need:

* joins
* complex relational transactions
* arbitrary analytical queries

We mainly need:

```
write latest state
read latest state
```

### Horizontal Scalability

DynamoDB provides a distributed, horizontally scalable storage model.

### High Availability

Playback state is user-facing state and should remain available even if individual storage nodes fail.

### Eventual Consistency

The application can tolerate a few seconds of staleness.

Important:

> DynamoDB supports different consistency options. It is our application requirement that allows eventual consistency.

---

# 9. Why Not PostgreSQL?

PostgreSQL is absolutely viable at smaller scale.

It's attractive when we need:

* relational constraints
* joins
* complex queries
* strong transactional semantics

At our scale:

```
~200K writes/sec
100M users
high availability
horizontal scalability
```

our workload is mostly predictable key-value access.

Therefore DynamoDB is a strong fit.

Interview answer:

> "I'd consider PostgreSQL for a smaller deployment or if we required richer relational queries. At this scale, our workload is mostly predictable key-value access with very high write throughput, so DynamoDB is a better fit."

---

# 10. Why Not Redis?

Redis is excellent for low-latency access, but playback state is durable user state.

Prefer:

```
DynamoDB = source of truth
Redis    = optional cache
```

if caching becomes necessary.

---

# 11. Write API

Authenticated client does not need to provide the authoritative user ID.

Example:

```
PUT /v1/videos/{videoId}/playback
```

Request:

```
{
  "playhead_seconds": 2120,
  "sequence_number": 105
}
```

The API validates the token and resolves:

```
user_id = authenticated identity
```

Then creates an enriched Kafka event:

```
{
  "user_id": "123",
  "video_id": "movieA",
  "playhead_seconds": 2120,
  "event_time": "...",
  "session_id": "...",
  "device_id": "...",
  "sequence_number": 105
}
```

Why put user_id in Kafka?

Because downstream consumers need it for:

* partitioning
* coalescing
* DynamoDB persistence

---

# 12. Read APIs

## Resume

```
GET /v1/videos/{videoId}/playback
```

Internally:

```
GetItem(user_id, video_id)
```

Returns:

```
{
  "video_id": "movieA",
  "playhead_seconds": 2120,
  "updated_at": "...",
  "completed": false
}
```

## Continue Watching

```
GET /v1/users/me/continue-watching?limit=20
```

Internally:

```
Query GSI
PK = user_id
ORDER BY updated_at DESC
LIMIT 20
```

Using `/me` is preferable when identity comes from authentication.

---

# 13. Async Write Architecture

```
Client
  ↓
Playback API
  ↓
Kafka
  ↓
Playback Consumer
  ↓
DynamoDB
```

The API can acknowledge after Kafka successfully accepts the event.

Example:

```
202 Accepted
```

Why async?

* Lower API latency.
* Decouples API from DB.
* Absorbs bursts.
* Consumers can batch.
* Consumers scale independently.
* DB issues don't immediately block ingestion.

Trade-off:

```
Write accepted
    ↓
DB persistence happens later
```

Therefore playback state is eventually consistent.

---

# 14. Playhead Update Frequency

Use approximately:

```
every 10 seconds
```

At 2M concurrent viewers:

```
2M / 10
≈ 200K events/sec
```

If we updated every second:

```
≈ 2M events/sec
```

10 seconds provides a good trade-off between:

* resume accuracy
* write volume
* cost
* scalability

### Important lifecycle events

Send immediately:

```
pause
background
app exit
seek
video ended
```

This improves resume behavior without requiring constant DB writes.

---

# 15. Batching

Consumer should batch writes.

Flush when:

```
batch_size >= N
OR
time_since_oldest_event >= T
```

Use BOTH size and time thresholds.

### Why size?

Under high traffic:

```
many events
   ↓
batch quickly fills
   ↓
flush
```

### Why time?

Under low traffic:

```
few events
   ↓
batch may never fill
```

Time threshold ensures bounded latency.

Example starting point:

```
N = modest batch size
T ≈ 1 second
```

Exact values should be tuned based on:

* DynamoDB limits
* consumer memory
* throughput
* latency requirement

---

# 16. Event Coalescing

Batching alone doesn't eliminate duplicate updates.

Example:

```
A, MovieX → 10:10
B, MovieY → 05:00
A, MovieX → 10:20
C, MovieZ → 15:00
A, MovieX → 10:30
```

Use:

```
Map<(user_id, video_id), latest_valid_event>
```

Result:

```
A, MovieX → 10:30
B, MovieY → 05:00
C, MovieZ → 15:00
```

Then write only those states.

Benefits:

* fewer DB writes
* fewer GSI writes
* less network traffic
* lower write amplification

---

# 17. Kafka Offset Commit

Correct ordering:

```
Kafka
 ↓
Consumer
 ↓
Coalesce
 ↓
DynamoDB write
 ↓
SUCCESS
 ↓
Commit Kafka offset
```

Incorrect:

```
Kafka
 ↓
Commit offset
 ↓
DynamoDB
 ↓
Crash
```

The incorrect design can lose data.

If DB write succeeds but consumer crashes before offset commit:

```
Kafka replays event
```

Therefore DB updates must be idempotent.

This gives:

```
At-least-once processing
+
Idempotent DB updates
=
Safe retry/recovery
```

---

# 18. Kafka Partitioning

Possible keys:

```
video_id
session_id
user_id
(user_id, video_id)
```

## video_id — bad

A hugely popular video can become a hot partition:

```
IPL Final
    ↓
millions of viewers
    ↓
same partition
```

## session_id

Good for ordering one device/session.

But:

```
Phone session → Partition 1
TV session    → Partition 7
```

No ordering guarantee between them.

## user_id

All devices for a user go to the same partition.

Provides ordering across the user's devices.

## user_id + video_id — strong choice

Our authoritative state is:

```
(user_id, video_id)
```

So:

```
Kafka key = hash(user_id + video_id)
```

Example:

```
User A + Movie A → P7
User A + Movie B → P3
User B + Movie A → P9
```

We don't need ordering between different videos.

Key principle:

> Partition according to the entity whose updates require ordering.

---

# 19. Kafka Partition Count

Don't arbitrarily say:

```
20 partitions
```

Estimate based on throughput.

Example:

```
Peak traffic ≈ 200K events/sec
Assumed safe partition throughput ≈ 10K events/sec
```

Minimum:

```
200K / 10K
≈ 20 partitions
```

Then provision additional headroom.

Consider:

* traffic spikes
* failures
* replication
* uneven distribution
* event size
* broker capacity
* consumer throughput

---

# 20. Concurrent Device Updates

Example:

```
Phone → MovieA → 20:00
TV    → MovieA → 15:00
```

Device-local sequence numbers are insufficient:

```
Phone seq 101
TV seq 51
```

They are independent sequences and cannot be compared.

Possible event fields:

```
user_id
video_id
playhead
device_id
session_id
server timestamp / ordering metadata
```

---

# 21. Conflict Resolution

Define a product rule:

> Latest server-observed playback update wins.

API attaches server-observed ordering metadata.

Store:

```
playhead
last_updated_at
```

DynamoDB condition:

```
Update only if:
incoming_updated_at > stored_updated_at
```

Example:

```
Stored:
20:00 @ 10:10:20

Incoming:
15:00 @ 10:10:15

→ reject incoming update
```

This prevents stale events from overwriting newer state.

### Do NOT use MAX(playhead)

Incorrect:

```
playhead = MAX(current, incoming)
```

because users can seek backward.

Example:

```
20:00
 ↓
user seeks back
 ↓
05:00
```

05:00 may legitimately be the latest state.

---

# 22. Cross-Device Resume

Device A:

```
Phone
Movie A → 35:20
```

Device B:

```
TV
 ↓
GET playback
 ↓
DynamoDB
 ↓
35:20
```

No polling required.

Important distinction:

> Cross-device Resume ≠ real-time cross-device synchronization.

If the requirement becomes:

> Pause on phone immediately pauses TV.

Then use:

* WebSocket
* SSE
* pub/sub

But that is a different requirement.

---

# 23. Async Write and Stale Reads

Possible flow:

```
Phone
 ↓
Kafka
 ↓
Consumer has not persisted yet

TV
 ↓
GET playback
 ↓
DynamoDB
```

TV may get an older value:

```
35:10
```

while phone has already reached:

```
35:20
```

This is acceptable because the requirement allows a few seconds of staleness.

Important interview distinction:

> The stale read may be caused by our asynchronous persistence pipeline, not necessarily DynamoDB's eventual-consistency mode.

Even a strongly consistent DB read cannot return an update that has not yet been persisted.

---

# 24. Stronger Consistency for Critical Events

Normal:

```
playhead every ~10 sec
 ↓
Kafka
 ↓
async DB write
```

Critical lifecycle event:

```
pause / exit / background
 ↓
stronger persistence path
```

This allows:

```
Normal playback
→ eventual consistency

Important user action
→ stronger guarantee
```

Not every operation needs the same consistency guarantee.

---

# 25. Completion

Playback state:

```
completed = false
```

When user reaches end / completion threshold:

```
completed = true
```

Completed videos should no longer appear in Continue Watching.

If the product requires historical Watch History:

```
Playback State
     ↓
completed
     ↓
Watch History
```

can be retained separately.

---

# 26. Retention

Separate:

### Continue Watching UI retention

Show only:

```
latest 10–20 unfinished videos
```

### Playback-state retention

Don't necessarily delete older unfinished videos just because they are not in the top 20.

Why?

User might open an older video and still expect Resume.

Therefore:

```
UI retention ≠ data retention
```

---

# 27. DynamoDB TTL

DynamoDB supports native TTL.

Store:

```
expires_at
```

Example:

```
{
  "user_id": "123",
  "video_id": "movieA",
  "playhead": 2120,
  "updated_at": "...",
  "expires_at": "..."
}
```

Configure `expires_at` as the TTL attribute.

DynamoDB asynchronously removes expired items.

Important:

> TTL deletion is not exact-time deletion.

If:

```
expires_at = 12:00
```

the item may remain temporarily after 12:00.

That's fine because retention cleanup doesn't require exact timing.

Prefer TTL over:

```
SCAN entire DB
→ find old records
→ delete
```

at huge scale.

---

# 28. Complete Architecture

```
                     Client
                       │
                playhead ~10 sec
                       │
                       ▼
              ┌─────────────────┐
              │  Playback API   │
              │ Auth + Validate │
              └────────┬────────┘
                       │
                       ▼
                ┌────────────┐
                │   Kafka    │
                │ key=user + │
                │   video    │
                └─────┬──────┘
                      │
                      ▼
            ┌────────────────────┐
            │ Playback Consumer  │
            │                    │
            │ Coalesce           │
            │ Batch              │
            │ Idempotency        │
            └─────────┬──────────┘
                      │
                      ▼
                ┌────────────┐
                │ DynamoDB   │
                │            │
                │ PK=user_id │
                │ SK=video_id│
                └─────┬──────┘
                      │
                     GSI
                      │
                      ▼
             user + updated_at
                      │
                      ▼
              Continue Watching
```

Read path:

```
Client
  ↓
Playback API
  ↓
DynamoDB
  ├── Base Table → Resume
  └── GSI        → Continue Watching
```

---

# 29. Interview-Ready 2–3 Minute Explanation

> "I'll model playback state per user and video because our main requirements are Resume and Continue Watching. Each active client sends a playhead update roughly every 10 seconds, with important lifecycle events such as pause or background sent immediately. We accept a few seconds of eventual consistency because exact second-level synchronization isn't required.
>
> The client sends the event through an authenticated stateless Playback API. The API resolves the user identity from the token and publishes an event containing user ID, video ID, playhead, session information and ordering metadata to Kafka. I'd partition Kafka by `(user_id, video_id)` so updates for the same playback record remain ordered while different users and videos can be processed in parallel.
>
> Consumers maintain a short coalescing buffer. If a user sends multiple updates for the same video within the buffer window, we retain only the latest valid state. We flush when either a size threshold or time threshold is reached and batch-write the resulting states to DynamoDB. We commit Kafka offsets only after successful persistence, giving us at-least-once processing, so the database updates need to be idempotent.
>
> For storage, I'd use DynamoDB with `(user_id, video_id)` as the primary key because Resume is a direct point lookup. I'd create a GSI with `user_id` as the partition key and `updated_at` as the sort key for Continue Watching, allowing us to efficiently retrieve the user's most recently watched items.
>
> For concurrent devices, device-local sequence numbers aren't sufficient across devices. I'd use server-observed ordering metadata and a conditional DynamoDB update so an older event cannot overwrite a newer state. I would not use MAX(playhead), because users can seek backwards.
>
> For cross-device Resume, Device B simply reads the latest persisted playback state when it opens the video. We don't need polling or real-time push unless the product explicitly requires live synchronization between simultaneously active devices.
>
> Finally, completed videos are removed from Continue Watching, while long-term playback state can use DynamoDB TTL for retention. The UI can show only the latest 10–20 unfinished videos without necessarily deleting older playback state."

---

# 30. Key HLD Lessons

### 1. Start from access patterns

```
Requirements
    ↓
Access patterns
    ↓
Data model
    ↓
Database
```

### 2. Don't persist unnecessary history

For Resume/Continue Watching:

```
latest state > complete event history
```

Kafka can retain the event stream for replay/history if required.

### 3. Separate ingestion from persistence

```
API → Kafka → Consumer → DB
```

### 4. Batch AND coalesce

Batching:

```
reduces network overhead
```

Coalescing:

```
reduces actual logical writes
```

### 5. At-least-once requires idempotency

Correct:

```
persist
 ↓
commit offset
```

### 6. Partition around ordering requirements

```
(user_id, video_id)
```

is the entity whose updates need ordering.

### 7. Don't confuse eventual consistency with asynchronous persistence

An eventual DB read is different from:

> The update hasn't reached the DB yet.

### 8. Cross-device Resume doesn't require real-time push

Add WebSockets/SSE only if real-time synchronization is a requirement.

### 9. Separate UI retention from data retention

Top 20 Continue Watching does not mean delete everything else.

### 10. Use TTL for long-term cleanup

Storage-native TTL avoids expensive application-side scans.

---

# 31. Likely Interview Follow-Ups

1. Why Kafka instead of a queue?
2. Why partition by `(user_id, video_id)`?
3. Why not partition by session ID?
4. How do you handle two devices simultaneously watching the same video?
5. How do you handle out-of-order events?
6. How do you make DynamoDB updates idempotent?
7. What happens if the consumer crashes after DB write but before Kafka offset commit?
8. How do you coalesce events safely?
9. How do you handle a huge traffic spike?
10. How do you prevent DynamoDB hot partitions?
11. Why DynamoDB over PostgreSQL?
12. Why not Redis?
13. Why use a GSI?
14. What happens when `updated_at` changes on every playhead update?
15. Would a separate Continue Watching materialized view be better?
16. How do you handle completed videos?
17. How long should playback state be retained?
18. How does DynamoDB TTL work?
19. What if the user is offline?
20. What if the product requires real-time synchronization across devices?

---

# 32. Core HLD Reasoning Framework

For HLD interviews, repeatedly follow:

```
Requirements
      ↓
Access Patterns
      ↓
Data Model
      ↓
Scale / Bottleneck
      ↓
Architecture
      ↓
Failure Modes
      ↓
Consistency
      ↓
Scaling
      ↓
Trade-offs
```

The goal is not to add as many technologies as possible.

The goal is:

> **Every component should exist because a specific requirement or bottleneck demands it.**
