## Requirements

#### Functional Requirements
###### Customers
* Book a ride from anywhere.
* Cab tracking
* ETA, Pricing information and nearby cabs in the area.
###### Drivers
* See nearby customers.
* Accept rides.
* Mark the trip complete after the completion of the trip.

#### Non-Functional Requirements
* High Reliability
* High availability with low latency 
* Scalable and efficient system

#### Extended Requirements
* Trip ratings
* Payment processing
* Metrics and Analytics


## Estimation and Constraints

#### Traffic
Let's take our Daily Active Users (DAU) as 50 million and 1 million of those are drives. On daily our platform does 10 million drives daily.
If on average each user performs 10 actions we will have to handle 

	50 million x 10 = 500 million actions / day

RPS of our system is

	500 million / (24x60x60) = ~6K requests

#### Storage
Let's say each message on average requires 400 bytes, we will require

	500 million x 400 bytes =~ 200 GB/day

For 10 ears, we will require

	200 GB x 10 x 365 =~ 0.7 PB

#### Bandwidth
As our system will take 200 GB ingress everyday, the minimum requires bandwidth will be

	200 GB / (24 x 60 x 60) = ~3 Mbps

#### High-level Estimate

|Type|Estimate|
|---|---|
|Daily active users (DAU)|50 million|
|Requests per second (RPS)|6K/s|
|Storage (per day)|~200 GB|
|Storage (10 years)|~0.7 PB|
|Bandwidth|~3 MB/s|


## Data Model

![[Uber Data Model.png]]

#### What kind of database should we use?
While our data model seems quite relational, we don't necessarily need to store everything in a single database, as this can limit our scalability and quickly become a bottleneck.
We will split the data between different services each having ownership over a particular table. Then we can use a relational database such as [PostgreSQL](https://www.postgresql.org) or a distributed NoSQL database such as [Apache Cassandra](https://cassandra.apache.org/_/index.html) for our use case.


## API Design

#### Request a ride (Customer)
```tsx

requestRide(customerID: UUID, source: Tuple<float>, destination: Tuple<float>, cabType: Enum<string>): Ride
```
Source (`Tuple<float>`): Tuple containing the latitude and longitude of the trip's starting location.
Destination (`Tuple<float>`): Tuple containing the latitude and longitude of the trip's destination.

**Returns**: Result (`Ride`): Associated ride information of the trip.

#### Cancel the Ride (Customer)
```tsx
cancelRide(customerID: UUID, reason: string): boolean
```

**Returns**: Result (`boolean`): Represents whether the operation was successful or not.

#### Accept/Deny Ride (Driver)
```tsx
acceptRide(driverID: UUID, rideID: UUID): boolean
denyRide(driverID: UUID, rideID: UUID): boolean
```

**Returns**: Result (`boolean`): Represents whether the operation was successful or not.

#### Start or End Trip (Driver)
```tsx
startTrip(driverID: UUID, tripID: UUID): boolean
endTrip(driverID: UUID, tripID: UUID): boolean
```

**Returns**: Result (`boolean`): Represents whether the operation was successful or not.

#### Rate the Trip(Customer/ Driver)
```tsx

rateTrip_customer(customerID: UUID, tripID: UUID, rating: int, feedback?:string): boolean
rateTrip_driver:(customerID: UUID, tripID: UUID, rating: int, feedback?:string): boolean
```

**Returns**: Result (`boolean`): Represents whether the operation was successful or not.


## High Level Design

### Architecture
We will be using  **[Microservices Architecture](/Dictionary/Architecture/Microservices)** since it will be easier to horizontally scale.  Each service will have ownership of its own data model. Let's try to divide our system into some core services.

##### Customer Service
This service will handle customer related concerns such as authentication and customer information.

##### Driver Service
This Service will handle driver related concerns such as authentication and driver information.

##### Ride Service
This service will be responsible for ride matching and quadtree aggregation.

##### Trip Service
This service will handle trip related functionality of our system.

##### Payment Service
This service will be responsible for handling payments in our system.

##### Notification Service
Service responsible for sending push notifications to the user.

##### Analytics Service
This service will be used for metrics and analytics use case.

###### Inter-Service Communication and Service Discovery
Since our architecture is microservices-based, services will be communicating with each other as well. Generally, **REST** or HTTP performs well but we can further improve the performance using **[[gRPC]]** which is more lightweight and efficient.

**[[Service discovery]]** is another thing we will have to take into account. We can also use a service mesh that enables managed, observable, and secure communication between individual services.

### Working of the Service
![[Pasted image 20230730110815.png]]

1. Customer requests a ride by specifying the source, destination, cab type, payment method, etc.
2. Ride service registers this request, finds nearby drivers, and calculates the estimated time of arrival (ETA).
3. The request is then broadcasted to the nearby drivers for them to accept or deny.
4. If the driver accepts, the customer is notified about the live location of the driver with the estimated time of arrival (ETA) while they wait for pickup.
5. The customer is picked up and the driver can start the trip.
6. Once the destination is reached, the driver will mark the ride as complete and collect payment.
7. After the payment is complete, the customer can leave a rating and feedback for the trip if they like.

### Location Tracking
We will use a **Push Model** to efficiently send and receive live location data from the client (customers and drivers) to our backend.

The client opens a long-lived connection with the server and once new data is available it will be pushed to the client. We will use **[[WebSockets]]** for this.

The pull model approach is not scalable as it will create unnecessary request overhead on our servers and most of the time the response will be empty, thus wasting our resources. To minimize latency, using the push model with **[[WebSockets]]** is a better choice because then we can push data to the client once it's available without any delay, given the connection is open with the client. Also, **[[WebSockets]]** provide full-duplex communication, unlike **[[Server-Sent Events (SSE)]]** which are only unidirectional.\

Additionally, the client application should have some sort of background job mechanism to ping GPS location while the application is in the background.

### Ride Matching
We need a way to efficiently store and query nearby drivers.

Using the customer's **geohash** (refer to [[Geohashing]]) we can determine the nearest available driver by simply comparing it with the driver's geohash. For better performance, we will index and store the **geohash** of the driver in memory for faster retrieval.

**[[Quadtree]]** seems perfect for our use case, we can update the Quadtree every time we receive a new location update from the driver. To reduce the load on the quadtree servers we can use an in-memory datastore such as [Redis](https://redis.io) to cache the latest updates. And with the application of mapping algorithms such as the [Hilbert curve](https://en.wikipedia.org/wiki/Hilbert_curve), we can perform efficient range queries to find nearby drivers for the customer.

##### Why not use SQL?
We already have access to the latitude and longitude of our customers, and with databases like [PostgreSQL](https://www.postgresql.org) and [MySQL](https://www.mysql.com) we can perform a query to find nearby driver locations given a latitude and longitude (X, Y) within a radius (R).

```sql

SELECT * FROM locations WHERE lat BETWEEN X-R AND X+R AND long BETWEEN Y-R AND Y+R
```

However, this is not scalable, and performing this query on large datasets will be quite slow.

##### What about race conditions?
Race conditions can easily occur when a large number of customers will be requesting rides simultaneously. To avoid this, we can wrap our ride matching logic in a **[[Mutex]]** to avoid any race conditions. Furthermore, every action should be **[transactional](obsidian://open?vault=System%20Design&file=Dictionary%2FTransactions%2FTransactions)** in nature.

##### How to find the best drivers nearby?
Once we have a list of nearby drivers from the Quadtree servers, we can perform some sort of ranking based on parameters like average ratings, relevance, past customer feedback, etc. This will allow us to broadcast notifications to the best available drivers first.

##### Dealing with high demand
In cases of high demand, we can use the concept of [Surge Pricing](https://www.uber.com/us/en/drive/driver-app/how-surge-works/). Surge pricing is a dynamic pricing method where prices are temporarily increased as a reaction to increased demand and mostly limited supply. This surge price can be added to the base price of the trip.

### Payments
Handling payments at scale is challenging, to simplify our system we can use a third-party payment processor like [Stripe](https://stripe.com) or [PayPal](https://www.paypal.com). Once the payment is complete, the payment processor will redirect the user back to our application and we can set up a **[[Webhooks]]** to capture all the payment-related data.

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

### Caching
In a social media application, we have to be careful about using cache as our users expect the latest data. So, to prevent usage spikes from our resources we can cache the top 20% of the tweets.

To further improve efficiency we can add pagination to our system APIs. This decision will be helpful for users with limited network bandwidth as they won't have to retrieve old messages unless requested.

##### Which cache eviction policy to use?
We can use solutions like [Redis](https://redis.io) or [Memcached](https://memcached.org) and cache 20% of the daily traffic but what kind of cache eviction policy would best fit our needs?

**[[Least Recently Used (LRU)]]** can be a good policy for our system.

##### How to handle cache miss?
Whenever there is a cache miss, our servers can hit the database directly and update the cache with the new entries.

## Identify and Resolve Bottlenecks

##### Bottlenecks:
- "What if one of our services crashes?"
- "How will we distribute our traffic between our components?"
- "How can we reduce the load on our database?"
- "How to improve the availability of our cache?"
- "How can we make our notification system more robust?"

##### Solutions:
- Running multiple instances of each of our services.
- Introducing **[[Load Balancers]]** between clients, servers, databases, and cache servers.
- Using multiple read replicas for our databases.
- Multiple instances and replicas for our distributed cache.
- Exactly once delivery and message ordering is challenging in a distributed system, we can use a dedicated **[[Message Broker]]** such as [Apache Kafka](https://kafka.apache.org) or [NATS](https://nats.io) to make our notification system more robust.


## Final Diagram
The design is done using AWS Services icons.

**[[Uber Design]]**






