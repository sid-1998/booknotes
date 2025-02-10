# High-Level Design (HLD) - News Feed System

## Functional Requirements
✅ Users can post content (text, images, videos, links).
✅ Users see friends' posts in their news feed.
✅ Feed is sorted in reverse chronological order (latest posts appear first).
✅ Supports up to 10 million DAUs.
✅ Supports media (images, videos).

## Non-Functional Requirements
- **Scalability:** Should handle millions of requests per second.
- **Low Latency:** Fetching the news feed should be fast.
- **High Availability:** No single point of failure.
- **Efficient Storage & Retrieval:** Posts should be quickly accessible.

---

## System Components

### 1. **Newsfeed API**
- Handles requests for publishing and retrieving news feeds.
- Endpoints:
    - `POST /v1/me/feed` - Publish a post.
    - `GET /v1/me/feed` - Retrieve news feed.

### 2. **Feed Publishing Flow**
- User publishes a post.
- **Load Balancer** distributes traffic.
- **Web Servers** handle authentication and rate limiting.
- **Post Service** stores the post in database and cache.
- **Fanout Service** pushes post to friends’ news feed.
- **Notification Service** informs friends of new content.

### 3. **News Feed Retrieval Flow**
- User requests their news feed.
- **Load Balancer** distributes traffic.
- **Web Servers** route to Newsfeed Service.
- **Newsfeed Service** fetches precomputed news feed from cache.
- **Post Hydration** - Post details, user info, media files are retrieved.

### 4. **Fanout Strategy**
- **Fanout on Write:** Precomputes feeds when a post is created.
- **Fanout on Read:** Fetches posts dynamically when user requests a feed.
- **Hybrid Approach:** Mix of both methods for optimal performance.

### 5. **Caching Strategy**
- **Redis Cache Layers:**
    - **News Feed Cache** - Stores precomputed feeds.
    - **Content Cache** - Stores popular posts.
    - **Social Graph Cache** - Stores user relationships.
    - **Action Cache** - Stores likes, replies.
    - **Counter Cache** - Stores counts for likes, replies, followers.

### 6. **Storage Architecture**
- **SQL Database:** Stores structured post metadata.
- **NoSQL Database:** Stores user-generated content efficiently.
- **Blob Storage:** Stores images/videos via a CDN.
- **Sharding:** Horizontal scaling to distribute data across multiple servers.

---

## **Conclusion**
This design ensures:
- **Scalability:** Supports 10M DAUs with sharded databases and caching.
- **Low Latency:** Precomputed feeds and caching enable fast retrieval.
- **High Availability:** No single point of failure.
- **Efficient Storage:** Proper DB choices and caching mechanisms.

This approach provides a solid foundation for a real-world, large-scale News Feed System. 🚀

