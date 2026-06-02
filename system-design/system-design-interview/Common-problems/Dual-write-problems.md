# Transactional Outbox & CDC for Reliable Cache Invalidation

## The Problem

In a distributed system, cache invalidation is often performed using events.

Example:

```text
User Service
    ↓
Update DB
    ↓
Publish UserUpdated Event
    ↓
Kafka
    ↓
Other Services Invalidate Cache
```

This works until a failure occurs.

---

# Dual Write Problem

Suppose:

```text
Update DB ✅
Publish Event ❌
```

Result:

```text
Database = Updated

Caches = Not Invalidated
```

Other services continue serving stale data.

The issue is that the application is trying to write to two different systems:

```text
1. Database
2. Kafka
```

These operations are not atomic.

---

# Why This Happens

Consider:

```text
Update DB
↓
Publish Kafka Event
```

Failure Scenario:

```text
Update DB succeeds
Publish Kafka fails
```

Result:

```text
Data changed
Event lost
```

Consumers never learn about the change.

---

# Bad Solutions

## Publish Event First

```text
Publish Event
↓
Update DB
```

Failure Scenario:

```text
Event Published ✅
DB Update ❌
```

Consumers believe data changed when it actually did not.

---

## DB Update Then Publish Event

```text
Update DB
↓
Publish Event
```

Failure Scenario:

```text
DB Updated ✅
Kafka Publish ❌
```

Data changes but consumers never receive the event.

Neither approach guarantees consistency.

---

# Transactional Outbox Pattern

The Transactional Outbox Pattern solves the dual write problem.

Instead of publishing directly to Kafka, store the event in an Outbox table within the same database transaction.

---

# Outbox Table

Example schema:

```sql
CREATE TABLE outbox (
    event_id UUID,
    event_type VARCHAR(100),
    payload JSON,
    created_at TIMESTAMP,
    status VARCHAR(20)
);
```

---

# Transaction Flow

Instead of:

```text
Update DB
↓
Publish Kafka Event
```

Do:

```text
BEGIN TRANSACTION

Update Business Data

Insert Outbox Event

COMMIT
```

Example:

```text
Users Table:
id=123
name=Johnny

Outbox Table:
eventType=UserUpdated
userId=123
```

---

# Why This Works

Both operations are part of the same database transaction.

Either:

```text
Business Data Updated
+
Outbox Event Inserted
```

or

```text
Neither Happens
```

No possibility of:

```text
DB Updated
But Event Lost
```

---

# Outbox Publisher

A separate component reads events from the Outbox table and publishes them to Kafka.

Flow:

```text
Outbox Table
↓
Outbox Publisher
↓
Kafka
```

---

# Publisher Workflow

```text
Read Unpublished Events
↓
Publish To Kafka
↓
Mark Event Published
```

Example:

```text
Outbox Event
↓
Kafka Publish
↓
Update status = PUBLISHED
```

---

# What If Publisher Crashes?

Scenario:

```text
Read Outbox Event
↓
Crash Before Publish
```

Since the event remains in the Outbox table:

```text
Next Publisher Run
↓
Read Event Again
↓
Publish Again
```

Eventually the event reaches Kafka.

This provides:

```text
At-Least-Once Delivery
```

---

# Drawbacks of Polling

Many Outbox implementations use polling:

```text
Every 1 second:
Read Outbox Table
```

Problems:

```text
Extra DB Queries
Publishing Delay
Scalability Concerns
```

---

# Change Data Capture (CDC)

CDC eliminates polling.

Instead of reading the Outbox table repeatedly, monitor database transaction logs.

Examples:

```text
MySQL Binlog
PostgreSQL WAL
```

Popular tools:

* Debezium
* Maxwell
* AWS DMS

---

# CDC Architecture

```text
Application
      │
      ▼
   Database
      │
      ▼
 Outbox Table
      │
      ▼
 Database Transaction Log
      │
      ▼
    Debezium
      │
      ▼
     Kafka
```

---

# CDC Flow

Application performs:

```text
BEGIN TRANSACTION

Update User

Insert Outbox Event

COMMIT
```

After commit:

```text
Debezium
↓
Reads Binlog/WAL
↓
Detects Outbox Insert
↓
Publishes Event To Kafka
```

No polling required.

---

# Why CDC Is Better

Benefits:

```text
Near Real-Time Publishing
Lower Database Load
Higher Scalability
No Polling
```

CDC simply streams committed database changes.

---

# Cache Invalidation Example

Initial State:

```text
User Name = John
```

User update:

```text
John → Johnny
```

Transaction:

```text
BEGIN

Update User

Insert UserUpdated Event Into Outbox

COMMIT
```

CDC detects:

```text
New Outbox Entry
```

Publishes:

```json
{
  "eventType": "UserUpdated",
  "userId": 123
}
```

Consumers receive:

```text
UserUpdated(123)
```

And perform:

```text
DELETE user:123
```

Cache invalidated.

Future reads rebuild cache using fresh data.

---

# Complete Flow

```text
Update Request
      │
      ▼
  Application
      │
      ▼
 Update User Table
      │
      ▼
 Insert Outbox Event
      │
      ▼
      Commit
      │
      ▼
    Binlog/WAL
      │
      ▼
    Debezium
      │
      ▼
      Kafka
      │
      ▼
 Consumer Services
      │
      ▼
 Invalidate Cache
```

---

# Interview Answer

If asked:

### What happens if DB update succeeds but Kafka publish fails?

Answer:

> This is the dual write problem. To solve it, I would use the Transactional Outbox Pattern. Instead of publishing directly to Kafka, I would write both the business data and an outbox event in the same database transaction. This guarantees that if the transaction commits, the event is persisted. I would then use CDC tools such as Debezium to stream outbox events from the database transaction log to Kafka. Consumer services would receive these events and invalidate their caches. This ensures reliable event-driven cache invalidation and eventual consistency.

---

# Key Concepts

## Dual Write Problem

```text
Database Write
+
Kafka Publish

Cannot be made atomic directly.
```

---

## Transactional Outbox

```text
Business Data
+
Outbox Event

Stored in same DB transaction.
```

---

## CDC

```text
Reads DB Transaction Log
Streams Events To Kafka
```

---

## Cache Invalidation

```text
Consumer Receives Event
↓
Delete Cache Entry
↓
Future Reads Rebuild Cache
```

---

# One-Line Summary

```text
Outbox solves atomicity.
CDC solves reliable publishing.
Kafka consumers solve cache invalidation.
```
