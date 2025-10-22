Lets move ahead from search page and cover the journey from Hotel detail page to checkout to making the booking

## PDP page
- user selects a hotel from search and move to PDP page
- Hotel metadata to populate te page is served from DB. We can cache info to reduce latency
- Dynamic Data like availability and price needs to be re checked by partner to make sure freshness
- if API call fails to partner we serve via cache
- we can use the same Redis+Dynamo arch for availability and price that powers the search
- we use relational DB to store hotel data as it helps to maintain relationship among hotel, room, ratePlan, ameneties as we need joins over multiple tables to serve the info
- Also load on PDP is less than search so RDBMS can handle it. Also, hotel details changes are low on writes so the RDBMS can handle read heavy, so no issue scaling.


## Checkout page and booking flow
- on moving to cko we need to hold the inventory. We can req partner to ack the req and send a hold token if they support else we hold internally.
- we persist token in dynamo and redis. Also decrement the availability count in availability dynamo(our inventory).
- Once again we check price at CKO and create a price token
- on CKO when we click book, price Token is used to check if price changed while booking or not. Hold token ttl is checked is the inventory is in hold or not.
- We call supplier to confirm booking. We can add retries on supplier api call to confirm the booking as they are slow and buggy.
- once partner confirm booking, we take payment and commit a booking record in booking DB.

Redis Hold token 
```
hold:{token} → JSON {hotel, roomType, dates[], rooms, version, expiresAt} (TTL=hold)
```

DynamoDB (inventory/availability source of truth)
```
	•	Holds table: PK holdToken → {hotel, roomType, dates[], rooms, state: ACTIVE|RELEASED|COMMITTED, expiresAt, createdAt, orderId?}
	•	Inventory table (truth): (hotel#date, roomType) counters (total, sold, roomsLeft, version)
```




