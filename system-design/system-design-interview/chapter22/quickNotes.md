![recalculation-example](images/recalculation-example.png)


- write heavy application- so db choice is Cassandra(nosql db optimized for heavy read/writes)
- we have Large amount of incoming data, so we decouple the aggregation service using queues to avoid overloading the service
- similar way we decoupled the aggregation writes to avoid overloading the DB
- What happens if we lose aggregated data. We store raw data(Again cassandra is used, much older data can be moved into cold storage to save costs). Add a Recalculation service to generate lost aggregated data.
- We also need to implement a reconciliation flow which is a batch job, running at the end of the day. It calculates the aggregated results from the raw data and compares them against the actual data stored in the aggregation database:

### Scaling
#### Message Queue(Kafka):
- partition based on ad_ids, Aggregator nodes will consume a range of ad_ids from a topic
- We could also consider partitioning the topic by geography, eg topic_na, topic_eu, etc.

#### Aggregator Nodes
- each node consume some range of ad_ids, for each ad_id aggregation we can spin up a separate thread to do parallel processing

#### Scaling DB(Cassandra)
- If we use Cassandra, it natively supports horizontal scaling utilizing consistent hashing
- If a new node is added to the cluster, data automatically gets rebalanced across all (virtual) nodes
- With this approach, no manual (re)sharding is required

### What if an aggregator node goes down
- as we are using kafka, and each aggregator node is acting as a consumer. We can group them in consumer group.
- Kafka rebalances the load is a consumer go down in a group.
- Using the offset stored in Zookeeper, the new consumer can start aggregating where the old one left
- **Read more if required from messagingQueue notes**