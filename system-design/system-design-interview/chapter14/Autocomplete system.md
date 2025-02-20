## Prerequisite
You should know what a Trie data structure is. Refer this: https://www.youtube.com/watch?v=AXjmTQ8LEoI


## Overview
**First go through the bookNotes, it's good**

Basically we can divide the system into two parts. One is **Data gathering Service** that gathers data based on analytics log and create/update the trie DB used to store the data

Other one is our **query service** which returns the relevant suggestions.

Details can be found in bookNotes.

**Here we will discuss what's not there**

## Scaling the storage
As there will be millions of queries, a single tri cant fit all the data.

- We can split the tri(shard it) based on prefix, say a-m goes to T1, m-z in T2

- This info can be kept with a zookeeper. So each time app server needs to query the db(cahce miss), it first checks with zookeeper to find which shard to query from.

- We can always keep multiple copies of sharded DB to increase availability by having a primary-secondary setup. 


## What aggregators do
- Lets say, they aggregate the hourly data coming from analytics log to dialy data. Get the frequency of search terms. And stores them in a no sql db.(schema can be refered in  notes)

- We can add functionalities in aggregators like drop terms with freq less than threshold. To skip irrelevant queries to get process.

- We can configure it to assign weights instead of freq to terms based on some ranking system(could be based on time if we need to suggest recently searched terms)


## What workers do
- Can be scheduled to run say every 30 mins/1 day to update the tri DB
- Each worker can pick a shard to dump the data after processing(shard info can be accessed from zookeeper)


## Further optimizations
- cache data at CDN level for faster queries based on geographies
- provide search terms for popular searched along with the suggested ones and get them cached at browser so that req don't come to server everytime
