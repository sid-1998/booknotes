### This notes explain ES with example of flight search system

# Elasticsearch Notes for System Design Interviews
## Flight Search System Example

---

# Why Elasticsearch Instead of DynamoDB?

## DynamoDB Strengths

DynamoDB is excellent for:

- Key-value lookups
- Known access patterns
- High scalability
- Low latency reads/writes

Example:

```text
PK = DEL#BOM#20260720
SK = DepartureTime
```

Query:

```text
Find all flights from DEL to BOM on 20 Jul 2026
```

This is extremely efficient.

---

## Problem with Flight Search

Flight search is not just a route lookup.

Users may search:

```text
DEL -> BOM
Date = 20 Jul 2026
Airline = Air India
Departure = Morning
Price < 7000
Stops = 0
Sort by Price
```

Tomorrow product may add:

- Cabin class
- Refundable fares
- Baggage included
- Aircraft type
- Loyalty benefits

With DynamoDB:

- Query can efficiently use only:
    - PK/SK OR
    - One GSI
- Additional filtering often happens in the application.
- Supporting every filter combination may require many GSIs.

---

# Core Difference: DynamoDB vs Elasticsearch

## DynamoDB

A query can efficiently use:

```text
Primary Key (PK/SK)
OR
One GSI
```

Example:

```text
GSI(Route)
```

Returns:

```text
50,000 flights
```

Then the application must:

```text
Filter Airline
Filter Departure Slot
Filter Price
Sort
Paginate
```

So a lot of search logic moves into the application.

---

## Elasticsearch

Elasticsearch supports:

```text
Route = DEL_BOM
AND Airline = AI
AND DepartureSlot = MORNING
AND Price < 7000
ORDER BY Price
LIMIT 50
```

All executed inside Elasticsearch.

---

# Inverted Index

Traditional storage:

```text
Flight1 -> Air India
Flight2 -> Indigo
Flight3 -> Air India
```

Inverted index:

```text
Air India -> [Flight1, Flight3]
Indigo    -> [Flight2]
```

For flight search:

```text
route=DEL_BOM
    -> [1,2,3,4,5]

airline=AI
    -> [1,3,5,7]

departureSlot=MORNING
    -> [1,4,5,8]

price<7000
    -> [1,5,9]
```

Elasticsearch internally performs:

```text
route
∩ airline
∩ departureSlot
∩ price
```

Result:

```text
[1,5]
```

This is why ES is ideal for search workloads.

---

# Why Not Create Reverse Indexes in DynamoDB?

Technically possible.

But:

```text
Route Index
Airline Index
Price Index
Departure Slot Index
```

would require:

1. Querying multiple indexes
2. Intersecting results
3. Sorting results
4. Paginating results

All inside the application.

Elasticsearch already does this efficiently.

---

# Source of Truth vs Search Read Model

Recommended architecture:

```text
Supplier Updates
      |
    Kafka
      |
Inventory Service
      |
   DynamoDB
      |
 CDC / Streams
      |
  ES Indexer
      |
Elasticsearch
```

### DynamoDB

Source of Truth

### Elasticsearch

Search-optimized read model

---

# What is ES Indexer?

ES Indexer is a service responsible for keeping Elasticsearch synchronized.

Responsibilities:

1. Consume CDC events
2. Transform records
3. Create/update documents in ES
4. Retry failures

Example:

Inventory update:

```json
{
  "flightId":"AI123",
  "availableSeats":11
}
```

ES Indexer performs:

```text
Update Elasticsearch document AI123
```

---

# Flight Search Document

Example ES document:

```json
{
  "flightId": "AI123",
  "from": "DEL",
  "to": "BOM",
  "airline": "AI",
  "departureSlot": "MORNING",
  "availableSeats": 12,
  "currentPrice": 6200
}
```

Search can be served entirely from ES.

---

# Availability & Pricing Design

## Bad Design

```text
Search
   ↓
ES
   ↓
Availability Service
   ↓
Pricing Service
```

Problem:

- Multiple network calls
- Higher latency
- Doesn't scale well

---

## Better Design

Store latest known:

- Price
- Seat count

inside Elasticsearch.

```text
Search
   ↓
ES
   ↓
Ranking
   ↓
Results
```

Validate inventory and pricing later during booking.

---

# Eventual Consistency Strategy

Search optimizes for:

```text
Latency
```

Booking optimizes for:

```text
Correctness
```

Therefore:

```text
Search
    ↓
Use latest known inventory and fare
```

```text
Booking
    ↓
Real-time validation
    ↓
Seat reservation
```

This is a common industry pattern.

---

# Updating Seat Availability

Booking happens:

```text
12 seats
   ↓
11 seats
```

Flow:

```text
Booking Service
      |
Update DynamoDB
      |
CDC Event
      |
ES Indexer
      |
Update ES Document
```

Example:

```json
{
  "flightId":"AI123",
  "availableSeats":11
}
```

---

# Does ES Rebuild Entire Index?

No.

Only the affected document is reindexed.

Example:

```text
Update Flight AI123
```

Elasticsearch updates:

```text
AI123 document
```

NOT:

```text
Entire flights index
```

---

# How Does Inverted Index Get Updated?

When a document update happens:

1. Old document version becomes obsolete
2. New document version is indexed
3. Inverted indexes are updated automatically

You do not manually update inverted indexes.

Lucene handles this internally.

---

# Lucene Internal Behavior

Conceptually:

```text
Update Document
```

Internally:

```text
Mark old version deleted
+
Index new version
```

This happens automatically.

---

# Elasticsearch Mapping

Suppose:

```json
{
  "airline": {
    "type": "keyword"
  }
}
```

Elasticsearch automatically builds the inverted index.

You do NOT create separate indexes like:

```text
airline_index
price_index
departure_index
```

Instead:

```text
flights_v1
```

contains all searchable fields.

---

# Product Adds New Filter

Suppose Product asks:

```text
Filter by Aircraft Type
```

If field already exists and is indexed:

```text
No action needed
```

Just start querying.

---

If field does not exist:

1. Update mapping
2. Create new index version
3. Reindex data

---

# Reindexing

Needed when:

- Mapping changes
- New searchable fields are introduced
- Index corruption
- Migration

Example:

Current:

```text
flights_v1
```

New requirement:

```text
aircraftType
```

Create:

```text
flights_v2
```

with new mapping.

---

# Zero-Downtime Reindexing

Never perform downtime.

Use aliases.

Current:

```text
flights_current
      |
      v
  flights_v1
```

Create:

```text
flights_v2
```

Backfill data.

Then atomically switch:

```text
flights_current
      |
      v
  flights_v2
```

Users experience no downtime.

---

# What Happens During Reindexing?

New updates continue arriving.

Approach:

```text
Inventory Update
       |
     Kafka
       |
   ES Indexer
       |
  flights_v1
       +
  flights_v2
```

Temporarily write to both versions.

After migration:

```text
Delete flights_v1
```

---

# Interview Sound Bites

## Why Elasticsearch instead of DynamoDB?

"DynamoDB is optimized for known key-based access patterns. Flight search requires multi-dimensional filtering, sorting, ranking, and evolving search requirements. Elasticsearch's inverted indexes make these searches efficient, while DynamoDB would require multiple GSIs and application-side filtering."

---

## Why store price and availability in Elasticsearch?

"Search prioritizes latency over perfect consistency. Therefore Elasticsearch stores the latest known inventory and fare snapshot. Final validation happens before booking."

---

## Why DynamoDB + Elasticsearch?

"DynamoDB acts as the source of truth, while Elasticsearch serves as a search-optimized materialized view."

---

## Why not dual-write to DynamoDB and Elasticsearch?

"Dual writes introduce consistency risks. Instead, DynamoDB remains the source of truth and CDC events asynchronously update Elasticsearch."

---

## What is Elasticsearch in one line?

"Elasticsearch is a distributed search engine that uses inverted indexes to efficiently support filtering, sorting, ranking, and full-text search across large datasets."

---