## Requirements

- Politeness/crawl rate
- DNS query
- Distributed crawling
- Priority crawling
- Duplicate detection

## Back of the envelope estimation
Given 1 billion pages per month -> ~400 pages per second
Peak QPS = 800 pages per second

Given average web page size is 500kb -> 500 TB per month -> 30 PB for 5y.

# Step 2 - Propose high-level design and get buy-in
![img.png](images/img.png)

- **Seed urls** - urls to start with. Popular urls that are feed initially to the frontier
- **Url Frontier** - FIFO queue. Handles priority, politeness. Gives next url to crawl
- **Fetcher + Renderer** - Impl using threads for concurrency. We can add more workers to scale this part. Each worker has fetcher and Render. Fetcher gets content of url, using curl(GET). Whenever a thread is free, it asks frontier to give url, gets IP of url and fetch page from internet. Earlier we had HTML content on pages which can be parsed directly, now days we have SPAs where lot of content is downloaded dynamically, so we need a Renderer for this. Fetcher also store the content in DB and cache too for quick processing. We cant query DB every time to get the content to be processed. Duh!! 
- **DNS Resolver** - We need to resolve each and every URL, so we keep a cache to reduce latency. We can update the DNS cache periodically via a cron job
- **Response Queue** - Once content is fetches, entry is made in this queue with ref to the stored content in cache. Each Processor receives a copy of this message, so they can work in parallel to do the processing
![img.png](images/img-1.png)

Processors which performs task that we want, url extractor is default one which we need as we need to get more urls to keep on crawling, we can have more processors like Duplicate Detection. This part of design is extensible based on the use cases we have.
- **URL extractor** - extract urls from content so that we can explore more pages. Checks for valid urls, remove blacklisted onces, do processing on urls like clubbing urls of same domains, remove urls with permalinks as they are actually the same urls that are processed.
- **URL Filter** - filters urls based on usecase. If we want mp3 files, ignore all urls that are not mp3. it checks mimetype and regex pattern of url for filtering.
- **Url Loader** - check if already crawled or not. We use Bloom filters to check if already crawled or not. (Read about Bloom filter if you forgot dummy). The loader can also support a feature of storing all the urls crawled for domain in Distributed Files system. Say we have a file google.com which contain all the urls of google.com domain that we have crawled. The unique urls are given to Frontier


![img_1.png](images/img_1.png)
## URL Frontier in depth
- Gives url to threads
- handle priority
- handle politeness (need to add delay in subsequent requests to avoid overloading the server)
- handle freshness (sites which updates frequently have high priority(eg: NEWS site))

### Frontier Components
- **Front queues**: stores url to be processed. we assign a priority to each queue, and based on the priority of the url it goes to the appropriate queue. If we have priorities 1 to 100, we will have 100 front queues.
- **Prioritizer** - decides priority, based on the historical data we have about the url, (freq of updates on site, etc). We prioritize URLs by usefulness, which can be determined based on PageRank web traffic, update frequency, etc.
- **Back Queue Router** - makes sure no **Back queue** is empty. Number of back queues is equal to number of workers in fetcher+render setup. 
- **Back queue** - a queue is mapped with website, to ensure urls of same host are in one queue so that we can regulate the flow of requests to that website(**Politeness**). We use a map to store `hostname -> queue number` to send url to appropriate queue.
- **BackQueue Selector** - Each worker thread is mapped to a FIFO queue and only downloads URLs from that queue. Queue selector chooses which worker processes which queue.

Ignore heap here, too much info. (Sad cry!)

#### How to check for updates in website?
Do HEAD request to get last modified dateTime of webpage, compare it with the time it was last crawled

#### How to check for duplicate content?
- BruteForce - match each and every word
- Hashing - Compare hashes of the page (works if content is completely identical)
- Algorithms like(minHash, SimHash, fuzzy search etc) calculates how similar the contents are. By setting a threshold of similarity we can filter duplicates.

## Storage
**where to store the content of pages.** 

* We have a lot of data here. SQL does not make sense has we don't have any relationships between content

* Can use NOSql but will be expensive as we have a lot of data. 

* S3 is better choice