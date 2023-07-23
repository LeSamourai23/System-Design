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





