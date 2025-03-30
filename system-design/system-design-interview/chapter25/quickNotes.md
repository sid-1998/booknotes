Refer this, its better written.
http://codesmith.io/blog/diagramming-system-design-s3-storage-system


### Core services 
- data service(stores object and returns object Id) 
- metaDataService(stores metaData info of the object stored)

### Data Model
For metaData DB
- Bucket table(has bucket id)
- Object table(has object Id, object name, bucketId as foreign key)


#### To create a bucket

- The client sends an HTTP PUT request to create a bucket.
- The client’s request reaches the API service, which calls identity and access management (IAM) for authentication and authorization.
- A bucket, as we mentioned, is merely metadata. On successful auth, the API service calls the metadata service to insert a row in a dedicated buckets table in the metadata database.
- The API service returns a success message to the client.


#### To upload an object

- With the bucket created, the client sends an HTTP PUT request to store an object in the bucket.
- Again, the API service calls IAM to perform authentication and authorization. We must check if the user has write permission on the bucket.
- After auth, the API service forwards the client’s payload to the data service, which persists the payload as an object and returns its id to the API service.
- The API service next calls the metadata service to insert a new row in a dedicated objects table in the metadata database. This row holds the object id, its name, and its bucket_id, among other details.
- The API service returns a success message to the client.

#### To download an object

- With the bucket created and the object uploaded, the client sends an HTTP GET request, specifying a name, to retrieve the object.

- The API service calls IAM to perform authentication and authorization. We must check that the client has read access to the bucket.

- Since the client typically sends in the object name (not its id), the API service must first map the object name to its id before it can retrieve the object. Hence, on successful auth, the API service calls the metadata service to query the objects table to locate the object id from its name. Note how, in the download flow, we visit the metadata service before the data service, as opposed to the upload flow.

- Having found the object id, the API service sends it to the data service, which responds with the object’s data.

- Finally, the API service returns the requested object to the client.

#### How is data exactly stored and managed
- Data is actually stored on disks(HDD/SSD)
- internally data service will have storage nodes running which interact with disks.
- We will have replication in place for the storage nodes to ensure high availability.
- In disk, we use blocks to store the data, each block has a size of 4kb. 
- in S3 we store object as a whole so files smaller than 4KB are stored in one block. Files more than that are split into chunks
- This cause space wastage as files smaller than a block size occupy the entire block, but thats how S3 choose to function
- In traditional HDFS, data is written in a Write Ahead log (WAL) fashion, where if space is left in a block. A separate object cna be written there. This optimizes space but adds complexity
- In both the cases we need an internal DB to store the locations of the chunks of objects against the unique object ID, which will help us in retrieving the object in case of a GET call


