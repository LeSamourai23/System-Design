---

excalidraw-plugin: parsed
tags: [excalidraw]

---
==⚠  Switch to EXCALIDRAW VIEW in the MORE OPTIONS menu of this document. ⚠==


# Text Elements
TinyURL ^tQqq2gJf

Functional Requirements: ^EpE8K117

1. Given a URL, our service should generate a shorter and unique alias of it. This is called a short
link.
2. When users access a short link, our service should redirect them to the original link.
3. Users should optionally be able to pick a custom short link for their URL.
4. Links will expire after a standard default timespan. Users should be able to specify the
expiration time.
   ^rawsCcE7

Non-Functional Requirements: ^mEQe7v26

1. The system should be highly available. This is required because, if our service is down, all the
URL redirections will start failing.
2. URL redirection should happen in real-time with minimal latency.
3. Shortened links should not be guessable (not predictable). ^J0OVVyY1

Extended Requirements: ^nVOzylku

1. Analytics; e.g., how many times a redirection happened?
2. Our service should also be accessible through REST APIs by other services. ^6haU7NeS

Requirements and Goals ^XMwuuQ8y

Estimation and Constraints ^eWLFUZzQ

Traffic Estimates: ^EluXRGoK

500 million new URL shortenings per month, with taking an estimate of 100:1 read/write ratio, makes it 50 billion redirections.

Query per second will be:
    
    500 million/ (30 days * 24 hours * 3600 seconds) ~ 200 URLs/sec

Considering 100:1 read/write ratio, URL redirections per second will be:

    100* 200 URLs/sec = 20K/sec
  ^AweMoKYy

Storage Estimates: ^YkT9ENuf

Assume we store every request for up to 5 years. Since, we have 500 million URLs per month, total number of objects we 
expect to store will be:

    500 million * 5 years * 12 months = 30 billion

Let's assume, each object will be around 500 bytes then we will need around:

    30 billion * 500 bytes = 15 TB ^dUtTWLIr

Bandwidth Estimates: ^a6SN8IlM

For write requests, since we expect 200 new URLs every second, total incoming data for our service will be:

    200 * 500 bytes = 100 KB/sec

For read requests, since we expect ~20K URLs redirections, total outgoing data will be:

    20K * 500 bytes = ~10 MB/sec

 ^oBtwHZNC

Memory Estimates: ^kQ1gGaHq

For cache, we can use the 80-20 rule as in storing the 20% hot URLs.

Since, we have 20K requests per second, the requests we get will be around:

    20K * 3600 * 23 = ~1.7 billion

To cache 20% of these requests, the memory we need will be:

    0.2 * 1.7 billion * 500 bytes = ~170 GB ^NuiqSsNZ

Summary: ^N70lfhrs

New URLs => 200 per sec
URL redirections => 20 K/sec
Incoming data => 200 KB/sec
Outgoing data => 10 MB/sec
Storage for 5 years => 15 TB
Memory for cache => 170 GB ^JOqxjS81

System APIs ^3ErjfWBt

Rest APIs will be used to expose functionality of our service. ^2AOJWoaj

Parameters: ^WDcLPV4d

api_dev_key  ^7AtM4Cfe

expire_date ^wQ710mCc

custom_url ^tzy2omSi

user_name ^BdxNniW6

original_url ^gAK8ZKoi

1.
2.
3.
4.
5. ^Thx5jljc

(string): API key for the registered account, will be used to allocate quota. ^P2mD9he3

(string): Original URL to be shortened. ^YE9vMpte

(string): Optional custom key for the URL. ^aOodXoz0

(string): Optional user name for encoding. ^8sFo7BUE

(string): Optional expiration date for URL. ^3iLhxfyM

Database  ^yoiwsHRS

We are anticipating storing billions of rows, and we don't need relationship between objects so for that
a NoSQL key-value store is a better choice as it will be easier to scale.  ^KN1Nt1Hl

Basic System Design and Algorithm ^Bx1b9sl6

We can compute a unique hash (e.g., MD5 or SHA256, etc.) of the given URL. The hash can then be
encoded for displaying. This encoding could be base36 ([a-z ,0-9]) or base62 ([A-Z, a-z, 0-9]) and if
we add ‘-’ and ‘.’ we can use base64 encoding. A reasonable question would be, what should be the
length of the short key? 6, 8 or 10 characters.

Using base64 encoding, a 6 letter long key would result in 64^6 = ~68.7 billion possible strings
Using base64 encoding, an 8 letter long key would result in 64^8 = ~281 trillion possible strings

With 68.7B unique strings, let’s assume six letter keys would suffice for our system.
If we use the MD5 algorithm as our hash function, it’ll produce a 128-bit hash value. After base64
encoding, we’ll get a string having more than 21 characters (since each base64 character encodes 6 bits
of the hash value). Since we only have space for 8 characters per short key, how will we choose our key
then? We can take the first 6 (or 8) letters for the key. This could result in key duplication though, upon
which we can choose some other characters out of the encoding string or swap some characters.
What are different issues with our solution? We have the following couple of problems with our
encoding scheme:

1. If multiple users enter the same URL, they can get the same shortened URL, which is not
acceptable.
2. What if parts of the URL are URL-encoded? e.g., http://www.educative.io/distributed.php?
id=design, and http://www.educative.io/distributed.php%3Fid%3Ddesign are identical except
for the URL encoding.

Workaround for the issues: 
We can append an increasing sequence number to each input URL to
make it unique, and then generate a hash of it. We don’t need to store this sequence number in the
databases, though. Possible problems with this approach could be an ever-increasing sequence number.
Can it overflow? Appending an increasing sequence number will also impact the performance of the
service.

Another solution could be to append user id (which should be unique) to the input URL. However, if
the user has not signed in, we would have to ask the user to choose a uniqueness key. Even after this, if
we have a conflict, we have to keep generating a key until we get a unique one ^EWd1Hd0R

Data Partitioning and Replication ^FiuUwqd7

To scale out our DB, we need to partition it so that it can store information about billions of URLs. We
need to come up with a partitioning scheme that would divide and store our data to different DB
servers.

We will use Hash-Based Partitioning for this.

Our hashing function will randomly distribute URLs into different partitions (e.g., our hashing function
can always map any key to a number between [1…256]), and this number would represent the partition
in which we store our object.
This approach can still lead to overloaded partitions, which can be solved by using Consistent Hashing.
 ^CDOFGScB

Cache ^uz9eq6GN

We can cache URLs that are frequently accessed. We can use some off-the-shelf solution like
Memcache, which can store full URLs with their respective hashes. The application servers, before
hitting backend storage, can quickly check if the cache has the desired URL.

How much cache should we have? 
We can start with 20% of daily traffic and, based on clients’ usage
pattern, we can adjust how many cache servers we need. As estimated above, we need 170GB memory
to cache 20% of daily traffic. Since a modern-day server can have 256GB memory, we can easily fit all
the cache into one machine. Alternatively, we can use a couple of smaller servers to store all these hot
URLs.

Which cache eviction policy would best fit our needs? 
When the cache is full, and we want to replace a link with a newer/hotter URL, how would we choose? 
Least Recently Used (LRU) can be a reasonable policy for our system. Under this policy, we discard the 
least recently used URL first. We can use a Linked Hash Map or a similar data structure to store our 
URLs and Hashes, which will also keep track of the URLs that have been accessed recently.
To further increase the efficiency, we can replicate our caching servers to distribute load between them.

How can each cache replica be updated? 
Whenever there is a cache miss, our servers would be hitting a backend database. Whenever this happens, 
we can update the cache and pass the new entry to all the cache replicas. Each replica can update their 
cache by adding the new entry. If a replica already has that entry, it can simply ignore it. ^hFOKLDpM

Load Balancer ^M6WAbE6g

We can add a Load balancing layer at three places in our system:
1. Between Clients and Application servers
2. Between Application Servers and database servers
3. Between Application Servers and Cache servers

Initially, we could use a simple Round Robin approach that distributes incoming requests equally
among backend servers. This LB is simple to implement and does not introduce any overhead. Another
benefit of this approach is that if a server is dead, LB will take it out of the rotation and will stop
sending any traffic to it.

A problem with Round Robin LB is that server load is not taken into consideration. If a server is
overloaded or slow, the LB will not stop sending new requests to that server. To handle this, a more
intelligent LB solution can be placed that periodically queries the backend server about its load and
adjusts traffic based on that. ^BzCJKEL8

Should entries stick around forever or should they be purged? If a user-specified expiration time is
reached, what should happen to the link?
If we chose to actively search for expired links to remove them, it would put a lot of pressure on our
database. Instead, we can slowly remove expired links and do a lazy cleanup. Our service will make
sure that only expired links will be deleted, although some expired links can live longer but will never
be returned to users.

• Whenever a user tries to access an expired link, we can delete the link and return an error to the
user.
• A separate Cleanup service can run periodically to remove expired links from our storage and
cache. This service should be very lightweight and can be scheduled to run only when the user
traffic is expected to be low.
• We can have a default expiration time for each link (e.g., two years).
• After removing an expired link, we can put the key back in the key-DB to be reused.
• Should we remove links that haven’t been visited in some length of time, say six months? This
could be tricky. Since storage is getting cheap, we can decide to keep links forever. ^1EXPxHXF

Client App ^cLtrOysC

Load Balancer ^EPtgFrIR

Servers ^l9eotsfs

Load Balancer ^Uv6tOyou

Key Generator ^npYEujtN

Key DB ^aMS4irLG

Key DB
(Standby) ^dSGgRYEk

DB Servers ^fVGHw8Np

Cleanup Service ^lWaI0ilj

CDN ^bKQHQEbp

DB Cleanup or Purging ^efsVAuc9


# Embedded files
be92df9ce570816fcb175a3fd25b9c309b49919a: [[TinyURL Logo.png]]
c8f88227c759128ebdb829a0a1b5abe3e837da1d: [[Client App.png]]
45ef7f47e1949e06a16126834858c467db0bbb89: [[Load Balancer.png]]
394cb3fb5676e05e1ad183c501110a923dc2b863: [[Server.png]]
d43bc7222c0170ee0ae3dbdc7e4de74f30c1d8c5: [[Key.png]]
fc6d4bfe4c3767f2a8be612b648adf5d57d8aca2: [[Database.png]]
d7704ead31974a1c8e641d2788cecf836ef139f4: [[Database Servers.png]]
4d80284b2779b752552bd7f9da162b7b925807b2: [[CDN.png]]
1fe30744631b3cb11961d1770f373a2104529cd8: [[CleanUp.png]]

%%
# Drawing
```json
{
	"type": "excalidraw",
	"version": 2,
	"source": "https://github.com/zsviczian/obsidian-excalidraw-plugin/releases/tag/1.9.8",
	"elements": [
		{
			"type": "image",
			"version": 296,
			"versionNonce": 1581353565,
			"isDeleted": false,
			"id": "5db6HG8CN177EnRswJOWr",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 101.32638125840612,
			"y": -330.08516549374076,
			"strokeColor": "transparent",
			"backgroundColor": "transparent",
			"width": 91.0755569082195,
			"height": 91.0755569082195,
			"seed": 809571645,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792670,
			"link": null,
			"locked": false,
			"status": "pending",
			"fileId": "be92df9ce570816fcb175a3fd25b9c309b49919a",
			"scale": [
				1,
				1
			]
		},
		{
			"type": "text",
			"version": 262,
			"versionNonce": 1211667955,
			"isDeleted": false,
			"id": "tQqq2gJf",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 222.88465338294304,
			"y": -306.0484529813179,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "transparent",
			"width": 151.83795166015625,
			"height": 48.06173951007396,
			"seed": 1161211197,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792670,
			"link": null,
			"locked": false,
			"fontSize": 38.44939160805917,
			"fontFamily": 1,
			"text": "TinyURL",
			"rawText": "TinyURL",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "TinyURL",
			"lineHeight": 1.25,
			"baseline": 34
		},
		{
			"type": "text",
			"version": 191,
			"versionNonce": 1706702525,
			"isDeleted": false,
			"id": "EpE8K117",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": -193.4500135203047,
			"y": -127.12288532230588,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "transparent",
			"width": 332.5838317871094,
			"height": 35,
			"seed": 993885971,
			"groupIds": [
				"Bn8Hhq7ds1M9YzUfRW3Gq",
				"j7sfxIEOaAlifcOeDuXGw"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792670,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Functional Requirements:",
			"rawText": "Functional Requirements:",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "Functional Requirements:",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "text",
			"version": 260,
			"versionNonce": 322396051,
			"isDeleted": false,
			"id": "rawsCcE7",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": -192.31946595405952,
			"y": -85.8753810039442,
			"strokeColor": "#1971c2",
			"backgroundColor": "transparent",
			"width": 996.4193115234375,
			"height": 175,
			"seed": 583829501,
			"groupIds": [
				"Bn8Hhq7ds1M9YzUfRW3Gq",
				"j7sfxIEOaAlifcOeDuXGw"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792670,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 1,
			"text": "1. Given a URL, our service should generate a shorter and unique alias of it. This is called a short\nlink.\n2. When users access a short link, our service should redirect them to the original link.\n3. Users should optionally be able to pick a custom short link for their URL.\n4. Links will expire after a standard default timespan. Users should be able to specify the\nexpiration time.\n  ",
			"rawText": "1. Given a URL, our service should generate a shorter and unique alias of it. This is called a short\nlink.\n2. When users access a short link, our service should redirect them to the original link.\n3. Users should optionally be able to pick a custom short link for their URL.\n4. Links will expire after a standard default timespan. Users should be able to specify the\nexpiration time.\n  ",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "1. Given a URL, our service should generate a shorter and unique alias of it. This is called a short\nlink.\n2. When users access a short link, our service should redirect them to the original link.\n3. Users should optionally be able to pick a custom short link for their URL.\n4. Links will expire after a standard default timespan. Users should be able to specify the\nexpiration time.\n  ",
			"lineHeight": 1.25,
			"baseline": 168
		},
		{
			"type": "text",
			"version": 304,
			"versionNonce": 1713264413,
			"isDeleted": false,
			"id": "mEQe7v26",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": -190.74239042629495,
			"y": 88.03621287280356,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "transparent",
			"width": 390.7117919921875,
			"height": 35,
			"seed": 809460829,
			"groupIds": [
				"Bn8Hhq7ds1M9YzUfRW3Gq",
				"j7sfxIEOaAlifcOeDuXGw"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792670,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Non-Functional Requirements:",
			"rawText": "Non-Functional Requirements:",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "Non-Functional Requirements:",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "text",
			"version": 123,
			"versionNonce": 1391195443,
			"isDeleted": false,
			"id": "J0OVVyY1",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": -189.248691720742,
			"y": 129.07881532823205,
			"strokeColor": "#1971c2",
			"backgroundColor": "transparent",
			"width": 953.5191650390625,
			"height": 100,
			"seed": 220935357,
			"groupIds": [
				"Bn8Hhq7ds1M9YzUfRW3Gq",
				"j7sfxIEOaAlifcOeDuXGw"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 1,
			"text": "1. The system should be highly available. This is required because, if our service is down, all the\nURL redirections will start failing.\n2. URL redirection should happen in real-time with minimal latency.\n3. Shortened links should not be guessable (not predictable).",
			"rawText": "1. The system should be highly available. This is required because, if our service is down, all the\nURL redirections will start failing.\n2. URL redirection should happen in real-time with minimal latency.\n3. Shortened links should not be guessable (not predictable).",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "1. The system should be highly available. This is required because, if our service is down, all the\nURL redirections will start failing.\n2. URL redirection should happen in real-time with minimal latency.\n3. Shortened links should not be guessable (not predictable).",
			"lineHeight": 1.25,
			"baseline": 93
		},
		{
			"type": "text",
			"version": 396,
			"versionNonce": 434072445,
			"isDeleted": false,
			"id": "nVOzylku",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": -190.12221960677113,
			"y": 259.99956993854454,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "transparent",
			"width": 322.8958435058594,
			"height": 35,
			"seed": 1814982611,
			"groupIds": [
				"Bn8Hhq7ds1M9YzUfRW3Gq",
				"j7sfxIEOaAlifcOeDuXGw"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Extended Requirements:",
			"rawText": "Extended Requirements:",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "Extended Requirements:",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "text",
			"version": 176,
			"versionNonce": 113039059,
			"isDeleted": false,
			"id": "6haU7NeS",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": -189.01467666506204,
			"y": 302.2217429492536,
			"strokeColor": "#1971c2",
			"backgroundColor": "transparent",
			"width": 788.099365234375,
			"height": 50,
			"seed": 1384897757,
			"groupIds": [
				"Bn8Hhq7ds1M9YzUfRW3Gq",
				"j7sfxIEOaAlifcOeDuXGw"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 1,
			"text": "1. Analytics; e.g., how many times a redirection happened?\n2. Our service should also be accessible through REST APIs by other services.",
			"rawText": "1. Analytics; e.g., how many times a redirection happened?\n2. Our service should also be accessible through REST APIs by other services.",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "1. Analytics; e.g., how many times a redirection happened?\n2. Our service should also be accessible through REST APIs by other services.",
			"lineHeight": 1.25,
			"baseline": 43
		},
		{
			"type": "rectangle",
			"version": 87,
			"versionNonce": 1899951069,
			"isDeleted": false,
			"id": "uvpGsUnKjo25nnz8puzq1",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -198.86295592514142,
			"y": -187.5076729291452,
			"strokeColor": "#2f9e44",
			"backgroundColor": "#b2f2bb",
			"width": 347.7820360553907,
			"height": 45.654784655920025,
			"seed": 568163571,
			"groupIds": [
				"j7sfxIEOaAlifcOeDuXGw"
			],
			"frameId": null,
			"roundness": {
				"type": 3
			},
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false
		},
		{
			"type": "text",
			"version": 110,
			"versionNonce": 2101380211,
			"isDeleted": false,
			"id": "XMwuuQ8y",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -188.12065365316022,
			"y": -179.45094622515927,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "transparent",
			"width": 330.65185546875,
			"height": 35,
			"seed": 2009275443,
			"groupIds": [
				"j7sfxIEOaAlifcOeDuXGw"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Requirements and Goals",
			"rawText": "Requirements and Goals",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "Requirements and Goals",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "rectangle",
			"version": 528,
			"versionNonce": 1481535549,
			"isDeleted": false,
			"id": "nSt4zOS1Q1LyWaULap53H",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -188.989313961294,
			"y": 415.0717641880558,
			"strokeColor": "#2f9e44",
			"backgroundColor": "#b2f2bb",
			"width": 396,
			"height": 49,
			"seed": 918964413,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 3
			},
			"boundElements": [
				{
					"type": "text",
					"id": "eWLFUZzQ"
				}
			],
			"updated": 1689632792671,
			"link": null,
			"locked": false
		},
		{
			"type": "text",
			"version": 380,
			"versionNonce": 1849523731,
			"isDeleted": false,
			"id": "eWLFUZzQ",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -179.61123168590336,
			"y": 422.0717641880558,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 377.24383544921875,
			"height": 35,
			"seed": 641638195,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Estimation and Constraints",
			"rawText": "Estimation and Constraints",
			"textAlign": "center",
			"verticalAlign": "middle",
			"containerId": "nSt4zOS1Q1LyWaULap53H",
			"originalText": "Estimation and Constraints",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "text",
			"version": 360,
			"versionNonce": 1022320797,
			"isDeleted": false,
			"id": "EluXRGoK",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -188.989313961294,
			"y": 483.70693832330403,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 260.1758728027344,
			"height": 35,
			"seed": 2019291229,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Traffic Estimates:",
			"rawText": "Traffic Estimates:",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "Traffic Estimates:",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "text",
			"version": 1116,
			"versionNonce": 1420444595,
			"isDeleted": false,
			"id": "AweMoKYy",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": -188.989313961294,
			"y": 523.9393246441405,
			"strokeColor": "#1971c2",
			"backgroundColor": "transparent",
			"width": 1177.5689697265625,
			"height": 237.43982167221557,
			"seed": 202615293,
			"groupIds": [
				"9B9pzvBDA1ROd_6YYaEe7",
				"MtrrwUYDsDUJbGZKu7vgK"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 18.995185733777245,
			"fontFamily": 1,
			"text": "500 million new URL shortenings per month, with taking an estimate of 100:1 read/write ratio, makes it 50 billion redirections.\n\nQuery per second will be:\n    \n    500 million/ (30 days * 24 hours * 3600 seconds) ~ 200 URLs/sec\n\nConsidering 100:1 read/write ratio, URL redirections per second will be:\n\n    100* 200 URLs/sec = 20K/sec\n ",
			"rawText": "500 million new URL shortenings per month, with taking an estimate of 100:1 read/write ratio, makes it 50 billion redirections.\n\nQuery per second will be:\n    \n    500 million/ (30 days * 24 hours * 3600 seconds) ~ 200 URLs/sec\n\nConsidering 100:1 read/write ratio, URL redirections per second will be:\n\n    100* 200 URLs/sec = 20K/sec\n ",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "500 million new URL shortenings per month, with taking an estimate of 100:1 read/write ratio, makes it 50 billion redirections.\n\nQuery per second will be:\n    \n    500 million/ (30 days * 24 hours * 3600 seconds) ~ 200 URLs/sec\n\nConsidering 100:1 read/write ratio, URL redirections per second will be:\n\n    100* 200 URLs/sec = 20K/sec\n ",
			"lineHeight": 1.25,
			"baseline": 230
		},
		{
			"type": "text",
			"version": 434,
			"versionNonce": 1423131901,
			"isDeleted": false,
			"id": "YkT9ENuf",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -188.989313961294,
			"y": 763.2106983863871,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 267.95989990234375,
			"height": 35,
			"seed": 1958995891,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Storage Estimates:",
			"rawText": "Storage Estimates:",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "Storage Estimates:",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "text",
			"version": 811,
			"versionNonce": 145577299,
			"isDeleted": false,
			"id": "dUtTWLIr",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -188.989313961294,
			"y": 803.6780438985412,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 1226.55908203125,
			"height": 200,
			"seed": 1807788627,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 1,
			"text": "Assume we store every request for up to 5 years. Since, we have 500 million URLs per month, total number of objects we \nexpect to store will be:\n\n    500 million * 5 years * 12 months = 30 billion\n\nLet's assume, each object will be around 500 bytes then we will need around:\n\n    30 billion * 500 bytes = 15 TB",
			"rawText": "Assume we store every request for up to 5 years. Since, we have 500 million URLs per month, total number of objects we \nexpect to store will be:\n\n    500 million * 5 years * 12 months = 30 billion\n\nLet's assume, each object will be around 500 bytes then we will need around:\n\n    30 billion * 500 bytes = 15 TB",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "Assume we store every request for up to 5 years. Since, we have 500 million URLs per month, total number of objects we \nexpect to store will be:\n\n    500 million * 5 years * 12 months = 30 billion\n\nLet's assume, each object will be around 500 bytes then we will need around:\n\n    30 billion * 500 bytes = 15 TB",
			"lineHeight": 1.25,
			"baseline": 193
		},
		{
			"type": "text",
			"version": 583,
			"versionNonce": 595303773,
			"isDeleted": false,
			"id": "a6SN8IlM",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -188.989313961294,
			"y": 1032.7993645629915,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 296.3518371582031,
			"height": 35,
			"seed": 1750075059,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Bandwidth Estimates:",
			"rawText": "Bandwidth Estimates:",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "Bandwidth Estimates:",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "text",
			"version": 634,
			"versionNonce": 429732595,
			"isDeleted": false,
			"id": "oBtwHZNC",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -188.989313961294,
			"y": 1081.2993645629915,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 1079.0391845703125,
			"height": 225,
			"seed": 46114909,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 1,
			"text": "For write requests, since we expect 200 new URLs every second, total incoming data for our service will be:\n\n    200 * 500 bytes = 100 KB/sec\n\nFor read requests, since we expect ~20K URLs redirections, total outgoing data will be:\n\n    20K * 500 bytes = ~10 MB/sec\n\n",
			"rawText": "For write requests, since we expect 200 new URLs every second, total incoming data for our service will be:\n\n    200 * 500 bytes = 100 KB/sec\n\nFor read requests, since we expect ~20K URLs redirections, total outgoing data will be:\n\n    20K * 500 bytes = ~10 MB/sec\n\n",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "For write requests, since we expect 200 new URLs every second, total incoming data for our service will be:\n\n    200 * 500 bytes = 100 KB/sec\n\nFor read requests, since we expect ~20K URLs redirections, total outgoing data will be:\n\n    20K * 500 bytes = ~10 MB/sec\n\n",
			"lineHeight": 1.25,
			"baseline": 218
		},
		{
			"type": "text",
			"version": 703,
			"versionNonce": 1077537213,
			"isDeleted": false,
			"id": "kQ1gGaHq",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -188.989313961294,
			"y": 1278.7993645629915,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 254.7158966064453,
			"height": 35,
			"seed": 2008283005,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Memory Estimates:",
			"rawText": "Memory Estimates:",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "Memory Estimates:",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "text",
			"version": 627,
			"versionNonce": 1744854163,
			"isDeleted": false,
			"id": "NuiqSsNZ",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -188.989313961294,
			"y": 1324.2993645629915,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 765.359375,
			"height": 225,
			"seed": 1296638067,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 1,
			"text": "For cache, we can use the 80-20 rule as in storing the 20% hot URLs.\n\nSince, we have 20K requests per second, the requests we get will be around:\n\n    20K * 3600 * 23 = ~1.7 billion\n\nTo cache 20% of these requests, the memory we need will be:\n\n    0.2 * 1.7 billion * 500 bytes = ~170 GB",
			"rawText": "For cache, we can use the 80-20 rule as in storing the 20% hot URLs.\n\nSince, we have 20K requests per second, the requests we get will be around:\n\n    20K * 3600 * 23 = ~1.7 billion\n\nTo cache 20% of these requests, the memory we need will be:\n\n    0.2 * 1.7 billion * 500 bytes = ~170 GB",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "For cache, we can use the 80-20 rule as in storing the 20% hot URLs.\n\nSince, we have 20K requests per second, the requests we get will be around:\n\n    20K * 3600 * 23 = ~1.7 billion\n\nTo cache 20% of these requests, the memory we need will be:\n\n    0.2 * 1.7 billion * 500 bytes = ~170 GB",
			"lineHeight": 1.25,
			"baseline": 218
		},
		{
			"type": "text",
			"version": 776,
			"versionNonce": 1546627613,
			"isDeleted": false,
			"id": "N70lfhrs",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -188.989313961294,
			"y": 1577.1641383441145,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 118.99993896484375,
			"height": 35,
			"seed": 1210265747,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Summary:",
			"rawText": "Summary:",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "Summary:",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "text",
			"version": 477,
			"versionNonce": 2139968051,
			"isDeleted": false,
			"id": "JOqxjS81",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -188.989313961294,
			"y": 1623.5360797899782,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 302.759765625,
			"height": 150,
			"seed": 1874783133,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 1,
			"text": "New URLs => 200 per sec\nURL redirections => 20 K/sec\nIncoming data => 200 KB/sec\nOutgoing data => 10 MB/sec\nStorage for 5 years => 15 TB\nMemory for cache => 170 GB",
			"rawText": "New URLs => 200 per sec\nURL redirections => 20 K/sec\nIncoming data => 200 KB/sec\nOutgoing data => 10 MB/sec\nStorage for 5 years => 15 TB\nMemory for cache => 170 GB",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "New URLs => 200 per sec\nURL redirections => 20 K/sec\nIncoming data => 200 KB/sec\nOutgoing data => 10 MB/sec\nStorage for 5 years => 15 TB\nMemory for cache => 170 GB",
			"lineHeight": 1.25,
			"baseline": 143
		},
		{
			"type": "rectangle",
			"version": 232,
			"versionNonce": 139353725,
			"isDeleted": false,
			"id": "puUdVzZQcmQ7JMy3D4cXN",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -190.0492889253407,
			"y": 1828.0050965337862,
			"strokeColor": "#2f9e44",
			"backgroundColor": "#b2f2bb",
			"width": 243,
			"height": 56,
			"seed": 1907397395,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 3
			},
			"boundElements": [
				{
					"type": "text",
					"id": "3ErjfWBt"
				}
			],
			"updated": 1689632792671,
			"link": null,
			"locked": false
		},
		{
			"type": "text",
			"version": 184,
			"versionNonce": 824873939,
			"isDeleted": false,
			"id": "3ErjfWBt",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -156.32925718705945,
			"y": 1838.5050965337862,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 175.5599365234375,
			"height": 35,
			"seed": 1520789405,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "System APIs",
			"rawText": "System APIs",
			"textAlign": "center",
			"verticalAlign": "middle",
			"containerId": "puUdVzZQcmQ7JMy3D4cXN",
			"originalText": "System APIs",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "text",
			"version": 157,
			"versionNonce": 995591901,
			"isDeleted": false,
			"id": "2AOJWoaj",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -190.0492889253407,
			"y": 1892.1907427816868,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 618.5394287109375,
			"height": 25,
			"seed": 866779133,
			"groupIds": [
				"bhbU0raVKZ2KtObVlOr-3"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 1,
			"text": "Rest APIs will be used to expose functionality of our service.",
			"rawText": "Rest APIs will be used to expose functionality of our service.",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "Rest APIs will be used to expose functionality of our service.",
			"lineHeight": 1.25,
			"baseline": 18
		},
		{
			"type": "text",
			"version": 43,
			"versionNonce": 1827578227,
			"isDeleted": false,
			"id": "WDcLPV4d",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -190.0492889253407,
			"y": 1936.4947263101408,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 166.09593200683594,
			"height": 35,
			"seed": 1115741811,
			"groupIds": [
				"bhbU0raVKZ2KtObVlOr-3"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Parameters:",
			"rawText": "Parameters:",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "Parameters:",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "text",
			"version": 160,
			"versionNonce": 400335677,
			"isDeleted": false,
			"id": "7AtM4Cfe",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -161.00297215430714,
			"y": 1981.6554597705292,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 140.625,
			"height": 24,
			"seed": 145554099,
			"groupIds": [
				"iHlT_037m0WgEKTmrhWxs",
				"bhbU0raVKZ2KtObVlOr-3"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 3,
			"text": "api_dev_key ",
			"rawText": "api_dev_key ",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "api_dev_key ",
			"lineHeight": 1.2,
			"baseline": 19
		},
		{
			"type": "text",
			"version": 266,
			"versionNonce": 1545599763,
			"isDeleted": false,
			"id": "wQ710mCc",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -155.54247188204602,
			"y": 2118.9272269907938,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 128.90625,
			"height": 24,
			"seed": 1979503315,
			"groupIds": [
				"iHlT_037m0WgEKTmrhWxs",
				"bhbU0raVKZ2KtObVlOr-3"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 3,
			"text": "expire_date",
			"rawText": "expire_date",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "expire_date",
			"lineHeight": 1.2,
			"baseline": 19
		},
		{
			"type": "text",
			"version": 246,
			"versionNonce": 1916744605,
			"isDeleted": false,
			"id": "tzy2omSi",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -155.54247188204602,
			"y": 2051.9944100836296,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 117.1875,
			"height": 24,
			"seed": 63157309,
			"groupIds": [
				"iHlT_037m0WgEKTmrhWxs",
				"bhbU0raVKZ2KtObVlOr-3"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 3,
			"text": "custom_url",
			"rawText": "custom_url",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "custom_url",
			"lineHeight": 1.2,
			"baseline": 19
		},
		{
			"type": "text",
			"version": 233,
			"versionNonce": 993337523,
			"isDeleted": false,
			"id": "BdxNniW6",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -156.80535521991698,
			"y": 2087.3551435440186,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 105.46875,
			"height": 24,
			"seed": 2040250291,
			"groupIds": [
				"iHlT_037m0WgEKTmrhWxs",
				"bhbU0raVKZ2KtObVlOr-3"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 3,
			"text": "user_name",
			"rawText": "user_name",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "user_name",
			"lineHeight": 1.2,
			"baseline": 19
		},
		{
			"type": "text",
			"version": 230,
			"versionNonce": 380680189,
			"isDeleted": false,
			"id": "gAK8ZKoi",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -158.0682385577881,
			"y": 2015.37079328537,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 140.625,
			"height": 24,
			"seed": 1581736285,
			"groupIds": [
				"iHlT_037m0WgEKTmrhWxs",
				"bhbU0raVKZ2KtObVlOr-3"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 3,
			"text": "original_url",
			"rawText": "original_url",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "original_url",
			"lineHeight": 1.2,
			"baseline": 19
		},
		{
			"type": "text",
			"version": 242,
			"versionNonce": 1943310931,
			"isDeleted": false,
			"id": "Thx5jljc",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -190.0492889253407,
			"y": 1986.9790366051839,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 25.09368896484375,
			"height": 159.09785012251763,
			"seed": 2057600317,
			"groupIds": [
				"iHlT_037m0WgEKTmrhWxs",
				"bhbU0raVKZ2KtObVlOr-3"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 25.45565601960282,
			"fontFamily": 1,
			"text": "1.\n2.\n3.\n4.\n5.",
			"rawText": "1.\n2.\n3.\n4.\n5.",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "1.\n2.\n3.\n4.\n5.",
			"lineHeight": 1.25,
			"baseline": 149
		},
		{
			"type": "text",
			"version": 165,
			"versionNonce": 1328722013,
			"isDeleted": false,
			"id": "P2mD9he3",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -27.137338339978953,
			"y": 1983.1698764598843,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 761.95947265625,
			"height": 25,
			"seed": 1282476957,
			"groupIds": [
				"iHlT_037m0WgEKTmrhWxs",
				"bhbU0raVKZ2KtObVlOr-3"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 1,
			"text": "(string): API key for the registered account, will be used to allocate quota.",
			"rawText": "(string): API key for the registered account, will be used to allocate quota.",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "(string): API key for the registered account, will be used to allocate quota.",
			"lineHeight": 1.25,
			"baseline": 18
		},
		{
			"type": "text",
			"version": 312,
			"versionNonce": 1993658355,
			"isDeleted": false,
			"id": "YE9vMpte",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -12.152410740117205,
			"y": 2016.1336766232412,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 377.3396911621094,
			"height": 25,
			"seed": 345127965,
			"groupIds": [
				"iHlT_037m0WgEKTmrhWxs",
				"bhbU0raVKZ2KtObVlOr-3"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 1,
			"text": "(string): Original URL to be shortened.",
			"rawText": "(string): Original URL to be shortened.",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "(string): Original URL to be shortened.",
			"lineHeight": 1.25,
			"baseline": 18
		},
		{
			"type": "text",
			"version": 416,
			"versionNonce": 697736381,
			"isDeleted": false,
			"id": "aOodXoz0",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -28.900449916122795,
			"y": 2054.0201767593717,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 417.6596374511719,
			"height": 25,
			"seed": 669853811,
			"groupIds": [
				"iHlT_037m0WgEKTmrhWxs",
				"bhbU0raVKZ2KtObVlOr-3"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 1,
			"text": "(string): Optional custom key for the URL.",
			"rawText": "(string): Optional custom key for the URL.",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "(string): Optional custom key for the URL.",
			"lineHeight": 1.25,
			"baseline": 18
		},
		{
			"type": "text",
			"version": 505,
			"versionNonce": 1446795667,
			"isDeleted": false,
			"id": "8sFo7BUE",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -45.27177304704077,
			"y": 2089.38091021976,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 402.919677734375,
			"height": 25,
			"seed": 1440677299,
			"groupIds": [
				"iHlT_037m0WgEKTmrhWxs",
				"bhbU0raVKZ2KtObVlOr-3"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 1,
			"text": "(string): Optional user name for encoding.",
			"rawText": "(string): Optional user name for encoding.",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "(string): Optional user name for encoding.",
			"lineHeight": 1.25,
			"baseline": 18
		},
		{
			"type": "text",
			"version": 593,
			"versionNonce": 2105408797,
			"isDeleted": false,
			"id": "3iLhxfyM",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -21.484309796319224,
			"y": 2120.952993666536,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 419.59967041015625,
			"height": 25,
			"seed": 1888761181,
			"groupIds": [
				"iHlT_037m0WgEKTmrhWxs",
				"bhbU0raVKZ2KtObVlOr-3"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 1,
			"text": "(string): Optional expiration date for URL.",
			"rawText": "(string): Optional expiration date for URL.",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "(string): Optional expiration date for URL.",
			"lineHeight": 1.25,
			"baseline": 18
		},
		{
			"type": "rectangle",
			"version": 214,
			"versionNonce": 524398387,
			"isDeleted": false,
			"id": "JSrzxiNraOx50Jd6UVLzx",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -182.73459367632856,
			"y": 2182.5919439796353,
			"strokeColor": "#2f9e44",
			"backgroundColor": "#b2f2bb",
			"width": 201,
			"height": 56,
			"seed": 923384925,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 3
			},
			"boundElements": [
				{
					"type": "text",
					"id": "yoiwsHRS"
				}
			],
			"updated": 1689632792671,
			"link": null,
			"locked": false
		},
		{
			"type": "text",
			"version": 159,
			"versionNonce": 844779901,
			"isDeleted": false,
			"id": "yoiwsHRS",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -158.4505742061137,
			"y": 2193.0919439796353,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 152.4319610595703,
			"height": 35,
			"seed": 211329043,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Database ",
			"rawText": "Database ",
			"textAlign": "center",
			"verticalAlign": "middle",
			"containerId": "JSrzxiNraOx50Jd6UVLzx",
			"originalText": "Database ",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "text",
			"version": 342,
			"versionNonce": 1951092947,
			"isDeleted": false,
			"id": "KN1Nt1Hl",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -182.73459367632856,
			"y": 2256.9474608595733,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 1026.0391845703125,
			"height": 50,
			"seed": 981171741,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792671,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 1,
			"text": "We are anticipating storing billions of rows, and we don't need relationship between objects so for that\na NoSQL key-value store is a better choice as it will be easier to scale. ",
			"rawText": "We are anticipating storing billions of rows, and we don't need relationship between objects so for that\na NoSQL key-value store is a better choice as it will be easier to scale. ",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "We are anticipating storing billions of rows, and we don't need relationship between objects so for that\na NoSQL key-value store is a better choice as it will be easier to scale. ",
			"lineHeight": 1.25,
			"baseline": 43
		},
		{
			"type": "rectangle",
			"version": 474,
			"versionNonce": 1164777949,
			"isDeleted": false,
			"id": "EhHi_pTnp78EUaWSOM2cH",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -182.38409602451713,
			"y": 2348.412008418386,
			"strokeColor": "#2f9e44",
			"backgroundColor": "#b2f2bb",
			"width": 504,
			"height": 56,
			"seed": 871342259,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 3
			},
			"boundElements": [
				{
					"type": "text",
					"id": "Bx1b9sl6"
				}
			],
			"updated": 1689632792671,
			"link": null,
			"locked": false
		},
		{
			"type": "text",
			"version": 336,
			"versionNonce": 1726987891,
			"isDeleted": false,
			"id": "Bx1b9sl6",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -169.75598323154838,
			"y": 2358.912008418386,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 478.7437744140625,
			"height": 35,
			"seed": 197394461,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792672,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Basic System Design and Algorithm",
			"rawText": "Basic System Design and Algorithm",
			"textAlign": "center",
			"verticalAlign": "middle",
			"containerId": "EhHi_pTnp78EUaWSOM2cH",
			"originalText": "Basic System Design and Algorithm",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "text",
			"version": 95,
			"versionNonce": 264754749,
			"isDeleted": false,
			"id": "EWd1Hd0R",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -182.38409602451713,
			"y": 2416.627774195008,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 1065.1990966796875,
			"height": 825,
			"seed": 146422099,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792672,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 1,
			"text": "We can compute a unique hash (e.g., MD5 or SHA256, etc.) of the given URL. The hash can then be\nencoded for displaying. This encoding could be base36 ([a-z ,0-9]) or base62 ([A-Z, a-z, 0-9]) and if\nwe add ‘-’ and ‘.’ we can use base64 encoding. A reasonable question would be, what should be the\nlength of the short key? 6, 8 or 10 characters.\n\nUsing base64 encoding, a 6 letter long key would result in 64^6 = ~68.7 billion possible strings\nUsing base64 encoding, an 8 letter long key would result in 64^8 = ~281 trillion possible strings\n\nWith 68.7B unique strings, let’s assume six letter keys would suffice for our system.\nIf we use the MD5 algorithm as our hash function, it’ll produce a 128-bit hash value. After base64\nencoding, we’ll get a string having more than 21 characters (since each base64 character encodes 6 bits\nof the hash value). Since we only have space for 8 characters per short key, how will we choose our key\nthen? We can take the first 6 (or 8) letters for the key. This could result in key duplication though, upon\nwhich we can choose some other characters out of the encoding string or swap some characters.\nWhat are different issues with our solution? We have the following couple of problems with our\nencoding scheme:\n\n1. If multiple users enter the same URL, they can get the same shortened URL, which is not\nacceptable.\n2. What if parts of the URL are URL-encoded? e.g., http://www.educative.io/distributed.php?\nid=design, and http://www.educative.io/distributed.php%3Fid%3Ddesign are identical except\nfor the URL encoding.\n\nWorkaround for the issues: \nWe can append an increasing sequence number to each input URL to\nmake it unique, and then generate a hash of it. We don’t need to store this sequence number in the\ndatabases, though. Possible problems with this approach could be an ever-increasing sequence number.\nCan it overflow? Appending an increasing sequence number will also impact the performance of the\nservice.\n\nAnother solution could be to append user id (which should be unique) to the input URL. However, if\nthe user has not signed in, we would have to ask the user to choose a uniqueness key. Even after this, if\nwe have a conflict, we have to keep generating a key until we get a unique one",
			"rawText": "We can compute a unique hash (e.g., MD5 or SHA256, etc.) of the given URL. The hash can then be\nencoded for displaying. This encoding could be base36 ([a-z ,0-9]) or base62 ([A-Z, a-z, 0-9]) and if\nwe add ‘-’ and ‘.’ we can use base64 encoding. A reasonable question would be, what should be the\nlength of the short key? 6, 8 or 10 characters.\n\nUsing base64 encoding, a 6 letter long key would result in 64^6 = ~68.7 billion possible strings\nUsing base64 encoding, an 8 letter long key would result in 64^8 = ~281 trillion possible strings\n\nWith 68.7B unique strings, let’s assume six letter keys would suffice for our system.\nIf we use the MD5 algorithm as our hash function, it’ll produce a 128-bit hash value. After base64\nencoding, we’ll get a string having more than 21 characters (since each base64 character encodes 6 bits\nof the hash value). Since we only have space for 8 characters per short key, how will we choose our key\nthen? We can take the first 6 (or 8) letters for the key. This could result in key duplication though, upon\nwhich we can choose some other characters out of the encoding string or swap some characters.\nWhat are different issues with our solution? We have the following couple of problems with our\nencoding scheme:\n\n1. If multiple users enter the same URL, they can get the same shortened URL, which is not\nacceptable.\n2. What if parts of the URL are URL-encoded? e.g., http://www.educative.io/distributed.php?\nid=design, and http://www.educative.io/distributed.php%3Fid%3Ddesign are identical except\nfor the URL encoding.\n\nWorkaround for the issues: \nWe can append an increasing sequence number to each input URL to\nmake it unique, and then generate a hash of it. We don’t need to store this sequence number in the\ndatabases, though. Possible problems with this approach could be an ever-increasing sequence number.\nCan it overflow? Appending an increasing sequence number will also impact the performance of the\nservice.\n\nAnother solution could be to append user id (which should be unique) to the input URL. However, if\nthe user has not signed in, we would have to ask the user to choose a uniqueness key. Even after this, if\nwe have a conflict, we have to keep generating a key until we get a unique one",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "We can compute a unique hash (e.g., MD5 or SHA256, etc.) of the given URL. The hash can then be\nencoded for displaying. This encoding could be base36 ([a-z ,0-9]) or base62 ([A-Z, a-z, 0-9]) and if\nwe add ‘-’ and ‘.’ we can use base64 encoding. A reasonable question would be, what should be the\nlength of the short key? 6, 8 or 10 characters.\n\nUsing base64 encoding, a 6 letter long key would result in 64^6 = ~68.7 billion possible strings\nUsing base64 encoding, an 8 letter long key would result in 64^8 = ~281 trillion possible strings\n\nWith 68.7B unique strings, let’s assume six letter keys would suffice for our system.\nIf we use the MD5 algorithm as our hash function, it’ll produce a 128-bit hash value. After base64\nencoding, we’ll get a string having more than 21 characters (since each base64 character encodes 6 bits\nof the hash value). Since we only have space for 8 characters per short key, how will we choose our key\nthen? We can take the first 6 (or 8) letters for the key. This could result in key duplication though, upon\nwhich we can choose some other characters out of the encoding string or swap some characters.\nWhat are different issues with our solution? We have the following couple of problems with our\nencoding scheme:\n\n1. If multiple users enter the same URL, they can get the same shortened URL, which is not\nacceptable.\n2. What if parts of the URL are URL-encoded? e.g., http://www.educative.io/distributed.php?\nid=design, and http://www.educative.io/distributed.php%3Fid%3Ddesign are identical except\nfor the URL encoding.\n\nWorkaround for the issues: \nWe can append an increasing sequence number to each input URL to\nmake it unique, and then generate a hash of it. We don’t need to store this sequence number in the\ndatabases, though. Possible problems with this approach could be an ever-increasing sequence number.\nCan it overflow? Appending an increasing sequence number will also impact the performance of the\nservice.\n\nAnother solution could be to append user id (which should be unique) to the input URL. However, if\nthe user has not signed in, we would have to ask the user to choose a uniqueness key. Even after this, if\nwe have a conflict, we have to keep generating a key until we get a unique one",
			"lineHeight": 1.25,
			"baseline": 818
		},
		{
			"type": "rectangle",
			"version": 588,
			"versionNonce": 301299731,
			"isDeleted": false,
			"id": "-nQcmCUdKQrqPLrCzqtgd",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -185.63273820628768,
			"y": 3277.0375055311524,
			"strokeColor": "#2f9e44",
			"backgroundColor": "#b2f2bb",
			"width": 504,
			"height": 56,
			"seed": 1579590749,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 3
			},
			"boundElements": [
				{
					"type": "text",
					"id": "FiuUwqd7"
				}
			],
			"updated": 1689632792672,
			"link": null,
			"locked": false
		},
		{
			"type": "text",
			"version": 483,
			"versionNonce": 1280050845,
			"isDeleted": false,
			"id": "FiuUwqd7",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -165.89261064281112,
			"y": 3287.5375055311524,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 464.5197448730469,
			"height": 35,
			"seed": 502010045,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792672,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Data Partitioning and Replication",
			"rawText": "Data Partitioning and Replication",
			"textAlign": "center",
			"verticalAlign": "middle",
			"containerId": "-nQcmCUdKQrqPLrCzqtgd",
			"originalText": "Data Partitioning and Replication",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "text",
			"version": 102,
			"versionNonce": 1934818739,
			"isDeleted": false,
			"id": "CDOFGScB",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -185.63273820628768,
			"y": 3356.324326533688,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 1059.9591064453125,
			"height": 275,
			"seed": 1071197597,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792672,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 1,
			"text": "To scale out our DB, we need to partition it so that it can store information about billions of URLs. We\nneed to come up with a partitioning scheme that would divide and store our data to different DB\nservers.\n\nWe will use Hash-Based Partitioning for this.\n\nOur hashing function will randomly distribute URLs into different partitions (e.g., our hashing function\ncan always map any key to a number between [1…256]), and this number would represent the partition\nin which we store our object.\nThis approach can still lead to overloaded partitions, which can be solved by using Consistent Hashing.\n",
			"rawText": "To scale out our DB, we need to partition it so that it can store information about billions of URLs. We\nneed to come up with a partitioning scheme that would divide and store our data to different DB\nservers.\n\nWe will use Hash-Based Partitioning for this.\n\nOur hashing function will randomly distribute URLs into different partitions (e.g., our hashing function\ncan always map any key to a number between [1…256]), and this number would represent the partition\nin which we store our object.\nThis approach can still lead to overloaded partitions, which can be solved by using Consistent Hashing.\n",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "To scale out our DB, we need to partition it so that it can store information about billions of URLs. We\nneed to come up with a partitioning scheme that would divide and store our data to different DB\nservers.\n\nWe will use Hash-Based Partitioning for this.\n\nOur hashing function will randomly distribute URLs into different partitions (e.g., our hashing function\ncan always map any key to a number between [1…256]), and this number would represent the partition\nin which we store our object.\nThis approach can still lead to overloaded partitions, which can be solved by using Consistent Hashing.\n",
			"lineHeight": 1.25,
			"baseline": 268
		},
		{
			"type": "rectangle",
			"version": 214,
			"versionNonce": 1571498749,
			"isDeleted": false,
			"id": "oGIBuxW9kUUIfLhaKFZqY",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -176.88434146570034,
			"y": 3631.5220228351595,
			"strokeColor": "#2f9e44",
			"backgroundColor": "#b2f2bb",
			"width": 129,
			"height": 54,
			"seed": 767251389,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 3
			},
			"boundElements": [
				{
					"type": "text",
					"id": "uz9eq6GN"
				}
			],
			"updated": 1689632792672,
			"link": null,
			"locked": false
		},
		{
			"type": "text",
			"version": 162,
			"versionNonce": 103731027,
			"isDeleted": false,
			"id": "uz9eq6GN",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -152.34032748864956,
			"y": 3641.0220228351595,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 79.91197204589844,
			"height": 35,
			"seed": 87818067,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792672,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Cache",
			"rawText": "Cache",
			"textAlign": "center",
			"verticalAlign": "middle",
			"containerId": "oGIBuxW9kUUIfLhaKFZqY",
			"originalText": "Cache",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "text",
			"version": 94,
			"versionNonce": 1416020829,
			"isDeleted": false,
			"id": "hFOKLDpM",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -176.88434146570034,
			"y": 3704.0393751893466,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 1070.019287109375,
			"height": 550,
			"seed": 488620723,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792672,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 1,
			"text": "We can cache URLs that are frequently accessed. We can use some off-the-shelf solution like\nMemcache, which can store full URLs with their respective hashes. The application servers, before\nhitting backend storage, can quickly check if the cache has the desired URL.\n\nHow much cache should we have? \nWe can start with 20% of daily traffic and, based on clients’ usage\npattern, we can adjust how many cache servers we need. As estimated above, we need 170GB memory\nto cache 20% of daily traffic. Since a modern-day server can have 256GB memory, we can easily fit all\nthe cache into one machine. Alternatively, we can use a couple of smaller servers to store all these hot\nURLs.\n\nWhich cache eviction policy would best fit our needs? \nWhen the cache is full, and we want to replace a link with a newer/hotter URL, how would we choose? \nLeast Recently Used (LRU) can be a reasonable policy for our system. Under this policy, we discard the \nleast recently used URL first. We can use a Linked Hash Map or a similar data structure to store our \nURLs and Hashes, which will also keep track of the URLs that have been accessed recently.\nTo further increase the efficiency, we can replicate our caching servers to distribute load between them.\n\nHow can each cache replica be updated? \nWhenever there is a cache miss, our servers would be hitting a backend database. Whenever this happens, \nwe can update the cache and pass the new entry to all the cache replicas. Each replica can update their \ncache by adding the new entry. If a replica already has that entry, it can simply ignore it.",
			"rawText": "We can cache URLs that are frequently accessed. We can use some off-the-shelf solution like\nMemcache, which can store full URLs with their respective hashes. The application servers, before\nhitting backend storage, can quickly check if the cache has the desired URL.\n\nHow much cache should we have? \nWe can start with 20% of daily traffic and, based on clients’ usage\npattern, we can adjust how many cache servers we need. As estimated above, we need 170GB memory\nto cache 20% of daily traffic. Since a modern-day server can have 256GB memory, we can easily fit all\nthe cache into one machine. Alternatively, we can use a couple of smaller servers to store all these hot\nURLs.\n\nWhich cache eviction policy would best fit our needs? \nWhen the cache is full, and we want to replace a link with a newer/hotter URL, how would we choose? \nLeast Recently Used (LRU) can be a reasonable policy for our system. Under this policy, we discard the \nleast recently used URL first. We can use a Linked Hash Map or a similar data structure to store our \nURLs and Hashes, which will also keep track of the URLs that have been accessed recently.\nTo further increase the efficiency, we can replicate our caching servers to distribute load between them.\n\nHow can each cache replica be updated? \nWhenever there is a cache miss, our servers would be hitting a backend database. Whenever this happens, \nwe can update the cache and pass the new entry to all the cache replicas. Each replica can update their \ncache by adding the new entry. If a replica already has that entry, it can simply ignore it.",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "We can cache URLs that are frequently accessed. We can use some off-the-shelf solution like\nMemcache, which can store full URLs with their respective hashes. The application servers, before\nhitting backend storage, can quickly check if the cache has the desired URL.\n\nHow much cache should we have? \nWe can start with 20% of daily traffic and, based on clients’ usage\npattern, we can adjust how many cache servers we need. As estimated above, we need 170GB memory\nto cache 20% of daily traffic. Since a modern-day server can have 256GB memory, we can easily fit all\nthe cache into one machine. Alternatively, we can use a couple of smaller servers to store all these hot\nURLs.\n\nWhich cache eviction policy would best fit our needs? \nWhen the cache is full, and we want to replace a link with a newer/hotter URL, how would we choose? \nLeast Recently Used (LRU) can be a reasonable policy for our system. Under this policy, we discard the \nleast recently used URL first. We can use a Linked Hash Map or a similar data structure to store our \nURLs and Hashes, which will also keep track of the URLs that have been accessed recently.\nTo further increase the efficiency, we can replicate our caching servers to distribute load between them.\n\nHow can each cache replica be updated? \nWhenever there is a cache miss, our servers would be hitting a backend database. Whenever this happens, \nwe can update the cache and pass the new entry to all the cache replicas. Each replica can update their \ncache by adding the new entry. If a replica already has that entry, it can simply ignore it.",
			"lineHeight": 1.25,
			"baseline": 543
		},
		{
			"type": "rectangle",
			"version": 146,
			"versionNonce": 484404467,
			"isDeleted": false,
			"id": "TW03Iwj9HXlrH70ORkFmz",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -171.039761651054,
			"y": 4286.253217468576,
			"strokeColor": "#2f9e44",
			"backgroundColor": "#b2f2bb",
			"width": 230,
			"height": 57,
			"seed": 1219191709,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 3
			},
			"boundElements": [
				{
					"type": "text",
					"id": "M6WAbE6g"
				}
			],
			"updated": 1689632792672,
			"link": null,
			"locked": false
		},
		{
			"type": "text",
			"version": 91,
			"versionNonce": 886067133,
			"isDeleted": false,
			"id": "M6WAbE6g",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -156.26572142888602,
			"y": 4297.253217468576,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 200.45191955566406,
			"height": 35,
			"seed": 1163391795,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792672,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Load Balancer",
			"rawText": "Load Balancer",
			"textAlign": "center",
			"verticalAlign": "middle",
			"containerId": "TW03Iwj9HXlrH70ORkFmz",
			"originalText": "Load Balancer",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "text",
			"version": 137,
			"versionNonce": 2062243475,
			"isDeleted": false,
			"id": "BzCJKEL8",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -171.039761651054,
			"y": 4368.789101779238,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 1022.2391357421875,
			"height": 350,
			"seed": 212860819,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792672,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 1,
			"text": "We can add a Load balancing layer at three places in our system:\n1. Between Clients and Application servers\n2. Between Application Servers and database servers\n3. Between Application Servers and Cache servers\n\nInitially, we could use a simple Round Robin approach that distributes incoming requests equally\namong backend servers. This LB is simple to implement and does not introduce any overhead. Another\nbenefit of this approach is that if a server is dead, LB will take it out of the rotation and will stop\nsending any traffic to it.\n\nA problem with Round Robin LB is that server load is not taken into consideration. If a server is\noverloaded or slow, the LB will not stop sending new requests to that server. To handle this, a more\nintelligent LB solution can be placed that periodically queries the backend server about its load and\nadjusts traffic based on that.",
			"rawText": "We can add a Load balancing layer at three places in our system:\n1. Between Clients and Application servers\n2. Between Application Servers and database servers\n3. Between Application Servers and Cache servers\n\nInitially, we could use a simple Round Robin approach that distributes incoming requests equally\namong backend servers. This LB is simple to implement and does not introduce any overhead. Another\nbenefit of this approach is that if a server is dead, LB will take it out of the rotation and will stop\nsending any traffic to it.\n\nA problem with Round Robin LB is that server load is not taken into consideration. If a server is\noverloaded or slow, the LB will not stop sending new requests to that server. To handle this, a more\nintelligent LB solution can be placed that periodically queries the backend server about its load and\nadjusts traffic based on that.",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "We can add a Load balancing layer at three places in our system:\n1. Between Clients and Application servers\n2. Between Application Servers and database servers\n3. Between Application Servers and Cache servers\n\nInitially, we could use a simple Round Robin approach that distributes incoming requests equally\namong backend servers. This LB is simple to implement and does not introduce any overhead. Another\nbenefit of this approach is that if a server is dead, LB will take it out of the rotation and will stop\nsending any traffic to it.\n\nA problem with Round Robin LB is that server load is not taken into consideration. If a server is\noverloaded or slow, the LB will not stop sending new requests to that server. To handle this, a more\nintelligent LB solution can be placed that periodically queries the backend server about its load and\nadjusts traffic based on that.",
			"lineHeight": 1.25,
			"baseline": 343
		},
		{
			"type": "rectangle",
			"version": 100,
			"versionNonce": 835825853,
			"isDeleted": false,
			"id": "Q5LUtS5hBwPIlEGzztagV",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -168.8912271614273,
			"y": 4766.820129017835,
			"strokeColor": "#2f9e44",
			"backgroundColor": "#b2f2bb",
			"width": 349,
			"height": 45,
			"seed": 1330001171,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 3
			},
			"boundElements": [
				{
					"type": "text",
					"id": "efsVAuc9"
				}
			],
			"updated": 1689632926427,
			"link": null,
			"locked": false
		},
		{
			"type": "text",
			"version": 38,
			"versionNonce": 1374455187,
			"isDeleted": false,
			"id": "efsVAuc9",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "dashed",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": -148.20915929033356,
			"y": 4771.820129017835,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#d0bfff",
			"width": 307.6358642578125,
			"height": 35,
			"seed": 574337043,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632926427,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "DB Cleanup or Purging",
			"rawText": "DB Cleanup or Purging",
			"textAlign": "center",
			"verticalAlign": "middle",
			"containerId": "Q5LUtS5hBwPIlEGzztagV",
			"originalText": "DB Cleanup or Purging",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "text",
			"version": 55,
			"versionNonce": 1795376253,
			"isDeleted": false,
			"id": "1EXPxHXF",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 0,
			"opacity": 100,
			"angle": 0,
			"x": -169.03807297761102,
			"y": 4828.17665233439,
			"strokeColor": "#1971c2",
			"backgroundColor": "#b2f2bb",
			"width": 1025.6990966796875,
			"height": 400,
			"seed": 431228125,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792672,
			"link": null,
			"locked": false,
			"fontSize": 20,
			"fontFamily": 1,
			"text": "Should entries stick around forever or should they be purged? If a user-specified expiration time is\nreached, what should happen to the link?\nIf we chose to actively search for expired links to remove them, it would put a lot of pressure on our\ndatabase. Instead, we can slowly remove expired links and do a lazy cleanup. Our service will make\nsure that only expired links will be deleted, although some expired links can live longer but will never\nbe returned to users.\n\n• Whenever a user tries to access an expired link, we can delete the link and return an error to the\nuser.\n• A separate Cleanup service can run periodically to remove expired links from our storage and\ncache. This service should be very lightweight and can be scheduled to run only when the user\ntraffic is expected to be low.\n• We can have a default expiration time for each link (e.g., two years).\n• After removing an expired link, we can put the key back in the key-DB to be reused.\n• Should we remove links that haven’t been visited in some length of time, say six months? This\ncould be tricky. Since storage is getting cheap, we can decide to keep links forever.",
			"rawText": "Should entries stick around forever or should they be purged? If a user-specified expiration time is\nreached, what should happen to the link?\nIf we chose to actively search for expired links to remove them, it would put a lot of pressure on our\ndatabase. Instead, we can slowly remove expired links and do a lazy cleanup. Our service will make\nsure that only expired links will be deleted, although some expired links can live longer but will never\nbe returned to users.\n\n• Whenever a user tries to access an expired link, we can delete the link and return an error to the\nuser.\n• A separate Cleanup service can run periodically to remove expired links from our storage and\ncache. This service should be very lightweight and can be scheduled to run only when the user\ntraffic is expected to be low.\n• We can have a default expiration time for each link (e.g., two years).\n• After removing an expired link, we can put the key back in the key-DB to be reused.\n• Should we remove links that haven’t been visited in some length of time, say six months? This\ncould be tricky. Since storage is getting cheap, we can decide to keep links forever.",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "Should entries stick around forever or should they be purged? If a user-specified expiration time is\nreached, what should happen to the link?\nIf we chose to actively search for expired links to remove them, it would put a lot of pressure on our\ndatabase. Instead, we can slowly remove expired links and do a lazy cleanup. Our service will make\nsure that only expired links will be deleted, although some expired links can live longer but will never\nbe returned to users.\n\n• Whenever a user tries to access an expired link, we can delete the link and return an error to the\nuser.\n• A separate Cleanup service can run periodically to remove expired links from our storage and\ncache. This service should be very lightweight and can be scheduled to run only when the user\ntraffic is expected to be low.\n• We can have a default expiration time for each link (e.g., two years).\n• After removing an expired link, we can put the key back in the key-DB to be reused.\n• Should we remove links that haven’t been visited in some length of time, say six months? This\ncould be tricky. Since storage is getting cheap, we can decide to keep links forever.",
			"lineHeight": 1.25,
			"baseline": 393
		},
		{
			"type": "rectangle",
			"version": 768,
			"versionNonce": 757009875,
			"isDeleted": false,
			"id": "8T8kOPYzdyleVXAXMA9Jx",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": -647.7026561553884,
			"y": 5654.94957402783,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#d0bfff",
			"width": 210.6548970427981,
			"height": 231.99117278209815,
			"seed": 1993151453,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 3
			},
			"boundElements": [
				{
					"id": "fqNaVqMAipjUvC-yXnMuI",
					"type": "arrow"
				}
			],
			"updated": 1689632792672,
			"link": null,
			"locked": false
		},
		{
			"type": "text",
			"version": 563,
			"versionNonce": 41921757,
			"isDeleted": false,
			"id": "cLtrOysC",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": -614.7814656446089,
			"y": 5679.94957402783,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "transparent",
			"width": 135.71592712402344,
			"height": 35,
			"seed": 1192792125,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792672,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Client App",
			"rawText": "Client App",
			"textAlign": "center",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "Client App",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "image",
			"version": 446,
			"versionNonce": 675535731,
			"isDeleted": false,
			"id": "Yh9NInIFgV0_xHf7S5W0U",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": -619.534292342041,
			"y": 5724.873299452089,
			"strokeColor": "transparent",
			"backgroundColor": "#b2f2bb",
			"width": 140.47932186320725,
			"height": 140.47932186320725,
			"seed": 1872061693,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792672,
			"link": null,
			"locked": false,
			"status": "pending",
			"fileId": "c8f88227c759128ebdb829a0a1b5abe3e837da1d",
			"scale": [
				1,
				1
			]
		},
		{
			"type": "arrow",
			"version": 285,
			"versionNonce": 59007923,
			"isDeleted": false,
			"id": "fqNaVqMAipjUvC-yXnMuI",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": -434.8906511390422,
			"y": 5775.727995386913,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 337.08756776997143,
			"height": 1.8117858011919452,
			"seed": 821918909,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 2
			},
			"boundElements": [],
			"updated": 1689632953043,
			"link": null,
			"locked": false,
			"startBinding": {
				"elementId": "8T8kOPYzdyleVXAXMA9Jx",
				"gap": 2.157107973548,
				"focus": 0.04598890752391342
			},
			"endBinding": {
				"elementId": "Owyg3Lrx-Ck-YkMryKkKe",
				"gap": 6.522063675119352,
				"focus": -0.050204141843542126
			},
			"lastCommittedPoint": null,
			"startArrowhead": null,
			"endArrowhead": "arrow",
			"points": [
				[
					0,
					0
				],
				[
					337.08756776997143,
					-1.8117858011919452
				]
			]
		},
		{
			"type": "rectangle",
			"version": 694,
			"versionNonce": 750646547,
			"isDeleted": false,
			"id": "Owyg3Lrx-Ck-YkMryKkKe",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": -91.28101969395141,
			"y": 5666.05474959408,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 250,
			"height": 204,
			"seed": 1198350685,
			"groupIds": [
				"tmV9DNWRhiU784HOmxnGa"
			],
			"frameId": null,
			"roundness": {
				"type": 3
			},
			"boundElements": [
				{
					"type": "text",
					"id": "EPtgFrIR"
				},
				{
					"id": "fqNaVqMAipjUvC-yXnMuI",
					"type": "arrow"
				},
				{
					"id": "fWGmaLzCl386roMxLS-a_",
					"type": "arrow"
				}
			],
			"updated": 1689632792672,
			"link": null,
			"locked": false
		},
		{
			"type": "text",
			"version": 263,
			"versionNonce": 2115361181,
			"isDeleted": false,
			"id": "EPtgFrIR",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": -64.00131846104125,
			"y": 5671.05474959408,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#ffec99",
			"width": 195.4405975341797,
			"height": 34.13312693498451,
			"seed": 156446141,
			"groupIds": [
				"tmV9DNWRhiU784HOmxnGa"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792672,
			"link": null,
			"locked": false,
			"fontSize": 27.306501547987608,
			"fontFamily": 1,
			"text": "Load Balancer",
			"rawText": "Load Balancer",
			"textAlign": "center",
			"verticalAlign": "top",
			"containerId": "Owyg3Lrx-Ck-YkMryKkKe",
			"originalText": "Load Balancer",
			"lineHeight": 1.25,
			"baseline": 24
		},
		{
			"type": "image",
			"version": 440,
			"versionNonce": 2047072947,
			"isDeleted": false,
			"id": "roJ0O-JSBNzn8Kd8b50hp",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": -27.321022091264012,
			"y": 5722.52372703803,
			"strokeColor": "transparent",
			"backgroundColor": "#b2f2bb",
			"width": 124.93718843469588,
			"height": 124.93718843469588,
			"seed": 964718109,
			"groupIds": [
				"tmV9DNWRhiU784HOmxnGa"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792672,
			"link": null,
			"locked": false,
			"status": "pending",
			"fileId": "45ef7f47e1949e06a16126834858c467db0bbb89",
			"scale": [
				1,
				1
			]
		},
		{
			"type": "rectangle",
			"version": 450,
			"versionNonce": 2093129213,
			"isDeleted": false,
			"id": "lECwMdtHB8Kxd0Jg6aOm2",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 360.0414027155332,
			"y": 5655.642899343733,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#a5d8ff",
			"width": 256,
			"height": 217,
			"seed": 176103155,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 3
			},
			"boundElements": [
				{
					"type": "text",
					"id": "l9eotsfs"
				},
				{
					"id": "fWGmaLzCl386roMxLS-a_",
					"type": "arrow"
				},
				{
					"id": "Zx7SsaFzzLvlApE1XbRYK",
					"type": "arrow"
				},
				{
					"id": "DiAWQJG6g4pp4h2Kbul7h",
					"type": "arrow"
				}
			],
			"updated": 1689632792672,
			"link": null,
			"locked": false
		},
		{
			"type": "text",
			"version": 188,
			"versionNonce": 482085971,
			"isDeleted": false,
			"id": "l9eotsfs",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 437.24942578682226,
			"y": 5660.642899343733,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 101.58395385742188,
			"height": 35,
			"seed": 560807997,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792672,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Servers",
			"rawText": "Servers",
			"textAlign": "center",
			"verticalAlign": "top",
			"containerId": "lECwMdtHB8Kxd0Jg6aOm2",
			"originalText": "Servers",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "image",
			"version": 351,
			"versionNonce": 639096413,
			"isDeleted": false,
			"id": "J34QVmPS",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 420.7338514963759,
			"y": 5709.688383027559,
			"strokeColor": "#000000",
			"backgroundColor": "transparent",
			"width": 125.19950786874757,
			"height": 125.19950786874757,
			"seed": 71499,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792672,
			"link": null,
			"locked": false,
			"status": "pending",
			"fileId": "394cb3fb5676e05e1ad183c501110a923dc2b863",
			"scale": [
				1,
				1
			]
		},
		{
			"type": "arrow",
			"version": 139,
			"versionNonce": 812602099,
			"isDeleted": false,
			"id": "fWGmaLzCl386roMxLS-a_",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 159.72014296169382,
			"y": 5761.0357029256265,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 198.11268404039097,
			"height": 1.2108480277556737,
			"seed": 1198418653,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 2
			},
			"boundElements": [],
			"updated": 1689632953045,
			"link": null,
			"locked": false,
			"startBinding": {
				"elementId": "Owyg3Lrx-Ck-YkMryKkKe",
				"gap": 1.001162655645203,
				"focus": -0.06080863748402687
			},
			"endBinding": {
				"elementId": "lECwMdtHB8Kxd0Jg6aOm2",
				"gap": 2.208575713448454,
				"focus": 0.046795017187036006
			},
			"lastCommittedPoint": null,
			"startArrowhead": null,
			"endArrowhead": "arrow",
			"points": [
				[
					0,
					0
				],
				[
					198.11268404039097,
					-1.2108480277556737
				]
			]
		},
		{
			"type": "rectangle",
			"version": 762,
			"versionNonce": 1961296573,
			"isDeleted": false,
			"id": "ZhxIMJwnw9B2_jhi29nlf",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 885.5574452293895,
			"y": 5661.452854352521,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 250,
			"height": 204,
			"seed": 1911186611,
			"groupIds": [
				"Dc6f5s3Om67ibTmC4PA3b"
			],
			"frameId": null,
			"roundness": {
				"type": 3
			},
			"boundElements": [
				{
					"type": "text",
					"id": "Uv6tOyou"
				},
				{
					"id": "Zx7SsaFzzLvlApE1XbRYK",
					"type": "arrow"
				},
				{
					"id": "xp6SdJaFfM4Xdg3qea4Vf",
					"type": "arrow"
				}
			],
			"updated": 1689632792672,
			"link": null,
			"locked": false
		},
		{
			"type": "text",
			"version": 329,
			"versionNonce": 1930253203,
			"isDeleted": false,
			"id": "Uv6tOyou",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 912.8371464622996,
			"y": 5666.452854352521,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#ffec99",
			"width": 195.4405975341797,
			"height": 34.13312693498451,
			"seed": 725702227,
			"groupIds": [
				"Dc6f5s3Om67ibTmC4PA3b"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792672,
			"link": null,
			"locked": false,
			"fontSize": 27.306501547987608,
			"fontFamily": 1,
			"text": "Load Balancer",
			"rawText": "Load Balancer",
			"textAlign": "center",
			"verticalAlign": "top",
			"containerId": "ZhxIMJwnw9B2_jhi29nlf",
			"originalText": "Load Balancer",
			"lineHeight": 1.25,
			"baseline": 24
		},
		{
			"type": "image",
			"version": 506,
			"versionNonce": 231842589,
			"isDeleted": false,
			"id": "EylVD6ZjnMiGlnwu03LCD",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 949.5174428320768,
			"y": 5717.921831796471,
			"strokeColor": "transparent",
			"backgroundColor": "#b2f2bb",
			"width": 124.93718843469588,
			"height": 124.93718843469588,
			"seed": 738792435,
			"groupIds": [
				"Dc6f5s3Om67ibTmC4PA3b"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [
				{
					"id": "jvBdh1-RImUGuZymWngV6",
					"type": "arrow"
				}
			],
			"updated": 1689632792672,
			"link": null,
			"locked": false,
			"status": "pending",
			"fileId": "45ef7f47e1949e06a16126834858c467db0bbb89",
			"scale": [
				1,
				1
			]
		},
		{
			"type": "arrow",
			"version": 113,
			"versionNonce": 867218387,
			"isDeleted": false,
			"id": "Zx7SsaFzzLvlApE1XbRYK",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 617.7703383586489,
			"y": 5759.827127212176,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#a5d8ff",
			"width": 264.6780812452067,
			"height": 1.2085757134482265,
			"seed": 2105734035,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 2
			},
			"boundElements": [],
			"updated": 1689632953048,
			"link": null,
			"locked": false,
			"startBinding": {
				"elementId": "lECwMdtHB8Kxd0Jg6aOm2",
				"gap": 1.728935643115733,
				"focus": -0.03413320333903945
			},
			"endBinding": {
				"elementId": "ZhxIMJwnw9B2_jhi29nlf",
				"gap": 3.1090256255338886,
				"focus": 0.05283449898082886
			},
			"lastCommittedPoint": null,
			"startArrowhead": null,
			"endArrowhead": "arrow",
			"points": [
				[
					0,
					0
				],
				[
					264.6780812452067,
					-1.2085757134482265
				]
			]
		},
		{
			"type": "arrow",
			"version": 715,
			"versionNonce": 1268652819,
			"isDeleted": false,
			"id": "DiAWQJG6g4pp4h2Kbul7h",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 482.2865013528185,
			"y": 5876.090091758313,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#a5d8ff",
			"width": 1.5801974913188133,
			"height": 84.30101861514777,
			"seed": 1639375197,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 2
			},
			"boundElements": [],
			"updated": 1689632953051,
			"link": null,
			"locked": false,
			"startBinding": {
				"elementId": "lECwMdtHB8Kxd0Jg6aOm2",
				"gap": 3.447192414580274,
				"focus": 0.060777630607132364
			},
			"endBinding": {
				"elementId": "Er63Fz6KaibE8dJ-pY7T4",
				"gap": 6.6517179936843185,
				"focus": -0.031451908382685335
			},
			"lastCommittedPoint": null,
			"startArrowhead": null,
			"endArrowhead": "arrow",
			"points": [
				[
					0,
					0
				],
				[
					1.5801974913188133,
					84.30101861514777
				]
			]
		},
		{
			"type": "rectangle",
			"version": 390,
			"versionNonce": 359726237,
			"isDeleted": false,
			"id": "Er63Fz6KaibE8dJ-pY7T4",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 368.3728063305273,
			"y": 5967.0428283671445,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#ffec99",
			"width": 242,
			"height": 163,
			"seed": 118275357,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 3
			},
			"boundElements": [
				{
					"type": "text",
					"id": "npYEujtN"
				},
				{
					"id": "DiAWQJG6g4pp4h2Kbul7h",
					"type": "arrow"
				},
				{
					"id": "EPw2C55npe4PioXW4_hCH",
					"type": "arrow"
				}
			],
			"updated": 1689632864795,
			"link": null,
			"locked": false
		},
		{
			"type": "text",
			"version": 322,
			"versionNonce": 1939892147,
			"isDeleted": false,
			"id": "npYEujtN",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 389.62285210689447,
			"y": 5972.0428283671445,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#a5d8ff",
			"width": 199.49990844726562,
			"height": 35,
			"seed": 874790995,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632864795,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Key Generator",
			"rawText": "Key Generator",
			"textAlign": "center",
			"verticalAlign": "top",
			"containerId": "Er63Fz6KaibE8dJ-pY7T4",
			"originalText": "Key Generator",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "image",
			"version": 490,
			"versionNonce": 81393555,
			"isDeleted": false,
			"id": "rRT0dtrC",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 440.50741820601934,
			"y": 6022.613968280697,
			"strokeColor": "#000000",
			"backgroundColor": "transparent",
			"width": 89.46657305190386,
			"height": 89.46657305190386,
			"seed": 2718,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632868189,
			"link": null,
			"locked": false,
			"status": "pending",
			"fileId": "d43bc7222c0170ee0ae3dbdc7e4de74f30c1d8c5",
			"scale": [
				1,
				1
			]
		},
		{
			"type": "rectangle",
			"version": 201,
			"versionNonce": 1232695827,
			"isDeleted": false,
			"id": "16XR9LwJZipDJ8WpI13Pj",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 371.0517433939257,
			"y": 6259.593861767751,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#ffc9c9",
			"width": 249,
			"height": 177,
			"seed": 1902501267,
			"groupIds": [
				"lVfTN3wmmjdXR4_XRaJdc"
			],
			"frameId": null,
			"roundness": {
				"type": 3
			},
			"boundElements": [
				{
					"type": "text",
					"id": "aMS4irLG"
				},
				{
					"id": "Zaw39afzNbnufuD8py1X0",
					"type": "arrow"
				},
				{
					"id": "9duigy2YDbQXK7oj74X65",
					"type": "arrow"
				},
				{
					"id": "EPw2C55npe4PioXW4_hCH",
					"type": "arrow"
				}
			],
			"updated": 1689632792672,
			"link": null,
			"locked": false
		},
		{
			"type": "text",
			"version": 167,
			"versionNonce": 90415283,
			"isDeleted": false,
			"id": "aMS4irLG",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 444.647766953496,
			"y": 6264.593861767751,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#ffc9c9",
			"width": 101.80795288085938,
			"height": 35,
			"seed": 1318644189,
			"groupIds": [
				"lVfTN3wmmjdXR4_XRaJdc"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632887419,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Key DB",
			"rawText": "Key DB",
			"textAlign": "center",
			"verticalAlign": "top",
			"containerId": "16XR9LwJZipDJ8WpI13Pj",
			"originalText": "Key DB",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "image",
			"version": 431,
			"versionNonce": 1279768499,
			"isDeleted": false,
			"id": "efgCkE1w",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 447.44026913336404,
			"y": 6316.1938958662095,
			"strokeColor": "#000000",
			"backgroundColor": "#ffc9c9",
			"width": 95.86626996439837,
			"height": 95.86626996439837,
			"seed": 15611,
			"groupIds": [
				"lVfTN3wmmjdXR4_XRaJdc"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792672,
			"link": null,
			"locked": false,
			"status": "pending",
			"fileId": "fc6d4bfe4c3767f2a8be612b648adf5d57d8aca2",
			"scale": [
				1,
				1
			]
		},
		{
			"type": "rectangle",
			"version": 341,
			"versionNonce": 1884783933,
			"isDeleted": false,
			"id": "CJkjwpFVjcMizA5ifMU3O",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": -35.339302642601126,
			"y": 6255.591380219664,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#ffc9c9",
			"width": 249,
			"height": 177,
			"seed": 229873459,
			"groupIds": [
				"OrKBAfhj8cVBuRQbQj8UM"
			],
			"frameId": null,
			"roundness": {
				"type": 3
			},
			"boundElements": [
				{
					"type": "text",
					"id": "dSGgRYEk"
				},
				{
					"id": "9duigy2YDbQXK7oj74X65",
					"type": "arrow"
				}
			],
			"updated": 1689632892428,
			"link": null,
			"locked": false
		},
		{
			"type": "text",
			"version": 320,
			"versionNonce": 1700641907,
			"isDeleted": false,
			"id": "dSGgRYEk",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 24.67673434470356,
			"y": 6260.591380219664,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#ffc9c9",
			"width": 128.96792602539062,
			"height": 70,
			"seed": 205535443,
			"groupIds": [
				"OrKBAfhj8cVBuRQbQj8UM"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632897692,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Key DB\n(Standby)",
			"rawText": "Key DB\n(Standby)",
			"textAlign": "center",
			"verticalAlign": "top",
			"containerId": "CJkjwpFVjcMizA5ifMU3O",
			"originalText": "Key DB\n(Standby)",
			"lineHeight": 1.25,
			"baseline": 60
		},
		{
			"type": "image",
			"version": 610,
			"versionNonce": 1398479357,
			"isDeleted": false,
			"id": "3674EcwixcVBr9jIP8o4i",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 44.03181148981565,
			"y": 6333.069533068972,
			"strokeColor": "#000000",
			"backgroundColor": "#ffc9c9",
			"width": 82.44462219599528,
			"height": 82.44462219599528,
			"seed": 501363,
			"groupIds": [
				"OrKBAfhj8cVBuRQbQj8UM"
			],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632892429,
			"link": null,
			"locked": false,
			"status": "pending",
			"fileId": "fc6d4bfe4c3767f2a8be612b648adf5d57d8aca2",
			"scale": [
				1,
				1
			]
		},
		{
			"type": "rectangle",
			"version": 512,
			"versionNonce": 929933043,
			"isDeleted": false,
			"id": "zQnF1_rdnLEqXFeurPUxd",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 892.2451610627963,
			"y": 5946.929912167673,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#a5d8ff",
			"width": 256,
			"height": 217,
			"seed": 331485629,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 3
			},
			"boundElements": [
				{
					"type": "text",
					"id": "fVGHw8Np"
				},
				{
					"id": "yTLdV6HvzUt9RRlxU3hYp",
					"type": "arrow"
				},
				{
					"id": "Rb5PMVabbOYFBq30MHbSM",
					"type": "arrow"
				}
			],
			"updated": 1689632792673,
			"link": null,
			"locked": false
		},
		{
			"type": "text",
			"version": 266,
			"versionNonce": 2094708157,
			"isDeleted": false,
			"id": "fVGHw8Np",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 941.355192190726,
			"y": 5951.929912167673,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 157.77993774414062,
			"height": 35,
			"seed": 1551028765,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792673,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "DB Servers",
			"rawText": "DB Servers",
			"textAlign": "center",
			"verticalAlign": "top",
			"containerId": "zQnF1_rdnLEqXFeurPUxd",
			"originalText": "DB Servers",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "image",
			"version": 381,
			"versionNonce": 1537364115,
			"isDeleted": false,
			"id": "gly8tJQG",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 947.0954829055665,
			"y": 5998.218088718742,
			"strokeColor": "#000000",
			"backgroundColor": "transparent",
			"width": 142.35336828466524,
			"height": 142.35336828466524,
			"seed": 51112,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [
				{
					"id": "yTLdV6HvzUt9RRlxU3hYp",
					"type": "arrow"
				}
			],
			"updated": 1689632792673,
			"link": null,
			"locked": false,
			"status": "pending",
			"fileId": "d7704ead31974a1c8e641d2788cecf836ef139f4",
			"scale": [
				1,
				1
			]
		},
		{
			"type": "rectangle",
			"version": 147,
			"versionNonce": 1658254877,
			"isDeleted": false,
			"id": "yNftLeH0kIrDSPrDcy2gN",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 886.0286833787616,
			"y": 6243.332980925322,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#ffec99",
			"width": 265.4503669750907,
			"height": 219.2202468839232,
			"seed": 1625137245,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 3
			},
			"boundElements": [
				{
					"id": "yTLdV6HvzUt9RRlxU3hYp",
					"type": "arrow"
				},
				{
					"id": "Zaw39afzNbnufuD8py1X0",
					"type": "arrow"
				}
			],
			"updated": 1689632792673,
			"link": null,
			"locked": false
		},
		{
			"type": "image",
			"version": 350,
			"versionNonce": 315672115,
			"isDeleted": false,
			"id": "gepeyOnY",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 960.0123475660562,
			"y": 6301.623103841751,
			"strokeColor": "#000000",
			"backgroundColor": "transparent",
			"width": 140.59809864608468,
			"height": 140.59809864608468,
			"seed": 37318,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632792673,
			"link": null,
			"locked": false,
			"status": "pending",
			"fileId": "1fe30744631b3cb11961d1770f373a2104529cd8",
			"scale": [
				1,
				1
			]
		},
		{
			"type": "text",
			"version": 122,
			"versionNonce": 1412180605,
			"isDeleted": false,
			"id": "lWaI0ilj",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 912.8719789155687,
			"y": 6253.772040300746,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#a5d8ff",
			"width": 210.6999053955078,
			"height": 35,
			"seed": 1765903539,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [
				{
					"id": "yTLdV6HvzUt9RRlxU3hYp",
					"type": "arrow"
				}
			],
			"updated": 1689632792673,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "Cleanup Service",
			"rawText": "Cleanup Service",
			"textAlign": "left",
			"verticalAlign": "top",
			"containerId": null,
			"originalText": "Cleanup Service",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "arrow",
			"version": 76,
			"versionNonce": 1944029139,
			"isDeleted": false,
			"id": "jvBdh1-RImUGuZymWngV6",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 1006.823513294393,
			"y": 5948.056730020446,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#a5d8ff",
			"width": 1.491294196489207,
			"height": 87.98635759286663,
			"seed": 761299997,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 2
			},
			"boundElements": [],
			"updated": 1689632792673,
			"link": null,
			"locked": false,
			"startBinding": null,
			"endBinding": {
				"elementId": "EylVD6ZjnMiGlnwu03LCD",
				"focus": 0.03653104925648734,
				"gap": 17.211352196412463
			},
			"lastCommittedPoint": null,
			"startArrowhead": null,
			"endArrowhead": "arrow",
			"points": [
				[
					0,
					0
				],
				[
					1.491294196489207,
					-87.98635759286663
				]
			]
		},
		{
			"type": "arrow",
			"version": 83,
			"versionNonce": 1637602013,
			"isDeleted": false,
			"id": "yTLdV6HvzUt9RRlxU3hYp",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 1011.3624651650052,
			"y": 6241.841686728832,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#a5d8ff",
			"width": 1.8530184154892595,
			"height": 80.52988661042036,
			"seed": 1677532349,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 2
			},
			"boundElements": [],
			"updated": 1689632792673,
			"link": null,
			"locked": false,
			"startBinding": {
				"elementId": "lWaI0ilj",
				"focus": -0.061541240608755016,
				"gap": 11.930353571913656
			},
			"endBinding": {
				"elementId": "gly8tJQG",
				"focus": 0.149390067771554,
				"gap": 20.740343115004634
			},
			"lastCommittedPoint": null,
			"startArrowhead": null,
			"endArrowhead": "arrow",
			"points": [
				[
					0,
					0
				],
				[
					-1.8530184154892595,
					-80.52988661042036
				]
			]
		},
		{
			"type": "arrow",
			"version": 146,
			"versionNonce": 1692545405,
			"isDeleted": false,
			"id": "Rb5PMVabbOYFBq30MHbSM",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 607.1566686352676,
			"y": 5869.018137606514,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#a5d8ff",
			"width": 283.34589733296207,
			"height": 83.51247500340105,
			"seed": 2035704093,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 2
			},
			"boundElements": [],
			"updated": 1689632953058,
			"link": null,
			"locked": false,
			"startBinding": null,
			"endBinding": {
				"elementId": "zQnF1_rdnLEqXFeurPUxd",
				"gap": 1.7425950945666955,
				"focus": 0.44218700610726414
			},
			"lastCommittedPoint": null,
			"startArrowhead": null,
			"endArrowhead": "arrow",
			"points": [
				[
					0,
					0
				],
				[
					283.34589733296207,
					83.51247500340105
				]
			]
		},
		{
			"type": "arrow",
			"version": 86,
			"versionNonce": 1917497939,
			"isDeleted": false,
			"id": "Zaw39afzNbnufuD8py1X0",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 881.554800789294,
			"y": 6344.7409862865925,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#a5d8ff",
			"width": 259.4851901891336,
			"height": 0,
			"seed": 2127586675,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 2
			},
			"boundElements": [],
			"updated": 1689632953053,
			"link": null,
			"locked": false,
			"startBinding": {
				"elementId": "yNftLeH0kIrDSPrDcy2gN",
				"gap": 4.473882589467621,
				"focus": 0.07482993197278476
			},
			"endBinding": {
				"elementId": "16XR9LwJZipDJ8WpI13Pj",
				"gap": 2.017867206234655,
				"focus": -0.03788559865716003
			},
			"lastCommittedPoint": null,
			"startArrowhead": null,
			"endArrowhead": "arrow",
			"points": [
				[
					0,
					0
				],
				[
					-259.4851901891336,
					0
				]
			]
		},
		{
			"type": "arrow",
			"version": 189,
			"versionNonce": 2079051571,
			"isDeleted": false,
			"id": "9duigy2YDbQXK7oj74X65",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "dashed",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 364.0757146075163,
			"y": 6354.3982956963555,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#a5d8ff",
			"width": 143.68112795906143,
			"height": 0.6984396021343855,
			"seed": 295337853,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 2
			},
			"boundElements": [],
			"updated": 1689632953056,
			"link": null,
			"locked": false,
			"startBinding": {
				"elementId": "16XR9LwJZipDJ8WpI13Pj",
				"gap": 6.976028786409415,
				"focus": -0.07785819392165143
			},
			"endBinding": {
				"elementId": "CJkjwpFVjcMizA5ifMU3O",
				"gap": 6.733889291055959,
				"focus": 0.10067361781234664
			},
			"lastCommittedPoint": null,
			"startArrowhead": null,
			"endArrowhead": "arrow",
			"points": [
				[
					0,
					0
				],
				[
					-143.68112795906143,
					-0.6984396021343855
				]
			]
		},
		{
			"type": "arrow",
			"version": 123,
			"versionNonce": 95058323,
			"isDeleted": false,
			"id": "EPw2C55npe4PioXW4_hCH",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 486.26949014131674,
			"y": 6133.216182175718,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#a5d8ff",
			"width": 0.5770905288287622,
			"height": 123.53844651800773,
			"seed": 976182067,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 2
			},
			"boundElements": [],
			"updated": 1689632953053,
			"link": null,
			"locked": false,
			"startBinding": {
				"elementId": "Er63Fz6KaibE8dJ-pY7T4",
				"gap": 3.1733538085736654,
				"focus": 0.028829061539736238
			},
			"endBinding": {
				"elementId": "16XR9LwJZipDJ8WpI13Pj",
				"gap": 2.839233074026197,
				"focus": -0.06627379338208603
			},
			"lastCommittedPoint": null,
			"startArrowhead": null,
			"endArrowhead": "arrow",
			"points": [
				[
					0,
					0
				],
				[
					0.5770905288287622,
					123.53844651800773
				]
			]
		},
		{
			"type": "rectangle",
			"version": 782,
			"versionNonce": 1408929597,
			"isDeleted": false,
			"id": "rpQWJh45eEWgpnAiq9Fba",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 887.6029606011323,
			"y": 5286.472911115005,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#a5d8ff",
			"width": 222,
			"height": 195,
			"seed": 127801213,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 3
			},
			"boundElements": [
				{
					"type": "text",
					"id": "bKQHQEbp"
				},
				{
					"id": "xp6SdJaFfM4Xdg3qea4Vf",
					"type": "arrow"
				},
				{
					"id": "FeW482wKiBQjXoqcHggme",
					"type": "arrow"
				}
			],
			"updated": 1689632845966,
			"link": null,
			"locked": false
		},
		{
			"type": "text",
			"version": 516,
			"versionNonce": 1780928275,
			"isDeleted": false,
			"id": "bKQHQEbp",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 969.6509647515229,
			"y": 5291.472911115005,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#b2f2bb",
			"width": 57.90399169921875,
			"height": 35,
			"seed": 172498909,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [],
			"updated": 1689632845966,
			"link": null,
			"locked": false,
			"fontSize": 28,
			"fontFamily": 1,
			"text": "CDN",
			"rawText": "CDN",
			"textAlign": "center",
			"verticalAlign": "top",
			"containerId": "rpQWJh45eEWgpnAiq9Fba",
			"originalText": "CDN",
			"lineHeight": 1.25,
			"baseline": 25
		},
		{
			"type": "arrow",
			"version": 529,
			"versionNonce": 2041626995,
			"isDeleted": false,
			"id": "xp6SdJaFfM4Xdg3qea4Vf",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 1008.0834390933562,
			"y": 5659.020118417296,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#ffec99",
			"width": 1.971201847657312,
			"height": 175.35127223216114,
			"seed": 197701181,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 2
			},
			"boundElements": [],
			"updated": 1689632953048,
			"link": null,
			"locked": false,
			"startBinding": {
				"elementId": "ZhxIMJwnw9B2_jhi29nlf",
				"gap": 2.432735935225537,
				"focus": -0.010305714915656837
			},
			"endBinding": {
				"elementId": "86QqfkTe",
				"gap": 26.737702676514346,
				"focus": -0.09992893757520457
			},
			"lastCommittedPoint": null,
			"startArrowhead": null,
			"endArrowhead": "arrow",
			"points": [
				[
					0,
					0
				],
				[
					-1.971201847657312,
					-175.35127223216114
				]
			]
		},
		{
			"type": "arrow",
			"version": 344,
			"versionNonce": 1991340243,
			"isDeleted": false,
			"id": "FeW482wKiBQjXoqcHggme",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 899.2940357225057,
			"y": 5484.821112798183,
			"strokeColor": "#1e1e1e",
			"backgroundColor": "#ffec99",
			"width": 289.1547786942589,
			"height": 187.34619087174633,
			"seed": 1575479421,
			"groupIds": [],
			"frameId": null,
			"roundness": {
				"type": 2
			},
			"boundElements": [],
			"updated": 1689632953060,
			"link": null,
			"locked": false,
			"startBinding": {
				"elementId": "rpQWJh45eEWgpnAiq9Fba",
				"gap": 3.3482016831773933,
				"focus": -0.21547290646152834
			},
			"endBinding": null,
			"lastCommittedPoint": null,
			"startArrowhead": null,
			"endArrowhead": "arrow",
			"points": [
				[
					0,
					0
				],
				[
					-289.1547786942589,
					187.34619087174633
				]
			]
		},
		{
			"type": "image",
			"version": 499,
			"versionNonce": 295608765,
			"isDeleted": false,
			"id": "86QqfkTe",
			"fillStyle": "hachure",
			"strokeWidth": 1,
			"strokeStyle": "solid",
			"roughness": 1,
			"opacity": 100,
			"angle": 0,
			"x": 933.9352497482822,
			"y": 5327.691150210208,
			"strokeColor": "#000000",
			"backgroundColor": "transparent",
			"width": 129.23999329841172,
			"height": 129.23999329841172,
			"seed": 42934,
			"groupIds": [],
			"frameId": null,
			"roundness": null,
			"boundElements": [
				{
					"id": "xp6SdJaFfM4Xdg3qea4Vf",
					"type": "arrow"
				}
			],
			"updated": 1689632848445,
			"link": null,
			"locked": false,
			"status": "pending",
			"fileId": "4d80284b2779b752552bd7f9da162b7b925807b2",
			"scale": [
				1,
				1
			]
		}
	],
	"appState": {
		"theme": "light",
		"viewBackgroundColor": "#f5faff",
		"currentItemStrokeColor": "#1e1e1e",
		"currentItemBackgroundColor": "#d0bfff",
		"currentItemFillStyle": "hachure",
		"currentItemStrokeWidth": 1,
		"currentItemStrokeStyle": "dashed",
		"currentItemRoughness": 1,
		"currentItemOpacity": 100,
		"currentItemFontFamily": 1,
		"currentItemFontSize": 28,
		"currentItemTextAlign": "left",
		"currentItemStartArrowhead": null,
		"currentItemEndArrowhead": "arrow",
		"scrollX": 878.4495917316661,
		"scrollY": -699.2386180916104,
		"zoom": {
			"value": 0.469148342013359
		},
		"currentItemRoundness": "round",
		"gridSize": null,
		"currentStrokeOptions": null,
		"previousGridSize": null
	},
	"files": {}
}
```
%%