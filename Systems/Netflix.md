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


