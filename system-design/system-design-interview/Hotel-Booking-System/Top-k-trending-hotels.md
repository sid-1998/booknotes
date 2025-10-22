🧠 Trending Hotels Aggregation System – Design Notes

Goal

Build a real-time Top-K Trending Hotels system using Kafka and distributed workers.
Supports multiple time windows:
•	Per minute, per hour, per day, per week, and all-time trends.

⸻

1️⃣ Event Ingestion
- Source: Kafka topic hotel_engagement containing hotel interaction events (clicks, views, bookings, etc.).
- Each event includes:
{ hotelId, cityId, timestamp, eventType, weight }
- Partitioning key: hotelId (ensures all events for a hotel go to the same worker).

2️⃣ Worker Tier (Sharded Aggregators)

Each worker:
- Consumes events from Kafka (one or more partitions).
- Maintains aggregated counters per (hotelId, cityId) for different time windows:
- Minute bucket → fine-grained sliding window support (e.g., last hour)
- Hour bucket → last hour, hourly stats
- Day bucket → daily, weekly stats
- All-time bucket → lifetime total interactions
- Performs incremental aggregation:
- For each incoming event, compute its corresponding minute/hour/day key.
- Increment the appropriate counters (in memory or RocksDB state store).

Example counters:
```
minuteCounts[cityId][hotelId][minuteEpoch] += weight
hourCounts[cityId][hotelId][hourEpoch]     += weight
dayCounts[cityId][hotelId][dayEpoch]       += weight
totalCounts[cityId][hotelId]               += weight
```

3️⃣ Local Top-K Computation
- Each worker maintains local Top-K per city using a min-heap or bounded priority queue.
- Since each worker only handles a subset of hotels (based on hotelId hash),
it computes Top-K (or Top-M with buffer) from its partition.

Why min-heap:
Efficient O(log K) insertion and quick retrieval of current Top-K.

Emit periodic updates (e.g., every minute/hour):
```
{ cityId, shardId, windowType: "hour|day", topM: [{hotelId, score}, ...], asOf: timestamp }
```
→ Published to a compacted Kafka topic: topM.shard.{window}.

4️⃣ Global Reducer (Aggregator)

A downstream worker (or set of workers):
* 	Consumes all shard Top-M streams.
* 	Performs K-way merge of sorted Top-M lists from all workers per city.
* 	Produces the final global Top-K per city for each window type.

Algorithm:
Merge K sorted lists using a min-heap of size K (O(K log N) time).
Each hotel belongs to exactly one shard, so merging requires no cross-shard summation.

Output example:
```
{ cityId: "BLR", window: "hour", asOf: "2025-10-22T09:00Z",
  items: [{hotelId: "H11", score: 523}, {hotelId: "H07", score: 489}, ...] }
```

5️⃣ Storage & Serving Layer

- Redis ZSET
- Key: trending:city:{cityId}:{window}
- Value: hotelId → score
- Provides sub-millisecond reads via ZREVRANGE.
- Database (Postgres/Dynamo)
- Stores periodic Top-K snapshots for analytics, history, and cache warm-up.

⸻

6️⃣ API
```
GET /v1/trending?cityId=BLR&window=hour&k=20
→ 200 OK
{
  "asOf": "2025-10-22T09:00Z",
  "items": [
    { "hotelId": "H11", "score": 523 },
    { "hotelId": "H07", "score": 489 },
    ...
  ]
}
```

7️⃣ Design Rationale
- Scalable: Horizontal scaling via Kafka partitioning; each worker handles only a subset of hotels.
- Accurate: Exact counts per window; no approximation needed.
- Low latency: Local in-memory aggregates + Redis reads.
- Flexible: Supports multiple time ranges and easy to extend to other event types.
- Fault-tolerant: Replayable from Kafka, idempotent updates, persistent state (RocksDB/Redis).