# Question

Consider a sale event happening on site like amazon, flipkart. users are expected to try and buy popular items like iphone as soon as the sale start.
We need to design a system that can handle such surge in traffic and ensure we don't oversell


# Solution

## Functional Req
- user should be able to place order
- ensure order placement only if item is in stock(do not oversell)

## Non functional Req
- Scalable to handle surge in traffic (100K users trying to place order)
- Availability - we don't need our system to crash at time of sale

## High level flow
- user place order request, which gets routed by our gateway to the shopping service
- shopping service needs a source of truth to check the stock before placing the order and start the payment process.
- we can store the stock per item in an inventory DB. We decrement the stock as we keep on selling. No shop req should pass if we dont have stock for that item


### How to handle traffic surge
- as soon as sale starts there will be a surge in requests
- we need to pre-scale te system to handle such scenarios as relying on auto-scaling is not enough as it is slow.
- Another thing we can do is to use kafka to queue the shop requests, this will help us throttle the incoming traffic based on what our shopping service(consumers) can handle
- we can have multiple partitions of shop topic with consumers subscribed to parallel process the incoming surge

### How to handle concurrency at DB
- we need to store item->stock values. We can se dynamoDB as KV store to fulfill our req
- DynamoDB is fully managed, so no maintenance overhead, fast reads and writes
- we need to check for every shop that the item is in stock, as we dont want to oversell
- we can apply pessimistic locking but it won't scale(also its available in SQL dbs only) as reqs will starve waiting for lock on the item quantity
- we can go ahead with atomic updates with conditions, so we dont have race conditions b/w requests
- atomic updates in DB are also based on locking so, other req will fail if unable to get lock and we have to retry them
- to enhance this we can keep item->stock mapping in redis. A single redis node can handle 1M/sec ops on good infra. And redis atomic ops are not based on locking but a single thread process req one by one so no retry mech needed.
- also note that that dynamo has a leader partition for every partition on which these conditional writes will go. So the throughput as a upper ceiling if one partition gets too hot due to popular product.
- This is also why we need a fast layer on top of DB to absorb the surge.(REDIS)

#### Flow
- req hits gateway
- gateway routes the req to kafka
- shop service picks the req and decr the quantity in redis.
- shop service req to order service to create order and process payment
- order service create order entry and calls payment service
- payment successful -> order service updates order status to confirm and also update inventory DB(source of truth for stock)
- payment fail/any other failure/ user did not make payment -> order service update order status to cancel and req shop service to incr the stock in redis