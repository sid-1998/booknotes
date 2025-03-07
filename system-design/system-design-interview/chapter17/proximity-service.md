## Core Problem
- how will the location based search work


### Approach discussed in notes

- we will store latitude and longitude of the place in DB(book says to use SQL DB) along with other info
- a service say Sync service will take the lat and long of the new entry convert it into geoHash_4, geoHash_5, geohash_6 precision levels and store in cache as a key value pair(geoHash -> listOf(place_ID)).
- precision 4(within 50Km), 5(within 5km), 6(within 1Km) are commonly searched radius so we will store all 3. **Refer notes for precision table**
- _Alternate caching approach is to use redis inbuilt GEOADD feature(to add businessID and its lat/long in cache) and geoSearch feature to query the cache_
- Also, a separate cache will store business info for these places.
- This will help to retrieve results from cache directly and reduce latency **(Non-Functional req achieved)**
- On cache miss, we have to query DB. Querying based on lat and long is supported in SQL DB like PostGres (PostgreSQL supports geospatial queries using the PostGIS extension).

### Using MongoDB instead of RDBMS
- Service needs to serve 5000 req/s. which is a high load. So a Distributed DB will handle load better. 
- We can partition(Shard) mongoDB based on geoHash and use its inbuilt geospatial searching to query the correct shard. 
- It takes lat/long and radius and calculates geoHash and then query the shard that matches the geohash.
- There is no need of strong consistency so we can go for a noSQL DB. Chances related to info of a business need not to be reflected in real time, so eventual consistency is sufficient here.
- Note that MongoDB does not support auto sharding, so manual partition creation will be needed.


### Other approaches
#### ElasticSearch
- provides ful text search (not there in mongo)
- geoSpatial searching
- provides all noSql features(auto-sharding, partitioning)
- can be used as primary data storage in place of a DB

