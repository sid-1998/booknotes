## Functional Req
- User profile
- Search by (place, lat/lon)
- Comments/Rating
- upload images

## Non Functional Req
- High availability
- Scalable
- Low latency


## Estimations
- 100 mill DAU
- 5 search per user on average
- 100 mil * 5 / 100000 secs = 5000 Request/sec


## User Profile

- can be handled by a dedicated user profile service, which expose APIs of login, user profile edit etc
- we can use RDBMS to store user info in a structured tabular form. Can be scaled via sharding based on user id. Also read replication can be used to horizontally scale the DB.

## Comments and Rating
- dedicated service to handle the comments and rating requests by exposing separate APIs
- comments and ratings and be stored in noSQL DB along with postID. Eventual consistency is enough in case of comments and rating so we can go with noSQL approach. 


## Search
- refer proximity service notes for this feature (use Elasticsearch approach)

## Upload images
- refer tinder notes, (basically we will use a Distributed File storage  like S3)



