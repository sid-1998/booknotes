## Functional req
- user can create post (will take text post for simplicity here)
- user can see feed

## Non Functional req
- low latency <500ms
- eventual consistency
- high availability
- Scale of 100M DAU


## API endpoints

- POST v1/post
```declarative
{
    "user_id": <uuid>,
    "post_id": <uuid>,
    "timestamp": <datetime>,
    "content": String
}
```

user auth can be done via JWT send along the request

- GET v1/feed

returns list of posts to populate news feed sorted as most recent

## POST Flow

- user make post req, gateway authenticates and forward it to post service
- post service writes into cache and post DB
- We can use DynamoDB to store post

### DB schema
```declarative
PK: user_id
SK: timestamp+post_id
{
  "post_id": "550e8400-e29b-41d4-a716-446655440000",
  "author_id": "5678",
  "created_at": 1631874123456,
  "content": "blah"
}
```

#### Why dynamoDB
- Posts are mostly queried by user_id + time (“get latest posts by this author”).
- DynamoDB’s PK+SK model (partition key = user_id, sort key = timestamp+post_id) matches this perfectly.
- Auto-partitioning so no manual shard management
- fast read/write so scales well
- Auto-scales to handle traffic
- fully managed by AWS
- Multi AZ replication so high availability

#### Alternatives
Cassandra (could be used)
Pros
- Similar data model (wide-column, PK+clustering key).
- Excellent for time-series and feed-like workloads.
- Tunable consistency model.
- Open-source, can run outside AWS.
Cons
- Ops burden: adding new nodes to scale, managing load distribution, basically cluster management is requoired
- For an interview: if the company runs on AWS → Dynamo is simpler & safer choice.

Mongo (could be used)
- You (or your ops team) must manage servers, scaling, backups, replication, monitoring, and upgrades (unless using Atlas).
- Even with Atlas, you still tune cluster sizing, regions, etc.
- DynamoDB is Completely serverless and managed. Scaling, replication, patching, failover — AWS handles it all.

SQL (Nahh!)
- no auto-sharding, manual sharding effort required as the scale is high
- we don't need strong consistency so additional ops overhead of manual sharding is waste


## Feed flow
- we need to provide updated feed in near real time <60s as per our req
- brute force idea can be to generate the feed at runtime by first fetching followers of users, 
  
  then getting each follower recent posts via post db and creating a feed for user.
- We can precompute the feed for each user and serve it directly for optimizaton.
- This can be done by maintaining a per-user feed store in cache/DB, by updating there feed store in real time.
- Here, we need to take into consideration for users like celebrities who cna have millions of followers, and using our approach we would be required to do millions of writes to generate precomputed feed
- For this we can have a hybrid approach to fan out on write(push model), or fan out on read(pull model), to create and maintain feeds

### fan out on write(push model)
- for normal users, we can update the follower feeds in real time via an event driven system
- idea will be to push a post event in kafka when a post is created.
- fanout service decided push vs pull model by checking the followers the user have
- if its a push model, fanout service fetches the list of followers of the users, splits them into batches(500-5k per batch)
  
  and performs batch write into feed cache(Redis can be used here, fallback can be a dynamoDB for persistent storage)
- Batch writes will parallelize work across workers.
- Push to cache first (Redis) to ensure low-latency reads; persist to DB asynchronously.
- Use worker sharding by author_id or post_id % N to distribute load.

### fan out on read(pull model)
- for celebrities, whenever they post, fanout service write it to a celebrity cache
- when a user calls GET /feed, the non-celebrity post are served via the feed cache
- fanout service gets celebrities followed by the user, and gets their recent post from celebrity cache
- we merge and sort both the feeds by timestamp and serve the result to user

### How fanout service decide the model
- we decide on the basis of a tunable threshold
- we can use the following criteria #followers * avg post rate by user < Real-time write capacity of workers

### Data models used

#### Follower DB
- fanout service fetches list of followers to precompute the feed in push model
- we will store user per row for followers, so adding/deleting of a follower is O(1)
- we can do this via a wide-column schema approach, Dynamo and cassandra both support this. Why to choose dynamo is already discussed above

Schema
```declarative
PK: user#user_id
SK: follower#follower_user_id
{
    "followed_at": timestamp
}
```

- the follower list can also be cached for active users
- Note: we need to also store which celebrities user follow to help create feed. We can store a set of celebs per user_id in cache for faster access.

#### Feed cache
- use redis list/set to store feed per user_id

#### Feed DB
- we will use this to persist feed data and fallback.

schema
```declarative
PK: feed#user_id
SK: timestamp+post_id
{
  post related data
}
```

#### celebrity cache
- Redis can be used to store celeb_id->list of last M posts type of mapping






