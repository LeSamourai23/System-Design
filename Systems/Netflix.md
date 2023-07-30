## Requirements

#### Functional Requirements
* Streaming Movies and TV Shows in the highest quality possible.
* Content team should be able to upload videos.
* Searchable content with name and tags.

#### Non-Functional Requirements
* High availability with minimal latency.
* High reliability so the videos don't get lost.
* Scalable and efficient system

#### Extended Requirements
* Certain content should be geo-blocked.
* Resume video playback from the point user left off.
* Record metrics and analytics of videos.


## Estimation and Constraints

#### Traffic
This is a **Read-Heavy system**. We will take our Daily Active Users (DAU) as 200 million. Let's say each user watches on an average 5 videos per day

	200 million x 5 videos = 1 billion per day

Now, videos uploaded per day on our system would range from 30-50 as during a month's end more content is uploaded. Let's take the max.

RPS of the system will be

	1 billion / (24 x 60 x 60) = ~12K requests per second

#### Storage
Since, we are a high quality content streaming service, videos will have high bitrate than a video sharing platform like YouTube. We'll take a video size to be 2 GB on average as movies will range from 7-10GBs and an episode of Sit-Com will be 700MB to a GB.

	50 videos x 3 GB = 150GB per day

Since, content license keep expiring, videos will keep deleting from our platform but we need to maintain a certain amount of minimum content on our platform to attract users. Let's assume that to be 50K videos

	100000 Videos x 3 GB =  300 TB

So, we need to maintain 300 TBs worth of content.

#### Bandwidth
We are taking 150GBs of ingress everyday, so the minimum required bandwidth would be

	150 GBs / (24 x 3600) = ~2 MBs per second

#### High Level Estimate
| Type                      | Estimate    |
| ------------------------- | ----------- |
| Daily active users (DAU)  | 200 million |
| Requests per second (RPS) | ~12K/s      |
| Storage (per day)         | 150 GB      |
| Storage for total content | 300 TB      |
| Bandwidth                 | ~2 MB/s     |


## Data Model
![[Netflix Data Model.png]]

#### What kind of database should we use?
For scalability, we will divide the different types of data in different databases. We can use a relational database and a distributed NoSQL database for our use case.


## API Design

#### Upload a Video
```tsx

uploadVideo(title: string, director: string, starring: string[], description:string, data: Stream<byte>, tags?: string[]): boolean
```
Data (`Byte[]`): Byte stream of the video data.
Tags (`string[]`): Tags for the video.

**Returns**: Result (`boolean`): Represents whether the operation was successful or not.

#### Streaming a video
```tsx

streamVideo(videoID: UUID, codec: Enum<string>, resolution: Tuple<int>, offset?: int): VideoStream
```
Codec (`Enum<string>`): Required [codec](https://en.wikipedia.org/wiki/Video_codec) of the requested video, such as `h.265`, `h.264`, `VP9`, etc.
Resolution (`Tuple<int>`): [Resolution](https://en.wikipedia.org/wiki/Display_resolution) of the requested video.
Offset (`int`): Offset of the video stream in seconds to stream data from any point in the video.

**Returns**: Stream (`VideoStream`): Data stream of the requested video.

#### Search for a video
```tsx
searchVideo(query: string, nextPage?: string): Video[]
```
Query (`string`): Search query from the user.
Next Page (`string`): Token for the next page, this can be used for pagination.

**Returns**: Videos (`Video[]`): All the videos available for a particular search query.


## High Level Design

### Architecture
We will be using  **[Microservices Architecture](/Dictionary/Architecture/Microservices)** since it will be easier to horizontally scale and decouple.  Each service will have ownership of its own data model. Let's try to divide our system into some core services.

##### User Service
The service handles user concerns such as authentication and user information.

##### Stream Service
This service will handle the video streaming functionality

##### Search Service
This service handle the searching of videos functionality of our system.

##### Media Service
This service will handle the video uploads and processing.

##### Analytics Service
This service will handle the metrics and analytics.

###### Inter-Service Communication and Service Discovery
Since our architecture is microservices-based, services will be communicating with each other as well. Generally, **REST** or HTTP performs well but we can further improve the performance using **[[gRPC]]** which is more lightweight and efficient.

**[[Service discovery]]** is another thing we will have to take into account. We can also use a service mesh that enables managed, observable, and secure communication between individual services.

### Video Processing
![[video-processing-pipeline.png]]
Since Big Budget Content is shot on high quality cameras which are typically 4K and sometimes even 8K, they have a typical output of more than a TB. So, we need some kind of video processing to reduce storage and delivery costs without sacrificing on the streaming quality.

Once the videos are uploaded by the content team, it is queued up for processing in our **[Message Queue]**.

##### Step 1: File Chunker
![[file-chunking.png]]
This step checks if the video adheres to the content policy of the platform. This can be pre-approved as in the case of Netflix according to Content Rating (like MPAA) of the media.

This entire process is done by a machine learning model which performs copyright and NSFW checks. If issues are found, we can push the task to a **dead-letter queue (DLQ)** and someone from the moderation team can do further inspection.

##### Step 2: Transcoder
[Transcoding](https://en.wikipedia.org/wiki/Transcoding) is a process in which the original data is decoded to an intermediate uncompressed format, which is then encoded into the target format. This process uses different [codecs](https://en.wikipedia.org/wiki/Video_codec) to perform bitrate adjustment, image downsampling, or re-encoding the media.

This results in a smaller size file and a much more optimized format for the target devices. Standalone solutions such as [FFmpeg](https://ffmpeg.org) or cloud-based solutions like [AWS Elemental MediaConvert](https://aws.amazon.com/mediaconvert) can be used to implement this step of the pipeline.

##### Step 3: Quality Conversion
This is the last step of the processing pipeline and as the name suggests, this step handles the conversion of the transcoded media from the previous step into different resolutions such as 4K, 1440p, 1080p, 720p, etc.

It allows us to fetch the desired quality of the video as per the user's request, and once the media file finishes processing, it gets uploaded to a distributed **File Storage** such as [HDFS](https://karanpratapsingh.com/courses/system-design/storage#hdfs), [GlusterFS](https://www.gluster.org), or an **[[Object Storage]]** such as [Amazon S3](https://aws.amazon.com/s3) for later retrieval during streaming.

_Note: We can add additional steps such as subtitles and thumbnails generation as part of our pipeline._

###### Why are we using a message queue?
Processing videos as a long-running task and using a **[[Message Queue]]** makes much more sense. It also decouples our video processing pipeline from the upload functionality. We can use something like [Amazon SQS](https://aws.amazon.com/sqs) or [RabbitMQ](https://www.rabbitmq.com) to support this.

### Video Streaming
Video streaming is a challenging task from both the client and server perspectives. Moreover, internet connection speeds vary quite a lot between different users. To make sure users don't re-fetch the same content, we can use a **Content Delivery Network (CDN)**.

Netflix takes this a step further with its **[[Open Connect]]** program. In this approach, they partner with thousands of Internet Service Providers (ISPs) to localize their traffic and deliver their content more efficiently.

### Searching
Sometimes traditional DBMS are not performant enough, we need something which allows us to store, search, and analyze huge volumes of data quickly and in near real-time and give results within milliseconds. [Elasticsearch](https://www.elastic.co) can help us with this use case.

[Elasticsearch](https://www.elastic.co) is a distributed, free and open search and analytics engine for all types of data, including textual, numerical, geospatial, structured, and unstructured. It is built on top of [Apache Lucene](https://lucene.apache.org).

##### How do we identify trending content?
Trending functionality will be based on top of the search functionality. We can cache the most frequently searched queries in the last `N` seconds and update them every `M` seconds using some sort of batch job mechanism.

### Sharing
Sharing content is an important part of any platform, for this, we can have some sort of [**URL Shortener Service**](obsidian://open?vault=System%20Design&file=System%20Diagrams%2FTinyURL%20Design) in place that can generate short URLs for the users to share.


## Detailed Design

### Data Partitioning
To scale out our databases we will need to partition our data. 
Horizontal partitioning (**[[Database Sharding]]**) can be a good first step. We can use partitions schemes such as:

- Hash-Based Partitioning
- List-Based Partitioning
- Range Based Partitioning
- Composite Partitioning

The above approaches can still cause uneven data and load distribution, we can solve this using **[[Consistent Hashing]]**.

### Geo-blocking
Platforms like Netflix and YouTube use [Geo-blocking](https://en.wikipedia.org/wiki/Geo-blocking) to restrict content in certain geographical areas or countries. This is primarily done due to legal distribution laws that Netflix has to adhere to when they make a deal with the production and distribution companies.

We can determine the user's location either using their **IP** or region settings in their profile then use services like [Amazon CloudFront](https://aws.amazon.com/cloudfront) which supports a geographic restrictions feature or a [geolocation routing policy](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geo.html) with [Amazon Route53](https://aws.amazon.com/route53) to restrict the content and re-route the user to an error page if the content is not available in that particular region or country.

### Recommendations
Netflix uses a machine learning model which uses the user's viewing history to predict what the user might like to watch next, an algorithm like **[[Collaborative Filtering]]** can be used.

However, Netflix (like YouTube) uses its own algorithm called Netflix Recommendation Engine which can track several data points such as:

- User profile information like age, gender, and location.
- Browsing and scrolling behavior of the user.
- Time and date a user watched a title.
- The device which was used to stream the content.
- The number of searches and what terms were searched.

_For more detail, refer to [Netflix recommendation research](https://research.netflix.com/research-area/recommendations)._

### Metrics and Analytics
Recording analytics and metrics is one of our extended requirements. We can capture the data from different services and run analytics on the data using [Apache Spark](https://spark.apache.org) which is an open-source unified analytics engine for large-scale data processing. Additionally, we can store critical metadata in the views table to increase data points within our data.

### Caching
In a streaming platform, caching is important. We have to be able to cache as much static media content as possible to improve user experience. We can use solutions like [Redis](https://redis.io) or [Memcached](https://memcached.org)

**[[Least Recently Used (LRU)]]** can be a good policy for our system.

##### How to handle cache miss?
Whenever there is a cache miss, our servers can hit the database directly and update the cache with the new entries.

### Media streaming and storage
As most of our storage space will be used for storing media files such as thumbnails and videos. Per our discussion earlier, the media service will be handling both the upload and processing of media files.

We will use distributed file storage such as [HDFS](https://karanpratapsingh.com/courses/system-design/storage#hdfs), [GlusterFS](https://www.gluster.org), or an **[[Object Storage]]** such as [Amazon S3](https://aws.amazon.com/s3) for storage and streaming of the content.

### Content Delivery Network (CDN)
**[[Content Delivery Network (CDN)]]** increases content availability and redundancy while reducing bandwidth costs. Generally, static files such as images, and videos are served from CDN. We can use services like [Amazon CloudFront](https://aws.amazon.com/cloudfront) or [Cloudflare CDN](https://www.cloudflare.com/cdn) for this use case.


## Identify and Resolve Bottlenecks

##### Bottlenecks:
- "What if one of our services crashes?"
- "How will we distribute our traffic between our components?"
- "How can we reduce the load on our database?"
- "How to improve the availability of our cache?"

##### Solutions:
- Running multiple instances of each of our services.
- Introducing **[[Load Balancers]]** between clients, servers, databases, and cache servers.
- Using multiple read replicas for our databases.
- Multiple instances and replicas for our distributed cache.

## Final Diagram
The design is done using AWS Services icons.

**[[Netflix Design]]**
