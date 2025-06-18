#### how to handle if one shard of db gets hot suddenly

First, protect the shard:

- Enable rate limiting at the API gateway to cap excessive requests.

- Use a queue or buffer for writes if possible — this prevents overload.

- Serve reads from cache:

    If there’s a cache layer (e.g., Redis), push hot data there temporarily.

    If there are read replicas for that shard, route reads to replicas.

- Monitor closely:

    Check CPU, memory, and IOPS — scale up instance size if vertical scaling is possible.

Step 2: Short-Term Fix

- If replicas aren’t enough:

    Spin up additional replicas dynamically and add them to the read pool.

- Update the service discovery or load balancer to distribute reads better.

Root Cause & Longer-Term Fix

- Check why this shard is hot:

    Is the sharding key poorly distributed?

    Is a single user/entity causing this? E.g., celebrity account or trending product.

- If so, re-shard or rebalance:

    For key-value stores, increase shard count and redistribute data using consistent hashing.

    For relational DBs, split the shard manually or move hot rows to a new shard.

    If the DB supports dynamic re-sharding (like DynamoDB auto-splits), trigger that.

- Consider changing the sharding key:

    Pick a key with better cardinality and more uniform distribution.

#### how to have concurrency for db writes

- avoid locking rows(pessimistic locking)
- use pessimistic locking based on versions or timestamps and fail fast the req which reads later. This is good, but it causes many req to fail in highly concurrent setup
- BEST : used atomic conditional writes instead of row locks as they are faster and req don't wait to take the lock. DB handles concurrency internally which is faster than app server taking a lock for a req 

#### how to handle backpressure in queue

if producers push faster than consumers can process, the queue grows → risk of:

    OOM (Out of Memory)

    High latency / timeouts

    System crash

This is backpressure: your system must slow down or reject input when downstream is overloaded.
- Add an API gateway rate limiter:

    Limit requests per user or per product.

    Shed excess load early — before it hits the queue.
- Scale consumers
- Add more partitions: more partitions allow higher parallelism and consumer scaling.
- DLQ (Dead Letter Queue): ensures failed or poison messages don’t block healthy ones. Also helps with retries and isolating bad data.

#### How would you handle this “hot key” problem to keep your cache and system healthy?
- Adding replicas: this is the most common fix for hot keys in Redis. Multiple read replicas let you spread read traffic so a single node isn’t overwhelmed.
- Local caching: For super-hot keys, add a short-lived in-process cache (like Caffeine, Guava, or simple local LRU) at the app level
- CDN cache: If it’s read-only data (like a public product page), cache it at the CDN or edge layer to take the heat off Redis altogether.
- Rate limiting: Throttle abusive or bot-like traffic hitting the hot key if it’s spiky.

#### what practical strategies would you use to handle imbalance in load on nodes of your key value store, given that consistent hashing is already in place?
- adding virtual nodes on hash rings helps redistribute/ share the load among nodes
- adding more physical nodes redistributes the spread on hash ring
- local or distributed  caching for hot keys
- caching at CDN level

#### How would you design your API layer and database access to handle high concurrency efficiently and avoid exhausting DB connections?
- adding caching
- add read replicas, route reads to replicas and writes to primary
- indexing
- for long term fix, do profiling to find out bottlenecks in system



#### As your user base grows to millions of active users, what are the practical ways you would scale this SQL database horizontally?
- sharding by user ID, which is a very common and practical horizontal scaling strategy for SQL databases.
- hash-based routing, which is correct for mapping requests to the right shard
- indexing, which is critical for query performance regardless of sharding.
- replication (primary-secondary) for high availability and fault tolerance
- functional / logical splitting (e.g., order DB vs payment DB vs user DB) — this is also a solid dimension for scaling beyond pure sharding.

