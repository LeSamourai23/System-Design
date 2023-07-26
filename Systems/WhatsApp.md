## Requirements
Our System should meet the following requirements:

#### Functional Requirements:
* One to one messaging
* Group Chats (let's say max 100 people in a group)
* File sharing (images, videos, documents, etc.)

#### Non-Functional Requirements:
* High Availability and Low Latency
* Scalable and efficient system

#### Extended Requirements:
* Sent, Delivered and Read receipts of the messages
* Last seen of users
* Push notifications


## Estimations and Constraints

#### Traffic
This is a **Write-Heavy System**. We will take our daily active users (DAU) as 100 million.
Let's assume a user sends on an average 10 messages to 5 different users a day

	100 million x 50 = 5 billion total messages per day

Now, we will assume that 5% of the total messages are media files

	5 billion x 0.05 = 250 million media files per day

So, the RPS of the system will be

	5 billion / (24 x 60 x 60) = ~60K RPS

#### Storage
For messages,

	5 billion x 1 KB = 500 GBs per day

For media,

	500 million x 500 KBs = 250 TBs per day

Total we need 250.5 TBs of storage every day.
Now, for 10 years we need,

	250.5 TBs x 365 = ~100 PBs 

#### Bandwidth
As our system will handle 250.5 TBs of ingress everyday, we will require a minimum bandwidth of around 120 MB per second

	250.5 TBs / (24 x 60 x 60) = ~3 GBs per second

#### High Level Estimate
| Type                      | Estimate    |
| ------------------------- | ----------- |
| Daily Active Users (DAU)  | 100 million |
| Requests per second (RPS) | 60K RPS     |
| Storage per day           | 250.5 TBs   |
| Storage after 10 years    | 100 PBs     |
| Bandwidth                 | 3 Gbps      |


## Data Model

![[WhatsApp Data Model.png]]

#### What kind of database should we use?
For scalability, we will divide the different types of data in different databases. We can use a relational database and a distributed NoSQL database for our use case.


## API Design
A basic API design for our service:

#### Get all chats or groups
```
getAll(userID: UUID): chat[] | Group[]
```

**Return**: Result (`Chat[] | Group[]`): All the chats and groups the user is a part of.

#### Get Messages
```
getMessages(userID: UUID, channelID): Messages[]
```
Channel ID (`UUID`): ID of the channel (chat or group) from which messages need to be retrieved.

**Returns**: Messages (`Message[]`): All the messages in a given chat or group.

#### Send Message
```
sendMessage(userID: UUID, channelID: Messages[], message: Messagee): boolean
```
Message (`Message`): The message (text, image, video, etc.) that the user wants to send.

**Returns**: Result (`boolean`): Represents whether the operation was successful or not.

#### Join or leave channel
```
joinGroup(userID: UUID, channelID:UUID): boolean
leaveGroup(userID: UUID, channelID: UUID): boolean
```
Channel ID (`UUID`): ID of the channel (chat or group) the user wants to join or leave.

**Returns**: Result (`boolean`): Represents whether the operation was successful or not.

## High-level design

### Architecture
We will be using **[Microservices Architecture](/Dictionary/Architecture/Microservices)** it will make it easier to horizontally scale and decouple our services. Each service will have ownership of its own data model. Let's try to divide our system into some core services.

##### User Service
HTTP based service that handles user-related concerns such as authentication and user info.

##### Chat Service
The chat service will use **[[WebSockets]]** and establish connections with the client to handle chat and group message-related functionality. We can also use cache to keep track of all the active connections sort of like sessions which will help us determine if the user is online or not.

##### Presence Service
The presence service will keep track of the _last seen_ status of all users. 

##### Media Service
This service will handle the media (images, videos, files, etc.) uploads.

##### Inter-Service Communication and Service Discovery
Since our architecture is microservices-based, services will be communicating with each other as well. Generally, **REST** or HTTP performs well but we can further improve the performance using **[[gRPC]]** which is more lightweight and efficient.

**[[Service discovery]]** is another thing we will have to take into account. We can also use a service mesh that enables managed, observable, and secure communication between individual services.

### Real-time messaging
How do we efficiently send and receive messages? We have two different options:

##### Pull model
The client can periodically send an HTTP request to servers to check if there are any new messages. This can be achieved via something like **[[Long polling]]**

##### Push model
The client opens a long-lived connection with the server and once new data is available it will be pushed to the client. We can use **[[WebSockets]]** or **Server-Sent Events (SSE)** for this.

The pull model approach is not scalable as it will create unnecessary request overhead on our servers and most of the time the response will be empty, thus wasting our resources. To minimize latency, using the push model with **[[WebSockets]]** is a better choice because then we can push data to the client once it's available without any delay, given the connection is open with the client. Also, **[[WebSockets]]** provide full-duplex communication, unlike **Server-Sent Events (SSE)** which are only unidirectional.

### Last Seen
To implement the last seen functionality, we can use a [heartbeat](https://en.wikipedia.org/wiki/Heartbeat_(computing)) mechanism, where the client can periodically ping the servers indicating its liveness. Since this needs to be as low overhead as possible, we can store the last active timestamp in the cache as follows:

|Key|Value|
|---|---|
|User A|2022-07-01T14:32:50|
|User B|2022-07-05T05:10:35|
|User C|2022-07-10T04:33:25|

This will give us the last time the user was active. This functionality will be handled by the presence service combined with [Redis](https://redis.io) or [Memcached](https://memcached.org) as our cache.

Another way to implement this is to track the latest action of the user, once the last activity crosses a certain threshold, such as _"user hasn't performed any action in the last 30 seconds"_, we can show the user as offline and last seen with the last recorded timestamp. This will be more of a lazy update approach and might benefit us over heartbeat in certain cases.

### Notifications
Once a message is sent in a chat or a group, we will first check if the recipient is active or not, we can get this information by taking the user's active connection and last seen into consideration.

If the recipient is not active, the chat service will add an event to a **[[Message Queues]]** with additional metadata such as the client's device platform which will be used to route the notification to the correct platform later on.

The notification service will then consume the event from the message queue and forward the request to [Firebase Cloud Messaging (FCM)](https://firebase.google.com/docs/cloud-messaging) or [Apple Push Notification Service (APNS)](https://developer.apple.com/documentation/usernotifications) based on the client's device platform (Android, iOS and web). We can also add support for email and SMS.

##### Why are we using Message Queue?
Since most **Message Queues** provide best-effort ordering which ensures that messages are generally delivered in the same order as they're sent and that a message is delivered at least once which is an important part of our service functionality.

While this seems like a classic **Pub-Sub Messaging** use case, it is actually not as mobile devices and browsers each have their own way of handling push notifications. Usually, notifications are handled externally via Firebase Cloud Messaging (FCM) or Apple Push Notification Service (APNS) unlike message fan-out which we commonly see in backend services. We can use something like [Amazon SQS](https://aws.amazon.com/sqs) or [RabbitMQ](https://www.rabbitmq.com) to support this functionality.

### Read receipts

Handling read receipts can be tricky, for this use case we can wait for some sort of [Acknowledgment (ACK)](https://en.wikipedia.org/wiki/Acknowledgement_(data_networks)) from the client to determine if the message was delivered and update the corresponding `deliveredAt` field. Similarly, we will mark the message as seen once the user opens the chat and update the corresponding `seenAt` timestamp field.


## Detailed Design

### Data Partitioning
To scale out our databases we will need to partition our data. 
Horizontal partitioning (**[[Database Sharding]]**) can be a good first step. We can use partitions schemes such as:

- Hash-Based Partitioning
- List-Based Partitioning
- Range Based Partitioning
- Composite Partitioning

The above approaches can still cause uneven data and load distribution, we can solve this using **[[Consistent Hashing]]**.

### Cache
In a messaging application, we have to be careful about using cache as our users expect the latest data, but many users will be requesting the same messages, especially in a group chat. So, to prevent usage spikes from our resources we can cache older messages.

Some group chats can have thousands of messages and sending that over the network will be really inefficient, to improve efficiency we can add pagination to our system APIs. This decision will be helpful for users with limited network bandwidth as they won't have to retrieve old messages unless requested.

##### Which cache eviction policy to use?
We can use solutions like [Redis](https://redis.io) or [Memcached](https://memcached.org) and cache 20% of the daily traffic but what kind of cache eviction policy would best fit our needs?

**[[Least Recently Used (LRU)]]** can be a good policy for our system. In this policy, we discard the least recently used key first.

##### How to handle cache miss?
Whenever there is a cache miss, our servers can hit the database directly and update the cache with the new entries.

### Media access and storage
As we know, most of our storage space will be used for storing media files such as images, videos, or other files. Our media service will be handling both access and storage of the user media files.

But where can we store files at scale? Well, **[[Object Storage]]** is what we're looking for. Object stores break data files up into pieces called objects. It then stores those objects in a single repository, which can be spread out across multiple networked systems. We can also use distributed file storage such as **HDFS** or [GlusterFS](https://www.gluster.org).

_Fun fact: WhatsApp deletes media on its servers once it has been downloaded by the user._

We can use object stores like [Amazon S3](https://aws.amazon.com/s3), [Azure Blob Storage](https://azure.microsoft.com/en-in/services/storage/blobs), or [Google Cloud Storage](https://cloud.google.com/storage) for this use case.

### Content Delivery Network (CDN)
**[[Content Delivery Network (CDN)]]** increases content availability and redundancy while reducing bandwidth costs. Generally, static files such as images, and videos are served from CDN. We can use services like [Amazon CloudFront](https://aws.amazon.com/cloudfront) or [Cloudflare CDN](https://www.cloudflare.com/cdn) for this use case.

### API gateway
Since we will be using multiple protocols like HTTP, WebSocket, TCP/IP, deploying multiple L4 (transport layer) or L7 (application layer) type load balancers separately for each protocol will be expensive. Instead, we can use an **[[API Gateway]]** that supports multiple protocols without any issues.

API Gateway can also offer other features such as authentication, authorization, rate limiting, throttling, and API versioning which will improve the quality of our services.

We can use services like [Amazon API Gateway](https://aws.amazon.com/api-gateway) or [Azure API Gateway](https://azure.microsoft.com/en-in/services/api-management) for this use case.


## Identify and Resolve Bottlenecks

##### Bottlenecks:
- "What if one of our services crashes?"
- "How will we distribute our traffic between our components?"
- "How can we reduce the load on our database?"
- "How to improve the availability of our cache?"
- "Wouldn't API Gateway be a single point of failure?"
- "How can we make our notification system more robust?"
- "How can we reduce media storage costs"?
- "Does chat service has too much responsibility?"

##### Solutions:
- Running multiple instances of each of our services.
- Introducing **[[Load Balancers]]** between clients, servers, databases, and cache servers.
- Using multiple read replicas for our databases.
- Multiple instances and replicas for our distributed cache.
- We can have a standby replica of our API Gateway.
- Exactly once delivery and message ordering is challenging in a distributed system, we can use a dedicated **[[Message Broker]]** such as [Apache Kafka](https://kafka.apache.org) or [NATS](https://nats.io) to make our notification system more robust.
- We can add media processing and compression capabilities to the media service to compress large files similar to WhatsApp which will save a lot of storage space and reduce cost.
- We can create a group service separate from the chat service to further decouple our services.


## Final Diagram
The design is done using AWS Services icons.

**[[WhatsApp Design]]**