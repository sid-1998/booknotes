## Core problems

### How data is modeled
- we use relational DB to model hotel data as we need relationships b/w hotel, romms, room_type
- we need ACID properties for reservation transactions
- application is read heavy so we cna scale via read replication
- A good catch is to precompute the inventory(see room_inventory table in notes), this helps reduce query time at the time of reservations. We can also cache this data to reduce latency. This approach avoid joins on tables to return the available inventory

###### Add On
- we can also have a nosql key-value db to store amenities offered in room by keeping room_type_id as key. This info can be cached. This helps us to build the hotel room details page

### Concurrency issues
- Two users can reserve same room at same time, leading to conflicts.
- This can be avoided via **Optimistic locking**. We will maintain a version in inventory table. Each update increments the version by one and no update is made if the version does not exceeds the existing version.(See in notes for more details)
- Optimistic locking also prevents consistency issues occurred during read replication lag.

### Scaling issues
- DB can be scaled using sharding based on hotel id. (Use consistent hashing)
- Services are stateless so can be horizontally scaled easily
- as we need to update multiple dbs during a transaction, we need to implement SAGA PATTERN.

**SAGA Pattern**
- Best when eventual consistency is acceptable.
- Breaks transactions into smaller steps, each with a compensating rollback action.

How Saga Works:
- Reservation Service reserves the room.
- Payment Service charges the user.
- If payment fails, Reservation Service cancels the booking (compensating action)

Two Types of Sagas:
- Choreography → Each service listens to events and reacts.
- Orchestration → A central Saga Orchestrator controls the flow.