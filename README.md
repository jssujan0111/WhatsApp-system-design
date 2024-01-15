# System Design Case Study
                        Topic: WhatsApp 
# Introduction:  
Case study is about WhatsApp . In today’s world, WhatsApp is a very popular messaging application. As per as the latest data, WhatsApp has the 2.8 billion monthly active users.

# Requirements

## Functional requirement: 
1.	The system should support one-on-one and group conversations between users.(conversation service)

2.	The system should support sharing media files such as images, videos and audios.(sharing/static content)

3.	The system should support the persistent storage of chat  messages when a user is offline until the successful delivery of messages.

4.	The system should be support push notification system when a user is offline. System should be able to notify offline users when their status became online.

## Non functional requirement::
1.	Low latency: User should be able to send and receive messages with low latency.

2.	Concistency:  Messages should be delivered in order they were sent. Moreover, users should be able to see the same chat history with on all of their devices.

3.	Availability: The system should be highly available

4.	Security: The system should support end-to-end  encryption. End-to-end encryption means that the system ensures that only two communicating  parties can see the content of messages.

## Extended requirement
1.	Multi devices support:  Allow users to access their account from multiple devices semaltineously.

2.	File and location sharing: Extend file sharing capabilities to support a variety of file formats ans enable users to share their live location.

3.	Customization: Allow  users to their app settings , themes and notification prefrences.

4.	Backup and Restored: Allow users to backup the chat history and media files and restore them when needed.

# Estimation and Constraints
## Traffic
Most of the chat messaging app is read and write heavy application. As per as latest news , WhatsApp has 2.8 billion monthly active users. But as our assignment requirement 300 million active users. 

So we have 300/30 = 10 million daily active users. We assume that a user can send 100 messages on average a day. 

As messaging app read/write ratio is about 50/50.

**10million * 100 messages   =  1 billion**

Message can also contain media files such as media files of variety format, images, videos and audios(voice messages).

**1 billion * 10 percent = 100 million**

What would be requests per second for our system?

**1 billion / (24hrs*3600s) ~= 12k request per second.**

## Storage::
If we assume that each message is about 100bytes, we will require about 100GB of database storage everyday.

**1billion * 100bytes =~  100GB /day**

As per our requirement , we also know that around 10 percent of daily active messages(100million) are media files. If we assume each message is 50kb on average, we will require 10TB of data storage a day.

**After  10 years , we will require 37PB of storage.**

**(10 TB + 0.1 TB) * 10years * 365 days =~ 37B /10 years**

## Bandwidth::

As our system is handling 10.1 TB of ingress , we will require minimum bandwidth of 120 MB per second.

**10.1 TB / ( 24 hrs * 3600s) =~ 120MB /per sec.**

## High Label Estimation
| Type	 | Estimate |
|--|--|
| Daily Active Users (DAU) | 100 million |
|Request Per Second (RPS) |12k/s | 
|Storage (Per Day)	 | 10.1 TB|
| Storage (10 years)| 37PB|
|Bandwidth	 |120MB /s |
		

# Data Model Design
This is the general data model which reflects our requirement.

![enter image description here](https://i.ibb.co/sqZWx6s/entity.png)
## Database
While our data model seems quite relational, we don't necessarily need to store everything in a single database, as this can limit our scalability and quickly become a bottleneck.

We will split the data between different services each having ownership over a particular table. Then we can use a relational database such as PostgreSQL or a distributed NoSQL database such as Apache Cassandra for our use case.

## API Design 
Here some basic API endpoints which reflects our requirement..
|Name	  |Method	  |Path	 |  Param	| Return  |
|--|--|--|--|--|
| Chats	 |GET	  | /chats	 |usersID	 |  Chats[] |
| Groups	 | GET	 | /groups |userID	 | Groups[]  |
| Messages	 | GET	 | /messages	 |chatID	 |  Messages[] |
| Group messages | GET	 | /messages | groupID	|  Groups msg[] |
| Send Message | POST	 | /messages |chatID	 |  New Message |
| Send Message | POST	 | 	/messages | GroupID	| New Message  |
| Join/leave Group |POST| /groups | GroupId,userId| Boolean  |


# High Label Design
Which Architecture should we choose?

1.	Monolithic 

2.	Microservice

We will be using microservices architecture since it will make easier to horizontally scale and decouple our services. Each service will have it’s own data model.  

Now, We can try to devide our system into core services.
User service:  This is an HTTP based service that handles user related concerns such as authentication and  user information.

Chat Service: This service will use Web sockets and Establish connections with the client to handle chat and group message related functionality.  To keep track of user connection ( offline or online), we can use cache mechanism .

Notification service:  To send notification to the users, we can use Apache Kafka.

Media Service:  This service will handle the media files ( images, videos, voices and files etc ).

Presence service: this service will keep tracking of message seen and unseen.
## Inter service communication 

Since our architecture is microservices-based, services will be communicating with each other as well. Generally, REST or HTTP performs well but we can further improve the performance using gRPC which is more lightweight and efficient. 

Service discovery is another thing we will have to take into account. We can also use a service mesh that enables managed, observable, and secure communication between individual services.

## Real Time messaging
How do we efficiently send and receive messages?  There are so many options..

1.	Pull Model: The client can periodically send an HTTP request to servers to check if there are any new messages. This can be achieved via something like Long polling.

2. Push Model:  The client opens a long-lived connection with the server and once new data is available it will be pushed to the client. We can use Web-Sockets or Server-Sent Events (SSE) for this.

The pull model approach is not scalable as it will create unnecessary request overhead on our servers and most of the time the response will be empty, thus wasting our resources. 

To minimize latency, using the push model with Web-Sockets is a better choice because then we can push data to the client once it's available without any delay given the connection is open with the client. Also, Web-Sockets provide full-duplex communication, unlike Server-Sent Events (SSE) which are only unidirectional.
# Last Seen
To implement the last seen functionality, we can use a heartbeat mechanism, where the client can periodically ping the servers indicating its liveness. Since this needs to be as low overhead as possible, we can store the last active timestamp in the cache as follows:

|Key	  |Value  |
|--|--|
| User A	 | 2024-01-04T17:09:19.574Z  |
| User B	 | '2024-01-04T17:10:44.378Z' |
| User C |  	'2024-01-04T17:11:32.216Z'|


This will give us the last time the user was active. This functionality will be handled by the presence service combined with Redis or Memcached as our cache.

# Notifications
Once a message is sent in a chat or a group, we will first check if the recipient is active or not, we can get this information by taking the user's active connection and last seen into consideration.

If the recipient is not active, the chat service will add an event to a message queue with additional metadata such as the client's device platform which will be used to route the notification to the correct platform later on.

The notification service will then consume the event from the message queue and forward the request to Firebase Cloud Messaging (FCM) or Apple Push Notification Service (APNS) based on the client's device platform (Android, iOS, web, etc). We can also add support for email and SMS.

Why are we using a message queue?

Since most message queues provide best-effort ordering which ensures that messages are generally delivered in the same order as they're sent and that a message is delivered at least once which is an important part of our service functionality.

While this seems like a classic publish-subscribe use case, it is actually not as mobile devices and browsers each have their own way of handling push notifications. Usually, notifications are handled externally via Firebase Cloud Messaging (FCM) or Apple Push Notification Service (APNS) unlike message fan-out which we commonly see in backend services. We can use something like Amazon SQS or RabbitMQ to support this functionality.

# Design
 we have defined our some core components. Now , We can design our first draft of our system design..

![enter image description here](https://i.ibb.co/23B03LM/draft-design.png)

# Detailed Design
## Data Partitioning::
To scale out our databases we will need to partition our data. Horizontal partitioning (aka Sharding) can be a good first step.
 We can use partitions schemes such as:
 
 ● Hash-Based Partitioning 
 
● List-Based Partitioning

 ● Range Based Partitioning
 
 ● Composite Partitioning
 
 The above approaches can still cause uneven data and load distribution, we can solve this using Consistent hashing.

	
## caching
In a messaging application, we have to be careful about using cache as our users expect the latest data, but many users will be requesting the same messages, especially in a group chat. So, to prevent usage spikes from our resources we can cache older messages.

Some group chats can have thousands of messages and sending that over the network will be really inefficient, to improve efficiency we can add pagination to our system APIs. This decision will be helpful for users with limited network bandwidth as they won't have to retrieve old messages unless requested.

**Which cache eviction policy to use?**

We can use solutions like Redis or Memcached and cache 20% of the daily traffic but what kind of cache eviction policy would best fit our needs?
Least Recently Used (LRU) can be a good policy for our system. In this policy, we discard the least recently used key first.

**How to handle cache miss?** 
Whenever there is a cache miss, our servers can hit the database directly and update the cache with the new entries.

## Media Access and Storage
 As we know, most of our storage space will be used for storing media files such as images, videos, or other files. Our media service will be handling both access and storage of the user media files. But where can we store files at scale? Well, object storage is what we're looking for. Object stores break data files up into pieces called objects. It then stores those objects in a single repository, which can be spread out across multiple networked systems. We can also use distributed file storage such as HDFS or GlusterFS.

## Content Delivery Network
 Content Delivery Network (CDN) 
Content Delivery Network (CDN) increases content availability and redundancy while reducing bandwidth costs. Generally, static files such as images, and videos are served from CDN. We can use services like Amazon CloudFront or Cloudflare CDN for this use case.

## API gateway
Since we will be using multiple protocols like HTTP, WebSocket, TCP/IP, deploying multiple L4 (transport layer) or L7 (application layer) type load balancers separately for each protocol will be expensive. Instead, we can use an API Gateway that supports multiple protocols without any issues.
API Gateway can also offer other features such as authentication, authorization, rate limiting, throttling, and API versioning which will improve the quality of our services.

We can use services like Amazon API Gateway or Azure API Gateway for this use case.

Detailed Design:

![enter image description here](https://i.ibb.co/FxJDjtN/what-s-app.png)


## Identify Bottlenecks

•	"What if one of our services crashes?"

•	"How will we distribute our traffic between our components?"

•	"How can we reduce the load on our database?"

•	"How to improve the availability of our cache?"

•	"Wouldn't API Gateway be a single point of failure?"

•	"How can we make our notification system more robust?"

•	"How can we reduce media storage costs"?

•	"Does chat service has too much responsibility?"

## Resolve Bottlenecks

•	Running multiple instances of each of our services.

•	Introducing load balancers between clients, servers, databases, and cache servers.

•	Using multiple read replicas for our databases.

•	Multiple instances and replicas for our distributed cache.

•	We can have a standby replica of our API Gateway.

•	Exactly once delivery and message ordering is challenging in a distributed system, we can use a dedicated message broker such as Apache Kafka or NATS to make our notification system more robust.

•	We can add media processing and compression capabilities to the media service to compress large files similar to Whatsapp which will save a lot of storage space and reduce cost.

•	We can create a group service separate from the chat service to further decouple our services.



			Thank You
