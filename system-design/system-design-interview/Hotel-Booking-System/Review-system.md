## How to persist hotel review and rating and update search recommendations

- client send a post req for review having text, rating and hotelId.
- we persist it in DB and publish the event in kafka.
- a stream job picks it up which is aggregating the reviews per hotel to calculate mean rating. We keep count and sum for hotelId.
- we can do a partial update on ES to update the new rating based on some threshold like num_reviews>10, new mean - old mean > 0.2, instead of doing it at fixed intervals so that hot hotels can be refreshed earlier
- partial update in ES does a delete+add for a hotel doc and on refresh the index is updated with new ratings instead of re-building the whole index.
- when search orchestrator query for hotels with rating filter we have now accommodated the reviews and rating to get updated results.


## Flow

Review POST -> DB -> kafka -> stream job -> ES update