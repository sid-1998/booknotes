## Core problems in Cloud storage

- upload/Concurrency
- latency
- bandwidth
- History and storage


![img.png](img.png)

If we go ahead with a simple S3 and a service that uploads the file. Every time user want to edit we again consume the same bandwidth to upload. And storage space to maintain versions
This will not work for large data.

### How to solve this problem

Break the file into chunks instead of treating it as a whole.

- We can upload chunks in parallel, thus reducing latency and bandwidth usage.
- User can sync to latest version by just downloading the chunk that has been updated instead of downloading the entire file again
- reduce storage as we store updated chunks only whenever changes are made.
- can store metaData in a SQL db to maintain all this info(which version is lates, what all chunks are related to what version of file)

**Refer notes for block servers and metaData cache and DB**




