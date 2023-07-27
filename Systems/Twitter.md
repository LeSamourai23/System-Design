## Requirements

#### Functional Requirements
* Post Tweet
* Follow other users
* Search Tweets
* Newsfeed

#### Non-Functional Requirements
* High availability and low latency
* System should be scalable and efficient

#### Extended Requirements
* Retweeting
* Bookmarked Tweets


## Estimation and Constraints

#### Traffic
This is a **Read-Heavy System**. We will take out daily active users (DAU) as 100 million and on an average each user tweets 5 times a day. 

	100 million x 5 tweets = 500 million tweets per day

Now, for media files, we'll take 5% of the tweets as attached with a certain type of media.

	500 million x 5% = 25 million tweets per dat

RPS of the system will be

	500 million / (24 x 60 x 60) =~ 6K RPS

#### Storage
We will take the size of the tweet as 100 bytes.

	500 million x 100 bytes = 50 GB per day

The size of the tweets attached with media (when average media size after compression taken as a 100 KBs)

	25 million x 100 KBs = 2.5 TB per day

For a scalable system, we'll have to see how much storage will be used after 10 years. so,

	2.55 TB x 30 x 12 x 10 =~ 10 PB in 10 years

#### Bandwidth
We are taking 2.55 TBs of Ingress everyday, that mean we need a bandwidth of 

	2.55 TB / (24 x 60 x 60) =~  30 MB/sec

#### High-level estimate

|Type|Estimate|
|---|---|
|Daily active users (DAU)|100 million|
|Requests per second (RPS)|6K/s|
|Storage (per day)|~2.55 TB|
|Storage (10 years)|~10 PB|
|Bandwidth|~30 MB/s|


## Data Model 

![[Twitter Data Model.png]]

#### What kind of database should we use?
For scalability, we will divide the different types of data in different databases. We can use a relational database and a distributed NoSQL database for our use case.


## API Design

#### Post a Tweet
```tsx
postTweet(userID: UUID, content: string, mediaURL?: string):boolean
```

**Returns**: Result (`boolean`): Represents whether the operation was successful or not.

#### Follow/ Unfollow User
```tsx
follow(userID: UUID, followeeID: UUID): boolean
unfollow(userID: UUID, followeeID: UUID): boolean
```

**Returns**: Result (`boolean`): Represents whether the operation was successful or not.

#### Get Newsfeed
```tsx
getNewsfeed(userIDL UUID): Tweet[]
```

**Returns**: Tweets(Tweet[]): All the tweets to be shown in the newsfeed


## High Level Design

### Architecture
We will be using  **[Microservices Architecture](/Dictionary/Architecture/Microservices)** since it will be easier to horizontally scale.  Each service will have ownership of its own data model. Let's try to divide our system into some core services.

##### User Service
This service handles user-related concerns such as authentication and user information.

##### Newsfeed Service
This service will handle the generation and publishing of user newsfeeds.

##### Tweet Service
The tweet service will handle tweet-related use cases such as posting a tweet, bookmarks, etc.

##### Search Service
The service is responsible for handling search-related functionality.

##### Media Service
The service will handle the media uploads.

##### Notification Service
This service will send push notifications to users.

##### Inter-Service Communication and Service Discovery
Since our architecture is microservices-based, services will be communicating with each other as well. Generally, **REST** or HTTP performs well but we can further improve the performance using **[[gRPC]]** which is more lightweight and efficient.

**[[Service discovery]]** is another thing we will have to take into account. We can also use a service mesh that enables managed, observable, and secure communication between individual services.

### Newsfeed
When it comes to the newsfeed, it seems easy enough to implement, but there are a lot of things that can make or break this feature. There are two major problems:

##### Generation
Let's assume we want to generate the feed for user A, we will perform the following steps:

1. Retrieve the IDs of all the users and entities (hashtags, topics, etc.) user A follows.
2. Fetch the relevant tweets for each of the retrieved IDs.
3. Use a **ranking algorithm** to rank the tweets based on parameters such as relevance, time, engagement, etc.
4. Return the ranked tweets data to the client in a paginated manner.

Feed generation is an intensive process and can take quite a lot of time, especially for users following a lot of people. To improve the performance, the feed can be pre-generated and stored in the cache, then we can have a mechanism to periodically update the feed and apply our ranking algorithm to the new tweets.

##### Publishing
Publishing is the step where the feed data is pushed according to each specific user. This can be a quite heavy operation, as a user may have millions of friends or followers. To deal with this, we have three different approaches:

1. **[Pull Model](https://raw.githubusercontent.com/karanpratapsingh/portfolio/master/public/static/courses/system-design/chapter-V/twitter/newsfeed-pull-model.png)**: When a user creates a tweet, and a follower reloads their newsfeed, the feed is created and stored in memory. The most recent feed is only loaded when the user requests it. This approach reduces the number of write operations on our database. The downside of this approach is that the users will not be able to view recent feeds unless they "pull" the data from the server, which will increase the number of read operations on the server.
2. **[Push Model](https://raw.githubusercontent.com/karanpratapsingh/portfolio/master/public/static/courses/system-design/chapter-V/twitter/newsfeed-push-model.png)**: In this model, once a user creates a tweet, it is "pushed" to all the follower's feeds immediately. This prevents the system from having to go through a user's entire followers list to check for updates. However, the downside of this approach is that it would increase the number of write operations on the database.
3. **Hybrid Model**: A third approach is a hybrid model between the pull and push model. It combines the beneficial features of the above two models and tries to provide a balanced approach between the two. The hybrid model allows only users with a lesser number of followers to use the push model. For users with a higher number of followers such as celebrities, the pull model is used.

### Ranking Algorithm
We'll be using **[[EdgeRank Algorithm]]** for our use. 

### Retweets
To implement this feature, we can simply create a new tweet with the user id of the user retweeting the original tweet and then modify the `type` enum and `content` property of the new tweet to link it with the original tweet.

For example, the `type` enum property can be of type tweet, similar to text, video, etc and `content` can be the id of the original tweet. Here the first row indicates the original tweet while the second row is how we can represent a retweet.

| id                  | userID              | type  | content                      | createdAt     |
| ------------------- | ------------------- | ----- | ---------------------------- | ------------- |
| ad34-291a-45f6-b36c | 7a2c-62c4-4dc8-b1bb | text  | Hey, this is my first tweet… | 1658905644054 |
| f064-49ad-9aa2-84a6 | 6aa2-2bc9-4331-879f | tweet | ad34-291a-45f6-b36c          | 1658906165427 |

### Search
Sometimes traditional DBMS are not performant enough, we need something which allows us to store, search, and analyze huge volumes of data quickly and in near real-time and give results within milliseconds. [Elasticsearch](https://www.elastic.co) can help us with this use case.

##### How do we identify trending topics?
Trending functionality will be based on top of the search functionality. We can cache the most frequently searched queries, hashtags, and topics in the last `N` seconds and update them every `M` seconds using some sort of batch job mechanism. Our ranking algorithm can also be applied to the trending topics to give them more weight and personalize them for the user.

### Notifications
Push notifications are an integral part of any social media platform. We can use a **[[Message Queue]]** such as [Apache Kafka](https://kafka.apache.org) with the notification service to dispatch requests to [Firebase Cloud Messaging (FCM)](https://firebase.google.com/docs/cloud-messaging) or [Apple Push Notification Service (APNS)](https://developer.apple.com/documentation/usernotifications) which will handle the delivery of the push notifications to user devices.


## Detailed Design

### Data Partitioning
To scale out our databases we will need to partition our data. 
Horizontal partitioning (**[[Database Sharding]]**) can be a good first step. We can use partitions schemes such as:

- Hash-Based Partitioning
- List-Based Partitioning
- Range Based Partitioning
- Composite Partitioning

The above approaches can still cause uneven data and load distribution, we can solve this using **[[Consistent Hashing]]**.

### Mutual Followers
For mutual friends, we can build a social graph for every user. Each node in the graph will represent a user and a directional edge will represent followers and followees. After that, we can traverse the followers of a user to find and suggest a mutual friend. This would require a graph database such as [Neo4j](https://neo4j.com) and [ArangoDB](https://www.arangodb.com).

This is a pretty simple algorithm, to improve our suggestion accuracy, we will need to incorporate a recommendation model which uses machine learning as part of our algorithm.

### Caching
In a social media application, we have to be careful about using cache as our users expect the latest data. So, to prevent usage spikes from our resources we can cache the top 20% of the tweets.

To further improve efficiency we can add pagination to our system APIs. This decision will be helpful for users with limited network bandwidth as they won't have to retrieve old messages unless requested.

##### Which cache eviction policy to use?
We can use solutions like [Redis](https://redis.io) or [Memcached](https://memcached.org) and cache 20% of the daily traffic but what kind of cache eviction policy would best fit our needs?

**[[Least Recently Used (LRU)]]** can be a good policy for our system.

##### How to handle cache miss?
Whenever there is a cache miss, our servers can hit the database directly and update the cache with the new entries.

## Media Access and Storage
As we know, most of our storage space will be used for storing media files such as images, videos, or other files. Our media service will be handling both access and storage of the user media files.

But where can we store files at scale? Well, **[[Object Storage]]** is what we're looking for. Object stores break data files up into pieces called objects. It then stores those objects in a single repository, which can be spread out across multiple networked systems. We can also use distributed file storage such as **HDFS** or [GlusterFS](https://www.gluster.org).

### Content Delivery Network (CDN)
**[[Content Delivery Network (CDN)]]** increases content availability and redundancy while reducing bandwidth costs. Generally, static files such as images, and videos are served from CDN. We can use services like [Amazon CloudFront](https://aws.amazon.com/cloudfront) or [Cloudflare CDN](https://www.cloudflare.com/cdn) for this use case.


## Identify and Resolve Bottlenecks

##### Bottlenecks:
- "What if one of our services crashes?"
- "How will we distribute our traffic between our components?"
- "How can we reduce the load on our database?"
- "How to improve the availability of our cache?"
- "How can we make our notification system more robust?"
- "How can we reduce media storage costs"?

##### Solutions:
- Running multiple instances of each of our services.
- Introducing **[[Load Balancers]]** between clients, servers, databases, and cache servers.
- Using multiple read replicas for our databases.
- Multiple instances and replicas for our distributed cache.
- We can have a standby replica of our API Gateway.
- Exactly once delivery and message ordering is challenging in a distributed system, we can use a dedicated **[[Message Broker]]** such as [Apache Kafka](https://kafka.apache.org) or [NATS](https://nats.io) to make our notification system more robust.
* We can add media processing and compression capabilities to the media service to compress large files which will save a lot of storage space and reduce cost.

## Final Diagram
The design is done using AWS Services icons.

**[[Twitter Design]]**