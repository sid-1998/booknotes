## Functional requirements
- Upload
- Stream

We majorly discussed this, other features can be search, recommendation
## Non Funtional requirements
- Availability
- Low latency
- Scalability

## HLD
![img.png](img.png)

## Upload
- Use S3 type storage to stores video data. Rest of metadata goes to noSql db. noSQL scales better as read wil be heavy. And no need to have heavy consistency.
- Encoding of videos are time-consuming so they can be done real time. So we add a message queue to process the raw videos.
- Encoding servers picks the raw videos based on queue and process them and stores them in a separate S3. After processing they update the metatdata in cache and DB. The encoded videos are distributed over CDNs
- Encoding consist of converting videos in different formats(to serve different clients) and resolutions to handle latencies(if user net connection is poor we start serving a lower resolution video)
- **indepth detail of encoders is in booknotes. Avoid discussing in detail as it requires a lot of domain knowledge regarding encoders**

**Question: How many encoders we need?**
- assuming 50mill/day upload traffic
- assuming 1 video takes 1 min to upload on an average
- 50 mill/100,000 = 500/sec (assuming we have 100,000 second in day for rough calculation)
- Work to be done = 500*60sec = 30k workers will be needed

**Question: what type of protocol is used to upload the video to server?**

- Client use **resumable HTTP uploads protocol** to upload the video to server. 
- Resumable HTTP uploads protocol is an application-layer mechanism that allows uploading files in chunks and resuming if interrupted

## Stream
- Videos are cached at CDN to reduces latency and are served from there
- In case of CDN miss. req goes to app server. and app server fetch metadata and video url from cache/DB and gets video from S3 based on url stored.  The video gets cached at CDN and served to user
- In real world app, videos are served in chunks to reduce latency. **YouTube uses DASH (Dynamic Adaptive Streaming over HTTP)**, which breaks videos into small chunks (~2-10 seconds each)
- these chunks are cached at CDN level. and gets downloaded on browser when user requests them.
- CDNs are expensive so we try to cache only popular videos. As reads on popular ones will be more
- Video stored as chunks helps to control resolution incase user have poor connection(we can start serving rest of the chunks in lower resolution to reduce buffering)

**Question: What protocol should be used for streaming?**

In On demand streaming(Youtube, netflix, prime)
- Reliability is more important than speed(You don't want frames getting dropped while watching a movie and missing the plot twist. Huh!!). So we use TCP
- YouTube/Netflix use DASH (Dynamic Adaptive Streaming over HTTP), which works on top of TCP.
- Content Delivery Networks (CDNs) Use HTTP/HTTPS, Which Runs on TCP
 