# Thundering Herd Problem

## What is the Thundering Herd Problem?

The Thundering Herd Problem occurs when a large number of requests simultaneously attempt to access or regenerate the same resource, causing a sudden spike in load on downstream systems.

A common example is cache expiration.

### Example

Suppose we cache product details:

```text
Key: product:123
TTL: 5 minutes
```

When the cache expires:

```text
10,000 requests
↓
Cache Miss
↓
10,000 DB Queries
↓
Database Overload
↓
Timeouts
↓
Retries
↓
Cascading Failure
```

This sudden synchronized surge is called a **Thundering Herd**.

---

## Where Does It Commonly Occur?

### 1. Cache Expiration (Cache Stampede)

A hot cache key expires and all requests try to regenerate it simultaneously.

Examples:

* Homepage data
* Trending products
* Inventory information
* Pricing data

### 2. Retry Storms

A service becomes unavailable and thousands of clients retry at the same time.

### 3. Lock Contention

Multiple workers continuously compete for the same lock or resource.

---

# Solving the Problem

A common production solution is:

```text
Serve Stale Data
+
Distributed Locking
```

This ensures:

* Users continue receiving responses.
* Only one server refreshes the cache.
* Database receives a single regeneration request.

---

# Stale-While-Revalidate Pattern

Instead of immediately invalidating cache data after TTL expiration, we keep serving the stale value temporarily.

### Example

```text
Cache TTL: 5 minutes
Stale Window: 2 minutes
```

At 5 minutes:

```text
Cache = Expired
Stale Data = Available
```

Requests can still use the stale value while the cache is refreshed in the background.

---

# Distributed Locking Using Redis

The goal is:

> Across all application instances, only one instance should refresh the cache.

## Redis Lock Command

```redis
SET lock:product:123 unique-id NX EX 10
```

### Meaning

* NX → Set only if key does not exist
* EX 10 → Expire lock after 10 seconds

Only one server can acquire the lock.

---

# Example Flow

Assume:

```text
Cache Key = product:123
Cache Entry = Stale
```

Three application servers receive requests.

```text
Server A
Server B
Server C
```

---

## Server A

```text
Read stale data
↓
Return stale data to user
↓
Acquire Redis Lock ✅
↓
Fetch latest data from DB
↓
Update Cache
↓
Release Lock
```

---

## Server B

```text
Read stale data
↓
Return stale data to user
↓
Try Redis Lock ❌
↓
Lock already held
↓
Do Nothing
```

---

## Server C

```text
Read stale data
↓
Return stale data to user
↓
Try Redis Lock ❌
↓
Do Nothing
```

---

# Visual Flow

```text
                   Cache Expired
                          │
                          ▼
               Stale Data Available
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
     Server A          Server B          Server C
        │                 │                 │
        ▼                 ▼                 ▼
 Return Stale       Return Stale      Return Stale
        │                 │                 │
        ▼                 ▼                 ▼
 Acquire Lock       Lock Failed       Lock Failed
        │
        ▼
     Fetch DB
        │
        ▼
   Update Cache
        │
        ▼
   Release Lock
```

Result:

```text
User Latency = Low
DB Calls = 1
```

---

# Java Example (Redisson)

```java
RLock lock = redissonClient.getLock("lock:product:123");

String staleValue = redis.get("product:123");

if (staleValue != null) {

    if (lock.tryLock()) {
        CompletableFuture.runAsync(() -> {
            try {
                String freshValue = db.fetchProduct("123");
                redis.set("product:123", freshValue);
            } finally {
                lock.unlock();
            }
        });
    }

    return staleValue;
}
```

---

# Why Distributed Locking Works

Without locking:

```text
1000 Requests
↓
1000 DB Calls
```

With locking:

```text
1000 Requests
↓
1 Lock Winner
↓
1 DB Call
```

Only one server regenerates the cache.

---

# Do We Still Need Request Coalescing?

Not necessarily.

If:

```text
Serve Stale Data
+
Distributed Lock
```

then all requests can immediately receive stale responses while only one request performs the refresh.

In many systems, this is sufficient.

Request coalescing becomes useful when:

* Stale data cannot be served.
* Requests must wait for fresh data.
* Redis lock traffic becomes significant at very high scale.

---

# Important Production Considerations

## 1. Lock Expiration

Always use TTL on locks.

```redis
SET lock:key value NX EX 10
```

This prevents deadlocks if a server crashes.

---

## 2. Use Unique Lock Values

Store a unique request ID as the lock value.

```redis
SET lock:key request-id NX EX 10
```

Only the owner should release the lock.

---

## 3. Add TTL Jitter

Avoid synchronized cache expiration.

Instead of:

```text
TTL = 300 seconds
```

Use:

```text
TTL = 300 + random(0-60)
```

This spreads refreshes over time.

---

# Interview Summary

### Problem

```text
Hot cache expires
↓
Many requests regenerate simultaneously
↓
Database overload
```

### Solution

```text
Serve Stale Data
+
Redis Distributed Lock
```

### Benefits

* Low user latency
* Single cache refresh
* Prevents database overload
* Works across multiple application instances

### Redis Lock

```redis
SET lock:key unique-id NX EX 10
```

### Key Interview Line

"To mitigate the thundering herd problem, I would serve stale cache data while asynchronously refreshing the cache. A Redis distributed lock ensures that only one application instance performs the refresh, preventing a cache stampede against the database."
