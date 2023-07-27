The Netflix Open Connect program provides opportunities for ISP partners to improve their customers' Netflix user experience by localizing Netflix traffic and minimizing the delivery of traffic that is served over a transit provider.

There are two main components of the program, which are architected in partnership with ISPs to provide maximum benefit in each individual situation: embedded Open Connect Appliances and settlement-free interconnection (SFI).

### Embedded Open Connect Appliances (OCAs)

Open Connect Appliances can be embedded in your ISP network. Embedded OCAs have the same capabilities as the OCAs that we use in our 60+ global data centers, and they are provided to qualifying ISP partners at no charge. Each embedded OCA deployment will offload a substantial amount of Netflix content traffic from peering or transport circuits. Multiple physical deployments can be distributed or clustered on a geographic or network basis to maximize local offload.

Netflix provides:

- Network architecture and technical turn-up expertise
- Ongoing monitoring and issue resolution
- Partner support

ISP partners work with Netflix to configure BGP sessions with the OCAs to steer traffic, and the appliances require a small amount of data center rack space, power, and connectivity.

If you have [substantial Netflix traffic](https://openconnect.netflix.com/en/deployment-guide/requirements-for-deploying-embedded-appliances/) destined to your ISP customers, deploying embedded OCAs is usually the most beneficial option. However, embedded OCAs are not always deployed, depending on your traffic levels, data center limitations, or other factors.

The [OCA Deployment Guide](https://openconnect.zendesk.com/hc/en-us/sections/360006922832) provides deployment details. If you are interested in OCA deployment at your ISP, please fill out our [OCA request form](https://openconnect.netflix.com/en/deployment-guide/appliance-request/).

### Settlement-free interconnection (SFI)

Connect via direct Private Network Interconnect (PNI) or IXP-based SFI peering to Netflix Open Connect Appliances in our data centers. Peering alone can be beneficial.

If you deploy embedded OCAs, you also set up SFI peering for additional resiliency and to enable nightly content fills and updates.

Netflix has the ability to interconnect at a number of global data center facilities and public Internet Exchange fabrics as listed on our [Peering Locations](https://openconnect.netflix.com/en/peering/#locations) page. We openly peer with any network at IXP locations where we are mutually present and we consider private interconnection as appropriate. If you are interested in interconnection, please review the information on the [Peering Locations](https://openconnect.netflix.com/en/peering/#locations) page.

ISPs who do not currently participate in public peering might want to consider that a single IX port can support multiple peering sessions, providing direct access to various content, cloud, and network providers. In addition to Netflix, many large organizations such as Akamai, Amazon, Facebook, and Google/YouTube widely participate in public peering and combine to deliver a substantial percentage of traffic to a typical ISP.

From a connectivity standpoint, IX ports can be reached locally in a data center or via transport. We recommend [http://peeringdb.com](http://peeringdb.com) as a detailed source of information that can help you find an IX that best meets your needs.

### What is the difference between Netflix's Open Connect and a traditional Content Delivery Network (CDN)?
Netflix Open Connect is a purpose-built **[[Content Delivery Network (CDN)]]** responsible for serving Netflix's video traffic. Around 95% of the traffic globally is delivered via direct connections between Open Connect and the ISPs their customers use to access the internet.

Currently, they have Open Connect Appliances (OCAs) in over 1000 separate locations around the world. In case of issues, Open Connect Appliances (OCAs) can failover, and the traffic can be re-routed to Netflix servers.

Additionally, we can use [Adaptive bitrate streaming](https://en.wikipedia.org/wiki/Adaptive_bitrate_streaming) protocols such as [HTTP Live Streaming (HLS)](https://en.wikipedia.org/wiki/HTTP_Live_Streaming) which is designed for reliability and it dynamically adapts to network conditions by optimizing playback for the available speed of the connections.

Lastly, for playing the video from where the user left off, we can simply use the `offset` property we stored in the `views` table to retrieve the scene chunk at that particular timestamp and resume the playback for the user.