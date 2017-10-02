# Messenger
## Scenario
### Core use cases
* One to one chatting
* Group chatting
* User online status

### Common use cases
* History info
* Log in from multiple devices
* Friendship / Contact book

### Estimation
* Requests sent from the client(analyzed from the use cases)
	1. Read (thread_ids, thread_name, thread_last_msg) <- (user_id)
	2. Read (latest_n_msgs(msg, timestamp, sender_id)) <- (thread_id)
	3. Write (sender_id, thread_id, msg, timestamp, new_thread_info)
	4. Read (thread_info) <- (user_id, thread_id)
	5. Write (thread_info, user_id, thread_id)
* DAU: 100M 
* QPS: Suppose a user posts 20 messages / day
	- Average Write QPS = 100M * 20 / 86400 ~ 20k
	- Peak Write QPS = 20k * 5 = 100k
	- If we send 1, 2 perodically -- every 1s, then the read QPS the system needs to support is 100M QPS -- too high and need to use a better way.
* Storage: Suppose A user sends 20 messages / day
	- 100M * 20 * 30 Bytes = 30G new data each day.

## Service
### Message service

## Storage
### Message table
* NoSQL database
	- Large amounts of data
	- No modification required
* Schema

	| Columns      | Type      |                  | 
	|-----------   |-----------|------------------| 
	| message_id   | integer   | sender_id+timestamp. Primary key    | 
	| thread_id    | integer   | the thread it belongs. Foreign key  | 
	| sender_id    | integer   |                  									 | 
	| content   	 | text      |                  									 | 
	| created_time | timestamp |                  									 |	

	* For message_id, the timestamp has the granularity of second, since two messages cannot be sent out by same user within 1 sec. We can use 64bits integer to store the composite id, where first 32bits storing the timestamp and the rest storing the sender_id. If we use epoch time starting from today, then for the next 100years, the time is 86400s/day x 365 x 100 = 3B, smaller than the largest number that can be represented by 32bits(4B, remember this!). The rest of bits can store as many as 4 billion people, which can be more if we use larger data type or reduce the space left for the timestamp.
	A better way might be remove the message_id and use (thread_id, created_time) as the primary key. Most of the NoSQL databases allow composite primary keys(https://aws.amazon.com/blogs/database/choosing-the-right-dynamodb-partition-key/). 

### User-thread table
* SQL database
	- One machine is probably enough. Assuming 8 billion people on earth all use this system. The size of this table = 8 x 10^9 x 10 x 30 = 2TB. 
  - Need to support multiple indices
  - Build the following indices:
    + user_id(+updated_time): Search all threads the user participates, and order by updated time. Since there are usually not many threads for each user, it is okay to query on user_id only and then sort the result in memory.
    + user_id + thread id: Get all detailed info about a thread (e.g. label, participants, and user-related info like mute or not).
    + participants_hash: Find whether a certain group of persons already has a chat group. This is used to prevent from creating another group with same participants, but I think it should be allowed, so this column can be removed.
* Schema

	| Columns          | Type      |                          | 
	|------------------|-----------|--------------------------| 
	| user_id           | integer   | id of a participant in the thread| 
	| thread_id         | integer   | creater_id + created_time | 
	| participants_ids   | text      | json                     | 
	| participants_hash | string    | avoid duplicates threads | 
	| created_time        | timestamp |                          | 
	| updated_time        | timestamp | index=true               | 
	| label            | string    |                          | 
	| mute             | boolean   |                          | 

	- Primary key is user_id + thread_id
		+ Why not use UUID as primary key? Need sharding. Not possible to maintain a global ID across different machines. Use UUID, really low efficiency.
	- If a thread has n participants, when a new thread is created, n rows must be inserted to Thread table with their corresponding user ids. The shared information can be stored in a separate table, but that would make getting full information of a thread slower since it requires two queries. 
	- thread_id can be stored using 64-bits integer, same ways as the message_id.

### Initial solution
* Sender sends message and message receiverId to server
* Server creates a thread for each receiver and sender
* Server creates a new message (with thread_id)
* How does user receives information
	- Pull server every 10 second

## Scale
### Sharding
* Message table
	- Shard by (thread_id, created_time). It can also be used as primary key since it can uniquely identify each message. Using thread_id along as shard key would cause hot spot for writes when a certain thread has many participants talking all the time. However reads using thread_id have to go to all the shards(how to improve this?). 
* User-thread table
	- Shard by userId. There shouldn't be write hot spot since users typically not creating new groups freqently(unless it is a bot). And users typically give even read to this table.

### Speed up with Push service
Even with pre-generated user-message table like the one in Twitter design, it is still slow as the latency includes push time and polling period. 
#### Socket
* HTTP vs Socket
	- HTTP: Only client can ask server for data
	- Socket: Server could push data to client
* What if user A does not connect to server
	- Relies on Android GCM/IOS APNS

#### Push service
##### Initialization and termination
* When a user opens the app, the user connects to one of the socket in Push Service
* If a user is inactive for a long period, drops the connection. 

##### Number of push servers
* Each socket connection needs a specific port. So needs a lot of push servers. 
	- The traffic could be sharded according to user_id

##### Steps
1. User A opens the App, it asks the address of push server.
2. User A stays in touch with push server by socket. 
3. User B sends msg to User A. msg is first sent to the message server.
4. Message service finds the responsible push service according to the user id.
5. Push service sends the msg to A.

### Channel service
* In case of large group chatting
	- If there are 500 people in a group, message service needs to send the message to 500 push service. If most of receivers within push service are not connected, it means huge waste of resources. 
* Add a channel service
	- Each thread should have an additional field called channel. Channel service knows which users are online/offline. For big groups, online users need to first subscribe to corresponding channels. 
		+ When users become online, message service finds the users' associated channel and notify channel service to subscribe. When users become offline, push service knows that the user is offline and will notify channel service to subscribe. 
	- Channel service stores all info inside memory. 

### How to check / update online status
#### Online status pull
* When users become online, send a heartbeat msg to the server every 3-5 seconds. 
* The server sends its online status to friends every 3-5 seconds. 
* If after 1 min, the server does not receive the heartbeat msg, considers the user is already offline. 

#### Online status push
* Wasteful because most users are not online
