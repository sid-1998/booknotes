Will focus on building end to end hotelk booking system like airbnb, expedia etc
We will focus on core functionalities one by one and come up with HLD and extend it as we solve for more functionalities

## Search
- user can search by location and dates. Plus additional filters like brand, rating etc
- We need to return top K(100) results to user
- then user can go one to select any of the hotel and hotels details page will open


### High Level flow
- client send search req with location and dates in req
- we handle this via a search orchestrator service
- we decode the location into lat/long to run a geo search over our inventory
- options are using geo search to run on DB like postgres or mongo. It will be slow. Instead we can add each hotel as a doc in Elastic search 

which provides faster geo search plus supports text search which can be fulfilled by rever indexes built on fields like brand and rating.
- We then filter this list by requesting to our availability service which tells which hotels have rooms available for given date range.
- After that we can past the filter list from a ranking service which sorts based on ML/analytics to provide personalisation recommendations

#### Why ES
- fast geo querying on load like 10K TPS
- provide text search extended capability
- inverted index on fields that can be used as filters
- Opensearch (AWS version of ES) can be used if system is already on AWS.
- if load is low we can use mongo/postgres but to boost performance we can leverage ES

#### Hotel doc in ES
```
{
  "hotelId": "HTL_987654",
  "name": "Daffodil Residency",
  "geo": { "lat": 28.6309, "lon": 77.2177 },
  "address": {
    "line1": "12 Barakhamba Rd",
    "locality": "Connaught Place",
    "city": "New Delhi",
    "state": "DL",
    "countryCode": "IN"
  },
  "starRating": 4.2,
  "brand": "Independent",
  "amenities": ["wifi", "ac", "breakfast", "parking"],
  "popularityScore": 0.78,
  "priceBand": "MID",
  "minCachedPrice": { "amount": 549900, "currency": "INR", "freshnessTs": "2025-10-20T11:45:12Z" },
  "availabilityHint": { "status": "AVAILABLE", "updatedAt": "2025-10-20T11:45:12Z" }
}
```

#### How to filter based on dates
- we cant store exact availability in ES as these are frequently changing fields, and if we do that we have to re-build the indexed on updated.

as rebuilding index is expensive we want to avoid that, we can store some indicators that might be stale just for filtering/sorting purpose.

but the exact availability and price should come from respective services.
- Source of truth of availability can be a dynamoDB, where count of rooms available, sold etc can be stored against id and date in wide column format
- if we get inventory from third party, we still maintain our source of truth instead of calling supplier API on every search
- On any update in availability, supplier push an event in kafka which is consumed and update is made in DB.
- To reduce latency we serve req from redis and only access db on cache miss. We can set TTL like 2-6h in cache to serve fresh entries. 
- On any updates in DB we invalidate the corresponding entry in cache too to maintain freshness

**DynamoDB row**
```
{
  "hotelId": "H123",
  "date": "2025-12-10",
  "summary": 7,
  "rooms": {
    "DLX": {"total": 5, "sold": 2, "roomsLeft": 3, "version": 128, "source": "OWN", "updatedAt": "2025-10-22T07:21:03Z"},
    "STD": {"total": 6, "sold": 2, "roomsLeft": 4, "version": 130, "source": "OWN", "updatedAt": "2025-10-22T07:21:03Z"}
  },
  "eventClock": {"OWN": 130, "PARTNER_X": 55}
}
```

**Redis entry**
```
avail:summary:H123:2025-12-10 = 7
avail:summary:H123:2025-12-11 = 5
```

#### How to get pricing info
- prices are dynamic and for partner inventory we need fresh prices from their end.
- we can have a similar arch like availability, a redis for price against hotel+roomType and a wide column store as source of truth.
- Always check by calling partner API when we land on hotel details page as the partner is the actual source of truth.
- Also, we can run background refresh jobs if we found cache entry is past a soft TTL. in Soft TTL we serve that price but call API in async to update the price.
- Hard TTL is when we ignore cache price and call API
- if partner API fails to give latest price, we serve the cached price with banner like price may change or final price will be shown at checkout.
- always have a local cache plus persistent store to avoid bombarding partner with API req to check availability and price as they cant handle such load.

Dynamo row
```
{
  "hotelId": "H123",
  "key": "2025-12-10:2025-12-14:A2C0R1:INR:SUPPLIER_X",
  "amount": 549900,
  "currency": "INR",
  "inclusive": true,
  "breakdown": { "base": 476000, "taxes": 53900, "fees": 20000 },
  "sourcePrice": { "amount": 66000, "currency": "USD" },
  "ratePlanId": "RP123",
  "refundable": true,
  "freshnessTs": "2025-10-22T08:05:12Z",
  "softTtlSec": 900,
  "ver": 812,
  "paramsHash": "7f1c…",
  "hotelStay": "H123#2025-12-10#2025-12-14#A2C0R1#INR"
}
```

Redis row
```
price:src:{partner}:{hotelId}:{checkIn}:{checkOut}:{occHash}:{currency}
```



