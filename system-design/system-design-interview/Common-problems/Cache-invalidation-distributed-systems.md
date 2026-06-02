# Cache Invalidation in Distributed Systems

## Why Cache Invalidation Is Hard

Cache invalidation is considered one of the hardest problems in distributed systems because multiple services and requests may observe different versions of data at the same time.

Example:

```text
T1:
Cache contains Price = ₹100

User A reads ₹100

T2:
Price updated to ₹105
Cache invalidated/refreshed

User B reads ₹105
```

Now:

```text
User A saw ₹100
User B saw ₹105
```

This inconsistency is expected in most large-scale systems and is called **eventual consistency**.

---

# Key Principle

For most read-heavy systems:

```text
Scalability
+
Availability
+
Low Latency

are preferred over

Strong Consistency
```

Examples:

* Product catalogs
* User profiles
* Search results
* Recommendations
* Social media feeds

These systems usually accept temporary inconsistency.

---

# Cache Aside Pattern (Most Common)

Also called:

```text
Lazy Loading
```

This is the most common caching strategy.

---

## Read Flow

```text
Request
 ↓
Redis Cache
 ↓
Hit?
 ├── Yes → Return Data
 └── No
       ↓
      Database
       ↓
  Populate Cache
       ↓
   Return Data
```

Example:

```java
User user = redis.get("user:123");

if(user == null) {
    user = db.getUser(123);
    redis.set("user:123", user);
}

return user;
```

---

## Write Flow

```text
Update DB
 ↓
Delete Cache
```

Example:

```java
db.updateUser(user);

redis.delete("user:123");
```

Next read:

```text
Cache Miss
 ↓
DB Read
 ↓
Repopulate Cache
```

---

# Why Delete Cache Instead of Updating Cache?

A common interview question.

---

## Option 1: Update Cache

```text
Update DB
 ↓
Update Cache
```

Sounds efficient, but creates consistency problems.

### Race Condition Example

Initial State:

```text
DB = John
Cache = John
```

Thread A:

```text
Cache Miss
 ↓
Reads DB (John)
```

Before A updates cache...

Thread B:

```text
Update DB → Johnny
Update Cache → Johnny
```

State:

```text
DB = Johnny
Cache = Johnny
```

Thread A resumes:

```text
Writes John into Cache
```

Result:

```text
DB = Johnny
Cache = John ❌
```

Old data overwrote fresh data.

---

## Option 2: Delete Cache

```text
Update DB
 ↓
Delete Cache
```

Result:

```text
DB = Source of Truth
Cache = Rebuilt Later
```

Much simpler and safer.

---

# Cache Invalidation at Scale

Single-service invalidation is easy.

The challenge is:

```text
User Service
Order Service
Search Service
Profile Service
```

All may cache the same data.

When User Service updates data:

```text
How do other services know?
```

---

# Event-Driven Cache Invalidation

Most common distributed approach.

Flow:

```text
Update DB
 ↓
Publish Event
(UserUpdated)
 ↓
Kafka
 ↓
Other Services
 ↓
Delete Cache
```

Diagram:

```text
          User Service
                │
                ▼
             MySQL
                │
                ▼
        UserUpdated Event
                │
                ▼
              Kafka
        ┌───────┼────────┐
        ▼       ▼        ▼
     Order   Search   Profile
     Cache   Cache    Cache
        │       │        │
        ▼       ▼        ▼
     Delete  Delete   Delete
```

Future reads rebuild the cache from DB.

---

# Why Use TTL as Well?

Suppose:

```text
Kafka Consumer Down
Message Lost
Bug in Event Processing
```

Cache may never get invalidated.

TTL acts as a safety net.

Example:

```text
TTL = 30 minutes
```

Worst case:

```text
Stale Data survives for 30 minutes
```

instead of forever.

---

# Handling Eventual Consistency

Example:

```text
Service A reads V1

Update occurs

Service B reads V2
```

This is usually acceptable.

Most systems accept:

```text
Different requests
may see different versions
for a short period.
```

Trying to eliminate this completely usually hurts scalability.

---

# Example: Product Pricing

Scenario:

```text
User sees Price = ₹100

Price updated to ₹105

User places order
```

What should happen?

---

## Option 1: Reject Order

Checkout validates latest price.

```text
Current Price = ₹105
Displayed Price = ₹100

Mismatch
 ↓
Ask User to Confirm
```

Common in e-commerce.

---

## Option 2: Price Locking

Create a quote:

```text
Quote:
Price = ₹100
Valid Until = 10:05 PM
```

Checkout honors the quoted price.

Common in:

* Flights
* Hotels
* Reservation systems

---

## Option 3: Hybrid (Most Common)

```text
Browse Pages → Cache

Checkout → Source of Truth
```

At checkout:

```text
Validate Price
Validate Inventory
Validate Discounts
Validate Taxes
```

Then create order.

---

# Approaches to Cache Invalidation

## 1. TTL-Based

```text
Cache expires automatically
```

Pros:

* Simple

Cons:

* Stale data window

---

## 2. Cache Aside + Delete

```text
Update DB
Delete Cache
```

Pros:

* Most common
* Easy to reason about

Cons:

* Temporary stale reads

---

## 3. Event-Driven Invalidation

```text
Update DB
Publish Event
Invalidate Everywhere
```

Pros:

* Scales well
* Near real-time invalidation

Cons:

* Event delivery complexity

---

## 4. Write Through Cache

```text
Write Cache
 ↓
Write DB
```

Pros:

* Cache always fresh

Cons:

* More complexity
* Higher write latency

---

# Recommended Interview Answer

If asked:

"How would you handle cache invalidation in a distributed system?"

Answer:

> I would use the Cache Aside pattern. On writes, I would update the database first and invalidate the cache instead of updating it. In a microservice architecture, I would publish domain events through Kafka so that all interested services can invalidate their caches. I would also keep a TTL as a safety mechanism in case invalidation events are missed. Since the system is read-heavy, I would accept eventual consistency and rely on the database as the source of truth.

---

# Interview Summary

## Read Path

```text
Cache
 ↓ miss
DB
 ↓
Populate Cache
```

## Write Path

```text
Update DB
 ↓
Delete Cache
```

## Distributed Invalidation

```text
DB Update
 ↓
Kafka Event
 ↓
All Services Delete Cache
```

## Safety Mechanism

```text
TTL
```

## Consistency Model

```text
Eventual Consistency
```

## Key Principle

```text
DB = Source of Truth

Cache = Optimization Layer
```
